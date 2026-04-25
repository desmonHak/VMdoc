# Header del formato VELA

La cabecera del archivo `.vela` contiene la informacion necesaria para que el
linker pueda leer la tabla de modulos y encontrar cada modulo dentro del archivo.

## Estructura de la cabecera

```c
typedef struct PACKED HeaderVELA {
    char magic[4];              // "VELA"
    uint32_t version;           // version del formato
    uint32_t module_count;      // numero de modulos
    uint64_t module_table_offset;
} HeaderVELA;
```

## Entrada de la tabla de modulos
```c
typedef struct PACKED VELA_ModuleEntry {
    uint64_t offset;            // offset al modulo dentro del archivo
    uint64_t size;              // tamano del modulo
    uint32_t symbol_count;
    uint64_t symbol_table_offset;
    uint32_t relocation_count;
    uint64_t relocation_table_offset;
} VELA_ModuleEntry;
```

## Tabla de simbolos del modulo
```c
typedef struct PACKED VELA_Symbol {
    uint64_t offset;            // offset dentro del modulo
    uint8_t  type;              // FUNC, DATA, GLOBAL, LOCAL
    char     name[32];          // nombre del simbolo
} VELA_Symbol;
```

## Tabla de relocaciones del modulo
```c
typedef struct PACKED VELA_Relocation {
    uint64_t offset;            // donde aplicar la relocacion
    uint8_t  type;              // ABS64, REL32, REL64
    char     symbol[32];        // simbolo a resolver
} VELA_Relocation;
```

----
## ?Que contiene un modulo dentro del `.vela`?
```c
[ModuleHeader]
[Bytecode]
[Context serializado]
[SymbolTable]
[RelocationTable]
```

## ?Como lo usa el Linker?
- Carga el `.vela`
- Lee la tabla de modulos
- Busca simbolos que necesita resolver
- Extrae solo los modulos necesarios
- Los trata como si fueran `.velo` (objetos normales)