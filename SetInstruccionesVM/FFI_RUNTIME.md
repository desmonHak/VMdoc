# FFI_RUNTIME — Instrucciones de contexto, FFI dinamico y runtime avanzado

Este grupo cubre las instrucciones extendidas en el rango 0x56–0x64 que proporcionan
acceso al contexto del proceso VM, soporte de FFI dinamico en runtime, concurrencia
avanzada, carga dinamica de modulos y herramientas de depuracion.

Todas usan el prefijo extendido `[0x00]` y encoding FIXED_4 (4 bytes), salvo donde
se indica.

---

## Tabla resumen

| Instruccion | opcode2 | Encoding | Descripcion |
| :----------- | :-----: | :----------- | :------------------------------------------------------- |
| `gchandle` | 0x56 | FIXED_4 REG | Convertir host_ptr a GcHandle via lookup inverso O(1) |
| `getpid` | 0x57 | FIXED_4 REG | PID encoded del proceso actual |
| `spawnon` | 0x58 | FIXED_4 REG | Spawn con scheduler hint (Here o Pinned) |
| `loadmod` | 0x59 | FIXED_4 REG | Carga dinamica de .velb adicional desde filesystem |
| `panic` | 0x5A | FIXED_4 REG | Lanzar FatalError(USER_ABORT) capturable |
| `setmethdbg` | 0x5B | FIXED_4 REG | Registrar debug info (file + linea) para un MethodInfo* |
| `fextend` | 0x5C | FIXED_4 REG | Convertir f32 -> f64 en banco ZMM |
| `fnarrow` | 0x5D | FIXED_4 REG | Convertir f64 -> f32 en banco ZMM |
| `dlopen` | 0x62 | FIXED_4 QUAD | Cargar DLL en runtime; retorna handle i64 |
| `dlsym` | 0x63 | FIXED_4 QUAD | Resolver simbolo en DLL; retorna puntero i64 |
| `callni` | 0x64 | FIXED_4 REG | Invocar funcion nativa por puntero |

---

## `gchandle` (0x56) — Convertir host_ptr a GcHandle

### Encoding

```
[0x00][0x56][byte2][0x00]
byte2 = (r_dst << 4) | r_src (convencion B)
```

- `r_src`: host_ptr a un objeto GC (apunta al payload start del ObjectHeader).
- `r_dst`: registro donde se escribe el GcHandle correspondiente.

### Comportamiento

- Hace lookup O(1) en `unordered_map<uint8_t*, GcHandle> GcHeap::ptr_to_handle_`.
- Si el ptr no corresponde a un objeto vivo en el GC heap, retorna `GC_NULL_HANDLE`.
- Necesario para pasar un host_ptr al sistema de monitors (`monenter`/`monexit`),
 que trabajan con `GcHandle` interno, no con host_ptr.

### Caso de uso

El frontend Vex emite `gchandle` al entrar en un bloque `synchronized(obj)`:

```
// synchronized (lock) { ... }
gchandle r14, r_obj // r14 = GcHandle del objeto lock
monenter r14 // adquirir el monitor
// ... body ...
tryleave
monexit r14 // liberar el monitor
```

---

## `getpid` (0x57) — PID encoded del proceso actual

### Encoding

```
[0x00][0x57][byte2][0x00]
byte2 = (r_dst << 4) | 0x00
```

- `r_dst`: registro donde se escribe el PID encoded (i64).

### Formato del PID encoded

```
bits 63-32: scheduler_id (que scheduler posee este proceso)
bits 31-0: local_pid (indice local dentro del scheduler)
```

Para PIDs locales: bit 63 = 0. Para PIDs remotos (cross-node): bit 63 = 1.

### Uso

```
getpid r0 // r0 = PID encoded del proceso actual
```

Builtin Vex: `i64 mi_pid = pid();`

---

## `spawnon` (0x58) — Spawn con scheduler hint

### Encoding

```
[0x00][0x58][byte2][0x00]
byte2 = (r_fn << 4) | r_hint
```

- `r_fn`: registro con la VA del punto de entrada del proceso hijo.
- `r_hint`: scheduler hint (int64 signed):
 - `< 0` -> **Here**: mismo scheduler que el padre (cooperativo, sin overhead cross-thread).
 - `>= 0` -> **Pinned**: scheduler = `hint % num_schedulers`.

### Diferencia con `spawn` (0xEE)

`spawn` (opcode primario 0xEE) hace round-robin automatico entre todos los schedulers.
`spawnon` permite controlar explicitamente donde vive el hijo.

### Comportamiento

- Crea un nuevo `ProcessVM` con PC = `r_fn`.
- Inicializa RSP/RBP del hijo en `0x10000000 + (local_pid % 0x1000) * 0x100000`.
- Copia el codigo (todos los executables cargados) al vm_mem del hijo via
 `Loader::copy_executables_to`.
- Retorna el PID encoded del hijo en R0.

### Uso desde Vex

```vex
i64 hijo_here = spawn here { ... }; // emite spawnon r_fn, -1
i64 hijo_pinned = spawn on(2) { ... }; // emite spawnon r_fn, 2
```

---

## `loadmod` (0x59) — Carga dinamica de modulo .velb

### Encoding

```
[0x00][0x59][byte2][0x00]
byte2 = (r_path_addr << 4) | r_path_len
```

- `r_path_addr`: VA en VM memory del path del archivo .velb (ASCII).
- `r_path_len`: longitud del path en bytes.

### Comportamiento

1. Lee el path de `vm_mem[r_path_addr, r_path_len]`.
2. Abre el archivo y parsea el `.velb` (formato version 0x2 con tabla de relocations).
3. Si hay solapamiento de VA con executables ya cargados, hace **rebase transparente**:
 asigna nueva VA en `next_dyn_base` (default `0x80000000`) y aplica relocations.
4. Copia el codigo al vm_mem de TODOS los procesos vivos.
5. Agrega el executable al pool; spawns futuros lo heredan via `copy_executables_to`.
6. Push del return address actual en el stack del proceso.
7. Salta al `init_pc` del modulo (ejecuta `__module_init` del plugin).
8. Tras el RET del init, el proceso continua en la instruccion siguiente al `loadmod`.
9. Retorna el valor de R0 que dejo el modulo en R0 (tipicamente 0 si ok).

### Rebase transparente (VERSION_VELB = 0x2)

El formato `.velb` version 2 incluye una tabla de relocations en el header:

```
Header offset +120: offset_reloc_table (u64)
Header offset +128: size_reloc_table (u32)
```

Cada entrada de la tabla es de 24 bytes:
```
+0 [8 bytes] bytecode_offset — offset dentro del code section
+8 [8 bytes] target_value — valor absoluto que fue escrito (Absolute64)
+16 [1 byte ] type — tipo de relocation
+17 [7 bytes] _pad
```

Al hacer rebase, el loader recalcula `delta = nueva_va - va_original` y patchea
cada slot cuyo `target_value` cae en el rango original del code section.

### Uso desde Vex

```vex
i64 ret = loadmodule("plugins/extra.velb"); // builtin que emite loadmod
```

---

## `panic` (0x5A) — Lanzar FatalError capturable

### Encoding

```
[0x00][0x5A][byte2][0x00]
byte2 = (r_msg_addr << 4) | r_msg_len
```

- `r_msg_addr`: VA del mensaje de error en VM memory.
- `r_msg_len`: longitud del mensaje en bytes.

### Comportamiento

1. Lee el mensaje de `vm_mem[r_msg_addr, r_msg_len]`.
2. Llama a `throw_fatal(vm, FATAL_USER_ABORT, mensaje)`.
3. Si hay un `ExceptionFrame` activo con handler compatible con `FatalError`,
 salta al handler (el proceso continua).
4. Si no hay handler, el proceso muere con `EVT_ERROR`.

### FatalKind para panic

`FATAL_USER_ABORT = 6` — identifica panics iniciados por el programador.

### Uso desde Vex

```vex
panic("Indice fuera de rango: ${idx}");

try {
    panic("algo salio mal");
} catch (FatalError e) {
    println("Capturado: kind=${e.kind}"); // kind=6
}
```

---

## `setmethdbg` (0x5B) — Registrar debug info para stack traces

### Encoding

```
[0x00][0x5B][byte2][0x00]
byte2 = (r_method << 4) | r_params
```

- `r_method`: `MethodInfo*` del metodo al que se asocia la info de debug.
- `r_params`: puntero a `SetMethDebugParams` en VM memory.

### SetMethDebugParams ABI (24 bytes)

```
+0 [8 bytes] method_ptr — MethodInfo* del metodo (redundante con r_method)
+8 [8 bytes] file_addr — VA del path del archivo fuente (ASCII)
+16 [4 bytes] file_len — longitud del path en bytes
+20 [4 bytes] start_line — linea de inicio del metodo en el fuente
```

### Comportamiento

- Registra la entrada `{source_file, start_line}` en la tabla global
 `g_method_debug` (protegida por mutex).
- Consumido por `build_stack_trace` para formatear:
 `at <method> [<class>] (<file>:<line>)`.

### Uso

El frontend Vex emite `setmethdbg` en `__module_init` justo despues de cada `defmethod`:

```
defmethod r1, r3 // anade el metodo
findmethod r5, r4 // obtiene el MethodInfo* recien creado
// construir SetMethDebugParams en stack
setmethdbg r5, r4 // registrar debug info
```

---

## `fextend` (0x5C) — Convertir f32 a f64

### Encoding

```
[0x00][0x5C][byte2][0x00]
byte2 = (r_dst << 4) | r_src (registros ZMM)
```

- `r_src`: registro ZMM con el f32 en los 4 bytes bajos.
- `r_dst`: registro ZMM donde se escribe el f64 resultante.

### Comportamiento

```cpp
dst.write_f64((double)src.read_f32());
```

La conversion es exacta para todos los valores representables en f32.

### Uso

El backend del IR Vex baja `IrOp::F32TOF64` a:

```
bitcast GP -> f0 (gp_to_zmm_bits: copiar bits del GP al ZMM via memoria)
fextend f1, f0 (convertir f32 -> f64)
bitcast f1 -> GP (zmm_to_gp_bits: copiar los bits del f64 al GP)
```

---

## `fnarrow` (0x5D) — Convertir f64 a f32

### Encoding

```
[0x00][0x5D][byte2][0x00]
byte2 = (r_dst << 4) | r_src (registros ZMM)
```

- `r_src`: registro ZMM con el f64 en los 8 bytes bajos.
- `r_dst`: registro ZMM donde se escribe el f32 resultante (en los 4 bytes bajos).

### Comportamiento

```cpp
dst.write_f32((float)src.read_f64());
```

La conversion puede perder precision si el valor f64 excede el rango exacto de f32.

---

## `dlopen` (0x62) — Cargar DLL en runtime

### Encoding

```
[0x00][0x62][byte2][byte3]
byte2 = (r_dst << 4) | r_path_addr
byte3 = (r_path_len << 4) (nibble alto = r_path_len, nibble bajo = 0)
```

- `r_path_addr`: VA del path de la DLL en VM memory (ASCII, sin NUL).
- `r_path_len`: longitud del path en bytes.
- `r_dst`: registro donde se escribe el handle (i64) de la DLL.

### Comportamiento

- Windows: `LoadLibraryA(path)` retorna `HMODULE` casteado a i64.
- Linux: `dlopen(path, RTLD_LAZY)` retorna el handle.
- Si la carga falla, lanza `FatalError(FATAL_NULL_POINTER, "dlopen failed: <path>")`.
- El handle es opaco; solo tiene sentido para `dlsym` y seguimiento interno.

### Uso

```
// En .vel assembly:
mov r1, @Absolute("data.s_kernel32") // VA del string "kernel32.dll"
mov r2, 12 // longitud del nombre
dlopen r3, r1, r2 // r3 = handle de kernel32.dll

// En Vex:
i64 handle = ffi_open("kernel32.dll");
```

---

## `dlsym` (0x63) — Resolver simbolo en DLL

### Encoding

```
[0x00][0x63][byte2][byte3]
byte2 = (r_dst << 4) | r_handle
byte3 = (r_name_addr << 4) | r_name_len
```

- `r_handle`: handle de DLL obtenido con `dlopen`.
- `r_name_addr`: VA del nombre del simbolo en VM memory.
- `r_name_len`: longitud del nombre en bytes.
- `r_dst`: registro donde se escribe la direccion del simbolo (i64).

### Comportamiento

- Windows: `GetProcAddress(handle, name)`.
- Linux: `dlsym(handle, name)`.
- Si el simbolo no existe, lanza `FatalError(FATAL_NULL_POINTER, "dlsym failed: <name>")`.
- La direccion retornada puede pasarse directamente a `callni`.

### Uso

```
dlsym r4, r3, r_name_addr, r_name_len // r4 = puntero a funcion
```

En Vex:
```vex
i64 sym_Sleep = ffi_sym(handle, "Sleep");
```

---

## `callni` (0x64) — Invocar funcion nativa por puntero

### Encoding

```
[0x00][0x64][byte2][byte3]
byte2 = (r_fn << 4) (nibble bajo = 0)
byte3 = 0x00
```

- `r_fn`: registro con la direccion de la funcion nativa (obtenida con `dlsym`).

### Calling convention

Identica a `CALLN` estatico:

```
R1..R12 = argumentos (en orden de declaracion)
R15 = argc (numero de argumentos, 0..12)
R0 = valor de retorno
```

### Comportamiento

- Invoca via `invoke_native_unchecked(fn, argc, vm)` — misma implementacion que
 el `CALLN` estatico (header inline `include/runtime/native_invoke.h`).
- Protegido con `try/catch(std::exception)` + `catch(...)` para excepciones C++.
- En MSVC: tambien protegido con `__try/__except` para crashes SEH.
- Si `argc > 12`, lanza `FatalError(FATAL_ILLEGAL_INSTRUCTION)`.

### Uso

```
// Preparar argumentos:
mov r1, 500 // primer argumento (Sleep: milisegundos)
mov r15, 1 // argc = 1
callni r4 // invocar Sleep(500)

// En Vex:
ffi_call(sym_Sleep, 500);
```

---

## Ejemplo completo: FFI dinamico desde Vex

```vex
// Abrir kernel32.dll en runtime:
i64 k32 = ffi_open("kernel32.dll");

// Resolver simbolos:
i64 sym_sleep = ffi_sym(k32, "Sleep");
i64 sym_getpid = ffi_sym(k32, "GetCurrentProcessId");

// Invocar GetCurrentProcessId() sin argumentos:
i64 pid = ffi_call(sym_getpid);
println("PID del proceso: ${pid}");

// Invocar Sleep(100ms):
ffi_call(sym_sleep, 100);
println("Desperto tras 100 ms");
```

---

## Instrucciones de contexto VM (GETPROC / GETVM / GETMGR)

Aunque estas instrucciones usan opcodes 0xC6, 0xC7, 0xC8 (parte del grupo meta-OOP),
se documentan aqui por su relacion con el FFI y el contexto runtime.

| Instruccion | opcode2 | Almacena |
| :---------- | :-----: | :------------------------------------- |
| `getproc` | 0xC6 | `ProcessVM*` del proceso actual |
| `getvm` | 0xC7 | `VM*` de la instancia VM |
| `getmgr` | 0xC8 | `ManageVM*` del manager global |

Encoding FIXED_4 REG: `[0x00][opcode2][byte2][0x00]` con `byte2 = (r_dst << 4)`.

Usadas para pasar el puntero de contexto a plugins que usan `g_api->vm_read_bytes`:

```
getproc r1 // r1 = ProcessVM* del proceso actual
mov r2, @Absolute("data.buf_vm_addr")
mov r3, 64
mov r15, 3
calln @Method("stdlib/native/io/vesta_io:vio_println")
```

---

Implementacion: `src/runtime/exec_instruction_gc.cpp` (gchandle, getproc, getvm, getmgr),
`src/runtime/exec_instruction_coro.cpp` (getpid, spawnon),
`src/loader/loader.cpp` (loadmod via Loader::load_module_dynamic),
`src/runtime/exception_runtime.cpp` (panic, setmethdbg),
`src/runtime/exec_instruction_float.cpp` (fextend, fnarrow),
`src/runtime/exec_instruction_meta.cpp` (dlopen, dlsym, callni).

Ver tambien: [[Vex/FFI]], [[GC/GC]], [[NativePluginAPI]],
[[NativeCall (CallN)]], [[CORO]], [[Vex/Async]]
