```c
typedef struct PACKED HeaderVELA {
    char magic[4];              // "VELA"
    uint32_t version;           // versión del formato
    uint32_t module_count;      // número de módulos
    uint64_t module_table_offset;
} HeaderVELA;
```

## Entrada de la tabla de módulos
```c
typedef struct PACKED VELA_ModuleEntry {
    uint64_t offset;            // offset al módulo dentro del archivo
    uint64_t size;              // tamaño del módulo
    uint32_t symbol_count;
    uint64_t symbol_table_offset;
    uint32_t relocation_count;
    uint64_t relocation_table_offset;
} VELA_ModuleEntry;
```

## Tabla de símbolos del módulo
```c
typedef struct PACKED VELA_Symbol {
    uint64_t offset;            // offset dentro del módulo
    uint8_t  type;              // FUNC, DATA, GLOBAL, LOCAL
    char     name[32];          // nombre del símbolo
} VELA_Symbol;
```

## Tabla de relocaciones del módulo
```c
typedef struct PACKED VELA_Relocation {
    uint64_t offset;            // dónde aplicar la relocación
    uint8_t  type;              // ABS64, REL32, REL64
    char     symbol[32];        // símbolo a resolver
} VELA_Relocation;
```

----
## ¿Qué contiene un módulo dentro del `.vela`?
```c
[ModuleHeader]
[Bytecode]
[Context serializado]
[SymbolTable]
[RelocationTable]
```

## ¿Cómo lo usa el Linker?
- Carga el `.vela`
- Lee la tabla de módulos
- Busca símbolos que necesita resolver
- Extrae solo los módulos necesarios
- Los trata como si fueran `.velo` (objetos normales)