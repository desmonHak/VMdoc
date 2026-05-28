# Opcodes de memoria compartida cross-process

Conjunto de 8 opcodes en la tabla extendida que dan soporte a la memoria
compartida cross-process. Complementan a `MONENTER`/`MONEXIT`/
`MONWAIT`/`MONNOTI`/`MONNOTA` (que funcionan transparentes con handles
shared via dispatch por bit-31).

| Instruccion | opcode0 | opcode1 | Aridad | Tamaño | Descripcion breve |
| :----------- | :-----: | :-----: | :----: | :-----: | :--------------------------------------------- |
| `newobjs` | 0x00 | 0xA6 | ONE | 4 bytes | NEWOBJ en SharedHeap + register en SharedHandleTable |
| `gcpromote` | 0x00 | 0xA7 | TWO | 4 bytes | Copia objeto local -> SharedHeap (idempotente) |
| `gcdemote` | 0x00 | 0xA8 | TWO | 4 bytes | Copia objeto shared -> gc_heap local (idempotente) |
| `atomicld` | 0x00 | 0xA9 | TWO | 4 bytes | Atomic load i64 desde host_ptr (acquire) |
| `atomicst` | 0x00 | 0xAA | TWO | 4 bytes | Atomic store i64 a host_ptr (release) |
| `atomiccas` | 0x00 | 0xAB | THREE | 4 bytes | Atomic CAS i64; R0 = old value |
| `atomicadd` | 0x00 | 0xAC | TWO | 4 bytes | Atomic fetch_add i64; R0 = old value |
| `sharedstat` | 0x00 | 0xAD | TWO | 4 bytes | Introspect (op=0 live_count, op=1 bytes, op=2 GC) |

Implementacion: `src/runtime/exec_instruction_gc.cpp` .

---

## Concepto accesible

VestaVM tiene dos heaps:

- **gc_heap local**: privado de cada `ProcessVM`. Default (sin annotations).
 Maximo throughput single-process; cero overhead.
- **SharedHeap global**: visible para todos los procesos de la MISMA `VM`.
 Allocator slab lock-free; cada objeto tiene handle con bit 31 set para
 distinguirlo de los handles locales.

Los opcodes de esta seccion permiten:

1. Alocar directamente en el SharedHeap (`newobjs`).
2. Mover objetos entre los dos heaps (`gcpromote` / `gcdemote`).
3. Operaciones atomicas lock-free sobre host pointers (`atomicld` / `atomicst`
 / `atomiccas` / `atomicadd`).
4. Introspeccion del SharedHeap + trigger manual de GC (`sharedstat`).

---

## `newobjs r_cls` (0x00 0xA6)

Identico a NEWOBJ pero usa `SharedHeap` + `SharedHandleTable`.

**Codificacion binaria** (FIXED_4, 4 bytes):

```text
+--------+--------+--------+--------+
| 0x00 | 0xA6 | 0x00 | r_cls |
+--------+--------+--------+--------+
    byte0 byte1 byte2 byte3
```

**Algoritmo**:

1. Resolver `cls = registers.regs[r_cls].qword()` como `ClassInfo*`.
2. Si `cls == nullptr`: retornar `SHARED_NULL_HANDLE` (0) en R0.
3. Llamar `vm_ref.shared_heap.alloc(cls->instance_size)` para reservar slot.
4. Si OOM: retornar `SHARED_NULL_HANDLE`.
5. Zero-init del payload completo (incluye ObjectHeader + fields).
6. Inicializar `ObjectHeader`: `class_ptr = cls`, `flags = OBJ_FLAG_GC_OWNED`,
 `monitor_word = 0` (libre).
7. Llamar `vm_ref.shared_handle_table.register_object(payload, size)` para
 obtener `GcHandle h` con bit 31 set.
8. Si registro falla (tabla llena): liberar slot y retornar `SHARED_NULL_HANDLE`.
9. Setear `hdr->hash_code = (uint32_t)h` (identidad estable).
10. Escribir `h` en R0.

**Coste**: ~10-15 ns (1 CAS en SharedHeap slab + 1 CAS en SharedHandleTable
free list + ~5 stores).

---

## `gcpromote r_dst, r_src` (0x00 0xA7)

Promociona un objeto del `gc_heap` local al `SharedHeap`. Idempotente: si
ya esta shared, no-op.

**Algoritmo**:

1. Leer `host_ptr = registers.regs[r_src]`.
2. Si `host_ptr == nullptr`: setear R0 = 0 y `r_dst` = 0.
3. Leer `hdr->hash_code`; si bit 31 esta set, ya esta shared -> retornar
 `host_ptr` sin tocar (no-op idempotente).
4. Localizar el objeto local via `gc_heap.handle_for_ptr(host_ptr)`.
5. Alocar nuevo slot en SharedHeap del mismo `size`.
6. Memcpy del ObjectHeader + fields del local al shared.
7. Resetear `monitor_word = 0` en el nuevo slot (no heredar lock state).
8. Llamar `shared_handle_table.register_object(new_ptr, size)` para obtener
 nuevo handle `h_shared` con bit 31.
9. Setear `hdr->hash_code = (uint32_t)h_shared` en el nuevo objeto.
10. Escribir `new_ptr` en `r_dst` y `h_shared` en R0.

**Nota**: el caller debe actualizar referencias al objeto viejo si las hay
(el binding del Vex local se actualiza automaticamente en el lowering de
`share()`). El objeto local queda en su sitio y sera colectado por el GC
local en el siguiente ciclo si nadie lo referencia.

---

## `gcdemote r_dst, r_src` (0x00 0xA8)

Simetrico a `gcpromote`. Copia un objeto shared al gc_heap local.

Casos de uso: cuando se quiere una copia local "fresca" para mutar sin
afectar a otros procesos, o para forzar liberacion deterministica del slot
shared.

Idempotente: si NO esta shared, no-op.

---

## Atomic primitives (0x00 0xA9 - 0x00 0xAC)

Operaciones lock-free sobre i64 host_ptrs. Cualquier alineacion 8-byte
sirve. Los memory orders son fijos (no parametrizables): acquire para
load, release para store, acq_rel para CAS y fetch_add.

### `atomicld r_dst, r_addr` (0x00 0xA9)

```c
r_dst = atomic_load_explicit((_Atomic int64_t *) r_addr,
memory_order_acquire);
```

### `atomicst r_addr, r_val` (0x00 0xAA)

```c
atomic_store_explicit((_Atomic int64_t *) r_addr,
r_val,
memory_order_release);
```

### `atomiccas r_addr, r_expected, r_desired` (0x00 0xAB)

```c
int64_t expected = r_expected;
atomic_compare_exchange_strong_explicit(
(_Atomic int64_t *) r_addr,
&expected,
r_desired,
memory_order_acq_rel,
memory_order_acquire);
R0 = expected; // old value antes del CAS (o el actual si fallo)
```

### `atomicadd r_addr, r_delta` (0x00 0xAC)

```c
R0 = atomic_fetch_add_explicit((_Atomic int64_t *) r_addr,
r_delta,
memory_order_acq_rel);
```

**Cobertura JIT**: en x86-64 estas bajan a una sola instruccion host:
`mov` (load acquire), `mov` (store release - TSO model), `lock cmpxchg`
(CAS), `lock xadd` (fetch_add).

---

## `sharedstat r_dst, r_op` (0x00 0xAD)

Introspeccion + control del SharedHeap. El registro `r_op` indica que
operacion:

| `r_op` | Que hace | R0 / `r_dst` |
| :----: | :--------------------------------------------- | :-------------------------- |
| 0 | live_count: numero de handles vivos en SharedHandleTable | uint32 en `r_dst` |
| 1 | total_allocated_bytes en SharedHeap | uint64 en `r_dst` |
| 2 | Triggerar `shared_gc_collect()` (mark+sweep STW) | 1 si ok, 0 si error |

Builtins Vex equivalentes:

- `shared_heap_live_count() -> u32`
- `shared_heap_total_bytes() -> u64`
- `shared_gc_collect() -> bool`

---

## Layout del ObjectHeader (ABI v3.1, 24 bytes)

El subsistema de memoria compartida requiere el header v3.1 con
`monitor_word` atómico, y usa un `encoded_pid` de 48 bits.

```cpp
struct alignas(8) ObjectHeader {
    ClassInfo *class_ptr; ///< 8 bytes
    uint32_t flags; ///< 4 bytes - OBJ_FLAG_*
    uint32_t hash_code; ///< 4 bytes - identidad + bit 31 si shared
    std::atomic<uint64_t> monitor_word; ///< 8 bytes - packed (owner_encoded:48 | depth:16)
};
```

`hash_code` con bit 31 set indica que el objeto vive en SharedHeap. Este
bit lo usa `gc_heap.handle_for_ptr` para dispatchear al
`shared_handle_table.lookup` correcto.

Ver [[../PhaseZ/SharedMemory.md]] para el modelo completo y
[[MONITOR.md]] para los detalles de `monitor_word`.

---

## Errores comunes y buenas practicas

1. **Modificar `monitor_word` directamente**: prohibido. Usar SIEMPRE
 `monenter`/`monexit`/`monwait` (o las APIs C `vrt_monitor_*` del JIT).

2. **`atomic_*` sobre punteros no alineados a 8 bytes**: comportamiento
 indefinido. Verificar que `r_addr` es 8-byte aligned.

3. **`atomic_*` cross-shared/local memory**: no hay diferencia desde el
 punto de vista del opcode. El host pointer es plain memory; el CPU se
 encarga del ordering.

4. **`shared_gc_collect()` desde un hilo bloqueado en monenter**: deadlock.
 El GC necesita STW; un hilo bloqueado nunca llega al safe-point. Solo
 invocar GC desde codigo que progresa.

5. **Olvidar promote antes de spawn**: si el spawn body intenta hacer
 `synchronized(obj)` sobre un objeto LOCAL del padre, el `gchandle` del
 child no encuentra el objeto en su gc_heap local -> deadlock silencioso
 (`monenter` sobre handle 0). El escape analysis emite warning
 compile-time pidiendo refactorizar a `shared`.

---

## Ejemplo: counter atomico cross-process

```vex
i32 main() {
    i64 counter_ptr = shared_malloc(8); // 8 bytes raw host memory
    atomic_store_i64(counter_ptr, 0); // inicializar a 0

    i64 parent_pid = pid();
    // 4 workers concurrentes que incrementan atomicamente
    i64 w1 = spawn {
        for (i32 i = 0; i < 100000; i++) {
            atomic_add_i64(counter_ptr, 1);
        }
        msgsend(parent_pid, 1);
    };
    // ... w2, w3, w4 ...

    // Join
    msgrecv(); msgrecv(); msgrecv(); msgrecv();

    i64 final_value = atomic_load_i64(counter_ptr); // = 400000 exacto
    shared_free(counter_ptr);
    return (i32) final_value;
}
```

Sin necesidad de `monenter`/`monexit`: el `atomic_add_i64` baja a una sola
instruccion `lock xadd` host que garantiza atomicidad. Throughput: ~150M
ops/sec por core en x86-64 moderno.

---

## Ver tambien

- [[MONITOR.md]] -- monenter/monexit/monwait/monnoti/monnota (sync sobre objetos)
- [[../PhaseZ/SharedMemory.md]] -- modelo de memoria cross-process completo
- [[GC/Allocator crudo para FFI y memoria manual.md]] -- raw allocator local
- [[../Hilos/Multihilo.md]] -- modelos de concurrencia
