## JMP dentro de la VM usando registros.
reg es un registro que contiene una direccion.

| instrucción | opcode1 |     byte2      | tamaño  |
| :---------: | :-----: | :------------: | :-----: |
|    JMPF     |  0x10   | ``0b0001``reg1 | 2 bytes |
>Si se pone `0b0000` en `byte2` la instrucción se convierte en un [[CALLVM]] en lugar de un salto (`JMP`),

## JMP dentro de la VM con dirección (JUMP FAR). 

| instrucción | opcode1 | opcode2 | dirección (64bits) |  tamaño  |
| :---------: | :-----: | ------- | :----------------: | :------: |
|    JMPF     |  0x00   | 0x23    | 0xFFFFFFFFFFFFFFFF | 10 bytes |

## JMP dentro de la VM usando desplazamiento de 20bits.
Permite hacer un salto con un desplazamiento de 20bits.
20 bits = `4.095` = ``0xfff``

| instrucción | opcode1 | opcode2 | 4 bits | 4 bits + byte2 = 20 bits | tamaño  |
| :---------: | :-----: | ------- | :----: | ------------------------ | :-----: |
| JMPU inmed  |  0x00   | 0x22    | 0b1000 | disp 20                  | 4 bytes |
>en caso de que el segundo bit sea 0 como en este caso, el desplazamiento es siempre positivo (solo salta hacia adelante).

| instrucción | opcode1 | opcode2 | 4 bits | 4 bits + byte2 = 20 bits | tamaño  |
| :---------: | :-----: | ------- | :----: | ------------------------ | :-----: |
| JMPS inmed  |  0x00   | 0x22    | 0b1100 | disp 20                  | 4 bytes |
>en caso de que el segundo bit sea 1 como en este caso, el desplazamiento es signed.
