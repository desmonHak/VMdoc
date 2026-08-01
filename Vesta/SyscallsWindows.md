# Syscalls NT de Windows

`std.syscall.windows` expone la capa **NT** de Windows: la interfaz que hay
justo por debajo de `kernel32.dll`. Sirve para hablar con el kernel sin pasar
por la capa Win32, que para cada operación añade validación de parámetros,
traducción de rutas DOS a rutas NT y su propia gestión de errores.

Estado: **x86-64 implementado y validado**. x86-32 declara el módulo pero sus
dos métodos de contexto siguen en stub.

---

## Índice

- [1. Cómo usarlo](#1-cómo-usarlo)
- [2. Envoltorios disponibles](#2-envoltorios-disponibles)
- [3. Convenciones de la NT API](#3-convenciones-de-la-nt-api)
  - [NTSTATUS](#ntstatus)
  - [Rutas en formato NT](#rutas-en-formato-nt)
  - [Parámetros de entrada/salida](#parámetros-de-entradasalida)
- [4. Ejemplo: escribir y releer un fichero](#4-ejemplo-escribir-y-releer-un-fichero)
- [5. Ejemplo: reservar memoria](#5-ejemplo-reservar-memoria)
- [6. Cómo se resuelve cada función](#6-cómo-se-resuelve-cada-función)
- [7. Puntos de extensión](#7-puntos-de-extensión)
- [Limitaciones](#limitaciones)

---

## 1. Cómo usarlo

Un solo import trae los envoltorios **y** sus tipos:

```vx
import std.syscall.windows;
```

El módulo reexporta `std.ntwindows`, donde viven los tipos de la API
(`NTSTATUS`, `HANDLE`, `UNICODE_STRING`, `OBJECT_ATTRIBUTES`,
`IO_STATUS_BLOCK`, …) y las constantes NT (`FILE_OPEN`, `FILE_CREATE`,
`OBJ_CASE_INSENSITIVE`, `STATUS_SUCCESS`, …). Los tipos están ahí y no en el
módulo de syscalls porque son tipos de la API de Windows en general, y porque
así se evita el ciclo `windows -> arquitectura -> windows`.

Las constantes de acceso, compartición y memoria (`GENERIC_WRITE`,
`SYNCHRONIZE`, `FILE_SHARE_READ`, `MEM_COMMIT`, `PAGE_READWRITE`, …) están
declaradas en `std.windows`, pero **hoy no se exportan** (les falta `public`),
así que de momento se escriben como literal con un comentario. Ver
[Limitaciones](#limitaciones).

---

## 2. Envoltorios disponibles

Todos llevan el **nombre canónico de la NT API**, con los mismos parámetros y
en el mismo orden que la documentación de Microsoft.

**Memoria**

| Función | Para qué |
| :------ | :------- |
| `NtAllocateVirtualMemory(proc, &base, zeroBits, &size, tipo, protec)` | reservar y/o confirmar páginas |
| `NtFreeVirtualMemory(proc, &base, &size, tipo)` | liberar (`MEM_RELEASE`) o descomprometer (`MEM_DECOMMIT`) |
| `NtProtectVirtualMemory(proc, &base, &size, nueva, &anterior)` | cambiar la protección de un rango ya confirmado |

**Ficheros y dispositivos**

| Función | Para qué |
| :------ | :------- |
| `NtCreateFile(&h, acceso, &attrs, &iosb, tam, atribs, comp, disposicion, opciones, ea, eaLen)` | crear o abrir |
| `NtOpenFile(&h, acceso, &attrs, &iosb, comp, opciones)` | abrir uno existente (versión reducida) |
| `NtWriteFile(h, ev, apc, ctx, &iosb, buf, len, &offset, key)` | escribir |
| `NtReadFile(h, ev, apc, ctx, &iosb, buf, len, &offset, key)` | leer |
| `NtFlushBuffersFile(h, &iosb)` | volcar a disco |
| `NtClose(h)` | cerrar cualquier handle del kernel |

**Auxiliares**

| Función | Para qué |
| :------ | :------- |
| `NtCurrentProcess()` | pseudo-handle del proceso actual |
| `RtlInitUnicodeString(&dst, wstr)` | rellenar un `UNICODE_STRING` desde UTF-16 NUL-terminado |
| `InitializeObjectAttributes(&attrs, &nombre, flags, root, sd)` | rellenar un `OBJECT_ATTRIBUTES` |
| `NtGetCurrentProcessorNumberEx(&n)` | procesador lógico actual (alias: `KeGetCurrentProcessorNumberEx`) |

`InitializeObjectAttributes` no es una syscall: en la NT API es una macro de
`ntdef.h`, y aquí es una función normal. Fija `Length` con
`sizeof<OBJECT_ATTRIBUTES>()` —el kernel usa ese campo para validar la versión
del descriptor, así que tomarlo del tipo y no a mano evita que se descuadre si
la estructura cambia.

---

## 3. Convenciones de la NT API

### NTSTATUS

No se devuelve un booleano ni se consulta un error aparte: el resultado **es**
el código. `STATUS_SUCCESS` vale 0 y cualquier valor negativo es un fallo.

```vx
NTSTATUS st = NtClose(h);
if (st != STATUS_SUCCESS) { /* fallo */ }
```

### Rutas en formato NT

El kernel no entiende rutas DOS. `C:\dir\f.txt` se escribe `\??\C:\dir\f.txt`,
y el nombre viaja como UTF-16 dentro de un `UNICODE_STRING`:

```vx
string ruta = "\\??\\C:\\tmp\\salida.txt";

UNICODE_STRING nombre;
RtlInitUnicodeString(&nombre, (PWSTR) ruta.wstr());
```

`wstr()` da el buffer UTF-16 NUL-terminado que la NT API espera; ver
[[Strings]].

### Parámetros de entrada/salida

Muchos parámetros son `[in/out]`: se pasa la dirección de una variable, y el
kernel la reescribe. En `NtAllocateVirtualMemory`, `BaseAddress` entra como
sugerencia (o `null` para que elija el kernel) y sale con la dirección
concedida; `RegionSize` entra con el tamaño pedido y sale redondeado a página.

El `IO_STATUS_BLOCK` de las operaciones de fichero es el que dice **cuánto** se
hizo: `Information` contiene los bytes escritos o leídos.

---

## 4. Ejemplo: escribir y releer un fichero

Sin tocar `kernel32.dll`:

```vx
import std.syscall.windows;

const u32 GENERIC_READ           = 0x80000000;
const u32 GENERIC_WRITE          = 0x40000000;
const u32 SYNCHRONIZE            = 0x00100000;
const u32 FILE_SHARE_READ        = 0x00000001;
const u32 FILE_ATTRIBUTE_NORMAL  = 0x00000080;

i32 main() {
    string ruta = "\\??\\C:\\tmp\\nt.txt";

    UNICODE_STRING nombre;
    RtlInitUnicodeString(&nombre, (PWSTR) ruta.wstr());

    OBJECT_ATTRIBUTES attrs;
    InitializeObjectAttributes(&attrs, &nombre, OBJ_CASE_INSENSITIVE,
                               (HANDLE) 0, (PVOID) 0);

    HANDLE h = (HANDLE) 0;
    IO_STATUS_BLOCK iosb;

    // Crear (o truncar) y escribir.
    NTSTATUS st = NtCreateFile(&h, GENERIC_WRITE | SYNCHRONIZE, &attrs, &iosb,
                               (PLARGE_INTEGER) 0, FILE_ATTRIBUTE_NORMAL,
                               FILE_SHARE_READ, FILE_OVERWRITE_IF,
                               FILE_SYNCHRONOUS_IO_NONALERT | FILE_NON_DIRECTORY_FILE,
                               (PVOID) 0, 0);
    if (st != STATUS_SUCCESS) { return 1; }

    string texto = "NT OK!";
    st = NtWriteFile(h, (HANDLE) 0, (PVOID) 0, (PVOID) 0, &iosb,
                     (PVOID) texto.cstr(), (ULONG) texto.bytes(),
                     (PLARGE_INTEGER) 0, (PULONG) 0);
    NtFlushBuffersFile(h, &iosb);
    NtClose(h);

    // Releer.
    st = NtOpenFile(&h, GENERIC_READ | SYNCHRONIZE, &attrs, &iosb,
                    FILE_SHARE_READ,
                    FILE_SYNCHRONOUS_IO_NONALERT | FILE_NON_DIRECTORY_FILE);
    if (st != STATUS_SUCCESS) { return 2; }

    u8* buf = (u8*) malloc(64);
    st = NtReadFile(h, (HANDLE) 0, (PVOID) 0, (PVOID) 0, &iosb,
                    (PVOID) buf, 64, (PLARGE_INTEGER) 0, (PULONG) 0);
    i64 leidos = (i64) iosb.Information;   // bytes efectivamente leidos
    NtClose(h);
    free(buf);

    return (i32) leidos;   // 6
}
```

---

## 5. Ejemplo: reservar memoria

```vx
const i32 MEM_COMMIT     = 0x00001000;
const i32 MEM_RESERVE    = 0x00002000;
const i32 MEM_RELEASE    = 0x00008000;
const i32 PAGE_READWRITE = 0x04;

HANDLE proc = NtCurrentProcess();

void*  base = (void*) 0;      // null -> que elija el kernel
SIZE_T tam  = 4096;

NTSTATUS st = NtAllocateVirtualMemory(proc, &base, 0, &tam,
                                      MEM_COMMIT | MEM_RESERVE,
                                      PAGE_READWRITE);
// A la vuelta: `base` tiene la direccion concedida y `tam` el tamano
// redondeado a pagina.

// Con MEM_RELEASE el tamano DEBE ser 0 y la direccion la que se concedio.
SIZE_T cero = 0;
NtFreeVirtualMemory(proc, &base, &cero, MEM_RELEASE);
```

---

## 6. Cómo se resuelve cada función

Los números de servicio de Windows **no son estables** entre versiones del
sistema, así que no se pueden compilar dentro del binario: hay que localizar
cada función al ejecutar.

El esquema por defecto es el del propio cargador del sistema:
`LoadLibraryA("ntdll.dll")` y `GetProcAddress` por nombre. Cada envoltorio
guarda el puntero resultante en un `static` propio, de modo que la resolución
ocurre **una sola vez** y las llamadas siguientes son una llamada indirecta:

```vx
public NTSTATUS NtClose(HANDLE Handle) {
    static syscall_ctx_invoke ctx = syscall_ctx_invoke("NtClose");
    if (ctx.target == null) {
        ctx.target = ctx.resolve_method("NtClose");   // una vez
    }
    return ((cfn(HANDLE) -> NTSTATUS)(ctx.target))(Handle);
}
```

El resolutor vive en `std.syscall.windows.ntdll`, un módulo **hoja** que sólo
depende de `std.types`. Está separado por dos razones: no recrear el ciclo
`windows -> arquitectura -> windows`, y quedar disponible para las dos
arquitecturas sin duplicarlo.

El `cfn(...) -> R` del cast es un puntero a función crudo (8 bytes, sin
entorno): llamar por él no añade nada sobre una llamada indirecta normal. Ver
[[FFI]].

---

## 7. Puntos de extensión

`syscall_ctx_invoke` deja dos campos reasignables:

| Campo | Qué hace |
| :---- | :------- |
| `resolve_method` | localiza el objetivo a partir del nombre |
| `invoke_method` | realiza la llamada |

Por defecto son la resolución vía cargador y la llamada indirecta descritas
arriba. Quien necesite otro esquema —por ejemplo *hooking* para instrumentar
llamadas, una convención de llamada distinta, o localizar los servicios por
otro medio— enchufa su propia implementación sin tocar los envoltorios, que
siguen usándose igual.

La estructura reserva además dos campos para esquemas que trabajen a nivel de
ISA en lugar de llamar al export: `id` (número de servicio) y `hash` (del
nombre, para no llevarlo en texto). El resolutor por defecto no los usa; `hash`
se deriva en el constructor `comptime`, y el nombre en sí es un campo
`comptime`, así que no ocupa espacio en la instancia ni se serializa.

---

## Limitaciones

1. **Sólo x86-64.** El módulo x86-32 existe y se selecciona por `@Target`, pero
   sus `resolve` e `invoke` siguen en stub.

2. **`hash` no se calcula todavía.** El constructor `comptime` lo deja a 0; hoy
   no lo usa nadie porque el resolutor por defecto trabaja por nombre.

3. **La cobertura es la mínima útil**: memoria, ficheros y los auxiliares
   necesarios para usarlos. No hay procesos, hilos, secciones, registro ni
   sincronización. Añadir un envoltorio nuevo es repetir el patrón de arriba
   con la firma correcta.

4. **Sin traducción de rutas.** Hay que escribir el nombre en formato NT
   (`\??\C:\...`) a mano; la conversión desde ruta DOS la hace Win32, que es
   justo la capa que se está evitando.

5. **Las constantes de `std.windows` no se exportan.** Están declaradas
   (`GENERIC_WRITE`, `MEM_COMMIT`, `PAGE_READWRITE`, …) pero sin `public`, así
   que `import std.windows only GENERIC_WRITE` no las encuentra. Mientras tanto
   se escriben como literal, igual que hace el resto de la stdlib. Las
   constantes propiamente NT (`FILE_OPEN`, `OBJ_CASE_INSENSITIVE`,
   `STATUS_SUCCESS`, …) sí son públicas y llegan con el import del módulo.

---

Ver también: [[FFI]] (`extern`, `cfn`, punteros a función), [[Strings]]
(`cstr()` / `wstr()` en la frontera nativa), [[TiposDatos]] (punteros y
memoria host), [[InlineAsm]] (syscalls escritas directamente en ensamblador).
