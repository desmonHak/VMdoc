# IPC y Mailbox - Comunicacion entre Procesos

VestaVM implementa dos mecanismos de comunicacion entre procesos (IPC):

| Mecanismo     | Alcance            | Instrucciones         | Garantia       |
| :------------ | :----------------- | :-------------------- | :------------- |
| **Mailbox**   | Mismo VM (local)   | `msgsend`, `msgrecv`  | Cola FIFO      |
| **VDP_MSGSEND**| Nodo remoto       | `msgsend` (PID bit63) | At-least-once  |

---

## Mailbox: el buzon de mensajes

Cada proceso que recibe mensajes tiene un `Mailbox` asociado.  Es una cola FIFO
thread-safe que puede recibir mensajes de cualquier hilo del sistema.

### Ciclo de vida del Mailbox

```
ProcessVM creado
  |
  | (primer msgsend o msgrecv)
  v
DistRuntime::get_or_create_mailbox(proc)
  --> Si proc->mailbox == nullptr: crear nuevo Mailbox y asignarlo
  --> Retornar el Mailbox existente o el recien creado
```

El Mailbox se crea de forma perezosa (lazy): solo existe si el proceso
alguna vez envio o recibio mensajes.  Esto evita overhead en procesos que
no usan IPC.

### Estructura de un mensaje

```cpp
struct MailboxMsg {
    uint64_t             sender_pid; // PID codificado del emisor
    std::vector<uint8_t> data;       // copia de los bytes del mensaje
};
```

Los bytes se copian al encolar el mensaje: el emisor puede liberar su
buffer inmediatamente despues de `msgsend`.

### Limites del Mailbox

| Parametro    | Valor por defecto | Descripcion                        |
| :----------- | :---------------- | :--------------------------------- |
| `max_msgs`   | 1024              | Maximo de mensajes encolados       |
| `max_bytes`  | 64 MiB            | Maximo de bytes totales encolados  |

Si se supera alguno de los limites, `push()` retorna `false` y el mensaje
se descarta silenciosamente.  El emisor puede detectarlo porque `msgsend`
retorna 0 en R0.

---

## Flujo completo de un msgsend local

```
Proceso A (emisor)                Proceso B (receptor, bloqueado en msgrecv)
------------------                -----------------------------------------

msgsend r_pid, r_buf, r_len
  |
  | DistRuntime::msgsend()
  |   1. Leer r_len bytes de vm_mem[r_buf]
  |   2. Resolver dest_pid -> ProcessVM *dest
  |   3. mb = get_or_create_mailbox(dest)
  |   4. mb->push(sender_pid, data, len)
  |   5. Si dest->state == WAIT_IO:
  |        vm_.make_ready(dest_pid)   <-- DESPIERTA a B
  |
  v
R0 = 1 (exito)

                                  [make_ready desprograma B de WAIT_IO]
                                  [B vuelve al estado READY]
                                  [scheduler le asigna quantum a B]

                                  msgrecv r_buf, r_max
                                    |
                                    | DistRuntime::msgrecv()
                                    |   1. mb = get_mailbox(this_proc)
                                    |   2. mb->try_pop() -> MailboxMsg
                                    |   3. Copiar data a vm_mem[r_buf]
                                    |   4. Retornar len en R0
                                    v
                                  R0 = bytes_recibidos
```

### Detalle del bloqueo en msgrecv

Cuando el Mailbox esta vacio en el momento de `msgrecv`:

```
1. exec_instr_msgrecv detecta mb->empty() == true
2. Marca decoded_ptr->flags_info.blocking = true
   --> El scheduler NO avanzara el PC al retornar
   --> MSGRECV se re-ejecutara en el proximo quantum de este proceso
3. on_event(EVT_IO_WAIT) -> estado WAIT_IO
4. El proceso sale de la cola READY hasta que llegue un mensaje
```

Cuando el emisor deposita el mensaje:
```
msgsend -> mb->push() -> if (dest->state == WAIT_IO) vm_.make_ready(pid)
  --> B vuelve a READY
  --> En el proximo quantum B ejecuta MSGRECV
  --> Ahora mb->empty() == false -> pop y retornar bytes
```

---

## Flujo de un msgsend remoto

```
Nodo A (emisor)                           Nodo B (receptor)
---------------                           -----------------

msgsend r_remote_pid, r_buf, r_len
  |
  | PID bit63 = 1 -> ruta remota
  | DistRuntime::msgsend()
  |   1. Leer bytes del mensaje
  |   2. node_idx = (pid >> 32) & 0x7FFFFFFF
  |   3. local_pid = pid & 0xFFFFFFFF
  |   4. Obtener VdpSession del nodo
  |   5. seq = sess->next_seq()
  |   6. Construir VdpPayloadMsgsend + datos
  |   7. sess->send_raw(MSGSEND, pkt, seq)
  |   8. pending_msgs_[seq] = { pkt, timestamp, retries=0 }
  v
R0 = 1 (mensaje encolado)

--- VDP_MSGSEND en vuelo ---
                                          VdpSession::reader_loop_()
                                          -> DistRuntime::on_vdp_msg_()
                                          -> handle_msgsend_()
                                             1. Extraer target_pid del payload
                                             2. Encontrar proceso por pid
                                             3. mb->push(sender_pid, data, len)
                                             4. Si WAIT_IO: make_ready()
                                             5. Enviar VDP_MSGSEND_ACK(seq_num)

--- VDP_MSGSEND_ACK en vuelo ---
Nodo A recibe ACK:
  -> handle_msgsend_ack_()
  -> pending_msgs_.erase(seq)   // entrega confirmada
```

---

## Reintento at-least-once

El hilo `retry_loop_` del DistRuntime corre cada 100 ms:

```
for cada entrada en pending_msgs_:
  si (ahora - sent_at_ms) > RETRY_MS (2 segundos):
    si retries < MAX_RETRIES (5):
      reenviar pkt con el mismo seq_num
      actualizar sent_at_ms
      ++retries
    else:
      eliminar entrada  // maximo de reintentos alcanzado
      registrar error
```

Los duplicados en el receptor son posibles si el ACK se pierde despues de
que el receptor ya proceso el mensaje.  El receptor puede detectar duplicados
comparando el campo `seq_num` de la cabecera VDP.

---

## Descubrimiento de procesos por PID

Para encontrar un proceso local a partir de su PID, `DistRuntime::msgsend`
llama a `VM::get_process(GlobalPID)`:

```cpp
struct GlobalPID {
    uint32_t scheduler_id; // ID del scheduler que gestiona el proceso
    uint32_t local_pid;    // ID local dentro del scheduler
};
```

Cuando `scheduler_id == 0` (es decir, el PID bajo no incluye informacion de
scheduler), `get_process` hace una busqueda lineal en todos los schedulers
del VM hasta encontrar el proceso con ese `local_pid`.

**Codificacion del PID devuelto por SPAWN:**

```
bits 63..32 = scheduler_id  (en que scheduler fue creado el proceso)
bits 31.. 0 = local_pid     (identificador dentro del scheduler)
```

Para IPC local entre procesos de la misma VM, se puede usar directamente
el valor que devolvio `spawn` en R0 como PID de destino para `msgsend`.

---

## Ejemplo: productor-consumidor con IPC local

```asm
// Buffer compartido y estructura de mensaje
datos:   .zero 128
buf_rx:  .zero 128

// ---- Proceso productor (padre) ----
productor:
    mov rsp, 0x00FF0000
    mov rbp, 0x00FF0000

    // Lanzar el consumidor y obtener su PID
    mov   r1, @Absolute("all.consumidor")
    spawn r1
    mov   r10, r0              // r10 = PID del consumidor

    // Preparar el mensaje en 'datos'
    mov   r5, @Absolute("all.datos")
    mov   r1, 0x41             // 'A'
    movc  [r5], r1             // datos[0] = 'A'
    // (rellenar el resto del mensaje...)

    // Enviar el mensaje al consumidor
    mov   r2, r10              // PID destino
    mov   r3, r5               // buffer origen
    mov   r4, 1                // 1 byte
    msgsend r2, r3, r4         // R0 = 1 si OK
    hlt

// ---- Proceso consumidor (hijo) ----
consumidor:
    mov rsp, 0x00FE0000
    mov rbp, 0x00FE0000

    mov   r5, @Absolute("all.buf_rx")
    mov   r6, 128
    msgrecv r5, r6             // bloqueante hasta recibir 'A'

    // R0 = 1 (bytes recibidos)
    // vm_mem[buf_rx] = 'A' (0x41)
    movc  r1, [r5]             // r1 = 0x41
    hlt
```

---

Ver tambien:
- [[VDP_Protocolo.md]] -- protocolo de red para IPC remoto
- [[ModoDistribuido.md]] -- arquitectura del DistRuntime
- [[../SetInstruccionesVM/DISTRIB.md]] -- referencia de instrucciones
- [[../SetInstruccionesVM/CORO.md]] -- spawn, yield, resume
