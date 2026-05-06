# Sistema de tipos de Vex

Vex es estaticamente tipado con inferencia local. El compilador conoce el tipo de cada
expresion en tiempo de compilacion. No hay tipos dinamicos ni boxing implicito.

---

## Tipos primitivos

### Enteros con signo

| Tipo    | Alias   | Tamano | Rango                                |
| :------ | :------ | :----- | :----------------------------------- |
| `i8`    | `int8_t`  | 1 byte  | -128 .. 127                         |
| `i16`   | `int16_t` | 2 bytes | -32 768 .. 32 767                   |
| `i32`   | `int32_t` | 4 bytes | -2 147 483 648 .. 2 147 483 647     |
| `i64`   | `int64_t` | 8 bytes | -9.2e18 .. 9.2e18                   |

### Enteros sin signo

| Tipo    | Alias    | Tamano | Rango                      |
| :------ | :------- | :----- | :------------------------- |
| `u8`    | `uint8_t`  | 1 byte  | 0 .. 255                  |
| `u16`   | `uint16_t` | 2 bytes | 0 .. 65 535               |
| `u32`   | `uint32_t` | 4 bytes | 0 .. 4 294 967 295        |
| `u64`   | `uint64_t` | 8 bytes | 0 .. 18.4e18              |

Ambas familias son aliases resueltos por el type checker; `i32` e `int32_t` son el mismo
tipo.

### Punto flotante

| Tipo    | Alias   | Tamano  | Precision         |
| :------ | :------ | :------ | :---------------- |
| `f32`   | `float` | 4 bytes | IEEE 754 simple   |
| `f64`   | `double`| 8 bytes | IEEE 754 doble    |

```java
f64 pi = 3.14159265358979;
f32 radio = 5.0f;           // sufijo f indica f32 literal
f64 area = pi * radio * radio;
println("Area: ${area}");
```

### Booleano

```java
bool activo = true;
bool listo  = false;
bool resultado = (x > 0) && !activo;
```

### Caracter

```java
char letra = 'A';       // codepoint Unicode (u32 internamente)
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
string mensaje = saludo + " " + nombre;   // concatenacion O(1) ROPE

i32 len  = mensaje.length();   // numero de code points
i32 bytes = mensaje.bytes();   // numero de bytes en encoding
u8* raw  = mensaje.cstr();     // puntero host nul-terminado (para FFI)
```

### Metodos de string

| Metodo              | Retorno    | Descripcion                               |
| :------------------ | :--------- | :---------------------------------------- |
| `s.length()`        | `i32`      | Numero de code points                     |
| `s.bytes()`         | `i32`      | Numero de bytes en el encoding actual     |
| `s.cstr()`          | `char*`    | Puntero host nul-terminado (ASCII/UTF-8)  |
| `s.wstr()`          | `char*`    | Puntero host UTF-16LE (para Win32 *W API) |
| `s.hash()`          | `u64`      | Hash FNV-1a (cacheado)                    |
| `s.intern()`        | `string`   | Handle canonico del pool de interning     |
| `s.equals(t)`       | `bool`     | Comparacion lexicografica de bytes        |
| `s.concat(t)`       | `string`   | Concatenacion (alias de `s + t`)          |

### Operadores de string

```java
string a = "abc";
string b = "def";

string c  = a + b;          // concatenacion: "abcdef"
bool igual = a == b;        // false
bool dist  = a != b;        // true

// Auto-coercion de literales:
string d = "inicio_" + a;   // el literal se promueve a StringObject
```

### Encodings

```java
// Constantes de encoding (para str_convert):
// ENC_ASCII = 0, ENC_ANSI = 1, ENC_UTF8 = 2, ENC_UTF16 = 3, ENC_UTF32 = 4

string utf8  = str_make(buffer_ptr, len);
string utf16 = str_convert(utf8, ENC_UTF16);
char*  wptr  = utf16.wstr();    // listo para Win32 *W API
```

---

## Tipos de puntero

Vex tiene dos tipos de puntero con semantica bien diferenciada:

| Tipo            | Apunta a       | Acceso en bytecode | Uso tipico              |
| :-------------- | :------------- | :----------------- | :---------------------- |
| `T*`            | Memoria HOST   | `movh`             | malloc, FFI, str_cstr() |
| `VirtualPtr<T>` | Memoria VM     | `mov`              | variables locales, `&x` |

### Puntero host (`T*`)

```java
// malloc devuelve un puntero host
u8* buf = malloc(256);
buf[0] = 0xFF;              // escribe en memoria host via movh
free(buf);

// FFI: puntero host a cadena de caracteres
char* raw = str.cstr();
// ... pasar raw a una API nativa ...
```

### Puntero virtual (`VirtualPtr<T>`)

```java
// address-of de una variable local produce VirtualPtr
i32 x = 42;
VirtualPtr<i32> p = &x;    // p apunta a la pila VM
*p = 100;                   // escribe en VM memory
println("x = ${*p}");       // x = 100
```

### Aritmetica de punteros

```java
u8* arr = malloc(10);
u8* ptr = arr;
ptr = ptr + 3;              // avanza 3 bytes (T* escala por sizeof(u8) = 1)
*ptr = 0xAB;                // escribe en arr[3]
ptr[2] = 0xCD;              // equivalente a *(ptr + 2) = 0xCD
```

### Conversiones de puntero

```java
void* generic = (void*)arr;         // cast a puntero generico
u32*  palabras = (u32*)arr;         // reinterpretacion del buffer
```

---

## Arrays nativos

```java
// Array de tamano fijo en pila (T[N])
i32 datos[5] = {10, 20, 30, 40, 50};

// Array dinamico (decay a T* al pasar como parametro)
i32[] param_arr;                         // parametro de funcion (sin tamano)

void procesar(i32[] arr, i32 len) {
    for (i32 i = 0; i < len; i++) {
        println("${arr[i]}");
    }
}

procesar(datos, 5);                      // decay implicito de datos a i32*

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
println("dist = ${p.distance()}");    // dist = 5.0
```

Caracteristicas:
- Tipo valor (se copia al asignar; no hay heap allocation).
- No hereda. No tiene metodos virtuales.
- Puede tener metodos (se bajan a funciones libres con puntero a struct).
- Publico por defecto.

### Bit fields

```java
struct Flags {
    u32 activo : 1;    // 1 bit
    u32 nivel  : 3;    // 3 bits (0..7)
    u32 codigo : 8;    // 8 bits (0..255)
    u32 resto  : 20;   // 20 bits restantes
}

Flags f;
f.activo = 1;
f.nivel  = 5;
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

---

## Enums (tipos algebraicos / ADTs)

```java
enum Color {
    Red,                    // variante sin payload
    Green,
    Blue
}

enum Shape {
    Circle(f64),            // radio
    Rectangle(f64, f64),    // ancho, alto
    Point                   // sin payload
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
    case Circle(r)    => 3.14159 * r * r;
    case Rectangle(w, h) => w * h;
    case Point        => 0.0;
};

// El compilador exige exhaustividad o un arm _ por defecto
match c {
    case Red => ...;
    case _   => println("otro color");   // arm por defecto
}
```

---

## ``Optional<T>``

`Optional<T>` modela un valor que puede existir o no. Es un tipo builtin del compilador
(no una clase generica), con layout de 16 bytes en pila del caller.

```java
Optional<i32> buscar(i32[] arr, i32 valor) {
    for (i32 i = 0; i < 10; i++) {
        if (arr[i] == valor) return Some(i);   // encontrado
    }
    return None();                              // no encontrado
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
Optional<i32> opt = 50;    // equivale a Some(50)
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
Animal a = new Animal(5, "Rex");   // nullable
Animal b = null;                    // valido

// Comprobar nulidad:
if (a != null) {
    println("${a.nombre}");
}

// Keyword nonnull (error de compilacion si se asigna null):
nonnull Animal c = new Animal(3, "Fido");
// c = null;   // ERROR en compilacion

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
fn() -> void accion = () => { 
	println("hola"); 
};

// HOF (Higher Order Function):
i32 aplicar(fn(i32) -> i32 f, i32 x) {
    return f(x);
}

println("${aplicar(doble, 21)}");    // 42
```

---

## Aliases de tipo

```c
typedef u32 Edad;            // alias estilo C
using MyInt = i64;           // alias estilo C++ moderno
using Callback = fn(i32) -> void;   // alias de tipo funcion

Edad edad = 25;
Callback cb = (x) => println("${x}");
```

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
  char*  (C-style, host pointer)
  cstring (alias de char*)

Tipos de puntero:
  T*           (host, acceso movh)
  VirtualPtr<T> (VM, acceso mov)
  void*        (puntero generico host)

Tipos de array:
  T[N]         (stack, tamano fijo)
  T[]          (decay a T* en parametros)
  Array<T>     (GC-managed, dinamico)

Tipos compuestos:
  struct       (valor, sin herencia)
  enum         (ADT, pattern matching)
  class        (referencia, GC, herencia)
  interface    (referencia abstracta)

Tipos funcionales:
  fn(T1,T2)->R (closure o puntero a funcion)

Tipos algebraicos builtin:
  Optional<T>  (16 bytes, pila)
  Result<V,E>  (24 bytes, pila)
  tuple (T1,T2,...) (valor, sin nombre)

Tipos genericos:
  Box<T>, List<T>, etc. (monomorphizacion compile-time)
```

---

Ver tambien: [[OOP]], [[Generics]], [[FFI]], [[SetInstruccionesVM/STRINGS]]
