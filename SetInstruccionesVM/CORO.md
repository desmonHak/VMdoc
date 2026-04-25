# Instrucciones de Concurrencia Cooperativa (CORO)

VestaVM implementa dos modelos de concurrencia cooperativa que permiten que multiples
tareas se ejecuten "al mismo tiempo" dentro de un solo hilo de ejecucion. No son hilos
del sistema operativo, sino mecanismos de control del flujo que permiten que el
programador decida cuando ceder el control a otra tarea.

| Modelo | Instrucciones              | Descripcion                                         |
| :----: | :------------------------: | :-------------------------------------------------- |
| **A**  | `yield`, `resume`, `spawn` | scheduler-aware: el planificador gestiona el turno  |
| **B**  | `swapctx`                  | fibras: intercambio de contexto sin el planificador |

---

## Para que sirve la concurrencia cooperativa

Imagina un servidor que atiende a varios clientes a la vez. Sin concurrencia, el servidor
atenderia a un cliente, luego al siguiente, etc. Con concurrencia cooperativa, cuando el
servidor esta esperando una respuesta del cliente A, puede empezar a atender al cliente B
sin desperdiciar tiempo. Cuando llega la respuesta de A, vuelve a el.

La diferencia con los hilos del SO es que con concurrencia cooperativa el programador
decide explicitamente cuando ceder el control (`yield`), y cuando retomarlo (`resume`).
Esto evita condiciones de carrera y es mucho mas predecible.

---

## Tabla de instrucciones

| Instruccion | opcode0 | opcode1 | Modo | Tamano   | Descripcion                                |
| :---------: | :-----: | :-----: | :--: | :------: | :----------------------------------------- |
| `yield`     |  0x00   |  0xEC   | NONE | 2 bytes  | ceder CPU al planificador (fin de quantum) |
| `resume r`  |  0x00   |  0xED   | REG  | 4 bytes  | reactivar proceso en estado BLOCKED/WAITING|
| `spawn r`   |  0x00   |  0xEE   | REG  | 4 bytes  | crear nuevo proceso hijo                   |
| `swapctx r1,r2` | 0x00| 0xEF   | REG  | 4 bytes  | intercambio de contexto entre fibras       |

Todas son instrucciones extendidas (prefijo `0x00`).

---

## Modelo A: YIELD, RESUME, SPAWN (con el scheduler)

### YIELD - ceder el turno de CPU

```c
yield    // sin operandos: ceder el quantum actual al planificador
```

`YIELD` le dice al planificador: "ya he hecho suficiente trabajo por ahora, dale el turno
a otro proceso". El proceso no termina ni se bloquea; simplemente va al final de la cola
de procesos listos y esperara su proximo turno.

Internamente, `YIELD` pone `reductions_remaining = 0`. El scheduler detecta esto al final
del ciclo actual y reencola el proceso.

```c
// Bucle cooperativo: trabajar un poco y luego ceder el turno
bucle:
    // ... hacer una tarea corta ...
    yield               // ceder CPU; otro proceso puede ejecutarse
    jmp bucle           // cuando nos toque de nuevo, continuar

// DIFERENCIA con HLT:
// HLT -> el proceso termina definitivamente
// YIELD -> el proceso sigue vivo, solo cede el turno temporalmente
```

### SPAWN rAddr - crear un proceso hijo

```c
spawn r1    // R0 = PID del proceso hijo creado (!=0 si exitoso)
```

Crea un nuevo proceso (`ProcessVM`) en el mismo scheduler, establece su PC (program
counter) al valor de `r1` y lo pone en estado READY para ejecutarse.

El PID del proceso hijo se codifica en 64 bits y se devuelve en **R0**:
```
bits 63..32 = scheduler_id  (que scheduler lo gestiona)
bits 31.. 0 = local_pid     (identificador dentro del scheduler)
```

Si `R0 == 0` tras el SPAWN: el spawn fallo (sin recursos).

```c
// Crear un proceso hijo que ejecuta fn_hijo:
mov   r1, @Absolute("code.fn_hijo")  // direccion de inicio del hijo
spawn r1                              // R0 = PID del hijo (si exito, != 0)
mov   r10, r0                         // guardar PID para posible resume

// El padre continua ejecutandose en paralelo con el hijo.
// El hijo comienza en fn_hijo:

fn_hijo:
    // ... codigo del proceso hijo ...
    hlt                 // terminar el proceso hijo
```

Consideraciones:
- El hijo nace con su propio espacio de registros vacio (r0-r15 = 0).
- El hijo tiene su propia pila y GC independientes del padre.
- El hijo y el padre no comparten memoria automaticamente.
- Si el PC del hijo apunta a memoria no mapeada, el proceso haltara sin abortar la VM.

### RESUME rPID - reactivar un proceso bloqueado

```c
resume r10    // reactivar el proceso cuyo PID esta en r10
```

Reactiva un proceso que esta en estado BLOCKED o WAITING. El registro debe contener el
PID codificado en el mismo formato que devuelve `SPAWN` (bits altos = scheduler_id,
bits bajos = local_pid).

El proceso que llama a `RESUME` **no se bloquea**: continua ejecutandose. El proceso
reactivado se pone en la cola de READY y esperara su turno.

```c
// Patron productor-consumidor:
// 1. El padre hace spawn del consumidor (que empieza bloqueado o esperando datos)
mov   r1, @Absolute("code.consumidor")
spawn r1
mov   r10, r0           // guardar PID del consumidor

// 2. El padre produce datos...
// ... producir datos en memoria compartida ...

// 3. El padre avisa al consumidor:
resume r10              // reactivar el consumidor

// El padre sigue ejecutandose; el consumidor procesara los datos cuando le toque
```

---

## Modelo B: SWAPCTX (fibras sin scheduler)

### SWAPCTX rDst, rSrc - intercambio de contexto de fibra

```c
swapctx r1, r2    // guardar contexto en r2, cargar contexto desde r1
```

Realiza un intercambio de contexto de baja latencia entre dos fibras **sin intervencion
del scheduler**. Las fibras son como procesos ultra-ligeros que el programador controla
totalmente.

La operacion es en dos fases atomicas:
1. **Guardar** el contexto actual (PC, SP, BP, R0-R15) en el buffer de 152 bytes en la
   VM memory apuntado por `rSrc`.
2. **Cargar** el contexto desde el buffer de 152 bytes en la VM memory apuntado por `rDst`.

El PC guardado apunta a la instruccion siguiente a `swapctx`, de forma que cuando la
fibra sea reanudada, continue desde el punto correcto.

### Layout del buffer de contexto (152 bytes)

Cada fibra necesita un buffer de 152 bytes en la VM memory para guardar su estado:

```
offset   0 [8 bytes]: PC    (registro de instruccion, donde continuar)
offset   8 [8 bytes]: SP    (puntero de pila)
offset  16 [8 bytes]: BP    (puntero base del frame)
offset  24 [8 bytes]: R0    (registro general 0)
offset  32 [8 bytes]: R1
...
offset 144 [8 bytes]: R15   (registro general 15)
Total: 3*8 + 16*8 = 152 bytes
```

```c
// Ejemplo: dos fibras que se alternan

// Seccion de datos: dos buffers de contexto
all:
    ctx_A: 0x00 x152   // 152 bytes para el contexto de la fibra A (ceros)
    ctx_B: 0x00 x152   // 152 bytes para el contexto de la fibra B

code:
    mov  rsp, 0x00FF0000
    mov  rbp, 0x00FF0000

    // Inicializar el PC de la fibra B (donde empieza cuando se activa):
    mov   r1, @Absolute("all.ctx_B")   // r1 = buffer de contexto de B
    mov   r2, @Absolute("code.fibra_b") // r2 = PC inicial de B
    // Escribir el PC en ctx_B[offset 0]:
    xchg  cur0, r1
    writecur cur0, r2                  // ctx_B.PC = fibra_b

    // --- Fibra A en ejecucion ---
    mov  r11, 0         // r11 = contador de veces que A ejecuta

bucle_a:
    addu r11, 1         // A incrementa su contador
    // Intercambiar a fibra B (guardar A en ctx_A, cargar B de ctx_B):
    mov   r1, @Absolute("all.ctx_B")   // destino: contexto de B
    mov   r2, @Absolute("all.ctx_A")   // origen: contexto de A
    swapctx r1, r2                     // A -> ctx_A, ctx_B -> B
    // Cuando fibra B llame swapctx de vuelta, la ejecucion continua aqui.

    cmpu  r11, 3
    jmp.jlt bucle_a    // A ejecuta 3 veces
    hlt

// --- Fibra B (se activa cuando A llama swapctx) ---
fibra_b:
    mov  r12, 0         // r12 = contador de veces que B ejecuta

bucle_b:
    addu r12, 1         // B incrementa su contador
    // Volver a fibra A:
    mov   r1, @Absolute("all.ctx_A")   // destino: contexto de A
    mov   r2, @Absolute("all.ctx_B")   // origen: contexto de B
    swapctx r1, r2                     // B -> ctx_B, ctx_A -> A
    jmp bucle_b         // cuando B sea activada de nuevo, continuar aqui
```

### YIELD vs SWAPCTX: diferencias clave

| Aspecto         | `yield`                                    | `swapctx`                           |
| :-------------- | :----------------------------------------- | :---------------------------------- |
| Involucra al scheduler | Si                                  | No                                  |
| A quien cede   | El planificador decide                      | La fibra decide explicitamente       |
| Cuantas fibras | Multiples procesos en la cola de READY      | Solo dos: la que llama y la destino |
| Overhead       | Mayor (planificador tiene que intervenir)   | Menor (solo swap de registros)      |
| Uso tipico     | Concurrencia general entre procesos         | Fibras/coroutinas de alta velocidad |

---

## Codificacion binaria

### YIELD (FIXED_2, modo NONE)

```
+--------+--------+
| 0x00   | 0xEC   |
+--------+--------+
  byte0    byte1
```

### RESUME / SPAWN (FIXED_4, modo REG)

```
+--------+--------+----------+----------+
| 0x00   | opcode |  ctrl    |  regs    |
+--------+--------+----------+----------+
  byte0    byte1    byte2      byte3

byte3:
  bits 7-4 = 0       (no usado)
  bits 3-0 = reg1    (indice del registro con el PID o direccion)
```

### SWAPCTX (FIXED_4, modo REG)

```
byte3:
  bits 7-4 = reg2    (nibble alto: registro con dir del contexto destino)
  bits 3-0 = reg1    (nibble bajo: registro con dir del contexto origen)
```

---

## Relacion entre procesos y el scheduler

El scheduler de VestaVM es cooperativo y preemptivo a la vez:

- **Preemptivo**: cada proceso tiene un contador de reducciones (`reductions_remaining`).
  Cuando llega a cero, el scheduler lo reencola automaticamente sin necesidad de YIELD.
- **Cooperativo**: YIELD permite al proceso ceder antes de agotar su quantum.

`SPAWN` incrementa `alive_count` del scheduler. Cuando un proceso hijo termina (`HLT`),
`alive_count` se decrementa. El scheduler se detiene cuando `alive_count` llega a cero.

---

Ver tambien: [[FUTURE_AWAIT]], [[MONITOR]], [[HLT]]
