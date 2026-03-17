## JMP dentro de la VM con dirección. 

| instrucción | opcode1 | dirección (56bits) | tamaño  |
| :---------: | :-----: | :----------------: | :-----: |
|     JMP     |  0x23   | 0xFFFFFFFFFFFFFFF  | 8 bytes |
## CALL dentro de la VM usando registros.

| instrucción | opcode1 | opcode2 |   byte1    | byte2          | tamaño  |
| :---------: | :-----: | ------- | :--------: | -------------- | :-----: |
|   JMP reg   |  0x00   | 0x22    | 0b00000001 | ``0b0000``reg1 | 4 bytes |
>Si se pone `0b00000000` en `byte1` la instrucción se convierte en un [[CALLVM]] en lugar de un salto (`JMP`)