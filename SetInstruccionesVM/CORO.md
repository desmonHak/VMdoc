# Instrucciones de Concurrencia Cooperativa (CORO)

VestaVM implementa dos modelos de concurrencia cooperativa:

| Modelo | instrucciones            | descripcion                                       |
| :----: | :----------------------: | :------------------------------------------------ |
| **A**  | YIELD, RESUME, SPAWN     | scheduler-aware: el planificador gestiona el turno |
| **B**  | SWAPCTX                  | stack-switching: fibras sin intervencion del scheduler |

---

## Tabla de instrucciones

| instruccion | opcode0 | opcode1 | modo | tamano   | descripcion                                |
| :---------: | :-----: | :-----: | :--: | :------: | :----------------------------------------- |
| `yield`     |  0x00   |  0xEC   | NONE | 2 bytes  | ceder CPU al planificador (fin de quantum) |
| `resume`    |  0x00   |  0xED   | REG  | 4 bytes  | reactivar proceso en BLOCKED/WAITING       |
| `spawn`     |  0x00   |  0xEE   | REG  | 4 bytes  | crear nuevo proceso hijo                   |
| `swapctx`   |  0x00   |  0xEF   | REG  | 4 bytes  | intercambio de contexto entre fibras       |

Todas son instrucciones extendidas (prefijo `0x00`).

---

## Codificacion binaria

### YIELD (FIXED_2, modo NONE)

```
+--------+--------+
| 0x00   | 0xEC   |
+--------+--------+
  byte0    byte1
```

Sin operandos; solo los dos bytes de opcode.

### RESUME / SPAWN (FIXED_4, modo REG)

```
+--------+--------+----------+----------+
| 0x00   | opcode |  ctrl    |  regs    |
+--------+--------+----------+----------+
  byte0    byte1    byte2      byte3
```

**byte3 (regs):**
```
bits 7-4 = 0       (no usado)
bits 3-0 = reg1    (indice del registro GP con el PID o direccion)
```

### SWAPCTX (FIXED_4, modo REG)

**byte3 (regs):**
```
bits 7-4 = reg2    (nibble alto: registro GP con dir del contexto destino)
bits 3-0 = reg1    (nibble bajo: registro GP con dir del contexto origen)
```

---

## Modelo A: YIELD, RESUME, SPAWN

### YIELD

Decrementa `reductions_remaining` a cero.  El scheduler detecta el fin del
quantum al finalizar el ciclo de ejecucion actual y reencola el proceso al
final de la cola FIFO de procesos listos.

El proceso no se bloquea: sigue en estado READY y sera ejecutado de nuevo
en cuanto le corresponda el turno.

```asm
; patron de bucle cooperativo
bucle:
    ; ... hacer trabajo ...
    yield               ; ceder CPU al final del quantum actual
    jmp bucle
```

Diferencia con `hlt`: YIELD no detiene el proceso; solo renuncia al
quantum actual.

### RESUME rPID

Reactiva un proceso que esta en estado BLOCKED o WAITING.  El registro
`rPID` debe contener el PID codificado del proceso a reactivar:

```
bits 63..32 = scheduler_id
bits 31.. 0 = local_pid
```

El proceso llamante no se bloquea; continua ejecutandose.

```asm
; asumiendo que r10 contiene el PID obtenido anteriormente con spawn
resume r10             ; poner el proceso en estado READY
```

### SPAWN rAddr

Crea un nuevo `ProcessVM` en el mismo scheduler, establece su PC al valor
del registro `rAddr` y lo pone en estado READY.

El PID del proceso hijo se codifica en 64 bits y se devuelve en **R0**:
```
bits 63..32 = scheduler_id
bits 31.. 0 = local_pid
```

Un valor de R0 distinto de cero indica que el spawn fue exitoso.

```asm
mov   r1, @Absolute("all.fn_hijo")
spawn r1               ; r0 = PID codificado del hijo (!=0)
mov   r10, r0          ; guardar PID para un posible resume posterior
```

**Consideraciones:**
- El proceso hijo nace con su propio espacio de memoria vacio.  Si necesita
  acceder al codigo del padre, se debe usar un mecanismo de memoria
  compartida o pasar las direcciones a traves de registros antes del spawn.
- El hijo comienza en el PC indicado por `rAddr`.  Si el PC apunta a una
  zona de memoria no mapeada, la instruccion decodificada sera nula y el
  proceso haltara automaticamente sin abortar la VM.
- El padre no se bloquea: continua ejecutandose en paralelo con el hijo
  dentro del mismo scheduler.

---

## Modelo B: SWAPCTX (fibras)

### SWAPCTX rDst, rSrc

Realiza un intercambio de contexto de baja latencia entre dos fibras sin
intervencion del scheduler.  La operacion es atomica en dos fases:

1. **Guardar** el contexto actual en el buffer VM apuntado por `rSrc`.
2. **Cargar**  el contexto desde el buffer VM apuntado por `rDst`.

El PC guardado apunta a la instruccion siguiente a `swapctx`, de forma
que la fibra reanudada continua desde el punto correcto.

#### Layout del buffer de contexto (152 bytes)

```
offset   0: PC    (uint64, 8 bytes)
offset   8: SP    (uint64, 8 bytes)
offset  16: BP    (uint64, 8 bytes)
offset  24: R0    (uint64, 8 bytes)
offset  32: R1    (uint64, 8 bytes)
...
offset 144: R15   (uint64, 8 bytes)
```

Total: 3*8 + 16*8 = 24 + 128 = 152 bytes.

Cada fibra debe tener reservado al menos 152 bytes en la VM memory para su
buffer de contexto.

```asm
; Ejemplo de inicializacion y primer intercambio entre dos fibras
;
; ctx_A y ctx_B son buffers de 152 bytes en la seccion de datos.
; La fibra A es la principal; la fibra B tiene su PC precargado en el buffer.

; Inicializar PC de la fibra B antes del primer swapctx
mov   r1, @Absolute("all.ctx_B")
mov   r2, @Absolute("all.fibra_b")
; escribir PC de fibra_b en ctx_B[0..7]
movc  [r1], r2               ; guardar PC de fibra B en su buffer

; --- Fibra A en ejecucion ---
; ... hacer trabajo de fibra A ...

; Intercambiar a fibra B (guardar A, cargar B)
mov   r1, @Absolute("all.ctx_B")   ; r1 = dir del contexto destino (B)
mov   r2, @Absolute("all.ctx_A")   ; r2 = dir del contexto origen  (A)
swapctx r1, r2                      ; A -> ctx_A, ctx_B -> B

; Cuando fibra B llame swapctx de vuelta, la ejecucion continua aqui.
```

---

## Relacion entre YIELD y el scheduler

El scheduler de VestaVM es cooperativo y preemptivo a la vez:

- **Preemptivo**: cada proceso tiene un contador de reducciones
  (`reductions_remaining`).  Cuando llega a cero, el scheduler lo reencola
  automaticamente al final de la cola sin necesidad de que el proceso llame
  a YIELD.
- **Cooperativo**: YIELD permite al proceso ceder antes de agotar su
  quantum, util para bucles de E/S o calculos largos que quieren ser
  "buenos ciudadanos" del scheduler.

SPAWN incrementa `alive_count` del scheduler.  Cuando un proceso hijo
termina (HLT), `alive_count` se decrementa.  El scheduler se detiene
cuando `alive_count` llega a cero en todos los schedulers activos.

---

## Ejemplos de uso

Vease [[../../../examples_codes_vm/ejemplo_coro_yield.vel]] para un ejemplo
completo de YIELD en un bucle cooperativo.

Vease [[../../../examples_codes_vm/ejemplo_coro_spawn.vel]] para crear un
proceso hijo con SPAWN y obtener su PID.
