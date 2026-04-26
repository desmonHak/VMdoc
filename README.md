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

## Indice de documentos nuevos (instrucciones 0x40-0x54)

### Instrucciones

| Documento                                                                    | Instrucciones cubiertas          |
| :--------------------------------------------------------------------------- | :------------------------------- |
| [MODS y MODU](./SetInstruccionesVM/MOD.md)                                   | mods, modu (0x40)                |
| [SETCC](./SetInstruccionesVM/SETCC.md)                                        | setcc (0x43)                     |
| [TRYENTER y TRYLEAVE](./SetInstruccionesVM/TRYENTER_TRYLEAVE.md)              | tryenter (0x44), tryleave (0x45) |
| [Sistema de Strings](./SetInstruccionesVM/STRINGS.md)                         | strmake..strfinalize (0x46-0x54) |

### Arquitectura interna

| Documento                                                                         | Contenido                                  |
| :-------------------------------------------------------------------------------- | :----------------------------------------- |
| [Sistema de Cadenas](./runtime/SistemaStrings.md)                                 | FLAT/ROPE/SLICE, Compact Strings, interning, hash, GC |
| [Convencion de Encoding](./SetInstruccionesVM/ConvencionDeLlamadas/ConvencionEncoding.md) | Convention A vs B, guia para nuevas instrucciones |

----

## Indice de documentos nuevos (HLL support)

### Generics y monomorphization

| Documento                                    | Contenido                                      |
| :------------------------------------------- | :--------------------------------------------- |
| [Generics](./Generics/Generics.md)           | SPECIALIZE (0x3A), GenericParam, reificacion   |

### Colecciones nativas

| Documento                                          | Contenido                                     |
| :------------------------------------------------- | :-------------------------------------------- |
| [Collections](./Collections/Collections.md)        | VestaList, VestaArrayList, VestaHashMap, VestaHashSet |

### Sistema de paquetes

| Documento                                          | Contenido                                     |
| :------------------------------------------------- | :-------------------------------------------- |
| [PackageSystem](./Packages/PackageSystem.md)        | package.vel, resolucion de dependencias, VESTA_PKG_PATH |

### IR SSA

| Documento                            | Contenido                                             |
| :----------------------------------- | :---------------------------------------------------- |
| [SSA IR](./IR/SSA.md)                | Tipos, opcodes, formato textual, API C++, integracion HLL |

### Protocolo de depuracion

| Documento                                       | Contenido                                    |
| :---------------------------------------------- | :------------------------------------------- |
| [DebugProtocol](./Debug/DebugProtocol.md)        | Protocolo TCP/JSON, comandos, eventos, integracion |

----