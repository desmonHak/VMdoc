# Instrucciones de Programacion Distribuida

VestaVM implementa cuatro instrucciones de bytecode que conectan el runtime distribuido
(DistRuntime + protocolo VDP) con el codigo .vel del usuario.  Estas instrucciones
permiten crear procesos remotos, intercambiar mensajes entre nodos y sincronizar regiones
de memoria, todo desde el mismo fichero fuente .vel.

| Instruccion | opcode0 | opcode1 | Modo   | Tamano  | Descripcion                            |
| :---------: | :-----: | :-----: | :----: | :-----: | :------------------------------------- |
| `rspawn`    |  0x00   |  0x3B   | REG    | 4 bytes | Crear proceso en nodo remoto           |
| `msgsend`   |  0x00   |  0x3C   | REG    | 4 bytes | Enviar mensaje (local o remoto)        |
| `msgrecv`   |  0x00   |  0x3D   | REG    | 4 bytes | Recibir mensaje del buzon propio       |
| `memsync`   |  0x00   |  0x3E   | REG    | 4 bytes | Sincronizar region de memoria remota   |

Implementacion: `src/runtime/exec_instruction_distrib.cpp`

---

## Prerequisitos

El DistRuntime es inicializado automaticamente cuando se crea una instancia VM
(modo IPC local, sin abrir puerto TCP).  Para operaciones de red hay que llamar
a `dist_runtime->start()` explicitamente (via la API C++ o el comando CLI
`dist start --port <P>`).

```
// Estado del DistRuntime segun la fase:
//
//  VM creada  ---(auto)---> DistRuntime creado (IPC local activo)
//                                    |
//                           dist start --port 7789
//                                    |
//                           Servidor VDP activo (acepta conexiones remotas)
//                                    |
//                           dist add-node <ip> <port>
//                                    |
//                           Nodo conectado y autenticado (operaciones remotas)
```

---

## RSPAWN - Crear proceso en nodo remoto

```
rspawn  r_fn, r_node     ->  R0 = GcHandle del FutureObject
```

**Operandos:**

| Registro  | Contenido                                                     |
| :-------- | :------------------------------------------------------------ |
| `r_fn`    | Direccion virtual del bytecode a ejecutar en el nodo remoto   |
| `r_node`  | Indice del nodo destino en el NodeRegistry (0..VDP_MAX_NODES) |
| `R0`      | GcHandle del FutureObject creado (estado PENDING)             |

Si R0 == `0xFFFFFFFF`: error (nodo no conectado, sesion inactiva, bytecode vacio).

**Que hace internamente:**

1. Lee hasta 64 KB de bytecode desde `r_fn` en la memoria VM del proceso.
2. Copia los argumentos `R1..R12` y `R15` (argc) al payload.
3. Crea un `FutureObject` en estado PENDING en el GC del proceso local.
4. Envia `VDP_RSPAWN` al nodo remoto con el bytecode y los argumentos.
5. Registra el future en la tabla `pending_futures_` para correlacionarlo con el ACK.
6. Cuando llega `VDP_RSPAWN_ACK`, resuelve el future con el PID remoto del proceso.

**Convencion de llamada para el proceso remoto:**

Los registros `R1..R12` del proceso local se copian directamente al paquete RSPAWN
y son restaurados en el proceso remoto antes de comenzar su ejecucion.  `R15`
contiene el numero de argumentos validos (`argc`).

**Limitaciones (v1):**

- El bloque de bytecode enviado es contiguo desde `r_fn` hasta `r_fn + 64 KB`.
- La funcion remota debe ser autocontenida en ese rango (sin referencias externas).
- El proceso remoto tiene su propio espacio de memoria completamente independiente.

**Ejemplo:**

```asm
    // Pasar el valor 42 al proceso remoto
    mov   r15, 1                          // argc = 1
    mov   r1,  42                         // argumento 1

    mov   r2, @Absolute("code.fn_remota") // bytecode a enviar
    mov   r3, 0                           // node_idx = 0 (primer nodo)
    rspawn r2, r3                         // R0 = GcHandle del future

    // Esperar el PID remoto del proceso creado
    mov   r1, r0
    await r1                              // R0 = PID remoto codificado

fn_remota:
    // este codigo se ejecutara en el nodo remoto
    // R1 = 42 (argumento recibido), R15 = 1 (argc)
    add   r0, r1, 100                     // r0 = 142
    hlt
```

**Codificacion binaria (FIXED_4, modo REG):**

```
+--------+--------+----------+----------+
| 0x00   | 0x3B   |  ctrl    |  regs    |
+--------+--------+----------+----------+
  byte0    byte1    byte2      byte3

byte3: bits 7-4 = r_node (nibble alto)
       bits 3-0 = r_fn   (nibble bajo)
```

---

## MSGSEND - Enviar mensaje a un proceso

```
msgsend  r_pid, r_addr, r_len    ->  R0 = 1 si OK, 0 si error
```

**Operandos:**

| Registro | Contenido                                         |
| :------- | :------------------------------------------------ |
| `r_pid`  | PID codificado del proceso destino (local o remoto) |
| `r_addr` | Direccion VM del buffer con los datos a enviar    |
| `r_len`  | Longitud del mensaje en bytes (max 64 MiB)        |
| `R0`     | 1 = enviado/encolado; 0 = error                   |

**Codificacion del PID:**

```
PID local  (bit 63 = 0):
  bits 31-0 = local_pid
  El mensaje va directamente al Mailbox del proceso en la misma VM.

PID remoto (bit 63 = 1):
  bits 62-32 = node_idx   (31 bits: indice del nodo en NodeRegistry)
  bits 31-0  = local_pid  (PID del proceso en el nodo remoto)
  El mensaje se envia como VDP_MSGSEND al nodo remoto.
```

**Formula para construir un PID remoto:**

```asm
mov   r1, 1
shl   r1, r1, 63          // r1 = 0x8000000000000000 (flag remoto)
shl   r3, r3, 32          // r3 = node_idx << 32
or    r5, r1, r3
or    r5, r5, r4          // r5 = PID remoto = (1<<63)|(node<<32)|local_pid
```

**Ruta local (mismo VM):**

1. Lee `r_len` bytes desde `r_addr` en la VM memory del proceso emisor.
2. Encuentra el proceso destino por `local_pid` en los schedulers del VM.
3. Deposita el mensaje en su `Mailbox` (FIFO thread-safe).
4. Si el proceso estaba bloqueado en `MSGRECV` (estado `WAIT_IO`), llama `make_ready()`.

**Ruta remota (nodo externo):**

1. Lee los bytes del mensaje desde la VM memory.
2. Construye el payload `VDP_MSGSEND` con la estructura `VdpPayloadMsgsend`.
3. Envia el paquete al nodo via la sesion VDP activa con numero de secuencia.
4. Registra el mensaje en `pending_msgs_` para reintento at-least-once.
5. Cuando llega `VDP_MSGSEND_ACK`, elimina la entrada de `pending_msgs_`.
6. Si no llega ACK en 2 segundos, el hilo de reintentos lo reenvía (hasta 5 intentos).

**Ejemplo (IPC local):**

```asm
// El padre envia "ping\0" a su hijo
msg: .byte 0x70, 0x69, 0x6E, 0x67, 0x00   // "ping\0"

    // spawn del receptor
    mov   r1, @Absolute("code.receptor")
    spawn r1                               // r0 = PID del hijo
    mov   r10, r0                          // guardar PID

    // enviar el mensaje
    mov   r2, @Absolute("all.msg")
    mov   r3, 5
    msgsend r10, r2, r3                    // R0 = 1 si OK

receptor:
    mov   r5, @Absolute("all.recv_buf")
    mov   r6, 64
    msgrecv r5, r6                         // R0 = 5 (bytes del "ping\0")
    hlt
```

**Codificacion binaria (FIXED_4, modo REG, 3 registros):**

```
+--------+--------+------------------+------------------+
| 0x00   | 0x3C   |  (r_pid<<4)|r_addr | (r_len<<4)   |
+--------+--------+------------------+------------------+
  byte0    byte1    byte2              byte3
```

---

## MSGRECV - Recibir mensaje del buzon propio

```
msgrecv  r_buf, r_max    ->  R0 = bytes copiados (0 si bloqueo)
```

**Operandos:**

| Registro | Contenido                                              |
| :------- | :----------------------------------------------------- |
| `r_buf`  | Direccion VM del buffer destino para el mensaje        |
| `r_max`  | Tamano maximo del buffer en bytes                      |
| `R0`     | Bytes copiados al buffer; 0 si el proceso se bloqueo   |

**Comportamiento:**

- Si el buzon tiene mensajes: extrae el primero (FIFO), lo copia a `r_buf`
  (hasta `r_max` bytes, truncando si el mensaje es mayor), y retorna el tamano.
- Si el buzon esta vacio: bloquea el proceso en estado `WAIT_IO` y retorna 0.
  La instruccion se re-ejecutara cuando llegue un mensaje (el emisor llama
  `make_ready()` al depositar en el Mailbox).

**Garantia de re-ejecucion:**

```
msgsend (emisor)
  --> Mailbox.push()
    --> vm.make_ready(dest_pid)    // despierta el receptor
      --> El receptor re-ejecuta MSGRECV desde el mismo PC
        --> Mailbox ya tiene el mensaje -> retorna bytes > 0
```

**Limites del Mailbox:**

| Parametro    | Valor por defecto |
| :----------- | :---------------- |
| max mensajes | 1024              |
| max bytes    | 64 MiB            |

Si el Mailbox esta lleno `msgsend` retorna 0 y el mensaje se descarta.

**Ejemplo:**

```asm
receptor:
    mov   r5, @Absolute("all.recv_buf")
    mov   r6, 256
    // Si el buzon esta vacio, se bloquea aqui y vuelve cuando llegue un mensaje.
    msgrecv r5, r6                  // R0 = bytes del mensaje recibido
    hlt
```

**Codificacion binaria (FIXED_4, modo REG):**

```
byte3: bits 7-4 = r_max (nibble alto)
       bits 3-0 = r_buf (nibble bajo)
```

---

## MEMSYNC - Sincronizar region de memoria con nodo remoto

```
memsync  r_params    (sin valor de retorno directo)
```

**Operando:**

| Registro   | Contenido                                            |
| :--------- | :--------------------------------------------------- |
| `r_params` | Direccion VM de la estructura `MemsyncParams` (40 B) |

**Estructura MemsyncParams (layout en VM memory, little-endian):**

```
+0  [8 bytes] uint64_t local_addr     dir VM local del inicio de la region
+8  [8 bytes] uint64_t len            tamano de la region en bytes
+16 [4 bytes] uint32_t node_idx       indice del nodo destino en NodeRegistry
+20 [4 bytes] uint32_t _pad           relleno de alineacion (poner a 0)
+24 [8 bytes] uint64_t remote_addr    dir virtual en el nodo remoto
+32 [4 bytes] uint32_t future_handle  GcHandle del future (0 = no esperar)
+36 [4 bytes] uint32_t _pad2          relleno de alineacion (poner a 0)
Total: 40 bytes
```

**Algoritmo de deteccion de cambios (3 niveles):**

```
Por cada pagina de 4 KB en la region [local_addr, local_addr+len):
  1. Calcular Adler-32 de la pagina (~1 ns/pagina intacta)
     Si Adler-32 == baseline.adler32 -> pagina intacta, saltar
  2. Calcular BLAKE2s-256 de la pagina (solo si Adler-32 cambio)
     Si BLAKE2s == baseline.blake2s  -> colision de Adler-32, saltar
  3. Marcar la pagina como "sucia" (cambio confirmado)

Transmitir solo las paginas sucias via VDP_MEMSYNC_DATA al nodo remoto.
Actualizar el baseline local tras confirmacion VDP_MEMSYNC_ACK.
```

La eficiencia es O(region_size) en el peor caso pero O(paginas_sucias) en la practica:
una region de 1 MB con 1 pagina modificada solo transmite 4 KB.

**Protocolo de sincronizacion (4 mensajes VDP):**

```
Local (emisor)                    Remoto (receptor)
--------------                    -----------------
[detectar paginas sucias]
--> VDP_MEMSYNC_HASH (hashes Adler+BLAKE2s de todas las paginas)
                                  [comparar con sus propios hashes]
                                  <-- VDP_MEMSYNC_DIFF (indices de paginas distintas)
[preparar solo esas paginas]
--> VDP_MEMSYNC_DATA (contenido de las paginas distintas)
                                  [aplicar los datos al nodo remoto]
                                  <-- VDP_MEMSYNC_ACK
[actualizar baseline local]
[resolver future si future_handle != 0]
```

**Uso con future (asincrono):**

```asm
    // Crear el future para conocer cuando termine la sincronizacion
    future                                  // R0 = GcHandle
    mov   r11, r0

    // Rellenar MemsyncParams en memoria
    mov   r13, @Absolute("all.sync_params")
    mov   r14, @Absolute("all.datos")       // local_addr
    movc  [r13], r14
    mov   r14, 4096                         // len = 1 pagina
    mov   r15, r13
    add   r15, r15, 8
    movc  [r15], r14
    // ... rellenar node_idx, remote_addr, future_handle ...
    mov   r15, r13
    add   r15, r15, 32
    movc  [r15], r11                        // future_handle = r11

    memsync r13                             // lanzar sincronizacion

    mov   r1, r11
    await r1                                // R0 = 1 cuando termine
```

**Codificacion binaria (FIXED_4, modo REG):**

```
byte3: bits 7-4 = 0 (no usado)
       bits 3-0 = r_params
```

---

## Tabla resumen de codificaciones

| Instruccion | byte0 | byte1 | byte2                | byte3                      |
| :---------- | :---: | :---: | :------------------- | :------------------------- |
| `rspawn`    | 0x00  | 0x3B  | ctrl (modo<<6)       | (r_node<<4) \| r_fn        |
| `msgsend`   | 0x00  | 0x3C  | (r_pid<<4) \| r_addr | (r_len<<4)                 |
| `msgrecv`   | 0x00  | 0x3D  | ctrl (modo<<6)       | (r_max<<4) \| r_buf        |
| `memsync`   | 0x00  | 0x3E  | ctrl (modo<<6)       | r_params (nibble bajo)     |

---

Ver tambien:
- [[../Distribuido/VDP_Protocolo.md]] -- protocolo VDP sobre TCP/TLS
- [[../Distribuido/IPC_Mailbox.md]] -- buzon de mensajes y IPC interno
- [[../Distribuido/ModoDistribuido.md]] -- arquitectura del modo distribuido
- [[CORO.md]] -- instrucciones de concurrencia local (spawn, yield, resume)
- [[FUTURE_AWAIT.md]] -- futuros y async/await
