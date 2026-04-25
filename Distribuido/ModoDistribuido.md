D# Modo Distribuido - Arquitectura del DistRuntime

VestaVM permite ejecutar procesos en multiples nodos fisicos y comunicarlos mediante el
protocolo VDP.  El modo distribuido esta implementado en el namespace `distrib` y se
activa de forma progresiva: la VM siempre inicializa un DistRuntime basico (solo IPC
local) y se puede escalar a operaciones de red bajo demanda.

---

## Capas del modo distribuido

```
+-------------------------------------------------------------+
|  Bytecode .vel                                              |
|  rspawn / msgsend / msgrecv / memsync                       |
+-------------------------------------------------------------+
|  exec_instr_* (exec_instruction_distrib.cpp)                |
|  Traduce cada instruccion a una llamada en DistRuntime       |
+-------------------------------------------------------------+
|  DistRuntime (dist_runtime.h / dist_runtime.cpp)            |
|  Router de mensajes, gestor de sesiones, at-least-once      |
+-------------------------------------------------------------+
|  VdpSession (vdp_session.h / vdp_session.cpp)               |
|  Handshake VDP, framing, CRAM, lectura de paquetes           |
+-------------------------------------------------------------+
|  NodeRegistry (node_registry.h / node_registry.cpp)         |
|  Tabla de nodos conocidos, descubrimiento UDP                |
+-------------------------------------------------------------+
|  Mailbox / DirtyTracker                                     |
|  Cola FIFO por proceso / deteccion de paginas sucias         |
+-------------------------------------------------------------+
|  TCPServer / TLS (net/)                                     |
|  Aceptar conexiones entrantes; cifrado OpenSSL 3.x           |
+-------------------------------------------------------------+
```

---

## Ciclo de vida del DistRuntime

### Fase 1 - IPC local (automatica)

El constructor `VM::VM()` inicializa siempre un DistRuntime minimo:

```cpp
distrib::DistRuntimeConfig cfg{};
cfg.local_node_id    = 0;        // sin ID de red
cfg.vdp_listen_port  = 0;        // sin puerto TCP
cfg.enable_discovery = false;    // sin broadcast UDP
std::snprintf(cfg.local_node_name, sizeof(cfg.local_node_name),
              "vm-%llu", (unsigned long long)id_vm);
dist_runtime = std::make_unique<distrib::DistRuntime>(*this, cfg);
```

En este estado:
- `msgsend` hacia PIDs locales (bit 63 = 0) funciona directamente.
- `msgrecv` con bloqueo y re-ejecucion funciona.
- `rspawn` y `memsync` hacia nodos remotos devuelven error (no hay sesiones).

### Fase 2 - Servidor VDP (bajo demanda)

Al llamar `dist_runtime->start()` (por CLI o API C++):

```
DistRuntime::start()
  -> Crea TCPServer escuchando en vdp_listen_port (7789 por defecto)
  -> Lanza hilo retry_loop_ (revisa pending_msgs_ cada 100 ms)
  -> Si enable_discovery: lanza NodeRegistry::start_discovery()
```

El servidor acepta conexiones TCP entrantes, realiza el handshake VDP (y TLS si
corresponde) y registra la nueva sesion via `on_inbound_session_()`.

### Fase 3 - Conexion a nodos remotos

```
dist_runtime->add_node(ip, port, auth, name)
  -> NodeRegistry::add_static()  -- registra la entrada en la tabla
  -> connect_node(node_idx)
       -> new VdpSession(node_idx, local_node_id)
       -> VdpSession::connect_to(ip, port, auth)
           -> socket() + connect() + (TLS handshake) + VDP handshake
       -> sessions_[node_idx] = sess
       -> NodeRegistry::set_state(AUTHENTICATED)
       -> Lanza hilo VdpSession::reader_loop_()
```

Cuando el estado es `AUTHENTICATED`, las instrucciones `rspawn`, `msgsend` (remoto) y
`memsync` pueden usarse con ese `node_idx`.

---

## Clase DistRuntime

```cpp
class DistRuntime {
    runtime::VM            &vm_;         // VM propietaria
    DistRuntimeConfig       config_;     // parametros de arranque
    NodeRegistry            registry_;   // tabla de nodos

    // sesiones activas indexadas por node_idx
    std::unordered_map<uint32_t, VdpSession *>  sessions_;

    // futuros pendientes de RSPAWN_ACK
    std::unordered_map<uint64_t, PendingFuture> pending_futures_;

    // mensajes pendientes de MSGSEND_ACK (at-least-once)
    std::unordered_map<uint32_t, PendingMsg>    pending_msgs_;

    // hilo de reintentos
    std::thread  retry_thread_;   // corre retry_loop_() cada 100 ms

    // servidor de escucha (clase interna)
    TCPServer *vdp_server_;
};
```

### Metodos publicos principales

| Metodo                         | Descripcion                                                   |
| :----------------------------- | :------------------------------------------------------------ |
| `start()`                      | Inicia servidor VDP y descubrimiento dinamico                 |
| `stop()`                       | Cierra el servidor, las sesiones y el hilo de reintentos      |
| `add_node(ip, port, auth, name)` | Agrega nodo estatico y conecta inmediatamente               |
| `connect_node(node_idx)`       | Conecta a un nodo ya registrado si no esta conectado          |
| `rspawn(proc, fn_addr, node_idx)` | Implementa instruccion RSPAWN; retorna GcHandle del future |
| `msgsend(proc, pid, addr, len)` | Implementa instruccion MSGSEND (local o remoto)              |
| `msgrecv(proc, buf, max_len)`  | Implementa instruccion MSGRECV; bloquea si mailbox vacio      |
| `memsync(proc, params_addr)`   | Implementa instruccion MEMSYNC; sincrona o asincrona          |
| `get_or_create_mailbox(proc)`  | Devuelve (o crea) el Mailbox del proceso indicado             |

---

## NodeRegistry - Tabla de nodos

Cada entrada de la tabla es un `NodeInfo`:

```cpp
struct NodeInfo {
    uint32_t       idx;          // indice inmutable en el vector
    uint64_t       node_id;      // identificador de 64 bits del nodo
    char           ip[64];       // IPv4, IPv6 o nombre de host
    uint16_t       port;         // puerto TCP del servidor VDP
    char           name[32];     // nombre legible para logs
    bool           is_static;    // configurado o descubierto dinamicamente
    NodeState      state;        // UNKNOWN / CONNECTING / AUTHENTICATED / DISCONNECTED / BANNED
    NodeAuthConfig auth;         // TLS + token por nodo
    void          *session;      // VdpSession activa (nullptr si desconectado)
};
```

### Estados de un nodo

```
add_static() / add_dynamic()
    |
    v
UNKNOWN ----connect_node()----> CONNECTING
                                    |
                        handshake OK|  handshake FAIL
                                    |       |
                            AUTHENTICATED  DISCONNECTED
                                    |
                        sesion caida|
                                    v
                             DISCONNECTED
                                    |
                3+ auth failures    |
                                    v
                                  BANNED
```

### Descubrimiento dinamico

Si `enable_discovery = true`, `NodeRegistry::start_discovery()` lanza un hilo UDP:

```
Cada 30 segundos:
  -> socket UDP broadcast en VDP_DISCOVER_PORT (7790)
  -> Emitir VdpPayloadNodeDiscover {node_id, vdp_port}
  -> Escuchar respuestas NODE_ANNOUNCE de otros nodos
  -> Si el node_id no esta en la tabla: add_dynamic() + callback on_new_node_()
```

El callback `on_new_node_` llama a `connect_node()` de forma asincrona para
establecer la sesion con el nodo recien descubierto.

---

## VdpSession - Ciclo de vida de una sesion

Una sesion puede ser **saliente** (iniciada por este nodo) o **entrante** (aceptada
por el servidor VDP).

### Sesion saliente

```cpp
// Constructor de sesion saliente
VdpSession(uint32_t node_idx, uint64_t local_id);

// Conexion y handshake
bool connect_to(const char *ip, uint16_t port, const NodeAuthConfig &auth);

// Hilo de lectura (lanzado tras autenticacion exitosa)
void reader_loop_();  // despacha on_msg_cb_ para cada paquete recibido
```

Secuencia interna de `connect_to()`:

```
socket() + connect()
  -> Si use_tls: SSL_connect() con el SSL_CTX configurado
  -> Enviar HELLO {node_id, auth_flags}
  -> Recibir HELLO_ACK {session_id, nonce[32]}
  -> Si use_token: calcular response = SHA256(token_hash_hex + ":" + nonce_hex)
                   Enviar AUTH_TOKEN {response}
  -> Recibir AUTH_OK o AUTH_FAIL
  -> Si AUTH_OK: lanzar reader_loop_() en hilo separado
  -> Retornar true
```

### Sesion entrante

```
TCPServer acepta fd de cliente
  -> DistRuntime crea VdpSession(fd, ssl, local_node_id)
  -> VdpSession::server_handshake(auth)
       -> Recibir HELLO
       -> Generar nonce[32] aleatorio
       -> Enviar HELLO_ACK {session_id, nonce}
       -> Si auth.use_token:
            -> Recibir AUTH_TOKEN
            -> Verificar SHA256(token_hash_hex + ":" + nonce_hex) en tiempo constante
            -> Enviar AUTH_OK o AUTH_FAIL
       -> Si no token: enviar AUTH_OK directamente
  -> on_inbound_session_(sess)
       -> Registrar sesion en sessions_[node_idx]
       -> Lanzar reader_loop_()
```

---

## Flujo completo de una operacion distribuida

### RSPAWN

```
Proceso local ejecuta: rspawn r_fn, r_node

exec_instr_rspawn()
  -> dist_runtime->rspawn(proc, fn_addr, node_idx)
       1. Leer hasta 64 KB de bytecode desde vm_mem[fn_addr]
       2. Copiar R1..R12, R15 del proceso al payload
       3. Crear FutureObject en estado PENDING en el GC del proceso
       4. Registrar PendingFuture{gc_handle, requester_pid} en pending_futures_
       5. Enviar VDP_RSPAWN al nodo via sessions_[node_idx]
       6. Retornar gc_handle en R0

-- En el nodo remoto --
on_vdp_msg_() -> handle_rspawn_()
  -> Crear nuevo proceso con el bytecode recibido
  -> Restaurar R1..R12, R15 en el nuevo proceso
  -> vm.make_ready(nuevo_pid)
  -> Enviar VDP_RSPAWN_ACK {future_id, remote_pid}

-- De vuelta en el nodo local --
on_vdp_msg_() -> handle_rspawn_ack_()
  -> Buscar PendingFuture por future_id
  -> Resolver el FutureObject con el remote_pid
  -> make_ready(requester_pid) si estaba bloqueado en AWAIT
```

### MSGSEND remoto

```
Proceso local ejecuta: msgsend r_pid, r_addr, r_len  (bit63 del PID = 1)

exec_instr_msgsend()
  -> dist_runtime->msgsend(proc, remote_pid, addr, len)
       1. Extraer node_idx = (pid >> 32) & 0x7FFFFFFF
       2. Extraer local_pid = pid & 0xFFFFFFFF
       3. Obtener sessions_[node_idx]
       4. seq = sess->next_seq()
       5. Construir VdpPayloadMsgsend + datos
       6. Guardar PendingMsg{seq, node_idx, payload, sent_at, retries=0}
       7. sess->send_raw(MSGSEND, pkt, seq)
       8. Retornar true -> R0 = 1

-- retry_loop_ (cada 100 ms) --
  Para cada PendingMsg:
    Si (now - sent_at) > 2000 ms:
      Si retries < 5: reenviar, ++retries, actualizar sent_at
      Si retries >= 5: eliminar, registrar error

-- En el nodo remoto --
on_vdp_msg_() -> handle_msgsend_()
  -> Encontrar proceso por local_pid en los schedulers del VM remoto
  -> get_or_create_mailbox(dest_proc)->push(sender_pid, data, len)
  -> Si dest_proc->state == WAIT_IO: vm.make_ready(dest_pid)
  -> Enviar VDP_MSGSEND_ACK {seq_num}

-- De vuelta en el nodo local --
on_vdp_msg_() -> handle_msgsend_ack_()
  -> pending_msgs_.erase(seq_num)  // entrega confirmada
```

---

## Configuracion de arranque (DistRuntimeConfig)

```cpp
struct DistRuntimeConfig {
    uint64_t local_node_id;       // ID de 64 bits de este nodo (0 = generar automatico)
    char     local_node_name[32]; // nombre para logs ("vm-0", "nodo-europa", etc.)
    uint16_t vdp_listen_port;     // puerto TCP de escucha (0 = usar 7789)
    uint16_t discover_port;       // puerto UDP de descubrimiento (0 = usar 7790)
    bool     enable_discovery;    // true = broadcast UDP de descubrimiento
    NodeAuthConfig server_auth;   // autenticacion del servidor propio
};
```

### Configuracion tipica de produccion

```cpp
distrib::DistRuntimeConfig cfg{};
cfg.local_node_id    = 0;             // se genera automaticamente
cfg.vdp_listen_port  = 7789;
cfg.discover_port    = 7790;
cfg.enable_discovery = true;

// TLS con token CRAM
cfg.server_auth.use_tls   = true;
cfg.server_auth.use_token = true;
std::memcpy(cfg.server_auth.token_hash, sha256_of_secret_token, 32);
std::strncpy(cfg.server_auth.cert_path, "/etc/vesta/server.crt", 256);
std::strncpy(cfg.server_auth.key_path,  "/etc/vesta/server.key", 256);
std::strncpy(cfg.server_auth.ca_path,   "/etc/vesta/ca.crt", 256);

auto dr = std::make_unique<distrib::DistRuntime>(vm, cfg);
dr->start();

// Agregar nodo fijo conocido
distrib::NodeAuthConfig node_auth{};
node_auth.use_tls   = true;
node_auth.use_token = true;
std::memcpy(node_auth.token_hash, sha256_of_secret_token, 32);
dr->add_node("192.168.1.50", 7789, node_auth, "nodo-b");
```

---

## Mailbox y DirtyTracker

### Mailbox

Cada `ProcessVM` que participa en IPC tiene un `Mailbox*` asignado de forma perezosa.
La clase es thread-safe: varios hilos de scheduler pueden hacer `push()` concurrentemente.

```
ProcessVM::mailbox (nullptr por defecto)
  |
  | (primer msgsend o msgrecv)
  v
DistRuntime::get_or_create_mailbox(proc)
  -> Crea Mailbox(max_msgs=1024, max_bytes=64*1024*1024)
  -> proc->mailbox = mb
  -> Retorna mb
```

Ver [[IPC_Mailbox.md]] para el protocolo completo de bloqueo/desbloqueo.

### DirtyTracker

`DirtyTracker` detecta las paginas de 4 KB que cambiaron dentro de una region de
memoria VM, para que `memsync` solo transmita las paginas modificadas.

```
Por region [base, base+len):
  Para cada pagina i:
    1. Adler-32(pagina[i])
       Si igual al baseline -> intacta (sin coste de memoria)
    2. Si Adler-32 cambio:
       BLAKE2s-256(pagina[i])
       Si igual al baseline -> colision Adler-32, la pagina esta intacta
    3. Si BLAKE2s cambio:
       Marcar pagina[i] como sucia
       Actualizar baseline[i] = {adler32, blake2s}
```

El resultado es un vector de indices de paginas sucias.  Solo esos indices se
transmiten en `VDP_MEMSYNC_DATA`.

---

## Activacion desde la CLI

El REPL de VestaVM expone comandos para controlar el modo distribuido:

```
> dist start [--port <P>] [--tls] [--token <tok>]
    Inicia el servidor VDP en el puerto P (default 7789).

> dist add-node <ip> <port> [--tls] [--token <tok>]
    Agrega un nodo estatico y conecta.

> dist list
    Lista todos los nodos del registro con su estado.

> dist send-msg <node_idx> <local_pid> <mensaje>
    Envia un mensaje directo (debug) al proceso indicado.

> dist stop
    Detiene el servidor y cierra todas las sesiones.
```

---

Ver tambien:
- [[VDP_Protocolo.md]] -- formato de paquetes VDP y handshake de sesion
- [[IPC_Mailbox.md]] -- sistema de buzones y comunicacion intra-VM
- [[../SetInstruccionesVM/DISTRIB.md]] -- referencia de instrucciones rspawn/msgsend/msgrecv/memsync
- [[../SetInstruccionesVM/CORO.md]] -- spawn, yield, resume (concurrencia local)
