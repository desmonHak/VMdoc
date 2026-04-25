# JMP / Jcc / JMPR / JREL - Instrucciones de salto

Las instrucciones de salto cambian el flujo de ejecucion del programa: en lugar de
ejecutar la siguiente instruccion en secuencia, saltan a otra direccion. Son la base de
todos los `if`, bucles (`while`, `for`) y llamadas a funciones de cualquier lenguaje.

Existen tres variantes segun como se especifica el destino del salto:
- **Inmediata** (absoluta): la direccion esta codificada en la instruccion.
- **Por registro**: la direccion esta en un registro.
- **Relativa**: la instruccion codifica un desplazamiento (offset) en lugar de una direccion absoluta.

---

## Para que sirven los saltos condicionales

Un salto condicional salta **solo si** una condicion es verdadera. La condicion se expresa
en los **flags** (banderas) del registro `rflags`, que se actualizan automaticamente despues
de una instruccion `CMP`, `CMPU` o `CMPS`.

Analogia: es como una bifurcacion en la carretera. Llegas a un cruce con un semaforo
(`CMP`); si esta en verde (`ZF=1`) sigues todo recto; si esta en rojo (`ZF=0`) giras.

```c
// Ejemplo de if/else con saltos:
cmpu  r1, r2        // comparar r1 con r2 (sin signo)
jmp.je igual        // si r1 == r2, saltar a 'igual'

// ... codigo si son diferentes ...
jmp fin

igual:
// ... codigo si son iguales ...

fin:
```

---

## Flags que afectan a los saltos

| Flag | Nombre         | Cuando se activa               |
| :--: | :------------- | :----------------------------- |
| ZF   | Zero Flag      | El resultado fue exactamente 0 |
| CF   | Carry Flag     | Hubo acarreo (overflow sin signo) |
| SF   | Sign Flag      | El resultado fue negativo       |
| OF   | Overflow Flag  | Hubo desbordamiento con signo   |

---

## JMP / Jcc - direccion inmediata absoluta (64 bits)

Salto incondicional o condicional a una direccion absoluta de 64 bits. El ensamblador
resuelve las etiquetas automaticamente a direcciones absolutas.

| Instruccion  | opcode1 | cond (1 byte) | direccion (64 bits)    | Tamano   |
| :----------: | :-----: | :-----------: | :--------------------: | :------: |
| `jmp addr`   |  0x11   |     0x0F      | 8 bytes little-endian  | 10 bytes |
| `jmp.cc addr`|  0x11   |    0x00-0x0D  | 8 bytes little-endian  | 10 bytes |

El byte `cond` selecciona la condicion evaluada contra los flags de `rflags`.
Si el salto **no se toma**, `RIP` avanza 10 bytes a la siguiente instruccion.

### Tabla de condiciones

| Codigo | Mnemonico        | Condicion evaluada               | Uso tipico             |
| :----: | :--------------- | :------------------------------- | :--------------------- |
| `0x00` | `jmp.je` / `jz`  | `ZF = 1` (igual / cero)          | `if (a == b)`          |
| `0x01` | `jmp.jne` / `jnz`| `ZF = 0` (no igual)              | `if (a != b)`          |
| `0x02` | `jmp.jcs` / `jae`| `CF = 1` (sin borrow, >= unsigned)| `if (a >= b)` unsigned |
| `0x03` | `jmp.jcc` / `jb` | `CF = 0` (con borrow, < unsigned) | `if (a < b)` unsigned  |
| `0x04` | `jmp.jmi`        | `SF = 1` (resultado negativo)     | resultado negativo      |
| `0x05` | `jmp.jpl`        | `SF = 0` (positivo o cero)        | resultado positivo      |
| `0x06` | `jmp.jvs`        | `OF = 1` (overflow)               | detectar desbordamiento |
| `0x07` | `jmp.jvc`        | `OF = 0` (sin overflow)           | sin desbordamiento      |
| `0x08` | `jmp.jhi`        | `CF=1 && ZF=0` (> unsigned)       | `if (a > b)` unsigned  |
| `0x09` | `jmp.jls`        | `CF=0 || ZF=1` (<= unsigned)      | `if (a <= b)` unsigned |
| `0x0A` | `jmp.jge`        | `SF == OF` (>= signed)            | `if (a >= b)` signed   |
| `0x0B` | `jmp.jlt`        | `SF != OF` (< signed)             | `if (a < b)` signed    |
| `0x0C` | `jmp.jgt`        | `ZF=0 && SF==OF` (> signed)       | `if (a > b)` signed    |
| `0x0D` | `jmp.jle`        | `ZF=1 || SF!=OF` (<= signed)      | `if (a <= b)` signed   |
| `0x0F` | `jmp` (sin cond) | siempre (incondicional)           | goto                   |

### Cuando usar CMPU vs CMPS antes del salto

- `CMPU` + `jmp.je/jne/jhi/jls/jcs/jcc`: comparacion **sin signo** (para valores unsigned como direcciones, tamanos, contadores).
- `CMPS` + `jmp.jge/jlt/jgt/jle`: comparacion **con signo** (para enteros que pueden ser negativos).

### Ejemplos

```c
// Salto incondicional (goto):
jmp mi_etiqueta

// if (r1 == r2): salto condicional
cmpu  r1, r2
jmp.je son_iguales

// if (r1 < r2) sin signo (unsigned):
cmpu  r1, r2
jmp.jcc r1_menor    // jcc = CF=0 = carry clear = r1 < r2 sin signo

// if (r1 < r2) con signo (signed):
cmps  r1, r2
jmp.jlt r1_negativo  // jlt = SF != OF = menor con signo

// Bucle while (r9 != 0):
inicio_while:
    cmpu  r9, 0
    jmp.je fin_while    // si r9 == 0, salir del bucle
    // ... cuerpo ...
    subu  r9, 1
    jmp inicio_while
fin_while:
```

---

## JMPR - salto por registro (incondicional)

El destino del salto esta en un registro. Util para llamadas indirectas o dispatch tables.

| Instruccion  | opcode1 | byte2 (registro)              | Tamano  |
| :----------: | :-----: | :---------------------------: | :-----: |
| `jmpr reg`   |  0x15   | `[ext:1][modo:2][reg:4][0:1]` | 2 bytes |

La direccion siempre se lee como **64 bits** del registro, independientemente del modo.

```c
// Llamada indirecta (salto a funcion calculada):
mov   r3, @Absolute("code.mi_funcion")
jmpr  r3            // saltar a la direccion en r3

// Tabla de saltos manual:
mov   r1, 2         // indice = 2
mulu  r1, 8         // offset = indice * 8 bytes (tamano de puntero)
mov   r2, @Absolute("all.tabla_funciones")
addu  r2, r1
mov   r3, [r2]      // r3 = tabla_funciones[indice]
jmpr  r3            // saltar a la funcion en la tabla
```

---

## JREL - salto relativo condicional (desplazamiento 32 bits)

Salto relativo desde el **fin** de la instruccion usando un desplazamiento de 32 bits con
signo. Permite saltos hacia adelante (desplazamiento positivo) y hacia atras (negativo).

| Instruccion        | opcode1 | opcode2 | cond   | padding | disp32     | Tamano  |
| :----------------: | :-----: | :-----: | :----: | :-----: | :--------: | :-----: |
| `jrel cond, disp`  |  0x00   |  0x2D   | 1 byte | 1 byte  | 4 bytes    | 8 bytes |

Los mismos codigos de condicion que `jmp`/`jcc` (tabla anterior).

**Calculo del destino cuando el salto se toma:**
```
RIP_nuevo = RIP_actual + 8 + disp32
```
donde `disp32` es un `int32_t` con signo (negativo para saltar atras).

Rango maximo: +/- 2 GB desde el fin de la instruccion.

`JREL` es mas compacto que `JMP` cuando el destino esta cerca (dentro de 2 GB).
El ensamblador suele elegir automaticamente entre `JMP` y `JREL` segun la distancia.

```c
// Salto relativo hacia adelante (+20 bytes desde fin de instruccion):
jrel 0x0F, 20       // 0x0F = siempre (incondicional)

// Bucle con salto relativo hacia atras (el ensamblador calcula el offset):
inicio_bucle:
    // ... cuerpo ...
    cmpu  r1, 0
    jrel 0x01, inicio_bucle - . - 8   // jne: si r1 != 0, volver al inicio
```

---

## Resumen de instrucciones de salto

| Instruccion    | Tamano   | Destino          | Condicional | Tipico uso             |
| :------------- | :------: | :--------------- | :---------: | :--------------------- |
| `jmp addr`     | 10 bytes | absoluto 64 bits | Opcional    | if/else, goto, bucles  |
| `jmpr reg`     | 2 bytes  | registro 64 bits | No          | despacho indirecto     |
| `jrel cond, d` | 8 bytes  | relativo 32 bits | Opcional    | bucles cortos, branches|

---

Ver tambien: [[Aritmetica y logica]] (instrucciones CMP), [[REGISTROS]] (flags), [[CALLVM.md]] (llamadas)
