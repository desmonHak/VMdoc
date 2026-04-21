Permite retornar a la dirección guardada en la pila por una instrucción [[CALLVM]] o [[CALLVM#CALLVM con dirección en registro|CALLVMR]] previa.

| opcode1 | tamaño  |
| :-----: | :-----: |
|  0xC3   | 1 byte  |

**Comportamiento:**
1. Lee `ret_addr = [RSP]`.
2. `RSP += 8`.
3. `RIP = ret_addr`.

> Siempre se empareja con [[LEAVE]] antes de retornar cuando la función usó [[ENTER]].

**Ejemplo:**
```asm
my_fn:
    enter 0
    ; ... cuerpo ...
    leave
    ret
```
