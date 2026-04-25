# Green Threads - Instrucciones de hilos verdes

Las instrucciones de green thread permiten crear y controlar procesos ligeros
dentro de VestaVM. Cada hilo tiene su propio estado (registros, pila, PC) pero
comparten la memoria de la instancia VM.

**Nota:** para corrutinas de alto nivel (yield, resume, spawn, swapctx), ver
[[../SetInstruccionesVM/CORO.md]]. Esta pagina documenta las instrucciones de bajo
nivel para control de hilos.

---

## Instrucciones de creacion y finalizacion

### `thinit` - Crear un nuevo hilo

Crea un nuevo hilo en la VM. Se configuran los registros antes de la instruccion:

| Registro | Contenido                                                     |
| :------: | :------------------------------------------------------------ |
| r00      | Puntero base de la pila del nuevo hilo                        |
| r01      | Limite de pila (si SP lo supera: THREAD_STACK_OVERFLOW)       |
| r02      | Puntero a la primera instruccion a ejecutar (PC inicial)      |
| r03      | Limite del codigo ejecutable (si PC lo supera: hilo muere)    |

```c
// Crear un hilo que ejecuta la funcion 'mi_funcion'
mov r00, pila_hilo       // puntero base de la pila
mov r01, limite_pila     // limite de la pila (64KB mas abajo)
mov r02, mi_funcion      // primera instruccion
mov r03, fin_mi_funcion  // fin del codigo (limite)
thinit                   // crear el hilo; TID retorna en r15
```

El hilo se marca como `READY` al crearlo. Su TID (Thread ID) se devuelve en `r15`.
Para ejecutarlo, cede el control con `yield r15`.

---

## Instrucciones de consulta y cambio de estado

### `thgetst r` - Obtener estado de un hilo

Lee el estado actual del hilo con TID en el registro `r` y lo devuelve en `r00`.

```c
mov r01, mi_tid     // TID del hilo que quiero consultar
thgetst r01         // r00 = estado del hilo (READY, RUNNING, BLOCKED, DEAD)
```

### `thsetst N, r` - Cambiar estado de un hilo

Cambia el estado del hilo con TID en `r` al estado `N`:

| Alias         | N | Estado    | Descripcion                                       |
| :------------ | : | :-------- | :------------------------------------------------ |
| `th_ready r`  | 0 | READY     | Listo para ejecutarse                             |
| `th_running r`| 1 | RUNNING   | No disponible: solo la VM puede cambiarlo         |
| `th_blocked r`| 2 | BLOCKED   | Bloqueado; otro hilo puede desbloquearlo          |
| `th_dead r`   | 3 | DEAD      | Hilo terminado; no puede reactivarse              |

```c
// Bloquear el hilo con TID en r03
th_blocked r03    // equivale a: thsetst 2, r03

// Marcar el hilo actual como terminado
mov r01, 0        // TID del hilo actual
th_dead r01
```

---

## Instrucciones de control

### `th_err` - Obtener codigo de error del hilo

Devuelve el ultimo codigo de error del hilo en `r00`:

| Codigo                      | Valor | Causa                                               |
| :-------------------------- | :---: | :-------------------------------------------------- |
| `THREAD_NO_ERROR`           | 0     | Sin error                                           |
| `THREAD_UNKNOWN_ERROR`      | 1     | Error no clasificado                                |
| `THREAD_SEGMENTATION_FAULT` | 2     | Acceso a memoria sin permisos                       |
| `THREAD_ILLEGAL_INSTRUCTION`| 3     | Instruccion no reconocida o prohibida               |
| `THREAD_DIVISION_BY_ZERO`   | 4     | Division por cero                                   |
| `THREAD_STACK_OVERFLOW`     | 5     | SP alcanzo el limite de pila (push mas alla)        |
| `THREAD_STACK_UNDERFLOW`    | 6     | SP intento decrementarse por debajo de BP           |
| `THREAD_INVALID_SYSCALL`    | 7     | Llamada al sistema invalida                         |

### `th_sleep` - Dormir el hilo actual

Duerme el hilo actual. `r00` debe contener la hora (en nanosegundos) a la que
despertar. El hilo se marca como `BLOCKED` y se reactiva automaticamente.

```c
mov r00, 1000000000    // dormir hasta el nanosegundo 1000000000
th_sleep               // bloquear el hilo hasta ese momento
```

### `yield` - Ceder el control

Cede el control al siguiente hilo listo para ejecutarse:

```c
yield         // ceder al siguiente hilo en la cola del scheduler

yield r05     // ceder el control especificamente al hilo con TID en r05
```

### `setts` - Configurar el quantum de ejecucion

Configura cuantas instrucciones puede ejecutar el hilo antes de que el scheduler
lo interrumpa. Un valor de 0 desactiva la preemption para el hilo actual.

```c
setts 1000    // el hilo puede ejecutar 1000 instrucciones antes de ceder
setts r02     // usar valor en r02 (hasta 64 bits)
setts 0       // desactivar el quantum (el hilo cede solo con yield o IO)
```

### `callthblock` - Llamar funcion bloqueante sin detener la VM

Cuando un hilo necesita llamar a una funcion nativa que puede bloquear (p.ej.
`read`, `recv`), usa `callthblock` en lugar de `calln`. Esto crea un hilo nativo
real para la llamada, permitiendo que el scheduler continue ejecutando otros hilos
mientras espera.

```c
mov r15, 1
mov r01, fd
callthblock @Method("libc.so.6:read")   // llamada bloqueante sin bloquear la VM
```

---

Ver tambien:
- [[Multihilo.md]] -- comparativa de modelos de concurrencia
- [[../SetInstruccionesVM/CORO.md]] -- corrutinas: yield/resume/spawn/swapctx
- [[../SetInstruccionesVM/MONITOR.md]] -- exclusion mutua con monitores
- [[../wmint (VM Interuptions).md]] -- interrupciones (Stack Fault, etc.)
