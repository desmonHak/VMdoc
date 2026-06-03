# Profile-Guided Optimization (PGO)

VestaVM incluye una infraestructura de **profile counters runtime** que
instrumenta el codigo en ejecucion para recolectar informacion estadistica
sobre el comportamiento dinamico del programa.  Esta informacion se vuelca
a un fichero binario `.vprof` consumible posteriormente por:

- El **JIT** para warm-start de funciones hot en ejecuciones sucesivas
  del mismo programa (Phase D.10).
- El **compilador AOT** futuro para tomar decisiones especulativas
  hard-coded en el ejecutable nativo (devirtualizacion monomorfica,
  hot/cold splitting, loop unrolling agresivo en hot paths, branch
  layout segun frecuencias, inlining basado en call counts).

## Activacion

Por defecto el sistema esta DESACTIVADO con coste cero (~1 ciclo de CPU
por hook: atomic load relaxed + branch predicted-not-taken).  Programas
sin instrumentacion no pagan overhead real.

Para activar:

```bash
# Flag CLI explicito (path opcional; default: program.vprof)
vm --run programa.velb --profile=mi_perfil.vprof

# O variable de entorno
VESTA_PROFILE_DUMP=mi_perfil.vprof vm --run programa.velb
```

Al finalizar el proceso (atexit handler), se genera el fichero binario
con el contenido completo de los contadores.

## Que se mide

### Branches condicionales

Por cada PC de un branch condicional (incluyendo opcodes fusionados
`cmpjmp`, `cmpjmpu`, `decjnz`), se registran dos contadores:

- `taken`: numero de veces que la condicion fue verdadera.
- `not_taken`: numero de veces que fue falsa.

Saltos incondicionales no se trackean (no aportan informacion al PGO).

### Tipos observados en call sites virtuales

Cada `callvirt` y `callm` registra hasta **4 clases distintas** observadas
en el receptor del metodo + un contador adicional de **megamorfismo**:

```
struct CallSiteCounter {
    TypeObservation types[4];   // class_ptr + count + class_name
    uint64_t megamorphic_count; // >4 clases distintas observadas
    uint8_t n_types;
};
```

Esto permite al optimizador clasificar cada call site en:

- **Monomorfico** (`n_types == 1`): devirtualizar a llamada directa
  + guard estatico de `class_ptr`.  Si el guard falla -> deopt (JIT)
  o fallback nativo inline (AOT).
- **Polimorfico estable** (`n_types <= 4`, `megamorphic_count == 0`):
  emitir un Polymorphic Inline Cache (PIC) inline con N entries.
- **Megamorfico** (`megamorphic_count > 0`): dispatch normal via vtable.

### Allocations

Por cada PC de un `newobj` o `gc_alloc`, se registra el numero total de
allocations.  Util para que el escape analysis priorice hot allocs
candidatos a stack allocation.

## Formato binario `.vprof`

```
Header (32 bytes):
  magic         u32   'VPRF' (0x46525056 little-endian)
  version       u16   1
  flags         u16   0
  n_branches    u32
  n_callsites   u32
  n_allocs      u32
  _reserved[12]

Branches (24 bytes cada uno):
  pc            u64
  taken         u64
  not_taken     u64

Callsites (variable):
  pc                u64
  n_types           u8
  _pad[7]
  megamorphic_count u64
  for each type:
    name_len        u32
    name            char[name_len]
    count           u64

Allocs (16 bytes cada uno):
  pc                u64
  count             u64
```

El nombre de la clase se almacena como string UTF-8 con length prefix,
NO como puntero a `ClassInfo` (que no sobrevive al exit del proceso).
El nombre se captura en la PRIMERA observacion de cada `(callsite, class)`
y se mantiene en memoria hasta el dump.

## Coste runtime

| Estado | Coste por hook |
|:-------|:--------------|
| Profile inactivo (default) | ~1 ciclo (atomic load relaxed + branch predicted-not-taken) |
| Profile activo (branch counter) | ~2-5 ciclos (lock mutex + map lookup + atomic fetch_add) |
| Profile activo (callvirt observation) | ~10-20 ciclos (lookup + posible insert + scan de hasta 4 slots) |
| Profile activo (newobj counter) | ~2-5 ciclos (igual que branch) |

Hooks runtime instrumentan 8 sitios distintos en el codigo:
- `exec_instr_callvirt` (dispatch via vtable)
- `exec_instr_callm` (dispatch dinamico via puntero a MethodInfo)
- `exec_instr_newobj` (allocation de objeto GC)
- `exec_instr_jmp` (branch condicional clasico)
- `exec_instr_cmpjmp` / `exec_instr_cmpjmpu` / `exec_instr_decjnz`
  (branches fusionados)
- 4 fast paths del scheduler (threaded + non-threaded x cmpjmp + JCC)
  porque el dispatch principal del scheduler bypassa las funciones
  `exec_instr_*` en hot loops via tabla inline.

## Interaccion con el JIT C1

Cuando el JIT C1 esta activo, los contadores SIGUEN funcionando: el JIT
emite codigo nativo que internamente sigue llamando a los runtime hooks
a traves del bridge.  Esto significa que **el perfil es valido sea cual
sea el modo de ejecucion** (interpretado, JIT-compilado, o mixto).

## Interaccion con el JIT C2 (Phase D.8 futuro)

El C2 consumira el `.vprof` para tomar decisiones especulativas:

- **Devirtualizacion**: callsite monomorfico -> reemplazar `callvirt`
  con call directo a la implementacion + guard de class_ptr.
- **PIC inline**: callsite con `n_types <= 4` y `megamorphic_count == 0`
  -> emitir una cadena de guards inline + jump tables.
- **Branch layout**: rama caliente (taken o not_taken >> 50%) se
  emite como fall-through; la fria con salto largo.
- **Inline policy**: callees con count alto y body pequeno se inlinean
  agresivamente.
- **Loop unrolling**: loops con iteraciones medias altas se desenrollan
  con factor 4/8.
- **Escape analysis priorizado**: allocs con count alto se prueban
  para stack allocation primero.

Si un guard falla en runtime, el JIT hace **deopt** al interprete
(reconstruye el estado del frame interp via stackmap) y continua
ahi.  Sin perdida de correctness.

## Interaccion con el compilador AOT (Phase AOT.6 futuro)

Mismo `.vprof`, mismas decisiones, pero el binario nativo NO puede
deoptar al interprete (no hay interp en el `.exe`).  Solucion:

- **Fallback inline**: por cada guard especulativo, el codigo nativo
  incluye una rama lenta inline que ejecuta el dispatch normal
  (vtable lookup completo).  Si el guard falla, salta a la rama
  lenta.  Penaliza tamano del `.exe` pero garantiza correctness.
- **Conservative compile**: si el perfil no es 100% monomorfico,
  el AOT NO devirtualiza.  La especulacion solo se aplica con alta
  confianza.

El flujo tipico de PGO en AOT:

```bash
# 1. Build instrumentado
vex --aot programa.vex -o programa_inst.exe --profile-gen

# 2. Ejecutar con workload representativo
./programa_inst.exe --workload typical
# -> genera programa.vprof

# 3. Build optimizado con PGO
vex --aot programa.vex -o programa.exe --profile-use=programa.vprof
```

## Aprendizaje empirico observado

Durante la validacion del sistema se observo que el frontend Vex hace
**devirtualizacion estatica agresiva** en compile-time.  Cuando el
frontend puede inferir el tipo concreto del receptor (por ejemplo:
`Shape c = new Circle(...)` usado directamente, sin reasignacion),
inlinea el cuerpo del metodo en el caller y NO emite `callvirt` en
bytecode.

Esto significa que el profiler **solo trackea callvirts REALES** que
el frontend no pudo resolver estaticamente (interface types con
asignacion dinamica, polimorfismo via condicionales, valores cargados
de campos, etc.).  Este es el comportamiento correcto y deseable: el
perfil debe contener exactamente la informacion que NO es deducible
estaticamente -- precisamente lo que el C2 y el AOT necesitan para
sus decisiones especulativas.
