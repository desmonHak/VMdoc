# GETPROC / GETVM / GETMGR

Instrucciones que cargan punteros de contexto de la VM en un registro.
Se usan principalmente para pasar el contexto del proceso actual a funciones
nativas que necesitan acceder a la memoria virtual de la VM.

> **Nota:** No confundir con la instruccion `GETPROC` (LOADLIB+GetProcAddress)
> documentada en [[GETPROC]] (resolucion de simbolos en tiempo de ejecucion).
> Estas son tres instrucciones nuevas con semantica completamente distinta.

---

## Instrucciones

| instruccion | opcode0 | opcode1 | modo | tamano |
| :---------: | :-----: | :-----: | :--: | :----: |
| `getproc`   |  0x00   |  0xC6   | REG  | 4 bytes |
| `getvm`     |  0x00   |  0xC7   | REG  | 4 bytes |
| `getmgr`    |  0x00   |  0xC8   | REG  | 4 bytes |

Todas son instrucciones extendidas (prefijo `0x00`), formato FIXED_4.

---

## GETPROC rDst

Carga el puntero al `ProcessVM` del proceso en ejecucion en `rDst`.

```asm
getproc r1      ; r1 = ProcessVM* del proceso actual (como uint64_t)
```

El valor almacenado es `reinterpret_cast<uint64_t>(vm)` donde `vm` es el
`ProcessVM*` del proceso que ejecuta la instruccion. Se usa para pasar el
contexto del proceso a funciones nativas que operan con memoria VM mediante
`VestaPluginAPI::vm_read_bytes` / `vm_write_bytes`.

---

## GETVM rDst

Carga el puntero a la instancia `VM` (nivel de instancia, encima del proceso)
en `rDst`.

```asm
getvm r1        ; r1 = VM* de la instancia propietaria del proceso
```

Recorre la cadena: `ProcessVM* -> Scheduler -> VM&`.

---

## GETMGR rDst

Carga el puntero al `ManageVM` (gestor global de instancias VM) en `rDst`.

```asm
getmgr r1       ; r1 = ManageVM* del gestor de instancias
```

Recorre la cadena: `ProcessVM* -> Scheduler -> VM& -> ManageVM&`.

---

## Caso de uso principal: funciones nativas con acceso a memoria VM

El patron estandar para llamar a una funcion nativa que lee o escribe en
la memoria virtual de la VM es:

```asm
; 1. Obtener el puntero al proceso actual
getproc r1

; 2. Direccion virtual del dato en la VM
mov     r2, @Absolute("all.mi_string")

; 3. Longitud del dato
mov     r3, 12

; 4. Llamar a la funcion nativa (3 argumentos: proc_ptr, vm_addr, len)
mov     r15, 3
calln   @Method("stdlib/native/io/vesta_io:vio_println")
```

En la funcion nativa (C):

```c
static const VestaPluginAPI *g_api;

uint64_t vio_println(uint64_t proc_ptr, uint64_t vm_addr, uint64_t len) {
    char buf[256];
    g_api->vm_read_bytes(proc_ptr, vm_addr, buf, len);
    buf[len] = '\0';
    puts(buf);
    return 0;
}
```

Vease [[NativePluginAPI]] para la descripcion completa de `VestaPluginAPI`.

---

## Por que proc_ptr en lugar de un puntero directo

La memoria virtual de la VM no es directamente accesible como memoria del
proceso host. El `ProcessVM*` opaco permite que `vm_read_bytes` y
`vm_write_bytes` traduzcan direcciones VM a direcciones host via el TLB y
ArenaManager sin exponer la estructura interna de la VM al plugin.
