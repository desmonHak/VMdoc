# VMINFO - Informacion de la instancia VM actual

Cada instancia de VestaVM tiene un conjunto de metadatos: cuanta memoria usa, que versiones
corre, que permisos tiene, cual es su punto de entrada, etc. La instruccion `vminfo` permite
que el bytecode lea esa informacion en tiempo de ejecucion.

Analogia: es como llamar a `uname -a` en Linux o a `GetSystemInfo` en Windows, pero desde
dentro del bytecode de la propia VM.

| Instruccion | opcode0 | opcode1 | Tamano  | Descripcion                                          |
| :---------: | :-----: | :-----: | :-----: | :--------------------------------------------------- |
| `vminfo`    |  0x00   |  0x01   | 2 bytes | Escribe la estructura VmInstanceInfoData en R0        |

> **Permiso requerido:** `permissions.VmInfo` debe ser 1 en la instancia. Si no, la
> instruccion puede ser un no-op o lanzar un error de permiso.

---

## Que informacion devuelve

`vminfo` devuelve en R0 un puntero host a una estructura `VmInstanceInfoData` con los
campos:

```cpp
typedef struct VmInstanceInfoData {
    uint8_t  version[2];       // version.subversion (ej. [1, 3] = v1.3)
    uint64_t total_memory;     // bytes reservados para esta instancia VM
    uint32_t id;               // identificador numerico de esta instancia

    uint64_t timestamp_ms;     // marca de tiempo de creacion (ms desde epoch)

    uint64_t code_address;     // direccion base del segmento de codigo (.code)
    uint64_t code_limit;       // direccion limite del segmento de codigo

    uint64_t data_address;     // direccion base del segmento de datos (.all)
    uint64_t data_limit;       // direccion limite del segmento de datos

    // Permisos: que instrucciones privilegiadas puede ejecutar esta instancia
    union {
        struct {
            uint8_t HLT:         1; // permiso para HLT (detener la VM)
            uint8_t VmInfoMgr:   1; // permiso para VMINFOMANAGER
            uint8_t VmInfo:      1; // permiso para VMINFO (esta instruccion)
            uint8_t EDM_EDMW:    1; // permiso para modo distribuido
            uint8_t ENC:         1; // permiso para Exec Native Code
        } flags;
        uint64_t raw;
    } permissions;

    uint16_t port;             // puerto TCP en el que escucha esta instancia

    union {
        struct {
            uint32_t ipv4;     // IPv4 del host en formato network (htonl)
        } HostIpv4;
        struct {
            uint8_t bytes[16]; // IPv6 del host en formato raw
        } HostIpv6;
    } client_ip;

    uint32_t entry_point;      // direccion de la primera instruccion ejecutada (PC inicial)
    uint32_t priority;         // prioridad de esta instancia (0=normal, 255=critica)
    uint64_t pid;              // PID del proceso del sistema operativo que hospeda la VM
} VmInstanceInfoData;
```

---

## Como usar VMINFO

`vminfo` no tiene operandos. El resultado es un puntero host (no un handle GC) a la
estructura de informacion. Para leerlo, usa las instrucciones cursor:

```c
// Obtener la estructura de informacion de la VM
vminfo                    // R0 = puntero host a VmInstanceInfoData

// Mover el puntero al cursor para acceder a los campos
mov    r14, r0
xchg   cur0, r14          // cur0 = puntero a la estructura

// Leer el campo total_memory (offset +2 del inicio = despues de version[2])
// Pero version es uint8_t[2] = 2 bytes, luego hay 6 bytes de padding para alinear uint64
// Aproximado: leer el campo a offset 8 (despues de version + padding de alineacion)
addcur cur0, 8            // avanzar al campo total_memory
readcur r1, cur0          // r1 = total_memory (8 bytes)

// r1 ahora contiene cuanta memoria RAM tiene esta instancia VM
```

### Leer la version

```c
vminfo
mov    r14, r0
xchg   cur0, r14          // cur0 = inicio de la estructura

readcur r1b, cur0         // r1 = version mayor (version[0])
addcur  cur0, 1
readcur r2b, cur0         // r2 = version menor (version[1])
// Ejemplo: r1=1, r2=3 -> version 1.3
```

### Comprobar permisos

```c
vminfo
mov    r14, r0
xchg   cur0, r14

// Los permisos estan en el campo permissions.raw
// Offset aproximado: 2(version) + 6(pad) + 8(total_memory) + 4(id) + 4(pad) + 8(timestamp) + 8+8+8+8(addresses) = ...
// En la practica, usar offsetof(VmInstanceInfoData, permissions) desde C/C++
// Aqui usamos un ejemplo con offset estimado 64:
addcur cur0, 64
readcur r3, cur0          // r3 = permissions.raw (64 bits)

// Bit 2 = VmInfo (permiso de esta instruccion)
shr    r3, 2
andu   r3, 1              // aislar bit
cmpu   r3, 1
jmp.jeq tiene_permiso

// sin permiso vminfo
jmp.jmp fin

tiene_permiso:
    // la instancia tiene permiso de usar vminfo
fin:
    hlt
```

---

## Uso tipico: diagnostico y logging

```c
// Emitir un mensaje de diagnostico con el ID de la instancia
vminfo
mov    r14, r0
xchg   cur0, r14

// Leer el ID de la instancia (offset +10 aprox, uint32_t)
addcur cur0, 10
readcur r2d, cur0          // r2 = id de esta instancia

// Imprimir el ID
getproc r1
mov    r15, 2
calln  @Method("stdlib/native/io/vesta_io:vio_print_int")
```

---

## Notas importantes

- La estructura devuelta es una **copia de solo lectura** gestionada por el runtime. No
  escribas en ella; los cambios no afectarian al estado real de la VM y podrian corromper
  el runtime.
- El puntero en R0 es valido mientras la instruccion siguiente se ejecuta; si hay ciclos
  de GC u otras operaciones entre `vminfo` y el acceso al puntero, la estructura puede
  haber sido reasignada. Lo mas seguro es leer los campos de forma inmediata.
- Los offsets exactos de cada campo dependen de la ABI de compilacion (alineacion del
  compilador). Para acceso preciso, compila una funcion C que use `offsetof()` y usa esos
  valores.

---

Ver tambien: [[VMINFOMANAGER]], [[GETPROC_GETVM_GETMGR]], [[EDM (Exit distributed mode)]]
