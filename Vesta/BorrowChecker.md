# Borrow checker en Vesta (estilo Rust)

Vesta implementa un **borrow checker estático** al estilo de Rust: tipos `borrow<T>`
(shared, múltiples lectores) y `borrow_mut<T>` (exclusivo, un escritor), con
4 reglas validadas en compile-time y 4 fases avanzadas (F1-F4: NLL, OwnerKind,
reborrow con suspend, lifetime elision).

**Cero overhead runtime**: un borrow ES un host_ptr de 8 bytes, idéntico a un
`T*` plano. Toda la seguridad está en el type checker.

---

## Indice

- [Borrow checker en Vesta (estilo Rust)](#borrow-checker-en-vx-estilo-rust)
 - [Indice](#indice)
 - [1. Por qué un borrow checker](#1-por-qué-un-borrow-checker)
 - [2. `borrow<T>` shared y `borrow_mut<T>` exclusive](#2-borrowt-shared-y-borrow_mutt-exclusive)
 - [3. Builtins: lend, lend\_mut, read\_borrow, write\_borrow](#3-builtins-lend-lend_mut-read_borrow-write_borrow)
 - [4. Las 4 reglas: R1-R4](#4-las-4-reglas-r1-r4)
 - [R1: Exclusividad mutable](#r1-exclusividad-mutable)
 - [R2: Inmutables compatibles](#r2-inmutables-compatibles)
 - [R3: No-uso-mientras-prestado](#r3-no-uso-mientras-prestado)
 - [R4: Lifetime](#r4-lifetime)
 - [5. Fase F1: Non-Lexical Lifetimes (NLL)](#5-fase-f1-non-lexical-lifetimes-nll)
 - [6. Fase F2: OwnerKind tracking](#6-fase-f2-ownerkind-tracking)
 - [7. Fase F3: Reborrow con suspend stack](#7-fase-f3-reborrow-con-suspend-stack)
 - [Reborrow desde shared (caso simple)](#reborrow-desde-shared-caso-simple)
 - [Reborrow desde mutable (con suspend semantics)](#reborrow-desde-mutable-con-suspend-semantics)
 - [Casos cubiertos](#casos-cubiertos)
 - [8. Fase F4: Lifetime elision (regla 1)](#8-fase-f4-lifetime-elision-regla-1)
 - [9. Mensajes de error estilo rustc](#9-mensajes-de-error-estilo-rustc)
 - [10. Limitaciones](#10-limitaciones)
 - [Resumen: tabla de validación](#resumen-tabla-de-validación)

---

## 1. Por qué un borrow checker

Vesta tiene smart pointers (`unique<T>`, `shared<T>`) y referencias raw (`T*`).
Para preservar **memory safety** sin sacrificar performance, el borrow checker
valida en compile-time que las referencias temporales (`borrow<T>`/`borrow_mut<T>`)
no creen aliasing peligroso:

- Dos `borrow_mut<T>` al mismo owner = data race.
- `borrow<T>` activo mientras se hace `move(owner)` = use-after-free.
- `borrow<T>` que sobrevive a su owner local = dangling reference.

Toda esta validación es **compile-time** (errores antes de generar `.velb`).
En runtime, los borrows son punteros normales sin checks.

---

## 2. `borrow<T>` shared y `borrow_mut<T>` exclusive

```vx
i32 main() {
    unique<i32> data = unique_box(42);

    borrow<i32> ro = lend(data); // shared borrow (read-only)
    i32 v = read_borrow(ro); // = 42
    println("${v}");
 
    borrow_mut<i32> rw = lend_mut(data); // exclusive borrow (read-write)
    write_borrow(rw, 100);
    i32 w = read_borrow(rw); // = 100
 
    return read_borrow(rw);
}
```

- `borrow<T>` (shared): múltiples coexisten al mismo owner. Sólo lectura.
- `borrow_mut<T>` (exclusive): uno solo permitido. Lectura + escritura.

**Layout**: ambos son `host_ptr` de 8 bytes (idéntico a `T*`). Cero overhead.

---

## 3. Builtins: lend, lend_mut, read_borrow, write_borrow

| Builtin | Retorno | Descripción |
| :------------------- | :----------------- | :--------------------------------- |
| `lend(owner)` | `borrow<T>` | Crea borrow shared del owner |
| `lend_mut(owner)` | `borrow_mut<T>` | Crea borrow exclusive (requiere R1) |
| `read_borrow(b)` | `T` | Lee el valor apuntado |
| `write_borrow(m, v)` | `void` | Escribe vía mut borrow |

```vx
unique<i32> data = unique_box(0);
borrow_mut<i32> m = lend_mut(data);
write_borrow(m, 42);
i32 v = read_borrow(m); // = 42
```

**Lowering**:
- `lend(owner)`/`lend_mut(owner)` emiten `LOAD slot+0` del owner (para
 unique<T>) o `LOAD slot+0 + ADD 16` (para shared<T>, offset del payload
 inline). Para variables raw con address-taken, devuelven el SSA value
 directo del puntero.
- `read_borrow(b)`/`write_borrow(m, v)` emiten LOAD/STORE i64 con
 `is_host_ptr=true` (el emisor usa `movh`).

---

## 4. Las 4 reglas: R1-R4

### R1: Exclusividad mutable

Si hay un `borrow_mut` activo sobre un owner, NO se permite NINGÚN otro borrow
del mismo owner.

```vx
unique<i32> p = unique_box(42);
borrow_mut<i32> m = lend_mut(p);
borrow<i32> r = lend(p); // ERROR R1: m sigue activo
```

### R2: Inmutables compatibles

N `borrow<T>` (shared) pueden coexistir al mismo owner. Pero NINGÚN `borrow_mut`
puede activarse mientras haya shared activos.

```vx
unique<i32> p = unique_box(42);
borrow<i32> r1 = lend(p);
borrow<i32> r2 = lend(p); // OK R2: dos shared OK
borrow_mut<i32> m = lend_mut(p); // ERROR R2: shared activos
```

### R3: No-uso-mientras-prestado

El owner NO puede ser movido (`move(p)`) ni mutado directamente mientras tenga
borrows activos.

```vx
unique<i32> p = unique_box(42);
borrow<i32> r = lend(p);
unique<i32> q = move(p); // ERROR R3: p tiene borrow activo
```

### R4: Lifetime

Borrows NO pueden escapar via `return` salvo si el owner es Param/Global/Field
(lifetime cubre la función completa).

```vx
borrow<i32> bad() {
    unique<i32> local = unique_box(42);
    return lend(local); // ERROR R4: local muere al ret
}

borrow<i32> ok(unique<i32> param) {
    return lend(param); // OK R4: param vive en el caller
}
```

---

## 5. Fase F1: Non-Lexical Lifetimes (NLL)

Los borrows se "liberan" en compile-time tras su **ÚLTIMO USO** real, no al exit
del scope.

```vx
unique<i32> p = unique_box(42);
borrow<i32> r = lend(p);
i32 v = read_borrow(r); // último uso de `r`
// -> NLL libera `r` aquí
unique<i32> q = move(p); // OK: r ya no está activo
```

**Implementación**: pre-pase DFS en `compute_borrow_last_uses` que numera cada
statement (saltando BlockStmt para coincidir con check_stmt) y registra el
índice del último uso de cada borrow. El índice se guarda en `pending_last_use_`
y se consume en `register_borrow`. `advance_stmt` dropea automáticamente cualquier
borrow cuyo `last_use_idx < current_stmt_idx_`.

**Param borrows self-referenciales** (owner == borrower) NO se dropean por NLL
(lifetime cubre la función).

---

## 6. Fase F2: OwnerKind tracking

Enum `OwnerKind` en `BorrowRecord`: `Local`, `Param`, `Global`, `Field`.

- **Local**: declarado en la función. Lifetime hasta exit del scope.
- **Param**: parámetro de la función. Lifetime hasta el RET.
- **Global**: variable estática global. Lifetime del programa.
- **Field**: campo de struct/class. Lifetime del objeto contenedor.

`on_borrow_escape` (regla R4) consulta `owner_kind_of(owner)`:

- Si Param/Global/Field -> permite escape (lifetime cubre la función).
- Si Local -> error con sugerencia: "cambia a `unique<T>`/`shared<T>` o devuelve
 read_borrow".

```vx
borrow<i32> ok_param(unique<i32> p) {
    return lend(p); // OK: owner Param
}

borrow<i32> bad_local() {
    unique<i32> local = unique_box(42);
    return lend(local); // ERROR: owner Local
}
```

Param borrows del tipo `borrow<T>` se auto-registran self-referenciales
(`owner == borrower`) para que `return p` con `p: borrow<T>` no de "owner
desconocido" en escape check.

---

## 7. Fase F3: Reborrow con suspend stack

`lend(borrow_var)` y `lend_mut(borrow_var)` derivan un nuevo borrow del **root
owner** siguiendo la cadena. `root_owner_of(borrower)` camina recursivamente
`borrows_->owner` hasta llegar a un nombre no-borrow (cota dura 64 niveles).

### Reborrow desde shared (caso simple)

```vx
borrow<i32> r = lend(p);
borrow<i32> r2 = lend(r); // shared reborrow: shared_count++
```

### Reborrow desde mutable (con suspend semantics)

```vx
borrow_mut<i32> m = lend_mut(p); // m mut activo
borrow_mut<i32> m2 = lend_mut(m); // reborrow mut desde m
// suspende `m` (push a stack)
// registra `m2` como mut activo
write_borrow(m2, 100);
// last use de m2 -> drop -> pop suspend stack -> `m` reactivado
write_borrow(m, 200); // OK: m restaurado
```

**Implementación**: cuando la fuente del `lend/lend_mut` es un `borrow_mut`
activo, `suspend_for_reborrow(source)` push el estado actual del owner
(kind/shared_count/loc_taken/borrower_name) a `BorrowRecord::suspend_stack`
(vector) y deja el owner en estado None temporalmente. Después `on_lend`
registra el reborrow nuevo sin violar R1.

Soporta cadenas arbitrarias: `m1 -> m2 -> m3 -> ...` con LIFO restore al
dropear cada reborrow.

### Casos cubiertos

- `lend(borrow_shared)` -> shared reborrow (sin suspend, increment shared_count).
- `lend(borrow_mut)` -> shared reborrow de mut (suspend mut, registrar shared).
- `lend_mut(borrow_mut)` -> mut reborrow (suspend mut, registrar nuevo mut).
- `lend_mut(borrow_shared)` -> rechazado por type checker (upgrade shared->mut
 prohibido).

---

## 8. Fase F4: Lifetime elision (regla 1)

Cuando una función `fn(borrow<T>) -> borrow<R>` devuelve un borrow, el output
hereda el `borrow_owner_source` del input.

```vx
borrow<i32> first_view(borrow<i32> data) {
    return data; // F4: output hereda owner de input
}

i32 main() {
    unique<i32> p = unique_box(42);
    borrow<i32> view = first_view(lend(p));
    // view efectivamente apunta a `p`; el borrow checker lo sabe
    return read_borrow(view);
}
```

Equivalente a la **regla 1** de lifetime elision en Rust (`fn f(&'a T) -> &'a U`
con un solo input borrow).

Implementación: `Expr::borrow_owner_source` (string) se propaga a través de
`lend`, `read_borrow` y llamadas a funciones de aridad 1-input-borrow. Permite
factories de borrows encadenadas:

```vx
borrow<i32> third_view(borrow<i32> b) { return b; }
borrow<i32> second_view(borrow<i32> b) { return third_view(b); }
borrow<i32> first_view(borrow<i32> b) { return second_view(b); }
// El checker traza el owner a través de 3 niveles
```

---

## 9. Mensajes de error estilo rustc

```
error: no se puede mover 'owner' porque tiene un prestamo shared activo
    --> 116_borrow_err_move_while_borrowed.vx:6:25
    |
note: prestamo shared por 'r' tomado aqui
    --> 5:25
    |
note: move(...) del owner aqui
    --> 6:25
    |
note: sugerencia: termina el scope del borrow antes de mover el owner,
    o usa 'shared<T>'.
```

Cada error cita 2-3 puntos relevantes (creación del borrow + uso conflictivo)
con líneas + columnas + sugerencias contextuales. Inspirado en rustc.

---

## 10. Limitaciones

1. **Lifetimes anotadas (`&'a T`) no soportadas**: borrows a través de
 fronteras de función con múltiples input-borrows requieren restricciones
 estáticas no implementadas (la regla 1 del elision-set cubre 1-input;
 reglas 2 y 3 no implementadas).

2. **Escape via field/slot/deref**: detectado solo via escape_detection (A.27),
 no via borrow_checker directo. Algunos casos podrían pasar (ej. `this.f =
 borrow_local` con `this` de tipo Class).

3. **Owner Global/Field tracking**: el OwnerKind soporta Global/Field pero los
 call sites no los marcan automáticamente todavía (default a Local,
 conservador).

4. **Mutaciones via setter**: `obj.prop = v` con borrow activo no se intercepta
 (requeriría tracking del receiver kind).

5. **Multi-input-borrows con elision regla 2 (`&self`)**: no implementada (la
 regla 1 con 1 input cubre la mayoría de casos comunes en práctica).

---

## Resumen: tabla de validación

| Acción | Permitida si... |
| :-------------------------------- | :-------------------------------------- |
| `borrow<T> r = lend(owner)` | NO hay `borrow_mut` activo del owner |
| `borrow_mut<T> m = lend_mut(owner)` | NO hay NINGÚN borrow activo del owner |
| `read_borrow(b)` | borrow no dropeado por NLL |
| `write_borrow(m, v)` | `m` es `borrow_mut` no dropeado |
| `move(owner)` | NO hay borrows activos del owner |
| `return borrow<T>` | owner es Param/Global/Field (R4 + F2) |
| `lend(borrow_mut)` (reborrow) | suspend del mut + registrar shared |
| `lend_mut(borrow_mut)` (reborrow) | suspend del mut + registrar nuevo mut |

---

Ver también: [[SmartPointers]] (`unique<T>`/`shared<T>` como owners típicos),
[[TiposDatos]] (modelo de punteros raw vs borrows), [[Excepciones]] (cleanup
seguro de borrows en early return).
