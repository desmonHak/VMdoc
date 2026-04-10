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
 
 Puede reservar nuevas paginas usando la instrucción [[RESBP]]
# IP: Puntero de instrucciones (``Instruction Pointer``)
-

# Registros generales
d = ``DWORD`` = ``32bits``
w = ``WORD`` = ``16bits``
b = ``BYTE`` = ``8 bit``

| Registro de 64 bits | 32 bits inferiores | 16 bits inferiores | 8 bits inferiores |
| ------------------- | ------------------ | ------------------ | ----------------- |
| r0                  | r0d                | r0w                | r0b               |
| r1                  | r1d                | r1w                | r1b               |
| r2                  | r2d                | r2w                | r2b               |
| r3                  | r3d                | r3w                | r3b               |
| r4                  | r4d                | r4w                | r4b               |
| r5                  | r5d                | r5w                | r5b               |
| r6                  | r6d                | r6w                | r6b               |
| r7                  | r7d                | r7w                | r7b               |
| r8                  | r8d                | r8w                | r8b               |
| r9                  | r9d                | r9w                | r9b               |
| r10                 | r10d               | r10w               | r10b              |
| r11                 | r11d               | r11w               | r11b              |
| r12                 | r12d               | r12w               | r12b              |
| r13                 | r13d               | r13w               | r13b              |
| r14                 | r14d               | r14w               | r14b              |
| r15                 | r15d               | r15w               | r15b              |
```c
reg  = 0b0001 = r01
mode = 0b00   = 8bits
r01b = 0b00 0001

reg  = 0b0001 = r01
mode = 0b01   = 16bits
r01w = 0b01 0001
 
reg  = 0b0001 = r01
mode = 0b10   = 32bits
r01d = 0b10 0001

reg  = 0b0001 = r01
mode = 0b10   = 64bits
r01  = 0b11 0001
```

Modos de los registros:

| 64 bits  | 32 bits  | 16 bits  |  8 bits  |
| :------: | :------: | :------: | :------: |
| ``0b11`` | ``0b10`` | ``0b01`` | ``0b00`` |
# Registros de extensión
Los registros de "extensión" son cualquier otro que no sea un registro de propósito general, por lo que los registros `rip`, `rbp`, `rsp`, registros cursores y demás se consideran de este tipo. No todas las instrucciones lo soportan o no tienen por que soportarlo completamente.

# Registro cursor

>Todo registro cursor es del tamaño de un puntero en la plataforma objetivo, excepto algún caso en especifico. Los registros cursores pueden contener cualquier tipo de valor ya que se considera un registro mas, pero se recomienda siempre que estos registros solo se usen con motivo de acceder a memoria del Host, usando direcciones conocidas. Por tanto se recomienda tener también direcciones validas en estos registros, o en caso de no usarse, tener el campo en 0 indicando que es "`NULL`" la dirección a acceder.

| registro(``reg_cur``) | codificacion |
| :-------------------: | :----------: |
|       ``cur0``        |    ``00``    |
|       ``cur1``        |    ``01``    |
|       ``cur2``        |    ``10``    |
|       ``cur3``        |    ``11``    |

# Registro de bandera (RFLAG)

## Flag de DM (Modo distribuido)