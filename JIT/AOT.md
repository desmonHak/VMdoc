# Plan AOT — Compilacion nativa standalone

VestaVM puede ejecutar bytecode `.velb` de tres maneras: **interpretado**
(siempre disponible), **JIT-compilado** en runtime (C1 baseline funcional;
C2 optimizante en plan), o **AOT-compilado** a binario nativo standalone
(en plan).

Este documento describe el plan completo de la fase AOT, que entregara
ejecutables nativos `.exe` (Windows PE), `.elf` (Linux/POSIX ELF) y
otros formatos (UEFI, raw binary para firmware) sin necesidad de
mantener la VM al lado.

## Filosofia: tres tiers de deployment, una sola fuente

El objetivo es que el mismo codigo Vex pueda compilarse a tres modos
distintos de deployment, eligiendo el tradeoff entre features del
lenguaje y tamano/dependencias del binario:

```
+--------------+-----------------+--------------+----------------+
| Tier         | Runtime         | Tamano       | Caso de uso    |
+--------------+-----------------+--------------+----------------+
| Vex Full     | libvesta_rt     | 3-5 MB       | CLI, servers,  |
|              | linkado dinam.  | (link dynam) | apps managed   |
+--------------+-----------------+--------------+----------------+
| Vex Embed    | Mini-runtime    | 500KB-1MB    | CLI tools,     |
|              | embebido static | (standalone) | ETL, scripts   |
+--------------+-----------------+--------------+----------------+
| Vex Bare     | Solo libc/no_std| 50-200 KB    | Kernels, dev   |
|              | (sin runtime)   | (standalone) | OS, drivers,   |
|              |                 |              | hot-path libs, |
|              |                 |              | embedded, FW   |
+--------------+-----------------+--------------+----------------+
```

Modelos analogos:

- **Vex Full** ≈ Go (runtime statically linked), Java con JRE,
  C# con CoreCLR.
- **Vex Embed** ≈ Rust con std, Swift native, OCaml native.
- **Vex Bare** ≈ Zig minimal, Rust `#![no_std]`, C/C++ embedded,
  Forth, Ada-pure.

## Casos de uso target

Vex Bare desbloquea casos inaccesibles para lenguajes managed:

1. **Desarrollo de sistemas operativos** propios.  Kernel + drivers
   compilados a un ELF/PE bootable.  Sin GC heap interfiriendo con
   IRQ handlers.  Sin allocator runtime que falle en initramfs.
2. **Drivers de kernel** (Windows `.sys`, Linux `.ko`).  El loader del
   kernel rechaza binarios con dependencias externas.
3. **Firmware embedded** (ARM Cortex-M, RISC-V).  Sin libc dinamica;
   stubs `_exit/_write` provistos por el board.
4. **Bootloaders** + UEFI applications.
5. **Hot-path libs** (audio DSP, image codecs, crypto).  Binarios
   `.dll`/`.so` cargables en programas C/C++/Python/Rust con cero
   overhead vs C++ optimizado.
6. **Hypervisors / unikernels** estilo MirageOS.
7. **WebAssembly minimal** sin runtime browser-side.
8. **Microservices serverless** (AWS Lambda, Cloudflare Workers) donde
   el tamano del binario es critico para cold-start.

Vex Full y Embed cubren el caso comun de aplicaciones desktop/server
con UI/networking/parseo complejo.

## Compatibility matrix de features por tier

| Feature del lenguaje                  | Bare | Embed | Full |
|:--------------------------------------|:----:|:-----:|:----:|
| Aritmetica int/float, bool, char      |  Si  |  Si   |  Si  |
| `T*`, `&x`, `*p`, `VirtualPtr<T>`     |  Si  |  Si   |  Si  |
| Structs value-type, bit fields        |  Si  |  Si   |  Si  |
| Arrays nativos `T[N]`, `T[]`          |  Si  |  Si   |  Si  |
| Functions + closures sin captures     |  Si  |  Si   |  Si  |
| `malloc/free` via libc                |  Si  |  Si   |  Si  |
| FFI extern (cualquier lib externa)    |  Si  |  Si   |  Si  |
| Inline asm `@Asm` / `asm{}`           |  Si  |  Si   |  Si  |
| Optional/Result builtins (SRET stack) |  Si  |  Si   |  Si  |
| Pattern matching ADTs (value-types)   |  Si  |  Si   |  Si  |
| `comptime` metaprogramacion           |  Si  |  Si   |  Si  |
| Generics (monomorfizacion compile-t.) |  Si  |  Si   |  Si  |
| `typedef new` + `cstring`             |  Si  |  Si   |  Si  |
| Borrow checker compile-time           |  Si  |  Si   |  Si  |
| Modulos + cache + paralelo            |  Si  |  Si   |  Si  |
| `unique<T>` / `shared<T>` con free    | Si*  |  Si   |  Si  |
| `panic("msg")` -> fputs+exit          |  Si  |  Si   |  Si  |
| Strings managed (StringObject GC)     |  NO  |  Si   |  Si  |
| Classes con `new` + GC                |  NO  |  Si   |  Si  |
| `try/catch/throw`                     | Si** |  Si   |  Si  |
| `synchronized` / `monitor`            | Si***|  Si   |  Si  |
| Closures con captures que escapan     |  NO  |  Si   |  Si  |
| `@Async` / `await` / `Future<T>`      |  NO  |  NO   |  Si  |
| `spawn` / `rspawn`                    |  NO  |  NO   |  Si  |
| `msgsend` / `msgrecv`                 |  NO  |  NO   |  Si  |
| Reflexion (`forName`, `getClass`)     |  NO  |  NO   |  Si  |
| AOP (`@Aspect`, etc.)                 |  NO  |Si**** |  Si  |
| `loadmodule(path)`                    |  NO  |  NO   |  Si  |
| Memoria compartida cross-process      |  NO  |  NO   |  Si  |

\* `shared<T>` en Bare degrada a refcount manual (sin GC).
\** Bare con unwinding nativo: requiere emit de `.pdata/.xdata` PE o
    `.eh_frame` ELF (opcional; alternativa: setjmp/longjmp via
    `@UnwindImpl`).
\*** Bare via FFI a pthread / WinAPI; no usa monitor del runtime.
\**** Embed soporta AOP estatico (compile-time weaving); no AROUND dinamico.

## Mapping IR -> nativo

El SSA IR ya esta abstraido del runtime VM en su forma estructural.  El
"runtime" entra solo en QUE HACE cada IR op a nivel bajo.  Para Vex
Bare, las IR ops se clasifican en tres grupos:

**Pure native** (1:1 a instrucciones x86-64/ARM):

```
CONST, MOV, ADD/SUB/MUL/DIV/MOD/AND/OR/XOR
SHL/SHR/SAR/NEG/NOT/POPCNT/CLZ/CTZ/BYTESWAP/ROTL/ROTR
CMP_* + BR_COND  (fusionado a cmpjmp en bytecode -> cmp+jcc nativo)
LOAD/STORE       (mov [reg+disp], reg)
ALLOCA           (sub rsp, N -- host stack, no VM stack)
RET/JMP/PHI/BITCAST/SEXT/ZEXT/TRUNC/CAST
FADD/FSUB/FMUL/FDIV/FCMP/FSQRT/FABS/FNEG  (XMM)
ITOF/UITOF/FTOI/FTOUI/F32TOF64/F64TOF32   (CVTSI2SD/CVTTSD2SI)
INLINE_ASM       (bytes verbatim del encoder)
```

**Libc-mapped** (lower a call a simbolo libc):

```
RAW_ALLOC          -> call malloc / call HeapAlloc
RAW_FREE           -> call free   / call HeapFree
CALL <fn>          -> call <symbol>     (linker resuelve)
CALLN <lib:fn>     -> call <symbol>     (extern via dlopen / ya cargada)
print/println      -> call write / fputs + libc-string
panic("msg")       -> fputs(msg, stderr); exit(1)
```

**Runtime-dependent** (rechazado en Bare con error claro;
disponible en Embed y Full):

```
NEWOBJ, GC_ALLOC, GC_DEREF, GC_PROMOTE, GC_DEMOTE
CALLVIRT, CALLM, CALLSUPER  (a menos que devirt total compile-time)
STRMAKE, STRCAT, STRCMP, STRLEN, STRRAW
MONENTER, MONEXIT, MONWAIT  (en Bare: usar pthread/WinAPI via FFI)
SPAWN, RSPAWN, MSGSEND, MSGRECV
FUTURE, AWAIT, FULFILL
LOADMOD, UNLOADMOD
FINDCLASS, FINDMETHOD, FINDFIELD
```

Mensaje de error tipico al intentar Bare con feature no soportada:

```
foo.vex:42:5: error: 'new Foo()' requiere GC heap; no compilable
  en --target=bare.
  hint: usa @c struct Foo { ... } (value-type) + asignacion explicita
        en stack o malloc, O cambia a --target=embed para mantener
        la semantica de class con GC managed.
```

## Pipeline AOT

```
main.vex
  | frontend (lexer + parser + typecheck + lower)
SSA IR
  | ir_optimizer (mismas pases que JIT: DCE/CSE/copy-prop/LICM/inline)
SSA IR optimizado
  | analisis pure_native
  +-- todas las ops son nativas o libc-mapped? -> continue
  +-- alguna op runtime-dependent? -> error con sugerencia
  | lower a MachineIR
MachineIR target-aware (x86-64 / aarch64 / armv7m / riscv64)
  | regalloc real (linear scan target-aware)
MachineIR con regs fisicos asignados + stackmaps
  | x86_encoder (o ARM encoder / RISC-V encoder)
bytes maquina + tabla de relocs sin resolver
  | object emitter (LibPEparse vendored)
program.obj (COFF Windows) / program.o (ELF Linux/POSIX)
  | linker propio
program.exe (PE64) / program (ELF64) / program.efi (UEFI PE)

Dependencias del binario final:
  - libc: msvcrt.dll / libc.so.6 / mingwex
    (NULL en --target=freestanding para kernels)
  - libs externas que el .vex declara con extern "lib" {}
  - NADA del runtime de Vex (no libvesta_rt.dll, no .so, no nada).
```

## Extensibilidad: el usuario implementa lo que falta

Vex Bare deliberadamente deja huecos que el usuario puede llenar segun
su caso de uso.  El modelo: el lenguaje provee primitivas; el usuario
provee implementaciones cuando el subset Bare no cubre algo necesario.

### 1. Custom allocator hooks (`@AllocatorOverride`)

```vex
@AllocatorOverride
fn __vex_malloc(u64 size) -> u8* {
    // Tu impl: bump allocator, slab, freelist...
    return kmalloc(size);  // FFI al kernel
}

@AllocatorOverride
fn __vex_free(u8* ptr) -> void { kfree(ptr); }
```

El frontend reescribe TODOS los `RAW_ALLOC` del programa al simbolo
user-provided.  Permite uso en kernels (kmalloc/kfree), embedded
(heap estatico), o programas que necesiten arenas.

### 2. Custom panic handler (`@PanicHandler`)

```vex
@PanicHandler
fn __vex_panic(u8* msg, u64 len) -> never {
    uart_write(msg, len);
    cpu_halt();
}
```

Reemplaza el default `fputs(stderr) + exit(1)`.  En kernel mode,
probablemente quieres halt + BSOD-style.

### 3. Custom GC (`@CustomGC`)

```vex
@CustomGC
namespace mi_gc {
    public fn alloc_object(u64 size, ClassInfo* cls) -> Object*;
    public fn deref(GcHandle h) -> u8*;
    public fn collect() -> void;
    public fn write_barrier(Object* obj, u32 field_offset, Object* val) -> void;
}
```

En lugar del GC del libvesta_rt, usa el tuyo (refcount, bump,
region-based, Boehm conservativo).  Para Vex Embed con caso de uso
especifico (real-time = sin pause; embedded = sin malloc).

### 4. Custom string impl (`@StringImpl`)

```vex
@StringImpl
namespace utf8_strings {
    public type StringObject = struct { u8* data; u64 len; u64 cap; };
    public fn make(u8* p, u64 len) -> StringObject*;
    public fn concat(StringObject* a, StringObject* b) -> StringObject*;
    // ...
}
```

Permite UTF-8 strings ligeros en Bare en lugar del StringObject
GC-managed.

### 5. Custom sync primitives (`@SyncImpl`)

```vex
@SyncImpl
namespace spinlock_sync {
    public fn enter(Object* obj) -> void {
        while (!cas(obj.lock, 0, 1)) pause();
    }
    public fn exit(Object* obj) -> void { obj.lock = 0; }
}
```

En kernels usar spinlocks; en userspace pthread mutex; en embedded
disable IRQ.

### 6. Custom unwinding (`@UnwindImpl`)

```vex
@UnwindImpl
namespace setjmp_unwind {
    public fn try_enter(JmpBuf* buf) -> i32 { return setjmp(buf); }
    public fn throw(i32 code, JmpBuf* target) -> never {
        longjmp(target, code);
    }
}
```

En lugar de `.pdata`/`.eh_frame` (que requieren unwinding nativo
estandar PE/ELF), usar `setjmp/longjmp` puro.  Mas lento pero
portable a cualquier toolchain.

### 7. Runtime-as-library

Vex Embed se distribuye tambien como `libvesta_embed.lib`/`.a`
linkable como cualquier lib externa.  Si el usuario quiere parte
del GC managed en Bare, hace:

```vex
extern "libvesta_embed" { fn vrt_gc_alloc(...) }
```

y linkea contra ese `.lib`.  Asi se transforma Bare en algo
intermedio entre Bare puro y Embed completo, eligiendo solo las
piezas del runtime que necesita.

## Manifest del proyecto en Vex Bare

```toml
# vex.toml para programa Bare (e.g. kernel)
[package]
name    = "my_kernel"
version = "0.1"

[build]
target          = "x86_64-unknown-none"  # freestanding
mode            = "bare"
no_libc         = true                    # kernel: no libc
no_unwind       = true                    # sin .eh_frame (usa setjmp)
custom_allocator = "kalloc"                # symbol nombre
custom_panic    = "kpanic"
entry_point     = "_start"                # o "kmain", "_start_efi", etc.

[output]
format = "elf64"   # o "pe64", "efi", "raw_binary"
link_script = "linker.ld"  # opcional, custom linker script

[dependencies]
# Sin dependencias = solo lo que tu codigo y libc/kernel proveen
```

## Integracion con PGO

El sistema de profile counters runtime (ver
[Profiling.md](./Profiling.md)) genera ficheros `.vprof` que **alimentan
tanto el JIT C2 como el AOT**.  El mismo perfil sirve ambos backends:

- **JIT con PGO**: warm-start de funciones hot del run anterior.
  Especulacion agresiva (devirt, type narrowing) con deopt al
  interprete si los guards fallan.

- **AOT con PGO**: decisiones especulativas hard-coded en el `.exe`.
  Devirtualizacion monomorfica, hot/cold splitting (`.text.hot` vs
  `.text.cold`), branch layout segun taken counts, loop unrolling
  agresivo en hot loops, inlining basado en call counts.  Si un
  guard falla, fallback nativo INLINE en el codigo (rama lenta con
  dispatch normal via vtable) -- no hay deopt al interp porque no
  hay interp en el binario standalone.

Flujo PGO en AOT:

```bash
# 1. Build instrumentado (incluye hooks runtime de profile)
vex --aot main.vex -o main_inst.exe --profile-gen

# 2. Ejecutar con workload representativo
./main_inst.exe --workload typical
# -> genera main.vprof

# 3. Build optimizado con PGO
vex --aot main.vex -o main.exe --profile-use=main.vprof
```

Optimizaciones AOT que **solo** son posibles con perfil:

| Optimizacion | Sin PGO | Con PGO |
|:---|:---|:---|
| Devirt monomorfica | No (dispatch generico) | Direct call + guard |
| Hot/cold splitting | Codigo unido | Hot path en `.text.hot`, cold en `.text.cold` |
| Layout de branch | Default fall-through al `then` | Hot branch fall-through (predicción CPU) |
| Inline policy | Heuristica por tamano + recursion | Inline si called >N veces |
| Loop unrolling factor | Conservador (2 o 4) | Agresivo (8/16) en hot loops; nada en frios |
| Stack-alloc por escape | Conservador | Hot allocs probados primero |
| PIC inline estable | No (asume megamorfico) | Inline N entries del perfil + guard |
| Constant propagation | No | Si una var es siempre `42`, propagar + guard |

## Resumen estrategico

Vex Bare NO es un downgrade del lenguaje.  Es un **modo de deployment
alternativo** del mismo compilador.  La misma fuente puede compilar
a Full (con runtime) para apps managed, a Embed (runtime minimal)
para CLI tools, o a Bare (sin runtime) para sistemas low-level.  Esa
eleccion es **politica del programador**, no politica del lenguaje.

A cambio del subset reducido, Vex Bare entrega:

- Tamano binario 50-200 KB (vs 5MB Go-style).
- Cold start <1ms (vs ~50ms con runtime managed).
- Sin runtime tax: cero coste de GC pause, scheduler scheduling,
  monitor overhead.
- Compilacion AOT a `.exe`/`.elf` standalone que el sistema operativo
  ejecuta directamente.
- Compatible con dev OS, drivers de kernel, embedded firmware,
  hot-path libs distribuidas como `.dll`/`.so`.
- PGO hiperoptimizado via `.vprof` -- en hot paths puros, paridad
  con `gcc -O3 -fprofile-use`.
- Extensibilidad via `@AllocatorOverride`, `@PanicHandler`, `@CustomGC`,
  `@StringImpl`, `@SyncImpl`, `@UnwindImpl` -- el programador
  implementa lo que necesite del subset faltante segun su caso de
  uso especifico.
