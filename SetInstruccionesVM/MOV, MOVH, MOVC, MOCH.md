## MOV

Permite mover valores entre registros. También permite mover valores inmediatos a registros.

## MOVH
Permite mover valores entre los registros de la VM y direcciones de host arbitrarias.
```c
// MOVH reg1, [index*scalar + base]
// MOVH [index*scalar + base], reg1

// permite leer 4 bytes de la direccion host resultante de la operacion
// r1 * 1 + r2
uint32_t r0w = *((uint32_t*)(((void*)r1) * 1) + r2)

MOVH r0w, [r1 * 1 + r2]

// ejemplo para leer memoria del host, por ejemplo la direccion 0x1f000
xor r1, r1
mov r2, 0x1f000
MOVH r0w, [r1 * 1 + r2] // r0w = *((uint32_t*)((0 * 1) + 0x1f000)
```
## MOVC
Permite mover valores si una codicion se da
```c
// MOVC dest, src, flag

// Copiar si el resultado anterior fue cero
CMP r1, r2
MOVC r3, r4, ZF  // si (r1 == r2) -> r3 = r4

// Selección condicional sin saltos
SUB r1, r2
MOVC r1, r3, ZF // si ((r1 - r2) == 0) -> r1 = r3

// Copiar si el resultado fue negativo
SUB r1, r2
MOVC r3, r4, SF // si r1 < r2 -> r3 = r4

// Implementar min(a, b)
CMP r1, r2
MOVC r1, r2, SF // si r1 < r2 -> r1 = r2 (max)

// Clamp negativo a cero
CMP r1, 0
MOVC r1, rZero, SF // si r1 < 0 -> r1 = 0

// Copiar si hubo acarreo (útil para unsigned)
ADDU r1, r2
MOVC r3, r4, CF // si hubo carry -> r3 = r4

// Detección de overflow en suma sin signo
ADDU rA, rB
MOVC rErr, rOne, CF // si CF=1 -> error

// Comparación unsigned
CMPU r1, r2
MOVC r3, r4, CF // si r1 < r2 (unsigned) -> r3 = r4

// Copiar si hubo overflow aritmético
ADDS r1, r2
MOVC rErr, rOne, OF // si overflow -> rErr = 1

// Saturación aritmética
ADDS r1, r2
MOVC r1, rMax, OF // si overflow -> r1 = MAX

// Detección de overflow en resta
SUBS rA, rB
MOVC rFlag, rOne, OF

// Copiar solo si estamos en modo distribuido
MOVC r1, r2, DM // si DM=1 -> r1 = r2

// Acceso a memoria condicional según modo
MOVC [rAddr], rVal, DM

// Selección booleana
CMP rCond, 1
MOVC rOut, rTrue, ZF
MOVC rOut, rFalse, NZ
```
## MOVCH
permite acceder a memoria del host bajo alguna clase de condicion.
**MOVCH** = _Conditional Move Host_
- `reg1` = destino
- `reg2` = fuente
- `flag` = (SF, ZF, CF, OF, DM)
Solo ejecuta la copia si el flag está activo.

```c
MOVCH reg1, [reg2], flag
MOVCH [reg1], reg2, flag
```


### Leer un valor host solo si el resultado anterior fue cero (ZF)
```c
CMP rA, rB
MOVCH r1, [r2], ZF // si rA == rB -> r1 = *(host_ptr = r2)
```
Útil para:
- cargar un valor alternativo
- seleccionar rutas sin saltos
- implementar “load if equal”

### Leer desde host solo si el resultado fue negativo (SF)
```c
SUB rX, rY
MOVCH r3, [r4], SF // si rX < rY -> r3 = *(host_ptr = r4)
```
Útil para:
- saturación
- clamps
- selección condicional

### Leer desde host solo si hubo carry (CF)
```c
ADDU r1, r2
MOVCH r3, [r4], CF // si hubo carry -> r3 = *(host_ptr = r4)
```
Útil para:
- aritmética sin signo
- detección de overflow unsigned

### Leer desde host solo si hubo overflow (OF)

### Leer desde host solo si estamos en modo distribuido (DM)
```c
MOVCH rThreadID, [rBase], DM

```
Útil para:
- sistemas paralelos
- modos especiales de ejecución
- lectura condicional de metadatos

----

## Codificación

Permite los mismos modos de codificación que las [[Aritmetica y logica|instrucciones arimetico logicas]], incluyendo el bit `has_index` del ctrl SIB (bit 1 de byte2 en formato SIB): cuando es 0 la dirección efectiva es solo `base`; cuando es 1 es `base + index*scale`.

| Instrucción                          | opcode1 | opcode2 |           byte3 (ctrl)        |      byte4 (regs)      | byte5 | byte6 | byte7 |   bytes 8-11 | total bytes |
| :----------------------------------- | :-----: | :-----: | :---------------------------: | :--------------------: | :---: | :---: | :---: | :----------: | :---------: |
| ``MOV reg1, reg2``                   |   0x0   |  0x14   | 0b`mode` 0 d 0000  d=0        | `reg2<<4`\|`reg1`      |       |       |       |              |      4      |
| ``MOV reg_ext, reg``                 |   0x0   |  0x14   | 0b`mode` 1 0 0000             | `(spc&0xF)<<4`\|`reg`  |       |       |       |              |      4      |
| ``MOV reg, reg_ext``                 |   0x0   |  0x14   | 0b`mode` 1 1 0000             | `(spc&0xF)<<4`\|`reg`  |       |       |       |              |      4      |
| ``MOV reg1, inmmed``                 |   0x0   |  0x15   | 0b`mode` 0 0 `reg1`           | 0xFF                   | 0xFF  | 0xFF  | 0xFF  | 0xFFFFFFFF   |     10      |
| ``MOV [reg], inmmed``                |   0x0   |  0x15   | 0b`mode` 0 1 `reg1`           | 0xFF                   | 0xFF  | 0xFF  | 0xFF  | 0xFFFFFFFF   |     10      |
| ``MOV reg_ext, inmmed``              |   0x0   |  0x15   | 0b`mode` 1 1 `reg1`           | 0xFF                   | 0xFF  | 0xFF  | 0xFF  | 0xFFFFFFFF   |     10      |
| ``MOV reg1, [base+idx*sc]``          |   0x0   |  0x16   | 0b`mode` 0 `d` `sc`  d=0      | `reg1<<4`\|`base`      | `idx` | 0x00  |       |              |      6      |
| ``MOV [base+idx*sc], reg1``          |   0x0   |  0x16   | 0b`mode` 0 `d` `sc`  d=1      | `reg1<<4`\|`base`      | `idx` | 0x00  |       |              |      6      |
| ``MOV reg1, [base]``                 |   0x0   |  0x16   | 0b`mode` 0 `d` 00  d=0  hi=0  | `reg1<<4`\|`base`      | 0x00  | 0x00  |       |              |      6      |
| ``MOVH reg1, [base+idx*sc]``         |   0x0   |  0x16   | 0b`mode` 1 `d` `sc`  d=0      | `reg1<<4`\|`base`      | `idx` | 0x00  |       |              |      6      |
| ``MOVH [base+idx*sc], reg1``         |   0x0   |  0x16   | 0b`mode` 1 `d` `sc`  d=1      | `reg1<<4`\|`base`      | `idx` | 0x00  |       |              |      6      |

### Codificacion de byte3 (ctrl) para MOV reg-reg

```
bits 7-6 = mode  (0=byte, 1=word, 2=dword, 3=qword)
bit  5   = s     (0=reg estandar, 1=reg especial/ext)
bit  4   = d     (0=destino es el reg general, 1=destino es mem o reg_ext)
bits 3-0 = reg1  (registro general; para INMED contiene el numero de registro)
```

### Codificacion de reg_ext (s=1)

Cuando `s=1` en byte3, el byte4 codifica:
```
bits 7-4 = spc & 0xF   (nibble bajo del codigo de registro especial)
bits 3-0 = reg         (registro general fuente/destino)
```

El codigo especial completo se reconstruye como: `special = (mode << 4) | byte4[7:4]`

- `d=0` -> `reg_ext <- reg`  (escribe en el registro especial)
- `d=1` -> `reg <- reg_ext`  (lee del registro especial)

>Posiblemente el introducir instrucciones de longitud variable dificulte implementar un modelo de "decodificación paralela estilo superscalar" en un futuro


| Instrucción                  | op1  | op2  | byte3 (ctrl)                   | byte4                    | bytes |
| ---------------------------- | :--: | :--: | ------------------------------ | ------------------------ | :---: |
| ``MOVC reg1, [reg2], flag``  | 0x00 | 0x1E | `0b00` \| `d=0` \| `reg1`      | `flag<<5` \| `reg2`      |   4   |
| ``MOVC [reg1], reg2, flag``  | 0x00 | 0x1E | `0b00` \| `d=1` \| `reg2`      | `flag<<5` \| `reg1`      |   4   |
| ``MOVCH reg1, [reg2], flag`` | 0x00 | 0x1E | `0b10` \| `d=0` \| `reg1`      | `flag<<5` \| `reg2`      |   4   |
| ``MOVCH [reg1], reg2, flag`` | 0x00 | 0x1E | `0b10` \| `d=1` \| `reg2`      | `flag<<5` \| `reg1`      |   4   |
| ``MOVC reg1, reg2, flag``    | 0x00 | 0x1F | `reg1 & 0xF`                   | `flag<<5` \| `reg2`      |   4   |

> **Nota:** Los opcodes `0x17` y `0x18` están tomados por `AND` y `OR`. MOVC/MOVCH usan `0x1E` (variante memoria) y `0x1F` (variante registro–registro).

### **Byte 3 (ctrl) — variante memoria (0x1E)**

```
bits 7-6 = host_bits  (0b00 = MOVC -> VM mem,  0b10 = MOVCH -> host mem)
bit  5   = d          (0 = reg es destino / [] es fuente;  1 = [] es destino / reg es fuente)
bits 4-0 = r_nonbracket  (registro que NO está entre corchetes, 5 bits)
```

### **Byte 3 (ctrl) — variante registro (0x1F)**

```
bits 7-4 = 0000       (reservado)
bits 3-0 = reg1       (registro destino)
```

### **Byte 4 (común a 0x1E y 0x1F)**

```
bits 7-5 = flag_code  (3 bits: SF=0, ZF=1, CF=2, OF=3, DM=4)
bits 4-0 = r_bracket  (variante 0x1E: registro dentro de [];  variante 0x1F: registro fuente)
```

### **Total: 4 bytes** (`0x00` + `opcode2` + `ctrl` + `byte4`)