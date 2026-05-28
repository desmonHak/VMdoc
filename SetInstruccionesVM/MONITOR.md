# MONENTER, MONEXIT, MONWAIT, MONNOTI, MONNOTA - Monitores de sincronizacion

Los **monitores** de VestaVM son mecanismos de exclusion mutua reentrantes
integrados directamente en cada objeto GC. Solo un proceso puede poseer el
monitor de un objeto a la vez; el mismo proceso puede adquirirlo varias veces
(lock reentrante).

| Instruccion | opcode0 | opcode1 | Modo | Tamaño | Descripcion breve |
| :---------: | :-----: | :-----: | :--: | :-----: | :----------------------------------------------------- |
| `monenter` | 0x00 | 0x35 | REG | 4 bytes | Adquirir monitor; bloquea si lo posee otro proceso |
| `monexit` | 0x00 | 0x36 | REG | 4 bytes | Liberar monitor; despierta al siguiente en la cola |
| `monwait` | 0x00 | 0x37 | REG | 4 bytes | Liberar completamente el monitor y suspender |
| `monnoti` | 0x00 | 0x38 | REG | 4 bytes | Despertar exactamente un proceso en espera condicional |
| `monnota` | 0x00 | 0x39 | REG | 4 bytes | Despertar todos los procesos en espera condicional |

Implementacion: `src/runtime/exec_instruction_sync.cpp`

---

## Concepto accesible: que es un monitor

Imagina que un objeto es una sala de reuniones con una sola llave. Un proceso
que quiere entrar a la sala toma la llave (MONENTER). Mientras tenga la llave
nadie mas puede entrar. Cuando termina, devuelve la llave (MONEXIT) y el
siguiente proceso en la cola puede tomarla.

Si un proceso ya tiene la llave puede entrar a la sala varias veces sin
esperar (reentrada): cada entrada incrementa un contador y cada salida lo
decrementa. La llave solo vuelve al tablero cuando el contador llega a cero.

MONWAIT es como salir de la sala voluntariamente, devolver la llave y sentarse
en una sala de espera hasta que alguien te avise (MONNOTI o MONNOTA) de que ya
puedes volver. Al despertar NO tienes la llave automaticamente: debes llamar
MONENTER de nuevo para obtenerla.

---

## Layout de ObjectHeader (ABI v3.1, 24 bytes)

El monitor vive dentro del ObjectHeader de cada objeto GC.Los antiguos campos separados `owner_pid` + `lock_depth` +
`_mon_pad` se **fusionaron** en un solo `monitor_word` atomic de 64 bits.

```
offset 0- 7 class_ptr (8B) -- puntero a ClassInfo
offset 8-11 flags (4B) -- OBJ_FLAG_* (bitfield)
offset 12-15 hash_code (4B) -- identidad del objeto
offset 16-23 monitor_word (8B) -- atomic uint64, empacado:
    bits 0-47 : owner_encoded (encoded_pid, 48 bits)
    bits 48-63 : lock_depth (uint16, 16 bits)
```

En C++:

```cpp
struct alignas(8) ObjectHeader {
    ClassInfo *class_ptr; ///< 8 bytes - puntero a ClassInfo
    uint32_t flags; ///< 4 bytes - OBJ_FLAG_*
    uint32_t hash_code; ///< 4 bytes - identidad del objeto
    std::atomic<uint64_t> monitor_word; ///< 8 bytes - monitor packed
};
```

El monitor_word permite operaciones MONENTER/MONEXIT/MONWAIT **lock-free via
CAS atomico** sobre el word completo. ABI-compatible en tamaño y alineacion
(sigue siendo 24 bytes total).

### Helpers `monitor_make` / `monitor_owner` / `monitor_depth`

```cpp
static constexpr uint64_t MONITOR_OWNER_MASK = 0x0000FFFFFFFFFFFFULL;

static constexpr inline uint64_t monitor_make(uint64_t owner_encoded,
uint32_t depth) noexcept {
    return (owner_encoded & MONITOR_OWNER_MASK)
    | (static_cast<uint64_t>(depth & 0xFFFFu) << 48);
}

static constexpr inline uint64_t monitor_owner(uint64_t word) noexcept {
    return word & MONITOR_OWNER_MASK;
}

static constexpr inline uint32_t monitor_depth(uint64_t word) noexcept {
    return static_cast<uint32_t>((word >> 48) & 0xFFFFu);
}
```

### Por que 48-bit owner_encoded (no 32-bit local_pid)

Cambio critico. Antes, el owner se almacenaba como
`uint32_t local_pid`. Pero **`local_pid` solo es unico por scheduler**: dos
workers en distintos schedulers pueden tener ambos `local_pid=1`. Esto
permitia que B comparara `monitor_owner(expected) == local_pid` -> `1 == 1`
TRUE -> creia ser reentrante -> data loss en synchronized cross-scheduler.

**Fix**: el owner ahora es `encoded_pid = (scheduler_id << 32) | local_pid`
(globalmente unico), almacenado en 48 bits. Cubre 65535 schedulers x 4G
procs/scheduler. `lock_depth` cae a 16 bits (max 65535 reentrancias - sobra
para cualquier caso realista).

Las APIs `GcHeap::monitor_try_acquire` y `monitor_release` reciben
`uint64_t owner_encoded` en vez de `uint32_t local_pid`. Los callers
(`exec_instr_monenter`/`exit`/`wait` + `vrt_monitor_enter`/`exit`) pasan
`encode_pid(vm)` directamente.

### Portabilidad a C

Traducir `std::atomic<uint64_t>` a `_Atomic uint64_t` (C11). Mismo ABI binario.
Para JIT y AOT, el offset del monitor_word es siempre 16.

---

## Cola de espera interna

las colas de espera viven en un `WaitTable` **lock-free**
con 4096 buckets per-VM, en vez del antiguo `unordered_map` mutex-protected.

```cpp
class WaitTable {
    struct Bucket {
        std::atomic_flag spinlock;
        std::vector<Entry> entries; // {handle, kind, encoded_pid}
    };
    Bucket buckets_[4096];

    public:
    void push(uint32_t handle, WaitKind kind, uint64_t encoded_pid);
    uint64_t pop_one(uint32_t handle, WaitKind kind); // UINT64_MAX si vacia
    std::vector<uint64_t> pop_all(uint32_t handle, WaitKind kind);
};

enum class WaitKind : uint8_t { MONITOR, CONDVAR };
```

Cada entrada asocia un GcHandle + kind con la lista de PIDs codificados. El
PID codificado es `(scheduler_id << 32) | local_pid`.

**Dispatch shared vs local**: `wait_table_for(h)` consulta el bit 31; si esta
set, usa el `shared_wait_table` per-VM (cross-process); sino el `wait_table_`
per-process. Sin esto un `wait` en el padre y un `notify` en el hijo sobre
el mismo objeto shared no se verian.

**Sentinel "no waiter" = `UINT64_MAX`** (NO 0), porque encoded_pid=0 es VALIDO
(main process: scheduler 0, local 0). Fix .

---

## Codificacion binaria (FIXED_4, 4 bytes)

Todas las instrucciones de monitor usan el mismo formato:

```
+--------+--------+--------+--------+
| 0x00 | opcode | 0x00 | r_reg |
+--------+--------+--------+--------+
    byte0 byte1 byte2 byte3
```

- `opcode` = 0x35 (monenter), 0x36 (monexit), 0x37 (monwait),
 0x38 (monnoti), 0x39 (monnota)
- `r_reg` = numero de registro que contiene el GcHandle del objeto (0-15)

---

## Semantica detallada

### `monenter r_handle`

```c
monenter r11 // adquirir el monitor del objeto en r11
```

**Algoritmo** (lock-free CAS):

1. Resolver el GcHandle en `r_handle` a un puntero de ObjectHeader. Si el
 bit 31 esta set (SHARED_HANDLE_BIT), el dispatch va al `SharedHandleTable`
 global del VM en vez del `ptr_to_handle_` local.
2. Calcular `epid = encode_pid(vm) = (scheduler_id << 32) | local_pid` (48 bits).
3. **Fast path lock-free**: CAS atomico sobre `monitor_word` de 0 -> `(epid, 1)`
 con memory_order_acquire. Si exito (~5 ns): monitor adquirido, continuar.
4. Si CAS fallo, leer `expected.owner = monitor_owner(actual_word)`.
5. Si `expected.owner == (epid & MONITOR_OWNER_MASK)` (reentrada):
 incrementar `lock_depth` via store relaxed. Continuar.
6. Si `expected.owner != epid` (bloqueado por otro proceso):
 - Anadir el PID codificado del proceso actual a la cola de espera via
 `WaitTable::push(handle, WaitKind::MONITOR, encoded_pid)`.
 - Marcar `blocking = true` en el proceso (para que el scheduler lo retire
 de la cola de listos).
 - El PC **no avanza**: cuando el proceso sea despertado (por MONEXIT del
 propietario), volvera a ejecutar MONENTER y lo adquirira.

**Complejidad**: O(1) para adquirir; O(1) para encolar si hay contension.
**Coste**: ~5 ns sin contencion; ~30 ns con contencion (algunas retries CAS).

### `monexit r_handle`

```c
monexit r11 // liberar el monitor del objeto en r11
```

**Algoritmo** (lock-free):

1. Resolver el GcHandle (con dispatch shared vs local segun bit 31).
2. Calcular `epid = encode_pid(vm)`.
3. Cargar `cur = monitor_word` con memory_order_relaxed.
4. Verificar `monitor_owner(cur) == (epid & MONITOR_OWNER_MASK)` (el proceso
 actual es el propietario). Si no, retornar sin tocar (no soy owner).
5. Decrementar `lock_depth`.
6. Si `lock_depth > 0`: store relaxed del nuevo `monitor_make(epid, d-1)`.
7. Si `lock_depth == 0`: el monitor queda libre.
 - Store **release** de 0 -> garantiza visibilidad de los writes del
 critical section al proximo MONENTER (que tiene CAS acquire).
 - Pop UN proceso de la `WaitTable` del handle via
 `WaitTable::pop_one(handle, WaitKind::MONITOR)`.
 - Si hay proceso popped, llamar `make_ready(pid)` para despertarlo.
 El proceso despertado reintentara MONENTER en su proximo quantum.

**IMPORTANTE:** MONEXIT solo despierta procesos bloqueados en MONENTER (cola
de exclusion mutua). Los procesos suspendidos por MONWAIT (cola condicional)
solo se despiertan con MONNOTI o MONNOTA.

### `monwait r_handle`

```c
monwait r11 // liberar el monitor y suspender en la cola condicional
```

**Algoritmo**

1. Verificar que el proceso actual posee el monitor
 (`monitor_owner(monitor_word) == (encode_pid(vm) & MONITOR_OWNER_MASK)`).
2. Si hay procesos en la cola **monitor (MONENTER)** del handle, sacar uno y
 despertarlo (herencia del lock: el proximo en la cola puede entrar).
3. Store release de `monitor_word = 0` (liberar completamente; depth queda 0).
4. Anadir el PID codificado del proceso actual a la **cola condvar** separada
 via `WaitTable::push(handle, WaitKind::CONDVAR, encoded_pid)`.
5. Marcar `blocking = true` para que el scheduler no lo ejecute.

**Nota**: cola monitor y cola condvar son SEPARADAS dentro de la
`WaitTable` (kind=MONITOR vs kind=CONDVAR). Antes compartian la
misma cola y un `notify` podia despertar accidentalmente a un MONENTER bloqueado.

**Despues de ser despertado** (por MONNOTI o MONNOTA), el proceso continua
desde la instruccion siguiente a MONWAIT. El monitor **no se readquiere
automaticamente**: el codigo debe llamar MONENTER explicitamente.

### `monnoti r_handle`

```c
monnoti r11 // despertar a un proceso de la cola condicional
```

Saca un proceso de la cola condicional `monitor_waiters_[handle]` y llama
`make_ready(pid)` para que el scheduler lo incluya en la cola de listos.

Si la cola esta vacia: no-op (sin efecto, sin error).

### `monnota r_handle`

```c
monnota r11 // despertar a todos los procesos de la cola condicional
```

Vacia completamente la cola condicional `monitor_waiters_[handle]` y llama
`make_ready(pid)` para cada proceso.

Si la cola esta vacia: no-op.

---

## Patrones de uso

### Patron basico: seccion critica

```c
monenter r_obj // adquirir el monitor
// --- seccion critica ---
// acceso a datos compartidos sin riesgo de condicion de carrera
// ---
monexit r_obj // liberar el monitor
```

### Patron reentrante

El mismo proceso puede llamar MONENTER varias veces sobre el mismo objeto.
Cada llamada incrementa `lock_depth`. Se necesita el mismo numero de MONEXIT.

```c
monenter r_obj // lock_depth = 1
monenter r_obj // lock_depth = 2 (reentrada)

// trabajo con lock_depth = 2
monexit r_obj // lock_depth = 1
monexit r_obj // lock_depth = 0, monitor libre
```

### Patron productor-consumidor (monwait + monnoti)

```c
// ---- Proceso consumidor ----
monenter r_obj
loop_espera:
// comprobar condicion (p.ej. buffer no vacio)
// si no hay datos aun:
monwait r_obj // libera monitor y espera
// al despertar, el monitor NO esta adquirido
monenter r_obj // readquirir
jmp.jmp loop_espera // re-chequear la condicion (patron while)
// datos disponibles: procesar
monexit r_obj

// ---- Proceso productor ----
monenter r_obj
// producir dato (escribir en buffer compartido)
monnoti r_obj // despertar a un consumidor
monexit r_obj
```

**AVISO:** Tras MONWAIT siempre hay que re-chequar la condicion en un bucle
(patron "while, no if") porque es posible recibir despertares espurios en
implementaciones futuras o en escenarios con varios productores y consumidores.

### Difusion con monnota

```c
// Despertar a todos los lectores/consumidores en espera:
monenter r_obj
// actualizar estado global
monnota r_obj // despertar a todos
monexit r_obj
```

---

## Interaccion con el scheduler

- Cuando MONENTER no puede adquirir el monitor, el proceso queda en estado
 **BLOCKED** (blocking=true) y el scheduler no lo ejecuta.
- Cuando MONEXIT despierta al siguiente proceso de la cola, llama internamente
 a `make_ready(pid)` que transiciona el proceso de BLOCKED a READY y lo
 encola en el scheduler.
- El proceso despertado no adquiere el monitor inmediatamente; lo hara cuando
 su quantum llegue y reeje cute MONENTER con exito.

Esta arquitectura es cooperativa con el scheduler y no requiere mecanismos de
exclusion a nivel de sistema operativo (no usa mutexes de SO).

---

## Diferencia entre cola de exclusion mutua y cola condicional

| Cola | Quien espera | Quien despierta |
| :--------------------- | :------------------------------ | :------------------------------- |
| Cola de exclusion (ME) | Procesos bloqueados en MONENTER | MONEXIT (cuando lock_depth -> 0) |
| Cola condicional | Procesos suspendidos en MONWAIT | MONNOTI / MONNOTA |

ambas colas viven en el mismo `WaitTable` pero usan `WaitKind`
distintos (MONITOR vs CONDVAR) y se filtran por kind en cada operacion.
Antes (pre-) compartian cola y un `notify` podia despertar un MONENTER
bloqueado por error.

Para objetos `shared` el dispatch va al `shared_wait_table` per-VM
(cross-process visible). Para objetos locales, al `wait_table_` per-process.

---

## Errores comunes y buenas practicas

1. **Llamar MONEXIT sin poseer el monitor**: comportamiento indefinido.
 Siempre garantizar que MONENTER precede a MONEXIT.

2. **No readquirir tras MONWAIT**: el proceso se despierta sin poseer el
 monitor. El codigo compilado debe generar MONENTER despues de MONWAIT.

3. **Llamar MONNOTI sin poseer el monitor**: correcto tecnicamente, pero puede
 provocar condiciones de carrera si se hace fuera de una seccion critica.
 Por convension, siempre notificar dentro del monitor.

4. **Deadlock**: dos procesos A y B esperan mutuamente el monitor del otro.
 VestaVM no detecta deadlocks automaticamente.

5. **Olvidar MONEXIT**: el monitor nunca se libera y los procesos en cola
 esperan indefinidamente.

---

## Ejemplo completo

Vease `examples_codes_vm/ejemplo_monitor.vel` para un ejemplo ejecutable que
recorre los tres escenarios principales:

- Escenario 1: lock simple (monenter + monexit).
- Escenario 2: lock reentrante (tres niveles de profundidad).
- Escenario 3: monnoti sobre una cola vacia (no-op seguro).
- Extra: monnota sobre cola vacia.

---

## Monitores en objetos compartidos

Los monitores funcionan identicos sobre objetos `shared` (alocados en
`SharedHeap`) y locales (en `gc_heap` per-process):

```vex
class Counter {
    public i64 value;
    public Counter() { this.value = 0; }
    public void add(i64 d) {
        synchronized(this) {
            this.value = this.value + d;
        }
    }
}

i32 main() {
    shared Counter c = new Counter();
    // 4 workers cross-scheduler mutando bajo synchronized
    spawn { for (i32 i=0;i<100000;i++) c.add(1); };
    spawn { for (i32 i=0;i<100000;i++) c.add(1); };
    spawn { for (i32 i=0;i<100000;i++) c.add(1); };
    spawn { for (i32 i=0;i<100000;i++) c.add(1); };
    // ... join ...
    return (i32) c.value; // exactamente 400000
}
```

El dispatch automatico via bit-31 del GcHandle envia el `monenter`/`monexit`
al `SharedHandleTable` + `shared_wait_table` del VM, garantizando exclusion
mutua cross-scheduler.

**Verificacion**: `bench_shared_contention --schedulers 4` retorna R0 = 400000
exacto (en 3/3 runs). Antes perdia ~1.7% por confusion entre
local_pid no-unicos cross-scheduler.

Ver [[../PhaseZ/SharedMemory.md]] para el modelo completo.

Resultado esperado: `R0 = 3` al llegar a `hlt`.
