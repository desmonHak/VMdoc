# Memoria compartida cross-process (Erlang/Java-style)

heap COMPARTIDO entre todos los procesos de la
misma `VM`, manteniendo el modelo actor por defecto (cero overhead local).


---

## Principio rector

Sharing es propiedad del **sitio de allocacion**, NO del tipo. La stdlib y el
codigo del usuario se escriben **sin annotations** especificas; el usuario
decide en el call site si una instancia es local o compartida.

```vex
ArrayList<i64> local = new ArrayList<i64>(); // gc_heap local (default)
shared ArrayList<i64> visible = new ArrayList<i64>(); // SharedHeap directo
spawn { visible.push(42); }; // OK cross-process
```

Cero overhead para el caso por defecto (single-process).

---

## Dos ejes ORTOGONALES

Dos conceptos distintos que NO se colapsan:

| | `shared` modifier | `synchronized` block |
| :----------- | :------------------------ | :---------------------------- |
| Controla | UBICACION del objeto | ATOMICIDAD de critical section |
| Sin el | Child no puede deref el host_ptr del padre (gc_heap per-process) | Otro thread puede leer/escribir entre tus ops |
| Coste | One-time alloc (~100 ns) | 1 CAS per acquire (~5 ns) |
| Equivalente Rust | `Arc<T>` | `Mutex<T>::lock()` |

Casos legales:

- `shared T x = new T()` **sin** `synchronized` -> datos read-only compartidos
 (config, lookup tables).
- `shared_malloc` + `atomic_*` sin `synchronized` -> contadores lock-free.
- Sin `shared`, **con** `synchronized` -> mutex intra-proceso (proteger contra
 reentrant signal handlers).
- **Ambos** -> operaciones compuestas read-modify-write cross-process.


---

## Modificador `shared` en var-decl

Cuando declaras una variable con `shared`, el constructor `new T()` despacha al
`newobjs` (opcode 0xA6) en lugar del `newobj` normal. El objeto se aloca
directamente en el `SharedHeap` y se registra en `SharedHandleTable`.

```vex
class Counter {
    public i64 value;
    public Counter() { this.value = 0; }
}

i32 main() {
    shared Counter c = new Counter(); // aloca en SharedHeap
    spawn {
        c.value = c.value + 1; // visible al spawn (mismo host_ptr)
    };
    return 0;
}
```

### Bit-31 de GcHandle

Los handles del `SharedHandleTable` tienen el **bit 31 set** (constante
`SHARED_HANDLE_BIT`). El opcode `deref` y todas las operaciones GC consultan
este bit para dispatchear al heap correcto sin coste extra (branch predicted
always-not-taken para handles locales).

```cpp
// En gc_heap.cpp::deref()
if (h & SHARED_HANDLE_BIT) {
    return owner_proc_->scheduler.vm_reference
    .shared_handle_table.lookup(h);
}
// resto: lookup local
```

### Builtin `is_shared(x) -> bool`

Test compile-time + runtime para distinguir handles shared de locales. Una
sola instr (test bit 31).

---

## Builtins runtime: `share()` / `unshare()`

Para promocion **in-place** (mover un objeto local al SharedHeap o viceversa):

```vex
ArrayList<i64> xs = new ArrayList<i64>(); // local
xs.push(1); xs.push(2);

share(xs); // gcpromote: copia a SharedHeap
spawn { xs.push(3); }; // ya visible cross-process

ArrayList<i64> copy = unshare(xs); // gcdemote: deep-copy a local
if (is_shared(xs)) { println("aun compartido"); }
```

**Opcode `gcpromote` (0xA7)**: copia el ObjectHeader + fields del objeto local
al SharedHeap, registra en SharedHandleTable, actualiza el handle del binding
para apuntar al nuevo objeto. Idempotente si ya esta shared (no-op).

**Opcode `gcdemote` (0xA8)**: simetrico (shared -> local).

---

## Tipos auxiliares: atomic primitives + shared_malloc

Para casos donde no es practico envolver primitivos en clases:

```vex
i64 ptr = shared_malloc(64); // raw host memory shared, sync manual
atomic_store_i64(ptr, 42);
i64 v = atomic_load_i64(ptr);
i64 old = atomic_cas_i64(ptr, 42, 100); // CAS
i64 prev = atomic_add_i64(ptr, 1); // fetch_add atomico
shared_free(ptr);
```

### Opcodes

| Opcode | Mnemonico | Descripcion |
|:------:|:------------|:------------|
| 0xA9 | `atomicld` | Atomic load i64 desde host_ptr (acquire) |
| 0xAA | `atomicst` | Atomic store i64 a host_ptr (release) |
| 0xAB | `atomiccas` | Atomic CAS i64; R0 = old value |
| 0xAC | `atomicadd` | Atomic fetch_add i64; R0 = old value |

---

## Tabla de tipos compartibles

| Categoria | Compartible | Mecanismo |
| :--------------------------------- | :---------: | :------------------------------- |
| Clase GC (`Counter`, `ArrayList<T>`) | si | `shared T x = new T()` o `share(x)` |
| Primitivos (`i64`, `bool`, `f64`) | si | `SharedBox<T>` o `Atomic<T>` |
| Strings (`string`) | si | `shared string s = "..."` (StringObject en SharedHeap) |
| Arrays managed (`Array<T>`) | si | `shared Array<i64>` |
| Host raw memory (`T*`) | si | `shared_malloc(size)` + `atomic_*` |
| Closures (function values) | si | `shared fn(...) -> R f = (x) => ...` (env block en SharedHeap) |
| Smart pointers (`unique<T>`) | NO | move-only single-owner por diseno |
| `VirtualPtr<T>` (VM stack) | NO | stack VM es por-proceso; el compilador rechaza con error |
| Stack locals (`i64 x = ...`) | NO | usar `SharedBox<i64>` |

---

## ABI v3.1 del ObjectHeader

`monitor_word` es un **atomic uint64** que reemplaza los antiguos campos
separados `owner_pid + lock_depth + _mon_pad` de v2.

```
offset 0- 7 class_ptr (8B) -- puntero a ClassInfo
offset 8-11 flags (4B) -- OBJ_FLAG_*
offset 12-15 hash_code (4B) -- identidad del objeto
offset 16-23 monitor_word (8B) -- atomic, packed:
    bits 0-47: owner_encoded (encoded_pid, 48 bits)
    bits 48-63: lock_depth (uint16, 16 bits)
```

### Por que 48-bit owner_encoded (no 32-bit local_pid)

**Bug descubierto**: con `monitor_word.owner` como `uint32_t local_pid`,
dos workers en distintos schedulers (sched 0 y sched 1) podian tener ambos
`local_pid=1`. Cuando worker A tenia el monitor con `monitor_word=(1, 1)`, B
intentaba `monitor_try_acquire`: CAS fallaba poblando `expected.owner=1`, luego
B comparaba `monitor_owner(expected) == local_pid` -> **`1 == 1` -> TRUE** ->
B creia que era reentrante. Resultado: ambos threads dentro del critical
section -> data loss en RMW (~1.7% perdido bajo contencion 4-way).

**Fix**: el `monitor_word` ahora almacena el `encoded_pid = (scheduler_id << 32) | local_pid`
(48 bits, globalmente unico). Cubre 65535 schedulers x 4G procs/scheduler.
`lock_depth` cae a 16 bits (max 65535 reentrancias - sobra).

```cpp
static constexpr uint64_t MONITOR_OWNER_MASK = 0x0000FFFFFFFFFFFFULL;

static constexpr inline uint64_t monitor_make(uint64_t owner_encoded,
uint32_t depth) noexcept {
    return (owner_encoded & MONITOR_OWNER_MASK)
    | (static_cast<uint64_t>(depth & 0xFFFFu) << 48);
}
```

`GcHeap::monitor_try_acquire`/`monitor_release` ahora reciben
`uint64_t owner_encoded` (en vez de `uint32_t local_pid`). Callers
(`exec_instr_monenter`/`exit`/`wait` + `vrt_monitor_enter`/`exit`) pasan
`encode_pid(vm)` directamente.

---

## SharedHeap: slab allocator lock-free con growth dinamico

12 size-classes (potencias de 2 de 16 B a 32 KB). Cada slab tiene una pila
Treiber lock-free con tag ABA de 16 bits en los bits altos del puntero
(canonical 48-bit addressing en x86-64 y AArch64).

### Growth dinamico

Antes: cada slab tenia `slot_count` FIJO (16-32K slots por clase) -> maximo
~6 MB total -> programas con millones de objetos `shared` agotaban el primer
chunk y `alloc()` devolvia nullptr.

Ahora: cada `SharedSlab` mantiene `chunks[SHARED_MAX_CHUNKS_PER_SLAB=256]`
de rangos lazy-allocados. Cuando `alloc()` encuentra `free_head==0`, llama
`grow_slab(cls)` que:

1. Toma el flag `growing` via CAS (sin pthread_mutex).
2. Aloca otro chunk de la misma talla via `vm::allocate_memory`.
3. Thread sus slots al free list via push masivo CAS.
4. Publica el rango del chunk en `chunks[]`.
5. Limpia el flag.

Los hilos perdedores del CAS spin-waitean con `_mm_pause()` / `yield` hasta
release; tras la espera reintentan el pop.

**Memoria maxima total** ahora ~1.5 GB (variable segun clase). `bench_shared_alloc`
con 1M+1M iter completa en ~1 s sin OOM.

### Costes (sin contencion, x86-64 moderno)

- `alloc` fast path: **~5 ns** (1 CAS exitoso).
- `alloc` con growth: ~100-200 ns (one-shot por chunk; amortizado).
- `free`: ~5 ns (1 CAS).
- `contains()`: O(12 size-classes * num_chunks); ~3-30 ns segun growth.
- `deref` local: ~5 ns.
- `deref` shared: **~7 ns** (1 branch + 2 atomic loads: chunk + slot).
- `monenter` libre: **~5 ns** (1 CAS atomic acquire).
- `monenter` contended: ~30 ns (algunas retries CAS).
- `share(obj)`: ~100-200 ns (alloc + memcpy + register). Una sola vez por objeto.
- `shared_gc_collect()`: O(N_handles + total_field_bytes); ~50 us para 100 objetos.

Sin contencion: **0% overhead** vs codigo single-process.

---

## SharedHandleTable: registry lock-free de handles compartidos

- 16384 slots por chunk + 256 chunks max = 4.19M handles soportados.
- Bit 31 del GcHandle uint32 indica "shared" -> 2^31 = 2.14G handles potenciales.
- Lock-free Treiber stack del free list con tag ABA.
- Lazy chunk allocation (solo cuando se necesita).

API publica:

```cpp
uint32_t register_object(uint8_t *host_ptr, uint32_t size_bytes); // alloc handle
uint8_t *lookup(uint32_t handle); // resolver
void unregister(uint32_t handle); // free
```

### Mark/sweep STW

Bitmap lazy paralelo a los slots (1 bit por slot, 2 KB por chunk). Cero
overhead durante operacion normal; solo se usa durante `shared_gc_collect()`.

Protocolo:

1. `clear_marks()`: zero el bitmap (start of GC).
2. **Mark conservativo**: scan stacks + GP regs de TODOS los procesos vivos,
 buscando handles validos (bit 31 set).
3. **Mark transitivo**: BFS via fields del objeto (igual que el GC local).
4. **Sweep**: para cada slot vivo NO marcado, llama `shared_heap.free(host_ptr)`
 + `unregister(handle)`.

### STW multi-scheduler coordination

Cuando `shared_gc_collect()` se invoca, todos los schedulers cooperan:

- Variable atomica `stw_request_flag` y `stw_ack_counter`.
- El proceso solicitante setea `stw_request_flag = 1`.
- Cada `Scheduler::run_loop` poll el flag entre instrucciones (safe-point natural).
- Schedulers con procesos vivos hacen ACK; idle workers se excluyen del conteo.
- Cuando `stw_ack_counter == N - 1` (donde N es schedulers con procesos vivos),
 el GC procede.
- Tras sweep, el solicitante despierta todos via condvar.

Timeout defensivo: 5 segundos si un syscall externo bloquea indefinidamente.

---

## Opcodes añadidos por

| Opcode | Mnemonico | Aridad | Descripcion |
|:------:|:-----------|:------:|:------------|
| 0xA6 | `newobjs` | ONE | NEWOBJ en SharedHeap + register en SharedHandleTable. Retorna handle con bit 31. |
| 0xA7 | `gcpromote`| TWO | Copia objeto local -> SharedHeap (idempotente). |
| 0xA8 | `gcdemote` | TWO | Copia objeto shared -> gc_heap local (idempotente). |
| 0xA9 | `atomicld` | TWO | Atomic load i64 desde host_ptr (acquire). |
| 0xAA | `atomicst` | TWO | Atomic store i64 a host_ptr (release). |
| 0xAB | `atomiccas`| THREE | Atomic CAS i64; R0 = old value. |
| 0xAC | `atomicadd`| TWO | Atomic fetch_add i64; R0 = old value. |
| 0xAD | `sharedstat`| TWO | Introspect (op=0 live_count, op=1 bytes, op=2 GC collect). |

Total: 8 opcodes en la tabla extendida.

---

## Tests y verificacion

| Test | Que valida |
| :---------------------------------- | :------------------------------------------------ |
| `tests/gc/test_shared_heap.cpp` | 8073 checks: alloc/free, ABA-safety, multi-thread stress, growth dinamico |
| `tests/gc/test_shared_handle_table.cpp` | 23 checks: register/lookup/unregister, lock-free correctness |
| `tests/gc/test_monitor_cas.cpp` | 23 checks: monitor_try_acquire/release lock-free CAS |
| `tests/gc/test_wait_table.cpp` | 22 checks: per-bucket spinlock + multi-bucket |
| `examples_codes_vex/166_z_shared_memory.vex` | 5 escenarios integradores (synchronized + atomics + introspect) |
| `examples_codes_vex/167_z_gc_sweep.vex` | GC sweep colecta huerfanos, preserva rooted |
| `examples_codes_vex/benchmark/bench_shared_alloc.vex` | Throughput SharedHeap vs gc_heap local (1M+1M iter) |
| `examples_codes_vex/benchmark/bench_shared_contention.vex` | Monitor cross-scheduler (4 workers x 100K) |
| `examples_codes_vex/benchmark/bench_shared_gc.vex` | GC sweep latency |
| `examples_codes_vex/benchmark/bench_shared_stw_impact.vex` | STW pause impact en multi-thread |



- **Vex e2e**: 237 pasos OK / 0 fallidos.
- **Tests Z unitarios**: 8141 PASS (SharedHeap 8073 + monitor CAS 23 + WaitTable 22 + SharedHandleTable 23).
- **t13 cross-process**: cerrado, R0=42.
- **`synchronized(shared) --schedulers 4`**: 400000 exacto en 3/3 runs (data loss eliminada).

---

## Garantias

- **Stdlib annotation-free**: `ArrayList<T>` y demas se escriben SIN annotations
 shared. El compilador dispatcha `new T()` segun el contexto.
- **Mismo tipo, instancias mixtas**: `Counter` puede tener instancias local
 (default) Y shared (`shared Counter c = ...`) simultaneamente.
- **Cero overhead local**: bit-31 dispatch en `deref` es 1 branch predicted
 always-not-taken; ~0 ns para handles locales.
- **Portable a C**: todos los componentes son POD + `_Atomic` C11;
 layouts ABI estables.
- **JIT-friendly**: `monenter` baja a `lock cmpxchg [obj+16], rdx` (1 instr
 CAS host); `deref` shared baja a `test eax, 0x80000000; jne shared_path`
 con shared_path inline.
- **GC sin leaks**: `shared_gc_collect()` libera objetos no
 alcanzables; STW coordinado funciona single Y multi-scheduler.

---

## Limitaciones conocidas


### Remanente (no bloqueante)

- **Compactacion del SharedHeap**: el GC del SharedHeap es non-moving (mark+sweep
 sin compactacion). Programas que crean y destruyen muchos objetos pueden
 fragmentar el slab. Mitigacion: el slab allocator reusa slots libres
 agresivamente; la fragmentacion solo afecta a alocaciones grandes que crucen
 size-classes. La compactación completa queda como trabajo futuro.

---

## Ver tambien

- [[../SetInstruccionesVM/MONITOR.md]] - opcodes monenter/monexit/monwait/monnoti/monnota
- [[../Hilos/Multihilo.md]] - modelos de concurrencia (green threads + JIT)
- [[../SetInstruccionesVM/GC/Allocator crudo para FFI y memoria manual.md]] - allocator raw
- [[../../BENCHMARKS.md]] - benchmarks de SharedHeap vs local
