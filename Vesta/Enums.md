# Enums: uniones etiquetadas y enums con valor

Vesta tiene **dos** clases de enum, distinguidas por la sintaxis y por lo que
cada variante *es*:

| Clase | Sintaxis | Una variante es... | Uso tipico |
| :---- | :------- | :----------------- | :--------- |
| **ADT** (union etiquetada) | `enum E { A, B(i32), C(i32, i32) }` | un *caso* con payload opcional | modelar datos con forma variable; se consume con `match` |
| **Con valor** (C-style) | `enum E : <tipo> { A = <valor>, ... }` | una *constante* del `<tipo>` base | opcodes, flags, codigos, tablas |

La regla para distinguirlas es el `: <tipo>` tras el nombre: si esta, el enum es
**con valor**; si no, es **ADT**. Ambas coexisten en el mismo programa y son
compile-time puras -- no hay descriptor de enum en runtime, ni dispatch dinamico,
ni coste oculto. Todo funciona identico en interprete, JIT y compilacion nativa
(AOT).

---

## 1. Enums ADT (uniones etiquetadas)

Un enum ADT es una union etiquetada: un valor es exactamente uno de sus casos,
y cada caso puede llevar un payload propio.

```vesta
enum Shape {
    Circle(i32),           // payload: un i32 (radio)
    Rect(i32, i32),        // payload: dos i32 (ancho, alto)
    Point                  // sin payload
}
```

Se construye nombrando el caso y, si lo tiene, su payload:

```vesta
Shape a = Shape.Circle(5);
Shape b = Shape.Rect(3, 4);
Shape c = Shape.Point;
```

Se consume con `match`, que hace destructuring del payload y exige
exhaustividad (todos los casos, o un comodin `_`):

```vesta
i32 area(Shape s) {
    match (s) {
        case Circle(r)   => return r * r * 3;
        case Rect(w, h)  => return w * h;
        case Point       => return 0;
    }
}
```

Los payloads pueden ser cualquier tipo (primitivos, structs, otros enums). El
detalle completo de `match` (guardas, bindings, comodines, rangos) esta en
[[ControlFlow]]; el modelado de datos con ADTs, en [[TiposDatos]].

Los ADT son **genericos**: `enum Maybe<T> { Some(T), None }` monomorfiza por tipo
concreto (ver [[Generics]]).

---

## 2. Enums con valor (C-style)

Un enum con valor declara constantes de un tipo base. Cada variante *es* un valor
de ese tipo, usable directamente donde ese tipo encaje -- sin cast, sin
desenvoltura.

```vesta
enum Op : u8 {
    ADD = 0x01,
    SUB = 0x29,
    MOV = 0x89
}
```

El tipo base (**backing**) puede ser entero, float, string, struct o clase.

### 2.1. Backing entero

Valor explicito o **auto-incremento** (empieza en 0, o en el ultimo valor + 1):

```vesta
enum Reg : u8 { RAX, RCX, RDX, RBX }        // 0, 1, 2, 3
enum Flag : u32 { NONE = 0, READ = 1, WRITE = 2, EXEC = 4 }
```

Una variante entera se usa **directo, sin cast**, en aritmetica y comparaciones
con su tipo base -- *es* un `u8` (o el ancho que declares):

```vesta
u8  opcode = Op.MOV;                 // 0x89
u8  reg    = Reg.RBX;                // 3
u8  modrm  = 0xC0 + reg;             // aritmetica directa
if (opcode == Op.MOV) { /* ... */ }  // comparacion directa
```

### 2.2. Backing float, string

```vesta
enum Angle : f64 {
    ZERO    = 0.0,
    QUARTER = 1.5707963,
    HALF    = 3.1415926
}

enum Verb : string {
    GET  = "GET",
    POST = "POST",
    PUT  = "PUT"
}
```

Un enum float es un `f64`; uno string es un `string` (con sus operadores `==`,
`+`, interpolacion):

```vesta
f64 h = Angle.HALF;                          // 3.1415926
if (Verb.PUT == "PUT") { /* ... */ }         // comparacion de string
string s = "verb=" + Verb.GET;               // "verb=GET"
```

### 2.3. Backing struct, clase

Cuando el backing es un struct o una clase, **una variante ES un valor de ese
tipo**: un `Color` *es* un `Rgb`, un `Anchor` *es* un `Point`. Cada variante es
una constante compile-time construida con la sintaxis normal de ese tipo.

```vesta
struct Rgb { u8 r; u8 g; u8 b; }
enum Color : Rgb {
    RED   = (Rgb){255, 0, 0},
    GREEN = (Rgb){0, 255, 0},
    BLUE  = (Rgb){0, 0, 255}
}

class Point {
    public i32 x;
    public i32 y;
    public Point(i32 a, i32 b) { this.x = a; this.y = b; }
}
enum Anchor : Point {
    ORIGIN = new Point(0, 0),
    CORNER = new Point(10, 20)
}
```

Se accede a los campos del valor de la variante directamente:

```vesta
Rgb   g = Color.GREEN;    u8  green = g.g;      // 255
Point c = Anchor.CORNER;  i32 sum   = c.x + c.y; // 30
```

Como un `Color` *es* un `Rgb`, es indistinguible de su tipo base a efectos de
tipos (esto es el diseno, no una limitacion): puedes pasarlo a cualquier sitio
que espere un `Rgb`.

### 2.4. Namespaces

Los enums con valor se cualifican por namespace como cualquier tipo:

```vesta
import isa.x86;

u8 op = x86.Op.MOV;
```

### 2.5. Estilo C: `typedef enum` y backing inferido

Para portar cabeceras C casi verbatim, un enum **sin** `: tipo` explicito tambien
es un enum con valor si alguna variante lleva un valor (o si usas la forma
`typedef enum`). El ancho del backing se **infiere**:

```vesta
// typedef enum al estilo C: SIEMPRE es un enum con valor entero.
typedef enum {
    A = 1,
    B,          // 2  (auto-incremento)
    C = 18,
    D           // 19
} E1;

// enum "pelado" con valores explicitos: tambien con valor (no ADT).
enum Color { Red = 0, Green, Blue }     // 0, 1, 2
```

Reglas de inferencia del backing (imitan a C):

- Si **todas** las variantes caben en `int` (`i32`), el backing es `i32`.
- Si algun valor no cabe en `i32` pero si en `u32` (p.ej. `0xFFFFFFFF`), se
  ensancha a `u32`; si tampoco, a `i64`.
- Si las variantes son **strings**, el backing se infiere `string`.
- Puedes forzar el ancho con `: tipo` explicito: `typedef enum : u8 { P = 40, Q } S;`.

```vesta
typedef enum { LOW = 0x1, BIG = 0xFFFFFFFF } Flags;   // backing u32 (BIG no cabe en i32)
typedef enum { OK = "ok", ERR = "err" } Names;        // backing string
```

Un `enum Nombre { Rojo, Verde }` **pelado y sin valores ni payloads** sigue siendo
un **ADT** (union etiquetada). La presencia de un `= valor` (o de la palabra
`typedef enum`) es lo que lo convierte en enum con valor. Los payloads
(`enum E { A(i32) }`) siempre marcan un ADT.

---

## 3. Enums y concepts

Un **concepto** es un predicado compile-time sobre un tipo (ver [[Generics]]).
Los enums participan del sistema de concepts de tres formas.

### 3.1. Introspeccion: `is_enum<T>()`

`is_enum<T>()` es un builtin comptime que da `true` para cualquier enum -- ADT o
con valor de backing entero/float/string. (Los de backing struct/clase son su
tipo base por diseno, asi que para esos da `false`: un `Color` se ve como un
`Rgb`.)

```vesta
enum Shape { Circle(i32) }           // ADT
enum Op : u8 { MOV = 0x89 }          // con valor

is_enum<Shape>()   // true
is_enum<Op>()      // true
is_enum<i32>()     // false
```

### 3.2. Concepts built-in `Enum` y `ValuedEnum`

Como bound (`<T: Concepto>`) o como predicado directo:

| Concepto | Se satisface por... |
| :------- | :------------------ |
| `Enum` | cualquier enum (ADT o con valor) |
| `ValuedEnum` | solo los C-style con valor (los ADT no tienen valor) |

```vesta
i32 count<T: Enum>(T x)      { return 1; }   // acepta ADT y con-valor
i32 tag<T: ValuedEnum>(T x)  { return 7; }   // solo con-valor
```

Ademas, un enum con valor **satisface el concepto de su backing**: un `enum : u8`
cumple `Numeric` e `Integer` (es un `u8`); uno `: string` cumple `String`; etc.

```vesta
i32 as_num<T: Numeric>(T x) { return (i32) x; }
as_num(Op.MOV);   // OK: Op es u8 -> Numeric
```

### 3.3. Conceptos de usuario apoyados en `is_enum`

```vesta
concept AnyEnum<T> = is_enum<T>();
concept IntEnum<T> = is_enum<T>() && Numeric<T>();   // composicion

i32 mark<T: AnyEnum>(T x) { return 1; }
```

### 3.4. Concepto como predicado directo

Cualquier concepto (built-in o de usuario) se puede invocar como una funcion
booleana comptime, no solo como bound. Se dobla a una constante -- cero runtime.

```vesta
if (Enum<Shape>())       { /* ... */ }
if (ValuedEnum<Op>())    { /* ... */ }
if (Numeric<Op>())       { /* ... */ }
bool b = AnyEnum<Op>();
```

Un bound violado es un **error de compilacion** con mensaje claro:

```
error: el tipo 'Shape' no satisface el concepto 'ValuedEnum' exigido por el parametro 'T'
```

---

## 4. Enums en comptime

Los enums son valores compile-time, asi que se usan sin restriccion dentro de
expresiones y bloques `comptime` (ver [[Metaprogramacion]]).

```vesta
// Constante compile-time desde un enum:
comptime i32 MOV_OPCODE = (i32) Op.MOV;   // 0x89

i32 main() {
    comptime {
        // Dentro de un comptime block TODA variable es comptime por contexto:
        // no hace falta anotar 'comptime' en cada una.
        i32 op = (i32) Op.MOV;
        static_assert(op == 0x89, "MOV == 0x89");

        i32 combo = (i32) Op.ADD + (i32) Op.SUB;   // aritmetica con enums
        static_assert(combo == 0x2a, "ADD + SUB == 0x2a");
    }
    return 0;
}
```

`static_assert` valida el estado en compile-time; si falla, la compilacion se
detiene con el mensaje dado.

---

## 5. Resumen

- Dos clases: **ADT** (`enum E { A, B(i32) }`, se consume con `match`) y **con
  valor** (`enum E : tipo { A = v }`, cada variante es una constante del tipo).
- Backing de un enum con valor: **entero, float, string, struct, clase**.
- Una variante con valor se usa **directo, sin cast**, como su tipo base.
- **Auto-incremento** para backing entero.
- **Genericos**: los ADT admiten `<T>` (monomorfizacion).
- **Concepts**: `is_enum<T>()`, `Enum`, `ValuedEnum`, backing concepts, conceptos
  de usuario, y concepto-como-predicado.
- **Comptime**: enums usables en expresiones y bloques `comptime` + `static_assert`.
- Cero runtime: todo se resuelve en compile-time. `interp = JIT = AOT`.

Referencias: [[TiposDatos]] (enum ADT + tipos), [[ControlFlow]] (`match`),
[[Generics]] (genericos + concepts), [[Metaprogramacion]] (comptime).
