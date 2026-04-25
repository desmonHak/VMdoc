# FUTURE, AWAIT, FULFILL y REJECT - Async/Await nativo

VestaVM implementa un modelo de concurrencia basado en **promesas** (futures) que permite
a los procesos esperar resultados de otros procesos de forma cooperativa y sin bloquear
hilos nativos del sistema operativo.

| Instruccion | opcode0 | opcode1 | Modo | Tamano  | Descripcion                              |
| :---------: | :-----: | :-----: | :--: | :-----: | :--------------------------------------- |
| `future`    |  0x00   |  0x29   | NONE | 2 bytes | Crea un FutureObject (estado PENDING)    |
| `await`     |  0x00   |  0x2A   | REG  | 4 bytes | Suspende hasta que el future se resuelva |
| `fulfill`   |  0x00   |  0x2B   | REG  | 4 bytes | Resuelve el future con un valor          |
| `reject`    |  0x00   |  0x2C   | REG  | 4 bytes | Rechaza el future con un error           |

Implementacion: `src/runtime/exec_instruction_async.cpp`

---

## Para que sirve async/await

Imagina un programa que necesita hacer varias tareas que tardan: descargar un archivo,
consultar una base de datos, calcular algo complejo. Sin async/await, tendria que esperar
que termine cada tarea antes de empezar la siguiente. Con async/await:

1. Lanzas la tarea (`spawn` + un lugar donde guardar el resultado: `future`).
2. El proceso que necesita el resultado hace `await` y se suspende.
3. El proceso que hace la tarea llama `fulfill` cuando termina.
4. El proceso suspendido se despierta automaticamente con el resultado.

Analogia: es como pedir pizza por telefono (`future`), hacer otras cosas mientras esperas
(`yield` / trabajar), y cuando llama el repartidor (`fulfill`), abrir la puerta (`await`
recibe el resultado).

---

## Estructura FutureObject

```cpp
// Estado posible de un future:
enum class FutureState : uint8_t {
    PENDING  = 0,   // aun no resuelto (nadie ha llamado fulfill/reject)
    RESOLVED = 1,   // resuelto con exito (fulfill fue llamado)
    REJECTED = 2,   // rechazado con error (reject fue llamado)
};

// El objeto que representa la promesa:
struct alignas(8) FutureObject {
    ObjectHeader header;    // gestionado por GC (OBJ_FLAG_GC_OWNED)
    FutureState  state;     // estado actual del future
    uint64_t     result;    // valor de resolucion o codigo de error
    uint64_t     waiter_pid;// PID codificado del proceso en AWAIT (0=nadie espera)
};
```

El `FutureObject` reside en el heap GC y se identifica mediante un `GcHandle`.
El campo `waiter_pid` codifica el PID del proceso bloqueado en `AWAIT`:

```
bits 63..32 = scheduler_id
bits 31.. 0 = local_pid
```

---

## Instrucciones en detalle

### `future` - crear un nuevo future

```c
future    // R0 = GcHandle del nuevo FutureObject (estado PENDING)
```

Sin operandos. Aloca un `FutureObject` en el GC heap, lo inicializa en estado `PENDING`
y devuelve el `GcHandle` en R0. Si no hay memoria disponible devuelve `GC_NULL_HANDLE`.

El `GcHandle` obtenido es el "billete" que tanto el consumidor (el que llama `await`)
como el productor (el que llama `fulfill`) deben compartir.

### `await r_fut` - esperar el resultado

```c
await r1    // bloquea hasta que el future en r1 este resuelto; R0 = resultado
```

**Primera ejecucion (estado PENDING):**
1. Lee el `GcHandle` de `r_fut` y desreferencia al `FutureObject`.
2. Registra el PID del proceso actual en `waiter_pid`.
3. Activa `blocking = true` -> el proceso pasa a estado `WAIT_IO`.
4. El PC **no avanza**: `AWAIT` se re-ejecutara cuando el proceso sea despertado.
5. El scheduler puede asignar este hilo a otro proceso mientras el proceso actual espera.

**Re-ejecucion (cuando `fulfill` o `reject` despiertan al proceso):**
1. Lee `fut->result`.
2. Escribe el resultado en R0.
3. Retorna normalmente: la ejecucion continua con la instruccion siguiente al `await`.

### `fulfill r_fut, r_val` - resolver con exito

```c
fulfill r1, r2    // resuelve el future en r1 con el valor de r2
```

1. Escribe `state = RESOLVED` y `result = regs[r_val]` en el `FutureObject`.
2. Si `waiter_pid != 0`, llama a `make_ready(target)` para despertar al proceso en await.
3. Limpia `waiter_pid = 0` para evitar despertar multiples veces.

El proceso que llama `fulfill` **no se bloquea**; continua ejecutandose.

### `reject r_fut, r_err` - rechazar con error

```c
reject r1, r3    // rechaza el future en r1 con el codigo de error de r3
```

Identico a `FULFILL` pero escribe `state = REJECTED`. El proceso en `AWAIT` recibira el
codigo de error como resultado en R0. Se necesita un protocolo adicional para distinguir
entre un resultado exitoso y un error (ver ejemplo mas abajo).

---

## Diagrama de estados del FutureObject

```
     future (crear)
          |
          v
       [PENDING]
      /          \
 fulfill         reject
    |               |
    v               v
[RESOLVED]    [REJECTED]
```

Una vez resuelto o rechazado, el estado es final: no puede volver a PENDING.

---

## Patron tipico: productor-consumidor

```c
// Seccion de datos (memoria compartida):
all:
    fut_handle: 0x00 0x00 0x00 0x00   // 4 bytes para guardar el GcHandle

code:
    mov   rsp, 0x00FF0000
    mov   rbp, 0x00FF0000

    // 1. Crear el future (antes de lanzar al productor):
    future                          // R0 = GcHandle del future
    mov   r11, r0                   // guardar handle en r11

    // Guardar el handle en memoria para que el hijo lo lea:
    mov   r5, @Absolute("all.fut_handle")
    xchg  cur0, r5
    writecur cur0, r11d             // guardar los 32 bits bajos del handle

    // 2. Lanzar el proceso productor:
    mov   r1, @Absolute("code.productor")
    spawn r1                        // el hijo se ejecuta concurrentemente

    // 3. Esperar el resultado del productor:
    mov   r1, r11
    await r1                        // R0 = valor producido (bloquea hasta FULFILL)
    // El proceso padre estaba en WAIT_IO; ahora se ha despertado con el resultado.

    // R0 = 42 (el valor que produjo el hijo)
    hlt

// Proceso productor (el hijo):
productor:
    // ... realizar trabajo ...
    // Leer el handle del future desde memoria compartida:
    mov   r5, @Absolute("all.fut_handle")
    xchg  cur0, r5
    readcur r1d, cur0               // r1 = GcHandle del future

    mov   r2, 42                    // resultado a enviar al padre
    fulfill r1, r2                  // despertar al proceso padre
    hlt
```

---

## Manejo de errores con REJECT

Como `await` siempre devuelve el valor en R0 (tanto en exito como en error), necesitas
un protocolo para distinguirlos. Una convencion comun es usar el bit mas significativo:

```c
// Productor: reportar un error
mov   r2, 0x8000000000000001   // bit 63 = 1 indica error; bits 0..62 = codigo de error
reject r1, r2

// Consumidor: distinguir exito de error despues del await
await r1                        // R0 = resultado o codigo de error

mov   r2, r0
shr   r2, 63                    // bit 63: 1 = error, 0 = exito
cmpu  r2, 1
jmp.je rechazado                // si bit 63 == 1, fue un reject

// Resultado exitoso en R0 (bit 63 = 0)
jmp fin

rechazado:
    // r0 tiene el codigo de error en los bits 0..62
    // Manejar el error aqui
fin:
    hlt
```

---

## Consideraciones importantes

- Un `FutureObject` puede tener **como maximo un proceso esperando** en `AWAIT`.
  Si un segundo proceso hace `AWAIT` sobre el mismo future PENDING, sobreescribira
  `waiter_pid` y el primer proceso nunca sera despertado.
- `FUTURE` debe ejecutarse **antes** de lanzar al proceso productor con `SPAWN`,
  para que el handle sea valido cuando el productor llame a `FULFILL`.
- El `FutureObject` esta gestionado por el GC. Guarda el `GcHandle` en un registro
  vivo o en memoria VM para que el GC no lo recolecte.
- `FULFILL` y `REJECT` son seguros desde cualquier proceso del mismo scheduler.
  No son thread-safe entre schedulers distintos en la version actual.

---

## Codificacion binaria

### FUTURE (FIXED_2, 2 bytes)

```
+--------+--------+
| 0x00   | 0x29   |
+--------+--------+
  byte0    byte1
```

Sin operandos. El resultado siempre va a R0.

### AWAIT / FULFILL / REJECT (FIXED_4, 4 bytes)

```
+--------+--------+--------+--------+
| 0x00   | opcode | ctrl   | regs   |
+--------+--------+--------+--------+
  byte0    byte1    byte2    byte3

byte3 para await:
  bits 3-0 = r_fut    (registro con el GcHandle del FutureObject)

byte3 para fulfill y reject:
  bits 7-4 = r_fut    (registro con el GcHandle del FutureObject)
  bits 3-0 = r_val    (registro con el valor / codigo de error)
```

---

Ver tambien: [[CORO]], [[SPAWN]], [[GC]], [[MONITOR]]
