# VDP (Vesta Distribution Protocol)

VDP es el protocolo de red propietario de VestaVM para la comunicacion distribuida
entre nodos.  Puede operar sobre TCP plano o sobre TLS (recomendado en produccion).

---

## Arquitectura de capas

```
+---------------------------------------------------------+
|  Bytecode VestaVM (.vel)                                |
|  instrucciones: rspawn, msgsend, msgrecv, memsync       |
+---------------------------------------------------------+
|  DistRuntime (C++)                                      |
|  logica de mensajes, futuros, reintentos, node registry |
+---------------------------------------------------------+
|  VdpSession (C++)                                       |
|  handshake VDP, framing de paquetes, CRAM               |
+---------------------------------------------------------+
|  Transporte: TCP plano  o  TLS (OpenSSL 3.x)            |
+---------------------------------------------------------+
|  Red (LAN / Internet)                                   |
+---------------------------------------------------------+
```

---

## Formato del paquete VDP

Cada paquete tiene una cabecera fija de 24 bytes seguida de un payload variable.

### VdpHeader (24 bytes)

```
offset  0 [4 bytes]  magic      = "VDP\0"  (0x56, 0x44, 0x50, 0x00)
offset  4 [1 byte ]  version    = 0x01
offset  5 [1 byte ]  flags      = bitmask de VDP_FLAG_*
offset  6 [2 bytes]  msg_type   = VdpMsgType (big-endian del tipo de mensaje)
offset  8 [8 bytes]  session_id = ID de la sesion activa (asignado por el servidor)
offset 16 [4 bytes]  seq_num    = numero de secuencia para at-least-once (0 = sin ACK)
offset 20 [4 bytes]  payload_len= longitud del payload en bytes
```

### Flags de cabecera

| Flag                | Valor | Descripcion                                            |
| :------------------ | :---: | :----------------------------------------------------- |
| `VDP_FLAG_TLS`      | 0x01  | La sesion esta protegida por TLS                       |
| `VDP_FLAG_AUTH_TOKEN`| 0x02 | Autenticacion por token habilitada                    |
| `VDP_FLAG_AUTH_CERT`| 0x04  | Autenticacion por certificado (mTLS)                   |
| `VDP_FLAG_HMAC`     | 0x08  | El paquete incluye HMAC-SHA256 al final del payload    |
| `VDP_FLAG_COMPRESSED`| 0x10 | Payload comprimido (reservado para v2)                |

### Tabla de tipos de mensaje (VdpMsgType)

| Tipo                | Valor  | Descripcion                                          |
| :------------------ | :----: | :--------------------------------------------------- |
| `HELLO`             | 0x0001 | Presentacion inicial del nodo (handshake inicio)     |
| `HELLO_ACK`         | 0x0002 | Respuesta del servidor con el nonce de desafio       |
| `AUTH_TOKEN`        | 0x0003 | Respuesta al desafio CRAM SHA-256                    |
| `AUTH_OK`           | 0x0004 | Sesion establecida correctamente                     |
| `AUTH_FAIL`         | 0x0005 | Fallo de autenticacion                               |
| `PING`              | 0x0006 | Keepalive                                            |
| `PONG`              | 0x0007 | Respuesta a PING                                     |
| `DISCONNECT`        | 0x00FF | Cierre limpio de la sesion                           |
| `RSPAWN`            | 0x0010 | Crear proceso en nodo remoto (bytecode + args)       |
| `RSPAWN_ACK`        | 0x0011 | PID remoto del proceso creado o codigo de error      |
| `MSGSEND`           | 0x0020 | Enviar mensaje a un proceso                          |
| `MSGSEND_ACK`       | 0x0021 | Confirmacion de entrega (cierra el at-least-once)    |
| `MEMSYNC_HASH`      | 0x0030 | Tabla de hashes por pagina (Adler-32 + BLAKE2s-256)  |
| `MEMSYNC_DIFF`      | 0x0031 | Indices de paginas que difieren en el receptor       |
| `MEMSYNC_DATA`      | 0x0032 | Contenido de las paginas modificadas                 |
| `MEMSYNC_ACK`       | 0x0033 | Confirmacion de sincronizacion aplicada              |
| `FUTURE_FULFILL`    | 0x0040 | Resolver future distribuido con un valor             |
| `FUTURE_REJECT`     | 0x0041 | Rechazar future distribuido con un codigo de error   |
| `NODE_DISCOVER`     | 0x0050 | Broadcast UDP: "busco nodos VDP en la LAN"           |
| `NODE_ANNOUNCE`     | 0x0051 | Respuesta unicast al descubrimiento dinamico         |

---

## Handshake de sesion

El handshake ocurre justo despues de establecer la conexion TCP (y el handshake
TLS si `use_tls = true`).  Consta de 3 a 5 mensajes:

### Flujo sin token (TCP plano o TLS, sin CRAM)

```
Cliente                                    Servidor
-------                                    --------
HELLO {node_id, version, auth_flags=0}
-->
                                           HELLO_ACK {session_id, server_node_id, nonce}
                                           <--
                                           AUTH_OK {session_id, server_caps}
                                           <--
=== sesion ACTIVE ===
```

### Flujo con token CRAM

```
Cliente                                    Servidor
-------                                    --------
HELLO {node_id, auth_flags=VDP_AUTH_TOKEN}
-->
                                           HELLO_ACK {session_id, nonce[32]}
                                           <--
AUTH_TOKEN {response = SHA256(token_hash || ":" || nonce_hex)}
-->
                                           [verifica response]
                                           AUTH_OK   (si correcto)
                                           AUTH_FAIL (si incorrecto)
                                           <--
=== sesion ACTIVE o ERROR ===
```

### Algoritmo CRAM SHA-256

```
nonce_hex   = hex(nonce[32])              // 64 caracteres hex del nonce
response    = SHA256(token_hash_hex + ":" + nonce_hex)

donde:
  token_hash_hex = hex(SHA256(plaintext_token))
  nonce          = 32 bytes aleatorios generados por el servidor
```

La comparacion de `response` se hace en tiempo constante (XOR de 32 bytes)
para evitar ataques de timing.

---

## Puertos por defecto

| Puerto | Protocolo | Uso                                              |
| :----: | :-------: | :----------------------------------------------- |
| 7789   | TCP       | Servidor VDP (acepta conexiones entrantes)       |
| 7790   | UDP       | Descubrimiento dinamico (broadcast/unicast)      |

Ambos puertos son configurables en `DistRuntimeConfig`.

---

## Descubrimiento dinamico (UDP)

Cuando `enable_discovery = true`, cada nodo:

1. Emite un broadcast UDP en `VDP_DISCOVER_PORT` cada 30 segundos con
   un `VdpPayloadNodeDiscover` que incluye su `node_id` y `vdp_port`.
2. Escucha en el mismo puerto y procesa los `NODE_ANNOUNCE` recibidos.
3. Al recibir un anuncio de un nodo desconocido: lo agrega al NodeRegistry
   como nodo dinamico y lanza `connect_node()` de forma asincrona.

El broadcast solo alcanza la LAN local (TTL = 1 en la mayoria de routers).
Para redes WAN se usan nodos estaticos configurados explicitamente.

---

## Garantia de entrega at-least-once (MSGSEND)

Los mensajes `VDP_MSGSEND` se envian con un `seq_num != 0`.  El emisor:

1. Guarda el payload en `pending_msgs_[seq_num]` con la marca de tiempo.
2. El hilo `retry_loop_` revisa la tabla cada 100 ms.
3. Si han pasado 2 segundos sin `VDP_MSGSEND_ACK`: reenviar.
4. Despues de 5 reintentos: eliminar el mensaje y registrar el error.
5. Al recibir `VDP_MSGSEND_ACK` con el mismo `seq_num`: eliminar de la tabla.

Esto garantiza entrega aunque haya perdida de paquetes, pero NO garantiza
exactamente-una-vez (el receptor puede recibir duplicados si el ACK se pierde).

---

## Seguridad

### TLS (capa de transporte)

Si `use_tls = true`, el handshake TLS ocurre antes del handshake VDP:

```
TCP connect
    -> TLS handshake (OpenSSL: TLS_client_method / TLS_server_method)
        -> VDP handshake (HELLO / HELLO_ACK / AUTH_TOKEN / AUTH_OK)
            -> sesion ACTIVE
```

La sesion usa el cifrado negociado por TLS (ej. `TLS_AES_256_GCM_SHA384`).
El servidor carga su certificado y clave privada (PEM) al arrancar.

### Token CRAM (capa de aplicacion)

El token CRAM es una segunda capa de autenticacion que puede usarse sola
(sobre TCP plano) o junto con TLS para doble factor.  El token se almacena
como `SHA256(plaintext_token)` en `NodeAuthConfig.token_hash`; el texto
en claro nunca viaja por la red.

### Mutual TLS (mTLS)

Si `require_client_cert = true`, el servidor exige que el cliente presente
un certificado valido firmado por la CA de confianza configurada en `ca_path`.
Esto permite autenticar nodos sin token (la identidad la da el certificado).

---

## Configuracion de un nodo (NodeAuthConfig)

```cpp
struct NodeAuthConfig {
    bool    use_tls;             // true = usar TLS sobre TCP
    bool    require_client_cert; // true = mutual TLS
    bool    use_token;           // true = CRAM de token
    uint8_t token_hash[32];      // SHA256(plaintext_token)
    char    cert_path[256];      // ruta al certificado PEM del servidor/cliente
    char    key_path[256];       // ruta a la clave privada PEM
    char    ca_path[256];        // ruta al CA bundle (para verificar el servidor)
};
```

---

Ver tambien:
- [[IPC_Mailbox.md]] -- buzon de mensajes y comunicacion intra-VM
- [[ModoDistribuido.md]] -- arquitectura completa del modo distribuido
- [[../SetInstruccionesVM/DISTRIB.md]] -- instrucciones rspawn/msgsend/msgrecv/memsync
