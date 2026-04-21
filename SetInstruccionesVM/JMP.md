Instrucciones de salto dentro de la VM. Existen tres variantes según la forma de especificar el destino: inmediata (absoluta), por registro, o relativa (desplazamiento).

---

## JMP / Jcc - dirección inmediata absoluta (64 bits)

Salto incondicional o condicional a una dirección absoluta de 64 bits.

| instrucción    | opcode1 | cond (1 byte) | dirección (64 bits) | tamaño   |
| :------------: | :-----: | :-----------: | :-----------------: | :------: |
| `jmp addr`     |  0x11   |     0x0F      | 0xFFFFFFFFFFFFFFFF  | 10 bytes |
| `jcc addr`     |  0x11   |    0x00-0x0D  | 0xFFFFFFFFFFFFFFFF  | 10 bytes |

El byte `cond` selecciona la condición evaluada contra los flags de `rflags`:

| Código | Mnemónico  | Condición                        |
| :----: | :--------: | -------------------------------- |
| `0x00` | `je` / `jz`  | ZF = 1  (igual / cero)         |
| `0x01` | `jne`/ `jnz` | ZF = 0  (no igual)             |
| `0x02` | `jcs`/ `jae` | CF = 1  (sin borrow, ≥ unsigned)|
| `0x03` | `jcc`/ `jb`  | CF = 0  (con borrow, < unsigned)|
| `0x04` | `jmi`        | SF = 1  (negativo)              |
| `0x05` | `jpl`        | SF = 0  (positivo o cero)       |
| `0x06` | `jvs`        | OF = 1  (overflow)              |
| `0x07` | `jvc`        | OF = 0  (sin overflow)          |
| `0x08` | `jhi`        | CF=1 && ZF=0  (> unsigned)      |
| `0x09` | `jls`        | CF=0 \|\| ZF=1 (≤ unsigned)    |
| `0x0A` | `jge`        | SF = OF  (≥ signed)             |
| `0x0B` | `jlt`        | SF ≠ OF  (< signed)             |
| `0x0C` | `jgt`        | ZF=0 && SF=OF  (> signed)       |
| `0x0D` | `jle`        | ZF=1 \|\| SF≠OF (≤ signed)     |
| `0x0F` | `jmp`        | siempre (incondicional)         |

Si el salto **no se toma**, `RIP` avanza 10 bytes a la siguiente instrucción.

---

## JMPR - salto por registro (incondicional)

El destino es el valor de 64 bits almacenado en un registro.

| instrucción | opcode1 | byte2 (reg)                   | tamaño  |
| :---------: | :-----: | :---------------------------: | :-----: |
| `jmpr reg`  |  0x15   | `[ext:1][modo:2][reg:4][0:1]` | 2 bytes |

El campo `byte2` sigue el mismo formato que [[PUSH]]/[[POP]]:
- Bit 7 (`ext`): `0` -> registro general, `1` -> registro especial.
- Bits 5-6 (`modo`): tamaño del registro (solo si `ext=0`).
- Bits 0-3 (`reg`): código del registro.

> La dirección siempre se lee como **64 bits** del registro, independientemente del modo.

---

## JREL - salto relativo condicional (desplazamiento 32 bits con signo)

Salto relativo desde el **fin** de la instrucción usando un desplazamiento de 32 bits con signo. Permite tanto saltos hacia adelante como hacia atrás (con desplazamiento negativo).

| instrucción           | opcode1 | opcode2 | cond   | padding | disp32 (32 bits) | tamaño  |
| :-------------------: | :-----: | :-----: | :----: | :-----: | :--------------: | :-----: |
| `jrel cond, disp`     |  0x00   |  0x2D   | 1 byte | 1 byte  | 4 bytes          | 8 bytes |

Los mismos códigos de condición que `jmp`/`jcc` (tabla anterior).

**Cálculo del destino cuando el salto se toma:**
```
RIP_nuevo = RIP_actual + 8 + disp32
```
donde `disp32` es un `int32_t` con signo (negativo para saltar atrás).

**Rango máximo:** ±2 GB desde el fin de la instrucción.

---

## Ejemplos

```asm
; salto incondicional absoluto
jmp 0x1000

; salto condicional absoluto (je)
cmp r01, r02
jmp.je target_label      ; cond=0x00

; salto por registro
mov r03, 0x2000
jmpr r03

; salto relativo hacia adelante (+20 bytes desde fin de instrucción)
jrel 0x0F, 20

; salto relativo hacia atrás (bucle)
loop_start:
    ; ...
    cmp r01, 0
    jrel 0x01, -30       ; jne: si r01 != 0 salta atrás
```
