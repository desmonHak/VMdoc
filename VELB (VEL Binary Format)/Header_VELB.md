# Header del formato VELB

La **cabecera VELB** es lo primero que lee el loader al cargar un ejecutable `.velb`.
Contiene la firma magica, la version del formato, informacion de compatibilidad con
versiones de VM, checksum de integridad, y la tabla de espacios de direcciones.

**Analogia:** es como el encabezado de un pasaporte: lo primero que mira el oficial
de frontera (el loader) para saber si el documento es valido, que version tiene y
que informacion contiene.

---

## Campos del header

- `velb_magic`: Un ejecutable del tipo ``VELB`` inicia con una firma de 4 bytes con las 4 letras `VELB`.
```c
/**
* A traves del numero magico, se puede conocer el endian de la maquina que
* genero el bytecode. EN?n caso de ser little?endian (LE) en el ejecutable pondra
* "VELB", pero en caso de ser big-endian (BE), pondra BLEV
*/
typedef union velb_magic {
	uint32_t firma;     // 0x424C4556 == "VELB" en LE
    uint8_t firma_byte[4];
} velb_magic;
```

- `velb_version_format`: A continuacion le sigue ``4 bytes`` para la version del formato.
```c
typedef uint32_t velb_version_format;
```

- ``max_vm_version``: ``4 bytes`` con la maxima version compatible en la que el codigo puede ser ejecutado en una VM.
- `min_vm_version`: ``4 bytes`` con la minima version compatible en la que el codigo puede ser ejecutado en una VM.
```c
typedef uint32_t max_vm_version;  
typedef uint32_t min_vm_version;
```

> En caso de que la version mayor fuera, version 0.0.0, se estara indicando que el codigo es compatible con cualquier futura version de la VM. 
> En caso de que la version menor fuera 0.0.0, quiere decir que el codigo es retro compatible con cualquier version de la VM.


- ``velb_checksum``: ``8 bytes`` para indicar un checksum del codigo, sino coincide, no se ejecuta.
```c
typedef uint64_t velb_checksum;
```

- `velb_flags`: ``8 bytes`` para indicar caracteristicas del ejecutable (meta-informacion adicional del ejecutable).
```c
typedef uint64_t velb_flags;
```

- `build_timestamp`: ``8 bytes`` de marca de tiempo para indicar la fecha de compilacion.
```c
typedef uint64_t build_timestamp;
```

- ``target_arch``:``4 bytes`` para indicar la arquitectura en la que se realizo la compilacion del ejecutable.
```c
typedef uint32_t target_arch;
```

- ``section_count``: ``4 bytes`` para el numero de secciones en la tabla de secciones.
```c
typedef uint32_t section_count;
```

- ``section_table_offset``:``8 bytes`` de desplazamiento respecto a la direccion base para indicar la tabla de secciones.
```c
typedef uint64_t section_table_offset;
```

- `number_spaces_address`: 8 bytes para indicar la cantidad de espacio de direcciones de la tabla que viene a continuacion.
```c
typedef uint64_t number_spaces_address;
```

- A continuacion va una tabla estatica de X entradas con el rango de  direcciones, indicado en el apartado de [[EspacioDeDireccionesDeLaMemoria]]. Cada entrada:
```c
typedef struct range_memory {
	uint64_t address_init;
	uint64_t address_final;
} range_memory;
```
debe definir una direccion inicial y una direccion final para cada espacio de direcciones, donde conviviran las distintas secciones. En caso de que estos dos valores sean iguales, se estara indicando que esta seccion no existe y por tanto no se usara memoria en esta seccion. Para el resto de los casos, si se define un rango, el codigo sera cargado en estos espacios de direcciones. Un espacio de direcciones puede tener varias secciones. Los espacios de direcciones no reservan toda su memoria en el momento, sino que lo haran segun el codigo lo necesite. Inicialmente solo se reserva memoria para los datos y codigo cargados/mapeados del ejecutable. 
Aunque los espacios de direcciones esta predefinidos, un usuario puede crear los suyos propios si lo cree necesario, pero debera definir la cantidad de espacio de direcciones existente

Header:
```c
typedef struct PACKED HeaderVELB {  
    velb_magic magic;  
    velb_version_format format_v;  
  
    max_vm_version max_v;  
    min_vm_version min_v;  
  
    velb_checksum checksum;  
  
    velb_flags flags;  
    build_timestamp timestamp;  
    target_arch arch;  
  
    section_count count;  
  
    section_table_offset table_offset;  
  
    number_spaces_address n_spaces;  
    range_memory *address_spaces; // tabla de espacios de direcciones  
} HeaderVELB;
```

La forma correcta de conocer el tamano del header es la siguiente:
```c
    uint64_t Linker::compute_sections_base_offset() const {
        // el hader siempre esta alineado a 16 bytes
        return align_up(

            // se resta el tamano del puntero de range_memory, ya que en el archivo,
            // la tabla va espacio de direcciones va direcamente incrustado, en lugar de
            // ser un puntero.
            (sizeof(HeaderVELB) - sizeof(range_memory *)) + (
                // el espacio real de todo_ el hader es la cantidad de espacios de direcciones
                // po el tamano de una entrada de rango de memoria (16 bytes para inicio y fin)
                final_header.n_spaces * sizeof(range_memory)
            ) +

            final_sections.size() * sizeof(section_range_memory)
        , 16);
    }

```
El ``header`` siempre debera estar alineado a 16 bytes.