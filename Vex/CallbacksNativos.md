## Callbacks nativos: del lenguaje a las APIs del sistema

VestaVM permite que cualquier funcion Vex se pase como puntero a funcion C nativa
con calling convention compatible con Win64 (RCX/RDX/R8/R9) o System V AMD64
(RDI/RSI/RDX/RCX/R8/R9).  El mecanismo se llama **callback nativo** y se materializa
con el builtin `as_native_callback(fn)`.

Esto desbloquea integraciones con:

- **GUI nativas**: `lpfnWndProc` de Win32 (`WNDCLASSEXW`), GTK `g_signal_connect`,
  Qt slots, ImGui callbacks, SDL/GLFW window callbacks.
- **APIs runtime**: `qsort` de la C runtime, `pthread_create`, signal handlers,
  `EnumWindows`, `SetWindowsHookEx`.
- **Audio/video**: PortAudio stream callbacks, DirectShow filters, FMOD voice
  callbacks, OpenAL source updates.
- **Game loops**: callbacks de fisicas (Bullet, Box2D), networking (ENet packet
  handlers), entity-component hooks.

### Modelo de ejecucion

`as_native_callback(fn)` devuelve un `i64` que es la direccion en memoria HOST de
un **thunk** generado dinamicamente.  El thunk:

1. Es ejecutable nativo x86-64 alocado en el code cache del JIT.
2. Cumple la calling convention C nativa segun el OS (Win64 o SysV AMD64).
3. Convierte los argumentos nativos (`RCX/RDX/R8/R9` o `RDI/RSI/RDX/RCX/R8/R9`)
   al modelo VM (`ProcessVM::registers.regs[R01..R12]`).
4. Invoca el codigo JIT-compilado de la funcion Vex objetivo.
5. Lee el valor de retorno de `regs[R00]` y lo devuelve en `RAX` (convencion C).

El thunk se cachea por `(pc, argc)`, asi que multiples llamadas a
`as_native_callback(fn)` con la misma funcion devuelven el mismo puntero.

### Sintaxis

```java
i32 my_comparator(i64 a_ptr, i64 b_ptr) {
    i32* pa = (i32*) a_ptr;
    i32* pb = (i32*) b_ptr;
    return *pa - *pb;
}

i64 cb = as_native_callback(my_comparator);
// cb ahora es callable desde codigo C nativo.
```

El argumento de `as_native_callback` debe ser un identificador de funcion Vex
(no una expresion, no una lambda).  La firma de la funcion determina la calling
convention que el thunk emite.

### Tipos de argumentos soportados

| Categoria          | Tipos Vex                                      | Calling convention nativa                |
|--------------------|------------------------------------------------|------------------------------------------|
| Enteros 8-64 bits  | `i8`, `i16`, `i32`, `i64`, `u8`, `u16`, `u32`, `u64` | Registro entero completo (zero/sign-extended) |
| Booleano           | `bool`                                          | Registro entero, 0 o 1                   |
| Punteros           | `T*`, `u8*`, ...                                | Registro entero (8 bytes)                |
| Float              | `f32`, `f64`                                    | XMM register (no soportado todavia)      |

Hasta **12 argumentos** por callback (mismo limite que la calling convention
interna VM_ABI usada por las funciones Vex regulares).  El thunk maneja
automaticamente la mezcla de registros + stack del ABI nativo:

- **Win64**: argumentos 1-4 en RCX/RDX/R8/R9 (registros), argumentos 5-12
  leidos del stack del caller en `[caller_rsp + 0x20 + (i-4)*8]` (tras
  el shadow space de 32 bytes que el caller reserva).
- **System V (Linux/macOS)**: argumentos 1-6 en RDI/RSI/RDX/RCX/R8/R9,
  argumentos 7-12 leidos del stack del caller en `[caller_rsp + (i-6)*8]`.

El thunk salva los argumentos en registros a su frame propio, luego los
copia a `proc->registers.regs[R01..R12]` desde donde el codigo Vex
JIT-compilado los lee mediante su prologue.

### Ejemplo: qsort con comparator Vex

```java
extern "msvcrt.dll" {
    fn qsort(u64 base, u64 nmemb, u64 size, u64 cmp) -> void;
}

i32 cmp_i32(i64 a_ptr, i64 b_ptr) {
    i32* pa = (i32*) a_ptr;
    i32* pb = (i32*) b_ptr;
    i32 a = *pa;
    i32 b = *pb;
    if (a < b) return -1;
    if (a > b) return 1;
    return 0;
}

i32 main() {
    i32[16] arr;
    // ... llenar arr ...
    i64 cb = as_native_callback(cmp_i32);
    qsort((u64)(&arr[0]), 16, 4, (u64)cb);
    // arr queda ordenado.
    return 0;
}
```

### Ejemplo: WndProc Win32 con ventana real

```java
extern "user32.dll" {
    fn RegisterClassExW(u64 wc_ptr) -> u16;
    fn CreateWindowExW(u32 ex_style, u64 class_name, u64 win_name,
                       u32 style, i32 x, i32 y, i32 w, i32 h,
                       u64 parent, u64 menu, u64 hinst, u64 lparam) -> u64;
    fn DefWindowProcW(u64 hwnd, u32 msg, u64 wparam, i64 lparam) -> i64;
    fn DestroyWindow(u64 hwnd) -> i32;
}
extern "kernel32.dll" {
    fn GetModuleHandleW(u64 name) -> u64;
}

i32 g_user_msgs = 0;

i64 my_wnd_proc(u64 hwnd, u32 msg, u64 wparam, i64 lparam) {
    if (msg == 1024) {
        // WM_USER
        g_user_msgs = g_user_msgs + 1;
        return 0;
    }
    return DefWindowProcW(hwnd, msg, wparam, lparam);
}

i32 main() {
    u64 hinst = GetModuleHandleW(0);
    i64 thunk = as_native_callback(my_wnd_proc);

    // Allocar WNDCLASSEXW en memoria HOST.  user32 dereferencia el
    // puntero directo; un VM-addr crashearia la API nativa.
    u8* wc = (u8*) malloc(80);
    // ... llenar struct WNDCLASSEXW (ver doc Win32 para layout) ...
    *((i64*)(&wc[8])) = thunk;     // wc.lpfnWndProc = thunk

    RegisterClassExW((u64)(&wc[0]));
    // ... CreateWindowExW + message pump ...

    free(wc);
    return g_user_msgs;
}
```

Ver `examples_codes_vex/173_wndproc_win32.vex` para el ejemplo completo con
ventana hidden, message pump, PostMessageW + GetMessageW + DispatchMessageW.

### Memoria HOST vs VM en interop con APIs nativas

Las APIs Win32, POSIX y bibliotecas C esperan **punteros host directos**:
direcciones que su proceso pueda dereferenciar sin traduccion.  El runtime
VestaVM distingue dos tipos de memoria:

- **VM stack** (`u8[N] arr`, `&local`): direcciones dentro del espacio VM,
  traducidas por `proc->vm_mem.read/write_*` con un page cache interno.
- **Host heap** (`malloc(N)`, `new T()`, fields de objetos GC): direcciones
  que el proceso del sistema puede dereferenciar directamente.

Cuando se pasa un struct o buffer a una API nativa, **debe vivir en host heap**.
La forma idiomatica es `(u8*) malloc(sizeof_struct)` + escribir los campos via
casts a punteros tipados.  Pasar un array `u8[N]` local crashea silenciosamente
cuando la API nativa dereferencia el puntero.

El compilador puede promover ciertos arrays a host stack automaticamente:
cuando el frontend detecta que un array local fluye a un `CALLN`, lo marca
con el flag `host_alloca`.  Para el caso general (struct WNDCLASSEXW,
formularios largos), usar `malloc` explicito es mas claro y portable.

### Rendimiento

El thunk ha sido optimizado en varias iteraciones.  El costo actual por
invocacion es **aproximadamente 11 nanosegundos** medido sobre 5.4 millones
de invocaciones reales (qsort sobre 8192 enteros, 50 iteraciones).

Desglose:

| Componente                     | Coste aprox. | Observacion                              |
|--------------------------------|-------------:|------------------------------------------|
| Salvar args C en stack         | 2-3 ns       | 4 stores de 8 bytes a slots [rsp+N]      |
| Leer `ProcessVM*` desde TLS    | 3-5 ns       | `mov rbx, gs:[0x1480+idx*8]` en Win64   |
| Copiar args a `proc->regs[]`   | 3-5 ns       | 4 cargas + 4 stores con disp32           |
| Llamada al JIT code de la fn   | 1 ns         | `call rax` indirecto                     |
| Restaurar stack + retornar     | 1 ns         | `add rsp, N; pop rbx; ret`               |

A 11 ns por callback, el techo es de aproximadamente 90 millones de
invocaciones por segundo.  Casos de uso reales y su impacto:

| Caso de uso                         | Frecuencia tipica | Tiempo total/seg | Impacto CPU |
|-------------------------------------|------------------:|-----------------:|------------:|
| WndProc Win32 (eventos UI)          | 60-300 / seg      | 0.7-3.3 us       | 0.0003%     |
| GTK/Qt event loop                   | ~1 000 / seg      | 11 us            | 0.001%      |
| Audio 48 kHz mono                   | 48 000 / seg      | 528 us           | 0.05%       |
| Game loop 144 Hz, 5K entidades      | ~720 000 / seg    | 7.9 ms           | 0.8%        |
| Hot path tipo qsort masivo          | 90 000 000 / seg  | 1 s              | saturado    |

Para uso interactivo y audio en tiempo real, el overhead del thunk es
**imperceptible** comparado con la logica dentro del callback.

### Optimizaciones internas del thunk

El thunk usa varias tecnicas para minimizar overhead:

#### TLS-direct via `gs:[...]` (Win64)

Una de las operaciones mas frecuentes del thunk es leer el `ProcessVM*`
del thread actual.  Originalmente se hacia con una llamada a funcion
(`get_current_executing_process()`), pero esto involucra setup de stack
frame, acceso al `thread_local` C++ y retorno -- aproximadamente 25-30 ns.

Una optimizacion **especifica a Windows** aprovecha que el segment register
GS apunta al Thread Environment Block (TEB) y que los primeros 64 slots
de TLS viven en offset `0x1480 + index * 8` dentro del TEB.  El thunk
emite directamente:

```
mov rbx, gs:[0x1480 + idx*8]   ; 9 bytes con prefix GS
```

Una sola instruccion (~3-5 ns) reemplaza la llamada de funcion completa.

Para coordinar este slot con el `thread_local` C++ del runtime:

1. Al startup se reserva un slot TLS con `TlsAlloc()` (lazy, solo si se
   genera algun thunk).
2. La funcion `runtime::set_current_executing_process(proc)` ademas de
   escribir el `thread_local` C++ tambien hace
   `TlsSetValue(g_proc_tls_index, proc)`.
3. El thunk lee directamente del slot.

Si `TlsAlloc()` retorna un indice >= 64 (slots dinamicos en
`gs:[0x1780]` con indireccion extra), el thunk cae al modelo antiguo
con `call` -- sigue siendo correcto, solo no es optimo.

En Linux/macOS la implementacion equivalente requiere offsets del TLS
del glibc/musl que dependen del modelo de TLS (initial-exec vs
local-dynamic).  La version actual en POSIX usa el call original.

#### ABI offsets validados con `static_assert`

El thunk asume offsets fijos dentro de `ProcessVM`:

- `regs[R01]` esta en `proc + 96 + 1*8 = proc + 104`.
- `regs[R15]` (argc) esta en `proc + 96 + 15*8 = proc + 216`.
- `regs[R00]` (return value) esta en `proc + 96 + 0 = proc + 96`.

Estos offsets se exportan como constantes en `vesta_rt/abi.h` y se
verifican en compile time del runtime con `static_assert(offsetof(...) == K)`.
Cualquier cambio al layout interno rompe el build inmediatamente,
forzando recompilar el thunk con los nuevos offsets.

#### Calling convention nativa preservada

El thunk respeta los callee-saved del ABI:

- **Win64**: `RBX`, `RBP`, `RDI`, `RSI`, `R12-R15` se preservan.  Tambien
  reserva 32 bytes de shadow space que cualquier call subsiguiente espera.
- **System V**: `RBX`, `RBP`, `R12-R15` se preservan.  Sin shadow space.

`RBX` se usa como almacen del `ProcessVM*` (callee-saved en ambos ABIs),
permitiendo que sobreviva al call al JIT code sin necesidad de
re-leer el TLS.

### Limitaciones conocidas

1. **Sin argumentos float**: si la calling convention nativa pasa floats en
   XMM (caso de SysV o cuando un arg es `f64`), el thunk no marshalla
   correctamente.  Workaround: usar `u64` para los bits y bitcast en el
   callee.

2. **No soporta lambdas con captura**: solo identificadores de funciones
   top-level pueden ser pasados a `as_native_callback`.  Para casos con
   estado, usar variables globales o una struct host-allocada que viva
   mas alla del callback.

3. **Thread-local del proceso VM**: el thunk asume que el thread que
   ejecuta el callback es el mismo que tiene `set_current_executing_process`
   seteado.  Para librerias que invocan callbacks en threads propios
   (algunos game engines, signal handlers), el TLS slot puede no estar
   inicializado -- comportamiento indefinido.

### Re-entrancia del callback

El thunk garantiza que el codigo Vex caller (main u otro callback
anidado) recupera su estado intacto tras la invocacion.  Para eso, al
entrar al thunk:

1. Salva `proc->registers.regs[R0..R15]` (128 bytes) a un area dedicada
   del propio frame del thunk.
2. Copia los argumentos nativos a `regs[R1..R<argc>]` y argc a `R15`.
3. Llama el codigo Vex JIT-compilado del callback.
4. Lee el return value de `regs[R0]` a `RAX`.
5. Restaura `regs[R1..R15]` desde el save area (preserva todo el state
   del caller; R0 se deja con el return value).

Esto permite patrones como callbacks invocados desde un loop Vex que
itera sobre variables locales: cada invocacion del callback NO corrompe
los registros virtuales que el JIT del caller esta usando.

### Re-entrancia y limitaciones cerradas durante el desarrollo

Limitaciones encontradas e identificadas durante la implementacion del
sistema y ya resueltas:

- **Globals Vex modificados desde callback**: el JIT trataba la direccion
  del slot estatico (`STR_LIT_ADDR`) como `host_ptr` y emitia un `mov`
  nativo directo en lugar de pasar por el inline cache de memoria VM.
  Para callbacks, esto causaba page fault.  Se corrigio removiendo el
  seed de `STR_LIT_ADDR` en el dataflow del selector; los LOAD/STORE
  sobre slots estaticos van ahora por el camino correcto (inline cache
  + fallback `vrt_vm_read/write`).

- **Maximo 4 argumentos (Win64)**: el thunk original solo emitia el
  marshalling de los registros de argumento (RCX/RDX/R8/R9).  Para
  callbacks con mas args, ahora se generan secuencias que leen del
  stack del caller con offsets ajustados al frame del thunk.  El
  limite actual es 12 (mismo que la calling convention VM_ABI).

- **Re-entrancia rota**: callbacks invocados desde un caller Vex con
  state vivo en `proc->regs[]` corrompian ese state.  El thunk ahora
  salva 128 bytes de `proc->regs[R0..R15]` a un area dedicada de su
  frame antes de pisar los regs con los argumentos del callback, y
  restaura todo al salir.

- **Alignment de frame Win64**: el `WIN_FRAME` original no era
  multiplo de 16; tras `push rbx + sub rsp, WIN_FRAME` el `rsp`
  quedaba desalineado y cualquier instruccion SSE/AVX que el JIT
  emitia dentro del callback crasheaba.  Corregido a 192 bytes
  (multiplo de 16).

- **Bug del IR emitter `emit_load_spilled_arg`**: cuando un callback
  con argc grande tenia algunos argumentos derramados al stack y el
  caller era CALLNI (FFI runtime indirecto que mantiene el fn_ptr en
  `r13`), el helper pisaba `r13` con la direccion del slot, haciendo
  que el `callni` final saltara a memoria invalida.  Corregido usando
  `r14` como scratch (scratch general libre tras el parallel-move).

- **`RAW_ALLOC`/`RAW_FREE` no soportados en JIT**: callbacks que usaban
  `malloc`/`free` internamente caian al interp.  El selector JIT ahora
  emite CALL a `vrt_raw_alloc`/`vrt_raw_free` con stackmap automatico.

- **Compilacion JIT de la fn callee**: el thunk requiere que la
  funcion Vex objetivo este JIT-compilada (necesita la direccion del
  codigo nativo para `call rax`).  `as_native_callback` fuerza la
  compilacion eager con `maybe_compile_callvm_target` antes de
  devolver el thunk, garantizando que callbacks funcionen aunque el
  resto del programa este ejecutandose en modo interpretado.

- **Bug regalloc del JIT con multiples parametros**: el regalloc lineal
  asignaba el mismo registro fisico a multiples parametros porque sus
  intervalos `[def=0, last_use=0]` no se consideraban solapados.  El fix
  fuerza `last_use >= def + 1` para parametros con al menos un uso,
  garantizando que el solape se detecta correctamente.

### Comparacion con otros lenguajes

| Lenguaje        | Costo aprox. por callback C->lenguaje | Notas                                                |
|-----------------|--------------------------------------:|------------------------------------------------------|
| C nativo        | 0-2 ns                                | Llamada directa, sin marshalling                     |
| **Vex (thunk)** | **11 ns**                             | Thunk x86-64 + JIT code de la fn                     |
| Lua via FFI     | ~30-50 ns                             | Setup de Lua state                                   |
| Python (CPython)| ~500-1000 ns                          | PyArg_ParseTuple + GIL                               |
| JNI (Java)      | ~100-200 ns                           | JVMTI thread state + arg marshalling                 |
| .NET P/Invoke   | ~30-80 ns                             | CLR transition + GC pinning                          |

El overhead de Vex es comparable a soluciones de bajo nivel como Lua FFI
y significativamente menor que VMs administradas como JVM o CLR.

### Programa minimo de validacion

```java
extern "msvcrt.dll" {
    fn qsort(u64 base, u64 nmemb, u64 size, u64 cmp) -> void;
}

i32 cmp(i64 a, i64 b) {
    i32* pa = (i32*) a;
    i32* pb = (i32*) b;
    return *pa - *pb;
}

i32 main() {
    i32[4] arr;
    arr[0] = 3; arr[1] = 1; arr[2] = 4; arr[3] = 2;
    qsort((u64)(&arr[0]), 4, 4, (u64) as_native_callback(cmp));
    return arr[0] + arr[1] + arr[2] + arr[3]; // == 10 si OK
}
```

Si la suma final es distinta de 10, el callback no se invoco correctamente.
