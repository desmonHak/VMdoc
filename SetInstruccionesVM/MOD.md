# MODS y MODU - Modulo entero (resto de la division)

`mods` y `modu` calculan el **resto** de la division entera entre dos registros.
Son la operacion complementaria a la division (`divs`/`divu`): si divides A entre B,
`mods`/`modu` te da lo que "sobra" despues de repartir.

| Instruccion | opcode0 | opcode1 | Modo | Tamano  | Descripcion                                     |
| :---------: | :-----: | :-----: | :--: | :-----: | :---------------------------------------------- |
| `mods`      |  0x00   |  0x40   | REG  | 4 bytes | r_dst = r_dst % r_src (con signo, semantica C)  |
| `modu`      |  0x00   |  0x40   | REG  | 4 bytes | r_dst = r_dst % r_src (sin signo)               |

Mismo opcode2 (0x40); el bit `_signed_instruct` del byte de control distingue
el modo con signo del sin signo.

Implementacion: `src/runtime/exec_instruction_alu.cpp`

---

## Para que sirve (nivel basico)

**Analogia:** si tienes 17 caramelos y los repartes en grupos de 5, puedes hacer 3
grupos completos y te quedan 2.  El resto es 2.  `mods 17, 5` = 2.

Usos tipicos:
- Saber si un numero es par o impar: `mods r0, 2` -> si r0=0 es par
- Calcular posicion en un array circular (ring buffer): `mods indice, tamano`
- Convertir segundos a minutos y segundos: `mods r_total_seg, 60`
- Hash tables: `modu r_hash, r_capacity` para obtener el indice en la tabla

---

## Variantes con y sin signo

La diferencia solo importa cuando el dividendo es negativo:

| Expresion   | Con signo (`mods`) | Sin signo (`modu`) |
| :---------: | :----------------: | :----------------: |
| 7 % 3       |  1                 |  1                 |
| -7 % 3      | -1                 | (valor grande)     |
| 7 % -3      |  1                 | (sin cambio, b>0)  |

`mods` sigue la semantica del operador `%` de C/C++: el signo del resultado es el
del dividendo.  `modu` trata ambos operandos como enteros de 64 bits sin signo.

Para la mayoria de casos practicos (indices, hashes, longitudes) usa `modu`.
Usa `mods` cuando el dividendo puede ser negativo y necesitas el signo correcto.

---

## Division por cero

Si el divisor (r_src) es cero, el comportamiento es seguro: el resultado se
define como 0 y se activan los flags CF y OF para que el programa pueda
detectar el error.

```asm
mods  r0, r1
; si r1 era 0: r0=0, CF=1, OF=1 -> se puede usar jmp.jc para manejar el error
```

---

## Sintaxis y ejemplos

```asm
; mods r_dst, r_src  ->  r_dst = r_dst % r_src  (con signo)
; modu r_dst, r_src  ->  r_dst = r_dst % r_src  (sin signo)

; Ejemplo: verificar paridad
mov   r0, 7
mov   r1, 2
modu  r0, r1       ; r0 = 7 % 2 = 1 (impar)

; Ejemplo: ring buffer de capacidad 8
mov   r5, 15       ; indice = 15
mov   r6, 8        ; capacidad = 8
modu  r5, r6       ; r5 = 15 % 8 = 7 (posicion en el anillo)

; Ejemplo: segundos a mm:ss
mov   r0, 137      ; 137 segundos totales
mov   r1, 60
mods  r2, r0       ; copiar r0 a r2 primero con mov
mov   r2, r0
mods  r2, r1       ; r2 = 137 % 60 = 17 (segundos sobrantes)
divu  r0, r1       ; r0 = 137 / 60 = 2 (minutos)
```

---

## Soporte de ancho de operando

Como todas las instrucciones ALU de VestaVM, `mods`/`modu` pueden operar sobre
distintos anchos usando los sufijos de registro:

| Sufijo | Bits | Ejemplo         |
| :----: | :--: | :-------------- |
| (ninguno) | 64 | `mods r0, r1`  |
| `b`    |  8  | `mods r0b, r1b` |
| `w`    | 16  | `mods r0w, r1w` |
| `d`    | 32  | `mods r0d, r1d` |

---

## Flags actualizados

| Flag | Cuando se activa                  |
| :--: | :-------------------------------- |
| CF   | Divisor es 0 (error)              |
| OF   | Divisor es 0 (error)              |
| ZF   | Resultado = 0                     |
| SF   | Resultado negativo (solo `mods`)  |

---

## Encoding (nivel tecnico)

`mods`/`modu` comparten opcode2 = 0x40 con la instruccion `modu`.
Usan Convention A (igual que el resto de instrucciones ALU, decode_instr_two_op_reg):

```
[0x00][0x40][ctrl_byte][byte3]
  ctrl_byte: bits 7-6 = mode (ancho: 00=8, 01=16, 10=32, 11=64)
             bit  5   = _signed_instruct (1 = mods, 0 = modu)
             bit  4   = direction (no usado para mod)
  byte3: nibble alto = r_src, nibble bajo = r_dst
```

Diferencia con las instrucciones de strings: `mods`/`modu` usan
`decode_instr_two_op_reg` (Convention A), no `decode_instr_raw_bytes`.

La implementacion usa el template `apply_alu_op<ModOp, true/false>` que
reutiliza la infraestructura de la ALU para soportar los cuatro anchos de
operando sin duplicar logica.
