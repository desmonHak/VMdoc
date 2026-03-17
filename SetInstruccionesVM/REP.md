# REP

Este es un prefijo de instruccion que permite hacer ciertas operaciones de tipo `loop` con algunas instrucciones, aunque no todas las instrucciones soportan este modo.

| opcode1       | opcode2       |
| :-----:       | :-----:       |
|   ``0x0``     |  ``0x66``     |

REP convierte una instrucción de cadena u otra en un bucle automático controlado por R09:
```c
while (R09 != 0) {
    // ejecutar instrucción
    R09--
}
```