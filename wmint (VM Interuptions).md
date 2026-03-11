La instrucción `vmint` permite invocar una interrupción virtual.

En 64 bytes la **Interrupt Vector Table (IVT)** tiene una cantidad de ``1024`` entradas (``(4096 * 2) / 8``), es decir, ocupa dos paginas de memoria donde cada entrada es un puntero de 8bytes, mientras que en 32 bytes la tabla tiene una cantidad de 1024 entrada compuesta de `(4096 / 4)`, por lo tanto las interrupciones van desde ``0x0`` hasta `0x400` (1024).

La VM usa el registro global ``IVT`` y contiene un puntero base que la ``IVT`` que usar, por ende se puede usar varias ``IVT`` y donde cada una puede tener un ID de interrupción que exprese distintas acciones, pero se espera que solo haya una normalmente, o que la VM auto gestione este registro para cambiar de IVT correctamente.

el opcode de la instrucción define como `0xf000 & int_id` por lo tanto, las instrucción son las siguientes:

|  opcodes   | representacion  |           raw access            |             uso              |
| :--------: | :-------------: | :-----------------------------: | :--------------------------: |
| ``0xf000`` |  ``vmint 0x0``  |  ``IVT + sizeof(void*) * 0x0``  |        Division por 0        |
| ``0xf001`` |  ``vmint 0x1``  |  ``IVT + sizeof(void*) * 0x1``  |          reservado           |
| ``0xf002`` |  ``vmint 0x2``  |  ``IVT + sizeof(void*) * 0x2``  |              -               |
| ``0xf003`` |  ``vmint 0x3``  |  ``IVT + sizeof(void*) * 0x3``  |          Breakpoint          |
|  `0xf004`  |  ``vmint 0x4``  |  ``IVT + sizeof(void*) * 0x4``  |           Overflow           |
|  `0xf005`  |  ``vmint 0x5``  |  ``IVT + sizeof(void*) * 0x5``  |              -               |
|  `0xf006`  |  ``vmint 0x6``  |  ``IVT + sizeof(void*) * 0x6``  |     Invalid opcode (UD2)     |
|  `0xf007`  |  ``vmint 0x7``  |  ``IVT + sizeof(void*) * 0x7``  |         Stack Fault          |
|  `0xf008`  |  ``vmint 0x8``  |  ``IVT + sizeof(void*) * 0x8``  |   General Protection Fault   |
|  `0xf009`  |  ``vmint 0x9``  |  ``IVT + sizeof(void*) * 0x9``  |          Page Fault          |
|  `0xf00A`  |  ``vmint 0xA``  |  ``IVT + sizeof(void*) * 0xA``  |          Code Fault          |
|  `0xf00B`  |  ``vmint 0xB``  |  ``IVT + sizeof(void*) * 0xB``  |                              |
|  `0xf00C`  |  ``vmint 0xC``  |  ``IVT + sizeof(void*) * 0xC``  |                              |
|  `0xf00D`  |  ``vmint 0xD``  |  ``IVT + sizeof(void*) * 0xD``  |                              |
|  `0xf00E`  |  ``vmint 0xE``  |  ``IVT + sizeof(void*) * 0xE``  |                              |
|  `0xf00F`  |  ``vmint 0xF``  |  ``IVT + sizeof(void*) * 0xF``  |    call vm [[VMSyscall]]     |
| ``0xfXXX`` | ``vmint 0xXXX`` | ``IVT + sizeof(void*) * 0xXXX`` | int definidas por el usuario |
| ``0xf400`` | ``vmint 0x400`` | ``IVT + sizeof(void*) * 0x400`` |           last int           |

Estas funciones que se describen únicamente existen en la primera IVT autodefinida por la VM, para las demás IVT dependerá de lo que el programador haya indicado, o de la VM si creo una nueva IVT:
- `Stack Fault`: ocurre automáticamente cuando un hilo accede fuera los limites de su pila definida.
- `General Protection Fault`: ocurre cuando un hilo viola los permisos de memoria de la vm o de la memoria real.
- `Page Fault`: se auto-invoca cuando un hilo usa alguna instrucción  para acceder modificar memoria no mapeada por la VM o de la maquina real.
- `Code Fault`: ocurre cuando un hilo intenta saltar o ejecutar código fuera de sus limites definidos