# LOADM (Load Memory)

Permite cargar un valor de la memoria en uno o varios registros.

| opcode1       | opcode2       |
| :-----:       | :-----:       |
|   ``0x0``     |  ``0x24``     |


```c
// cargar valor de 64 bits indicado en r00 en el registro r01
LOADM r01, [r00] 
```

| opcode1 | opcode2 |     reg1      |      reg2       |
| :-----: | :-----: | :-----------: | :-------------: |
|  0x00   |  0x24   | `0b0000 0000` | `0b00` ``reg2`` |

```c
LOADM r01w, [r00w + 0x12345678] // suma 0x1234 a la direcion base, max disp32
```

|opcode1 |opcode2 |     reg1        |       reg2       |disp32 (byte1)|disp32 (byte2)|disp32 (byte3)|disp32 (byte4)|
|:------:|:------:|:---------------:|:----------------:|:------:|:------:|:------:|:------:|
|  0x00  |  0x25  | `0b00` ``reg1`` |  `0b00` ``reg2`` | 0x12| 0x34| 0x56| 0x78|