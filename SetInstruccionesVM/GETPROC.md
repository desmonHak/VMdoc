- **R0 debe contener la base address de la DLL** previamente cargada con [[LOADLIB]].
- `rDest` = registro donde se almacenará la **dirección de la función nativa**.
- `rName` = registro que contiene una **dirección en memoria virtual** donde hay una cadena con el nombre de la función.
 
- Internamente llama a `GetProcAddress` / `dlsym`.
- Si la función no existe, la VM lanza una excepción.

```c
GETPROC rDest, rName

// Ejemplos
// cargar DLL
mov r14, @str("kernel32.dll")
LOADLIB r0, r14

// obtener función
mov r12, @str("GetTickCount")
GETPROC r11, r12

// r11 = puntero a GetTickCount


LOADLIB r0, r14
GETPROC r11, r12
// rX = pointer to native function
```
>**GETPROC espera que el registro R0 contenga una dirección base a una dll**.
>
>El primer registro (`r11`) será donde se almacena la dirección de la función nativa
>El segundo registro (`r12`) debe ser una dirección a una cadena con el nombre de la función a cargar.

Puede obtener la dirección de la librería usando [[LOADLIB]]

