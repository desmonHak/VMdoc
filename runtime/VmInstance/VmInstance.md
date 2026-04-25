# VmInstance (Instancia de VM)

Una **VmInstance** (instancia de VM) es una unidad de ejecucion independiente dentro
de VestaVM. Cada instancia tiene su propio espacio de direcciones virtuales, su propio
conjunto de registros y su propio estado de ejecucion.

**Analogia:** piensa en un ordenador virtual. Dentro de un mismo ordenador fisico
(el proceso host) puedes tener varios ordenadores virtuales independientes. Cada uno
tiene su "RAM virtual", sus "programas" y sus "procesos". Una VmInstance es ese
ordenador virtual.

---

## Que contiene una VmInstance

Cada instancia de VM mantiene:

```c
// Componentes de una VmInstance (simplificado)
struct VmInstance {
    uint64_t         instance_id;    // identificador unico de la instancia
    ArenaManager*    memory;         // gestor de memoria virtual (arenas + TLB)
    GcHeap*          gc;             // heap generacional para objetos OOP
    RawAllocator*    raw_alloc;      // allocator manual para FFI
    Loader*          loader;         // cargador de modulos .velb
    Scheduler*       scheduler;      // planificador de procesos
    ProcessVM**      processes;      // lista de procesos activos
    size_t           process_count;
    VmPermissions    permissions;    // permisos habilitados (ENC, EDM, ...)
};
```

---

## Ciclo de vida de una VmInstance

```
1. Crear instancia (new_vm_instance)
   -> Reservar arenas de memoria
   -> Inicializar GC heap
   -> Inicializar el loader

2. Cargar un programa (.velb)
   -> El loader valida la cabecera del binario
   -> Mapea las secciones en el espacio de direcciones virtual
   -> Resuelve simbolos externos (imports FFI)

3. Crear el proceso principal
   -> new ProcessVM con PC = entry_point
   -> vm->make_ready(proc->pid)  // transicion NEW -> READY

4. Ejecutar
   -> El scheduler asigna el proceso a un hilo nativo
   -> El interprete ejecuta instrucciones hasta hlt o excepcion

5. Finalizar
   -> vm->stop()  // senaliza al scheduler que pare
   -> Esperar a que todos los hilos terminen
   -> Liberar memoria
```

---

## Permisos de una VmInstance

Cada instancia puede tener habilitadas o deshabilitadas ciertas capacidades:

| Permiso         | Descripcion                                                |
| :-------------- | :--------------------------------------------------------- |
| `ENC`           | Ejecucion de codigo nativo (shellcode) via instruccion ENC |
| `EDM_EDMW`      | Modo distribuido (enviar instrucciones a maquinas remotas) |
| `RESBP`         | Reservar paginas de memoria virtual directamente           |
| `NETWORK`       | Comunicaciones de red                                      |
| `FILESYSTEM`    | Acceso al sistema de archivos                              |

Los permisos se consultan y configuran via la instruccion `VMINFO`.
Ver [[../SetInstruccionesVM/VMINFO.md]] para la referencia completa.

---

## Relacion entre VmInstance, ProcessVM y Scheduler

```
VmInstance
  |
  +-- Scheduler (pool de hilos nativos)
  |     |
  |     +-- Hilo 0  -> ejecuta ProcessVM_A, ProcessVM_C
  |     +-- Hilo 1  -> ejecuta ProcessVM_B
  |     +-- ...
  |
  +-- [ProcessVM_A] estado=RUNNING
  +-- [ProcessVM_B] estado=READY
  +-- [ProcessVM_C] estado=WAITING
```

Varios `ProcessVM` (hilos verdes / procesos ligeros) se ejecutan concurrentemente
dentro de una sola `VmInstance`. El `Scheduler` los reparte entre los hilos del
pool nativo de forma cooperativa con prioridades y cuantos de tiempo.

---

## Comunicacion entre instancias

Varias `VmInstance` pueden comunicarse:
- **Memoria compartida**: via `RESBP` con flags de manager (memoria publica).
- **Red**: via `EDMW/EDM` para enviar bytecode a una instancia remota.
- **Memoria mapeada**: via `MapMemory` para compartir regiones especificas.

Ver [[../MapMemory.md]] y [[../SetInstruccionesVM/EDMW (Enter Distributed Mode With...).md]].

---

## Instrucciones para introspeccion desde bytecode

```c
getvm  r1   // r1 = puntero a la VM* de esta instancia
getmgr r1   // r1 = puntero al ManageVM* global
vminfo r0   // r0 = cursor al bloque VmInstanceInfoData
```

Ver [[../SetInstruccionesVM/VMINFO.md]] y [[../SetInstruccionesVM/VMINFOMANAGER.md]].

---

Ver tambien:
- [[VmManager.md]] - gestion de multiples instancias
- [[../SetInstruccionesVM/VMINFO.md]] - informacion y permisos de la instancia
- [[../MapMemory.md]] - mapeo de memoria entre instancias
- [[../ConvencionDeLlamadas/ConvencionDeLlamadas.md]] - convencion de llamadas
