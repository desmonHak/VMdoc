Pone un dato en la pila, se puede sacar usando [[POP]]

| instrucción  | opcode1 |     reg     | tamaño  |
| :----------: | :-----: | :---------: | :-----: |
| ``PUSH reg`` |  0x12   | ``0b00``reg | 2 bytes |
>`reg` en este caso es un campo que codifica el registro general junto a su modo, por lo que `reg` en este caso ocupa `6 bits`.

----
Pone un dato en la pila, de los [[REGISTROS#Registros de extensión|registros extendidos]].

| instrucción  | opcode1 |       reg       | tamaño  |
| :----------: | :-----: | :-------------: | :-----: |
| ``PUSH reg`` |  0x12   | ``0b01``reg_ext | 2 bytes |

|  REGISTRO  | CODIFICACIÓN |
| :--------: | :----------: |
|  ``cur0``  |  ``000000``  |
|  ``cur1``  |  ``000001``  |
|  ``cur2``  |  ``000010``  |
|  ``cur3``  |  ``000011``  |
|    ...     |  ``000100``  |
|    ...     |  ``000101``  |
|    ...     |  ``000110``  |
|  ``rip``   |  ``001000``  |
|  ``rbp``   |  ``001001``  |
|  ``rsp``   |  ``001010``  |
| ``rflags`` |  ``001011``  |

