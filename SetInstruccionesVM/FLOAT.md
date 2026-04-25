# Instrucciones de Punto Flotante (FLOAT)

Los numeros enteros (R0-R15) no pueden representar fracciones como 3.14 o 2.718. Para
calcular con numeros decimales, la VM tiene un conjunto separado de registros y
instrucciones de **punto flotante** (floating point), que usan el estandar **IEEE 754**
de 64 bits (tambien llamado `double` en C/Java o `float64` en otros lenguajes).

Analogia: los registros R0-R15 son como contadores de billetes enteros. Los registros
flotantes (f0-f15) son como balanzas de precision que pueden medir 3.14159 kilogramos.

El estandar IEEE 754 de 64 bits codifica un numero decimal como 64 bits repartidos en:
- 1 bit de signo (positivo/negativo)
- 11 bits de exponente
- 52 bits de mantisa (la parte decimal)

Por eso, para cargar el numero `1.0` en un registro flotante, se escribe su representacion
en bits: `0x3FF0000000000000`. La tabla al final de esta seccion muestra las constantes mas
utiles.

Implementacion: `src/runtime/exec_instruction_alu.cpp` (decodificacion en `decode_table.cpp`)

---

## Registros ZMM

Los registros ZMM son **independientes** de los registros de proposito general (R0-R15).
Cada registro ZMM almacena un double de 64 bits (IEEE 754). Se nombran `f0` a `f15` en
el ensamblador. Hay 16 registros flotantes: `f0`, `f1`, ..., `f15`.

Los datos no pasan automaticamente de R0-R15 a f0-f15. Para eso existen las instrucciones
de conversion (`fcvt`) y carga directa (`fmowi`, `fload`).

---

## Tabla de instrucciones

| instruccion | opcode0 | opcode1 | modo  |  tamano  | descripcion                       |      |     |
| :---------: | :-----: | :-----: | :---: | :------: | :-------------------------------- | ---- | --- |
|   `fmowi`   |  0x00   |  0xFA   | INMED | 11 bytes | cargar inmediato u64 en ZMM       |      |     |
|   `fmov`    |  0x00   |  0xF0   |  REG  | 4 bytes  | copiar registro ZMM a ZMM         |      |     |
|   `fadd`    |  0x00   |  0xF1   |  REG  | 4 bytes  | suma: fDst += fSrc                |      |     |
|  `fadd.pd`  |  0x00   |  0xF1   |  REG  | 4 bytes  | suma (alias vectorial)            |      |     |
|  `fadd.ps`  |  0x00   |  0xF1   |  REG  | 4 bytes  | suma (alias vectorial)            |      |     |
|   `fsub`    |  0x00   |  0xF2   |  REG  | 4 bytes  | resta: fDst -= fSrc               |      |     |
|  `fsub.pd`  |  0x00   |  0xF2   |  REG  | 4 bytes  | resta (alias vectorial)           |      |     |
|  `fsub.ps`  |  0x00   |  0xF2   |  REG  | 4 bytes  | resta (alias vectorial)           |      |     |
|   `fmul`    |  0x00   |  0xF3   |  REG  | 4 bytes  | multiplicacion: fDst *= fSrc      |      |     |
|  `fmul.pd`  |  0x00   |  0xF3   |  REG  | 4 bytes  | multiplicacion (alias vectorial)  |      |     |
|  `fmul.ps`  |  0x00   |  0xF3   |  REG  | 4 bytes  | multiplicacion (alias vectorial)  |      |     |
|   `fdiv`    |  0x00   |  0xF4   |  REG  | 4 bytes  | division: fDst /= fSrc            |      |     |
|  `fdiv.pd`  |  0x00   |  0xF4   |  REG  | 4 bytes  | division (alias vectorial)        |      |     |
|  `fdiv.ps`  |  0x00   |  0xF4   |  REG  | 4 bytes  | division (alias vectorial)        |      |     |
|   `fcmp`    |  0x00   |  0xF5   |  REG  | 4 bytes  | comparacion: flags <- fDst - fSrc |      |     |
|  `fcmp.pd`  |  0x00   |  0xF5   |  REG  | 4 bytes  | comparacion (alias vectorial)     |      |     |
|  `fcmp.ps`  |  0x00   |  0xF5   |  REG  | 4 bytes  | comparacion (alias vectorial)     |      |     |
|   `fsqrt`   |  0x00   |  0xF6   |  REG  | 4 bytes  | raiz cuadrada: fDst = sqrt(fSrc)  |      |     |
| `fsqrt.pd`  |  0x00   |  0xF6   |  REG  | 4 bytes  | raiz cuadrada (alias vectorial)   |      |     |
| `fsqrt.ps`  |  0x00   |  0xF6   |  REG  | 4 bytes  | raiz cuadrada (alias vectorial)   |      |     |
|   `fabs`    |  0x00   |  0xF7   |  REG  | 4 bytes  | valor absoluto: fDst =            | fSrc |     |
|  `fabs.pd`  |  0x00   |  0xF7   |  REG  | 4 bytes  | valor absoluto (alias vectorial)  |      |     |
|  `fabs.ps`  |  0x00   |  0xF7   |  REG  | 4 bytes  | valor absoluto (alias vectorial)  |      |     |
|   `fneg`    |  0x00   |  0xF8   |  REG  | 4 bytes  | negacion: fDst = -fSrc            |      |     |
|  `fneg.pd`  |  0x00   |  0xF8   |  REG  | 4 bytes  | negacion (alias vectorial)        |      |     |
|  `fneg.ps`  |  0x00   |  0xF8   |  REG  | 4 bytes  | negacion (alias vectorial)        |      |     |
|   `fcvt`    |  0x00   |  0xF9   |  REG  | 4 bytes  | GP entero -> ZMM double (s=0)     |      |     |
|  `fcvt.ps`  |  0x00   |  0xF9   |  REG  | 4 bytes  | ZMM double -> GP entero (s=1)     |      |     |
|   `fload`   |  0x00   |  0xFB   |  REG  | 4 bytes  | leer 8 bytes VM memory -> ZMM     |      |     |
|  `fstore`   |  0x00   |  0xFC   |  REG  | 4 bytes  | escribir ZMM -> VM memory         |      |     |

Todas son instrucciones extendidas (prefijo `0x00`).

---

## Codificacion binaria

### Formato FIXED_4 (instrucciones REG, 4 bytes)

```
+--------+--------+----------+----------+
| 0x00   | opcode |  ctrl    |  regs    |
+--------+--------+----------+----------+
  byte0    byte1    byte2      byte3
```

**byte2 (ctrl):**
```
bits 7-3 = 0        (reservado)
bit  2   = s        (modo de fcvt: 0=GP->ZMM, 1=ZMM->GP)
bits 1-0 = 0        (reservado)
```

**byte3 (regs):**
```
bits 7-4 = reg2  (nibble alto: registro GP  en fload/fstore/fcvt)
bits 3-0 = reg1  (nibble bajo: registro ZMM en fmov/fadd/...)
```

Para instrucciones ZMM-ZMM (fmov, fadd, fsub, fmul, fdiv, fcmp, fsqrt,
fabs, fneg) ambos nibbles contienen indices de registros ZMM:
- `reg1` (nibble bajo)  = registro ZMM destino
- `reg2` (nibble alto)  = registro ZMM fuente

Para instrucciones ZMM-GP (fload, fstore, fcvt):
- `reg1` (nibble bajo)  = registro ZMM
- `reg2` (nibble alto)  = registro GP (R0..R15)

### Formato FIXED_11 (fmowi, 11 bytes)

```
+--------+--------+----------+---------------------+
| 0x00   | 0xFA   |  ctrl    |  inmediato (u64 LE) |
+--------+--------+----------+---------------------+
  byte0    byte1    byte2      bytes 3..10
```

**byte2 (ctrl):**
```
bits 7-6 = mode     (ancho: 0=8b, 1=16b, 2=32b, 3=64b; siempre 3 para f64)
bit  5   = is_f32   (0 para double, 1 para float; siempre 0 en fmowi)
bits 4   = 0        (reservado)
bits 3-0 = reg1     (indice del registro ZMM destino, 0..15)
```

Los bytes 3..10 contienen el valor de 64 bits en little-endian que se
interpreta como un double IEEE 754 al cargarse en el registro ZMM.

---

## Instrucciones de carga

### FMOWI fDst, imm64

Carga el inmediato de 64 bits `imm64` interpretado como bits IEEE 754
directamente en el registro ZMM `fDst`.

```c
fmowi f0, 0x3FF0000000000000   // f0 = 1.0
fmowi f1, 0x4000000000000000   // f1 = 2.0
fmowi f2, 0xC014000000000000   // f2 = -5.0
```

Constantes IEEE 754 utiles:

| valor | bits (hex)           |
| :---: | :------------------- |
|  0.0  | 0x0000000000000000   |
|  1.0  | 0x3FF0000000000000   |
|  1.5  | 0x3FF8000000000000   |
|  2.0  | 0x4000000000000000   |
|  3.0  | 0x4008000000000000   |
|  4.0  | 0x4010000000000000   |
| -5.0  | 0xC014000000000000   |

### FMOV fDst, fSrc

Copia el contenido del registro ZMM `fSrc` en `fDst`.

```c
fmowi f0, 0x4000000000000000   // f0 = 2.0
fmov  f1, f0                   // f1 = f0 = 2.0
```

---

## Instrucciones aritmeticas escalares

Todas operan en la forma `fDst op= fSrc` (resultado se almacena en fDst):

### FADD fDst, fSrc

```c
fmowi f0, 0x3FF0000000000000   // f0 = 1.0
fmowi f1, 0x4000000000000000   // f1 = 2.0
fmov  f2, f0
fadd  f2, f1                   // f2 = 1.0 + 2.0 = 3.0
```

### FSUB fDst, fSrc

```c
fmowi f0, 0x4008000000000000   // f0 = 3.0
fmowi f1, 0x3FF0000000000000   // f1 = 1.0
fmov  f2, f0
fsub  f2, f1                   // f2 = 3.0 - 1.0 = 2.0
```

### FMUL fDst, fSrc

```c
fmowi f0, 0x4000000000000000   // f0 = 2.0
fmowi f1, 0x4008000000000000   // f1 = 3.0
fmov  f2, f0
fmul  f2, f1                   // f2 = 2.0 * 3.0 = 6.0
```

### FDIV fDst, fSrc

```c
fmowi f0, 0x4018000000000000   // f0 = 6.0
fmowi f1, 0x4000000000000000   // f1 = 2.0
fmov  f2, f0
fdiv  f2, f1                   // f2 = 6.0 / 2.0 = 3.0
```

---

## Comparacion flotante: FCMP

### FCMP fDst, fSrc

Realiza `fDst - fSrc` y actualiza los flags del proceso sin modificar los
registros ZMM.

| condicion       | ZF | SF | CF |
| :-------------- | :-: | :-: | :-: |
| fDst == fSrc    |  1  |  0  |  0  |
| fDst < fSrc     |  0  |  1  |  1  |
| fDst > fSrc     |  0  |  0  |  0  |

Los flags son los mismos que usan los saltos condicionales. Los saltos
disponibles son `jmp.je` (ZF=1, igual) y `jmp.jne` (ZF=0, distinto).
Para detectar menor-que (SF=1) o mayor-que (SF=0 y ZF=0) se puede usar
la logica del salto opuesto:

```c
fmowi f0, 0x4008000000000000   // f0 = 3.0
fmowi f1, 0x4010000000000000   // f1 = 4.0

fmov  f2, f0
fcmp  f2, f1                   // flags <- f2 - f1 (3.0 - 4.0, negativo)
jmp.je  lbl_igual              // salta si f0 == f1 (ZF=1)
// aqui: f0 != f1 (puede ser menor o mayor)
lbl_igual:
```

---

## Instrucciones unarias

Todas toman dos operandos: `fDst, fSrc`. El resultado se escribe en `fDst`
y la fuente `fSrc` no se modifica. Destino y fuente pueden ser el mismo
registro para operar in-place.

### FSQRT fDst, fSrc

Calcula la raiz cuadrada de `fSrc` y almacena el resultado en `fDst`.

```c
fmowi f0, 0x4010000000000000   // f0 = 4.0
fsqrt f1, f0                   // f1 = sqrt(f0) = 2.0  (f0 no cambia)

fmowi f2, 0x4022000000000000   // f2 = 9.0
fsqrt f2, f2                   // f2 = sqrt(9.0) = 3.0  (in-place)
```

### FABS fDst, fSrc

Elimina el bit de signo de `fSrc` (valor absoluto) y lo almacena en `fDst`.

```c
fmowi f0, 0xC014000000000000   // f0 = -5.0
fabs  f1, f0                   // f1 = |-5.0| = 5.0  (f0 no cambia)
```

### FNEG fDst, fSrc

Invierte el bit de signo de `fSrc` y almacena el resultado en `fDst`.

```c
fmowi f0, 0x4008000000000000   // f0 = 3.0
fneg  f1, f0                   // f1 = -3.0  (f0 no cambia)

fneg  f0, f0                   // f0 = -f0  (in-place)
```

---

## Conversion de tipo: FCVT

### FCVT fDst, rSrc  (s=0, GP entero -> ZMM double)

Convierte el valor entero con signo de 64 bits del registro GP `rSrc` a
double IEEE 754 y lo almacena en `fDst`.

```c
mov   r1, 42
fcvt  f0, r1                   // f0 = (double) 42 = 42.0

mov   r2, 0xFFFFFFFFFFFFFFF9   // r2 = -7 (complemento a dos de 64 bits)
fcvt  f1, r2                   // f1 = (double)(-7) = -7.0
```

### FCVT.PS rDst, fSrc  (s=1, ZMM double -> GP entero)

Convierte el double de `fSrc` a entero con signo de 64 bits truncando
hacia cero (igual que el cast `(int64_t)x` de C) y lo almacena en `rDst`.

```c
fmowi f0, 0x400F333333333333   // f0 = 3.9
fcvt.ps r1, f0                 // r1 = (int64_t) 3.9 = 3

fmowi f1, 0xC005851EB851EB85   // f1 = -2.7
fcvt.ps r2, f1                 // r2 = (int64_t)(-2.7) = -2
```

---

## Acceso a memoria VM: FLOAD / FSTORE

Leen y escriben un double de 64 bits en la memoria virtual de la VM usando
una direccion almacenada en un registro GP.

### FLOAD fDst, rAddr

Lee 8 bytes de `vm_mem[rAddr]` y los carga en el registro ZMM `fDst`.

```c
mov   r1, @Absolute("all.mi_double")
fload f0, r1                   // f0 = vm_mem[r1] (double de 8 bytes)
```

### FSTORE rAddr, fSrc

Escribe los 8 bytes del registro ZMM `fSrc` en `vm_mem[rAddr]`.

```c
fmowi f0, 0x4010000000000000   // f0 = 4.0
mov   r1, @Absolute("all.mi_double")
fstore r1, f0                  // vm_mem[r1] = f0  (4.0)
```

Ejemplo completo de round-trip:

```c
fmowi f0, 0x3FF8000000000000   // f0 = 1.5
mov   r1, @Absolute("all.buf")
fstore r1, f0                  // guardar 1.5 en memoria
fload  f1, r1                  // cargar de vuelta en f1
// f1 debe ser 1.5
```

---

## Ejemplos de uso

Vease [[../../../examples_codes_vm/ejemplo_float_basico.vel]] para un ejemplo
completo de fmowi, fmov, fadd, fsub, fmul y fdiv.

Vease [[../../../examples_codes_vm/ejemplo_float_cmp_saltos.vel]] para el uso
de fcmp con saltos condicionales.

Vease [[../../../examples_codes_vm/ejemplo_float_unario.vel]] para fsqrt,
fabs y fneg.

Vease [[../../../examples_codes_vm/ejemplo_float_conversion.vel]] para fcvt
y fcvt.ps.

Vease [[../../../examples_codes_vm/ejemplo_float_memoria.vel]] para fload y
fstore.

Vease [[../../../examples_codes_vm/ejemplo_float_pipeline.vel]] para un calculo
real que combina varias instrucciones (distancia euclidiana 2D).
