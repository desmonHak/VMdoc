# VELA - VEL Archive (Libreria estatica)

Un archivo **.vela** es una **libreria estatica** para VestaVM.

**Analogia:** si el `.velb` es un programa ejecutable, el `.vela` es una caja de
herramientas. Contiene multiples modulos objeto precompilados que otros programas
pueden usar. El linker coge solo las herramientas que necesita de la caja, sin
incluir todas las demas en el ejecutable final.

Es exactamente lo que es un `.a` en Linux o un `.lib` en Windows.

Un archivo **.vela** es una **libreria estatica** para VestaVM.
Es exactamente lo que es un `.a` en Linux o un `.lib` en Windows:

- contiene **multiples modulos objeto**
- cada modulo tiene:
    - bytecode
    - contexto
    - simbolos exportados
    - simbolos requeridos
    - relocaciones

- el linker solo extrae los modulos necesarios para resolver simbolos

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