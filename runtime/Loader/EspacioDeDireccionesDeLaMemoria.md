# Espacio de direcciones

Se usa la direccion `0x1 0000 0000` para cargar el codigo en esa seccion, esta direccion por defecto puede cambiarse con la directiva `org`, como tal no es una instruccion de la VM pero permite notificarle ciertas acciones.

Este valor se escogio para permitir tener direcciones arriba y abajo donde poder hacer reservas de memoria. La cantidad de memoria inferior va desde kis siguientes rangos:
```c
0x00000000  <- "inicio de la memoria mapeada"
0xFFFFFFFF  <- "final de la memoria limite"
// partes baja de la memoria de inicio
0x100000000 <- "limite de inicio para los programas por defecto"
// partes alta de la memoria de inicio
```

La cantidad de memoria que se puede acceder usando 32bits de direccionamiento que es lo descrito por el rango de direcciones `0x00000000 - 0xFFFFFFFF` es de ``2^32bytes``, que equivale a ``4GB memoria ram`` se espera que 4Gb de menoria ram en la perte inferior sea suficiente para los programas, pero en caso contrario, el usuario puede usar memoria de direcciones altas.

La cantidad de paginas a mapear en este rango de 32bits es de unas `0xFFFFFFFF / 0xFFF = 1.048.832` de paginas donde cada una es de ``4096 bytes = 4KB``

----

- La configuracion de pila se puede personalizar, se recomienda usar direcciones apartir de `0x100 0000 0000`, y hacer por defecto reservas de `0x800000` que equivale a 8MB de stack lo cual es bastante. 
- Usar este rango permite tener entre `0x1 0000 0000 - 0x100 0000 0000` una cantidad de 1Tb de ram solo para stack de programa cargado directamente en crudo por la VM.
```c
0xFFFFFFFFFFF = 0x100000000 de paginas = 4.294.967.296
```

Se fija un limite de tamaño de stack de 4GB (``0x00000000-0xFFFFFFFF``) aunque los programas pueden alterar este tamaño si es necesario. Este tamaño permite ejecutar en paralelo una cantidad de `4.096` instancias en un mismo manager donde cada instancia tiene un maximo de ``4Gb`` para stacks

```c
----------

0x0000000000000000  <- Uso "libre"
...
0x00000000FFFFFFFF  <- Uso "libre"

----------

0x0000000100000000  <- Uso "stack"
...
0x00000FFFFFFFFFFF  <- Uso "stack"

----------

0x0000100000000000  <- Uso "codigo"
...
0x00001FFFFFFFFFFF  <- Uso "codigo"

----------

0x0000200000000000  <- Uso "metadatos de codigo"
...
0x00002FFFFFFFFFFF  <- Uso "metadatos de codigo"

----------

0x0000300000000000  <- Uso "zona de codigo JIT"
...
0x00003FFFFFFFFFFF  <- Uso "zona de codigo JIT"

----------

0x1000000000000000  <- Uso "mapeo de direcciones Host reales o virtuales"
...
0x1FFFFFFFFFFFFFFF  <- ptrHostLocal a ptrVirtualLocal

----------

0x2000000000000000  <- ptrManager a ptrVirtualLocal
...
0x2FFFFFFFFFFFFFFF  <- ptrManager a ptrVirtualLocal

----------

0x3000000000000000  <- ptrVirtualHostRemoto a ptrVirtualLocal
...
0x3FFFFFFFFFFFFFFF  <- ptrVirtualHostRemoto a ptrVirtualLocal

----------
```