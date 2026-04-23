
|     Operacion      | **Sin signo** | **Con signo** | No aplica |
| :----------------: | :-----------: | :-----------: | :-------: |
|      **Suma**      |    `ADDU`     |    `ADDS`     |           |
|     **Resta**      |    `SUBU`     |    `SUBS`     |           |
| **Multiplicacion** |    `MULU`     |    `MULS`     |           |
|    **Division**    |    `DIVU`     |    `DIVS`     |           |
|  **Comparacion**   |    `CMPU`     |    `CMPS`     |           |
|    Incrementar     |               |               |    INC    |
|    Decrementar     |               |               |    DEC    |
|      AND bit       |               |               |    AND    |
|       OR bit       |               |               |     OR    |
|      XOR bit       |               |               |    XOR    |
|      NOT bit       |               |               |    NOT    |
| Despl. izquierda   |               |               |    SHL    |
|  Despl. logico der |               |               |    SHR    |
| Despl. arit. der   |               |               |    SAR    |

# Nomenclatura
- `reg1`, `reg2`: se usa 6 bits para expresar cada registro general, revise [[REGISTROS#Registros generales|Registros generales y su codificacion.]] En caso de que la instruccion use dos registros, como `ADD`, `CMP` y etc, el modo (2 bits) puede indicarse en otro lado y los registros se pueden expresar ambos en un byte usando 4 bits para cada uno.

# Modos de direccionamiento
Actualmente solo hay dos modos de direccionamiento, un modo basico de reg-reg
## Modo Basico

El modo basico agrupa el direccionamiento de registro a registro, de memoria a registro y de registro a memoria, sirve para operaciones basicas e intercambio.
## Modo SIB (Scale + Index + Base)
Este modo de acceso sirve principalmente para el acceso a arrays y estructuras indexadas, equivalente al modo SIB de Intel x86.

La direccion efectiva se calcula como: `EA = base + index * scale`

donde `scale` puede ser ×1, ×2, ×4 o ×8.

### Codificacion SIB (6 bytes)

```
+--------+--------+----------------------+-------------------+-------------------+--------+
| 0x00   | opcode |  ctrl                |  regs             |  index            |  pad   |
+--------+--------+----------------------+-------------------+-------------------+--------+
  byte0    byte1    byte2                  byte3               byte4               byte5
```

**byte2 (ctrl):**
```
bits 7-6 = mode      (0=byte, 1=word, 2=dword, 3=qword)
bit  5   = signed    (0=unsigned, 1=signed)
bit  4   = dir       (0=reg<-mem, 1=mem<-reg)
bits 3-2 = scale     (0=×1, 1=×2, 2=×4, 3=×8)
bit  1   = has_index (0=solo base [reg], 1=base+index*scale [base+idx*sc])
bit  0   = 0         (reservado)
```

**byte3 (regs):**
```
bits 7-4 = dst_reg  (registro destino/fuente segun dir)
bits 3-0 = base_reg (registro base)
```

**byte4 (index):**
```
bits 7-4 = index_reg (registro indice)
bits 3-0 = 0000      (reservado)
```

**byte5:** `0x00` (padding)

### Sintaxis

```asm
; dir=0: reg recibe el valor de mem[base + index*scale]
adds r1, [r14 + r13*8]    ; r1 = r1 + mem64[r14 + r13*8]

; dir=1: mem[base + index*scale] recibe el resultado de la operacion
adds [r14 + r12*8], r6    ; mem64[r14 + r12*8] = mem64[r14+r12*8] + r6

; scale=1 (sin multiplicador explicito)
adds r4, [r14 + r13]      ; r4 = r4 + mem64[r14 + r13*1]

; solo base, sin indice (has_index=0 en ctrl, byte4/index ignorado)
adds r4, [r14]            ; r4 = r4 + mem64[r14]
mov  r1, [r14]            ; r1 = mem64[r14]
```

> Cuando se escribe `[reg]` sin indice, el ensamblador emite `has_index=0` en el bit 1 del ctrl. En ejecucion la VM calcula `EA = base` directamente, sin leer el registro de indice.

# Peculiaridades de las instrucciones aritmeticas y logicas
La mayoria de instrucciones de este tipo que operan con memoria, usan solo `5bytes * 8 = 40bits` para acceder a memoria, por tanto `0x000000000000 - 0xFFFFFFFFFFFFFF` es su rango de operacion (Si el Registro de paginado (`RP`) esta configurado en 0).
Sin embargo, estas pueden usar un registro de `64bits` (`RP`) para acceder a otras partes de la memoria si es que esto fuera realmente necesario.

----
# Peculiaridades de las instrucciones de acceso a memoria
Algunas instrucciones como los `mov` y las instrucciones aritmetico-logicas permiten el acceso a memoria para cambiar u obtener valores, vease por ejemplo:

```c
mov r0, 0x123456789abcdef  
adds [r0w], 0x1000
```
Aqui lo que esta ocurriendo es que estamos accediendo a la direccion virtual `0x123456789abcdef`
y estamos sumando un WORD (valor de 16 bits) el cual es `0x1000`, esto es equivalente a hacer en C:
```c
uint8_t* mem[...] = { 0 }; // supongamos que esto es memoria.

// mov r0, 0x123456789abcdef
int64_t r0 = mem + 0x123456789abcdef;

// adds [r0w], 0x1000
r0 = *((int32_t*)r0) + 0x1000;
```
Por tanto el tamanio del registro dentro de los `brackets` (corchetes `[]`) indica el tamanio de la operacion. 

----

# INC
Permite incrementar un registro en 1

| Instruccion | opcode1 | byte (relleno o extension o registro) | total bytes |
| :---------: | :-----: | :-----------------------------------: | :---------: |
|     INC     |   0x4   |             `0b00` `reg`              |      2      |
# DEC
Permite decrementar un registro en 1

| Instruccion | opcode1 | byte (relleno o extension o registro) | total bytes |
| :---------: | :-----: | :-----------------------------------: | :---------: |
|     DEC     |   0x4   |             `0b01` `reg`              |      2      |

# ADD - Con signo (S) y sin signo (U)

| Instruccion                          | opcode1 | opcode2 |           1byte           |       1byte       | 1byte | 1byte | 1byte |   4byte    | total bytes |
| :----------------------------------- | :-----: | :-----: | :-----------------------: | :---------------: | :---: | :---: | :---: | :--------: | :---------: |
| `ADDU reg1, reg2`                    |   0x0   |   0x5   |  0b`mode`0d0000 d = 0     | `reg2` `reg1`     |       |       |       |            |      4      |
| `ADDS reg1, reg2`                    |   0x0   |   0x5   |  0b`mode`1d0000 d = 0     | `reg2` `reg1`     |       |       |       |            |      4      |
| `ADDU reg1, inmmed`                  |   0x0   |   0x6   | 0b`mode`0d`reg1` d = 0    |       0xFF        | 0xFF  | 0xFF  | 0xFF  | 0xFFFFFFFF |     10      |
| `ADDU [reg], inmmed`                 |   0x0   |   0x6   | 0b`mode`0d`reg1` d = 1    |       0xFF        | 0xFF  | 0xFF  | 0xFF  | 0xFFFFFFFF |     10      |
| `ADDS reg1, inmmed`                  |   0x0   |   0x6   | 0b`mode`1d`reg1` d = 0    |       0xFF        | 0xFF  | 0xFF  | 0xFF  | 0xFFFFFFFF |     10      |
| `ADDS [reg], inmmed`                 |   0x0   |   0x6   | 0b`mode`1d`reg1` d = 1    |       0xFF        | 0xFF  | 0xFF  | 0xFF  | 0xFFFFFFFF |     10      |
| `ADDU reg, [base+idx*sc]`            |   0x0   |   0x7   | 0b`mode`0`d``sc` d = 0    | `reg` `base`      | `idx` | 0x00  |       |            |      6      |
| `ADDU [base+idx*sc], reg`            |   0x0   |   0x7   | 0b`mode`0`d``sc` d = 1    | `reg` `base`      | `idx` | 0x00  |       |            |      6      |
| `ADDS reg, [base+idx*sc]`            |   0x0   |   0x7   | 0b`mode`1`d``sc` d = 0    | `reg` `base`      | `idx` | 0x00  |       |            |      6      |
| `ADDS [base+idx*sc], reg`            |   0x0   |   0x7   | 0b`mode`1`d``sc` d = 1    | `reg` `base`      | `idx` | 0x00  |       |            |      6      |

# SUB - Con signo (S) y sin signo (U)
| Instruccion                        | opcode1 | opcode2 |            1byte            |     1byte     | 1byte | 1byte | 1byte |   4byte    | total bytes |
| :--------------------------------- | :-----: | :-----: | :-------------------------: | :-----------: | :---: | :---: | :---: | :--------: | :---------: |
| `SUBU reg1, reg2`                  |   0x0   |   0x8   |  0b`mode`0d0000 d = 0       | `reg2` `reg1` |       |       |       |            |      4      |
| `SUBS reg1, reg2`                  |   0x0   |   0x8   |  0b`mode`1d0000 d = 0       | `reg2` `reg1` |       |       |       |            |      4      |
| `SUBU reg1, inmmed`                |   0x0   |   0x9   | 0b`mode`0d`reg1` d = 0      |     0xFF      | 0xFF  | 0xFF  | 0xFF  | 0xFFFFFFFF |     10      |
| `SUBU [reg], inmmed`               |   0x0   |   0x9   | 0b`mode`0d`reg1` d = 1      |     0xFF      | 0xFF  | 0xFF  | 0xFF  | 0xFFFFFFFF |     10      |
| `SUBS reg1, inmmed`                |   0x0   |   0x9   | 0b`mode`1d`reg1` d = 0      |     0xFF      | 0xFF  | 0xFF  | 0xFF  | 0xFFFFFFFF |     10      |
| `SUBS [reg], inmmed`               |   0x0   |   0x9   | 0b`mode`1d`reg1` d = 1      |     0xFF      | 0xFF  | 0xFF  | 0xFF  | 0xFFFFFFFF |     10      |
| `SUBU reg, [base+idx*sc]`          |   0x0   |   0xA   | 0b`mode`0`d``sc` d = 0      | `reg` `base`  | `idx` | 0x00  |       |            |      6      |
| `SUBU [base+idx*sc], reg`          |   0x0   |   0xA   | 0b`mode`0`d``sc` d = 1      | `reg` `base`  | `idx` | 0x00  |       |            |      6      |
| `SUBS reg, [base+idx*sc]`          |   0x0   |   0xA   | 0b`mode`1`d``sc` d = 0      | `reg` `base`  | `idx` | 0x00  |       |            |      6      |
| `SUBS [base+idx*sc], reg`          |   0x0   |   0xA   | 0b`mode`1`d``sc` d = 1      | `reg` `base`  | `idx` | 0x00  |       |            |      6      |

# MUL - Con signo (S) y sin signo (U)
| Instruccion                        | opcode1 | opcode2 |            1byte            |     1byte     | 1byte | 1byte |   4byte    | total bytes |
| :--------------------------------- | :-----: | :-----: | :-------------------------: | :-----------: | :---: | :---: | :--------: | :---------: |
| `MULU reg1, reg2`                  |   0x0   |   0xB   |  0b`mode`0d0000 d = 0       | `reg2` `reg1` |       |       |            |      4      |
| `MULS reg1, reg2`                  |   0x0   |   0xB   |  0b`mode`1d0000 d = 0       | `reg2` `reg1` |       |       |            |      4      |
| `MULU reg, inmmed`                 |   0x0   |   0xC   | 0b`mode`0d`reg1` d = 0      |     0xFF      | 0xFF  | 0xFF  | 0xFFFFFFFF |     10      |
| `MULU [reg], inmmed`               |   0x0   |   0xC   | 0b`mode`0d`reg1` d = 1      |     0xFF      | 0xFF  | 0xFF  | 0xFFFFFFFF |     10      |
| `MULS reg, inmmed`                 |   0x0   |   0xC   | 0b`mode`1d`reg1` d = 0      |     0xFF      | 0xFF  | 0xFF  | 0xFFFFFFFF |     10      |
| `MULS [reg], inmmed`               |   0x0   |   0xC   | 0b`mode`1d`reg1` d = 1      |     0xFF      | 0xFF  | 0xFF  | 0xFFFFFFFF |     10      |
| `MULU reg, [base+idx*sc]`          |   0x0   |   0xD   | 0b`mode`0`d``sc` d = 0      | `reg` `base`  | `idx` | 0x00  |            |      6      |
| `MULU [base+idx*sc], reg`          |   0x0   |   0xD   | 0b`mode`0`d``sc` d = 1      | `reg` `base`  | `idx` | 0x00  |            |      6      |
| `MULS reg, [base+idx*sc]`          |   0x0   |   0xD   | 0b`mode`1`d``sc` d = 0      | `reg` `base`  | `idx` | 0x00  |            |      6      |
| `MULS [base+idx*sc], reg`          |   0x0   |   0xD   | 0b`mode`1`d``sc` d = 1      | `reg` `base`  | `idx` | 0x00  |            |      6      |

# DIV - Con signo (S) y sin signo (U)

| Instruccion                        | opcode1 | opcode2 |            1byte            |     1byte     | 1byte | 1byte | 1byte |   4byte    | total bytes |
| :--------------------------------- | :-----: | :-----: | :-------------------------: | :-----------: | :---: | :---: | :---: | :--------: | :---------: |
| `DIVU reg1, reg2`                  |   0x0   |   0xE   |  0b`mode`0d0000 d = 0       | `reg2` `reg1` |       |       |       |            |      4      |
| `DIVS reg1, reg2`                  |   0x0   |   0xE   |  0b`mode`1d0000 d = 0       | `reg2` `reg1` |       |       |       |            |      4      |
| `DIVU reg, inmmed`                 |   0x0   |   0xF   | 0b`mode`0d`reg1` d = 0      |     0xFF      | 0xFF  | 0xFF  | 0xFF  | 0xFFFFFFFF |     10      |
| `DIVU [reg], inmmed`               |   0x0   |   0xF   | 0b`mode`0d`reg1` d = 1      |     0xFF      | 0xFF  | 0xFF  | 0xFF  | 0xFFFFFFFF |     10      |
| `DIVS reg, inmmed`                 |   0x0   |   0xF   | 0b`mode`1d`reg1` d = 0      |     0xFF      | 0xFF  | 0xFF  | 0xFF  | 0xFFFFFFFF |     10      |
| `DIVS [reg], inmmed`               |   0x0   |   0xF   | 0b`mode`1d`reg1` d = 1      |     0xFF      | 0xFF  | 0xFF  | 0xFF  | 0xFFFFFFFF |     10      |
| `DIVU reg, [base+idx*sc]`          |   0x0   |  0x10   | 0b`mode`0`d``sc` d = 0      | `reg` `base`  | `idx` | 0x00  |       |            |      6      |
| `DIVU [base+idx*sc], reg`          |   0x0   |  0x10   | 0b`mode`0`d``sc` d = 1      | `reg` `base`  | `idx` | 0x00  |       |            |      6      |
| `DIVS reg, [base+idx*sc]`          |   0x0   |  0x10   | 0b`mode`1`d``sc` d = 0      | `reg` `base`  | `idx` | 0x00  |       |            |      6      |
| `DIVS [base+idx*sc], reg`          |   0x0   |  0x10   | 0b`mode`1`d``sc` d = 1      | `reg` `base`  | `idx` | 0x00  |       |            |      6      |

# CMP - Con signo (S) y sin signo (U)
| Flag              | **Condicion**          | **Uso tipico**       |
| ----------------- | ---------------------- | -------------------- |
| **ZF** (Zero)     | `op1 == op2`           | `JE/JZ`, `JNE/JNZ`   |
| **CF** (Carry)    | `op1 < op2` (unsigned) | `JB/JC`, `JAE/JNC`   |
| **SF** (Sign)     | `op1 < op2` (signed)   | `JL/JNGE`, `JGE/JNL` |
| **OF** (Overflow) | Overflow signed        | `JO`, `JNO`          |

| Instruccion                        | opcode1 | opcode2 |            1byte            |     1byte     | 1byte | 1byte | 1byte |   4byte    | total bytes |
| :--------------------------------- | :-----: | :-----: | :-------------------------: | :-----------: | :---: | :---: | :---: | :--------: | :---------: |
| `CMPU reg1, reg2`                  |   0x0   |  0x11   |  0b`mode`0d0000 d = 0       | `reg2` `reg1` |       |       |       |            |      4      |
| `CMPS reg1, reg2`                  |   0x0   |  0x11   |  0b`mode`1d0000 d = 0       | `reg2` `reg1` |       |       |       |            |      4      |
| `CMPU reg, inmmed`                 |   0x0   |  0x12   | 0b`mode`0d`reg1` d = 0      |     0xFF      | 0xFF  | 0xFF  | 0xFF  | 0xFFFFFFFF |     10      |
| `CMPU [reg], inmmed`               |   0x0   |  0x12   | 0b`mode`0d`reg1` d = 1      |     0xFF      | 0xFF  | 0xFF  | 0xFF  | 0xFFFFFFFF |     10      |
| `CMPS reg, inmmed`                 |   0x0   |  0x12   | 0b`mode`1d`reg1` d = 0      |     0xFF      | 0xFF  | 0xFF  | 0xFF  | 0xFFFFFFFF |     10      |
| `CMPS [reg], inmmed`               |   0x0   |  0x12   | 0b`mode`1d`reg1` d = 1      |     0xFF      | 0xFF  | 0xFF  | 0xFF  | 0xFFFFFFFF |     10      |
| `CMPU reg, [base+idx*sc]`          |   0x0   |  0x13   | 0b`mode`0`d``sc` d = 0      | `reg` `base`  | `idx` | 0x00  |       |            |      6      |
| `CMPU [base+idx*sc], reg`          |   0x0   |  0x13   | 0b`mode`0`d``sc` d = 1      | `reg` `base`  | `idx` | 0x00  |       |            |      6      |
| `CMPS reg, [base+idx*sc]`          |   0x0   |  0x13   | 0b`mode`1`d``sc` d = 0      | `reg` `base`  | `idx` | 0x00  |       |            |      6      |
| `CMPS [base+idx*sc], reg`          |   0x0   |  0x13   | 0b`mode`1`d``sc` d = 1      | `reg` `base`  | `idx` | 0x00  |       |            |      6      |

## Saltos por IGUALDAD (ZF)

| Instruccion | **Condicion** | **Uso**           | **Assembly**           |
| ----------- | ------------- | ----------------- | ---------------------- |
| `JE/JZ`     | `ZF=1`        | Igual/Cero        | `cmp r1,r2; je ok`     |
| `JNE/JNZ`   | `ZF=0`        | Diferente/No cero | `cmp r1,r2; jne error` |

## Saltos UNSIGNED (CF)

| Instruccion | **Condicion**  | **Uso**                | **Assembly**            |
| ----------- | -------------- | ---------------------- | ----------------------- |
| `JB/JC`     | `CF=1`         | Menor (unsigned)       | `cmpu r1,r2; jb menor`  |
| `JAE/JNC`   | `CF=0`         | Mayor/Igual (unsigned) | `cmpu r1,r2; jae mayor` |
| `JBE/JNA`   | CF=1 || ZF=1   | Menor/Igual (unsigned) | `cmpu r1,r2; jbe mayor` |
| `JA/JNBE`   | `CF=0 && ZF=0` | Mayor (unsigned)       | `cmpu r1,r2; ja loop`   |

## Saltos SIGNED (SF, OF)
| Instruccion | **Condicion**      | **Uso**              | **Assembly**          |
| ----------- | ------------------ | -------------------- | --------------------- |
| `JL/JNGE`   | `SF != OF`         | Menor (signed)       | `cmps r1,r2; jl neg`  |
| `JGE/JNL`   | `SF == OF`         | Mayor/Igual (signed) | `cmps r1,r2; jge pos` |
| `JLE/JNG`   | CF=1 || SF!=OF     | Menor/Igual (signed) | `cmps r1,r2; jle pos` |
| `JG/JNLE`   | `ZF=0 && SF == OF` | Mayor (signed)       | `cmps r1,r2; jg max`  |

## Saltos especiales (OF)
| Instruccion | **Condicion** | **Uso**         | **Assembly**           |
| ----------- | ------------- | --------------- | ---------------------- |
| `JO`        | `OF=1`        | Overflow signed | `cmps r1,r2; jo error` |
| `JNO`       | `OF=0`        | No overflow     | `cmps r1,r2; jno ok`   |

## Salto incondicional
| Instruccion | **Condicion** | **Assembly** |
| ----------- | ------------- | ------------ |
| [[JMP]]     | Siempre       | `jmp loop`   |

---

# Operaciones logicas a nivel de bits

## AND - Y logico bit a bit

```asm
and r1, r2          ; r1 = r1 & r2
```

| Instruccion   | opcode1 | opcode2 | byte3                  | byte4          | total bytes |
| :------------ | :-----: | :-----: | :--------------------: | :------------: | :---------: |
| `AND r1, r2`  |  0x00   |  0x17   | 0b`mode`0d0000 d=0     | `r2<<4 | r1`   |      4      |

No tiene variante con signo (los bits no tienen signo). Actualiza ZF y SF. CF y OF se ponen a 0.

```asm
mov r1, 0xFF
mov r2, 0x0F
and r1, r2          ; r1 = 0x0F (mascara de 4 bits bajos)

mov r3, 0xAA        ; 1010 1010
mov r4, 0x55        ; 0101 0101
and r3, r4          ; r3 = 0x00 (bits no solapan)
```

## OR - O logico bit a bit

```asm
or r1, r2           ; r1 = r1 | r2
```

| Instruccion   | opcode1 | opcode2 | byte3                  | byte4          | total bytes |
| :------------ | :-----: | :-----: | :--------------------: | :------------: | :---------: |
| `OR r1, r2`   |  0x00   |  0x18   | 0b`mode`0d0000 d=0     | `r2<<4 | r1`   |      4      |

```asm
mov r1, 0xF0
mov r2, 0x0F
or  r1, r2          ; r1 = 0xFF (union de bits)

; Activar el bit 7:
or  r1, 0x80        ; instruccion pendiente: or con inmediato (no implementado aun)
```

## XOR - O exclusivo bit a bit

```asm
xor r1, r2          ; r1 = r1 ^ r2
```

| Instruccion   | opcode1 | opcode2 | byte3                  | byte4          | total bytes |
| :------------ | :-----: | :-----: | :--------------------: | :------------: | :---------: |
| `XOR r1, r2`  |  0x00   |  0x19   | 0b`mode`0d0000 d=0     | `r2<<4 | r1`   |      4      |

```asm
mov r1, 0xAA
mov r2, 0x55
xor r1, r2          ; r1 = 0xFF (todos los bits distintos -> todos a 1)

xor r3, r3          ; r3 = 0x00 (poner registro a cero, mas rapido que mov r3, 0)
```

## NOT - Complemento a 1 (negacion bit a bit)

```asm
not r1              ; r1 = ~r1
```

| Instruccion   | opcode1 | opcode2 | byte3          | byte4    | total bytes |
| :------------ | :-----: | :-----: | :------------: | :------: | :---------: |
| `NOT r1`      |  0x00   |  0x1A   | 0b`mode`000000 | `0 | r1` |      4      |

NOT invierte todos los bits del registro. Actualiza ZF y SF. CF y OF se ponen a 0.

```asm
mov r1, 0
not r1              ; r1 = 0xFFFFFFFFFFFFFFFF (todos los bits a 1)

mov r2, 0xFFFFFFFFFFFFFFFF
not r2              ; r2 = 0x00 (todos los bits a 0)

; Mascara inversa:
mov r3, 0x0F
not r3              ; r3 = 0xFFFFFFFFFFFFFFF0
```

---

# Desplazamientos de bits

## SHL - Desplazamiento logico a la izquierda

Multiplica por 2^n. Mueve bits hacia la izquierda, rellena con ceros por la derecha.

```asm
shl r1, r2          ; r1 = r1 << (r2 & 63)
```

| Instruccion   | opcode1 | opcode2 | byte3                  | byte4          | total bytes |
| :------------ | :-----: | :-----: | :--------------------: | :------------: | :---------: |
| `SHL r1, r2`  |  0x00   |  0x1B   | 0b`mode`0d0000 d=0     | `r2<<4 | r1`   |      4      |

El segundo operando se enmascara con `& (sizeof(T)*8 - 1)` para evitar desplazamientos fuera de rango.
CF recibe el ultimo bit desplazado fuera.

```asm
mov r1, 0x01
mov r2, 7
shl r1, r2          ; r1 = 0x80 (1 desplazado 7 posiciones a la izquierda)

mov r1, 1
mov r2, 3
shl r1, r2          ; r1 = 8 (1 * 2^3)
```

## SHR - Desplazamiento logico a la derecha

Divide por 2^n sin signo. Mueve bits hacia la derecha, rellena con ceros por la izquierda.

```asm
shr r1, r2          ; r1 = r1 >> (r2 & 63)  (relleno con 0)
```

| Instruccion   | opcode1 | opcode2 | byte3                  | byte4          | total bytes |
| :------------ | :-----: | :-----: | :--------------------: | :------------: | :---------: |
| `SHR r1, r2`  |  0x00   |  0x1C   | 0b`mode`0d0000 d=0     | `r2<<4 | r1`   |      4      |

```asm
mov r1, 0x80
mov r2, 7
shr r1, r2          ; r1 = 0x01

mov r1, 0xFF00
mov r2, 8
shr r1, r2          ; r1 = 0xFF
```

## SAR - Desplazamiento aritmetico a la derecha

Divide por 2^n con signo. Propaga el bit de signo (bit 63) hacia la izquierda.

```asm
sar r1, r2          ; r1 = r1 >> (r2 & 63)  (relleno con bit de signo)
```

| Instruccion   | opcode1 | opcode2 | byte3                  | byte4          | total bytes |
| :------------ | :-----: | :-----: | :--------------------: | :------------: | :---------: |
| `SAR r1, r2`  |  0x00   |  0x1D   | 0b`mode`0d0000 d=0     | `r2<<4 | r1`   |      4      |

A diferencia de SHR, SAR mantiene el signo del valor:
- Si el bit 63 era 1 (negativo), se rellena con 1s.
- Si el bit 63 era 0 (positivo), se rellena con 0s (igual que SHR).

```asm
; Numero positivo: igual que SHR
mov r1, 0x0F00
mov r2, 4
sar r1, r2          ; r1 = 0x00F0 (igual que SHR, bit signo era 0)

; Numero negativo: propaga el signo
mov r1, 0xFF00000000000000   ; int64 negativo (bit 63 = 1)
mov r2, 55
sar r1, r2          ; r1 = 0xFFFFFFFFFFFFFFFE (= -2 con signo)
                    ; (0xFF00000000000000 como int64 = -72057594037927936; >> 55 = -2)
```

### Diferencia SHR vs SAR con valores negativos

```asm
mov r1, 0xFF00000000000000  ; mismo valor para ambas
mov r2, 0xFF00000000000000
mov r3, 55
mov r4, 55

shr r1, r3          ; r1 = 0x000000000000001F  (rellena con 0s, trata como unsigned)
sar r2, r4          ; r2 = 0xFFFFFFFFFFFFFFFE  (rellena con 1s, propaga signo)
```

---

# Codificacion binaria comun

Todas las instrucciones logicas y de desplazamiento usan el formato de la tabla extendida (prefijo `0x00`):

```
+--------+--------+--------------------+--------------------+
| 0x00   | opcode | mode<<6 | 0 | 0000 |  reg2<<4 | reg1   |
+--------+--------+--------------------+--------------------+
  byte 0   byte 1       byte 2               byte 3
```

- `mode` (2 bits, 7-6): 0=byte, 1=word, 2=dword, 3=qword
- `reg1` (4 bits, 3-0): registro destino
- `reg2` (4 bits, 7-4): registro fuente
- Las instrucciones NOT usan reg2=0 (solo un operando)

| Instruccion | opcode2 | Tipo     |
| :---------- | :-----: | :------- |
| `and`       |  0x17   | binaria  |
| `or`        |  0x18   | binaria  |
| `xor`       |  0x19   | binaria  |
| `not`       |  0x1A   | unaria   |
| `shl`       |  0x1B   | binaria  |
| `shr`       |  0x1C   | binaria  |
| `sar`       |  0x1D   | binaria  |
| `movc/movch mem` | 0x1E | 3-op (ver [[MOV, MOVH, MOVC, MOCH]]) |
| `movc reg`  |  0x1F   | 3-op    |

---

## Resumen global de opcodes extendidos (prefijo 0x00)

| opcode2 | Instruccion         | Variante        |
| :-----: | :------------------ | :-------------- |
|  0x05   | ADD reg,reg         | REG             |
|  0x06   | ADD reg/mem,imm     | INMED           |
|  0x07   | ADD reg/mem,[SIB]   | SIB             |
|  0x08   | SUB reg,reg         | REG             |
|  0x09   | SUB reg/mem,imm     | INMED           |
|  0x0A   | SUB reg/mem,[SIB]   | SIB             |
|  0x0B   | MUL reg,reg         | REG             |
|  0x0C   | MUL reg/mem,imm     | INMED           |
|  0x0D   | MUL reg/mem,[SIB]   | SIB             |
|  0x0E   | DIV reg,reg         | REG             |
|  0x0F   | DIV reg/mem,imm     | INMED           |
|  0x10   | DIV reg/mem,[SIB]   | SIB             |
|  0x11   | CMP reg,reg         | REG             |
|  0x12   | CMP reg/mem,imm     | INMED           |
|  0x13   | CMP reg/mem,[SIB]   | SIB             |
|  0x14   | MOV reg,reg         | REG             |
|  0x15   | MOV reg/mem,imm     | INMED           |
|  0x16   | MOV/MOVH SIB        | SIB (s=1->host)  |
|  0x17   | AND reg,reg         | REG             |
|  0x18   | OR  reg,reg         | REG             |
|  0x19   | XOR reg,reg         | REG             |
|  0x1A   | NOT reg             | REG (unaria)    |
|  0x1B   | SHL reg,reg         | REG             |
|  0x1C   | SHR reg,reg         | REG             |
|  0x1D   | SAR reg,reg         | REG             |
|  0x1E   | MOVC/MOVCH mem      | MEM (3 operandos) |
|  0x1F   | MOVC reg,reg        | REG (3 operandos) |
