# Documentacion VM

Version VM: -.-.-

----

**Esta documentacion fue realizada usando la aplicacion Obsidian.**
Recomendamos descargarla para una correcta lectura.

- Enlace a la VestaVM de la documentacion: [https://github.com/desmonHak/VM/](https://github.com/desmonHak/VM/)

----

## Licencia

[Leeme para saber que puedes hacer con el codigo y la documentacion](./LICENSE.md)

----

## Indice de documentos nuevos (instrucciones 0x40-0x64)

### Instrucciones

| Documento | Instrucciones cubiertas |
| :--------------------------------------------------------------------------- | :------------------------------- |
| [MODS y MODU](./SetInstruccionesVM/MOD.md) | mods, modu (0x40) |
| [SETCC](./SetInstruccionesVM/SETCC.md) | setcc (0x43) |
| [TRYENTER y TRYLEAVE](./SetInstruccionesVM/TRYENTER_TRYLEAVE.md) | tryenter (0x44), tryleave (0x45) |
| [Sistema de Strings](./SetInstruccionesVM/STRINGS.md) | strmake..strfinalize (0x46-0x54) |
| [STATIC_FIELDS](./SetInstruccionesVM/STATIC_FIELDS.md) | getstatic (0x60), setstatic (0x61) |
| [FFI_RUNTIME](./SetInstruccionesVM/FFI_RUNTIME.md) | gchandle (0x56), getpid (0x57), spawnon (0x58), loadmod (0x59), panic (0x5A), setmethdbg (0x5B), fextend (0x5C), fnarrow (0x5D), dlopen (0x62), dlsym (0x63), callni (0x64) |
| [META_OOP](./SetInstruccionesVM/META_OOP.md) | defclass (0xC9), deffield (0xCA), defmethod (0xCB), findclass (0xCC), findmethod (0xCD), addadvice (0xCE), findfield (0xCF), callm (0xFD), proceed (0xFE) |
| [SUPER_INSTRUCCIONES](./SetInstruccionesVM/SUPER_INSTRUCCIONES.md) | cmpjmp.cc (0x68), cmpjmpu.cc (0x69), decjnz (0x6A), mvtake (0x72), alu3 family (0x73-0x7B: adds3/subs3/muls3/addu3/subu3/mulu3/and3/or3/xor3), loadz (0x7C), loadzh (0x7D) |

### Arquitectura interna

| Documento | Contenido |
| :-------------------------------------------------------------------------------- | :----------------------------------------- |
| [JIT C1 baseline](./JIT/JIT.md) | MachineIR, encoder x86-64 hand-rolled, selector, stackmaps, JitRegistry, auto-trigger, ABI vesta_rt |
| [Profile-Guided Optimization](./JIT/Profiling.md) | Contadores runtime de branches/tipos/allocs, format `.vprof`, integracion con JIT (warm-start) y AOT (PGO en ejecutables) |
| [Sistema de Cadenas](./runtime/SistemaStrings.md) | FLAT/ROPE/SLICE, Compact Strings, interning, hash, GC |
| [Convencion de Encoding](./SetInstruccionesVM/ConvencionDeLlamadas/ConvencionEncoding.md) | Convention A vs B, guia para nuevas instrucciones |
| [Plan AOT (Full / Embed / Bare)](./JIT/AOT.md) | Tres tiers de deployment a binarios nativos, mapping IR -> codigo nativo, extensibilidad via annotations |

----

## Indice de documentos nuevos (HLL support)

### Generics y monomorphization

| Documento | Contenido |
| :------------------------------------------- | :--------------------------------------------- |
| [Generics](./Generics/Generics.md) | SPECIALIZE (0x3A), GenericParam, reificacion |

### Colecciones nativas

| Documento | Contenido |
| :------------------------------------------------- | :-------------------------------------------- |
| [Collections](./Collections/Collections.md) | VestaList, VestaArrayList, VestaHashMap, VestaHashSet |

### Sistema de paquetes

| Documento | Contenido |
| :------------------------------------------------- | :-------------------------------------------- |
| [PackageSystem](./Packages/PackageSystem.md) | package.vel, resolucion de dependencias, VESTA_PKG_PATH |

### IR SSA

| Documento | Contenido |
| :----------------------------------- | :---------------------------------------------------- |
| [SSA IR](./IR/SSA.md) | Tipos, opcodes, formato textual, API C++, integracion HLL |

### Protocolo de depuracion

| Documento | Contenido |
| :---------------------------------------------- | :------------------------------------------- |
| [DebugProtocol](./Debug/DebugProtocol.md) | Protocolo TCP/JSON, comandos, eventos, integracion |

----

## Lenguaje Vesta

Documentacion del lenguaje de alto nivel Vesta que compila a VestaVM bytecode.

### Vision general y tipos

| Documento | Contenido |
| :-------- | :-------- |
| [Vesta](./Vesta/Vesta.md) | Vision general, pipeline, anotaciones |
| [TiposDatos](./Vesta/TiposDatos.md) | Primitivos, string, punteros, structs, arrays, Optional, Result |
| [Enums](./Vesta/Enums.md) | Uniones etiquetadas y enums con valor |
| [OptionalResult](./Vesta/OptionalResult.md) | `Optional<T>`, `Result<T,E>`, `?`, `!!` y la afirmacion de no-nulo |
| [Operadores](./Vesta/Operadores.md) | Precedencia, semantica y sobrecarga |
| [ControlFlow](./Vesta/ControlFlow.md) | `if`/`while`/`for`, `match`, `break`/`continue`, `goto` |
| [Strings](./Vesta/Strings.md) | Metodos, interpolacion `${expr}`, especificadores de formato, codificaciones |
| [Matematicas](./Vesta/Matematicas.md) | Las 18 funciones de `vesta_math` |
| [EstiloYFormato](./Vesta/EstiloYFormato.md) | Lo que `vm fmt` garantiza, y por que |

### Funciones y parametros

| Documento | Contenido |
| :-------- | :-------- |
| [Parametros](./Vesta/Parametros.md) | Las doce formas de parametro por los siete contextos |
| [DireccionParametros](./Vesta/DireccionParametros.md) | `in` / `out` / `inout` |
| [LlamadaUniforme](./Vesta/LlamadaUniforme.md) | `x.f(a)` == `f(x, a)`, el hueco `_`, argumentos con nombre, sobrecarga |
| [Closures](./Vesta/Closures.md) | Lambdas, capturas, `fn(...)` frente a `cfn(...)` |
| [ClosuresEnCampos](./Vesta/ClosuresEnCampos.md) | Guardar una lambda en un campo y quien libera su entorno |

### Programacion orientada a objetos

| Documento | Contenido |
| :-------- | :-------- |
| [OOP](./Vesta/OOP.md) | Clases, herencia, interfaces, properties, constructores, RAII |
| [ReflexionAOP](./Vesta/ReflexionAOP.md) | forName, getClass, getField, getMethod, invoke, @Aspect, proceed() |
| [Generics](./Vesta/Generics.md) | Monomorfizacion al compilar, conceptos, especializacion |

### Memoria y ownership

| Documento | Contenido |
| :-------- | :-------- |
| [SmartPointers](./Vesta/SmartPointers.md) | `unique<T>`, `shared<T>`, `move`, deleters propios |
| [BorrowChecker](./Vesta/BorrowChecker.md) | `borrow<T>` / `borrow_mut<T>`, las cuatro reglas, NLL, reborrow |
| [Overlays](./Vesta/Overlays.md) | Vistas tipadas sobre memoria binaria |
| [DisposicionSecciones](./Vesta/DisposicionSecciones.md) | `place_section`, datos crudos y consultar las secciones desde el programa |

### Concurrencia y distribucion

| Documento | Contenido |
| :-------- | :-------- |
| [Async](./Vesta/Async.md) | @Async, await, spawn, rspawn, Future, msgsend/msgrecv, fibras |
| [Sincronizacion](./Vesta/Sincronizacion.md) | `synchronized`, monitores, `wait`/`notify` |

### Gestion de errores

| Documento | Contenido |
| :-------- | :-------- |
| [Excepciones](./Vesta/Excepciones.md) | try/catch/finally, FatalError, panic(), stack trace, RAII |
| [RuntimeHooks](./Vesta/RuntimeHooks.md) | `@AllocatorOverride`, `@PanicHandler`, `@UnwindImpl` y los demas ganchos |

### Metaprogramacion

| Documento | Contenido |
| :-------- | :-------- |
| [Metaprogramacion](./Vesta/Metaprogramacion.md) | `comptime`, introspeccion, macros |
| [ConstructorComptime](./Vesta/ConstructorComptime.md) | Constructor que se ejecuta al compilar y recibe la expresion sin evaluar |
| [CompilacionCondicional](./Vesta/CompilacionCondicional.md) | `@Target` y las ramas por objetivo |
| [Instrumentacion](./Vesta/Instrumentacion.md) | `@Hook(enter/exit/unwind)`, selectores y `@NoInstrument` |

### Modulos y paquetes

| Documento | Contenido |
| :-------- | :-------- |
| [Namespaces](./Vesta/Namespaces.md) | `namespace`, visibilidad, `import`, `impl` |
| [Modulos](./Vesta/Modulos.md) | `.vxi`, cache incremental, compilacion paralela |
| [CargaDinamica](./Vesta/CargaDinamica.md) | `loadmodule` / `unloadmodule` y hot-reload |
| [PackageManager](./Vesta/PackageManager.md) | `vm pkg`: manifiesto, lockfile, firmas, auditoria |
| [Sandbox](./Vesta/Sandbox.md) | Capabilities sobre modulos cargados |

### Colecciones y FFI

| Documento | Contenido |
| :-------- | :-------- |
| [Colecciones](./Vesta/Colecciones.md) | ArrayList/HashMap/HashSet/Queue/Deque/TreeMap/TreeSet/Stack como keywords |
| [FFI](./Vesta/FFI.md) | extern declarativo, ffi_open/sym/call, plugins, vesta_io, vesta_math |
| [InteropC](./Vesta/InteropC.md) | Interop con C y ownership de structs |
| [CallbacksNativos](./Vesta/CallbacksNativos.md) | `as_native_callback`: pasar una funcion Vesta a una libreria nativa |
| [SyscallsWindows](./Vesta/SyscallsWindows.md) | La capa NT de `std.syscall.windows` |

### Compilacion nativa y bajo nivel

| Documento | Contenido |
| :-------- | :-------- |
| [CompilacionNativa](./Vesta/CompilacionNativa.md) | `-m aot`, objetivos, `gc<T>` opt-in, sin runtime obligatorio |
| [InlineAsm](./Vesta/InlineAsm.md) | `asm { }`, `register("reg")`, lista de operandos, `@Naked` |
| [Vectorizacion](./Vesta/Vectorizacion.md) | Auto-vectorizacion SSE2/AVX2/AVX512 y `cpu_features()` |
| [Enlazador](./Vesta/Enlazador.md) | `vm --link` y `vm --ar` |
| [Terminal](./Vesta/Terminal.md) | Color de 24 bits, cursor y pantalla |

----