# Smart pointers en Vex: `unique<T>` y `shared<T>`

Smart pointers nativos del lenguaje (PrimitiveKind dedicado, no templates) con
cleanup automatico via RAII y move semantics zero-overhead. Soportan deleters
custom para adoptar cualquier recurso del sistema (memoria, file handles,
sockets, handles OS).

---

## Indice

1. [Por que smart pointers en una VM con GC](#1-por-que-smart-pointers-en-una-vm-con-gc)
2. [`unique<T>` - Ownership exclusivo](#2-uniquet---ownership-exclusivo)
3. [`shared<T>` - Reference counting](#3-sharedt---reference-counting)
4. [Construccion: `unique_box` / `shared_box`](#4-construccion-unique_box--shared_box)
5. [Deleters custom: `unique_with` / `shared_with`](#5-deleters-custom-unique_with--shared_with)
6. [`move(p)` - Transferencia de ownership](#6-movep---transferencia-de-ownership)
7. [`ptr_of(p)` / `use_count(s)` - Acceso e inspeccion](#7-ptr_ofp--use_counts---acceso-e-inspeccion)
8. [Smart pointer como return de funcion (Tier 1)](#8-smart-pointer-como-return-de-funcion-tier-1)
9. [Ejemplos: VirtualAlloc, fopen, sockets](#9-ejemplos-virtualalloc-fopen-sockets)
10. [Layout runtime](#10-layout-runtime)
11. [Limitaciones](#11-limitaciones)

---

## 1. Por que smart pointers en una VM con GC

El GC de Vesta libera memoria automáticamente, pero NO ejecuta finalizers
deterministicamente. Para recursos del SO (file descriptors, mutex, sockets,
páginas VirtualAlloc) necesitas liberación EN UN MOMENTO ESPECIFICO (al exit del
scope), no "cuando el GC decida correr". Smart pointers proveen ese RAII
determinista.

Beneficio vs `malloc/free` manual:
- **No olvidas el free**: el compilador inserta cleanup automatico al exit del
  scope (incluido en early returns y exceptions).
- **No double-free**: `move` zerifica el slot origen; cleanups subsecuentes son
  no-op.
- **Sin overhead**: un smart pointer es 8-16 bytes en stack; el cleanup son
  ~3-5 instrucciones VM.

---

## 2. `unique<T>` - Ownership exclusivo

```vex
unique<i32> p = unique_box(42);   // crea + inicializa
i32 v = *ptr_of(p);                // = 42
// al exit del scope: free automatico
```

- Un solo owner del recurso. NO copiable; sólo movible via `move(p)`.
- Cleanup automatico al exit del scope: invoca el deleter del slot.
- Layout: 16 bytes en stack (`[ptr_8 | deleter_8]`).

---

## 3. `shared<T>` - Reference counting

```vex
shared<i32> s = shared_box(78);
i32 u = use_count(s);              // = 1
i32 v = *ptr_of(s);                // = 78
// al exit del scope: refcount-- (si llega a 0, GC libera ctrl block)
```

- Múltiples owners cuentan referencia.
- Layout: 8 bytes en stack (puntero al ctrl block en GcHeap).
- Ctrl block: 24 bytes (`refcount + deleter + payload_inline`).
- El GC libera el ctrl block cuando refcount llega a 0 y no hay roots.

---

## 4. Construccion: `unique_box` / `shared_box`

```vex
unique<i32> u1 = unique_box(42);
unique<i64> u2 = unique_box(100LL);
unique<f64> u3 = unique_box(3.14);

shared<i32> s1 = shared_box(99);
shared<string> s2 = shared_box("hello");
```

**Default deleter**: `unique_box(v)` usa el built-in `RAW_FREE` (free del raw
allocator); `shared_box(v)` usa decrement de refcount (cuando llega a 0, GC
libera).

`unique_box(v)` emite:

```asm
ALLOCA 16              ; slot stack [ptr | deleter]
RAW_ALLOC sizeof(T)    ; aloca memoria host
STORE v en host_ptr    ; inicializa el valor
STORE host_ptr en slot+0
STORE 0 en slot+8      ; sentinel deleter = NULL (= RAW_FREE)
```

---

## 5. Deleters custom: `unique_with` / `shared_with`

Para adoptar CUALQUIER recurso del sistema, especifica el deleter como nombre
de función:

```vex
extern "kernel32.dll" {
    fn VirtualAlloc(u64 addr, u64 size, u32 type, u32 protect) -> u64;
    fn VirtualFree(u64 addr, u64 size, u32 type) -> u32;
}

// Wrapper de 1 arg (porque VirtualFree toma 3):
void release_vmem(i64 p) {
    VirtualFree(p, 0, 0x8000);
}

// Adopta la memoria con su deleter custom:
unique<i64> g = unique_with(VirtualAlloc(0, 4096, 0x3000, 0x04), release_vmem);
// al exit: release_vmem(g) -> VirtualFree
```

**Reglas**:
- El segundo argumento (`deleter`) es un **nombre de función**, no una llamada.
- La función debe tener aridad 1, parámetro compatible con el tipo del primer arg.
- Puede ser Vesta normal O extern wrapper (cualquiera).

**Implementación** (A.35.next): el type checker `unique_with(v, deleter_fn)`
valida que `deleter_fn` sea `IdentExpr` resolviendo a `Function` con aridad 1.
El lowering registra el deleter en la `CleanupAction` SMARTPTR_FREE.
`emit_cleanups_all` distingue 3 rutas: `"free"` (RAW_FREE directo), `"<vesta_fn>"`
(CALLVM al wrapper), `"@extern:<lib>:<fn>"` (CALLN al símbolo nativo).

---

## 6. `move(p)` - Transferencia de ownership

```vex
unique<i32> a = unique_box(42);
unique<i32> b = move(a);          // ownership transferido
// `a` queda invalidado (slot = 0); cleanup de `a` es no-op
i32 v = *ptr_of(b);                // = 42
// al exit: solo `b` libera el recurso
```

**Bytecode**: `move(a)` baja a una sola instrucción VM `mvtake [dst_slot], [src_slot]`
(opcode 0x72, FIXED_4):

```
mov rax, [src_slot]
mov [dst_slot], rax
mov qword [src_slot], 0     ; zerifica el slot origen
```

3 host instrucciones cuando llegue el JIT. Sin LOCK prefix porque move siempre
es intra-thread (no atomic cross-thread necesario).

**Cleanup safety**: como el slot de `a` queda zerificado, cuando el cleanup
automático del scope corre, ve `ptr = 0`. El deleter `RAW_FREE` es null-safe
(retorna sin crash). Los deleters custom deben ser null-safe también o el
cleanup chequea null antes de invocar.

---

## 7. `ptr_of(p)` / `use_count(s)` - Acceso e inspeccion

```vex
unique<i32> p = unique_box(42);
i32* raw = ptr_of(p);              // T*: extrae el puntero sin consumir
i32 v = *raw;                       // = 42; o equivalente: *ptr_of(p)

shared<i32> s = shared_box(100);
i32 v = *ptr_of(s);                 // payload via offset +16 del ctrl block
i32 count = use_count(s);           // refcount actual (lee LOAD del ctrl block)
```

- `ptr_of(p)`: devuelve `T*` (host pointer). NO modifica el smart pointer ni
  transfiere ownership.
- `use_count(s)`: lee el refcount del `shared<T>`. Útil para debug o garantizar
  ownership único (`use_count(s) == 1`).
- **No hay `get()`** (nombre reservado para properties); usar `ptr_of()`.

---

## 8. Smart pointer como return de funcion (Tier 1)

```vex
unique<i32> make_buffer() {
    return unique_box(42);
}

unique<i64> alloc_vmem(u64 size) {
    i64 mem = VirtualAlloc(0, size, 0x3000, 0x04);
    return unique_with(mem, release_vmem);
}

i32 main() {
    unique<i32> a = make_buffer();              // SRET: caller aloca 16 bytes
    unique<i64> b = alloc_vmem(4096);           // SRET: con deleter custom
    return 0;
    // al exit: ambos liberados en orden inverso (LIFO)
}
```

**Tier 1 (A.35.tier1)**: el slot pasa de 8 bytes a **16 bytes** (`[ptr | deleter]`).
Permite que cualquier deleter custom (incluyendo extern wrappers) viaje cross-funcion.

**ABI SRET para FUNCTION smart-ptr**: funciones que retornan `unique<T>` se
transforman en `void f(retbuf:ptr, args...)` con `retbuf` como primer parámetro
hidden de 16 bytes. El caller aloca, pasa puntero. El callee escribe 2 qwords
(ptr + deleter) al retbuf vía `lower_return`. Cero heap extra.

`shared<T>` sigue siendo 8 bytes porque su ctrl_block en GcHeap ya contiene el
deleter.

**Cleanup dispatch dinámico**: cuando `literal_deleter` está vacío (caso SRET:
el smart pointer vino de una factory function), el cleanup LEE el deleter de
`slot+8` y dispatcha en runtime:

```asm
cmpu ptr, 0; je done           ; null = no-op
cmpu deleter, 0; je default    ; sentinel = RAW_FREE
mov r1, ptr; mov r15, 1
callvmr deleter; jmp done      ; deleter custom
default: free ptr
done:
```

---

## 9. Ejemplos: VirtualAlloc, fopen, sockets

### Memoria virtual

```vex
extern "kernel32.dll" {
    fn VirtualAlloc(u64 a, u64 s, u32 t, u32 p) -> u64;
    fn VirtualFree(u64 a, u64 s, u32 t) -> u32;
}

void release_vmem(i64 p) { VirtualFree(p, 0, 0x8000); }

i32 main() {
    unique<i64> mem = unique_with(VirtualAlloc(0, 4096, 0x3000, 0x04), release_vmem);
    // ...usar mem...
    return 0;
    // al exit: VirtualFree(mem, 0, 0x8000)
}
```

### File handles

```vex
extern "msvcrt.dll" {
    fn fopen(u8* path, u8* mode) -> i64;
    fn fclose(i64 fp) -> i32;
}

void close_file(i64 fp) { fclose(fp); }

i32 main() {
    string path = "data.txt";
    string mode = "rb";
    unique<i64> file = unique_with(fopen(str_cstr(path), str_cstr(mode)), close_file);
    // ...leer file...
    return 0;
    // al exit: fclose(file)
}
```

### Sockets, mutex, etc.

Patrón idéntico: wrapper de 1 arg + `unique_with(create_call, wrapper)`.

---

## 10. Layout runtime

### `unique<T>` (Tier 1, 16 bytes en stack)

```
slot[0..7]  = host_ptr al payload (resultado de RAW_ALLOC o adopcion)
slot[8..15] = deleter_addr
              0 = sentinel: usar RAW_FREE
              otro = puntero a funcion VM (deleter custom)
```

### `shared<T>` (8 bytes en stack + 24 bytes en GcHeap)

```
stack slot[0..7] = puntero al ctrl block en GcHeap

ctrl block (24 bytes):
  +0  : refcount (i64)
  +8  : deleter  (i64) - 0 = decrement refcount + GC; otro = deleter custom
  +16 : payload inline (8 bytes; T extra grandes requieren indirección via GC)
```

---

## 11. Limitaciones

1. **Tier 1 actual**: slot de 16 bytes. Tier 2 (con destructor inline + flags
   extra) no implementado todavía.

2. **Tipos > 8 bytes inline**: `unique<MyBigStruct>` con T > 8 bytes requiere
   indirección manual (puntero a T). Phase B candidate para soporte directo.

3. **`shared<T>` no es thread-safe** sobre el refcount (no usa atomic
   increment/decrement). Modelo single-thread por scope; para sharing
   cross-thread usar futures/mailbox.

4. **Cleanup en scope interno** (`{ ... }` dentro de función): NO se emite
   cleanup automático al exit del bloque, sólo al RET de la función. Para
   liberar antes, refactorizar a helper auxiliar cuyo return dispare el
   cleanup, o llamar `move()` para invalidar tempranamente.

5. **No hay `weak_ptr<T>`** equivalente. Para evitar ciclos en `shared<T>`,
   usar referencias raw `T*` (no cuentan en refcount; cuidado con dangling).

---

Ver también: [[BorrowChecker]] (`borrow<T>` para acceso temporal sin transferir
ownership), [[Excepciones]] (cleanup es exception-safe), [[FFI]] (uso con
extern para recursos OS).
