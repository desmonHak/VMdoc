Un archivo **.vela** es una **librería estática** para VestaVM.
Es exactamente lo que es un `.a` en Linux o un `.lib` en Windows:

- contiene **múltiples módulos objeto**
- cada módulo tiene:
    - bytecode
    - contexto
    - símbolos exportados
    - símbolos requeridos
    - relocaciones

- el linker solo extrae los módulos necesarios para resolver símbolos

```c
VELA
│
├── HeaderVELA
│     - magic = "VELA"
│     - version
│     - module_count
│     - module_table_offset
│
├── ModuleTable[] (module_count entradas)
│     - offset
│     - size
│     - symbol_count
│     - symbol_table_offset
│     - relocation_count
│     - relocation_table_offset
│
├── Module 0
│     ├── bytecode
│     ├── context serializado
│     ├── symbol table
│     └── relocation table
│
├── Module 1
│     └── ...
│
└── ...
```