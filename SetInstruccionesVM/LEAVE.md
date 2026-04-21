Destruye el stack frame creado por [[ENTER]], restaurando `rsp` y `rbp` a sus valores anteriores. Debe ir seguida de [[RET]] para retornar al llamador.

| opcode1 | tamaño  |
| :-----: | :-----: |
|  0x29   | 1 byte  |

**Comportamiento equivalente:**
```asm
mov rsp, rbp   ; libera todo el espacio local (rsp vuelve al nivel de rbp)
pop rbp        ; restaura el base pointer del frame anterior
```

**Ejemplo:**
```asm
my_fn:
    enter 16
    ; ...
    leave
    ret
```
