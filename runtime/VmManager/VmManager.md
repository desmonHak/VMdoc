# VmManager (Gestor de instancias)

El **VmManager** (o `ManagerRuntime`) es el componente de nivel mas alto de VestaVM.
Gestiona multiples `VmInstance` y coordina la comunicacion entre ellas, tanto localmente
(dentro del mismo proceso host) como de forma remota (a traves de la red).

**Analogia:** si una `VmInstance` es un ordenador virtual, el `VmManager` es el
**hipervisor**: el sistema que crea, destruye y conecta los ordenadores virtuales.
Tambien actua como servidor de red para que ordenadores en otras maquinas puedan
comunicarse con los locales.

---

## Responsabilidades del VmManager

```
VmManager
  |
  +-- [VmInstance_1] -- programa A
  +-- [VmInstance_2] -- programa B
  +-- [VmInstance_3] -- programa C
  |
  +-- TCPServer (escucha conexiones entrantes)
  |     +-- TLSConnection_1 -> VmInstance remota en 192.168.1.5
  |     +-- TLSConnection_2 -> VmInstance remota en 10.0.0.3
  |
  +-- RequestRouter (enruta comandos al VmInstance correcto)
  +-- ArenaManager (pool de memoria compartida entre instancias)
```

---

## Memoria compartida del manager

El `VmManager` mantiene un pool de memoria accesible por todas las instancias locales.
A diferencia de la memoria privada de cada `VmInstance`, la memoria del manager es
**publica**: cualquier instancia puede mapearla en su propio espacio de direcciones.

```c
// Reservar una pagina en el manager (publica)
mov   r0, 0x2000          // direccion virtual en el manager
mov   r1, 0b00010010      // bit 4 = nueva pagina, bit 1 = manager local
resbp                     // reservar en el manager

// Mapear la pagina del manager en la instancia actual
mov   r1, 0b00100001      // bit 5 = mapear dir virtual del manager
mov   r2, 0x2000          // misma dir en el manager
resbp                     // ahora la instancia puede acceder a 0x2000

// Compartir un valor entre instancias via memoria del manager
mov   r3, 42
mov   [r0], r3            // escribir en la pagina compartida
```

Ver [[../SetInstruccionesVM/RESBP.md]] para el sistema completo de reserva de paginas.

---

## Red y comunicacion remota

El `VmManager` incluye un servidor TCP con TLS para recibir conexiones de instancias
remotas. Cuando un programa ejecuta `EDMW` con una IP y puerto, el `VmManager` remoto
recibe el bytecode y lo ejecuta en una instancia local.

```c
// Enviar un bloque de codigo para ejecutarse en 192.168.1.10:9000
edmw 192.168.1.10, 9000     // conectar y entrar en modo distribuido

mov   r1, 100
mov   r2, 200
addu  r0, r1, r2            // este codigo se ejecuta en la maquina remota

edm                         // enviar y salir del modo distribuido
```

Ver [[EDMW (Enter Distributed Mode With...)]].

---

## Introspection desde bytecode

Se pueden obtener punteros al manager y a la instancia actual directamente desde
el codigo VestaVM:

```c
getmgr r1   // r1 = puntero al ManageVM* global
getvm  r2   // r2 = puntero a la VM* de la instancia actual
getproc r3  // r3 = puntero al ProcessVM* del proceso actual
```

Y consultar informacion del manager via `VMINFOMANAGER`:

```c
vminfom r0  // r0 = cursor al bloque VmInfoManagerData
```

Ver [[VMINFOMANAGER]].

---

## Tabla comparativa: VmInstance vs VmManager

| Aspecto                | VmInstance                          | VmManager                              |
| :--------------------- | :---------------------------------- | :------------------------------------- |
| Nivel                  | Una unidad de ejecucion             | Gestor de multiples unidades           |
| Memoria                | Privada (solo esa instancia)        | Compartida (visible por todas)         |
| Red                    | No gestiona conexiones              | Servidor TCP/TLS para instancias remotas |
| Procesos               | Tiene su propio scheduler           | Coordina schedulers de instancias      |
| Ciclo de vida          | Creada/destruida por el manager     | Unico en el proceso host               |
| Acceso desde bytecode  | `getvm r1` / `vminfo r0`            | `getmgr r1` / `vminfom r0`             |

---

Ver tambien:
- [[VmInstance.md]] - una instancia individual
- [[TLB.md]] - tabla de traduccion de direcciones
- [[VMINFOMANAGER]] - informacion del manager
- [[../MapMemory.md]] - tipos de memoria mapeada
- [[Direccionamiento]] - tipos de direcciones remotas
