Permite intercambiar los valores de los registros.

| instrucción | opcode1 | byte2 | REG 1                  | REG 2                  | tamaño  |
| :---------: | :-----: | :---: | ---------------------- | ---------------------- | :-----: |
|    XCHG     |  0x14   | flags | 0b0`t` `00reg`/reg_ext | 0b0`t` `00reg`/reg_ext | 4 bytes |
- bit `t`: indica si los siguientes 6 bits indican un registro especial o un registro general

```c
# Antes de la ejecucion de la instruccion:
# R0=1, CUR0 = 0
# Despues:
# R0=0, CUR0 = 1
xchg r0, cur0

# Antes de la ejecucion de la instruccion:
# R0=0x2000, CUR0 = 0x1000
# Despues:
# R0=0x1000, CUR0 = 0x2000
xchg cur1, cur0

xchg cur0, r1
xchg r2, r1
```