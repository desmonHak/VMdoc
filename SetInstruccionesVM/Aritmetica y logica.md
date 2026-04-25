# Aritmetica y Logica - Instrucciones de calculo

Las instrucciones aritmeticas y logicas realizan operaciones matematicas y de bits sobre
los registros de la VM. Son la base de cualquier calculo: sumar, restar, multiplicar,
dividir, comparar, y operar bit a bit.

---

## Tabla resumen de operaciones

|     Operacion      | **Sin signo** | **Con signo** | **Sin variante** |
| :----------------: | :-----------: | :-----------: | :--------------: |
|      **Suma**      |    `ADDU`     |    `ADDS`     |                  |
|     **Resta**      |    `SUBU`     |    `SUBS`     |                  |
| **Multiplicacion** |    `MULU`     |    `MULS`     |                  |
|    **Division**    |    `DIVU`     |    `DIVS`     |                  |
|  **Comparacion**   |    `CMPU`     |    `CMPS`     |                  |
|    Incrementar     |               |               |    `INC`         |
|    Decrementar     |               |               |    `DEC`         |
|      AND bit       |               |               |    `AND`         |
|       OR bit       |               |               |    `OR`          |
|      XOR bit       |               |               |    `XOR`         |
|      NOT bit       |               |               |    `NOT`         |
| Despl. izquierda   |               |               |    `SHL`         |
|  Despl. logico der |               |               |    `SHR`         |
| Despl. arit. der   |               |               |    `SAR`         |

---

## Sin signo (U) vs con signo (S): por que importa

Un registro de 64 bits almacena 64 bits binarios. La interpretacion de esos bits como
"numero positivo" o "numero con signo (positivo o negativo)" cambia el resultado de
algunas operaciones:

- **Sin signo (U)**: los 64 bits representan un numero entre 0 y 2^64-1.
  Ejemplo: `0xFFFFFFFFFFFFFFFF` = 18446744073709551615.
- **Con signo (S)**: el bit mas significativo indica el signo (0 = positivo, 1 = negativo).
  Ejemplo: `0xFFFFFFFFFFFFFFFF` = -1 (en complemento a dos).

Para suma, resta y multiplicacion la diferencia afecta principalmente a los **flags** de
desbordamiento (OF, CF). Para division, afecta al resultado.

Regla practica:
- Usa variantes **U** para: contadores, tamanos, punteros, mascaras de bits.
- Usa variantes **S** para: temperaturas, diferencias, cualquier valor que pueda ser negativo.

---

## Nomenclatura de registros

En la tabla de codificacion: `reg1`, `reg2` son indices de 4 bits (0-15) que identifican
los registros r0-r15. Cuando una instruccion usa dos registros, ambos caben en un solo byte.

---

## Modos de direccionamiento

### Modo basico (registro a registro)

El mas simple: opera directamente con dos registros.

```c
addu r1, r2     // r1 = r1 + r2
subu r3, r4     // r3 = r3 - r4
mulu r5, r6     // r5 = r5 * r6
```

### Modo inmediato (registro con valor constante)

El segundo operando es una constante de 64 bits codificada en la instruccion.

```c
addu r1, 10     // r1 = r1 + 10
subu r3, 5      // r3 = r3 - 5
cmpu r7, 0      // comparar r7 con 0
```

### Modo memoria con indice base (SIB: Scale + Index + Base)

Este modo accede a memoria usando una formula: `EA = base + index * scale`.
Es equivalente al modo SIB de Intel x86. Se usa para acceder a arrays y estructuras.

```c
// dir=0: reg recibe el valor de mem[base + index*scale]
addu r1, [r14 + r13*8]    // r1 = r1 + mem64[r14 + r13*8]
adds r1, [r14 + r13*8]    // (con signo)

// dir=1: mem[base + index*scale] recibe el resultado
addu [r14 + r12*8], r6    // mem64[r14 + r12*8] += r6

// solo base, sin indice:
addu r4, [r14]            // r4 = r4 + mem64[r14]
mov  r1, [r14]            // r1 = mem64[r14]
```

El `scale` puede ser x1, x2, x4 o x8, permitiendo indexar arrays de elementos de
1, 2, 4 u 8 bytes respectivamente sin multiplicar explicitamente.

#### Codificacion SIB (6 bytes)

```
+--------+--------+--------+--------+--------+--------+
| 0x00   | opcode |  ctrl  |  regs  | index  |  pad   |
+--------+--------+--------+--------+--------+--------+
  byte0    byte1    byte2    byte3    byte4    byte5

ctrl (byte2):
  bits 7-6 = mode      (0=byte, 1=word, 2=dword, 3=qword)
  bit  5   = signed    (0=unsigned, 1=signed)
  bit  4   = dir       (0=reg<-mem, 1=mem<-reg)
  bits 3-2 = scale     (0=x1, 1=x2, 2=x4, 3=x8)
  bit  1   = has_index (0=solo base, 1=base+index*scale)
  bit  0   = 0         (reservado)

regs (byte3):
  bits 7-4 = dst_reg  (registro destino/fuente segun dir)
  bits 3-0 = base_reg (registro base)

index (byte4):
  bits 7-4 = index_reg
  bits 3-0 = 0000     (reservado)
```

---

## Acceso a memoria con sufijos de tamano

El tamano del registro dentro de corchetes `[]` indica cuantos bytes se leen o escriben:

```c
mov r0, 0x123456789ABCDEF   // r0 = la direccion base

// Leer distintos tamanos desde esa direccion:
adds [r0b], 0x10    // leer/escribir 1 byte  (uint8_t)
adds [r0w], 0x10    // leer/escribir 2 bytes (uint16_t)
adds [r0d], 0x10    // leer/escribir 4 bytes (uint32_t)
adds [r0],  0x10    // leer/escribir 8 bytes (uint64_t, por defecto)
```

Equivalente en C:
```c
uint8_t  *p = (uint8_t*)0x123456789ABCDEF;
*p += 0x10;                     // adds [r0b], 0x10

uint32_t *p32 = (uint32_t*)p;
*p32 += 0x1000;                 // adds [r0d], 0x1000
```

---

## INC y DEC - Incrementar y decrementar en 1

```c
inc r1    // r1 = r1 + 1
dec r1    // r1 = r1 - 1
```

| Instruccion | opcode1 | byte2            | Tamano |
| :---------: | :-----: | :--------------: | :----: |
|     INC     |  0x4    | `0b00` \| `reg`  |   2 B  |
|     DEC     |  0x4    | `0b01` \| `reg`  |   2 B  |

Son equivalentes a `addu reg, 1` y `subu reg, 1` pero en solo 2 bytes en lugar de 10.

---

## ADD - Suma (con y sin signo)

```c
addu r1, r2          // r1 = r1 + r2 (sin signo)
adds r1, r2          // r1 = r1 + r2 (con signo)
addu r1, 42          // r1 = r1 + 42 (inmediato)
addu r1, [r14 + r13*8]  // r1 = r1 + mem[r14+r13*8] (SIB)
```

La suma actualiza los flags: ZF (si resultado es 0), SF (si negativo), CF (acarreo sin signo), OF (overflow con signo).

| Instruccion                   | opcode1 | opcode2 | Tamano |
| :---------------------------- | :-----: | :-----: | :----: |
| `ADDU/ADDS reg1, reg2`        |  0x00   |  0x05   |   4 B  |
| `ADDU/ADDS reg1, inmmed`      |  0x00   |  0x06   |  10 B  |
| `ADDU/ADDS reg, [SIB]`        |  0x00   |  0x07   |   6 B  |

---

## SUB - Resta (con y sin signo)

```c
subu r1, r2          // r1 = r1 - r2 (sin signo)
subs r1, r2          // r1 = r1 - r2 (con signo)
subu r1, 5           // r1 = r1 - 5 (inmediato)
```

| Instruccion                   | opcode1 | opcode2 | Tamano |
| :---------------------------- | :-----: | :-----: | :----: |
| `SUBU/SUBS reg1, reg2`        |  0x00   |  0x08   |   4 B  |
| `SUBU/SUBS reg1, inmmed`      |  0x00   |  0x09   |  10 B  |
| `SUBU/SUBS reg, [SIB]`        |  0x00   |  0x0A   |   6 B  |

---

## MUL - Multiplicacion (con y sin signo)

```c
mulu r1, r2          // r1 = r1 * r2 (sin signo)
muls r1, r2          // r1 = r1 * r2 (con signo)
mulu r1, 3           // r1 = r1 * 3 (inmediato)
```

| Instruccion                   | opcode1 | opcode2 | Tamano |
| :---------------------------- | :-----: | :-----: | :----: |
| `MULU/MULS reg1, reg2`        |  0x00   |  0x0B   |   4 B  |
| `MULU/MULS reg1, inmmed`      |  0x00   |  0x0C   |  10 B  |
| `MULU/MULS reg, [SIB]`        |  0x00   |  0x0D   |   6 B  |

---

## DIV - Division (con y sin signo)

```c
divu r1, r2          // r1 = r1 / r2 (sin signo)
divs r1, r2          // r1 = r1 / r2 (con signo)
```

> **IMPORTANTE:** Division por cero produce `EVT_ERROR` (el proceso muere).
> Siempre comprueba que el divisor no sea cero antes de dividir.

| Instruccion                   | opcode1 | opcode2 | Tamano |
| :---------------------------- | :-----: | :-----: | :----: |
| `DIVU/DIVS reg1, reg2`        |  0x00   |  0x0E   |   4 B  |
| `DIVU/DIVS reg1, inmmed`      |  0x00   |  0x0F   |  10 B  |
| `DIVU/DIVS reg, [SIB]`        |  0x00   |  0x10   |   6 B  |

---

## CMP - Comparacion (con y sin signo)

```c
cmpu r1, r2          // comparar r1 con r2 sin signo (actualiza flags)
cmps r1, r2          // comparar r1 con r2 con signo (actualiza flags)
cmpu r1, 0           // comparar r1 con 0
```

`CMP` no modifica los registros; solo actualiza los flags en `rflags`. Los flags
se usan despues con instrucciones de salto condicional (`jmp.je`, `jmp.jlt`, etc.).

| Flag             | Condicion           | Uso tipico                |
| :--------------- | :------------------ | :------------------------ |
| **ZF** (Zero)    | `op1 == op2`        | `jmp.je/jne`              |
| **CF** (Carry)   | `op1 < op2` unsigned| `jmp.jcc/jcs` (jb/jae)   |
| **SF** (Sign)    | resultado negativo  | `jmp.jlt/jge`             |
| **OF** (Overflow)| overflow con signo  | `jmp.jvs/jvc`             |

```c
// Ejemplo: if (r1 == 0) { ... }
cmpu r1, 0
jmp.je es_cero

// Ejemplo: if (r1 < r2) (sin signo) { ... }
cmpu r1, r2
jmp.jcc r1_menor   // jcc = CF=0 = r1 < r2 sin signo

// Ejemplo: if (r1 < r2) (con signo) { ... }
cmps r1, r2
jmp.jlt r1_negativo // jlt = SF != OF = r1 < r2 con signo
```

| Instruccion                   | opcode1 | opcode2 | Tamano |
| :---------------------------- | :-----: | :-----: | :----: |
| `CMPU/CMPS reg1, reg2`        |  0x00   |  0x11   |   4 B  |
| `CMPU/CMPS reg1, inmmed`      |  0x00   |  0x12   |  10 B  |
| `CMPU/CMPS reg, [SIB]`        |  0x00   |  0x13   |   6 B  |

---

# Operaciones logicas a nivel de bits

Las operaciones de bits operan sobre cada bit del registro de forma independiente.
Son esenciales para mascaras de bits, flags de estado, y protocolo de datos.

## AND - Y logico bit a bit

```c
and r1, r2          // r1 = r1 & r2
```

El resultado tiene un `1` solo donde AMBOS operandos tienen `1`. Se usa para aislar bits
(mascara):

```c
mov r1, 0xFF        // 0b11111111
mov r2, 0x0F        // 0b00001111
and r1, r2          // r1 = 0x0F (solo los 4 bits bajos)

// Comprobar si el bit 3 esta activo:
and r1, 0x08        // r1 = 0 o 8; si ZF=0, el bit 3 estaba activo
```

| Instruccion   | opcode1 | opcode2 | Tamano |
| :------------ | :-----: | :-----: | :----: |
| `AND r1, r2`  |  0x00   |  0x17   |   4 B  |

## OR - O logico bit a bit

```c
or r1, r2           // r1 = r1 | r2
```

El resultado tiene un `1` donde AL MENOS UNO de los operandos tiene `1`. Se usa para
activar bits:

```c
mov r1, 0xF0        // 0b11110000
mov r2, 0x0F        // 0b00001111
or  r1, r2          // r1 = 0xFF (union de bits)
```

| Instruccion   | opcode1 | opcode2 | Tamano |
| :------------ | :-----: | :-----: | :----: |
| `OR r1, r2`   |  0x00   |  0x18   |   4 B  |

## XOR - O exclusivo bit a bit

```c
xor r1, r2          // r1 = r1 ^ r2
```

El resultado tiene un `1` donde los operandos son DIFERENTES (uno es 0 y el otro es 1).
Se usa para invertir bits y para poner registros a cero eficientemente:

```c
xor r3, r3          // r3 = 0 (poner registro a cero; mas compacto que mov r3, 0)

mov r1, 0xAA        // 1010 1010
mov r2, 0x55        // 0101 0101
xor r1, r2          // r1 = 0xFF (todos los bits distintos -> todos a 1)
```

| Instruccion   | opcode1 | opcode2 | Tamano |
| :------------ | :-----: | :-----: | :----: |
| `XOR r1, r2`  |  0x00   |  0x19   |   4 B  |

## NOT - Complemento a 1 (negacion bit a bit)

```c
not r1              // r1 = ~r1  (invierte todos los bits)
```

Invierte cada bit: los 0 se convierten en 1 y viceversa.

```c
mov r1, 0
not r1              // r1 = 0xFFFFFFFFFFFFFFFF (todos los bits a 1)

// Mascara inversa:
mov r3, 0x0F
not r3              // r3 = 0xFFFFFFFFFFFFFFF0 (la mascara inversa de los 4 bits bajos)
```

| Instruccion   | opcode1 | opcode2 | Tamano |
| :------------ | :-----: | :-----: | :----: |
| `NOT r1`      |  0x00   |  0x1A   |   4 B  |

---

# Desplazamientos de bits

Los desplazamientos mueven los bits hacia la izquierda o la derecha. Son equivalentes a
multiplicar o dividir por potencias de 2, y a menudo mas rapidos.

## SHL - Desplazamiento logico a la izquierda (shift left)

```c
shl r1, r2          // r1 = r1 << (r2 & 63)
```

Multiplica por 2^n. Mueve bits hacia la izquierda, rellena con ceros por la derecha.
El ultimo bit desplazado fuera va a CF.

```c
mov r1, 1
mov r2, 3
shl r1, r2          // r1 = 8 (1 * 2^3 = 8)

mov r1, 0x01
mov r2, 7
shl r1, r2          // r1 = 0x80
```

| Instruccion   | opcode1 | opcode2 | Tamano |
| :------------ | :-----: | :-----: | :----: |
| `SHL r1, r2`  |  0x00   |  0x1B   |   4 B  |

## SHR - Desplazamiento logico a la derecha (shift right, sin signo)

```c
shr r1, r2          // r1 = r1 >> (r2 & 63)  (rellena con 0)
```

Divide por 2^n sin signo. Mueve bits hacia la derecha, rellena con ceros por la izquierda.

```c
mov r1, 0xFF00
mov r2, 8
shr r1, r2          // r1 = 0xFF  (desplazar 8 bits a la derecha)
```

| Instruccion   | opcode1 | opcode2 | Tamano |
| :------------ | :-----: | :-----: | :----: |
| `SHR r1, r2`  |  0x00   |  0x1C   |   4 B  |

## SAR - Desplazamiento aritmetico a la derecha (shift arithmetic right, con signo)

```c
sar r1, r2          // r1 = r1 >> (r2 & 63)  (rellena con bit de signo)
```

Divide por 2^n con signo. Propaga el bit de signo (bit 63) hacia la izquierda:
- Si el bit 63 era 1 (negativo), se rellena con 1s.
- Si el bit 63 era 0 (positivo), se rellena con 0s (igual que SHR).

```c
// Numero positivo: igual que SHR
mov r1, 0x0F00
mov r2, 4
sar r1, r2          // r1 = 0x00F0 (bit signo era 0, rellena con 0s)

// Numero negativo: propaga el signo
mov r1, 0xFF00000000000000   // bit 63 = 1 (negativo como int64)
mov r2, 55
sar r1, r2          // r1 = 0xFFFFFFFFFFFFFFFE (rellena con 1s, preserva el signo)

// Comparacion SHR vs SAR con el mismo valor negativo:
// shr: r1 = 0x000000000000001F (trata como unsigned, rellena con 0s)
// sar: r1 = 0xFFFFFFFFFFFFFFFE (trata como signed, rellena con 1s)
```

| Instruccion   | opcode1 | opcode2 | Tamano |
| :------------ | :-----: | :-----: | :----: |
| `SAR r1, r2`  |  0x00   |  0x1D   |   4 B  |

---

# Codificacion binaria comun (formato extendido)

Todas las instrucciones logicas y de desplazamiento usan el formato extendido (prefijo `0x00`):

```
+--------+--------+--------------------+--------------------+
| 0x00   | opcode | mode<<6 | 0 | 0000 |  reg2<<4 | reg1   |
+--------+--------+--------------------+--------------------+
  byte 0   byte 1       byte 2               byte 3

mode (2 bits, 7-6): 0=byte, 1=word, 2=dword, 3=qword
reg1 (4 bits, 3-0): registro destino
reg2 (4 bits, 7-4): registro fuente
Las instrucciones NOT usan reg2=0 (solo un operando)
```

| Instruccion | opcode2 | Tipo    |
| :---------- | :-----: | :------ |
| `and`       |  0x17   | binaria |
| `or`        |  0x18   | binaria |
| `xor`       |  0x19   | binaria |
| `not`       |  0x1A   | unaria  |
| `shl`       |  0x1B   | binaria |
| `shr`       |  0x1C   | binaria |
| `sar`       |  0x1D   | binaria |

---

Ver tambien: [[JMP]] (saltos condicionales basados en flags), [[REGISTROS]] (flags de rflags), [[MOV, MOVH, MOVC, MOCH]]
