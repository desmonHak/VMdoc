# API de Plugins Nativos (Modelo A)

Un plugin nativo es una libreria dinamica (`.dll` en Windows, `.so` en Linux)
que VestaVM carga al resolver la tabla de importaciones del ejecutable `.velb`.
El **Modelo A** es el mecanismo principal: el plugin exporta `vesta_init` y
recibe un puntero a `VestaPluginAPI`, que le da acceso controlado a la VM.

---

## Header publico

```c
#include "include/ffi/vesta_plugin.h"
```

El header es C puro (compatible con GCC, Clang y MSVC sin flags C++).

---

## Punto de entrada: vesta_init

El plugin **debe** exportar esta funcion:

```c
void vesta_init(const VestaPluginAPI *api);
```

VestaVM la llama **una sola vez** justo despues de resolver todos los imports
del ejecutable. El plugin debe guardar el puntero `api` en una variable
estatica para usarlo despues:

```c
static const VestaPluginAPI *g_api = NULL;

void vesta_init(const VestaPluginAPI *api) {
    g_api = api;
    /* registrar funciones, crear VMs auxiliares, etc. */
}
```

> `g_api` es valido durante toda la vida del ejecutable. VestaVM garantiza
> que la tabla `VestaPluginAPI` vive en el objeto `Loader` del proceso,
> que persiste hasta el cierre del programa.

---

## VESTA_PLUGIN_EXPORT

Todas las funciones que el bytecode .vel va a llamar con `calln` deben
exportarse con esta macro:

```c
VESTA_PLUGIN_EXPORT uint64_t mi_funcion(uint64_t a, uint64_t b) {
    return a + b;
}
```

En Windows se expande a `__declspec(dllexport)`. En Linux a
`__attribute__((visibility("default")))`.

---

## Convencion de llamada nativa

Las funciones registradas via `@Import` en el bytecode siguen la convencion
estandar de VestaVM **sin cambios**:

| registro | rol                  |
| :------: | :------------------- |
| `r0`     | valor de retorno     |
| `r1-r12` | argumentos (max 12)  |
| `r15`    | numero de argumentos |

No existe ningun argumento implicito (no hay `void *ctx` ni similar).

```c
// bytecode .vel
mov     r1, 42
mov     r15, 1
calln   @Method("mi_lib:mi_funcion")
// r0 = resultado
```

```c
/* plugin C */
VESTA_PLUGIN_EXPORT uint64_t mi_funcion(uint64_t x) {
    return x * 2;
}
```

---

## Acceso a memoria virtual de la VM

Las funciones que necesitan leer o escribir datos en la memoria virtual de
la VM usan el patron `(proc_ptr, vm_addr, len)`:

1. El bytecode obtiene el `ProcessVM*` con `getproc`.
2. Lo pasa como primer argumento (`r1`) junto a la direccion VM (`r2`) y
   la longitud (`r3`).
3. La funcion C llama a `g_api->vm_read_bytes` o `g_api->vm_write_bytes`.

```c
// bytecode .vel
getproc r1
mov     r2, @Absolute("all.buffer")
mov     r3, 64
mov     r15, 3
calln   @Method("mi_lib:leer_datos")
```

```c
/* plugin C */
VESTA_PLUGIN_EXPORT uint64_t leer_datos(uint64_t proc_ptr,
                                        uint64_t vm_addr,
                                        uint64_t len) {
    char buf[256];
    g_api->vm_read_bytes(proc_ptr, vm_addr, buf, len);
    /* ... procesar buf ... */
    return 0;
}
```

---

## Estructura VestaPluginAPI

```c
typedef struct VestaPluginAPI {
    uint32_t       api_version;   /* comparar con VESTA_PLUGIN_API_VERSION */
    VestaManager  *manager;       /* handle al ManageVM activo */

    /* gestion de instancias VM */
    uint64_t   (*create_vm)  (VestaManager *mgr, uint32_t num_schedulers);
    int        (*destroy_vm) (VestaManager *mgr, uint64_t vm_id);
    VestaVM_t *(*get_vm)     (VestaManager *mgr, uint64_t vm_id);
    int        (*has_vm)     (VestaManager *mgr, uint64_t vm_id);
    uint64_t   (*vm_count)   (VestaManager *mgr);

    /* ciclo de vida de la VM */
    void       (*start_vm)   (VestaVM_t *vm);
    void       (*stop_vm)    (VestaVM_t *vm);
    void       (*wait_vm)    (VestaVM_t *vm);

    /* procesos virtuales */
    void       (*spawn_process)(VestaVM_t *vm, uint32_t *out_sched,
                                uint64_t *out_pid);
    void       (*make_ready)   (VestaVM_t *vm, uint32_t sched_id,
                                uint64_t local_pid);

    /* acceso a memoria VM */
    uint64_t   (*vm_read_bytes) (uint64_t proc_ptr, uint64_t vm_addr,
                                 void *dst, uint64_t len);
    uint64_t   (*vm_write_bytes)(uint64_t proc_ptr, uint64_t vm_addr,
                                 const void *src, uint64_t len);

    /* utilidades */
    void       (*log)(const char *msg);
} VestaPluginAPI;
```

### vm_read_bytes

Lee `len` bytes desde la direccion virtual `vm_addr` de la VM al buffer host
`dst`. `proc_ptr` es el valor `uint64_t` obtenido con `getproc`.
Devuelve el numero de bytes leidos.

### vm_write_bytes

Escribe `len` bytes desde el buffer host `src` a la direccion virtual
`vm_addr`. Devuelve el numero de bytes escritos.

### log

Emite un mensaje de texto hilo-seguro a la salida de VestaVM. Util para
depuracion dentro de `vesta_init`.

```c
void vesta_init(const VestaPluginAPI *api) {
    g_api = api;
    api->log("[mi_plugin] cargado");
}
```

---

## Macro de depuracion

Se recomienda proteger los mensajes de carga con una macro para produccion:

```c
#ifndef MI_PLUGIN_DEBUG
#  define MI_PLUGIN_DEBUG 0
#endif

void vesta_init(const VestaPluginAPI *api) {
    g_api = api;
#if MI_PLUGIN_DEBUG
    if (api) api->log("[mi_plugin] cargado");
#else
    (void)api;
#endif
}
```

Compilar con `-DMI_PLUGIN_DEBUG=1` para ver el mensaje.

---

## Ejemplo minimo de plugin

```c
/* mi_plugin.c */
#include "include/ffi/vesta_plugin.h"
#include <stdint.h>

static const VestaPluginAPI *g_api = NULL;

VESTA_PLUGIN_EXPORT void vesta_init(const VestaPluginAPI *api) {
    g_api = api;
}

VESTA_PLUGIN_EXPORT uint64_t suma(uint64_t a, uint64_t b) {
    return a + b;
}
```

```c
// bytecode .vel
@Import {
    @Method {
        @Lib("mi_plugin")
        @Name("suma")
    }
}

code:
    mov     r1, 10
    mov     r2, 32
    mov     r15, 2
    calln   @Method("mi_plugin:suma")
    // r0 = 42
    hlt
end_code:
```

Vease [[GETPROC_GETVM_GETMGR]] para instrucciones de acceso al contexto VM.
Vease [[StdlibNativa]] para los modulos de la libreria estandar nativa.
