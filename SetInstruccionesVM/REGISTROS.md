Estos registros solo existen dentro de la VM.
## RP: Registro de mapeado de memoria (``Register Page``)

El registro ``RP`` esta formado por un registro [[REGISTROS#DP desplazamiento en la pagina (``Disp Page``)|DP]] de ``12 bits`` y un registro [[REGISTROS#IDP Identificador de pagina (``ID Page``)|IDP]] de ``52 bits``.
```c
uint16_t  DP:12;
uint64_t IDP:52;
uint64_t RP = (IDP << 12) | DP; // CONCATENACIÓN BITS
```

### DP: desplazamiento en la pagina (``Disp Page``)

 rango de valores posibles:
 ```c
0x0000 - 0x0FFF = 4096 bytes (4KB)
0x1000 - 0x1FFF = 4096 bytes (4KB)
0x2000 - 0x2FFF = 4096 bytes (4KB)
 ```

Las instrucciones puede hacer uso de este registro para acceder a ``4096 bytes`` a la vez.
El tamaño de este registro se debe al tamaño de pagina que la VM gestiona. 
Siempre que el usuario mantenga el registro [[REGISTROS#IP Puntero de instrucciones (``Instruction Pointer``)|IP]] dentro de un rango de direcciones de memoria con instrucciones, no habra problemas.

Para acceder mas allá de ``4096 bytes`` de memoria se usa el registro IDP

### IDP: Identificador de pagina (``ID Page``)

 rango de valores posibles:
 ```c
0x0000000000000  - 0xFFFFFFFFFFFFF
 ```

Este registro permite acceder a distintas paginas gestionadas por la instancia de la VM ([[VmInstance]]) o del manager ([[VmManager]]).

----

 ```c
RP = 0x0000000000000000 => IDP = 0x0000000000000 DP = 0x000 // pagina 1
RP = 0x0000000000001000 => IDP = 0x0000000000001 DP = 0x000 // pagina 2
RP = 0x0000000000002000 => IDP = 0x0000000000002 DP = 0x000 // pagina 3
 ```
 >Puede obtener información de la pagina usando [[PAGEINFO]]
 
 Puede reservar nuevas paginas usando la instrucción RESBP
# IP: Puntero de instrucciones (``Instruction Pointer``)