# FFI - Interoperabilidad con codigo nativo en Vesta

Vesta ofrece tres mecanismos para llamar a funciones escritas en C/C++ u otras DLLs del
sistema, desde las mas simples hasta las mas flexibles:

| Mecanismo | Cuando usarlo |
| :------------------------- | :--------------------------------------------------- |
| **extern declarativo** | DLL conocida en compile-time; type-safe; zero overhead |
| **Plugins VestaPluginAPI** | Extension compleja con init, callbacks, GC awareness |
| **ffi_open/sym/call** | DLL desconocida en compile-time; maxima flexibilidad |

---

## 1. extern declarativo

La forma mas sencilla: declarar que funciones se importan y de donde.

```java
// Importar funciones de una DLL del sistema:
extern "kernel32.dll" {
    fn GetCurrentProcessId() -> u32;
    fn GetTickCount() -> u32;
    fn Sleep(ms: u32) -> void;
}

// Usar directamente:
u32 pid = GetCurrentProcessId();
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
R1..R12 = argumentos (en orden de declaracion)
R15 = argc (numero de argumentos, 0..12)
R0 = valor de retorno
```

Las funciones nativas NO reciben parametro implicito de contexto. Solo los argumentos
declarados en la firma.

### Tipos admitidos en firmas FFI

| Tipo Vesta | Tipo C equivalente | Tamaño |
| :---------------- | :-------------------- | :----- |
| `i8`..`i64` | `int8_t`..`int64_t` | 1..8B |
| `u8`..`u64` | `uint8_t`..`uint64_t` | 1..8B |
| `f32` | `float` | 4B |
| `f64` | `double` | 8B |
| `bool` | `int` (0/1) | 8B |
| `char*`/`cstring` | `const char*` | 8B |
| `void*` | `void*` | 8B |
| `T*` | `T*` | 8B |
| `void` | `void` | - |

### Los EFECTOS de una externa

Una externa es codigo que no esta en el programa: no hay cuerpo que leer. Sin
decir nada, lo unico honesto es suponer que hace **cualquier cosa**, y eso
tumba todas las propiedades de quien la llame. El `extern` es la frontera donde
el lenguaje deja de ser nuestro, y por eso es donde se **definen**.

Definir no es **contratar**, aunque se escriba igual. En una funcion Vesta
`@nopanic` es un contrato que el compilador comprueba contra el codigo; aqui no
hay codigo, asi que es la palabra de quien lo escribe. Una palabra, dos sitios;
cambia quien responde por ella.

Y es una descripcion **completa**: lo que no se escribe, no ocurre. De ahi que
haya formas positivas, que un contrato no necesita -- un contrato acota y le
bastan las negativas --.

#### Lo que puede hacer

| Palabra | Que dice | Que le quita a quien llama |
| :------ | :------- | :------------------------- |
| `@io` | E/S observable: consola, fichero, puerto | `pure` `mem_free` `deterministic` `freestanding` |
| `@throws` | lanza. Admite **de quien** | `pure` `mem_free` `nothrow` |
| `@panics` | **aborta**, que no es lanzar. Tambien admite de quien | `pure` `nopanic` |
| `@alloc` | reserva memoria del monton | `pure` `mem_free` `heap_free` `gc_free` |
| `@allocator` | es un **asignador**: lo que devuelve es **fresco** | lo de `@alloc`, y ADEMAS da una garantia |
| `@maps` | **mapea** espacio de direcciones; puede aliasar el mundo | lo de `@alloc`, mas `readonly` `deterministic` |
| `@frees(n)` | **libera** el argumento n; despues ya no vale | `pure` `mem_free` `readonly` |
| `@nondet` | dos llamadas iguales pueden dar cosas distintas | `deterministic` |
| `@keeps_state` | tiene estado **suyo**: `strtok` recuerda, `errno` queda escrito | `pure` `mem_free` `readonly` `deterministic` |
| `@reads_env(..)` | lee el mundo de fuera. Admite **que parte** | `mem_free`, y `deterministic` solo si lo que lee cambia |
| `@writes_env(..)` | lo cambia. Tambien admite que parte | `pure` `mem_free` `readonly` `deterministic` `freestanding` |
| `@blocks` | puede quedarse **esperando** | `pure` `mem_free` |
| `@traps` | puede fallar en el **procesador**. Admite **cual** | `pure` `mem_free` |

Lo **nuestro** no esta en la tabla, y no es un olvido: a nuestros datos una
externa solo llega por un puntero que le pasamos, y eso se dice en el
**parametro** con `in`/`out`/`inout` -- con su localizacion concreta, que es
mucho mas de lo que un booleano podria decir --. Ver [[DireccionParametros]].

#### Las seis que no encienden nada, y como se relacionan

    @pure  @nothrow  @nopanic  @noblock  @notrap  @det

Ninguna pone nada, y esto hay que entenderlo antes de usarlas porque no
funcionan como se espera de un conjunto de opciones.

**Lo que cambia el juego es escribir la PRIMERA palabra, cualquiera.** Hay dos
estados y solo dos:

| Lo que hay escrito | Como se lee |
| :----------------- | :---------- |
| nada | nadie dijo nada -> se supone que hace **cualquier cosa** |
| una palabra o mas | descripcion **completa** -> lo que no se escribe, no ocurre |

De ahi se sigue todo lo demas. Las seis negativas no se diferencian entre si en
lo que PRODUCEN -- las seis producen lo mismo: nada --, y no se acumulan ni se
refuerzan: `@nothrow @nopanic` no dice mas que `@nothrow` a solas, porque la
segunda ya estaba dicha por el silencio.

Y `@pure` no es "la suma de las otras cinco": no hay suma. Es la que se lee
mejor cuando la respuesta es "no hace **nada**", igual que las otras cinco se
leen mejor cuando lo que se quiere subrayar es una en concreto.

**La trampa, y es la razon de contar todo esto.** Escribir una sola negativa
describe la funcion ENTERA, no solo ese eje:

    extern "kernel32.dll" {
        @nothrow                    // "solo digo que no lanza"...
        fn Sleep(u32 ms) -> void;   // ...pero tambien dijo que no bloquea,
    }                               //    que no hace E/S y que es pura

Lo correcto ahi es `@blocks`, y anadir `@nothrow` si ademas se quiere dejar
dicho que se miro. La regla de fondo -- lo que no se escribe, no ocurre -- es lo
que hace util este sistema, y es tambien lo que lo vuelve peligroso a medio
escribir: **una descripcion incompleta no es una descripcion conservadora, es
una descripcion equivocada**.

Por eso las negativas existen: escribirlas dice que **se penso** en ello, y
dejar la linea en blanco no lo dice. Son la diferencia entre "revise que no
bloquea" y "no me acorde de mirarlo" -- pero solo para quien LEE, porque el
compilador ya daba por supuestas las dos. `@det` es el opuesto de `@nondet`.

#### De quien es lo que sale

    @throws(vesta)    lo lanza NUESTRO runtime: un `catch` lo recoge.
    @throws(native)   lo lanza el OTRO LADO -- una excepcion de C++, un SEH --.
                      Nuestro `catch` NO lo recoge, y desenrollar a traves de
                      nuestros marcos no esta garantizado.
    @panics(vesta)    nuestro gancho de panico.
    @panics(native)   un `abort()` de la libreria.

Sin argumento se supone lo peor. La distincion importa porque **no son el mismo
mecanismo**: modelar solo lo que el lenguaje lanza deja fuera justo la frontera
donde el lenguaje se acaba, que es donde vive el FFI.

#### Que falla, y que parte del mundo

    @traps(div0, access, align, illegal, stack_overflow, fp)
    @reads_env(file, net, clock, random, env, config, process, console, device)
    @writes_env(...)

Desnudas valen por todo. Partir estos dos sacos no es cosmetica:

- dos llamadas que tocan partes **disjuntas** no se estorban -- se pueden
  reordenar, y una que no se usa se puede quitar --; con una sola palabra
  cualquier par choca con cualquier par;
- leer el **reloj** o la **entropia** no da lo mismo dos veces y leer la
  **configuracion** si:

      @reads_env(clock)    -> pierde `deterministic`
      @reads_env(config)   -> lo conserva

  Con un unico "lee el mundo" habia que suponer lo primero siempre, o sea
  tratar a la mayoria por el peor caso.

**Multi-ISA**: cual de esos fallos existe de verdad depende del juego de
instrucciones -- en x86-64 una division entera entre cero atrapa, y en aarch64
**no**: devuelve cero --. Por eso no se deduce de la arquitectura, se declara.

#### Reservar, mapear y liberar NO son lo mismo

Se parecen y confundirlos cuesta caro en las dos direcciones.

- **`@alloc`** dice un **coste**: puede reservar. Con eso se cae `heap_free`, y
  nada mas.
- **`@allocator`** dice una **garantia**: lo que devuelve es memoria **fresca**,
  nadie mas la apunta y quien llama se queda con ella. Eso es lo que permite
  tratar el resultado como una localizacion **propia** -- igual que la de un
  `malloc` nuestro -- en vez de como "puede apuntar a cualquier cosa". Sin esta
  palabra, describir un `HeapAlloc` no servia de nada: se sabia lo que costaba
  y no lo que daba.
- **`@maps`** reserva **espacio de direcciones**, que es otro recurso. Un `mmap`
  o un `MapViewOfFile` pueden mapear algo que **ya existe y esta compartido** --
  un fichero, un dispositivo, la memoria de otro proceso --, asi que lo que
  devuelven **puede aliasar el mundo de fuera**. Decir `@allocator` de un `mmap`
  seria prometer que no, y eso no da un error: da otro resultado en cuanto el
  optimizador se lo crea.
- **`@frees(n)`** dice que libera el argumento `n`. Se modela como una
  **escritura** de lo apuntado -- para que los pases que ya miran eso no tengan
  que aprender nada nuevo -- y ademas queda apuntado CUAL, que es lo que hace
  falta para poder avisar de un uso despues de liberar.

El mismo `mmap` **es** un asignador si se le pasa anonimo y privado. Eso se
decide en la **llamada** y no en la declaracion, asi que lo que se declara es lo
conservador y quien conozca su caso envuelve. Y `realloc` es las dos cosas a la
vez, lo que se dice con las dos palabras:

    @allocator
    @frees(0)
    fn reallocarray(inout u8* p, u64 n, u64 sz) -> u64;

#### Lo que le hace a NUESTRA memoria no es un efecto

Un efecto habla de la **funcion entera**. Lo que la funcion le hace a la memoria
que le pasamos se dice en el **parametro**, con `in`/`out`/`inout`, y ahi se
puede decir mucho mas: el analisis lo resuelve a la **localizacion concreta** del
sitio de llamada.

    @blocks
    @traps(access)
    fn read(i32 fd, out u8* buf, u64 n) -> i64;   // escribe ESE buffer

Ver [[DireccionParametros]]. Un booleano "escribe memoria" no distingue cual, y
por eso las dos mitades hacen falta.

#### `when:`: lo que varia con el objetivo

Cualquiera de las palabras admite un `when:`, que se resuelve **al compilar**:

    @blocks(when: os == "linux")
    @traps(div0, when: arch == "x86-64")

Es lo que distingue esto de repetir la declaracion con `@Target`: alli lo que
cambia es **que simbolo existe**; aqui la funcion es una sola y lo que varia es
lo que hace. Lo escrito sin `when:` vale en todos los objetivos, y lo
condicionado se suma donde case.

#### Ejemplo completo

`examples_codes_vx/530_efectos_ffi.vx` declara las tres capas de FFI que hay de
verdad -- nuestro plugin, la API del sistema (NT/Win32) y las llamadas al
nucleo de Linux -- con una palabra por linea, y el test comprueba que cada una
quita **exactamente** lo que esta tabla dice.

---

## 2. Plugins VestaPluginAPI

Para integraciones mas complejas que necesitan: inicializacion, acceso a la VM, lectura
de memoria VM, callbacks de ciclo de vida.

### Estructura de un plugin

```c
// mi_plugin.c (compilado como .dll/.so)
#include "vesta_plugin.h"

static const VestaPluginAPI *g_api = NULL;

// Punto de entrada obligatorio: llamado UNA VEZ al cargar el plugin
VESTA_PLUGIN_EXPORT void vesta_init(const VestaPluginAPI *api) {
    g_api = api; // guardar el puntero (valido toda la vida del proceso)
}

// Funcion exportada al bytecode Vesta:
VESTA_PLUGIN_EXPORT int64_t mi_funcion(void *proc, int64_t vm_addr, int64_t len) {
    // Leer bytes de la memoria VM:
    uint8_t buf[256];
    g_api->vm_read_bytes(proc, vm_addr, buf, (size_t)len);
    // ... procesar ...
    return 42;
}
```

### Cargar el plugin desde Vesta

```java
// Import estatico via ruta relativa al ejecutable:
extern import "stdlib/native/io/vesta_io";

// Usar la funcion del plugin:
mi_funcion(proc_ptr, buffer_addr, buffer_len);
```

### VestaPluginAPI v2

```c
struct VestaPluginAPI {
    uint32_t api_version; // VESTA_PLUGIN_API_VERSION = 2

    void *manager; // ManageVM* global

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
ffi_call(sym_Sleep, 500); // Sleep(500ms)

// Otro ejemplo: MessageBoxA de user32:
i64 user32 = ffi_open("user32.dll");
i64 msgbox = ffi_sym(user32, "MessageBoxA");
ffi_call(msgbox, 0, "Texto".cstr(), "Titulo".cstr(), 0);
```

### Tabla de instrucciones FFI runtime

| Instruccion | Opcode | Descripcion |
| :------------ | :------ | :--------------------------------------------------- |
| `ffi_open(s)` | `0x62` dlopen | Carga DLL; retorna handle (i64) en R0 |
| `ffi_sym(h,s)` | `0x63` dlsym | Resuelve simbolo; retorna puntero (i64) en R0 |
| `ffi_call(fn, ...)` | `0x64` callni | Invoca por puntero; args en R1..R12; R0 = ret |

`ffi_call` soporta de 0 a 12 argumentos. El numero real de argumentos pasa en R15.
La implementacion interna es identica a la del `CALLN` estatico
(`invoke_native_unchecked`), sin overhead adicional.

---

## Patrones de uso habituales

### Pasar un string Vesta a una API nativa

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
char* ptr = ruta.cstr(); // host pointer nul-terminado (ASCII/UTF-8)

i64 hfile = CreateFileA(ptr, 0x40000000, 0, null, 2, 0x80, null);
// GENERIC_WRITE CREATE_ALWAYS FILE_ATTRIBUTE_NORMAL
```

### Pasar un string UTF-16 a una API Win32 *W

```java
extern "kernel32.dll" {
    fn GetFileAttributesW(lpFileName: char*) -> u32;
}

string ruta = "directorio\archivo.txt";
char* wptr = ruta.wstr();   // buffer UTF-16LE NUL-terminado (wchar_t*)

u32 attrs = GetFileAttributesW(wptr);
```

### Buffer de bytes VM para leer/escribir via FFI

```java
// Reservar un buffer en memoria HOST (para FFI):
u8* buf = malloc(1024);

// Llamar a una funcion nativa que lee/escribe en el buffer:
extern "msvcrt.dll" { fn memset(ptr: void*, val: i32, n: i64) -> void*; }
memset((void*)buf, 0, 1024); // limpiar el buffer

// Leer bytes del buffer:
u8 primer_byte = buf[0];

free(buf);
```

### getproc + vm_read_bytes desde un plugin

```java
// En el lado Vesta, pasar el puntero al proceso actual:
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
void* proc = getproc(); // ProcessVM* del proceso actual
void* vm = getvm(); // VM* de la instancia VM
void* mgr = getmgr(); // ManageVM* del manager global
```

Necesarios para pasar como primer argumento a plugins que usan `vm_read_bytes`.

---

## Direccion inversa: callbacks Vesta desde codigo nativo

Los tres mecanismos anteriores cubren el caso *Vesta llama a nativo*.  El caso
*nativo llama a Vesta* (callbacks, WndProc, signal handlers, qsort comparator)
se implementa con el builtin `as_native_callback(fn)` que devuelve un puntero
a un thunk x86-64 generado dinamicamente.  El thunk respeta la calling
convention del OS (Win64 o System V) y delega al codigo JIT-compilado de la
funcion Vesta.

```java
extern "msvcrt.dll" {
    fn qsort(u64 base, u64 nmemb, u64 size, u64 cmp) -> void;
}

i32 my_cmp(i64 a, i64 b) {
    i32* pa = (i32*) a;
    i32* pb = (i32*) b;
    return *pa - *pb;
}

i32 main() {
    i32[16] arr;
    // ... llenar arr ...
    i64 cb = as_native_callback(my_cmp);
    qsort((u64)(&arr[0]), 16, 4, (u64)cb);
    return 0;
}
```

El overhead actual del bridge nativo->Vesta es aproximadamente 11 nanosegundos
por invocacion -- suficiente para WndProc Win32, audio en tiempo real,
game loops y comparator de qsort.  Detalles completos del mecanismo,
optimizaciones del thunk (TLS-direct via `gs:[]`), gestion de memoria HOST
vs VM, limitaciones y ejemplos extensos en [[CallbacksNativos]].

---

Ver tambien: [[TiposDatos]], [[CallbacksNativos]],
[[SetInstruccionesVM/NativePluginAPI]],
[[SetInstruccionesVM/NativeCall (CallN)]], [[SetInstruccionesVM/GETPROC_GETVM_GETMGR]]
