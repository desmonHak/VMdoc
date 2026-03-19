Permite ejecutar una llamada a una dirección, poniendo en la pila la dirección de retorno para volver después usando la instrucción [[RET]].
Lea el apartado [[ConvenciónDeLlamadas]] para saber como hacer llamadas a funciones correctamente.

## CALL dentro de la VM con dirección. 

| instrucción | opcode1 | dirección (56bits) | tamaño  |
| :---------: | :-----: | :----------------: | :-----: |
|   CALLVM    |  0x10   | 0xFFFFFFFFFFFFFFF  | 8 bytes |
## CALL dentro de la VM usando registros.

| instrucción | opcode1 | opcode2 |   byte1    | byte2          | tamaño  |
| :---------: | :-----: | ------- | :--------: | -------------- | :-----: |
| CALLVM reg  |  0x00   | 0x22    | 0b00000000 | ``0b0000``reg1 | 4 bytes |
>Si se pone `0b00000001` en `byte1` la instrucción se convierte en un [[JMP]] (salto) en lugar de un `callvm`