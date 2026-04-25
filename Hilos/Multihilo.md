# Multihilo en VestaVM

VestaVM soporta dos modelos de concurrencia segun el modo de ejecucion:

---

## Modelo 1: Green threads (hilos verdes, modo interprete)

Cuando la VM ejecuta bytecode en el interprete (sin JIT), usa un modelo de
**green threads**: todos los "hilos" del programa son simulados dentro de un
mismo hilo nativo del sistema operativo.

**Analogia:** imagina a un malabarista que tiene varias pelotas en el aire.
El malabarista (hilo nativo) solo puede sujetar una pelota a la vez, pero las
lanza al aire y las atrapa rapidamente, creando la ilusion de que todas estan
en el aire al mismo tiempo.

### Caracteristicas del modelo green thread

- Los hilos del programa (ProcessVM) son estructuras de datos que el scheduler
  multiplexa sobre los hilos del pool nativo.
- El scheduler asigna cuantos de ejecucion: cada proceso puede ejecutar un numero
  de instrucciones antes de ceder el control.
- Las llamadas externas bloqueantes (syscalls, FFI) se ejecutan en hilos nativos
  reales para no bloquear el scheduler.

### Ventajas

- **Ligero**: crear miles de procesos VM tiene coste minimo (sin overhead del SO).
- **Cooperativo + preemptivo**: los procesos pueden ceder voluntariamente (`yield`)
  o ser interrumpidos por el scheduler al agotar su quantum.
- **Sin carreras de datos**: el modelo cooperativo facilita la sincronizacion.

---

## Modelo 2: JIT con hilos nativos

Cuando se usa el modo JIT (Just-In-Time compilation), el bytecode se compila a
codigo nativo y los procesos pueden ejecutarse en hilos nativos reales del SO,
aprovechando todos los nucleos del procesador.

### Comparativa

| Aspecto                | Green threads (interprete) | JIT con hilos nativos      |
| :--------------------- | :------------------------- | :------------------------- |
| Paralelismo real       | No (un hilo por scheduler) | Si (todos los nucleos)     |
| Overhead por hilo      | Muy bajo                   | Coste del SO               |
| Sincronizacion         | Cooperativa (simple)       | Primitivas del SO (mutexes)|
| Llamadas bloqueantes   | `callthblock` (hilo real)  | Directo                    |
| Preemption             | Quantum de instrucciones   | Preemption del SO          |

---

## Multiples instancias como multihilo

Una alternativa es usar **multiples instancias de VestaVM** para emular multiples
hilos: cada instancia ejecuta un segmento de trabajo independiente y se comunican
via memoria compartida del VmManager o mediante mensajes de red (EDM/EDMW).

---

Ver tambien:
- [[MultithreadGreen.md]] -- instrucciones de hilos verdes (thinit, yield, thgetst...)
- [[../SetInstruccionesVM/CORO.md]] -- corrutinas: yield, resume, spawn, swapctx
- [[../SetInstruccionesVM/MONITOR.md]] -- sincronizacion con monitores
- [[../runtime/VmInstance/VmInstance.md]] -- arquitectura de una instancia VM
