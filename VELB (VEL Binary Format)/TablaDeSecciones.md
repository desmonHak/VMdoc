La tabla de secciones contienen la información de donde se ubica cada sección. Esta tabla no tiene por que estar ubicada después del header, pues en la practica podría estar en cualquier parte siempre que la dirección de esta se defina en el header.

La tabla de secciones debe indicar donde esta cada sección y su dirección de inicio y final:
```c
typedef struct PACKED name_section {  
    union {  
        struct {  
            uint64_t hi;  
            uint64_t lo;  
        } as_u64;  
  
        char name[16];  
    };  
} name_section;  
  
typedef struct PACKED section_range_memory {  
    range_memory address;  
  
    // nombre de la seccion, maximo 16 bytes.  
    name_section name;  
    uint64_t offset; // donde empieza el bytecode o los datos dentro del archivo
} section_range_memory;
```
Cada entrada en la tabla indicara su dirección inicio y final, por tanto, indirectamente esta indicando a que espacio de direcciones pertenece, y por tanto estará indicando si se trata de una sección de código, datos, meta-datos u otro.

En caso de ser el archivo, y no una vez cargado el codigo en la VM, el rango de memoria sera aquel sin aplicar la dirección virtual, teniendo el offset de inico y final del bytecode.

> En el nombre de la sección se permite un máximo de 16 bytes para el nombre, pero el usuario puede poner nombres mas cortos que eso, el nombre leído será el que se lea hasta llegar a los 16 bytes o hasta el encontrar un carácter nulo (`\0` que en hexadecimal es `0x00`).