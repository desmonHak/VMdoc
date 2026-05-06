# Concurrencia y programacion asincrona en Vex

Vex tiene soporte de primer nivel para concurrencia: procesos ligeros (green threads),
futuros/promesas, mensajeria entre procesos (IPC) y ejecucion remota en nodos
distribuidos.

Toda esta infraestructura existe en el bytecode de VestaVM. Vex la expone con sintaxis
de alto nivel que baja directamente a las instrucciones correspondientes, sin overhead
adicional.

---

## Spawn: crear un proceso

```java
// Lanzar un proceso con un bloque de codigo (devuelve el PID encoded):
i64 hijo = spawn {
    println("Hola desde el hijo!");
};

// El padre continua ejecutandose concurrentemente con el hijo.
// El bloque hijo termina con hlt (no ret).
```

La sintaxis `spawn { body }` compila el cuerpo como una funcion sintetica `__spawn_N(void)`
y emite la instruccion `spawn r_fn`.

### Placement del spawn

```java
// Auto (default): round-robin entre schedulers disponibles
i64 p1 = spawn { println("auto"); };

// Here: mismo scheduler que el proceso padre (cooperativo, sin overhead cross-thread)
i64 p2 = spawn here { println("mismo scheduler"); };

// On(N): scheduler especifico (N % num_schedulers)
i64 p3 = spawn on(2) { println("scheduler 2"); };
```

| Modo     | Instruccion     | Semantica                                |
| :------- | :-------------- | :--------------------------------------- |
| `spawn`  | `spawn r_fn`    | Round-robin entre schedulers             |
| `spawn here` | `spawnon r_fn, -1` | Mismo scheduler (cooperativo)       |
| `spawn on(N)` | `spawnon r_fn, N` | Pinned al scheduler N % num_scheds  |

### Numero de schedulers

El flag `--schedulers N` controla el paralelismo:

```bash
./vm --run prog.velb --schedulers 1   # cooperativo (1 OS thread)
./vm --run prog.velb --schedulers 4   # 4 OS threads paralelos
./vm --run prog.velb --schedulers 8   # 8 OS threads (recomendado = num CPUs)
```

---

## PID de proceso

```java
i64 mi_pid = pid();    // PID encoded del proceso actual

// Formato: (scheduler_id << 32) | local_pid
i32 local_id    = (i32)(mi_pid & 0xFFFFFFFF);
i32 sched_id    = (i32)((mi_pid >> 32) & 0xFFFFFFFF);
```

---

## IPC: mensajeria entre procesos

```java
// Enviar un entero al proceso con PID destino:
msgsend(destino_pid, valor_i64);

// Recibir el proximo mensaje de la propia bandeja de entrada:
i64 msg = msgrecv();              // bloquea si la bandeja esta vacia

// Patron tipico: hijo envia resultado al padre
i64 padre_pid = pid();

i64 hijo = spawn {
    // El hijo hace un calculo:
    i64 resultado = 40 + 2;
    msgsend(padre_pid, resultado);    // envia al padre
};

i64 resultado = msgrecv();            // el padre espera
println("Resultado del hijo: ${resultado}");  // 42
```

### PID local vs remoto

`msgsend` acepta PIDs locales y remotos:

| PID (bit 63)  | Tipo     | Ruta                         |
| :------------ | :------- | :--------------------------- |
| bit 63 = 0    | Local    | Entrega directa en la misma VM |
| bit 63 = 1    | Remoto   | Envio via protocolo VDP      |

---

## Futuros y await

```java
// Crear un future (promesa):
i64 fut = future_alloc();

// En el productor (otro proceso):
fulfill(fut, 42);       // resuelve el future con el valor 42

// En el consumidor:
i64 resultado = await fut;   // bloquea hasta que se resuelva
println("Resultado: ${resultado}");   // 42
```

### Patron completo productor-consumidor

```java
i64 padre_pid = pid();

i64 fut = future_alloc();
i64 fut_handle = fut;

i64 hijo = spawn {
    // Simular trabajo
    i64 calculo = 100 * 7 / 25 * 3 - 58;    // 42 - 58 + 100 = 84? No: 100*7=700/25=28*3=84-58=26
    // Simplificado: valor fijo para el ejemplo:
    fulfill(fut_handle, 42);
};

// El padre puede hacer otras cosas mientras espera:
println("Esperando resultado del hijo...");
i64 r = await fut;
println("Listo: ${r}");    // 42
```

### Estados del FutureObject

```
future_alloc() -> [PENDING]
                       |
               fulfill(fut, v)     reject(fut, e)
                       |                 |
                 [RESOLVED]         [REJECTED]
```

---

## @Async: funciones asincronas

La anotacion `@Async` convierte una funcion en asincrona: en lugar de ejecutarse en el
proceso del llamante, lanza un proceso hijo y retorna un `Future` que se resolverá cuando
el hijo termine.

```java
@Async
i64 calcular() {
    // este cuerpo se ejecuta en un proceso hijo
    return 42;
}

// El llamante obtiene el future:
i64 fut = calcular();        // tipo de retorno real: i64 (handle del future)
// ... hacer otras cosas mientras el hijo calcula ...
i64 resultado = await fut;   // bloquea hasta que calcular() termine
println("Resultado: ${resultado}");
```

**Limitacion MVP:** la version actual no soporta parametros en funciones `@Async`.
Workaround: usar `msgsend`/`msgrecv` para pasar datos al hijo.

---

## Spawn distribuido (rspawn)

```java
// Ejecutar un bloque en un nodo remoto (devuelve un Future):
i64 fut = rspawn(node_idx) {
    // Este codigo se ejecuta en el nodo 'node_idx'
    i64 resultado = hacer_calculo_remoto();
    return resultado;
};

i64 r = await fut;    // espera el resultado del nodo remoto
```

El `return` dentro del bloque de `rspawn` se intercepta por el compilador y se convierte
en una instruccion `hlt` que el runtime distribuido captura para enviar VDP_FUTURE_FULFILL
al nodo origen.

En modo no-distribuido (sin `--dist-port`), `rspawn` retorna `0xFFFFFFFF` (handle invalido)
pero la estructura del codigo es correcta y lista para distribucion real.

---

## Ejemplo: pipeline de datos paralelo

```java
i32 main(string[] args) {
    i64 padre = pid();

    // Lanzar 4 trabajadores
    i64[4] futures;
    for (i32 i = 0; i < 4; i++) {
        i64 idx = i;
        futures[i] = future_alloc();
        i64 fh = futures[i];
        spawn {
            i64 resultado = procesar_chunk(idx);
            fulfill(fh, resultado);
        };
    }

    // Recolectar resultados
    i64 total = 0;
    for (i32 i = 0; i < 4; i++) {
        total += await futures[i];
    }
    println("Total: ${total}");
    return 0;
}
```

---

## Yield y contextos de fibra

```java
// yield: cede el quantum al scheduler voluntariamente
// (el proceso vuelve a READY y puede ser replanificado)

i64 fibra_a = spawn {
    for (i32 i = 0; i < 10; i++) {
        println("Fibra A: ${i}");
        yield;    // cede al scheduler, permite que fibra_b avance
    }
};

// swapctx: switch cooperativo entre dos fibras SIN pasar por el scheduler
// (ver instruccion swapctx 0xEF en la documentacion de bytecode)
```

---

## Sincronizacion: monitor

El `synchronized` de Vex es el mecanismo de sincronizacion de nivel superior:

```java
class Buffer {
    private i64[] datos = {0, 0, 0, 0, 0, 0, 0, 0};
    private i32 escrituras = 0;

    public void escribir(i64 valor) {
        synchronized (this) {
            datos[escrituras % 8] = valor;
            escrituras += 1;
            notify(this);            // despertar un lector si hay alguno esperando
        }
    }

    public i64 leer() {
        synchronized (this) {
            while (escrituras == 0) {
                wait(this);          // libera el monitor y espera notify
            }
            i32 idx = (escrituras - 1) % 8;
            return datos[idx];
        }
    }
}
```

Los primitivos de monitor disponibles dentro de `synchronized`:

| Builtin         | Instruccion bytecode | Efecto                                        |
| :-------------- | :------------------- | :-------------------------------------------- |
| `wait(obj)`     | `monwait`            | Libera el monitor y suspende hasta notify     |
| `notify(obj)`   | `monnoti`            | Despierta un proceso en espera                |
| `notifyAll(obj)` | `monnota`           | Despierta todos los procesos en espera        |

---

## Tabla de instrucciones de concurrencia

| Instruccion Vex       | Opcode bytecode | Descripcion                               |
| :-------------------- | :-------------- | :---------------------------------------- |
| `spawn { }`           | `0xEE` spawn    | Crear proceso local                       |
| `spawn here { }`      | `0x58` spawnon  | Crear proceso en mismo scheduler          |
| `spawn on(N) { }`     | `0x58` spawnon  | Crear proceso en scheduler N              |
| `pid()`               | `0x57` getpid   | PID encoded del proceso actual            |
| `msgsend(pid, val)`   | `0x3C` msgsend  | Enviar mensaje a proceso                  |
| `msgrecv()`           | `0x3D` msgrecv  | Recibir mensaje de bandeja                |
| `future_alloc()`      | `0x29` future   | Crear FutureObject PENDING                |
| `fulfill(fut, val)`   | `0x2B` fulfill  | Resolver future con valor                 |
| `await fut`           | `0x2A` await    | Esperar hasta que el future se resuelva   |
| `rspawn(N) { }`       | `0x3B` rspawn   | Crear proceso en nodo remoto N            |
| `yield`               | `0xEC` yield    | Ceder quantum al scheduler                |
| `synchronized(obj)`   | `0x35/0x36` monenter/monexit | Seccion critica      |
| `wait(obj)`           | `0x37` monwait  | Liberar monitor y suspender               |
| `notify(obj)`         | `0x38` monnoti  | Despertar un proceso en espera            |
| `notifyAll(obj)`      | `0x39` monnota  | Despertar todos los procesos en espera    |

---

Ver tambien: [[OOP]], [[SetInstruccionesVM/FUTURE_AWAIT]], [[SetInstruccionesVM/CORO]],
[[SetInstruccionesVM/MONITOR]], [[Distribuido/IPC_Mailbox]]
