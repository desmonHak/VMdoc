Permite obtener la dirección base de una DLL o librería.

- `rDest` = registro donde se almacenará la **dirección base** de la DLL cargada.

- `rName` = registro que contiene una **dirección en memoria virtual** donde hay una cadena con el nombre de la DLL.
 
```c
LOADLIB rDest, rName

// ejemplos:
LOADLIB r10, r11

mov r11, @str("kernel32.dll")
LOADLIB r10, r11
// r10 = base address de kernel32.dll

```
>R10 = registro donde almacenar la dirección base.
>R11 = registro con dirección a un lugar de la memoria donde haya una cadena con el nombre de la dll.

Puede obtener una dirección e función de la librería usando [[GETPROC]]