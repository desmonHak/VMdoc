# GETPROC / GETVM / GETMGR - Obtener contexto de la VM

Estas tres instrucciones cargan punteros de contexto de la VM en un registro general.
Se usan principalmente para pasar el contexto del proceso actual a funciones nativas
(C/C++) que necesitan acceder a la memoria virtual de la VM o a sus estructuras internas.

> **Nota:** No confundir con la instruccion `GETPROC` de resolucion de simbolos de
> bibliotecas dinamicas documentada en [[GETPROC]] (aquella es sobre `GetProcAddress`).
> Estas tres instrucciones tienen semantica completamente distinta.

---

## Para que sirven

VestaVM tiene una jerarquia de contextos:

```
ManageVM (gestor global de instancias)
  +-- VM (una instancia de la maquina virtual)
        +-- Scheduler (asigna procesos a hilos)
              +-- ProcessVM (un proceso: registros, pila, GC propio)
```

Cuando escribes una funcion nativa en C (una biblioteca `.dll` o `.so`) que necesita
leer o escribir en la memoria virtual de un proceso VestaVM, necesitas pasar un puntero
al proceso actual como argumento. Eso es exactamente lo que hacen estas instrucciones.

Analogia: son como "quien soy yo" (GETPROC), "de que instancia VM formo parte" (GETVM)
y "quien gestiona toda la VM" (GETMGR).

---

## Instrucciones

| Instruccion | opcode0 | opcode1 | Modo | Tamano  |
| :---------: | :-----: | :-----: | :--: | :-----: |
| `getproc r` |  0x00   |  0xC6   | REG  | 4 bytes |
| `getvm   r` |  0x00   |  0xC7   | REG  | 4 bytes |
| `getmgr  r` |  0x00   |  0xC8   | REG  | 4 bytes |

Todas son instrucciones extendidas (prefijo `0x00`), formato FIXED_4.

---

## `GETPROC rDst` - puntero al proceso actual

```c
getproc r1      // r1 = ProcessVM* del proceso en ejecucion (como uint64_t)
```

Carga el puntero al `ProcessVM` del proceso que ejecuta esta instruccion en `rDst`.
El valor almacenado es `reinterpret_cast<uint64_t>(vm)` donde `vm` es el `ProcessVM*`
del proceso actual.

Se usa para pasar el contexto del proceso a funciones nativas que acceden a memoria VM
mediante `VestaPluginAPI::vm_read_bytes` / `vm_write_bytes`.

---

## `GETVM rDst` - puntero a la instancia VM

```c
getvm r1        // r1 = VM* de la instancia propietaria del proceso actual
```

Carga el puntero a la instancia `VM` que contiene al proceso actual. La VM es el nivel
encima del proceso: contiene el pool de hilos, los schedulers y la configuracion global.

Recorre internamente: `ProcessVM* -> Scheduler -> VM&`.

---

## `GETMGR rDst` - puntero al gestor global

```c
getmgr r1       // r1 = ManageVM* del gestor global de instancias
```

Carga el puntero al `ManageVM`, el componente que gestiona todas las instancias VM
activas. Necesario para operaciones de administracion: crear o destruir instancias VM,
listar procesos activos globalmente, etc.

Recorre internamente: `ProcessVM* -> Scheduler -> VM& -> ManageVM&`.

---

## Caso de uso principal: funciones nativas con acceso a memoria VM

El patron estandar para llamar a una funcion nativa que lee o escribe en la memoria
virtual de la VM:

```c
// 1. Obtener el puntero al proceso actual (para pasarlo a la funcion nativa)
getproc r1

// 2. Direccion virtual del dato en la VM
mov     r2, @Absolute("all.mi_string")

// 3. Longitud del dato (en bytes)
mov     r3, 12

// 4. Llamar a la funcion nativa (3 argumentos: proc_ptr, vm_addr, len)
mov     r15, 3
calln   @Method("stdlib/native/io/vesta_io:vio_println")
```

En la funcion nativa (C):

```c
static const VestaPluginAPI *g_api;  // inicializada en vesta_init()

uint64_t vio_println(uint64_t proc_ptr, uint64_t vm_addr, uint64_t len) {
    char buf[256];
    // Leer len bytes desde la direccion virtual vm_addr en el proceso proc_ptr:
    g_api->vm_read_bytes(proc_ptr, vm_addr, buf, len);
    buf[len] = '\0';
    puts(buf);
    return 0;
}
```

---

## Por que proc_ptr en lugar de un puntero directo a la memoria

La memoria virtual de la VM no es directamente accesible como memoria del proceso host.
El `ProcessVM*` opaco permite que `vm_read_bytes` y `vm_write_bytes` traduzcan direcciones
VM a direcciones host via el TLB y ArenaManager sin exponer la estructura interna al plugin.

En otras palabras: la VM usa su propia tabla de paginas (TLB) para mapear las direcciones
que el bytecode usa a direcciones reales de RAM. Una funcion nativa no puede simplemente
leer de esas direcciones sin pasar por la API de traduccion.

---

## Codificacion binaria

```
+--------+--------+--------------------+--------------------+
| 0x00   | opcode |  mode<<6 | 0x00   |  0000 | reg_dst    |
+--------+--------+--------------------+--------------------+
  byte0    byte1       byte2                 byte3
```

- `opcode` = `0xC6` (getproc), `0xC7` (getvm), `0xC8` (getmgr)
- `reg_dst` (4 bits bajos de byte3) = indice del registro destino
- `mode` (2 bits altos de byte2) = siempre `11` (64 bits, puntero)

---

## Ejemplo completo: leer y escribir en memoria VM desde una funcion nativa

```c
// En el bytecode Vesta:
@Format("raw")
@SpaceAddress { @Name("mem"), @IniAddress(0x0), @EndAddress(0xFFFFFFFFFFFFFFFF) }
@Section { @Name("all"), @SpaceAddress("mem") @Align(0x1000) }
@Section { @Name("code"), @SpaceAddress("mem") @Align(0x1000) }

all:
    mi_buffer: 0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00   // 8 bytes de espacio

code:
    mov  rsp, 0x00FF0000
    mov  rbp, 0x00FF0000

    // Obtener contexto del proceso
    getproc r1                              // r1 = ProcessVM*

    // Direccion virtual del buffer
    mov     r2, @Absolute("all.mi_buffer")  // r2 = direccion virtual

    // Llamar funcion nativa para escribir un valor en el buffer
    mov     r3, 8                           // r3 = 8 bytes
    mov     r4, 42                          // r4 = valor a escribir
    mov     r15, 4                          // 4 argumentos
    calln   @Method("miplugin/mi_plugin:escribir_valor")

    hlt
```

```c
// En el plugin nativo (C):
uint64_t escribir_valor(uint64_t proc_ptr, uint64_t vm_addr,
                        uint64_t size, uint64_t value) {
    uint64_t v = value;
    // Escribir el valor en la memoria virtual de la VM:
    g_api->vm_write_bytes(proc_ptr, vm_addr, &v, size);
    return 0;
}
```

---

Ver tambien: [[NativePluginAPI]], [[NativeCall (CallN)]], [[cursor]]
