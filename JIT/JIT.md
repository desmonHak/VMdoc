# JIT C1 baseline de VestaVM

Documentacion tecnica del compilador Just-In-Time C1 (template-based, sin
regalloc). Cubre arquitectura, subsistemas, pipeline de compilacion, y
limitaciones actuales.

> **Modulos**: `include/jit/`, `src/jit/`, `include/vesta_rt/`.

---

## Indice

1. [Que es el JIT C1 baseline](#1-que-es-el-jit-c1-baseline)
2. [Subsistemas principales](#2-subsistemas-principales)
3. [Pipeline de compilacion](#3-pipeline-de-compilacion)
4. [MachineIR](#4-machineir)
5. [Encoder x86-64 hand-rolled](#5-encoder-x86-64-hand-rolled)
6. [Selector (IR -> MachineIR)](#6-selector-ir---machineir)
7. [CodeCache](#7-codecache)
8. [JitRegistry y stackmaps](#8-jitregistry-y-stackmaps)
9. [Bridge interp -> JIT](#9-bridge-interp---jit)
10. [Auto-JIT trigger](#10-auto-jit-trigger)
11. [Cobertura del Selector](#11-cobertura-del-selector)
12. [Flags y env vars de control](#12-flags-y-env-vars-de-control)
13. [Limitaciones actuales](#13-limitaciones-actuales)
14. [API publica vesta_rt](#14-api-publica-vesta_rt)

---

## 1. Que es el JIT C1 baseline

El JIT C1 (tomado de la nomenclatura HopSpot) es un compilador **template-based
sin asignacion de registros real**: cada SSA value tiene un slot fijo en el
stack frame, y cada IR op se traduce a una secuencia template fija (load
operandos -> compute -> store result). Compile-time es microsegundos y el
speedup esperado es 10-20x sobre el interprete en metodos hot.

**Diseno deliberado**: simplicidad sobre optimizacion. La cobertura es ~52%
de metodos reales (el resto cae a interp). C2 optimizing (D.8) anyadira
regalloc, inline, escape analysis y especulacion para acercarse a paridad
con C nativo.

---

## 2. Subsistemas principales

| Modulo                          | Responsabilidad                                  |
| :------------------------------ | :----------------------------------------------- |
| `jit/code_cache.h/.cpp`         | Aloca memoria RWX para emit de codigo nativo     |
| `jit/runtime_entries.h/.cpp`    | Tabla resuelta de wrappers C `vrt_*`             |
| `jit/interp_jit_bridge.h`       | Trampoline para `enter_jit(fn, proc)`            |
| `jit/jit_compiler.h/.cpp`       | API `JitCompiler::compile(ir_fn) -> JitFn`       |
| `jit/machine_ir.h`              | Tipos de MachineIR (MInstr 32 bytes)             |
| `jit/selector.h/.cpp`           | IR SSA -> MachineIR                              |
| `jit/x86_encoder.h/.cpp`        | MachineIR -> bytes x86-64                        |
| `jit/jit_registry.h/.cpp`       | Lookup `pc -> stackmap`, registro de JitFns      |
| `jit/auto_jit.h/.cpp`           | `maybe_compile_method` (trigger automatico)      |
| `vesta_rt/public.h`             | API C estable para JIT y futuro AOT              |
| `vesta_rt/abi.h`                | Constantes ABI con `static_assert`               |

---

## 3. Pipeline de compilacion

Cuando se decide compilar un metodo:

```text
MethodInfo (cuando invocation_count >= threshold)
        |
        v  Lookup en Executable::ir_lookup
        |  (clave: ClassName__methodName o ClassName__ctor)
        v  IrFunction (de la seccion @ir del .velb)
        |
        v  Selector::select(ir_fn, mode=VM_ABI) -> MFunction
        |  - Reusa interp slots para SSA values (template C1).
        |  - Emite SAFEPOINT en cada back-edge.
        |  - Construye stackmaps por cada CALL/SAFEPOINT.
        |  - Genera labels stable para BR/BR_COND.
        |
        v  X86Encoder::encode(mfunc) -> Vector<uint8_t> + fixups
        |  - Hand-rolled (no Keystone): ~50 ns/instr vs 10 us con Keystone.
        |  - Encoding directo de ALU/MOV/CMP/Jcc/CALL/etc.
        |  - Resuelve fixups (forward branches) en pasada final.
        |
        v  CodeCache::alloc(size) + memcpy + commit (flush icache)
        |
        v  JitRegistry::register_function(start, end, stackmaps, frame_size)
        |
        v  method->jit_code = JitFn  (puntero al codigo emitido)
        |
        +-> Proximo CALLVIRT lo dispatcha directo (sin pasar por interp)
```

---

## 4. MachineIR

Tipos cache-friendly disenados para microarquitectura moderna.

### `MInstr` (32 bytes exactos)

```cpp
struct MInstr {
    MOp        op;          // 1 byte: ADD, SUB, MOV, CMP, Jcc, CALL, ...
    MCond      cc;          // 1 byte: condition code (alineado con SETcc/Jcc x86)
    uint8_t    pad[2];
    MOperand   dst;         // 8 bytes
    MOperand   src1;        // 8 bytes
    MOperand   src2;        // 8 bytes (para ops 3-address; ignorado en 1-2 ops)
    uint16_t   flags;       // index de stackmap, etc.
    uint16_t   pad2;
};

static_assert(sizeof(MInstr) == 32, "MInstr cache-line packed");
```

2 instrucciones por cache line de 64 bytes (verificado en compile-time).

### `MOperand` (8 bytes)

```cpp
struct MOperand {
    enum Kind : uint8_t { NONE, REG, IMM, MEM, LABEL };
    Kind   kind;
    uint8_t reg;     // si kind == REG: GP 0..15, XMM 16..31
    uint8_t width;   // 1/2/4/8 bytes; index reg packed aqui para MEM
    uint8_t flags;   // scale para MEM (1/2/4/8)
    int32_t value;   // imm valor / displacement de MEM / label id
};
```

Inmediatos 64-bit van por `MFunction::imm64_pool` (no inflar cada MInstr).

### `MBlock` / `MFunction`

Vectores contiguos, indices `uint16_t` / `uint32_t` (no punteros) para
predecessores/sucesores. Cero fragmentation, cero heap chasing.

### Labels y fixups

```cpp
using MLabelId = uint32_t;

struct MFixup {
    uint32_t code_offset;   // byte offset del JCC/JMP/CALL
    MLabelId target_label;
    uint8_t  size;          // 1 = rel8, 4 = rel32
};
```

`resolve_fixups` patchea todas las branches rel32 en una sola pasada final
con range check int32.

---

## 5. Encoder x86-64 hand-rolled

Emite bytes directamente sin Keystone. **Decision tecnica**: ~50 ns/instr vs
~10 us con Keystone, sin nuevas dependencias.

### Cobertura

- **MOV**: reg-reg, reg-imm32, reg-imm64 (via pool), reg-mem, mem-reg, mem-imm32.
- **LEA**: para aritmetica de punteros.
- **PUSH / POP**.
- **ALU**: ADD/SUB/AND/OR/XOR (con optimizacion imm8 cuando cabe -> ahorra
  3 bytes).
- **IMUL**: formas r/r y r/r/imm.
- **SHL/SHR/SAR**: con imm8.
- **NEG/NOT**.
- **CMP/TEST**.
- **SETcc, CMOVcc**.
- **JMP rel32, Jcc rel32**.
- **CALL rel32/reg/mem**.
- **RET, INT3**.
- **IDIV + CQO** (para DIV/MOD signed).
- **SAFEPOINT poll** (expanded a 21 bytes: cmp byte [rbx+0],0 + je skip + mov
  rdi/rcx,rbx + mov rax,imm64 + call rax + skip:).

### Helpers de bajo nivel

```cpp
rex_byte(W, R, B, X)          // REX prefix con detection automatica
modrm(mod, reg, rm)
sib(scale, index, base)
emit_modrm_mem(reg_or_opcode, mem)
  // maneja casos especiales: SIB requerido para r/m=4 o RSP/R12 base,
  // disp32 forzado para r/m=5 base RBP/R13 con mod=0.
```

### Performance

Reserva capacidad anticipada (`out.reserve(N*6)` con promedio empirico 6
bytes/instr). Para un metodo de 100 IR ops, encoding total < 10 us.

---

## 6. Selector (IR -> MachineIR)

Template C1-style minimal. Cada SSA value tiene slot stack fijo (offset =
`-8 * (vid + 1)` desde RBP). Cada `IrInstr` emite la plantilla:

```text
LOAD operandos a scratch (RAX/RCX/RDX)
COMPUTE
STORE result al slot
```

### Modos

- **`SelectorMode::NATIVE_ABI`**: args en rdi/rsi/rdx/rcx/r8/r9 (SysV) o
  rcx/rdx/r8/r9 (Win64), return en rax. Usado en tests sin ProcessVM.
- **`SelectorMode::VM_ABI`**: args en `proc->registers.regs[1..N+1]`,
  return en `regs[0]`. Usado en runtime.

### Prologue / epilogue (VM_ABI)

```asm
; prologue
push rbp
mov  rbp, rsp
sub  rsp, frame_size     ; alineado a 16 bytes + shadow space Win64 si aplica
push rbx
mov  rbx, rdi            ; rdi/rcx = ProcessVM*; movemos a RBX (callee-saved)
                         ; -> persiste todo el metodo sin spill

; cargar params: regs[1..N] a slots locales
mov  rax, [rbx + REGISTERS_OFFSET + 1*8]
mov  [rbp - 8], rax
...

; epilogue
mov  [rbx + REGISTERS_OFFSET + 0], rax    ; return value a regs[0]
mov  rsp, rbp
pop  rbp
pop  rbx
ret
```

### IR ops soportadas

Cubre v1 actual:

- **NOP / MOV / CONST**.
- **ADD/SUB/MUL** (via IMUL), **AND/OR/XOR**, **NEG/NOT**.
- **DIV/MOD signed** (IDIV + CQO).
- **SHL/SHR/SAR con imm const** (variable shift via CL no soportado).
- **SEXT/ZEXT/TRUNC/CAST/BITCAST** entre tipos enteros.
- **LOAD/STORE** (memoria cruda).
- **CMP_EQ/NE/LT/GT/LE/GE** + variants signed/unsigned con `cond_for_cmp_op`
  table -> MCond. Patron correcto: `XOR rax,rax; CMP rcx,rdx; SETCC al`
  (zero PRIMERO para no destruir flags).
- **BR/BR_COND/RET** + labels stable por IR block.
- **CALL** a runtime entries con stackmap automatico.
- **CALLVIRT con inline dispatch** (5 loads + comparaciones para llegar a
  `method->jit_code`, luego call directo; fallback a `vrt_callvirt` si
  jit_code es null).
- **CALLM, CALLCLOSURE**.
- **NEWOBJ** (via `vrt_gc_alloc`), **GC_ALLOC**, **GC_DEREF**,
  **GC_HANDLE_FOR_PTR**.
- **ALLOCA** (sub rsp, N; mov [dst], rsp).
- **raw_asm pattern recognition** para los 6 textos mas frecuentes:
  `gchandle`, `gcderef`, `subsp/addsp`, `mov [r], imm`, `findclass`,
  `newobj`. Con symbol resolution para `@Absolute("section.label")`.

IR ops NO soportadas marcan `out_unsupported=true` -> el JIT decae a interp
para ese metodo + warning detallado con `--jit-warn`.

### PHI elimination

PHI nodes se eliminan emitiendo phi-copies al final de cada predecessor
(parallel-move sobre slots de stack).

---

## 7. CodeCache

`include/jit/code_cache.h` + `src/jit/code_cache.cpp` (~250 LOC).

### Alocacion

- **Windows**: `VirtualAlloc(MEM_RESERVE+MEM_COMMIT, PAGE_EXECUTE_READWRITE)`.
- **POSIX**: `mmap(PROT_READ|PROT_WRITE|PROT_EXEC, MAP_PRIVATE|MAP_ANON)`.

Chunks de 1 MiB (configurable), hasta 64 MiB total (configurable).

### API

```cpp
uint8_t* alloc(size_t size, size_t align = 16);
  // Bump pointer + nuevo chunk si OOM en el actual.

void commit(uint8_t* ptr, size_t size);
  // Flush instruction cache:
  // - Windows: FlushInstructionCache(GetCurrentProcess(), ptr, size)
  // - POSIX:   __builtin___clear_cache(ptr, ptr + size)

void invalidate(uint8_t* ptr, size_t size);
  // Rellena con 0xCC (INT3) para que cualquier ejecucion post-deopt crashee
  // limpiamente con FATAL_ILLEGAL_INSTRUCTION.

bool contains(uint8_t* ptr) const;
size_t used_bytes() const;
size_t chunk_count() const;
```

### Modo W^X

Reservado para Phase E. Hoy es RWX permanente; futuras versiones haran
`mprotect` para alternar RW (durante emit) <-> RX (durante ejecucion) por
chunk para cumplir con politicas de seguridad endurecidas (macOS, Linux
hardened).

---

## 8. JitRegistry y stackmaps

### JitRegistry

Singleton thread-safe que mapea `(ip range) -> JitFunctionInfo`:

```cpp
struct JitFunctionInfo {
    uint8_t*           code_start;
    uint8_t*           code_end;
    uint32_t           frame_size;
    std::vector<Stackmap> stackmaps;
    std::string        name;
};

// API:
void register_function(code_start, code_end, stackmaps, frame_size, name);
void unregister_function(code_start);
JitFunctionInfo* lookup(uint8_t* rip);                  // O(log N) binary search
Stackmap*        lookup_stackmap(uint8_t* rip);         // mayor con pc_offset <= rip_offset
size_t           size() const;
```

Sorted by `code_start` -> binary search O(log N) en lookup.

### Stackmaps

Cada CALL y SAFEPOINT genera un `Stackmap`:

```cpp
struct StackmapSlot {
    int16_t rbp_offset;       // offset desde RBP del slot
    uint8_t gc_kind;          // HANDLE, HOSTPTR, STRING
    uint8_t pad;
};

struct Stackmap {
    uint32_t              pc_offset;   // offset desde code_start
    std::vector<StackmapSlot> slots;
};

enum StackmapGcKind : uint8_t {
    HANDLE  = 0,   // GcHandle (uint32)
    HOSTPTR = 1,   // host_ptr crudo al payload
    STRING  = 2    // StringObject*
};
```

El GC, durante `major_gc`, usa `scan_jit_roots_precise` que walks la cadena
RBP nativa desde la base del stack. Por cada JIT frame encontrado via
`lookup_stackmap(rip)`, itera sus slots y marca cada handle/host_ptr como
root.

**Modo additive con conservativo (Fase 1)**: el scan precise corre EN
PARALELO con el conservativo. Si un stackmap omite un slot por bug, el
conservativo lo cubre como root accidental. Cero use-after-free posible
durante el periodo de validacion.

Metricas en `GcStats`:
- `precise_roots_marked`
- `precise_frames_scanned`
- `conservative_roots_marked`

---

## 9. Bridge interp -> JIT

`include/jit/interp_jit_bridge.h` (~100 LOC, header-only):

```cpp
using JitFn = uint64_t(*)(vrt_proc* proc);

inline uint64_t enter_jit(JitFn fn, vrt_proc* proc) {
    return fn(proc);   // SysV: proc en rdi. Win64: proc en rcx.
}

uint64_t return_from_jit(vrt_proc* proc, uint64_t pc);
  // Stub para OSR / deopt al interprete (Phase D.5).
```

### Convencion VM_ABI

Para que el codigo JIT-eated pueda llamar y ser llamado uniformemente:

- **Entry**: `ProcessVM*` en rdi (SysV) o rcx (Win64). Args VM van en
  `proc->registers.regs[1..N]`. argc en `regs[15]`.
- **Exit**: return value en `proc->registers.regs[0]`.

Esto permite que un metodo JIT invoque otro metodo JIT (o caiga a interp) sin
traduccion de calling convention en cada cross-call.

---

## 10. Auto-JIT trigger

`include/jit/auto_jit.h` + `src/jit/auto_jit.cpp`.

### Hook desde el interprete

`runtime::g_callvirt_post_hook` es un function pointer global registrado al
arranque del binario `vm`:

```cpp
// En main.cpp:
runtime::g_callvirt_post_hook = &jit::maybe_compile_method;
```

El interprete (`exec_instr_callvirt`) lo invoca tras incrementar el counter
de invocaciones del metodo. Esto desacopla `vesta_rt` del JIT (vesta_rt es
self-contained y reutilizable por AOT futuro).

### `maybe_compile_method`

```cpp
void maybe_compile_method(runtime::VM* vm, MethodInfo* method);
```

Fast path: `if (g_jit_threshold == UINT32_MAX || method->jit_code != nullptr ||
method->invocation_count < threshold) return;` (~1 ns, branch predicted).

Slow path:
1. Construye lookup key `<ClassName>__<methodName>` desde `MethodInfo::owner_class->name`
   + `MethodInfo::name`. Fallback critico: para constructores (donde
   `MethodInfo::name == ClassName`), tambien prueba `<ClassName>__ctor`.
2. Busca en `Executable::ir_lookup` de cada executable cargado.
3. Si encuentra, compila con `SelectorMode::VM_ABI`.
4. Si exito, setea `method->jit_code = res.fn`.
5. Si falla (`unsupported=true` o IR no encontrado), setea
   `method->invocation_count = UINT32_MAX` para evitar reintentos infinitos.

### Subsistema lazy

El subsistema JIT (`CodeCache` + `RuntimeEntries` + `JitCompiler`) se
construye al primer trigger via `std::call_once`. Programas que jamas alcanzan
el threshold no pagan el coste de inicializar el JIT (10-50 KB de VM
reservada para code cache).

---

## 11. Cobertura del Selector

Estado actual: **~52% de metodos reales** JIT-compilados (medido sobre los
143 ejemplos de `examples_codes_vex/`).

### Programas class-heavy donde TODOS los metodos JIT-compilan

- `12_clases_basico` (2/2)
- `15_herencia_basica` (3/3)
- `21_interfaces_basico` (2/2)
- `27_generics_box` (4/4)
- `28_generics_avanzado` (6/6)
- `42_propiedades_basico` (3/3)
- `60_stack_iface` (2/2)
- `67_fatal_isolation` (2/2)

### Limitaciones que hacen caer a interp

1. **raw_asm complejo** (sync/findclass/monenter prologues): patron no
   reconocido por el mini-parser de raw_asm. Cubierto parcialmente con
   pattern recognition de los 6 textos mas frecuentes.
2. **Float arith** (FADD/FMUL/FCMP/...): el selector v1 no soporta XMM regs;
   los metodos numericos con f64 caen a interp.
3. **CALLN/CALLM/CALLCLOSURE a usuario** sin runtime entry: soportados con
   resolver via symbol section pero algunos patrones siguen incompletos.
4. **Eager-compile de `main`**: DESACTIVADO temporalmente porque cuando los
   callees no son compilables, el JIT-eated main crashea. Pendiente: trampoline
   JIT->interp para callees no compilables.

### Performance medida

| Bench               | Interp      | JIT (-m jit) | Speedup |
| :------------------ | ----------: | -----------: | ------: |
| `bench_jit_method`  | 1700 ms     | **85 ms**    | **20×** |
| `bench_callvirt_hot`| 360 ms      | 332 ms (slow path para callees) | 1.1× |

El JIT da maximo speedup cuando el hot loop esta DENTRO de un metodo virtual
(que SI compila) en lugar de en `main` (que cae a interp por callees).

---

## 12. Flags y env vars de control

### CLI

| Flag                  | Significado                                    |
| :-------------------- | :--------------------------------------------- |
| `--jit-threshold N`   | Umbral de invocaciones para compilar           |
| `-m jit`              | Preset = `--jit-threshold 1`                   |
| `-m vm` / `-m interp` | Desactiva JIT (threshold = UINT32_MAX)         |
| `--jit-warn`          | Imprime warnings por IR ops no soportadas      |
| `--jit-stats`         | Stats finales (compiled / unsupported / no_ir) |

### Variables de entorno

| Var                              | Significado                            |
| :------------------------------- | :------------------------------------- |
| `VESTA_JIT_THRESHOLD=N`          | Equivalente a `--jit-threshold N`      |
| `VESTA_JIT_WARN_UNSUPPORTED=1`   | Equivalente a `--jit-warn`             |
| `VESTA_JIT_DISASM=1`             | Dumpea bytes + disasm Capstone del codigo JIT generado |

### Stats output (ejemplo)

```text
[jit stats] threshold=1 compiled=2 unsupported=0 no_ir=0
```

Indica que: el JIT estuvo activo con threshold 1, 2 metodos compilados
exitosamente, 0 fallaron por IR no soportada, 0 por falta de IR en el `.velb`.

---

## 13. Limitaciones actuales

1. **Sin regalloc real**: cada SSA value tiene slot fijo en stack. Cada IR
   op hace 2 LOAD + compute + 1 STORE. Aceptable para C1 (objetivo 10-20×);
   C2 con regalloc real lo arreglara.

2. **Sin inline caches**: CALLVIRT inline dispatch hace 5 loads (obj ->
   class -> vtable -> method -> jit_code) sin caching. D.4 implementara MIC
   + PIC con slot mutable.

3. **Sin OSR (On-Stack Replacement)**: para entrar al JIT desde un loop ya
   en ejecucion en interp, hay que esperar al proximo CALLVIRT. D.5
   implementara OSR.

4. **Sin deopt**: optimizaciones especulativas (devirt PGO-driven, type
   narrowing) no son posibles sin un mecanismo de deopt para revertir cuando
   las asunciones fallan. D.8 (C2) implementara deopt.

5. **Float no soportado**: FADD/FMUL/FSQRT/FCMP/FNEG/FABS caen a interp.
   Requiere regalloc target-aware sobre XMM regs (D.7).

6. **AOT no implementado**: el codigo generado vive en el code cache de la
   VM y se descarta al exit. AOT a `.velao` (D.10) permitira persistir el
   JIT entre ejecuciones.

7. **`vrt_safepoint_handler` es skeleton**: solo limpia el flag y retorna.
   No implementa la coordinacion completa GC-thread-pause-stack-walk-resume
   (Phase E.full).

---

## 14. API publica vesta_rt

Header `include/vesta_rt/public.h` (~250 LOC). API C estable que el codigo
JIT-eated puede llamar. Pensada para ser linkable contra futuro AOT.

### Tipos opacos

```c
typedef void* vrt_proc;     // ProcessVM*
typedef void* vrt_vm;       // VM*
typedef void* vrt_class;    // ClassInfo*
typedef void* vrt_method;   // MethodInfo*
typedef uint32_t vrt_handle;  // GcHandle
```

### Categorias de funciones

| Categoria  | Funciones                                                          |
| :--------- | :----------------------------------------------------------------- |
| GC         | `vrt_gc_alloc`, `vrt_gc_alloc_pinned`, `vrt_gc_deref`, `vrt_gc_handle_for_ptr`, `vrt_gc_drop`, `vrt_gc_addref`, `vrt_gc_release`, `vrt_gc_write_barrier` |
| Monitores  | `vrt_monitor_enter`, `vrt_monitor_exit`, `vrt_monitor_wait`, `vrt_monitor_notify`, `vrt_monitor_notify_all` |
| Excepciones| `vrt_throw_fatal`, `vrt_tryenter`, `vrt_tryleave`                  |
| FFI        | `vrt_invoke_native`                                                |
| Safepoint  | `vrt_safepoint_poll`, `vrt_safepoint_handler`                      |
| Introsp.   | `vrt_api_version`, `vrt_proc_vm`, `vrt_proc_pid`                   |
| Class reg  | `vrt_findclass`, `vrt_newobj`, `vrt_defclass`, `vrt_deffield`, `vrt_defmethod`, `vrt_addadvice`, `vrt_findmethod`, `vrt_findfield` |
| VM memory  | `vrt_vm_read_u64`, `vrt_vm_write_u64` (traduce vm_addr -> host_ptr via page cache) |
| Call       | `vrt_callvirt`, `vrt_callm`, `vrt_callclosure`, `vrt_calln`        |

### Versionado

```c
#define VRT_API_VERSION_MAJOR 1
#define VRT_API_VERSION_MINOR 0

uint32_t vrt_api_version();   // (major<<16) | (minor<<8) | patch
```

Bumpear major cuando cambien firmas (breaking change). Bumpear minor cuando
se anyadan funciones sin romper firmas existentes.

### ABI validation

`include/vesta_rt/abi.h` declara constantes de offset:

```c
#define VESTA_OBJ_HDR_CLASS_PTR_OFFSET     0
#define VESTA_OBJ_HDR_FLAGS_OFFSET         8
#define VESTA_OBJ_HDR_HASH_CODE_OFFSET    12
#define VESTA_OBJ_HDR_OWNER_PID_OFFSET    16
#define VESTA_OBJ_HDR_LOCK_DEPTH_OFFSET   20

#define VESTA_PROC_SAFEPOINT_FLAG_OFFSET   0
#define VESTA_PROC_REGISTERS_OFFSET       96
#define VESTA_REGISTER_SIZE                8
#define VESTA_PROC_REGISTER_COUNT         16

#define VESTA_CLASSINFO_VTABLE_OFFSET     80
#define VESTA_METHODINFO_JIT_CODE_OFFSET 104
```

En `src/vesta_rt/abi_checks.cpp`:

```cpp
static_assert(offsetof(ObjectHeader, class_ptr) == VESTA_OBJ_HDR_CLASS_PTR_OFFSET);
static_assert(offsetof(ObjectHeader, flags)     == VESTA_OBJ_HDR_FLAGS_OFFSET);
// ... etc para CADA constante
```

Si el layout C++ cambia sin actualizar `abi.h`, el build aborta inmediatamente.
Cero riesgo de incompatibilidad silenciosa entre runtime y JIT/AOT.

---

Ver tambien:

- [doc/ARCHITECTURE.md](../../ARCHITECTURE.md) - arquitectura general de la VM
- [doc/BENCHMARKS.md](../../BENCHMARKS.md) - performance numbers
- [doc/ROADMAP.md](../../ROADMAP.md) - sub-hitos D.4-D.10 pendientes
- [doc/VMdoc/IR/SSA.md](../IR/SSA.md) - SSA IR y pasadas de optimizacion
- [doc/VMdoc/SetInstruccionesVM/SUPER_INSTRUCCIONES.md](../SetInstruccionesVM/SUPER_INSTRUCCIONES.md)
  - opcodes super-instr que el JIT debe reconocer
