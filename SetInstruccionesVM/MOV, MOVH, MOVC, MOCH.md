# Instrucciones MOV - Mover datos

Las instrucciones MOV son las mas fundamentales de la VM: sirven para **copiar valores de
un lugar a otro**. Sin MOV no se puede hacer nada util, ya que todos los demas calculos
trabajan sobre registros que primero hay que llenar con datos.

Analogia: imagina que los registros (R0 a R15) son cajones en un escritorio. MOV es la
accion de copiar un numero de un cajon a otro, o escribir un numero directamente en un
cajon. El cajon de origen no cambia; solo el destino recibe la copia.

| Instruccion   | opcode0 | opcode1 | Tamano   | Descripcion                                         |
| :-----------: | :-----: | :-----: | :------: | :-------------------------------------------------- |
| `mov r, r`    |  0x00   |  0x14   | 4 bytes  | Copiar de registro a registro                       |
| `mov r, imm`  |  0x00   |  0x15   | 10 bytes | Escribir un valor constante en un registro          |
| `mov r, [m]`  |  0x00   |  0x16   | 6 bytes  | Leer de memoria VM (modo SIB)                       |
| `movh r, [m]` |  0x00   |  0x16   | 6 bytes  | Leer de memoria HOST (modo SIB, bit s=1)            |
| `movc r, [m]` |  0x00   |  0x1E   | 4 bytes  | Copiar condicionalmente (VM memory)                 |
| `movch r,[m]` |  0x00   |  0x1E   | 4 bytes  | Copiar condicionalmente (host memory, bits 7-6=10)  |
| `movc r, r`   |  0x00   |  0x1F   | 4 bytes  | Copiar registro a registro si se cumple condicion   |

Implementacion: `src/runtime/exec_instruction_alu.cpp` (y `decode_instruction.cpp`)

---

## MOV - copiar entre registros o cargar un inmediato

### Registro a registro

```c
mov r1, r2    // r1 = r2 (r2 no cambia)
```

La operacion mas simple: copia el contenido de `r2` a `r1`. Ambos registros son de 64 bits
por defecto, pero se puede usar un sufijo para operar con un subconjunto de bits:

| Sufijo | Bits leidos/escritos | Ejemplo          |
| :----: | :------------------: | :--------------- |
| (nada) | 64 bits (qword)      | `mov r1, r2`     |
| `d`    | 32 bits bajos        | `mov r1d, r2d`   |
| `w`    | 16 bits bajos        | `mov r1w, r2w`   |
| `b`    | 8 bits bajos         | `mov r1b, r2b`   |

```c
mov  r0, r1      // r0 = r1 (64 bits completos)
mov  r0d, r1d    // r0[31:0] = r1[31:0], r0[63:32] = 0 (zero-extend)
mov  r0w, r1w    // r0[15:0] = r1[15:0], r0[63:16] = 0
mov  r0b, r1b    // r0[7:0]  = r1[7:0],  r0[63:8]  = 0
```

### Registro a/desde registro especial

Algunos registros de la VM no son los R0-R15 de proposito general. Son registros especiales
como `rip` (program counter), `rsp` (stack pointer), `rbp` (base pointer), los flags, etc.
Para acceder a ellos se usa la variante con `s=1` en el ctrl byte:

```c
// Leer el program counter actual (en que instruccion estamos):
mov r0, rip     // r0 = valor actual del PC

// Leer el stack pointer:
mov r1, rsp     // r1 = cima actual de la pila
```

### Inmediato a registro

```c
mov r0, 42             // r0 = 42
mov r1, 0xFF00         // r1 = 65280
mov r2, 0x3FF0000000000000  // r2 = bits de 1.0 en IEEE 754 doble
```

Un inmediato es un valor constante codificado directamente en la instruccion. La instruccion
`mov r, imm` ocupa 10 bytes: 2 de prefijo+opcode + 1 de ctrl + 1 padding + 8 de inmediato.

El inmediato siempre se escribe como valor de 64 bits aunque el numero sea pequeno.

---

## MOV con acceso a memoria VM (modo SIB)

Cuando se quiere leer o escribir en la **memoria virtual de la VM**, se usa el modo SIB
(Scale + Index + Base). La direccion efectiva se calcula como:

```
direccion = base + index * scale
```

Donde `base` e `index` son registros y `scale` es un multiplo de 1, 2, 4 u 8.

```c
// Leer 4 bytes de VM memory en la direccion almacenada en r2:
// (index = r1 con valor 0, scale = 1, base = r2)
mov r0d, [r1*1 + r2]    // r0 = *(uint32_t*)(0*1 + r2) en VM memory

// Patron mas comun: acceder a un array de qwords (8 bytes cada uno)
// r3 = indice del array, r4 = direccion base del array
mov r0, [r3*8 + r4]     // r0 = array[r3]

// Acceder a una estructura: r5 = base de la estructura
// El campo a offset +16 se accede con index=r_cero (= 0), base=r5
mov r1, 16
mov r0, [r1*1 + r5]     // r0 = *(uint64_t*)(r5 + 16) en VM memory
```

Cuando `has_index = 0` (bit 1 del ctrl SIB), la direccion es simplemente `base`:

```c
// Sin index: solo base
mov r0, [r2]    // r0 = *(uint64_t*)r2 en VM memory (has_index = 0)
```

---

## MOVH - acceso a memoria HOST (hardware real)

`MOVH` es identico a `MOV` en modo SIB, salvo que opera sobre la **memoria real del proceso
host** en lugar de la memoria virtual de la VM. Se diferencia por el bit `s=1` en el ctrl.

### Para que sirve MOVH

La VM tiene su propio espacio de memoria virtual (gestionado por TLB y ArenaManager). Pero
a veces necesitas leer/escribir directamente en la memoria del proceso host: por ejemplo,
para pasar datos a una funcion nativa (FFI), o para acceder a estructuras del runtime.

```c
// Leer 4 bytes de la direccion host 0x1F000:
// (index = r1 con valor 0, scale = 1, base = r2 = 0x1F000)
xor  r1, r1         // r1 = 0 (index)
mov  r2, 0x1F000    // r2 = direccion host
movh r0d, [r1*1 + r2]  // r0 = *(uint32_t*)(host addr 0x1F000)

// Escribir en memoria host: r3 = ptr host, r4 = valor a escribir
xor  r1, r1
movh [r1*1 + r3], r4   // *(uint64_t*)(host addr en r3) = r4
```

> **Advertencia:** MOVH te da acceso directo a la RAM del proceso host. Un acceso a una
> direccion invalida producira una violacion de segmento (SIGSEGV / Access Violation) que
> tumbara la VM entera. Usa MOVH solo cuando sabes con certeza que la direccion es valida.

> **Diferencia con cursores:** Las instrucciones `readcur`/`writecur` hacen lo mismo que
> MOVH pero de forma mas segura y flexible. Prefiere cursores para acceso FFI. MOVH es
> util cuando el calculo de la direccion via SIB encaja exactamente.

---

## MOVC - mover condicionalmente (sin salto)

`MOVC` copia un valor de un lugar a otro **solo si un flag de la VM esta activo**. Si el
flag no esta activo, la instruccion no hace nada.

### Para que sirve

En procesadores modernos, los saltos condicionales (`jmp.je`, `jmp.jlt`, etc.) pueden ser
lentos cuando el predictor de ramas falla. `MOVC` permite implementar selecciones
condicionales simples sin ningun salto, lo que es mas rapido y predecible.

Analogia: en lugar de "si llueve, coge el paraguas; si no, no hagas nada", `MOVC` es
"coge el paraguas (y si no lluvia, lo sueltas)" pero en un solo paso atomico.

### Flags disponibles

| Codigo | Nombre | Cuando esta activo                                  | Uso tipico                    |
| :----: | :----: | :-------------------------------------------------- | :---------------------------- |
| 0      | SF     | El resultado de la ultima operacion fue negativo    | Comparacion con signo         |
| 1      | ZF     | El resultado de la ultima operacion fue cero        | Comparacion de igualdad       |
| 2      | CF     | Hubo acarreo (carry) en la ultima suma/resta        | Aritmetica sin signo          |
| 3      | OF     | Hubo desbordamiento en la ultima operacion con signo| Deteccion de overflow         |
| 4      | DM     | El proceso esta en modo distribuido                 | Comportamiento condicional VM |

### MOVC registro a registro

```c
movc r3, r4, ZF    // si ZF == 1: r3 = r4; si ZF == 0: r3 no cambia
```

```c
// Ejemplo: calcular el maximo de r1 y r2 sin salto
cmp  r1, r2        // establece flags segun r1 - r2
movc r1, r2, SF    // si r1 < r2 (SF=1), r1 = r2 -> r1 = max(r1, r2)

// Ejemplo: poner a cero si negativo (clamp a 0)
xor  r3, r3        // r3 = 0
cmp  r1, r3        // comparar r1 con 0
movc r1, r3, SF    // si r1 < 0 (SF=1), r1 = 0

// Ejemplo: detectar overflow en suma
adds r1, r2        // suma con signo (actualiza OF)
movc r0, r5, OF    // si desbordamiento (OF=1), r0 = valor de error en r5
```

### MOVC con memoria VM

```c
movc r1, [r2], ZF     // si ZF==1: r1 = *(vm_addr en r2)  (leer de VM memory)
movc [r1], r2, ZF     // si ZF==1: *(vm_addr en r1) = r2  (escribir en VM memory)
```

```c
// Cargar un valor alternativo desde memoria si la comparacion fue igual
cmp  r0, r3
movc r1, [r4], ZF     // si r0 == r3: r1 = *(VM memory en r4)
```

---

## MOVCH - mover condicionalmente en memoria HOST

`MOVCH` es identico a `MOVC` pero opera sobre la **memoria real del host** en lugar de
la VM memory. Se usa en combinacion con cursores o punteros FFI.

```c
movch r1, [r2], ZF    // si ZF==1: r1 = *(host_ptr en r2)
movch [r1], r2, ZF    // si ZF==1: *(host_ptr en r1) = r2
```

```c
// Cargar desde un buffer host solo si una comparacion fue igual:
mov   r2, 0x1F800     // r2 = puntero host al buffer
cmp   r0, r3
movch r1, [r2], ZF    // si r0 == r3: r1 = *(uint64_t*)0x1F800 en host
```

Las mismas advertencias que para MOVH aplican aqui: la direccion host debe ser valida.

---

## Resumen de usos comunes

| Tarea                                | Instruccion preferida         |
| :----------------------------------- | :---------------------------- |
| Copiar entre registros               | `mov r1, r2`                  |
| Cargar constante                     | `mov r1, 42`                  |
| Leer de VM memory en offset fijo     | `mov r0, [r_zero*1 + r_base]` |
| Leer de array en VM memory           | `mov r0, [r_idx*8 + r_base]`  |
| Leer de host memory (FFI)            | `movh r0, [r_zero*1 + r_ptr]` |
| Seleccion condicional sin salto      | `movc r_dst, r_src, ZF`       |
| Cargar de host condicionalmente      | `movch r_dst, [r_ptr], CF`    |

---

## Codificacion binaria

### MOV registro a registro (opcode2 = 0x14, 4 bytes)

```
+--------+--------+--------+--------+
| 0x00   | 0x14   | ctrl   | regs   |
+--------+--------+--------+--------+

ctrl (byte2):
  bits 7-6 = mode  (00=byte, 01=word, 10=dword, 11=qword)
  bit  5   = s     (0=registro estandar R0-R15, 1=registro especial)
  bit  4   = d     (0=destino es el reg general, 1=fuente es el reg especial)
  bits 3-0 = reg1  (registro R0-R15)

regs (byte3):
  bits 7-4 = reg2_or_spc   (reg fuente normal, o nibble bajo del codigo especial)
  bits 3-0 = reg1           (indice del registro destino)
```

### MOV inmediato a registro (opcode2 = 0x15, 10 bytes)

```
+--------+--------+--------+--------+--------+--------+--------+--------+--------+--------+
| 0x00   | 0x15   | ctrl   | 0xFF   | imm[0] | imm[1] | imm[2] | imm[3] | imm[4] |  ...  |
+--------+--------+--------+--------+--------+--------+--------+--------+--------+--------+

ctrl (byte2):
  bits 7-6 = mode
  bit  5   = 0 (siempre para inmediato estandar)
  bit  4   = d (0=destino es reg; 1=destino es [mem] apuntado por reg)
  bits 3-0 = reg1

bytes 4-11 = inmediato de 64 bits en little-endian
```

### MOV/MOVH modo SIB (opcode2 = 0x16, 6 bytes)

```
+--------+--------+--------+--------+--------+--------+
| 0x00   | 0x16   | ctrl   | regs   | index  | 0x00   |
+--------+--------+--------+--------+--------+--------+

ctrl (byte2):
  bits 7-6 = mode
  bit  5   = s     (0 = MOV VM memory; 1 = MOVH host memory)
  bit  4   = d     (0 = reg es destino / mem es fuente; 1 = mem es destino)
  bits 3-2 = scale (00=x1, 01=x2, 10=x4, 11=x8)
  bit  1   = has_index (0 = addr=base solamente; 1 = addr=base+index*scale)
  bit  0   = reservado

regs (byte3):
  bits 7-4 = reg1  (registro de datos: destino o fuente)
  bits 3-0 = base  (registro base de la direccion)

byte4 = index (registro indice; solo relevante si has_index=1)
byte5 = 0x00 (padding)
```

### MOVC/MOVCH con memoria (opcode2 = 0x1E, 4 bytes)

```
+--------+--------+--------+--------+
| 0x00   | 0x1E   | ctrl   | byte4  |
+--------+--------+--------+--------+

ctrl (byte2):
  bits 7-6 = host_bits (00=MOVC VM memory; 10=MOVCH host memory)
  bit  5   = d  (0=reg es destino, mem es fuente; 1=mem es destino, reg es fuente)
  bits 4-0 = r_nonbracket (registro que NO esta entre corchetes)

byte4:
  bits 7-5 = flag_code (SF=0, ZF=1, CF=2, OF=3, DM=4)
  bits 4-0 = r_bracket  (registro que esta entre corchetes: la direccion)
```

### MOVC registro a registro (opcode2 = 0x1F, 4 bytes)

```
+--------+--------+--------+--------+
| 0x00   | 0x1F   | ctrl   | byte4  |
+--------+--------+--------+--------+

ctrl (byte2):
  bits 7-4 = 0000 (reservado)
  bits 3-0 = reg_dst (registro destino)

byte4:
  bits 7-5 = flag_code (SF=0, ZF=1, CF=2, OF=3, DM=4)
  bits 4-0 = reg_src  (registro fuente)
```

---

## Tabla completa de variantes

| Instruccion                   | op0  | op1  | Modo  | Bytes |
| :---------------------------- | :--: | :--: | :---: | :---: |
| `mov reg1, reg2`              | 0x00 | 0x14 | REG   |  4    |
| `mov reg, reg_especial`       | 0x00 | 0x14 | REG   |  4    |
| `mov reg, imm64`              | 0x00 | 0x15 | IMM   | 10    |
| `mov [reg], imm64`            | 0x00 | 0x15 | IMM   | 10    |
| `mov reg, [base+idx*sc]`      | 0x00 | 0x16 | SIB   |  6    |
| `mov [base+idx*sc], reg`      | 0x00 | 0x16 | SIB   |  6    |
| `movh reg, [base+idx*sc]`     | 0x00 | 0x16 | SIB   |  6    |
| `movh [base+idx*sc], reg`     | 0x00 | 0x16 | SIB   |  6    |
| `movc reg, [reg], flag`       | 0x00 | 0x1E | COND  |  4    |
| `movc [reg], reg, flag`       | 0x00 | 0x1E | COND  |  4    |
| `movch reg, [reg], flag`      | 0x00 | 0x1E | COND  |  4    |
| `movch [reg], reg, flag`      | 0x00 | 0x1E | COND  |  4    |
| `movc reg, reg, flag`         | 0x00 | 0x1F | COND  |  4    |

---

Ver tambien: [[Aritmetica y logica]], [[JMP]], [[cursor]], [[XCHG - Exchange]]
