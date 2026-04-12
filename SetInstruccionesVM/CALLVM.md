Permite ejecutar una llamada a una dirección, poniendo en la pila la dirección de retorno para volver después usando la instrucción [[RET]].
Lea el apartado [[ConvenciónDeLlamadas]] para saber como hacer llamadas a funciones correctamente.

## CALL dentro de la VM con dirección. 

| instrucción | opcode1 |     byte2      | tamaño  |
| :---------: | :-----: | :------------: | :-----: |
|   CALLVM    |  0x10   | ``0b0000``reg1 | 2 bytes |

>Si se pone `0b0001` en `byte2` la instrucción se convierte en un [[JMP]] (salto) en lugar de un `callvm`
## CALL dentro de la VM usando registros.

| instrucción | opcode1 | opcode2 | dirección (64bits) |  tamaño  |
| :---------: | :-----: | ------- | :----------------: | :------: |
| CALLVM reg  |  0x00   | 0x22    | 0xFFFFFFFFFFFFFFFF | 10 bytes |
