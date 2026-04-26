# SETCC - Materializar condicion de flags en registro

`setcc` convierte el resultado de una comparacion (almacenado en los flags del procesador)
en un valor entero que puede guardarse en un registro y usarse como dato.

Sin `setcc`, para saber si dos valores son iguales habria que usar un salto condicional:
ir a un bloque que pone 1, saltar el bloque que pone 0, etc.  Con `setcc` es una sola
instruccion: `setcc r_dst, cond` -> r_dst = 1 si la condicion se cumple, 0 si no.

| Instruccion | opcode0 | opcode1 | Modo  | Tamano  | Descripcion                              |
| :---------: | :-----: | :-----: | :---: | :-----: | :--------------------------------------- |
| `setcc`     |  0x00   |  0x43   | INMED | 4 bytes | r_dst = (cond ? 1 : 0) segun flags       |

Implementacion: `src/runtime/exec_instruction_alu.cpp`

---

## Para que sirve

Imagina que quieres guardar en una variable si dos cadenas son iguales para usarlo
despues en un calculo.  En ensamblador clasico tendrias que hacer malabarismos con
saltos.  `setcc` es la forma limpia: compara, y luego captura el resultado como un 0 o 1.

Caso de uso tipico: verificar invariantes, contar cuantas condiciones se cumplen en un
bucle, o pasar una condicion booleana a una funcion nativa.

---

## Tabla de codigos de condicion

El operando `cond` es un numero literal del 0 al 15.  Los mismos codigos que usa `jmp.jXX`.

| cond | Nombre  | Flag(s)         | Condicion                  |
| :--: | :-----: | :-------------- | :------------------------- |
|  0   | JO      | OF=1            | Overflow                   |
|  1   | JNO     | OF=0            | Sin overflow               |
|  2   | JB/JC   | CF=1            | Por debajo / carry         |
|  3   | JNB/JNC | CF=0            | No por debajo / no carry   |
|  4   | JE/JZ   | ZF=1            | Igual / zero               |
|  5   | JNE/JNZ | ZF=0            | No igual / no zero         |
|  6   | JBE     | CF=1 o ZF=1     | Por debajo o igual         |
|  7   | JNBE    | CF=0 y ZF=0     | Por encima                 |
|  8   | JS      | SF=1            | Negativo (sign)            |
|  9   | JNS     | SF=0            | Positivo                   |
| 10   | JP      | PF=1            | Paridad par                |
| 11   | JNP     | PF=0            | Paridad impar              |
| 12   | JL      | SF != OF        | Menor (con signo)          |
| 13   | JNL/JGE | SF == OF        | Mayor o igual (con signo)  |
| 14   | JLE     | ZF=1 o SF!=OF   | Menor o igual (con signo)  |
| 15   | JNLE/JG | ZF=0 y SF==OF   | Mayor (con signo)          |

---

## Sintaxis

```asm
; Forma general:
setcc r_dst, cond_literal

; Ejemplos:
cmpu  r0, r1           ; compara r0 y r1 (sin signo), actualiza ZF/CF/SF/OF
setcc r2, 4            ; r2 = 1 si ZF=1 (iguales), 0 en caso contrario

cmps  r5, r6           ; comparacion con signo
setcc r7, 12           ; r7 = 1 si r5 < r6 (con signo)
```

---

## Ejemplo: interning con setcc

```asm
; verificar que dos handles internados son el mismo (igualdad por puntero)
strintern r5, r4                       ; r5 = handle canonico de "Hello"
strintern r7, r6                       ; r7 = handle canonico de otro "Hello"

cmpu  r5, r7                           ; comparar los handles
setcc r1, 4                            ; r1 = 1 si son iguales (cond JE = 4)
mov   r15, 1
calln @Method("stdlib/native/io/vesta_io:vio_print_uint")  ; imprime 1
```

---

## Encoding (nivel tecnico)

`setcc` usa Convention B (igual que las instrucciones de strings, opcode2 0x43):

```
[0x00][0x43][byte2][0x00]
  byte2 = (cond << 4) | r_dst
```

La funcion de emit es `emit_setcc` en `src/emmit/emmit_decl.cpp`.
La funcion de decode es `decode_instr_raw_bytes` en `src/runtime/decode_instruction.cpp`.

El exec extrae:
```cpp
r_dst  =  instr.data_instruction.reg_data.reg1        & 0xF;  // nibble bajo de byte2
cond   = (instr.data_instruction.reg_data.reg1 >> 4)  & 0xF;  // nibble alto de byte2
```

Nota: el modo de addressing en el InstrTable es `INMED` (no `REG`) porque el operando
`cond` es un numero literal, no un nombre de registro.  Esto afecta a como el emitter
selecciona la variante de la instruccion.
