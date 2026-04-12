```c
LOADLIB r15, r14
GETPROC r11, r12
// rX = pointer to native function
```
>GETPROC espera que el registro R15 contenga una dirección base a una dll.
>El primer registro (`r11`) será donde se almacena la dirección de la función nativa
>El segundo registro (`r12`) debe ser una dirección a una cadena con el nombre de la función a cargar.