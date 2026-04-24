# Instrucciones cursor - acceso a memoria real del host

Las instrucciones cursor permiten leer y escribir directamente en memoria real del proceso host (no en la memoria virtual de la VM). Son el puente entre el bytecode Vesta y:

- Bloques reservados con [[Allocator crudo para FFI  y memoria manual|ALLOC]] (punteros host reales).
- Payloads de objetos gestionados por el [[Generacional (para objetos OOP)|GcHeap]] (accedidos mediante `GCDEREF`).
- Cualquier dirección de host válida almacenada en un registro cursor.

> **¿Por qué hace falta esto?**
> La memoria virtual de la VM se gestiona a través de una capa TLB/VirtualMemory propia. Las instrucciones MOV, ADD, etc. operan sobre registros. Para acceder a un bloque de RAM del host (por ejemplo para rellenar un buffer antes de una llamada nativa), se necesita un mecanismo que desreferencie directamente la dirección host - eso son las instrucciones cursor.

---

## Registros cursor

Los cuatro registros cursor son registros especiales de 64 bits que contienen **direcciones del espacio de host**. Se codifican igual que el resto de registros especiales y pueden usarse con `XCHG` y `MOV` para cargar/guardar direcciones.

| Registro | Codificación |
| :------: | :----------: |
| `cur0`   |   `0b000000` |
| `cur1`   |   `0b000001` |
| `cur2`   |   `0b000010` |
| `cur3`   |   `0b000011` |

> Se recomienda que los registros cursor **solo contengan direcciones host válidas**. Cuando no se usen deben valer `0` (equivalente a `NULL`). Desreferenciar un cursor nulo o una dirección inválida produce comportamiento indefinido.

---

## Instrucciones

### `readcur` - leer desde memoria host

```c
readcur  dest_reg, curN
```

Lee N bytes de la dirección host almacenada en `curN` y los escribe en `dest_reg`. El tamaño viene determinado por el sufijo del registro destino (b=1, w=2, d=4, q=8 bytes).

| Opcode1 | Opcode2 | ctrl_byte     | reg_byte    | Total |
| :-----: | :-----: | :------------ | :---------- | :---: |
| `0x00`  | `0xC0`  | `mm cc 00 00` | `0000 rrrr` |  4 B  |

- `mm` = tamaño: `00`=byte, `01`=word, `10`=dword, `11`=qword
- `cc` = índice del cursor (0-3)
- `rrrr` = registro general destino (R0-R15)

**Ejemplo:**
```c
alloc  r1           // reservar memoria - R0 = ptr host
xchg   cur0, r0     // cur0 = ptr host
readcur r2q, cur0   // r2 = *(uint64_t*)cur0
readcur r3b, cur0   // r3 = *(uint8_t*)cur0  (solo el primer byte)
```

---

### `writecur` - escribir en memoria host

```c
writecur  curN, src_reg
```

Escribe N bytes del registro `src_reg` en la dirección host almacenada en `curN`. El tamaño viene determinado por el sufijo del registro fuente.

| Opcode1 | Opcode2 | ctrl_byte     | reg_byte    | Total |
| :-----: | :-----: | :------------ | :---------- | :---: |
| `0x00`  | `0xC1`  | `mm cc 00 00` | `0000 rrrr` |  4 B  |

Misma codificación que `readcur`; el primer operando es el cursor (destino en memoria) y el segundo es el registro fuente.

**Ejemplo:**
```c
mov    r5, 0xFF00FF00   // valor a escribir
writecur cur0, r5       // *(uint64_t*)cur0 = r5
writecur cur1, r6d      // *(uint32_t*)cur1 = r6d (solo 32 bits)
```

---

### `gcderef` - desreferenciar handle del GC -> cursor

```c
gcderef  curN, handle_reg
```

Toma el `GcHandle` contenido en `handle_reg`, lo desreferencia mediante la `HandleTable` del GC y almacena el puntero raw al payload del objeto en `curN`. A partir de ese momento `curN` contiene una dirección host válida que puede usarse con `readcur` / `writecur`.

| Opcode1 | Opcode2 | ctrl_byte     | reg_byte    | Total |
| :-----: | :-----: | :------------ | :---------- | :---: |
| `0x00`  | `0xC2`  | `00 cc 00 00` | `0000 rrrr` |  4 B  |

- `cc` = índice del cursor destino (0-3)
- `rrrr` = registro general que contiene el handle

> **⚠ Advertencia de invalidación**
> El puntero obtenido con `gcderef` **puede quedar inválido** tras cualquier ciclo de GC (`GCRUN`, o asignación que dispare un minor GC). Si el GC mueve el objeto la dirección en el cursor queda obsoleta. Úsalo solo en fragmentos cortos de código donde no haya asignaciones intermedias, o llama a `gcderef` de nuevo tras cada posible ciclo.

**Ejemplo:**
```c
mov    r1, 64        // 64 bytes de payload
newobj r1            // R0 = GcHandle
gcderef cur2, r0     // cur2 = puntero al payload del objeto
writecur cur2, r3    // escribe el primer campo (qword)
readcur  r4, cur2    // lee el primer campo de vuelta
```

---

### `addcur` - avanzar o retroceder el cursor

```c
addcur  curN, imm16
```

Suma el inmediato con signo de 16 bits al valor del registro cursor `curN`. Permite avanzar (`imm16 > 0`) o retroceder (`imm16 < 0`) sin necesidad del patrón `xchg`/`adds`/`xchg` de tres instrucciones.

| Opcode1 | Opcode2 | ctrl_byte | pad    | imm_lo | imm_hi | Total |
| :-----: | :-----: | :-------- | :----- | :----- | :----- | :---: |
| `0x00`  | `0xC3`  | `00 cc 00 00` | `0x00` | `lo`  | `hi`   |  6 B  |

- `cc` = índice del cursor (0-3) en bits 5-4 del ctrl_byte
- `imm16` = desplazamiento con signo en little-endian (bytes 4-5)
- Rango: −32768 .. +32767 bytes

> Los literales negativos se expresan en complemento a dos hexadecimal: `−8 = 0xFFF8`, `−16 = 0xFFF0`, `−1 = 0xFFFF`.

**Ejemplo:**
```c
// avanzar 8 bytes hacia adelante
addcur cur0, 8

// retroceder 8 bytes (volver a releer desde el inicio)
addcur cur0, 0xFFF8    // 0xFFF8 = -8 en int16

// avanzar al siguiente campo de una estructura de 24 bytes
addcur cur1, 24
```

---

### `vmcopy` - copiar de VM memory a host memory

```c
vmcopy  curN, rSrc, rLen
```

Copia `rLen` bytes desde la VM memory a partir de la dirección virtual contenida en `rSrc` al buffer host apuntado por `curN`. Tras la copia **ambos punteros avanzan automáticamente** `rLen` bytes, lo que facilita copias secuenciales sin instrucciones adicionales.

La copia usa la ruta SIMD más rápida disponible, detectada **una sola vez en tiempo de ejecución**:

| Nivel detectado | Instrucciones | Ancho de bloque |
| :-------------- | :------------ | :-------------: |
| AVX-512F        | `vmovdqu512`  | 64 bytes        |
| AVX2            | `vmovdqu`     | 32 bytes        |
| SSE2            | `movdqu`      | 16 bytes        |
| Escalar         | `memcpy`      | —               |

| Opcode1 | Opcode2 | byte_A               | byte_B           | Total |
| :-----: | :-----: | :------------------- | :--------------- | :---: |
| `0x00`  | `0xC4`  | `00 cc rrrr`         | `llll 0000`      |  4 B  |

- `cc` = índice del cursor destino (0-3) en bits 5-4
- `rrrr` = registro fuente VM (dirección virtual), bits 3-0
- `llll` = registro de longitud, bits 7-4 del byte_B

> Tras `vmcopy`, `curN += rLen` y `rSrc += rLen`. Si no deseas que avancen guarda los valores antes de la instrucción.

**Ejemplo:**
```c
mov    r1, @Absolute("all.hello")   // VM address del string
mov    r2, 13                       // longitud (13 bytes con el null)
alloc  r3                           // host buffer
mov    r10, r0                      // guardar host ptr
xchg   cur0, r0                     // cur0 = host ptr

vmcopy cur0, r1, r2                 // copia 13 bytes de VM -> host, ambos avanzan
```

**Copias secuenciales:**
```c
// copiar dos regiones contiguas de VM memory al mismo buffer host
mov    r3, 8
vmcopy cur0, r1, r3   // copia 8 bytes; cur0 y r1 avanzan 8
vmcopy cur0, r1, r3   // copia los siguientes 8 bytes; cur0 y r1 avanzan 8 más
```

---

### `vcopyh` - copiar de host memory a VM memory

```c
vcopyh  rDst, curN, rLen
```

Inverso de `vmcopy`: copia `rLen` bytes desde el buffer host apuntado por `curN` a la VM memory a partir de la dirección virtual en `rDst`. Tras la copia ambos punteros avanzan `rLen` bytes.

| Opcode1 | Opcode2 | byte_A               | byte_B           | Total |
| :-----: | :-----: | :------------------- | :--------------- | :---: |
| `0x00`  | `0xC5`  | `00 cc rrrr`         | `llll 0000`      |  4 B  |

Misma codificación que `vmcopy`; `rrrr` es el registro destino VM en lugar del fuente.

**Ejemplo:**
```c
// leer resultado de una función nativa y guardarlo en VM memory
mov    r15, 0
calln  @Method("msvcrt.dll:gets")   // r0 = host ptr al string leído
xchg   cur0, r0                     // cur0 = host ptr con resultado

mov    r5, @Absolute("all.buffer")  // VM address del buffer destino
mov    r6, 256                      // longitud máxima
vcopyh r5, cur0, r6                 // copia host -> VM memory
```

---

## Avanzar el cursor - comparativa de patrones

### Patrón antiguo (3 instrucciones)

```c
// avanzar cur0 en N bytes
xchg   cur0, r14     // r14 = cur0 (host ptr)
adds   r14, 8        // r14 += 8
xchg   cur0, r14     // cur0 = r14 + 8
```

### Patrón nuevo con `addcur` (1 instrucción)

```c
addcur cur0, 8       // cur0 += 8
```

### Copia completa VM -> host - comparativa

**Antes** (8+ instrucciones para 16 bytes):
```c
mov    r3, [r1 + r0*8]    // leer qword 0 de VM memory
writecur cur0, r3          // escribir en host[0..7]
xchg   cur0, r4 / adds r4, 8 / xchg cur0, r4   // avanzar cursor
adds   r1, 8
mov    r3, [r1 + r0*8]    // leer qword 1 de VM memory
writecur cur0, r3          // escribir en host[8..15]
```

**Ahora** con `vmcopy` (2 instrucciones):
```c
mov    r2, 16
vmcopy cur0, r1, r2        // copia 16 bytes de VM -> host en una instrucción
```

---

## Patrones de uso completos

### Leer/escribir un buffer de raw allocator

```c
// Reservar buffer de 256 bytes
mov    r1, 256
alloc  r1               // R0 = ptr host

// Guardar puntero en cursor
xchg   cur0, r0

// Escribir 8 bytes en el inicio
mov    r5, 0xDEADBEEFCAFEBABE
writecur cur0, r5

// Avanzar y escribir 4 bytes más
addcur cur0, 8
mov    r5, 0x12345678
writecur cur0, r5d      // solo 32 bits

// Liberar
xchg   r0, cur0
free   r0
```

### Copiar un string desde VM memory para llamada nativa

```c
mov    r1, @Absolute("all.mi_string")   // VM address del string
mov    r2, 32                           // longitud con null
mov    r3, r2
alloc  r3                               // r0 = host buffer
mov    r10, r0                          // guardar para free + calln
xchg   cur0, r0                         // cur0 = host buffer

vmcopy cur0, r1, r2                     // VM -> host en una instrucción (SIMD)

mov    r15, 1
mov    r1, r10
calln  @Method("msvcrt.dll:puts")       // puts(host_ptr)
free   r10
```

### Inicializar un objeto GC campo a campo

```c
mov    r1, 24           // objeto de 3 campos qword (24 bytes)
newobj r1               // R0 = GcHandle
mov    r8, r0

gcderef cur1, r8        // cur1 -> inicio del payload

// campo 0 (offset 0)
mov    r2, 42
writecur cur1, r2

// campo 1 (offset 8) - usando addcur en lugar del patrón xchg/adds/xchg
addcur cur1, 8
mov    r2, 100
writecur cur1, r2

// campo 2 (offset 16)
addcur cur1, 8
mov    r2, 0
writecur cur1, r2
```

### Leer resultado de API nativa de vuelta a VM memory

```c
// Asumiendo que puts devuelve algo y queremos guardarlo
mov    r15, 1
mov    r1, r10
calln  @Method("msvcrt.dll:fgets")      // r0 = host ptr al resultado

xchg   cur0, r0                         // cur0 = host ptr del resultado

mov    r5, @Absolute("all.dest_buf")    // VM address del buffer destino
mov    r6, 128                          // bytes a copiar
vcopyh r5, cur0, r6                     // host -> VM memory
```

---

## Codificación detallada

### Instrucciones de 4 bytes (`readcur`, `writecur`, `gcderef`, `vmcopy`, `vcopyh`)

```
┌────────┬────────┬──────────────────────┬──────────────────────┐
│ 0x00   │ opcode │      byte_A          │      byte_B          │
└────────┴────────┴──────────────────────┴──────────────────────┘
          C0 readcur  : [mm cc 0000] [0000 rrrr]
          C1 writecur : [mm cc 0000] [0000 rrrr]
          C2 gcderef  : [00 cc 0000] [0000 rrrr]
          C4 vmcopy   : [00 cc rrrr] [llll 0000]
          C5 vcopyh   : [00 cc rrrr] [llll 0000]

  mm   = modo de acceso: 00=byte 01=word 10=dword 11=qword
  cc   = cursor index (0-3)
  rrrr = registro general (0-15)
  llll = registro de longitud (0-15)
```

### Instrucción de 6 bytes (`addcur`)

```
┌────────┬────────┬──────────┬──────────┬──────────┬──────────┐
│ 0x00   │  0xC3  │  ctrl    │  0x00    │  imm_lo  │  imm_hi  │
└────────┴────────┴──────────┴──────────┴──────────┴──────────┘

  ctrl: [00 cc 0000]  — bits 5-4 = cursor index (0-3)
  imm_lo / imm_hi: inmediato de 16 bits con signo en little-endian
```

| Opcode2 | Mnemónico | Bytes | Operandos         | Descripción                        |
| :-----: | :-------- | :---: | :---------------- | :--------------------------------- |
| `0xC0`  | `readcur` | 4     | `dest_reg, curN`  | host memory → registro             |
| `0xC1`  | `writecur`| 4     | `curN, src_reg`   | registro → host memory             |
| `0xC2`  | `gcderef` | 4     | `curN, handle_reg`| GcHandle → host ptr en cursor      |
| `0xC3`  | `addcur`  | 6     | `curN, imm16`     | cursor ±= imm16 (avanzar/retroceder)|
| `0xC4`  | `vmcopy`  | 4     | `curN, rSrc, rLen`| VM memory → host memory (SIMD)     |
| `0xC5`  | `vcopyh`  | 4     | `rDst, curN, rLen`| host memory → VM memory            |

---

Ver también: [[GC]], [[Generacional (para objetos OOP)|GcHeap]], [[Allocator crudo para FFI  y memoria manual|RawAllocator]], [[NativeCall (CallN)]], [[XCHG - Exchange]], [[Ejemplos/HolaMundo|Hello World]]
