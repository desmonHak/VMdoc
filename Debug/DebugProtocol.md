# Protocolo de Depuracion de VestaVM

Modulo: `include/debug/debugger.h`, `src/debug/debugger.cpp`

## Vision general

VestaVM incluye un servidor de depuracion TCP in-process con un protocolo
propio basado en JSON. El servidor acepta conexiones de clientes externos
(IDEs, depuradores) en el puerto TCP configurado.

Soporta dos modelos de despliegue:

| Modelo         | Descripcion                                              |
| :------------- | :------------------------------------------------------- |
| **In-process** | El servidor corre en un hilo del mismo proceso VM. El bytecode sigue ejecutandose mientras el cliente esta conectado. |
| **Out-of-process** | El cliente externo se conecta al mismo puerto. El servidor es identico. |

---

## Constantes del protocolo

| Constante             | Valor        | Descripcion                        |
| :-------------------- | :----------- | :--------------------------------- |
| `DBG_DEFAULT_PORT`    | `9229`       | Puerto TCP por defecto             |
| `DBG_MAX_MSG_SIZE`    | `4 * 1024 * 1024` | Tamano maximo de mensaje (4 MB)|
| `DBG_HANDSHAKE_MAGIC` | `0x56444247` | Magic "VDBG" en los 4 bytes del handshake |

---

## Protocolo de red

### Framing de mensajes

Cada mensaje tiene la estructura:

```
[uint32_t len LE][payload JSON UTF-8 (len bytes)]
```

- Los primeros 4 bytes codifican la longitud del payload en little-endian.
- El payload es UTF-8 JSON.

### Handshake inicial

Al conectar un cliente, el servidor envia 8 bytes:

```
bytes [0..3] = 0x56444247  (magic VDBG, LE)
bytes [4..7] = 0x00000001  (version del protocolo = 1, LE)
```

---

## Formato de mensajes

### Comando (cliente -> servidor)

```json
{"cmd": "nombre_comando", "seq": 1, ...campos_especificos...}
```

- `cmd`: nombre del comando (string)
- `seq`: numero de secuencia para correlacionar respuesta (uint32)

### Respuesta (servidor -> cliente)

```json
{"ok": true,  "seq": 1, "data": {...}}
{"ok": false, "seq": 1, "error": "mensaje de error"}
```

### Evento assincrono (servidor -> cliente, sin seq)

```json
{"event": "nombre_evento", ...campos...}
```

---

## Comandos soportados

### attach

Adjunta el depurador a un proceso virtual.

```json
{"cmd": "attach", "seq": 1, "pid": 42}
```

Respuesta: `{"ok": true, "seq": 1, "data": {"pid": 42}}`

### detach

Desadjunta el depurador de un proceso. El proceso reanuda ejecucion normal.

```json
{"cmd": "detach", "seq": 2, "pid": 42}
```

### set_break

Establece un breakpoint en una direccion VM.

```json
{"cmd": "set_break", "seq": 3, "addr": 4096, "pid": 42}
```

- `addr`: direccion VM absoluta del breakpoint
- `pid`: 0 = afecta a todos los procesos; != 0 = solo ese proceso

Respuesta: `{"ok": true, "seq": 3, "data": {"id": 1, "addr": 4096}}`

### del_break

Elimina un breakpoint por su id.

```json
{"cmd": "del_break", "seq": 4, "id": 1}
```

### list_breaks

Lista todos los breakpoints activos.

```json
{"cmd": "list_breaks", "seq": 5}
```

Respuesta:

```json
{"ok": true, "seq": 5, "data": [
  {"id": 1, "addr": 4096, "pid": 0, "enabled": true, "hits": 3}
]}
```

### continue

Reanuda la ejecucion de un proceso pausado.

```json
{"cmd": "continue", "seq": 6, "pid": 42}
```

### step

Ejecuta una instruccion y vuelve a pausar (step-into).

```json
{"cmd": "step", "seq": 7, "pid": 42}
```

### next

Step-over: ejecuta hasta la siguiente instruccion al mismo nivel de llamada.

```json
{"cmd": "next", "seq": 8, "pid": 42}
```

### registers

Obtiene el estado de los registros del proceso.

```json
{"cmd": "registers", "seq": 9, "pid": 42}
```

### memory

Lee bytes de la memoria VM del proceso.

```json
{"cmd": "memory", "seq": 10, "pid": 42, "addr": 1000, "length": 32}
```

### stack

Obtiene la traza de pila del proceso.

```json
{"cmd": "stack", "seq": 11, "pid": 42}
```

### info_proc

Informacion general de un proceso virtual.

```json
{"cmd": "info_proc", "seq": 12, "pid": 42}
```

### eval

Evalua una expresion simple (nombre de registro).

```json
{"cmd": "eval", "seq": 13, "pid": 42, "expr": "r0"}
```

### pause

Fuerza la pausa del proceso en la proxima instruccion.

```json
{"cmd": "pause", "seq": 14, "pid": 42}
```

### list_procs

Lista todos los procesos activos en la VM.

```json
{"cmd": "list_procs", "seq": 15}
```

---

## Eventos asincronos

### break

Emitido cuando un proceso alcanza un breakpoint.

```json
{"event": "break", "pid": 42, "pc": 4096, "bp_id": 1}
```

### stepped

Emitido cuando un proceso completa un step.

```json
{"event": "stepped", "pid": 42, "pc": 4100}
```

### exit

Emitido cuando un proceso termina.

```json
{"event": "exit", "pid": 42}
```

### exception

Emitido cuando un proceso lanza una excepcion no capturada.

```json
{"event": "exception", "pid": 42, "class": "NullPointerException"}
```

### spawned

Emitido cuando un proceso crea un proceso hijo.

```json
{"event": "spawned", "parent": 42, "child": 43}
```

---

## Integracion en el codigo de la VM

### Iniciar el servidor

```cpp
#include "debug/debugger.h"
#include "runtime/runtime.h"

runtime::VM vm;
debug::Debugger dbg(vm);
dbg.start(9229);  // inicia en background

// ... ejecutar la VM normalmente ...

dbg.stop();  // detiene el servidor
```

### Intercepcion antes de cada instruccion

En el bucle de ejecucion del scheduler, llamar antes de cada instruccion:

```cpp
// En Scheduler::run_loop() o similar:
if (debugger_) {
    debugger_->on_before_exec(proc);
}
// ejecutar instruccion...
```

`on_before_exec` implementa un fast path con `atomic<bool> any_bp_`:
si no hay breakpoints y el proceso no esta en step_mode, retorna sin adquirir
ningun mutex. El overhead en produccion (sin breakpoints) es una sola carga
atomica con `memory_order_relaxed`.

### Notificaciones de eventos

```cpp
// Cuando un proceso termina:
dbg.on_process_exit(proc->pid.local_pid);

// Cuando un proceso lanza excepcion:
dbg.on_exception(proc->pid.local_pid, "NullPointerException");

// Cuando se crea un proceso hijo:
dbg.on_process_spawn(parent->pid.local_pid, child->pid.local_pid);
```

---

## Compatibilidad de plataformas

| Plataforma  | Sockets         | Abstraccion                      |
| :---------- | :-------------- | :------------------------------- |
| Linux/macOS | BSD sockets     | `SOCK_T = int`, `CLOSE_SOCK = close()` |
| Windows     | Winsock2        | `SOCK_T = SOCKET`, `CLOSE_SOCK = closesocket()` |

En Windows, la clase `WinsockInit` inicializa y finaliza Winsock automaticamente
mediante RAII (`WSAStartup` / `WSACleanup`).
