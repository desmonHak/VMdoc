# MONENTER, MONEXIT, MONWAIT, MONNOTI, MONNOTA - Monitores de sincronizacion

Los **monitores** de VestaVM son mecanismos de exclusion mutua reentrantes
integrados directamente en cada objeto GC.  Solo un proceso puede poseer el
monitor de un objeto a la vez; el mismo proceso puede adquirirlo varias veces
(lock reentrante).

| Instruccion | opcode0 | opcode1 | Modo | Tamano  | Descripcion breve                                      |
| :---------: | :-----: | :-----: | :--: | :-----: | :----------------------------------------------------- |
| `monenter`  |  0x00   |  0x35   | REG  | 4 bytes | Adquirir monitor; bloquea si lo posee otro proceso     |
| `monexit`   |  0x00   |  0x36   | REG  | 4 bytes | Liberar monitor; despierta al siguiente en la cola     |
| `monwait`   |  0x00   |  0x37   | REG  | 4 bytes | Liberar completamente el monitor y suspender           |
| `monnoti`   |  0x00   |  0x38   | REG  | 4 bytes | Despertar exactamente un proceso en espera condicional |
| `monnota`   |  0x00   |  0x39   | REG  | 4 bytes | Despertar todos los procesos en espera condicional     |

Implementacion: `src/runtime/exec_instruction_sync.cpp`

---

## Concepto accesible: que es un monitor

Imagina que un objeto es una sala de reuniones con una sola llave.  Un proceso
que quiere entrar a la sala toma la llave (MONENTER).  Mientras tenga la llave
nadie mas puede entrar.  Cuando termina, devuelve la llave (MONEXIT) y el
siguiente proceso en la cola puede tomarla.

Si un proceso ya tiene la llave puede entrar a la sala varias veces sin
esperar (reentrada): cada entrada incrementa un contador y cada salida lo
decrementa.  La llave solo vuelve al tablero cuando el contador llega a cero.

MONWAIT es como salir de la sala voluntariamente, devolver la llave y sentarse
en una sala de espera hasta que alguien te avise (MONNOTI o MONNOTA) de que ya
puedes volver.  Al despertar NO tienes la llave automaticamente: debes llamar
MONENTER de nuevo para obtenerla.

---

## Layout de ObjectHeader (ABI v2, 24 bytes)

El monitor vive dentro del ObjectHeader de cada objeto GC:

```
offset  0- 7  class_ptr  (8B) -- puntero a ClassInfo
offset  8-11  flags      (4B) -- OBJ_FLAG_* (bitfield)
offset 12-15  hash_code  (4B) -- identidad del objeto
offset 16-19  owner_pid  (4B) -- local_pid del propietario (0 = libre)
offset 20-21  lock_depth (2B) -- profundidad de lock reentrante
offset 22-23  _mon_pad   (2B) -- relleno de alineacion
```

En C++:

```cpp
struct alignas(8) ObjectHeader {
    ClassInfo *class_ptr;  // 8 bytes
    uint32_t   flags;      // 4 bytes
    uint32_t   hash_code;  // 4 bytes
    uint32_t   owner_pid;  // 4 bytes - propietario del monitor (0 = libre)
    uint16_t   lock_depth; // 2 bytes - contador de reentradas
    uint16_t   _mon_pad;   // 2 bytes - alineacion
};
```

**NOTA:** Esta es la ABI v2 (24 bytes).  La ABI v1 tenia solo 16 bytes y no
incluia owner_pid ni lock_depth.  Todo codigo que asignaba 16 bytes para un
ObjectHeader debe actualizarse a 24.

---

## Cola de espera interna

Ademas de los campos en ObjectHeader, el GcHeap mantiene un mapa de colas:

```cpp
std::unordered_map<GcHandle, std::vector<uint64_t>> monitor_waiters_;
```

Cada entrada asocia un GcHandle con la lista de PIDs codificados de los
procesos que esperan en la cola condicional de ese objeto (MONWAIT).

El PID codificado es `(scheduler_id << 32) | local_pid`.

---

## Codificacion binaria (FIXED_4, 4 bytes)

Todas las instrucciones de monitor usan el mismo formato:

```
+--------+--------+--------+--------+
| 0x00   | opcode | 0x00   | r_reg  |
+--------+--------+--------+--------+
  byte0    byte1    byte2    byte3
```

- `opcode` = 0x35 (monenter), 0x36 (monexit), 0x37 (monwait),
             0x38 (monnoti), 0x39 (monnota)
- `r_reg`  = numero de registro que contiene el GcHandle del objeto (0-15)

---

## Semantica detallada

### `monenter r_handle`

```c
monenter r11    // adquirir el monitor del objeto en r11
```

**Algoritmo:**

1. Resolver el GcHandle en `r_handle` a un puntero de ObjectHeader.
2. Si `owner_pid == 0` (libre): asignar `owner_pid = local_pid` del proceso
   actual, poner `lock_depth = 1`.  Continuar ejecucion.
3. Si `owner_pid == local_pid` (reentrada): incrementar `lock_depth`.
   Continuar ejecucion.
4. Si `owner_pid != 0` y `!= local_pid` (bloqueado por otro):
   - Anadir el PID codificado del proceso actual a la cola de espera del
     handle en `monitor_waiters_`.
   - Marcar `blocking = true` en el proceso (para que el scheduler lo retire
     de la cola de listos).
   - El PC **no avanza**: cuando el proceso sea despertado (por MONEXIT del
     propietario), volvera a ejecutar MONENTER y lo adquirira.

**Complejidad:** O(1) para adquirir; O(1) para encolar si hay contension.

### `monexit r_handle`

```c
monexit r11    // liberar el monitor del objeto en r11
```

**Algoritmo:**

1. Resolver el GcHandle.
2. Verificar que `owner_pid == local_pid` (el proceso actual es el
   propietario).  Si no, comportamiento indefinido (error de programacion).
3. Decrementar `lock_depth`.
4. Si `lock_depth > 0`: el proceso aun posee el monitor (reentrada).
   Continuar ejecucion.
5. Si `lock_depth == 0`: el monitor queda libre.
   - Si hay procesos en la cola de espera del monitor (aquellos bloqueados en
     MONENTER): sacar el primero, llamar `make_ready(pid)` para despertarlo.
     El proceso despertado reintentara MONENTER en su proximo quantum.
   - Poner `owner_pid = 0`.

**IMPORTANTE:** MONEXIT solo despierta procesos bloqueados en MONENTER (cola
de exclusion mutua).  Los procesos suspendidos por MONWAIT (cola condicional)
solo se despiertan con MONNOTI o MONNOTA.

### `monwait r_handle`

```c
monwait r11    // liberar el monitor y suspender en la cola condicional
```

**Algoritmo:**

1. Verificar que el proceso actual posee el monitor (`owner_pid == local_pid`).
2. Si hay procesos en la cola de espera del monitor (MONENTER), sacar uno y
   despertarlo (herencia del lock: el proximo en la cola puede entrar).
3. Poner `owner_pid = 0` y `lock_depth = 0` (liberar completamente).
4. Anadir el PID codificado del proceso actual a la cola condicional
   (`monitor_waiters_[handle]`).
5. Marcar `blocking = true` para que el scheduler no lo ejecute.

**Despues de ser despertado** (por MONNOTI o MONNOTA), el proceso continua
desde la instruccion siguiente a MONWAIT.  El monitor **no se readquiere
automaticamente**: el codigo debe llamar MONENTER explicitamente.

### `monnoti r_handle`

```c
monnoti r11    // despertar a un proceso de la cola condicional
```

Saca un proceso de la cola condicional `monitor_waiters_[handle]` y llama
`make_ready(pid)` para que el scheduler lo incluya en la cola de listos.

Si la cola esta vacia: no-op (sin efecto, sin error).

### `monnota r_handle`

```c
monnota r11    // despertar a todos los procesos de la cola condicional
```

Vacia completamente la cola condicional `monitor_waiters_[handle]` y llama
`make_ready(pid)` para cada proceso.

Si la cola esta vacia: no-op.

---

## Patrones de uso

### Patron basico: seccion critica

```c
monenter r_obj          // adquirir el monitor
// --- seccion critica ---
// acceso a datos compartidos sin riesgo de condicion de carrera
// ---
monexit  r_obj          // liberar el monitor
```

### Patron reentrante

El mismo proceso puede llamar MONENTER varias veces sobre el mismo objeto.
Cada llamada incrementa `lock_depth`.  Se necesita el mismo numero de MONEXIT.

```c
monenter r_obj          // lock_depth = 1
monenter r_obj          // lock_depth = 2 (reentrada)

// trabajo con lock_depth = 2
monexit  r_obj          // lock_depth = 1
monexit  r_obj          // lock_depth = 0, monitor libre
```

### Patron productor-consumidor (monwait + monnoti)

```c
// ---- Proceso consumidor ----
monenter r_obj
loop_espera:
    // comprobar condicion (p.ej. buffer no vacio)
    // si no hay datos aun:
    monwait  r_obj          // libera monitor y espera
    // al despertar, el monitor NO esta adquirido
    monenter r_obj          // readquirir
    jmp.jmp loop_espera     // re-chequear la condicion (patron while)
// datos disponibles: procesar
monexit  r_obj

// ---- Proceso productor ----
monenter r_obj
// producir dato (escribir en buffer compartido)
monnoti  r_obj              // despertar a un consumidor
monexit  r_obj
```

**AVISO:** Tras MONWAIT siempre hay que re-chequar la condicion en un bucle
(patron "while, no if") porque es posible recibir despertares espurios en
implementaciones futuras o en escenarios con varios productores y consumidores.

### Difusion con monnota

```c
// Despertar a todos los lectores/consumidores en espera:
monenter r_obj
// actualizar estado global
monnota  r_obj              // despertar a todos
monexit  r_obj
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

| Cola                   | Quien espera              | Quien despierta     |
| :--------------------- | :------------------------ | :------------------ |
| Cola de exclusion (ME) | Procesos bloqueados en MONENTER | MONEXIT (cuando lock_depth -> 0) |
| Cola condicional       | Procesos suspendidos en MONWAIT | MONNOTI / MONNOTA |

Internamente, ambas colas se almacenan en `monitor_waiters_` del GcHeap, pero
se gestionan por separado en la logica de cada instruccion.

---

## Errores comunes y buenas practicas

1. **Llamar MONEXIT sin poseer el monitor**: comportamiento indefinido.
   Siempre garantizar que MONENTER precede a MONEXIT.

2. **No readquirir tras MONWAIT**: el proceso se despierta sin poseer el
   monitor.  El codigo compilado debe generar MONENTER despues de MONWAIT.

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

Resultado esperado: `R0 = 3` al llegar a `hlt`.
