# Estructuras (struct) en Vesta

Una **estructura** (`struct`) es un tipo de dato que agrupa varios campos relacionados
bajo un mismo nombre. A diferencia de las clases OOP, los structs son tipos de datos
**ligeros**: no heredan de `Object`, no tienen metodos virtuales por defecto, y pueden
ser tipos de valor (copiados en lugar de referenciados).

**Analogia:** piensa en un formulario de papel. Un formulario de "Persona" tiene
campos para el nombre, la edad y la altura. No contiene logica - solo datos.

A nivel de bytecode VestaVM, los campos de un struct se almacenan de forma contigua
en memoria, y el compilador calcula los offsets de cada campo en tiempo de compilacion.

---

## Forma 1: struct clasico (estilo C)

La forma mas basica, directamente equivalente a C:

```c
// Declaracion
struct Punto {
    uint8_t x;
    uint8_t y;
};

// Uso: siempre hay que escribir "struct Punto"
struct Punto p1 = { 1, 2 };  // inicializar con valores
struct Punto p2;
p2.x = 10;
p2.y = 20;
```

---

## Forma 2: typedef (mas comodo)

Con `typedef` puedes omitir la palabra `struct` al declarar variables:

```c
// Declaracion con typedef
typedef struct {
    uint8_t x;
    uint8_t y;
} Punto;

// Uso: sin "struct"
Punto p1 = { 1, 2 };
Punto p2;
p2.x = 10;
```

---

## Forma 3: typedef con nombre de tag

Combina ambas formas, permitiendo referenciar el tipo con o sin `struct`:

```c
typedef struct Punto {
    uint8_t x;
    uint8_t y;
} Punto;

// Ambas formas son validas:
Punto        p1 = { 1, 2 };
struct Punto p2 = { 3, 4 };
```

---

## Forma 4: declaracion inline

Util para definir el tipo justo donde se necesita la variable:

```c
typedef struct { uint8_t x; uint8_t y; } Punto;
```

---

## Forma 5: struct anidado

Un struct puede contener otro struct como campo:

```c
typedef struct {
    uint8_t x;
    uint8_t y;
    struct {
        float z;  // campo z en una sub-estructura anonima
    } coord3d;
} Punto3D;

// Inicializacion anidada
Punto3D p = { 1, 2, { 3.14f } };

// Acceso
float altura = p.coord3d.z;
```

---

## Forma 6: struct con funciones

Los structs pueden tener funciones que los manipulan. Esta es la base del estilo
de programacion orientada a objetos "ligero" sin herencia:

```c
typedef struct Punto {
    uint8_t x;
    uint8_t y;
} Punto;

// Funcion que opera sobre el struct
Punto sumar(Punto a, Punto b) {
    Punto resultado = { a.x + b.x, a.y + b.y };
    return resultado;
}

// Uso
Punto p1 = { 1, 2 };
Punto p2 = { 3, 4 };
Punto suma = sumar(p1, p2);  // suma.x = 4, suma.y = 6
```

---

## Struct al estilo C++ (con metodos)

Vesta tambien permite structs con metodos directamente dentro de la definicion,
como en C++. Esto crea clases ligeras que NO heredan de `Object` y no participan
en el sistema GC por defecto:

```c
struct Rectangulo {
    Punto esq_sup;
    Punto esq_inf;

    // Constructor
    Rectangulo(Punto sup, Punto inf) : esq_sup(sup), esq_inf(inf) { }

    // Metodo
    float area() {
        return (esq_inf.x - esq_sup.x) * (esq_inf.y - esq_sup.y);
    }

    // Metodo con referencia
    bool contiene(Punto p) {
        return p.x >= esq_sup.x && p.x <= esq_inf.x &&
               p.y >= esq_sup.y && p.y <= esq_inf.y;
    }
}

// Uso
Punto a = { 0, 0 };
Punto b = { 10, 10 };
Rectangulo r = Rectangulo(a, b);

float superficie = r.area();         // 100.0
bool dentro = r.contiene({ 5, 5 }); // true
```

---

## Struct vs Clase OOP

| Caracteristica          | struct                          | class (OOP)                        |
| :---------------------- | :------------------------------ | :--------------------------------- |
| Hereda de Object        | No (por defecto)                | Si (por defecto)                   |
| Metodos virtuales       | No                              | Si                                 |
| GC automatico           | No (manual o stack)             | Si (heap GC)                       |
| Overhead por objeto     | 0 bytes (solo los campos)       | 24 bytes (ObjectHeader)            |
| Uso tipico              | Datos simples, FFI, geometria   | Entidades con comportamiento rico  |

---

## Representacion en memoria

Los campos de un struct se almacenan de forma contigua. Para `struct Punto { uint8_t x; uint8_t y; }`:

```
offset 0: x (1 byte)
offset 1: y (1 byte)
total:     2 bytes
```

El compilador puede anadir **padding** (relleno) para alinear los campos a sus
tamanos naturales:

```c
struct EjemploAlineacion {
    uint8_t  a;   // offset 0, 1 byte
    // 7 bytes de padding aqui (para alinear b a 8 bytes)
    uint64_t b;   // offset 8, 8 bytes
};
// sizeof = 16, no 9
```

---

## Nivel de bytecode

A nivel de bytecode VestaVM, acceder al campo `x` de un `Punto` en la direccion
almacenada en `r1` seria:

```c
// Leer el campo x (offset 0) del Punto apuntado por r1:
// xchg cur0, r1              // cur0 apunta al struct
// readcur cur0, r0b          // leer 1 byte (uint8_t) en r0
```

El compilador calcula los offsets automaticamente basandose en el tamano de cada campo.

---

Ver tambien:
- [[POO.md]] - clases completas con herencia y GC
- [[Estructuras de datos.md]] - arrays, listas, hashmaps
- [[SetInstruccionesVM/GC/Allocator crudo para FFI y memoria manual.md]] - structs en FFI
