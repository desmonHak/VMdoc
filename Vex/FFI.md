# FFI - Interoperabilidad con codigo nativo en Vex

Vex ofrece tres mecanismos para llamar a funciones escritas en C/C++ u otras DLLs del
sistema, desde las mas simples hasta las mas flexibles:

| Mecanismo                  | Cuando usarlo                                        |
| :------------------------- | :--------------------------------------------------- |
| **extern declarativo**     | DLL conocida en compile-time; type-safe; zero overhead |
| **Plugins VestaPluginAPI** | Extension compleja con init, callbacks, GC awareness |
| **ffi_open/sym/call**      | DLL desconocida en compile-time; maxima flexibilidad |

---

## 1. extern declarativo

La forma mas sencilla: declarar que funciones se importan y de donde.

```java
// Importar funciones de una DLL del sistema:
extern "kernel32.dll" {
    fn GetCurrentProcessId() -> u32;
    fn GetTickCount()        -> u32;
    fn Sleep(ms: u32)        -> void;
}

// Usar directamente:
u32 pid  = GetCurrentProcessId();
u32 tick = GetTickCount();
Sleep(100);
println("PID del proceso: ${pid}");
println("Tick count: ${tick}");
```

El compilador emite `CALLN @Method("kernel32.dll:GetCurrentProcessId")` que el loader
resuelve via `LoadLibraryA` + `GetProcAddress` en tiempo de carga. Cero overhead en
runtime comparado con llamadas directas.

### Convencion de llamada de funciones nativas

```
R1..R12  = argumentos (en orden de declaracion)
R15      = argc (numero de argumentos, 0..12)
R0       = valor de retorno
```

Las funciones nativas NO reciben parametro implicito de contexto. Solo los argumentos
declarados en la firma.

### Tipos admitidos en firmas FFI

| Tipo Vex          | Tipo C equivalente    | Tamano |
| :---------------- | :-------------------- | :----- |
| `i8`..`i64`       | `int8_t`..`int64_t`   | 1..8B  |
| `u8`..`u64`       | `uint8_t`..`uint64_t` | 1..8B  |
| `f32`             | `float`               | 4B     |
| `f64`             | `double`              | 8B     |
| `bool`            | `int` (0/1)           | 8B     |
| `char*`/`cstring` | `const char*`         | 8B     |
| `void*`           | `void*`               | 8B     |
| `T*`              | `T*`                  | 8B     |
| `void`            | `void`                | -      |

---

## 2. Plugins VestaPluginAPI

Para integraciones mas complejas que necesitan: inicializacion, acceso a la VM, lectura
de memoria VM, callbacks de ciclo de vida.

### Estructura de un plugin

```c
// mi_plugin.c  (compilado como .dll/.so)
#include "vesta_plugin.h"

static const VestaPluginAPI *g_api = NULL;

// Punto de entrada obligatorio: llamado UNA VEZ al cargar el plugin
VESTA_PLUGIN_EXPORT void vesta_init(const VestaPluginAPI *api) {
    g_api = api;   // guardar el puntero (valido toda la vida del proceso)
}

// Funcion exportada al bytecode Vex:
VESTA_PLUGIN_EXPORT int64_t mi_funcion(void *proc, int64_t vm_addr, int64_t len) {
    // Leer bytes de la memoria VM:
    uint8_t buf[256];
    g_api->vm_read_bytes(proc, vm_addr, buf, (size_t)len);
    // ... procesar ...
    return 42;
}
```

### Cargar el plugin desde Vex

```java
// Import estatico via ruta relativa al ejecutable:
extern import "stdlib/native/io/vesta_io";

// Usar la funcion del plugin:
mi_funcion(proc_ptr, buffer_addr, buffer_len);
```

### VestaPluginAPI v2

```c
struct VestaPluginAPI {
    uint32_t api_version;       // VESTA_PLUGIN_API_VERSION = 2

    void *manager;              // ManageVM* global

    // Ciclo de vida:
    void (*vm_start)(void *vm);
    void (*vm_stop)(void *vm);

    // Acceso a memoria VM (seguro desde fuera del bytecode):
    void (*vm_read_bytes)(void *proc, uint64_t vm_addr,
                          void *host_buf, size_t len);
    void (*vm_write_bytes)(void *proc, uint64_t vm_addr,
                           const void *host_buf, size_t len);

    // Log:
    void (*log)(const char *msg);

    // GC write barrier (v2):
    void (*gc_addref)(uint64_t proc, uint64_t gc_handle);
    void (*gc_release)(uint64_t proc, uint64_t gc_handle);
};
```

Los metodos `gc_addref`/`gc_release` permiten que el plugin registre GcHandles como
raices externas para que el GC no los recolecte mientras el plugin los referencia.

---

## 3. ffi_open / ffi_sym / ffi_call (dinamico)

Cuando la DLL o la funcion se conoce solo en runtime:

```java
// Abrir una DLL en runtime:
i64 handle = ffi_open("kernel32.dll");

// Resolver un simbolo:
i64 sym_Sleep = ffi_sym(handle, "Sleep");

// Llamar la funcion (los argumentos van despues del simbolo):
ffi_call(sym_Sleep, 500);    // Sleep(500ms)

// Otro ejemplo: MessageBoxA de user32:
i64 user32  = ffi_open("user32.dll");
i64 msgbox  = ffi_sym(user32, "MessageBoxA");
ffi_call(msgbox, 0, str_cstr("Texto"), str_cstr("Titulo"), 0);
```

### Tabla de instrucciones FFI runtime

| Instruccion   | Opcode  | Descripcion                                          |
| :------------ | :------ | :--------------------------------------------------- |
| `ffi_open(s)` | `0x62` dlopen | Carga DLL; retorna handle (i64) en R0          |
| `ffi_sym(h,s)` | `0x63` dlsym | Resuelve simbolo; retorna puntero (i64) en R0 |
| `ffi_call(fn, ...)` | `0x64` callni | Invoca por puntero; args en R1..R12; R0 = ret |

`ffi_call` soporta de 0 a 12 argumentos. El numero real de argumentos pasa en R15.
La implementacion interna es identica a la del `CALLN` estatico
(`invoke_native_unchecked`), sin overhead adicional.

---

## Patrones de uso habituales

### Pasar un string Vex a una API nativa

```java
extern "kernel32.dll" {
    fn CreateFileA(
        lpFileName: char*,
        dwDesiredAccess: u32,
        dwShareMode: u32,
        lpSecurityAttributes: void*,
        dwCreationDisposition: u32,
        dwFlagsAndAttributes: u32,
        hTemplateFile: void*
    ) -> i64;
}

string ruta = "output.txt";
char*  ptr  = ruta.cstr();           // host pointer nul-terminado (ASCII/UTF-8)

i64 hfile = CreateFileA(ptr, 0x40000000, 0, null, 2, 0x80, null);
//                          GENERIC_WRITE           CREATE_ALWAYS FILE_ATTRIBUTE_NORMAL
```

### Pasar un string UTF-16 a una API Win32 *W

```java
extern "kernel32.dll" {
    fn GetFileAttributesW(lpFileName: char*) -> u32;
}

string ruta = "directorio\archivo.txt";
string utf16 = str_convert(ruta, ENC_UTF16);   // convertir a UTF-16LE
char*  wptr  = utf16.wstr();                    // host pointer a wchar_t*

u32 attrs = GetFileAttributesW(wptr);
```

### Buffer de bytes VM para leer/escribir via FFI

```java
// Reservar un buffer en memoria HOST (para FFI):
u8* buf = malloc(1024);

// Llamar a una funcion nativa que lee/escribe en el buffer:
extern "msvcrt.dll" { fn memset(ptr: void*, val: i32, n: i64) -> void*; }
memset((void*)buf, 0, 1024);    // limpiar el buffer

// Leer bytes del buffer:
u8 primer_byte = buf[0];

free(buf);
```

### getproc + vm_read_bytes desde un plugin

```java
// En el lado Vex, pasar el puntero al proceso actual:
extern import "mi_plugin";
extern "mi_plugin" { fn leer_datos(proc: void*, vm_addr: i64, len: i64) -> i64; }

// getproc() devuelve el ProcessVM* del proceso actual:
void* proc = getproc();
leer_datos(proc, buffer_vm_addr, 64);
```

El `ProcessVM*` es necesario para que el plugin acceda a `g_api->vm_read_bytes`.

---

## Proteccion de crashes en plugins

El runtime envuelve cada llamada CALLN en un handler de excepciones:

- En MSVC: `__try/__except` captura crashes SEH (segfault, AV, hardware div/0).
- En MinGW: `try/catch(std::exception)` y `catch(...)` capturan excepciones C++.

Si el plugin crashea, el runtime lanza `FatalError` con kind `FATAL_NATIVE_CRASH` (SEH)
o `FATAL_NATIVE_EXCEPTION` (C++). El bytecode puede capturar este error con
`try/catch (FatalError e)`.

---

## loadmodule: cargar modulos .velb en runtime

```java
// Cargar un .velb adicional y ejecutar su __module_init:
i64 ret = loadmodule("plugins/extra.velb");

// Ahora las clases del modulo estan disponibles:
Class cls = Class.forName("ExtraServicio");
Object svc = cls.newInstance();
```

El modulo cargado se rebasa automaticamente si hay colision de direcciones virtuales con
modulos ya cargados (tabla de relocations en VERSION_VELB=0x2).

---

## Builtins de contexto del runtime

```java
void* proc = getproc();    // ProcessVM* del proceso actual
void* vm   = getvm();      // VM* de la instancia VM
void* mgr  = getmgr();     // ManageVM* del manager global
```

Necesarios para pasar como primer argumento a plugins que usan `vm_read_bytes`.

---

Ver tambien: [[TiposDatos]], [[SetInstruccionesVM/NativePluginAPI]],
[[SetInstruccionesVM/NativeCall (CallN)]], [[SetInstruccionesVM/GETPROC_GETVM_GETMGR]]
