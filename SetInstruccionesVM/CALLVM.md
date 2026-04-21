Permite ejecutar una llamada a una dirección dentro del código de la VM, empujando la dirección de retorno en la pila para poder volver después usando [[RET]].

Vea la convención de llamadas completa en [[ConvenciónDeLlamadas]].

---

## CALLVM con dirección inmediata (absoluta)

La dirección destino se codifica directamente en la instrucción como un valor de 64 bits.

| instrucción   | opcode1 | byte2 (reservado) | dirección (64 bits) | tamaño   |
| :-----------: | :-----: | :---------------: | :-----------------: | :------: |
| `callvm addr` |  0x10   |       0x00        | 0xFFFFFFFFFFFFFFFF  | 10 bytes |

**Comportamiento:**
1. Calcula `ret_addr = RIP + 10` (dirección de la siguiente instrucción).
2. `RSP -= 8; [RSP] = ret_addr` (empuja la dirección de retorno).
3. `RIP = addr` (salta a la función destino).

---

## CALLVM con dirección en registro

La dirección destino se obtiene del valor de un registro general o especial.

| instrucción    | opcode1 | byte2 (reg)              | tamaño  |
| :------------: | :-----: | :----------------------: | :-----: |
| `callvmr reg`  |  0x16   | `[ext:1][modo:2][reg:4][0:1]` | 2 bytes |

El campo `byte2` sigue el mismo formato que [[PUSH]]/[[POP]]:
- Bit 7 (`ext`): `0` -> registro general, `1` -> registro especial.
- Bits 5-6 (`modo`): `00`=8b, `01`=16b, `10`=32b, `11`=64b (solo si `ext=0`).
- Bits 0-3 (`reg`): código de registro (0-15 para generales).

> La dirección se lee siempre como **64 bits** del registro.

**Comportamiento:**
1. Lee la dirección destino del registro indicado.
2. Calcula `ret_addr = RIP + 2`.
3. `RSP -= 8; [RSP] = ret_addr`.
4. `RIP = addr`.

---

## Ejemplo

```asm
; llamada con dirección inmediata
callvm my_function

; llamada con dirección en registro
mov r01, my_function
callvmr r01

my_function:
    enter 16
    ; ...
    leave
    ret
```
