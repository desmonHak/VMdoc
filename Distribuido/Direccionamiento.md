# Direccionamiento distribuido en VestaVM

En una red de nodos VestaVM, cada nodo puede tener multiples instancias de VM
(`VmInstance`) y cada una puede tener sus propios procesos, arenas de memoria y
registros. El **sistema de direccionamiento distribuido** define como referenciar
memoria en cualquier nodo de la red.

---

## Organizacion de un nodo

```
Nodo fisico (maquina real)
  |
  +-- VmManager (gestor local)
  |     +-- ArenaManager (memoria publica del manager)
  |
  +-- VmInstance_1
  |     +-- ArenaManager (memoria privada)
  |     +-- Procesos (ProcessVM)
  |     +-- Registros propios
  |
  +-- VmInstance_2
        +-- ArenaManager (memoria privada)
        +-- Procesos
        +-- Registros propios
```

Cada nodo tiene sus propias instancias de VM, cada instancia puede tener sus
propios hilos, arenas de memoria y registros. Las instancias son independientes
entre si pero pueden compartir memoria a traves del VmManager local.

---

## Tipos de direcciones de memoria

### Direccion virtual local

Una direccion que existe dentro de la misma instancia VM. Se accede con
instrucciones normales (`mov`, `readcur`, etc.):

```c
// Direccion virtual local en el rango de codigo (0x100000000...)
mov r1, [0x1234]   // leer la direccion virtual local 0x1234
```

### Direccion host local (mapeada)

Una direccion de la memoria real del proceso host, mapeada en el espacio virtual
de la VM. Tiene la forma `VM_virtual:host_real`:

```
0x1000:0xFFFFFFFFFFFF1234
```

Donde `0x1000` es la direccion virtual en la VM y `0xFFFFFFFFFFFF1234` es la
direccion real del proceso host.

### Direccion remota de VmInstance

Referencia a memoria en una instancia VM especifica de una maquina remota:

```
VMI<1>{0x2000:192.168.1.10:80@0x2000}
```

| Campo            | Descripcion                                            |
| :--------------- | :----------------------------------------------------- |
| `VMI`            | Indica instancia VM remota (VmInstance)                |
| `<1>`            | ID de la instancia remota                              |
| `0x2000`         | Direccion virtual en la instancia local (destino)      |
| `192.168.1.10`   | IP de la maquina remota                                |
| `:80`            | Puerto del VmManager remoto                            |
| `@0x2000`        | Direccion virtual dentro de la instancia remota        |

### Direccion remota del VmManager

Referencia a memoria en el manager (memoria publica) de una maquina remota:

```
VMM{0x1000:192.168.1.10:80@0x1000}
```

| Campo  | Descripcion                                                  |
| :----- | :----------------------------------------------------------- |
| `VMM`  | Indica el manager de una maquina remota (memoria publica)    |

El resto de campos tienen el mismo significado que en VMI.

La diferencia clave es que la memoria del **VMM** es publica (accesible por todas
las instancias del nodo remoto), mientras que la memoria de **VMI** es privada a
una instancia especifica.

---

## Conflictos de direcciones

El VmManager y una VmInstance pueden tener la misma direccion virtual pero apuntar
a memoria diferente. En ese caso, la instancia debe mapear la direccion del manager
bajo una direccion virtual distinta internamente.

---

## Sincronizacion entre nodos

Para detectar cambios en memoria compartida remota, el enfoque recomendado es usar
un **contador de escrituras**: un entero que se incrementa con cada escritura. El
lector compara el contador en lugar de comparar toda la memoria, lo que es mucho
mas eficiente.

---

## Instrucciones relacionadas

```c
// Entrar en modo distribuido (enviar codigo a una maquina remota)
edmw 192.168.1.10, 9000    // conectar y comenzar modo distribuido
// instrucciones que se ejecutaran en el nodo remoto...
edm                        // enviar y ejecutar en el nodo remoto
```

---

Ver tambien:
- [[../runtime/VmManager/VmManager.md]] -- gestion de instancias y red
- [[../SetInstruccionesVM/EDMW (Enter Distributed Mode With...).md]] -- modo distribuido
- [[../SetInstruccionesVM/EDM (Exit distributed mode).md]] -- fin del modo distribuido
- [[../runtime/MapMemory.md]] -- mapeo de memoria entre instancias
