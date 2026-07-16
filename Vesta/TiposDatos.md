# Sistema de tipos de Vesta

Vesta es estaticamente tipado con inferencia local. El compilador conoce el tipo de cada
expresion en tiempo de compilacion. No hay tipos dinamicos ni boxing implicito.

---

## Tipos primitivos

### Enteros con signo

| Tipo | Alias | Tamaño | Rango |
| :------ | :------ | :----- | :----------------------------------- |
| `i8` | `int8_t` | 1 byte | -128 .. 127 |
| `i16` | `int16_t` | 2 bytes | -32 768 .. 32 767 |
| `i32` | `int32_t` | 4 bytes | -2 147 483 648 .. 2 147 483 647 |
| `i64` | `int64_t` | 8 bytes | -9.2e18 .. 9.2e18 |

### Enteros sin signo

| Tipo | Alias | Tamaño | Rango |
| :------ | :------- | :----- | :------------------------- |
| `u8` | `uint8_t` | 1 byte | 0 .. 255 |
| `u16` | `uint16_t` | 2 bytes | 0 .. 65 535 |
| `u32` | `uint32_t` | 4 bytes | 0 .. 4 294 967 295 |
| `u64` | `uint64_t` | 8 bytes | 0 .. 18.4e18 |

Ambas familias son aliases resueltos por el type checker; `i32` e `int32_t` son el mismo
tipo.

### Punto flotante

| Tipo | Alias | Tamaño | Precision |
| :------ | :------ | :------ | :---------------- |
| `f32` | `float` | 4 bytes | IEEE 754 simple |
| `f64` | `double`| 8 bytes | IEEE 754 doble |

```java
f64 pi = 3.14159265358979;
f32 radio = 5.0f; // sufijo f indica f32 literal
f64 area = pi * radio * radio;
println("Area: ${area}");
```

### Booleano

```java
bool activo = true;
bool listo = false;
bool resultado = (x > 0) && !activo;
```

### Caracter

```java
char letra = 'A'; // codepoint Unicode (u32 internamente)
char newline = '\n';
char acento = 'é';
```

### void

Tipo de retorno para funciones sin resultado. No se puede usar como tipo de variable.

---

## Tipo string (GC-managed)

`string` es un tipo de primera clase gestionado por el GC. Internamente es un
`StringObject` con soporte para FLAT, ROPE y SLICE.

```java
string saludo = "Hola";
string nombre = "Mundo";
string mensaje = saludo + " " + nombre; // concatenacion O(1) ROPE

i32 len = mensaje.length(); // numero de code points
i32 bytes = mensaje.bytes(); // numero de bytes en encoding
u8* raw = mensaje.cstr(); // puntero host nul-terminado (para FFI)
```

### Metodos de string

| Metodo | Retorno | Descripcion |
| :------------------ | :--------- | :---------------------------------------- |
| `s.length()` | `i32` | Numero de code points |
| `s.bytes()` | `i32` | Numero de bytes en el encoding actual |
| `s.cstr()` | `char*` | Puntero host nul-terminado (ASCII/UTF-8) |
| `s.wstr()` | `char*` | Puntero host UTF-16LE (para Win32 *W API) |
| `s.hash()` | `u64` | Hash FNV-1a (cacheado) |
| `s.intern()` | `string` | Handle canonico del pool de interning |
| `s.equals(t)` | `bool` | Comparacion lexicografica de bytes |
| `s.concat(t)` | `string` | Concatenacion (alias de `s + t`) |

### Operadores de string

```java
string a = "abc";
string b = "def";

string c = a + b; // concatenacion: "abcdef"
bool igual = a == b; // false
bool dist = a != b; // true

// Auto-coercion de literales:
string d = "inicio_" + a; // el literal se promueve a StringObject
```

### Encodings

```java
// Constantes de encoding (para str_convert):
// ENC_ASCII = 0, ENC_ANSI = 1, ENC_UTF8 = 2, ENC_UTF16 = 3, ENC_UTF32 = 4

string utf8 = str_make(buffer_ptr, len);
string utf16 = str_convert(utf8, ENC_UTF16);
char* wptr = utf16.wstr(); // listo para Win32 *W API
```

---

## Tipos de puntero

Vesta tiene dos tipos de puntero con semantica bien diferenciada:

| Tipo | Apunta a | Acceso en bytecode | Uso tipico |
| :-------------- | :------------- | :----------------- | :---------------------- |
| `T*` | Memoria HOST | `movh` | malloc, FFI, str_cstr() |
| `VirtualPtr<T>` | Memoria VM | `mov` | variables locales, `&x` |

### Puntero host (`T*`)

```java
// malloc devuelve un puntero host
u8* buf = malloc(256);
buf[0] = 0xFF; // escribe en memoria host via movh
free(buf);

// FFI: puntero host a cadena de caracteres
char* raw = str.cstr();
// ... pasar raw a una API nativa ...
```

### Puntero virtual (`VirtualPtr<T>`)

```java
// address-of de una variable local produce VirtualPtr
i32 x = 42;
VirtualPtr<i32> p = &x; // p apunta a la pila VM
*p = 100; // escribe en VM memory
println("x = ${*p}"); // x = 100
```

### Aritmetica de punteros

```java
u8* arr = malloc(10);
u8* ptr = arr;
ptr = ptr + 3; // avanza 3 bytes (T* escala por sizeof(u8) = 1)
*ptr = 0xAB; // escribe en arr[3]
ptr[2] = 0xCD; // equivalente a *(ptr + 2) = 0xCD
```

### Conversiones de puntero

```java
void* generic = (void*)arr; // cast a puntero generico
u32* palabras = (u32*)arr; // reinterpretacion del buffer
```

La conversion **entero <-> puntero** requiere SIEMPRE un cast explicito -- nunca es
implicita. Con cast, ambas direcciones funcionan:

```java
u8* p    = (u8*)malloc(16);
u64 addr = (u64)p;          // puntero -> entero
u8* q    = (u8*)addr;       // entero  -> puntero
u32* w   = (u32*)addr;      // entero  -> otro tipo de puntero
u8* null = (u8*)0;          // literal entero -> puntero
```

Sin el cast, la asignacion es un **error de compilacion** que indica que apliques
el cast:

```java
u64 bad = p;   // error: la conversion entero<->puntero no es implicita,
               //        usa un cast explicito: (u64)expr
```

---

## Arrays nativos

```java
// Array de tamaño fijo en pila (T[N])
i32 datos[5] = {10, 20, 30, 40, 50};

// Array dinamico (decay a T* al pasar como parametro)
i32[] param_arr; // parametro de funcion (sin tamaño)

void procesar(i32[] arr, i32 len) {
    for (i32 i = 0; i < len; i++) {
        println("${arr[i]}");
    }
}

procesar(datos, 5); // decay implicito de datos a i32*

// Array de structs
Point puntos[3];
puntos[0].x = 1.0;
puntos[0].y = 2.0;
```

---

## Structs (tipos valor)

```java
struct Point {
    f64 x;
    f64 y;
    f64 distance() => sqrt(this.x * this.x + this.y * this.y);
}

Point p;
p.x = 3.0;
p.y = 4.0;
println("dist = ${p.distance()}"); // dist = 5.0
```

Caracteristicas:
- Tipo valor (se copia al asignar; no hay heap allocation).
- No hereda. No tiene metodos virtuales.
- Puede tener metodos (se bajan a funciones libres con puntero a struct).
- Publico por defecto.

### Bit fields

```java
struct Flags {
    u32 activo : 1; // 1 bit
    u32 nivel : 3; // 3 bits (0..7)
    u32 codigo : 8; // 8 bits (0..255)
    u32 resto : 20; // 20 bits restantes
}

Flags f;
f.activo = 1;
f.nivel = 5;
f.codigo = 0xAB;
println("activo=${f.activo} nivel=${f.nivel} codigo=${f.codigo}");
```

### Init lists

```java
// Posicional (en orden de declaracion de campos)
Point p1 = {3.0, 4.0};

// Designado (con nombres de campos, cualquier orden)
Point p2 = {.y = 4.0, .x = 3.0};

// Parcial (campos no inicializados quedan con valor del allocator)
Point p3 = {.x = 1.0};

// Typedef struct estilo C
typedef struct { f64 x; f64 y; } Vec2;
Vec2 v = {1.0, 2.0};
```

### Multi-declarador de campo (`T a, b;`)

Varios campos del mismo tipo se declaran en una sola linea, al estilo C. Cada
declarador puede llevar sus propias dimensiones de array y bit-width:

```java
struct Rect {
    i32 x0, y0;          // dos campos i32
    i32 x1, y1;
    u8  flags : 4, kind : 4;   // dos bit fields
}
```

Funciona en struct plano, en `typedef struct { ... }` y en agregados anonimos
inline (union/struct anidados).

### Struct C-tagless (nombre al final)

Al portar cabeceras C se acepta la forma con el nombre **despues** del cuerpo, sin
`typedef`, incluyendo alias de puntero:

```java
struct {
    u8  request_type;
    u16 value, index, length;
} SETUP, *PSETUP;        // SETUP es el tipo; PSETUP = SETUP*
```

Es equivalente a `typedef struct { ... } SETUP, *PSETUP;`.

### Struct opaco (forward-decl, `typedef struct Tag *P;`)

Un `typedef struct Tag *P, *LP;` **sin cuerpo** declara un struct OPACO
(incompleto), idioma C de handle opaco.  Registra `Tag` como un tipo incompleto
usable solo **via puntero** (8 bytes); los alias `P`, `LP` apuntan a `Tag`.  Una
definicion posterior `struct Tag { ... }` lo **completa** (a partir de ahi es
derefenciable).  Mientras siga incompleto, sirve como handle que se pasa a APIs
sin conocer su contenido.

```java
// forward-decl opaco: PCTX/LPCTX son punteros a un _CTX aun sin cuerpo.
typedef struct _CTX *PCTX, *LPCTX;

i64 as_handle(PCTX c) { return (i64)c; }   // se usa como handle, sin derefenciar

// completacion: definir el struct despues -> ahora es derefenciable.
struct _CTX { i32 a; i32 b; }

_CTX c; c.a = 1; c.b = 2;
PCTX p = &c;
i32 s = (*p).a + (*p).b;   // OK: _CTX ya esta completo
```

### Alineacion del struct (`@align(N)`)

`@align(N)` sobre una declaracion de struct fuerza la alineacion del layout a
`max(natural, N)` y padea el tamano a un multiplo de esa alineacion (nunca reduce
la natural). Equivale a `__declspec(align(N))` / `_Alignas(N)` de C:

```java
@align(16)
struct Vec4 { f32 x, y, z, w; }    // alineado a 16, tamano padeado a 16
```

Ver tambien la seccion "Alineacion forzada" mas abajo (para newtypes).

### Enum ADT como campo

Un enum **ADT sin payload** (`enum Color { Red, Green, Blue }`) puede usarse como
campo de struct: su valor es un unico qword (el tag), ocupa 8 bytes y se
asigna/lee como un escalar.

```java
enum Color { Red, Green, Blue }
struct Cell {
    u8    x;
    Color c;          // enum ADT como campo (8 bytes)
    u8    y;
}
Cell cell;
cell.c = Color.Blue;
match cell.c { case Blue => { /* ... */ } /* ... */ }
```

Para un ancho C-exacto (4 bytes) usa un enum con backing entero: `enum Color : u32
{ ... }`. Un enum **con payload** todavia no puede ser campo de struct (se rechaza
con un error claro; usa un puntero o un enum con backing).

---

## Enums (tipos algebraicos / ADTs)

> Vesta tiene **dos** clases de enum: los **ADT** (union etiquetada) de esta
> seccion, y los **con valor** C-style (`enum Op : u8 { A = 0x01 }`, backing
> int/float/string/struct/clase). La referencia completa de ambos -- mas
> concepts (`is_enum`/`Enum`/`ValuedEnum`) y uso en comptime -- esta en
> [[Enums]].

```java
enum Color {
    Red, // variante sin payload
    Green,
    Blue
}

enum Shape {
    Circle(f64), // radio
    Rectangle(f64, f64), // ancho, alto
    Point // sin payload
}
```

### Pattern matching

```java
Color c = Color.Red;

// Match sobre enum sin payload
match c {
    case Red => println("rojo");
    case Green => println("verde");
    case Blue => println("azul");
}

// Match sobre enum con payload (destructuring)
Shape s = Shape.Circle(5.0);

f64 area = match s {
    case Circle(r) => 3.14159 * r * r;
    case Rectangle(w, h) => w * h;
    case Point => 0.0;
};

// El compilador exige exhaustividad o un arm _ por defecto
match c {
    case Red => ...;
    case _ => println("otro color"); // arm por defecto
}
```

### Payloads complejos: mezcla de aridades y tipos heterogeneos

Los enums no estan limitados a un solo tipo de payload por variante.  Cada
variante puede declarar entre 0 y N campos, con tipos mezclados libremente:

```java
enum Shape {
    Empty,                              // 0 payloads
    Circle(i32),                        // 1 payload entero
    Rectangle(i32, i32),                // 2 payloads enteros
    Triangle(f64, f64, f64),            // 3 payloads f64
    Labeled(string, i32)                // mixto: string + i32
}
```

El layout en memoria es uniforme: 8 bytes para el tag + slots de 8 bytes
para cada payload (max payload count entre todas las variantes).  Variantes
con menos payloads dejan los slots sobrantes sin usar.  Pasarlos por valor
entre funciones usa SRET: el caller aloca el buffer y el callee escribe el
tag + payloads.

### Match con returns tempranos y bindings tipados

Cada arm del `match` puede tener un bloque `{ ... }` con statements
arbitrarios, incluyendo `return` temprano para salir de la funcion
enclosing:

```java
i32 score_of(Shape s) {
    match s {
        case Empty => return 0;
        case Circle(r) => return r * 3;
        case Rectangle(w, h) => return (w + h) * 2;
        case Triangle(a, b, c) => {
            f64 sum = a + b + c;
            return (i32)sum;
        }
        case Labeled(name, weight) => {
            // name es string, weight es i32
            return weight * 2;
        }
    }
    return -1; // inalcanzable si el match es exhaustivo
}
```

Los bindings (`r`, `w`, `h`, `a`, `b`, `c`, `name`, `weight`) se tipan
automaticamente segun el orden de declaracion de la variante.  El type
checker valida que la aridad del binding coincide con la variante.

### Combinacion con structs

Los payloads de enum pueden ser cualquier tipo, incluyendo structs
definidos por el usuario.  Combinado con `match`, permite construir
representaciones intermedias tipo AST:

```java
struct Point {
    i32 x;
    i32 y;
}

enum Drawable {
    Pixel(Point),
    Line(Point, Point),
    Circle(Point, i32)            // centro + radio
}

i32 area_or_length(Drawable d) {
    match d {
        case Pixel(p) => return 1;
        case Line(a, b) => return (b.x - a.x) + (b.y - a.y);
        case Circle(c, r) => return r * r * 3;
    }
    return 0;
}
```

Ejemplo extenso: `examples_codes_vx/176_adt_complex_payloads.vx` combina
5 variantes con aridades 0-3, payloads de tipos mezclados (i32, f64,
string), retornos tempranos en cada arm, y multiples ADTs interactuando.

---

## ``Optional<T>``

`Optional<T>` modela un valor que puede existir o no. Es un tipo builtin del compilador
(no una clase generica), con layout de 16 bytes en pila del caller.

```java
Optional<i32> buscar(i32[] arr, i32 valor) {
    for (i32 i = 0; i < 10; i++) {
        if (arr[i] == valor) return Some(i); // encontrado
    }
    return None(); // no encontrado
}

Optional<i32> idx = buscar(datos, 42);

// Comprobar y extraer:
if (isPresent(idx)) {
    i32 pos = unwrap(idx);
    println("Encontrado en posicion ${pos}");
}

// Operador !! (lanza FatalError si None):
i32 pos = !!buscar(datos, 42);

// Auto-wrap: asignar primitivo a Optional<T>
Optional<i32> opt = 50; // equivale a Some(50)
Optional<i32> vacio = null; // equivale a None()
```

---

## Result<V, E>

`Result<V, E>` modela exito con valor `V` o fracaso con error `E`. Layout de 24 bytes en
pila del caller (cero heap). El tipo de error puede ser primitivo, struct o string.

```java
Result<i32, string> dividir(i32 a, i32 b) {
    if (b == 0) return Err("division por cero");
    return Ok(a / b);
}

// Must-handle: usar el resultado de una funcion que retorna Result es obligatorio
Result<i32, string> r = dividir(10, 2);

if (isOk(r)) {
    println("Resultado: ${value(r)}");
} else {
    println("Error: ${error(r)}");
}

// También: implicit Some no aplica a Result (solo a Optional)
```

---

## Tipos de referencia (clases)

Las instancias de clases son referencias nullable por defecto:

```java
Animal a = new Animal(5, "Rex"); // nullable
Animal b = null; // valido

// Comprobar nulidad:
if (a != null) {
    println("${a.nombre}");
}

// Keyword nonnull (error de compilacion si se asigna null):
nonnull Animal c = new Animal(3, "Fido");
// c = null; // ERROR en compilacion

// Operador !! en llamadas (lanza FatalError si null):
Animal d = getAnimal()!!;
```

---

## Tipo funcion

```java
// Tipo funcion: fn(T1, T2, ...) -> R
fn(i32, i32) -> i32 suma = (a, b) => a + b;
fn(i32) -> i32 doble = (x) => x * 2;

// Sin parametros:
fn() -> void accion = => { 
    println("hola"); 
};

// HOF (Higher Order Function):
i32 aplicar(fn(i32) -> i32 f, i32 x) {
    return f(x);
}

println("${aplicar(doble, 21)}"); // 42
```

---

## Aliases de tipo

Hay dos familias: **aliases transparentes** (renombran sin crear un tipo
nuevo) y **newtypes** (crean un tipo nominalmente distinto con la misma
representacion runtime).

### Aliases transparentes

Sintaxis equivalente entre `typedef` (estilo C) y `using` (estilo C++):

```c
typedef u32 Edad; // alias estilo C
using MyInt = i64; // alias estilo C++ moderno
using Callback = fn(i32) -> void; // alias de tipo funcion

Edad edad = 25; // Edad y u32 son intercambiables
Callback cb = (x) => println("${x}");
```

Estos aliases NO crean tipos nuevos: pasar un `Edad` donde se espera un
`u32` (o viceversa) es legal sin cast. Util para legibilidad y para
abreviar tipos largos (`fn(i32, i32) -> Result<i64, string>`).

### Newtypes (`typedef T name new`)

Cuando lo que se busca es **distinguir** dos valores que tienen el
mismo ancho fisico pero significan cosas distintas (un `user_id` no es
intercambiable con un `group_id` aunque ambos sean `u64`), se añade el
keyword `new`:

```c
typedef u64 user_id new;
typedef u64 group_id new;

user_id alice = (user_id) 101;
group_id admins = (group_id) 7;

// alice = admins; // ERROR: tipos distintos
// u64 x = alice; // ERROR: requiere cast explicito
u64 raw = (u64) alice; // OK: cast explicito a underlying
```

Diferencias clave respecto al alias transparente:

| Operacion | Alias (`using Edad = u32`) | Newtype (`typedef u32 Edad new`) |
|:----------|:---------------------------|:----------------------------------|
| `Edad e = 25;` | OK | ERROR (literal u32) |
| `u32 x = e;` | OK | ERROR (sin cast) |
| `Edad e = (Edad) 25;` | OK | OK |
| `u32 x = (u32) e;` | OK | OK |
| `is_newtype<Edad>()` | `false` | `true` |

**Coste runtime:** cero. El newtype ocupa exactamente los mismos bytes
que su underlying; el cast explicito baja al mismo BITCAST que cualquier
otra conversion entre tipos del mismo ancho. Toda la distincion vive
en el type checker.

#### Sobre tipos propios: struct, clase y enum

El underlying no tiene que ser un primitivo. Un newtype sobre un tipo propio
conserva **todo** lo suyo — campos, métodos, variantes, tamaño — y sólo añade la
identidad:

```vx
struct Punto { i64 x; i64 y; public i64 suma() { return this.x + this.y; } }

typedef Punto Coord new;
typedef Punto Pixel new;      // misma forma, otro significado

i64 distancia(Coord c) { return c.suma(); }   // solo acepta Coord

Coord c;
c.x = 1;  c.y = 2;            // los campos, como en Punto
distancia(c);                  // OK

Pixel p;
distancia(p);                  // error: Pixel no es Coord
```

Ese último error es el motivo de todo: dos cosas con la misma representación
pero distinto significado (metros y pies, un id de sesión y uno de cuenta) dejan
de poder confundirse, y se dice en compilación en vez de en producción.

Funciona igual con clases y enums:

```vx
class Caja { public i64 v = 0; public i64 leer() { return this.v; } }
typedef Caja Sesion new;

Sesion s = new Sesion();      // construye la Caja de debajo
s.v = 7;
s.leer();                      // sus metodos

enum Color { Rojo, Verde, Azul }
typedef Color Tinta new;

Tinta t = Tinta.Verde;         // sus variantes
match t {
    case Verde => ...;
    case _     => ...;
}
```

Un newtype no declara nada propio: no genera constructor, ni layout, ni copia
del código. `new Sesion()` construye la `Caja`, y sus métodos son los de `Caja`.
Lo único que no comparte es la identidad.

Ejemplo completo: `320_newtype_tipos_usuario.vx`.

### Newtypes opacos (`@opaque`)

Para handles cuya representacion interna no debe leer el cliente:

```c
typedef u64 session_id new @opaque;

// session_id s = (session_id) 0xABCD; // ERROR: opaque bloquea el cast
// u64 raw = (u64) my_session; // ERROR: bits ocultos
```

Sin el bloque `{ ... }` (ver mas abajo), un newtype `@opaque` no admite
NINGUN cast. La unica forma de obtener un valor es desde una funcion
que ya lo devuelva (por ejemplo, un `extern fn open() -> session_id`
o un builtin que lo construya internamente).

### Alineacion forzada (`@align(N)`)

Util para SIMD, GPU descriptors, DMA, formatos binarios donde el
padding debe ser exacto:

```c
typedef u8 simd_byte new @align(16); // align 16-byte
typedef u8 cl_byte new @align(64); // align cache-line

struct Packet {
    u32 magic; // offset 0
    simd_byte payload; // offset 16 (forzado a 16)
}
// sizeof(Packet) es multiplo de 16
```

`N` debe ser potencia de 2 en `[1, 4096]` (1, 2, 4, 8, 16, 32, 64,
128, 256, 512, 1024, 2048, 4096). El frontend padea el campo y
redondea el tamaño total del struct contenedor.

Las anotaciones `@opaque` y `@align(N)` se pueden combinar en cualquier
orden:

```c
typedef u64 aligned_handle new @opaque @align(8);
typedef u64 aligned_handle new @align(8) @opaque; // equivalente
```

### Conversiones declaradas (`explicit from/to T`)

A veces se quiere relajar (en opaque) o restringir (en no-opaque) el
conjunto de casts permitidos. El bloque `{ ... }` declara las
conversiones legales:

```c
typedef u64 file_id new {
    explicit from u32; // (file_id) algun_u32 -- OK
    public explicit to u64; // (u64) algun_file_id -- OK cross-file
};

file_id fid = (file_id) (u32) 42; // OK
// file_id bad = (file_id) (u64) 9; // ERROR: u64 no esta en from-list
u64 raw = (u64) fid; // OK
```

Reglas:

- Si el newtype es **no-opaco** y NO declara bloque, acepta cast libre
 con su underlying (default permisivo).
- Si declara bloque, solo las conversiones listadas son legales.
- Match estricto por kind: `explicit from u32` rechaza `u64`, `i32`, etc.

### Privacidad por modulo (`public` vs por defecto)

Cada `typedef ... new` recuerda en que fichero `.vx` se declaro. Las
conversiones declaradas son **privadas al fichero** por defecto; el
modificador `public` las hace accesibles desde otros ficheros que
importen el tipo:

```c
// session.vx
typedef u64 session_handle new @opaque {
    public explicit from u64; // cualquier .vx puede construir
    explicit to u64; // solo este fichero extrae bits
};
```

Casos cubiertos:

| Contexto | `explicit` sin `public` | `public explicit` |
|:---------|:------------------------|:------------------|
| Cast en el fichero del typedef | OK | OK |
| Cast en otro fichero | ERROR | OK |

Esto permite tener APIs publicas de construccion (factory) mientras se
mantienen las operaciones de extraccion confinadas al modulo
implementador.

### Introspeccion compile-time

Los builtins comptime distinguen newtypes de aliases transparentes:

```c
is_newtype<user_id>() // true
is_newtype<u64>() // false
is_opaque<session_id>() // true
is_opaque<user_id>() // false
underlying_of<user_id>() // "u64" (typename del underlying)

// is_same compara identidad nominal, no underlying:
is_same<user_id, group_id>() // false (ambos son u64, pero IDs distintos)
is_same<user_id, user_id>() // true
```

Util en macros `@Macro` para generar codigo distinto segun el tipo:

```c
@Macro string emit_validator(Type T) {
    if (is_newtype<T>() && is_opaque<T>()) {
        return "/* opaque: nada que validar externamente */";
    }
    return "/* validacion estandar del underlying */";
}
```

### Patron recomendado: handles de sistema

```c
// fs.vx
typedef u64 fd new @opaque {
    public explicit from u64; // el FFI puede convertir el handle OS
    explicit to u64; // solo fs.vx serializa internamente
};

// Esto compila en fs.vx pero NO en otro fichero:
fd open_file(string path) {
    u64 raw = native_open(str_cstr(path)); // FFI
    return (fd) raw; // OK aqui (mismo fichero)
}

// fs.vx tambien puede serializar a u64 para logging interno:
void log_fd(fd f) {
    println("fd raw=${(u64) f}"); // OK (sin public, mismo file)
}
```

```c
// main.vx (otro fichero)
fd f = open_file("/tmp/test"); // OK: open_file devuelve fd
// u64 raw = (u64) f; // ERROR: 'to u64' no es public

// Pero podemos crear uno via la public-from si lo necesitamos:
fd f2 = (fd) (u64) raw_imported_from_protocol; // OK (from u64 es public)
```

### Tabla de comportamiento (resumen)

| Caso | Sin bloque | Con bloque |
|:-----------------------------------------------|:----------:|:-----------:|
| Newtype no-opaco, cast con underlying | OK | Solo si listado |
| Newtype no-opaco, cast con otro tipo | ERROR | Solo si listado |
| @opaque, cualquier cast | ERROR | Solo si listado |
| Cast en mismo fichero, sin `public` | (regla anterior) | OK si listado |
| Cast en otro fichero, sin `public` | (regla anterior) | ERROR |
| Cast cualquier fichero, con `public` | (regla anterior) | OK si listado |

---

## Resumen de la jerarquia de tipos

```java
Tipos primitivos:
i8 i16 i32 i64
u8 u16 u32 u64
f32 f64
bool char void

Tipos de cadena:
string (GC-managed StringObject)
char* (C-style, host pointer)
cstring (alias de char*)

Tipos de puntero:
T* (host, acceso movh)
VirtualPtr<T> (VM, acceso mov)
void* (puntero generico host)

Tipos de array:
T[N] (stack, tamaño fijo)
T[] (decay a T* en parametros)
Array<T> (GC-managed, dinamico)

Tipos compuestos:
struct (valor, sin herencia)
enum (ADT, pattern matching)
class (referencia, GC, herencia)
interface (referencia abstracta)

Tipos funcionales:
fn(T1,T2)->R (closure o puntero a funcion)

Tipos algebraicos builtin:
Optional<T> (16 bytes, pila)
Result<V,E> (24 bytes, pila)
tuple (T1,T2,...) (valor, sin nombre)

Tipos genericos:
Box<T>, List<T>, etc. (monomorphizacion compile-time)
```

---

Ver tambien: [[OOP]], [[Generics]], [[FFI]], [[SetInstruccionesVM/STRINGS]]
