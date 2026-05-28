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
| [Sistema de Cadenas](./runtime/SistemaStrings.md) | FLAT/ROPE/SLICE, Compact Strings, interning, hash, GC |
| [Convencion de Encoding](./SetInstruccionesVM/ConvencionDeLlamadas/ConvencionEncoding.md) | Convention A vs B, guia para nuevas instrucciones |

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

## Lenguaje Vex (Phase A)

Documentacion del lenguaje de alto nivel Vex que compila a VestaVM bytecode.

### Vision general y tipos

| Documento | Contenido |
| :------------------------------------------- | :----------------------------------------------------------- |
| [Vex](./Vex/Vex.md) | Vision general, pipeline, anotaciones, inline asm, fases A-H |
| [TiposDatos](./Vex/TiposDatos.md) | Primitivos, string, punteros, structs, arrays, Optional, Result |

### Programacion orientada a objetos

| Documento | Contenido |
| :------------------------------------ | :-------------------------------------------------------------------- |
| [OOP](./Vex/OOP.md) | Clases, herencia, interfaces, properties, constructores, RAII |
| [ReflexionAOP](./Vex/ReflexionAOP.md) | forName, getClass, getField, getMethod, invoke, @Aspect, proceed() |
| [Generics](./Vex/Generics.md) | Monomorphizacion compile-time, name mangling, especialize (0x3A) |

### Concurrencia y distribucion

| Documento | Contenido |
| :-------------------------------- | :--------------------------------------------------------------------------------- |
| [Async](./Vex/Async.md) | @Async, await, spawn, spawn here/on(N), rspawn, Future, msgsend, msgrecv, synchronized |

### Gestion de errores

| Documento | Contenido |
| :-------------------------------------- | :------------------------------------------------------------- |
| [Excepciones](./Vex/Excepciones.md) | try/catch/finally, FatalError, panic(), stack trace, RAII |

### Colecciones y FFI

| Documento | Contenido |
| :-------------------------------------- | :------------------------------------------------------------------------ |
| [Colecciones](./Vex/Colecciones.md) | ArrayList/HashMap/HashSet/Queue/Deque/TreeMap/TreeSet/Stack como keywords |
| [FFI](./Vex/FFI.md) | extern declarativo, ffi_open/sym/call, plugins, vesta_io, vesta_math |

----