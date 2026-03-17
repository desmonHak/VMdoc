# LOADS (Load String)

Se suele usar para procesar un buffer, o cadenas.

```c
string db "Hola mundo"

mov r00, string // direccion de la cadena (inmediato en este caso)
loadsb // cargar un valor de 64 bits del buffer
// despues tenemos en R01 el primer byte de la cadena
// R01b = 'H'

loadsw // cargar ahora 2 bytes
// R01w = 'ol'
// LOADM r01, [r00]
// ADD r00, 2
```

La VM hace exactamente esto:

- Carga ``R01`` = ``[R00]``:
```c
LOADM r01, [r00]
```

- Ajusta R00 según el Direction Flag (``DF``):

  - ``DF = 0`` -> ``R00++``:
    ```c
    INC r00
    ```

  - ``DF = 1`` -> ``R00--``:
    ```c
    DEC r00
    ```

>El incremento y el decremento no tiene por que ser necesariamente de uno, esto depende del tamaño de la instruccion loads, si se usa `loadsb` se incrementera en uno por que solo se esta cargando un byte,
si se usa `loadsw` se incrementa en 2 y de esta manera con los respectivos tamaños `QWORD`, `DWORD`, `WORD`, `BYTE`.
