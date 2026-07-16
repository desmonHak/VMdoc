# Operadores en Vesta

Referencia completa de operadores aritméticos, lógicos, de comparación, bitwise,
asignación, unarios, postfix y conversiones.

---

## Indice

- [Operadores en Vesta](#operadores-en-vesta)
 - [Indice](#indice)
 - [1. Operadores aritméticos](#1-operadores-aritméticos)
 - [2. Operadores de comparación](#2-operadores-de-comparación)
 - [3. Operadores lógicos](#3-operadores-lógicos)
 - [4. Operadores bitwise](#4-operadores-bitwise)
 - [5. Operadores de desplazamiento](#5-operadores-de-desplazamiento)
 - [6. Asignación y asignación compuesta](#6-asignación-y-asignación-compuesta)
 - [7. Operadores unarios](#7-operadores-unarios)
 - [8. Operadores de punteros](#8-operadores-de-punteros)
 - [9. Postfix: `!!`, `?`](#9-postfix--)
 - [`!!a` — unwrap-or-fail](#a--unwrap-or-fail)
 - [`?` postfix (en `Result<V, E>`) — early-return](#-postfix-en-resultv-e--early-return)
 - [10. Cast C-style: `(T)expr`](#10-cast-c-style-texpr)
 - [11. Precedencia y asociatividad](#11-precedencia-y-asociatividad)
 - [12. Sobrecarga de operadores](#12-sobrecarga-de-operadores)
 - [Tabla completa de métodos](#tabla-completa-de-métodos)
 - [`+=` no es `a = a + b`](#-no-es-a--a--b)
 - [`==` da `!=` gratis; `__bool__` conserva el cortocircuito](#-da--gratis-__bool__-conserva-el-cortocircuito)
 - [Operadores de acceso: `*x`, `x = v`, `x(...)`, `x{...}`](#operadores-de-acceso-x-x--v-x-x-1)
 - [Qué NO se puede sobrecargar](#qué-no-se-puede-sobrecargar)

---

## 1. Operadores aritméticos

| Operador | Operación | Tipos válidos |
| :------: | :--------------- | :----------------- |
| `+` | Suma | int, float, string (concat) |
| `-` | Resta | int, float |
| `*` | Multiplicación | int, float |
| `/` | División | int (trunca), float |
| `%` | Módulo (resto) | int |

```vx
i32 a = 10 + 3; // 13
i32 b = 10 - 3; // 7
i32 c = 10 * 3; // 30
i32 d = 10 / 3; // 3 (división entera)
i32 e = 10 % 3; // 1
f64 f = 10.0 / 3.0; // 3.333...
string s = "hola " + name; // concatenación (StringObject)
```

**División entera**: `/` con operandos `int` trunca hacia cero (no redondea). División
por cero en `int` produce `FATAL_DIV_ZERO` capturable. División en `float` produce
`Inf` o `NaN` según estándar IEEE 754.

**Overflow**: la VM NO detecta overflow signed por defecto. Operaciones que
desbordan wraparound en complemento a dos.

---

## 2. Operadores de comparación

| Operador | Significado |
| :------: | :----------------- |
| `==` | Igual |
| `!=` | Distinto |
| `<` | Menor que |
| `<=` | Menor o igual |
| `>` | Mayor que |
| `>=` | Mayor o igual |

Resultado: siempre `bool`. Operandos: deben ser del mismo tipo (o promovibles).

```vx
if (age >= 18) { ... }
bool eq = (a == b);
bool ne = (s != "default"); // comparación string (byte-a-byte)
```

**Comparación de strings**: `==` y `!=` comparan contenido (no identidad), via opcode
bytecode `strcmp`. Equivalente a `str_equals(a, b)`.

**Comparación de referencias** (clases): `obj == null`, `obj != null`, `obj1 == obj2`
comparan punteros (identidad).

---

## 3. Operadores lógicos

| Operador | Significado | Cortocircuito |
| :------: | :-------------- | :-----------: |
| `&&` | AND lógico | sí |
| `\|\|` | OR lógico | sí |
| `!` | NOT lógico | - |

Operandos: `bool`. Resultado: `bool`.

```vx
if (p != null && p.is_valid()) { ... } // p.is_valid() NO se evalúa si p==null
if (cached || compute_expensive()) { ... } // compute() NO se evalúa si cached==true
bool flag = !done;
```

Ver [[ControlFlow]] sección 7 para detalles del cortocircuito.

---

## 4. Operadores bitwise

| Operador | Operación | Tipos válidos |
| :------: | :--------------- | :--------------- |
| `&` | AND bit a bit | int unsigned/signed |
| `\|` | OR bit a bit | int |
| `^` | XOR bit a bit | int |
| `~` | NOT bit a bit | int (unario) |

```vx
u32 mask = 0xFF00FF00;
u32 r = value & mask; // AND
u32 r = value | 0x80000000; // OR
u32 r = value ^ key; // XOR (encriptación trivial)
u32 r = ~value; // complemento a uno (~0 = 0xFFFFFFFF)
```

A diferencia de `&&`/`||`, estos operadores SIEMPRE evalúan ambos lados (no hay
cortocircuito). Funcionan sobre TODOS los bits del entero.

---

## 5. Operadores de desplazamiento

| Operador | Operación | Tipos válidos |
| :------: | :---------------------- | :------------ |
| `<<` | Desplazamiento izquierda | int |
| `>>` | Desplazamiento derecha (aritmético si signed, lógico si unsigned) | int |

```vx
u32 r = value << 2; // multiplica por 4
i32 r = value >> 4; // divide por 16, preserva signo (sar)
u32 r = value >> 4; // divide por 16, rellena con 0 (shr)
```

El compilador elige `shr` (lógico) o `sar` (aritmético) según el tipo del operando
izquierdo. Cantidad de desplazamiento toma los bits bajos (mod 64).

---

## 6. Asignación y asignación compuesta

| Operador | Equivalente a        |
| :------: | :------------------- |
|   `=`    | (asignación directa) |
|   `+=`   | `x = x + v`          |
|   `-=`   | `x = x - v`          |
|   `*=`   | `x = x * v`          |
|   `/=`   | `x = x / v`          |
|   `%=`   | `x = x % v`          |
|   `&=`   | `x = x & v`          |
|   `\|=`    | `x = x \| v`              |
|   `^=`   | `x = x ^ v`          |
|  `<<=`   | `x = x << v`         |
|  `>>=`   | `x = x >> v`         |

```vx
i32 i = 0;
i += 5; // i = 5
i *= 2; // i = 10
i >>= 1; // i = 5

obj.field += 10; // class/struct field compound
arr[i] *= 2; // array element compound
*p &= 0xFF; // deref compound
c.prop |= flag; // property compound (invoca getter + setter)
```

**Compound assign sobre lvalues complejos**:, los 10 operadores compuestos
funcionan sobre los 4 tipos de lvalue: identifier simple, `obj.field` (struct/class),
`arr[i]`, `*p`. Bit fields y propiedades (getter/setter) heredan automáticamente.

---

## 7. Operadores unarios

| Operador | Operación | Posición |
| :------: | :--------------- | :------- |
| `-` | Negación (cambio de signo) | prefix |
| `+` | Identidad (no-op) | prefix |
| `!` | NOT lógico | prefix |
| `~` | NOT bitwise | prefix |
| `++` | Incremento | prefix/postfix |
| `--` | Decremento | prefix/postfix |

```vx
i32 x = -5;
i32 y = +x; // y = -5
bool b = !cond;
u32 m = ~mask;
i32 i = 0;
i++; // i = 1 (postfix: usa valor previo, luego incrementa)
++i; // i = 2 (prefix: incrementa, luego usa)
```

**Importante con `++`/`--`**: en Vesta, NO se permiten dentro de expresiones complejas
para evitar UB. Usar en statements solos: `i++;` está bien, `a = b++ + ++c;` no se
acepta (ambiguous order of evaluation).

---

## 8. Operadores de punteros

| Operador | Operación |
| :------: | :--------------- |
| `&x` | Dirección de `x` (toma puntero a lvalue) |
| `*p` | Desreferencia (lee/escribe lo apuntado) |
| `p[i]` | Acceso por índice (equivale a `*(p + i)`) |
| `p + n` | Aritmética de punteros (escalado por sizeof) |
| `p - q` | Diferencia de punteros (elementos entre ambos) |

```vx
i32 v = 42;
i32* p = &v; // p apunta a v
*p = 100; // v = 100

i32[10] arr;
i32* pa = arr; // decay: arr -> arr[0]
i32 first = pa[0]; // = *(pa + 0) = arr[0]
i32 third = pa[2]; // = *(pa + 2) = arr[2]
i32* p2 = pa + 2; // p2 apunta a arr[2]
i64 diff = p2 - pa; // diff = 2 (elementos, no bytes)
```

Ver [[TiposDatos]] sección "Punteros" para los tipos `T*` (host pointer),
`VirtualPtr<T>` (VM pointer), y la propagación del bit `is_host_ptr`.

---

## 9. Postfix: `!!`, `?`

### `!!a` — unwrap-or-fail

Sugar para `unwrap(a)` cuando `a` es `Optional<T>` o referencia nullable:

```vx
Optional<i32> opt = Some(42);
i32 v = !!opt; // = 42; falla con FATAL si opt == None

i32? maybe = nullable_thing();
i32 v = !!maybe; // unwrap nullable; lanza FATAL si null
```

### `?` postfix (en `Result<V, E>`) — early-return

Aplica a `Result<V, E>` (NO a `Optional<T>`). Dentro de una función que retorna un
`Result<V, E>`, el postfix `?` sobre una expresión de tipo `Result<V, E>`:

- Si es `Err(e)`, copia el error al buffer de retorno de la función y hace `return`
  inmediato (early-return).
- Si es `Ok(v)`, extrae el valor `v` y la expresión continúa con `v`.

El tipo del error `E` debe coincidir entre la expresión y la función contenedora.

```vx
Result<i64, i64> parse_pos(i64 n) {
    if (n < 0) return Err(99);
    return Ok(n);
}

Result<i64, i64> sum_two_pos(i64 a, i64 b) {
    i64 x = parse_pos(a)?; // si Err, sum_two_pos retorna ese Err; si Ok, x = valor
    i64 y = parse_pos(b)?; // idem
    return Ok(x + y);
}
```

Desugar conceptual de `i64 x = parse_pos(a)?;`:

```vx
Result<i64, i64> __tmp = parse_pos(a);
if (!isOk(__tmp)) {
    return __tmp; // propaga el Err al caller
}
i64 x = value(__tmp); // extrae el Ok
```

Cero overhead runtime frente al patrón `if`-`return` manual. Ver
[[OptionalResult]] para el detalle del operador y de `Result<V, E>`.

---

## 10. Cast C-style: `(T)expr`

```vx
i64 big = 0x123456789ABCDEF0;
i32 small = (i32) big; // trunca a low 32 bits = 0x9ABCDEF0

f64 pi = 3.14159;
i32 trunc = (i32) pi; // = 3 (trunca decimal)

i32 i = -5;
i64 ext = (i64) i; // sign-extend = -5 (i64)

u32 ui = 100;
i64 ext = (i64) ui; // zero-extend = 100

T* p = (T*) raw_ptr_int; // bitcast int -> T*
VirtualPtr<u8> bp = (VirtualPtr<u8>) some_ptr; // PTR<->PTR bitcast
```

**Casos cubiertos**:
- **Numérico int<->int**: el compiler elige TRUNC (narrowing), SEXT (signed widen),
 ZEXT (unsigned widen) o BITCAST (same width).
- **float<->int**: usa BITCAST si se quieren preservar bits IEEE 754, o FTOI/ITOF
 para convertir VALOR. El cast `(i32) 3.14` convierte valor (= 3).
- **PTR<->PTR**: BITCAST, propaga `is_host_ptr` según naturaleza del tipo destino
 (`VirtualPtr<T>` = VM, `T*` = HOST).
- **PTR<->int**: BITCAST.

**Warnings de cast implícito**: el compilador emite warning cuando
asignaciones reducen ancho de entero (narrowing), trunca float a int, o pierde
precisión `int grande -> f32`. Usar cast explícito `(T)` para silenciar.

```vx
i64 big = ...;
i32 small = big; // WARNING: narrowing implícito
i32 small = (i32) big; // OK: cast explícito silencia warning
```

---

## 11. Precedencia y asociatividad

De mayor a menor precedencia (mismo estilo que C):

| Nivel | Operadores | Asociatividad |
| :---: | :--------------------------------------------- | :-----------: |
| 1 | `()` `[]` `.` `->` postfix `++` postfix `--` | izquierda |
| 2 | prefix `++` `--` `+` `-` `!` `~` `&` `*` `(T)` | derecha |
| 3 | `*` `/` `%` | izquierda |
| 4 | `+` `-` | izquierda |
| 5 | `<<` `>>` | izquierda |
| 6 | `<` `<=` `>` `>=` | izquierda |
| 7 | `==` `!=` | izquierda |
| 8 | `&` | izquierda |
| 9 | `^` | izquierda |
| 10 | `\|` | izquierda |
| 11 | `&&` | izquierda |
| 12 | `\|\|` | izquierda |
| 13 | `?:` (ternario) | derecha |
| 14 | `=` `+=` `-=` `*=` etc. | derecha |

```vx
// 3 + 4 * 5 = 23 (no 35) porque * tiene mayor precedencia
// (3 + 4) * 5 = 35 explícito con parens
// a & b == c parses as a & (b == c) porque == > &
// (a & b) == c explícito si es lo que se quiere
```

**Regla práctica**: cuando hay duda, usar parens explícitos. El compilador no
emite warnings por precedencia ambigua (TODO: futuro `-Wparentheses`).

---

## 12. Sobrecarga de operadores

Un `struct` o una `class` puede dar significado propio a los operadores
declarando métodos con nombres reservados (los *dunder*, por *double
underscore*). El compilador reescribe el operador a una llamada a ese método:

```vx
struct Vec {
    i64 x;
    i64 y;

    public Vec __add__(Vec o) {
        Vec r;
        r.x = this.x + o.x;
        r.y = this.y + o.y;
        return r;
    }
}

Vec a; a.x = 1; a.y = 2;
Vec b; b.x = 3; b.y = 4;
Vec c = a + b;          // a.__add__(b)  ->  (4, 6)
```

No hay palabra clave `operator` ni sintaxis especial: son métodos normales.
Se declaran, se heredan y se llaman como cualquier otro, y el que no exista
simplemente significa que ese operador no está disponible para ese tipo. El
receptor puede ser un `struct` (despacho estático) o una `class` (despacho
virtual); en ambos casos el compilador los inlinea igual que a cualquier otro
método, así que la sobrecarga no cuesta una llamada.

### Tabla completa de métodos

| Categoría | Expresión | Método | Firma |
| :--- | :--- | :--- | :--- |
| Aritméticos | `a + b` | `__add__` | `(V) -> R` |
| | `a - b` | `__sub__` | `(V) -> R` |
| | `a * b` | `__mul__` | `(V) -> R` |
| | `a / b` | `__div__` | `(V) -> R` |
| | `a % b` | `__mod__` | `(V) -> R` |
| Comparación | `a == b` | `__eq__` | `(V) -> bool` |
| | `a != b` | `__ne__` | `(V) -> bool` |
| | `a < b` | `__lt__` | `(V) -> bool` |
| | `a > b` | `__gt__` | `(V) -> bool` |
| | `a <= b` | `__le__` | `(V) -> bool` |
| | `a >= b` | `__ge__` | `(V) -> bool` |
| Bitwise | `a & b` | `__and__` | `(V) -> R` |
| | `a \| b` | `__or__` | `(V) -> R` |
| | `a ^ b` | `__xor__` | `(V) -> R` |
| | `a << b` | `__lshift__` | `(V) -> R` |
| | `a >> b` | `__rshift__` | `(V) -> R` |
| Compuestos | `a += b` | `__iadd__` | `(V) -> R` |
| | `a -= b` | `__isub__` | `(V) -> R` |
| | `a *= b` | `__imul__` | `(V) -> R` |
| | `a /= b` | `__idiv__` | `(V) -> R` |
| | `a %= b` | `__imod__` | `(V) -> R` |
| | `a &= b` | `__iand__` | `(V) -> R` |
| | `a \|= b` | `__ior__` | `(V) -> R` |
| | `a ^= b` | `__ixor__` | `(V) -> R` |
| | `a <<= b` | `__ilshift__` | `(V) -> R` |
| | `a >>= b` | `__irshift__` | `(V) -> R` |
| Unarios | `-a` | `__neg__` | `() -> R` |
| | `~a` | `__invert__` | `() -> R` |
| | `!a`, `a && b`, `if (a)` | `__bool__` | `() -> bool` |
| Acceso | `a[i]` | `__index__` | `(I) -> R` |
| | `*a` | `__deref__` | `() -> R` |
| | `a = v` | `__assign__` | `(V) -> R` |
| | `a(x, y)` | `__call__` | `(...) -> R` |
| | `a{x, y}` | `__braces__` | `(...) -> R` |
| Copia | copia implícita | `__clone__` | ver [[SmartPointers]] |

### `+=` no es `a = a + b`

`a += b` tiene su propio método (`__iadd__`), distinto del binario. La
diferencia no es de estilo: leer, sumar y escribir son tres pasos, y para un
tipo que promete atomicidad otro hilo cabe en medio.

Ahora bien, **no hace falta declararlo**. Un tipo que solo define `__add__`
obtiene `+=` y `++` gratis: el compilador desazucara `a += b` a `a = a + b`. El
método in-place solo lo paga quien lo necesita:

```vx
struct Contador {
    i64 n;
    public Contador __add__(i64 k) { Contador r; r.n = this.n + k; return r; }
}

Contador c; c.n = 0;
c += 5;     // sin __iadd__: se desazucara a c = c + 5
c++;        // sin __iadd__: se desazucara a c = c + 1
```

El desazucarado exige que el binario devuelva algo asignable al propio tipo
(`Contador.__add__ -> Contador`). Un `__add__` que devuelve otra cosa no define
un `+=`, y el error de tipos lo dice.

`++a` y `a++` usan la misma resolución con `1`. Como en C, el prefijo evalúa al
valor ya modificado y el sufijo al anterior.

### `==` da `!=` gratis; `__bool__` conserva el cortocircuito

Declarar `__eq__` basta para tener `!=`: si no hay `__ne__` propio, el
compilador niega el `bool` que devuelve `__eq__`. Declarar `__ne__` solo tiene
sentido si la negación no es la respuesta correcta.

`__bool__` es el que da la verdad de un valor. Lo usan `!a`, `a && b`, `a || b`
y las condiciones de `if` / `while` / `for`, con **el cortocircuito intacto**:
en `a && b`, `b.__bool__()` solo se llama si `a` fue verdad.

```vx
struct Opcion {
    i64 v;
    bool presente;
    public bool __bool__() { return this.presente; }
}

Opcion o; o.v = 7; o.presente = true;
if (o) { println("hay valor: ${o.v}"); }    // o.__bool__()
if (!o) { println("vacio"); }               // !o.__bool__()
```

### Operadores de acceso: `*x`, `x = v`, `x(...)`, `x{...}`

Los cuatro son independientes y cada uno con su método.

**`*x` (`__deref__`)** permite escribir punteros inteligentes en el propio
lenguaje: el tipo decide qué significa leer a través de él.

**`x = v` (`__assign__`)** hace que la escritura la haga el tipo, en lugar de la
copia campo a campo por defecto. Es lo que permite que un tipo atómico prometa
un store indivisible. Un tipo que no lo declara conserva la copia de siempre.

Ojo: `Caja c = 5;` es una **declaración** (construcción), no una asignación —
no pasa por `__assign__`.

**`x(a, b)` (`__call__`) y `x{a, b, c}` (`__braces__`)** son dos operadores de
invocación **distintos**. Un tipo puede declarar uno, el otro o los dos, y
darles el significado que quiera:

```vx
struct Caja {
    i64 v;
    public i64 __deref__()        { return this.v; }
    public i64 __assign__(i64 n)  { this.v = n * 2; return this.v; }
}

Caja c;
c.v = 5;
i64 leido = *c;     // 5
c = 10;             // v = 20  (no es una copia de Caja)
i64 tras = *c;      // 20
```

`a{...}` es el postfijo sobre un valor ya construido, y no tiene nada que ver
con `Punto p = {2, 3}` (la lista de inicialización de los campos, que sigue
siendo válida para cualquier struct). Si el tipo no declara `__braces__`, la
sintaxis `a{...}` no existe para él y el compilador la rechaza — igual que con
los demás operadores.

La única expresión del lenguaje que no lleva paréntesis y va seguida de `{` es
el *scrutinee* de un `match`. Ahí el `{` abre el cuerpo del match; para usar el
operador hay que parentizar: `match (a{1, 2}) { ... }`.

Ejemplos completos: `315_operadores_llamada_deref.vx` y
`314_operadores_sobrecargados.vx`.

### Qué NO se puede sobrecargar

`&a` (dirección de), `.` (acceso a campo), `->`, `?:`, `?`, `!!`, los casts
`(T)expr` y la precedencia. La precedencia y la asociatividad son siempre las
de la tabla del apartado 11: sobrecargar `+` no cambia que se evalúe antes que
`<<`.

Tampoco se sobrecargan los operadores sobre tipos primitivos: los métodos viven
en un `struct` o una `class`, así que `i64 + i64` es siempre la suma del
lenguaje. Si se quiere otra semántica, el camino es un tipo propio (ver
[[TiposDatos]], sección de newtypes).

---

Ver también: [[TiposDatos]] (tipos primitivos y promociones), [[ControlFlow]]
(uso de operadores en condiciones), [[Strings]] (interpolación y format specifiers),
[[OptionalResult]] (operadores `!!`/`?`), [[OOP]] (métodos, herencia y despacho),
[[Sincronizacion]] (tipos atómicos y monitores).
