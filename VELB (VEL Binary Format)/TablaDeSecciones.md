# Tabla de secciones del VELB

La tabla de secciones es el directorio del archivo `.velb`. Describe donde se
encuentra cada seccion (codigo, datos, metadatos) dentro del archivo binario y
que rango de direcciones virtuales le corresponde cuando se carga en la VM.

**Analogia:** es como el indice de un libro. Dice "el codigo esta en las paginas 10-50,
los datos en las paginas 51-70, los metadatos en las paginas 71-100". Sin este indice,
el loader no sabria donde encontrar cada parte del programa.

La tabla de secciones contienen la informacion de donde se ubica cada seccion. Esta tabla no tiene por que estar ubicada despues del header, pues en la practica podria estar en cualquier parte siempre que la direccion de esta se defina en el header.

La tabla de secciones debe indicar donde esta cada seccion y su direccion de inicio y final:
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
Cada entrada en la tabla indicara su direccion inicio y final, por tanto, indirectamente esta indicando a que espacio de direcciones pertenece, y por tanto estara indicando si se trata de una seccion de codigo, datos, meta-datos u otro.

En caso de ser el archivo, y no una vez cargado el codigo en la VM, el rango de memoria sera aquel sin aplicar la direccion virtual, teniendo el offset de inico y final del bytecode.

> En el nombre de la seccion se permite un maximo de 16 bytes para el nombre, pero el usuario puede poner nombres mas cortos que eso, el nombre leido sera el que se lea hasta llegar a los 16 bytes o hasta el encontrar un caracter nulo (`\0` que en hexadecimal es `0x00`).