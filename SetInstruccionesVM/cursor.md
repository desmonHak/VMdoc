# Instrucciones cursor - acceso a memoria real del host

Las instrucciones cursor permiten leer y escribir directamente en memoria real del proceso
host (no en la memoria virtual de la VM). Son el puente entre el bytecode Vesta y:

- Bloques reservados con `ALLOC` (punteros host reales, para FFI).
- Payloads de objetos gestionados por el GcHeap (accedidos mediante `GCDEREF`).
- Cualquier direccion de host valida almacenada en un registro cursor.

> **Por que hace falta esto?**
> La memoria virtual de la VM se gestiona a traves de una capa TLB/VirtualMemory propia.
> Las instrucciones MOV, ADD, etc. operan sobre registros y memoria virtual de la VM.
> Para acceder a un bloque de RAM del host (por ejemplo para rellenar un buffer antes de
> una llamada nativa), se necesita un mecanismo que desreferencie directamente la
> direccion host. Eso son las instrucciones cursor.

---

## Que son los registros cursor

Los cuatro registros cursor (`cur0`-`cur3`) son registros especiales de 64 bits que
contienen **direcciones del espacio de memoria del proceso host**. Son como punteros en C:
guardan la direccion de un bloque de RAM real.

Analogia: si la VM es una oficina con su propio sistema de archivado (memoria virtual),
los cursores son la "direccion postal" real en el mundo exterior (la RAM del host).

| Registro | Codificacion |
| :------: | :----------: |
| `cur0`   |  `0b000000`  |
| `cur1`   |  `0b000001`  |
| `cur2`   |  `0b000010`  |
| `cur3`   |  `0b000011`  |

> Los registros cursor **solo deben contener direcciones host validas**. Cuando no se usen
> deben valer `0` (equivalente a NULL). Desreferenciar un cursor nulo o invalido produce
> comportamiento indefinido (crash).

---

## Instrucciones

### `readcur` - leer desde memoria host

```c
readcur  dest_reg, curN
```

Lee N bytes de la direccion host almacenada en `curN` y los escribe en `dest_reg`. El
tamano viene determinado por el sufijo del registro destino:
- `b` = 1 byte (`uint8_t`)
- `w` = 2 bytes (`uint16_t`)
- `d` = 4 bytes (`uint32_t`)
- sin sufijo o `q` = 8 bytes (`uint64_t`)

| Opcode1 | Opcode2 | ctrl_byte     | reg_byte    | Total |
| :-----: | :-----: | :------------ | :---------- | :---: |
| `0x00`  | `0xC0`  | `mm cc 0000`  | `0000 rrrr` |  4 B  |

- `mm` = modo de tamano: `00`=byte, `01`=word, `10`=dword, `11`=qword
- `cc` = indice del cursor (0-3), en bits 5-4 del ctrl_byte
- `rrrr` = indice del registro general destino (R0-R15)

```c
// Reservar un bloque de memoria y leer de el:
mov    r1, 16
alloc  r1           // R0 = puntero host real al bloque de 16 bytes

// Mover el puntero al cursor para poder acceder:
mov    r14, r0
xchg   cur0, r14    // cur0 = puntero host; r14 = valor anterior de cur0 (descartado)

readcur r2, cur0    // r2 = *(uint64_t*)cur0  (leer 8 bytes)
readcur r3b, cur0   // r3 = *(uint8_t*)cur0   (leer solo el primer byte)
```

---

### `writecur` - escribir en memoria host

```c
writecur  curN, src_reg
```

Escribe N bytes del registro `src_reg` en la direccion host almacenada en `curN`. El
tamano viene determinado por el sufijo del registro fuente.

| Opcode1 | Opcode2 | ctrl_byte     | reg_byte    | Total |
| :-----: | :-----: | :------------ | :---------- | :---: |
| `0x00`  | `0xC1`  | `mm cc 0000`  | `0000 rrrr` |  4 B  |

```c
mov    r5, 0xFF00FF00
writecur cur0, r5       // *(uint64_t*)cur0 = r5 (8 bytes)
writecur cur1, r6d      // *(uint32_t*)cur1 = r6d (solo 4 bytes)
writecur cur2, r7b      // *(uint8_t*)cur2  = r7b (solo 1 byte)
```

---

### `gcderef` - convertir GcHandle a puntero host en cursor

```c
gcderef  curN, handle_reg
```

Toma el `GcHandle` (indice opaco del GC) contenido en `handle_reg`, lo desreferencia
mediante la `HandleTable` del GC y almacena el puntero real al payload del objeto en
`curN`. Despues de esto, `curN` contiene una direccion host valida para usar con
`readcur`/`writecur`.

| Opcode1 | Opcode2 | ctrl_byte     | reg_byte    | Total |
| :-----: | :-----: | :------------ | :---------- | :---: |
| `0x00`  | `0xC2`  | `00 cc 0000`  | `0000 rrrr` |  4 B  |

> **ADVERTENCIA CRITICA de invalidacion:**
> El puntero obtenido con `gcderef` **puede quedar invalido** tras cualquier ciclo de GC
> (provocado por `GCRUN` o por una asignacion que dispare un minor GC). Si el GC mueve
> el objeto, la direccion en el cursor queda obsoleta. Llama a `gcderef` de nuevo tras
> cada posible ciclo de GC, o en fragmentos cortos sin asignaciones intermedias.

```c
mov    r1, 64
gcalloc r1           // R0 = GcHandle
mov    r8, r0        // guardar el handle

gcderef cur2, r8     // cur2 = puntero real al payload del objeto

mov    r3, 42
writecur cur2, r3    // escribe 42 en el primer campo (8 bytes)

readcur  r4, cur2    // r4 = 42 (leer de vuelta)
```

---

### `addcur` - avanzar o retroceder el cursor

```c
addcur  curN, imm16
```

Suma el inmediato con signo de 16 bits al valor del cursor `curN`. Permite avanzar o
retroceder dentro de un bloque de memoria sin el patron costoso de 3 instrucciones
(`xchg` + `adds` + `xchg`).

| Opcode1 | Opcode2 | ctrl_byte      | pad    | imm_lo | imm_hi | Total |
| :-----: | :-----: | :------------- | :----- | :----- | :----- | :---: |
| `0x00`  | `0xC3`  | `00 cc 0000`   | `0x00` | `lo`   | `hi`   |  6 B  |

- `cc` = indice del cursor (0-3) en bits 5-4 del ctrl_byte
- `imm16` = desplazamiento con signo en little-endian (bytes 4-5)
- Rango: -32768 .. +32767 bytes

Los literales negativos se expresan en complemento a dos: `-8 = 0xFFF8`, `-16 = 0xFFF0`.

```c
// Avanzar cur0 en 8 bytes (para leer el siguiente campo qword):
addcur cur0, 8

// Avanzar al siguiente elemento en una estructura de 24 bytes:
addcur cur1, 24

// Retroceder 8 bytes (para releer el primer campo):
addcur cur0, 0xFFF8    // 0xFFF8 = -8 en int16
```

### Patron antiguo vs. addcur

```c
// Patron antiguo (3 instrucciones para avanzar 8 bytes):
xchg   cur0, r14     // r14 = cur0
adds   r14, 8        // r14 += 8
xchg   cur0, r14     // cur0 = r14 + 8

// Patron nuevo con addcur (1 instruccion):
addcur cur0, 8
```

---

### `vmcopy` - copiar de VM memory a host memory

```c
vmcopy  curN, rSrc, rLen
```

Copia `rLen` bytes desde la VM memory (a partir de la direccion virtual en `rSrc`) al
buffer host apuntado por `curN`. Tras la copia **ambos punteros avanzan automaticamente**
`rLen` bytes, facilitando copias secuenciales sin instrucciones adicionales.

La copia usa la ruta SIMD mas rapida disponible (AVX-512, AVX2, SSE2, o escalar),
detectada una sola vez en tiempo de ejecucion.

| Opcode1 | Opcode2 | byte_A        | byte_B        | Total |
| :-----: | :-----: | :------------ | :------------ | :---: |
| `0x00`  | `0xC4`  | `00 cc rrrr`  | `llll 0000`   |  4 B  |

- `cc` = indice del cursor destino (bits 5-4 del byte_A)
- `rrrr` = registro fuente VM (direccion virtual), bits 3-0 del byte_A
- `llll` = registro de longitud, bits 7-4 del byte_B

```c
// Copiar un string desde VM memory a un buffer host:
mov    r1, @Absolute("all.hello")   // VM address del string
mov    r2, 13                       // longitud (13 bytes)
mov    r3, r2
alloc  r3                           // R0 = host buffer
mov    r10, r0                      // guardar host ptr
xchg   cur0, r0                     // cur0 = host buffer

vmcopy cur0, r1, r2                 // copia 13 bytes de VM -> host
                                    // (cur0 y r1 avanzan 13 bytes cada uno)
```

### `vcopyh` - copiar de host memory a VM memory

```c
vcopyh  rDst, curN, rLen
```

Inverso de `vmcopy`: copia `rLen` bytes desde el buffer host apuntado por `curN` a la
VM memory a partir de la direccion virtual en `rDst`. Tras la copia ambos punteros
avanzan `rLen` bytes.

| Opcode1 | Opcode2 | byte_A        | byte_B        | Total |
| :-----: | :-----: | :------------ | :------------ | :---: |
| `0x00`  | `0xC5`  | `00 cc rrrr`  | `llll 0000`   |  4 B  |

```c
// Recibir resultado de una funcion nativa y guardarlo en VM memory:
mov    r15, 1
calln  @Method("msvcrt.dll:fgets")  // R0 = host ptr al resultado

xchg   cur0, r0                     // cur0 = host ptr con resultado

mov    r5, @Absolute("all.buffer")  // VM address del buffer destino
mov    r6, 256                      // bytes a copiar
vcopyh r5, cur0, r6                 // host -> VM memory
```

---

## Patron completo: leer/escribir un buffer de raw allocator

```c
// Reservar un buffer de 256 bytes con el allocator manual
mov    r1, 256
alloc  r1               // R0 = puntero host real

// Mover el puntero al cursor usando el patron seguro:
mov    r14, r0          // copiar el puntero a r14 (scratch)
xchg   cur0, r14        // cur0 = r0 (puntero al buffer); r14 = valor anterior de cur0

// Escribir 8 bytes al inicio del buffer:
mov    r5, 0xDEADBEEFCAFEBABE
writecur cur0, r5

// Avanzar 8 bytes y escribir 4 bytes mas:
addcur cur0, 8
mov    r5, 0x12345678
writecur cur0, r5d      // solo 32 bits

// Liberar el buffer (necesitamos recuperar el puntero original):
// El cursor ha avanzado 8 bytes; retrocedemos para obtener la base
addcur cur0, 0xFFF8     // cur0 -= 8 (retroceder al inicio)
mov    r14, cur0        // r14 = puntero base
// Nota: si se ha perdido el puntero base, guardarlo antes en un registro
free   r0               // liberar el puntero original (guardado antes)
```

## Patron completo: inicializar un objeto GC campo a campo

```c
// Crear un objeto con 3 campos qword (24 bytes de payload + 24 bytes de ObjectHeader = 48 total)
mov    r1, 48
gcalloc r1              // R0 = GcHandle
mov    r8, r0           // guardar el handle

gcderef cur1, r8        // cur1 -> inicio del objeto (offset 0)

// El ObjectHeader ocupa los primeros 24 bytes; los campos de usuario empiezan en +24:
addcur cur1, 24         // avanzar al primer campo de usuario

// Escribir campo 0:
mov    r2, 42
writecur cur1, r2

// Escribir campo 1 (avanzar 8 bytes):
addcur cur1, 8
mov    r2, 100
writecur cur1, r2

// Escribir campo 2 (avanzar 8 bytes mas):
addcur cur1, 8
mov    r2, 0
writecur cur1, r2
```

---

## Codificacion detallada

### Instrucciones de 4 bytes (`readcur`, `writecur`, `gcderef`, `vmcopy`, `vcopyh`)

```
+--------+--------+-------------------------------+-------------------------------+
| 0x00   | opcode |         byte_A                |         byte_B                |
+--------+--------+-------------------------------+-------------------------------+
          0xC0 readcur  : [mm cc 0000]  [0000 rrrr]
          0xC1 writecur : [mm cc 0000]  [0000 rrrr]
          0xC2 gcderef  : [00 cc 0000]  [0000 rrrr]
          0xC4 vmcopy   : [00 cc rrrr]  [llll 0000]
          0xC5 vcopyh   : [00 cc rrrr]  [llll 0000]

  mm   = modo de acceso: 00=byte 01=word 10=dword 11=qword
  cc   = cursor index (0-3)
  rrrr = registro general (0-15)
  llll = registro de longitud (0-15)
```

### Instruccion de 6 bytes (`addcur`)

```
+--------+--------+----------+--------+---------+---------+
| 0x00   | 0xC3   |  ctrl    | 0x00   | imm_lo  | imm_hi  |
+--------+--------+----------+--------+---------+---------+

  ctrl: [00 cc 0000]  - bits 5-4 = cursor index (0-3)
  imm_lo / imm_hi: inmediato de 16 bits con signo en little-endian
```

### Tabla de opcodes

| Opcode2 | Mnemonico  | Bytes | Operandos             | Descripcion                          |
| :-----: | :--------- | :---: | :-------------------- | :----------------------------------- |
| `0xC0`  | `readcur`  | 4     | `dest_reg, curN`      | host memory -> registro              |
| `0xC1`  | `writecur` | 4     | `curN, src_reg`       | registro -> host memory              |
| `0xC2`  | `gcderef`  | 4     | `curN, handle_reg`    | GcHandle -> host ptr en cursor       |
| `0xC3`  | `addcur`   | 6     | `curN, imm16`         | cursor +/-= imm16 (avanzar/retroceder)|
| `0xC4`  | `vmcopy`   | 4     | `curN, rSrc, rLen`    | VM memory -> host memory (SIMD)      |
| `0xC5`  | `vcopyh`   | 4     | `rDst, curN, rLen`    | host memory -> VM memory             |

---

Ver tambien: [[GC]], [[XCHG - Exchange]], [[NativeCall (CallN)]], [[NEWOBJRAW y NEWOBJ]]
