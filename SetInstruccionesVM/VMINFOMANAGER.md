# VMINFOMANAGER - Informacion del gestor de instancias VM

Mientras `VMINFO` da informacion de **una** instancia concreta, `VMINFOMANAGER` da
informacion del **gestor** que coordina a todas las instancias de la VM en el proceso.

Un proceso VestaVM puede tener multiples instancias VM corriendo en paralelo, coordinadas
por un `ManageVM`. Esta instruccion expone los datos globales del gestor.

| Instruccion     | opcode0 | opcode1 | Tamano  | Descripcion                                            |
| :-------------: | :-----: | :-----: | :-----: | :----------------------------------------------------- |
| `vminfomanager` |  0x00   |  0x02   | 4 bytes | Puntero host a VmInfoManagerData en R0                 |

> **Permiso requerido:** `permissions.VmInfoMgr` debe ser 1 en la instancia actual. Si no,
> la instruccion puede ser un no-op o lanzar un error de permiso.

---

## Que informacion devuelve

```cpp
typedef struct VmInfoManagerData {
    uint8_t  version[2];         // version.subversion del manager
    uint32_t active_instances;   // cuantas instancias VM estan activas en este momento
    uint64_t total_memory_used;  // suma de bytes usados por todas las instancias

    uint64_t timestamp_ms;       // marca de tiempo de creacion del manager (ms desde epoch)

    union {
        struct {
            uint8_t ipv4_or_6;   // 0 = IPv4, 1 = IPv6
        };
        uint16_t raw;
    } flags1;

} VmInfoManagerData;
```

---

## Como usar VMINFOMANAGER

```c
// Obtener informacion del manager
vminfomanager               // R0 = puntero host a VmInfoManagerData

mov    r14, r0
xchg   cur0, r14            // cur0 = inicio de la estructura

// Leer cuantas instancias estan activas (offset 2, uint32_t)
addcur cur0, 2
readcur r1d, cur0           // r1 = numero de instancias activas

// Leer la memoria total usada (offset 8 aprox despues del padding)
addcur cur0, 6              // avanzar al campo total_memory_used
readcur r2, cur0            // r2 = bytes totales usados por todas las instancias
```

---

## Diferencia con VMINFO

| Aspecto               | `vminfo`                           | `vminfomanager`                         |
| :-------------------- | :--------------------------------- | :-------------------------------------- |
| Alcance               | Una instancia VM                   | El gestor de todas las instancias       |
| Dato clave            | Memoria, permisos, codigo/datos    | Instancias activas, memoria total       |
| Uso tipico            | Diagnostico de una instancia       | Diagnostico global, balanceo de carga   |

---

Ver tambien: [[VMINFO]], [[GETPROC_GETVM_GETMGR]], [[EDM (Exit distributed mode)]]
