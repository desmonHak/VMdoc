# Metaprogramacion e introspeccion

Vesta tiene un sistema de metaprogramacion compile-time potente que combina:

- **Macros** (`@Macro`) que ejecutan codigo arbitrario en tiempo de compilacion
 y emiten codigo Vesta que se inyecta en el call site.
- **Captura raw** de expresiones (`expr` param) para crear DSLs embebidos.
- **Introspeccion** de tipos con cero overhead runtime (sizeof, alignof,
 typename, kind, field_count, etc.).
- **FFI compile-time** que invoca DLLs del sistema u funciones in-process
 durante la compilacion -- los resultados se "fosilizan" como literales en
 el binario.
- **Estructuras de datos** del lenguaje (arrays, structs, strings) usables
 dentro del cuerpo de un macro.

Todo en compile-time, sin runtime cost: cuando el `.velb` carga, todos los
macros ya estan resueltos. El binario solo contiene el codigo generado.

---

## Indice

1. [@Macro basicos](#1-macro-basicos)
2. [Capturando codigo arbitrario con `expr`](#2-capturando-codigo-arbitrario-con-expr)
3. [Builtins comptime y azucar sintactico](#3-builtins-comptime-y-azucar-sintactico)
4. [Introspeccion de tipos](#4-introspeccion-de-tipos)
5. [Estructuras de datos en compile-time](#5-estructuras-de-datos-en-compile-time)
6. [FFI en tiempo de compilacion](#6-ffi-en-tiempo-de-compilacion)
7. [`static_assert` y el virtual lib `vesta_comptime`](#7-static_assert-y-el-virtual-lib-vesta_comptime)
8. [Patrones comunes](#8-patrones-comunes)
9. [Limitaciones y workarounds](#9-limitaciones-y-workarounds)

---

## `const` y `comptime`: valores compile-time

En Vesta **toda constante es comptime por definicion**. Un `const` cuyo
inicializador es evaluable en compile-time entra en el "mundo comptime":
lo pueden **leer y consumir** los macros, los builtins comptime y la
introspeccion, exactamente igual que un valor `comptime`.

```vx
const i64 N = 5;
const string E = comptime_concat("(", comptime_concat(comptime_to_str(N), ") * 8"));
const i64 R = comptime_compile(E);          // 40, calculado al compilar
static_assert(R == 40, "5 * 8");
```

| Forma | Para que sirve |
|---|---|
| `const X = expr;` | un **valor** (runtime-inmutable, mutable durante comptime). |
| `comptime X = expr;` | idem -- un valor comptime; equivalente a `const` para globales. |
| `comptime T fn(...) { ... }` | una **funcion** comptime. |
| `comptime { ... }` | un **bloque** imperativo comptime. |

**Mutabilidad: solo en compile-time.** Un global `const` o `comptime` es
**mutable mientras compila** (el codigo comptime lo puede alterar: id
counters, tablas acumuladas, maquinas de estado) e **inmutable en runtime**
(reasignarlo desde codigo runtime es error; se congela a su valor final).
Da igual con cual de las dos palabras lo declares: funciona igual.

```vx
const i64 g_id = 0;                 // runtime-inmutable
@Macro comptime string fresh() {
    g_id = g_id + 1;               // mutado EN comptime -> OK
    return to_str(g_id);
}
// en runtime `g_id = 5;` seria un error de compilacion.
```

Regla practica: **usa `const`/`comptime` para valores; `comptime fn` /
`comptime { }` para logica meta** (loops, generacion de codigo).

> Ya no existen `comptime const` ni `comptime var`: eran redundantes
> (`comptime`/`const` ya implican "constante en runtime, mutable en
> comptime").  Para inferencia de tipo usa `auto` (valido en comptime y en
> runtime; distinto de `var`, que se elimino).

**Dentro de un `comptime fn` o `comptime { }`** los locales se declaran
tipo-primero, sin prefijo -- el contexto ya es comptime y son mutables por
defecto:

```vx
comptime i64 factorial(i64 n) {
    i64 p = 1;              // local pelado: mutable y comptime
    i64 i = 1;
    while (i <= n) { p = p * i; i = i + 1; }
    return p;
}
const i64 F6 = factorial(6);   // 720, horneado en el binario
```

**Eliminacion del const muerto**: si una constante se usa **solo** en
comptime (por ejemplo dentro de `static_assert` o como argumento de un
builtin comptime), el optimizador la elimina por completo -- no deja ningun
slot en el binario. Una constante usada en runtime se inlina (escalares) o
se emite en la seccion de datos (strings).

---

## 1. @Macro basicos

Un `@Macro` es una funcion comptime que devuelve un `string` con codigo Vesta
valido. El compilador re-parsea esa string en el call site y la inyecta como
si el usuario la hubiera escrito a mano.

```vx
@Macro
comptime string double_it(i64 n) {
    return to_str(n * 2);
}

i32 main() {
    i32 r = double_it(21); // se compila como `i32 r = 42`
    return r;
}
```

**Anatomia**:

- `@Macro` marca la funcion como macro.
- `comptime` indica que el body se ejecuta en compile-time.
- El tipo de retorno es siempre `string` (el codigo emitido).
- Los parametros aceptan tipos primitivos (`i64`, `string`, `bool`, etc.).

El cuerpo es codigo Vesta normal con ciertas restricciones (ver Limitaciones).
Dentro del cuerpo los locales se declaran tipo-primero (`i64 x = 0;`) y son
mutables por defecto -- el body es integramente comptime, no hace falta ningun
prefijo para marcarlos.

Marca tu macro con `@Pure` cuando el resultado dependa SOLO de los args
(no de globales mutables ni FFI no-determinista):

```vx
@Pure @Macro
comptime string lookup_table(i64 idx) {
    i64 vals[8] = {10, 20, 30, 40, 50, 60, 70, 80};
    return to_str(vals[idx]);
}
```

`@Pure` activa memoization: llamadas repetidas con los mismos args se
sirven del cache sin re-ejecutar el body. Util para macros pesados.

---

## 2. Capturando codigo arbitrario con `expr`

El tipo especial `expr` (solo valido como parametro de `@Macro`) captura
el texto **raw** del call site -- sin parsearlo como expresion Vesta. Permite
crear DSLs embebidos:

```vx
@Macro
comptime string walk(expr code) {
    // `code` recibe la cadena verbatim del call site, p.ej.
    // "root -> 0x100 -> 0x10"
    return parse_and_emit(code);
}

i32 main() {
    u64 v = walk(root -> 0x100 -> 0x10); // sintaxis arbitraria valida
    return 42;
}
```

**Reglas de captura**:

- Captura hasta la siguiente coma a depth 0 o el `)` que cierra la llamada.
- Tracking de paren/bracket/brace: subexpresiones con comas internas
 funcionan (`f(walk(a, b), c)` captura `a, b` como una sola expr).
- El macro debe estar declarado **antes** de su uso en el archivo.
- Trim automatico de whitespace al final del span capturado.

**Ejemplo real**: macro `walk` que parsea `base -> off1 -> off2 -> ...` y
emite pointer chase anidado:

```vx
@Macro
comptime string walk(expr code) {
    i64 n = strlen(code);
    i64 i = 0;
    i64 last_start = 0;
    string current = "";
    i64 token_count = 0;
    while (i < n) {
        // detectar "->"
        if (i + 1 < n && substr(code, i, 1) == "-" && substr(code, i + 1, 1) == ">") {
            // extraer token previo y trim
            i64 ts = last_start;
            i64 te = i;
            while (ts < te && substr(code, ts, 1) == " ") ts = ts + 1;
            while (te > ts && substr(code, te - 1, 1) == " ") te = te - 1;
            string tok = substr(code, ts, te - ts);
            if (token_count == 0) {
                current = "( " + tok + " )";
            } else {
                current = "( *(u64*)((u64)" + current + " + " + tok + ") )";
            }
            token_count = token_count + 1;
            last_start = i + 2;
            i = i + 2;
        } else {
            i = i + 1;
        }
    }
    // ultimo token
    i64 ts = last_start;
    i64 te = n;
    while (ts < te && substr(code, ts, 1) == " ") ts = ts + 1;
    while (te > ts && substr(code, te - 1, 1) == " ") te = te - 1;
    string last_tok = substr(code, ts, te - ts);
    if (token_count == 0) return "( " + last_tok + " )";
    return "*(u64*)((u64)" + current + " + " + last_tok + ")";
}

// Uso:
u64 v = walk(root -> 0x100 -> 0 -> 0);
// Se compila a:
// u64 v = *(u64*)((u64)( *(u64*)((u64)( *(u64*)((u64)( root ) + 0x100) ) + 0) ) + 0);
```

El `.vel` emitido tiene exactamente tres instrucciones `movh` (host memory
load) consecutivas -- una por hop. Cero overhead vs escribirlo a mano.

**Equivalente en C** (lo que el macro evita escribir):
```c
uint64_t v = *(uint64_t*)((char*)*(uint64_t**)((char*)root + 0x100) + 0);
```

### Llamar a otras funciones desde el cuerpo

Un `@Macro` o una `comptime fn` puede llamar, dentro de su cuerpo, a **otras
funciones** -- comptime o runtime:

```vesta
i32 rt_add(i32 x, i32 y) { return x + y; }        // funcion runtime
comptime i32 ct_double(i32 n) { return n * 2; }    // funcion comptime

// comptime fn que llama a otra comptime fn:
comptime i32 chain(i32 n) { return ct_double(n) + 1; }

// comptime fn que llama a una funcion runtime (se pliega en compile-time):
comptime i32 uses_rt(i32 n) { return rt_add(n, 2); }
const i32 K = uses_rt(40);      // 42, calculado al compilar

// @Macro que GENERA codigo que llama a una funcion runtime en el call site:
@Macro comptime string gen(u32 n) {
    return "rt_add(40, " + to_str(n) + ")";
}
i32 r = gen(2);                 // inyecta: rt_add(40, 2) -> 42
```

**Forwarding de `expr` anidado.**  Un macro/comptime fn con un parametro `expr`
puede pasar ese parametro a otra fn `expr`-capture; el texto capturado se
propaga (no se re-captura el identificador):

```vesta
comptime string source(expr code) { return code; }

// comptime fn (VALOR):
comptime string outer(expr code) { return source(code); }
string s = outer(a + b);        // "a + b"

// @Macro (inyecta CODIGO):
@Macro comptime string twice(expr e) {
    return "(" + source(e) + ") + (" + source(e) + ")";
}
i32 r2 = twice(a + b);          // inyecta:  (a + b) + (a + b)
```

### `source(...)` como quasi-quote: huecos `${...}` e interpolacion runtime `\${...}`

`source(...)` no solo captura el texto de su argumento: tambien es un
**quasi-quote**.  Escribes una **plantilla de codigo** (que el IDE resalta como
codigo real, no como un string opaco) y rellenas los huecos con `${expr}`:

```vesta
comptime string bf_gen(string src, u32 size_stack) {
    return source(
        () => {
            i32[${size_stack}] tape;        // hueco COMPTIME: se evalua al compilar
            i32 p = 0;
            ${bf_compile_body(src)}         // hueco COMPTIME: splice del cuerpo
            return tape[p];
        }
    );
}
```

Hay **dos** tipos de hueco, y la distincion es importante:

| Sintaxis   | Cuando se evalua | Que produce |
| :--------- | :--------------- | :---------- |
| `${expr}`  | **compile-time** (en la ComptimeVM) | el resultado se **splicea** en el texto generado |
| `\${expr}` | **runtime** (en el codigo generado) | queda `${expr}` **literal** en la salida -> interpolacion del programa final |

`${...}` es un hueco que el compilador rellena AHORA.  `\${...}` **atraviesa**
hasta el codigo generado tal cual, para que la interpolacion normal de strings
del programa lo resuelva en ejecucion.  Ejemplo (un compilador que genera un
`print` de una celda como caracter, en runtime):

```vesta
comptime string emit_out() {
    // El `\${...}` NO se evalua al compilar; el lambda generado hara
    // `print("${tape[p]:char}")` y lo interpolara en RUNTIME.
    return source( print("\${tape[p]:char}"); );
}
```

Esta interpolacion es una propiedad de **todo el lenguaje**: el `${...}` que
sobrevive al codigo generado se baja por el mismo mecanismo que cualquier
`"...${x}..."` (STRMAKE + STRCAT) y funciona identico en **interprete, JIT y
AOT**.

Regla mnemotecnica: si el valor lo conoce el **compilador**, usa `${...}`; si lo
conoce solo el **programa en ejecucion**, usa `\${...}`.

---

## 3. Builtins comptime y azucar sintactico

El AST evaluator de macros soporta operadores nativos y builtins cortos.
**Siempre usa la version corta cuando este disponible**:

| Verbose | Corto / azucar |
|---|---|
| `comptime_concat(a, b)` | `a + b` |
| `comptime_streq(a, b)` | `a == b` |
| `comptime_streq(a, b) == false` | `a != b` |
| `comptime_strlen(s)` | `strlen(s)` |
| `comptime_substr(s, a, b)` | `substr(s, a, b)` |
| `comptime_to_str(n)` | `to_str(n)` |
| `comptime_chr(cp)` | `chr(cp)` |
| `comptime_ord(s)` | `ord(s)` |
| `comptime_repeat(s, n)` | `repeat(s, n)` |
| `comptime_replace(s, from, to)` | `replace(s, from, to)` |
| `comptime_contains(haystack, needle)` | `contains(haystack, needle)` |
| `gensym()` | `gensym()` (genera identificador unico) |

**Ejemplo refactorizado** (antes vs ahora):

```vx
// Verbose (legacy):
return comptime_concat(
"( *(u64*)((u64)",
comptime_concat(current,
comptime_concat(" + ",
comptime_concat(tok, ") )"))));

// Compacto (preferido):
return "( *(u64*)((u64)" + current + " + " + tok + ") )";
```

Los operadores `+`/`==`/`!=` sobre strings funcionan tanto en el AST
evaluator del macro como en codigo runtime. Mismo bytecode emitido.

---

## 4. Introspeccion de tipos

Vesta ofrece un conjunto completo de builtins de introspeccion
**zero-overhead**: cada uno recibe uno o dos type-args entre `<...>` y
se resuelve en compile-time a un solo literal en el `.velb` (un `CONST`
IR op para enteros/bool, un `STRMAKE` para strings). No hay runtime
dispatch: cuando el programa corre, la respuesta ya esta horneada.

**Sintaxis general**: `builtin<T>()`, `builtin<T>(arg)` o
`builtin<A, B>()`. Todos se invocan con parentesis. Los argumentos de
runtime (cuando existen) deben ser **literales compile-time**: un string
literal no interpolado para nombres de campo/metodo, o un literal entero
para indices.

### 4.1 Tamanos y layout

| Builtin | Firma | Devuelve | Notas |
|---|---|---|---|
| `sizeof<T>()` | `<T>()` | `u64` | Bytes del tipo. Para `CLASS` devuelve 8 (es un puntero al objeto, no el tamano de la instancia). |
| `alignof<T>()` | `<T>()` | `u64` | Alineamiento en bytes. |
| `offsetof<T>("campo")` | `<T>(string_lit)` | `u64` | Offset del campo dentro del struct/clase. |

```vx
u64 sz_i64 = sizeof<i64>();        // 8
u64 sz_punto = sizeof<Punto>();    // suma de fields alineada
u64 al_f64 = alignof<f64>();       // 8
u64 off_y = offsetof<Punto>("y");  // p.ej. 8
```

### 4.2 Identidad del tipo

| Builtin | Firma | Devuelve | Notas |
|---|---|---|---|
| `typename<T>()` | `<T>()` | `string` | Nombre canonico sin espacios: `"i32"`, `"Punto"`, `"Box<i32>"`, `"i32*"`, `"i32[8]"`. |
| `type_id<T>()` | `<T>()` | `u32` | Hash FNV-1a de 32 bits del nombre canonico. Estable cross-build; permite "es el tipo X?" con 1 comparacion entera. |
| `kind<T>()` | `<T>()` | `i32` | Categoria del tipo (ver tabla de codigos). |

**Codigos de `kind<T>()`** (valores estables; nuevos kinds se agregan al
final):

| Codigo | Categoria | Codigo | Categoria |
|---:|---|---:|---|
| 0 | `Primitive` | 8 | `Function` |
| 1 | `Class` | 9 | `String` |
| 2 | `Struct` | 10 | `Borrow` |
| 3 | `Enum` | 11 | `Future` |
| 4 | `Optional` | 12 | `Unique` |
| 5 | `Result` | 13 | `Shared` |
| 6 | `Array` | 14 | `Collection` |
| 7 | `Ptr` | 99 | `Unknown` |

```vx
string name = typename<Box<i32>>();  // "Box<i32>"
u32 id = type_id<i32>();             // hash FNV-1a estable
i32 k = kind<Punto>();               // 2 (Struct)
```

### 4.3 Campos

| Builtin | Firma | Devuelve | Notas |
|---|---|---|---|
| `field_count<T>()` | `<T>()` | `u32` | Numero de campos (incluye heredados en clases). |
| `has_field<T>("f")` | `<T>(string_lit)` | `bool` | Cierto si el campo existe. |
| `field_name<T>(idx)` | `<T>(int_lit)` | `string` | Nombre del campo idx-esimo (orden de declaracion); vacio si fuera de rango. |
| `field_type<T>("f")` | `<T>(string_lit)` | `string` | Nombre del tipo del campo. |
| `field_type_at<T>(idx)` | `<T>(int_lit)` | `Type` | Tipo del campo idx-esimo como valor de tipo (ver 4.7). |

```vx
u32 nf = field_count<Punto>();       // 2
bool hf = has_field<Punto>("z");     // false
string fn = field_name<Punto>(0);    // "x"
string ft = field_type<Punto>("x");  // "i64"
```

### 4.4 Metodos

| Builtin | Firma | Devuelve | Notas |
|---|---|---|---|
| `method_count<T>()` | `<T>()` | `u32` | Numero de metodos (incluye heredados). |
| `has_method<T>("m")` | `<T>(string_lit)` | `bool` | Cierto si el metodo existe. |
| `method_name<T>(idx)` | `<T>(int_lit)` | `string` | Nombre del metodo idx-esimo; vacio si fuera de rango. |
| `method_return_type<T>(idx)` | `<T>(int_lit)` | `Type` | Tipo de retorno del metodo idx-esimo como valor de tipo (ver 4.7). |

```vx
u32 nm = method_count<Animal>();       // p.ej. 3
bool hm = has_method<Animal>("speak"); // true
string m0 = method_name<Animal>(0);    // "speak"
```

### 4.5 Predicados de categoria

Todos devuelven `bool` y toman un unico type-arg.

| Builtin | Cierto cuando `T` es... |
|---|---|
| `is_class<T>()` | una clase (reference type). |
| `is_struct<T>()` | un struct value-type. |
| `is_primitive<T>()` | un primitivo (`i8..u64`, `f32`, `f64`, `bool`, `char`, `void`). |
| `is_integer<T>()` | un entero con signo o sin signo. |
| `is_signed<T>()` | un entero con signo. |
| `is_unsigned<T>()` | un entero sin signo. |
| `is_float<T>()` | `f32` o `f64`. |
| `is_numeric<T>()` | entero o float. |
| `is_bool<T>()` | `bool`. |
| `is_char<T>()` | `char`. |
| `is_pointer<T>()` | un puntero raw `T*`. |
| `is_string<T>()` | `string`. |

```vx
bool ic = is_class<Animal>();     // true
bool iu = is_unsigned<u32>();     // true
bool inum = is_numeric<f64>();    // true
bool ip = is_pointer<i64*>();     // true
```

### 4.6 Newtypes y relaciones entre tipos

| Builtin | Firma | Devuelve | Notas |
|---|---|---|---|
| `is_newtype<T>()` | `<T>()` | `bool` | Cierto si `T` es un `typedef ... name new`. |
| `is_opaque<T>()` | `<T>()` | `bool` | Cierto si es un newtype opaco. |
| `underlying_of<T>()` | `<T>()` | `string` | Nombre del tipo subyacente de un newtype (`"u64"`, ...). |
| `is_subtype<D, B>()` | `<D, B>()` | `bool` | Cierto si `D` deriva de `B` (herencia). |
| `is_same<A, B>()` | `<A, B>()` | `bool` | Identidad nominal. `is_same<user_id, group_id>` es `false` aunque ambos sean `u64`. |

```vx
bool nt = is_newtype<user_id>();       // true (typedef new)
bool op = is_opaque<session_id>();     // true (@opaque)
string un = underlying_of<user_id>();  // "u64"

bool sub = is_subtype<Perro, Animal>();  // true
bool same = is_same<i32, i32>();         // true
```

### 4.7 Tipos como valores (Type-as-first-class-value)

Estos builtins devuelven un **valor de tipo** que se guarda en un
`const Type X = ...` y luego se reutiliza en cualquier posicion de tipo
(por ejemplo dentro de `sizeof<X>()` o `typename<X>()`).

| Builtin | Firma | Devuelve | Notas |
|---|---|---|---|
| `comptime_type<T>()` | `<T>()` | `Type` | Materializa `T` como valor reutilizable. |
| `parent_class<T>()` | `<T>()` | `Type` | Superclase de `T`; tipo meta vacio si no hay super o `T` no es clase. |
| `element_type<T>()` | `<T>()` | `Type` | Tipo interior de wrappers: `T*`->`T`, `T[N]`->`T`, `Optional<T>`->`T`, `Future<T>`->`T`, `borrow<T>`/`unique<T>`/`shared<T>`->`T`. |
| `error_type<T>()` | `<T>()` | `Type` | Para `Result<V, E>` devuelve `E`; tipo meta vacio si no es `Result`. |

```vx
const Type Base = parent_class<Perro>();  // Animal
string base_name = typename<Base>();      // "Animal"

const Type Elem = element_type<i32*>();   // i32
u64 elem_sz = sizeof<Elem>();             // 4
```

### 4.8 Iteracion y acceso directo a campos

```vx
// Loop desenrollado en compile-time: el callback se invoca UNA vez
// por cada campo/metodo, sin bucle runtime.
for_each_field<Punto>((string name) => { /* ... */ });
for_each_method<Animal>((string name) => { /* ... */ });

// Acceso directo por nombre de campo cuando el tipo es conocido en
// compile-time.  Bypass del path getfield/setfield virtual.
i32 val = field_get<Punto>(p, "x");  // baja a LOAD directo al offset
field_set<Punto>(p, "y", 100);       // baja a STORE directo al offset
```

`field_get<T>(obj, "f")` devuelve el tipo del propio campo;
`field_set<T>(obj, "f", value)` devuelve `void` y exige que `value` sea
asignable al tipo del campo.

### 4.9 Depuracion de metaprogramas

`comptime_print(val)` imprime un valor a la salida de error **durante la
compilacion** (no en runtime). Acepta enteros, strings, valores de tipo,
arrays o structs. Devuelve `0` (`u64`) para poder componerlo dentro de
`static_assert`:

```vx
static_assert(comptime_print(sizeof<Punto>()) == 0, "solo para debug");
```

**Verificacion empirica**: cada builtin escalar baja a un solo `CONST`
IR op (`const.u64 8`, `const.u32 ...`); los que devuelven `string` bajan
a un `STRMAKE`. Sin runtime dispatch en ningun caso.

### Bloque `comptime if` y `comptime for`

```vx
// Dead-branch elimination en compile-time.
comptime if (sizeof<u64>() == 8) {
    // bytecode emitido solo para esta rama
}

// Loop unrolled completamente.
comptime for (i in 0..N) {
    // N iteraciones expandidas inline; `i` es constante comptime en cada copia
}
```

### Iteracion sobre fields/methods

```vx
@Macro
comptime string dump_fields() {
    for_each_field<Punto>((string name) => {
        // body invocado UNA vez por campo en compile-time
});
    return "...";
}
```

`for_each_field<T>(cb)` y `for_each_method<T>(cb)` invocan el callback N
veces en compile-time, una por cada field/method. Sin loop runtime.

### `field_get<T>` y `field_set<T>` directos

```vx
i32 val = field_get<Punto>(p, "x"); // baja a LOAD directo al offset
field_set<Punto>(p, "y", 100); // baja a STORE directo al offset
```

Bypass del path getfield/setfield virtual. Util cuando el tipo es conocido
en compile-time.

---

## 5. Estructuras de datos en compile-time

| Estructura | Compile-time | Runtime |
|---|:---:|:---:|
| `T[N]` arrays nativos | si | si |
| Arrays de strings | si | si |
| Arrays de structs | si | si |
| `struct` con cualquier campo | si | si |
| Structs anidados | si | si |
| Bit fields | si | si |
| `string` (con concat, substr, etc.) | si | si |
| `ArrayList<T>` | no (plugin runtime) | si |
| `HashMap<K,V>` | no | si |
| `HashSet/Queue/Deque/TreeMap` | no | si |

### Arrays

```vx
@Macro
comptime string fib_at(i64 idx) {
    i64 fibs[16] = {0, 1, 1, 2, 3, 5, 8, 13, 21, 34, 55, 89, 144, 233, 377, 610};
    if (idx < 0 || idx >= 16) return "0";
    return to_str(fibs[idx]);
}

i64 f10 = fib_at(10); // se reemplaza por el literal 55
```

### Structs anidados

```vx
struct Punto { i64 x; i64 y; }

@Macro
comptime string mid_point() {
    Punto a = {.x = 10, .y = 20};
    Punto b = {.x = 30, .y = 40};
    Punto mid = {.x = (a.x + b.x) / 2, .y = (a.y + b.y) / 2};
    return "(" + to_str(mid.x) + "," + to_str(mid.y) + ")";
}
```

### Diccionario en compile-time (via arrays paralelos)

Como no hay `HashMap` comptime, se modela con dos arrays sincronizados:

```vx
@Macro
comptime string lookup(i64 key) {
    i64 keys[5] = {1, 2, 5, 10, 42};
    i64 vals[5] = {100, 200, 555, 1000, 9999};
    i64 i = 0;
    while (i < 5) {
        if (keys[i] == key) return to_str(vals[i]);
        i = i + 1;
    }
    return "-1"; // not found
}

i64 v = lookup(5); // se compila como `i64 v = 555;`
```

Busqueda lineal O(N) pero solo en compile-time. Para N < 100 entries la
compilacion sigue siendo instantanea.

### Generacion de codigo iterativo

```vx
@Macro
comptime string sum_squares(i64 n) {
    string result = "0";
    i64 i = 1;
    while (i <= n) {
        result = result + " + " + to_str(i * i);
        i = i + 1;
    }
    return result;
}

i64 s = sum_squares(4);
// Se compila como `i64 s = 0 + 1 + 4 + 9 + 16;`
// El constant folder del IR optimizer lo pliega a 30.
```

### Tabla 2D (matriz como array 1D)

```vx
@Macro
comptime string trace_3x3() {
    i64 mat[9] = {1, 2, 3, 4, 5, 6, 7, 8, 9};
    return to_str(mat[0] + mat[4] + mat[8]); // diagonal -> "15"
}
```

### Switch-via-ternario generado

```vx
@Macro
comptime string switch_gen() {
    i64 cases[3] = {1, 2, 3};
    i64 results[3] = {10, 20, 30};
    string code = "0";
    i64 i = 2;
    while (i >= 0) {
        code = "((x == " + to_str(cases[i]) + ") ? " + to_str(results[i]) + " : " + code + ")";
        i = i - 1;
    }
    return code;
}

i64 sw = switch_gen();
// Se compila como ternario anidado:
// ((x == 1) ? 10 : ((x == 2) ? 20 : ((x == 3) ? 30 : 0)))
```

---

## 6. FFI en tiempo de compilacion

Cualquier funcion declarada como `extern` puede invocarse en compile-time
desde un `@Macro`. El AST evaluator hace `LoadLibraryA + GetProcAddress`
durante la compilacion e invoca la funcion en el proceso del compilador.
El resultado se "fosiliza" como literal en el `.velb`:

```vx
extern "kernel32.dll" {
    fn GetCurrentProcessId() -> u32;
    fn GetTickCount() -> u32;
}

// Huella unica por compilacion.
@Macro
comptime string build_id() {
    u64 pid = GetCurrentProcessId(); // FFI en compile-time
    u64 tick = GetTickCount(); // FFI en compile-time
    u64 mix = (pid * 31) ^ (tick * 17);
    return to_str(mix);
}

i32 main() {
    u64 fingerprint = build_id();
    // El binario final NO llama a kernel32 en runtime.
    // Solo contiene `mov rN, <literal>` con el valor calculado al compilar.
    return 42;
}
```

**Diferencias clave**:

- **FFI runtime**: el `.velb` contiene `calln @Method("kernel32.dll:Get...")`.
 La llamada se ejecuta cada vez que el programa corre.
- **FFI compile-time** (este patron): el `.velb` contiene solo
 `mov rN, <literal>`. La llamada se hizo UNA vez al compilar. No hay
 referencia a la DLL en el binario.

**Capabilities** del FFI compile-time:

- Hasta 12 argumentos primitivos (`int`, `bool`, `char`, `null`, `string literal`,
 `float bitcast`).
- Return types soportados: `STRING`, `F32`, `F64`, `int`, `bool`, `char`, `ptr`.
- Strings interpolados (`"a=${a}"`) son validos como argumentos.
- Strings literales se pasan como `c_str` (null-terminated host pointer).
- Sin marshalling de structs/arrays todavia (deferred).

---

## 7. `static_assert` y el virtual lib `vesta_comptime`

`static_assert(cond, "msg")` verifica una condicion en compile-time. Si
la cond es falsa, el compilador emite error y NO genera el `.velb`:

```vx
static_assert(sizeof<u64>() == 8, "u64 debe ser 8 bytes");
static_assert(sizeof<u32>() == 4, "u32 debe ser 4 bytes");

@Macro
comptime string size_table() {
    static_assert(sizeof<u64>() + sizeof<u32>() == 12, "size mismatch");
    return to_str(sizeof<u64>() + sizeof<u32>()); // "12"
}
```

### `vesta_comptime`: virtual lib in-process

`vesta_comptime` es un namespace logico que expone funciones C++ del propio
binario como si fueran un DLL. **No requiere declaracion `extern`**: el
compilador lo detecta automaticamente cuando ve un nombre registrado.

Funciones disponibles:

```vx
@Macro
comptime string demo_virtual_lib() {
    // Queries de tipos via virtual lib (sin extern explicito).
    u64 sz = comptime_type_sizeof("u64"); // 8
    u64 al = comptime_type_alignof("f64"); // 8
    u32 k = comptime_type_kind("Punto"); // STRUCT
    return to_str(sz);
}
```

Si quisieras ser explicito, podrias declarar:

```vx
extern "vesta_comptime" {
    fn comptime_type_sizeof(string name) -> u64;
    fn comptime_type_alignof(string name) -> u64;
    fn comptime_type_kind(string name) -> u32;
    fn static_assert(bool cond, string msg) -> u64;
}
```

Pero no es necesario: el resolver de FFI implicit los encuentra solos
(`ffi::lookup_virtual_fn("vesta_comptime", name)`).

---

## 8. Patrones comunes

### Tabla constante generada en compile-time

```vx
@Pure @Macro
comptime string fact(i64 n) {
    i64 r = 1;
    i64 i = 2;
    while (i <= n) { r = r * i; i = i + 1; }
    return to_str(r);
}

i64 f10 = fact(10); // se compila como `i64 f10 = 3628800;`
```

`@Pure` permite que el compilador cachee resultados de invocaciones
repetidas con los mismos args.

### Template parametrizado

```vx
@Macro
comptime string make_getter(string field_name) {
    return "i64 get_" + field_name + "() { return this." + field_name + "; }";
}

// Uso (DEPENDE del contexto de inyeccion -- ver Limitaciones).
// En la mayoria de casos, mejor usar properties get/set del lenguaje.
```

### Pointer chase generado (DSL con `expr`)

Ver seccion 2 -- `walk(root -> 0x100 -> 0)` emite el chain de derefs.

### Build fingerprint compile-time

```vx
extern "kernel32.dll" { fn GetTickCount() -> u32; }

@Macro
comptime string compile_timestamp() {
    return to_str(GetTickCount());
}

const u64 BUILD_TIME = compile_timestamp();
// El binario contiene literalmente el tick count del momento de compilar.
```

### Operaciones de string como builders

```vx
@Macro
comptime string banner(string text) {
    string border = repeat("=", strlen(text) + 4);
    return "\"" + border + "\\n= " + text + " =\\n" + border + "\"";
}

string banner_msg = banner("Vesta v1.0");
// banner_msg recibe la string con el banner pre-formateado.
```

### Compilador embebido: Brainfuck -> Vesta (solo comptime)

Un `@Macro` puede implementar un **compilador completo** en compile-time: recibe
el fuente de otro lenguaje y EMITE codigo Vesta valido, que luego se compila y
ejecuta como cualquier otro codigo (cero interprete en runtime).  El ejemplo
[`311_bf_comptime.vx`](../../../examples_codes_vx/311_bf_comptime.vx) compila
Brainfuck a un lambda Vesta.  Muestra que las comptime fn soportan **`enum` +
`match`**, indexado de string **`s[i]`**, y llamadas anidadas entre comptime fn:

```vx
// Los 8 comandos Brainfuck como enum de tokens.
enum BfTok { Right, Left, Inc, Dec, Out, In, Loop, EndLoop, Nop }

// caracter -> token (match sobre char).
comptime BfTok bf_classify(char c) {
    match c {
        case '>' => return BfTok.Right;   case '<' => return BfTok.Left;
        case '+' => return BfTok.Inc;     case '-' => return BfTok.Dec;
        case '.' => return BfTok.Out;     case ',' => return BfTok.In;
        case '[' => return BfTok.Loop;    case ']' => return BfTok.EndLoop;
        case _   => return BfTok.Nop;
    }
    return BfTok.Nop;
}

// token -> fragmento de codigo Vesta (match sobre el enum).
comptime string bf_emit(BfTok t) {
    match t {
        case Right   => return " p = p + 1;";
        case Inc     => return " tape[p] = tape[p] + 1;";
        case Loop    => return " while (tape[p] != 0) {";
        case EndLoop => return " }";
        case Out     => return " print_char(tape[p]);";
        // ... resto de arms ...
        case Nop     => return "";
    }
    return "";
}

comptime string bf_compile_body(string src) {
    string body = "";
    for (i64 i = 0; i < strlen(src); i++) {
        BfTok t = bf_classify(src[i]);   // s[i] devuelve el byte i-esimo
        body += bf_emit(t);
    }
    return body;
}

// El macro EMITE un lambda; su tipo de retorno se infiere del `return`.
@Macro comptime string bf(expr src) {
    return "() => { i32[4096] tape; i32 p = 0;" + bf_compile_body(src) +
           " return tape[p]; }";
}

i32 main() {
    // 6*7 = 42 en la celda 1; el fuente BF se pasa como expresion cruda.
    fn() -> i32 programa = bf(++++++[>+++++++<-]>);
    return programa();   // 42
}
```

Notas del ejemplo:

- **`enum` + `match` en comptime**: las comptime fn se bajan a bytecode y corren
  en la ComptimeVM; `match` (sobre char, entero, string o enum) funciona igual
  que en runtime.
- **`s[i]`**: el indexado de string por byte funciona en compile-time (y en
  interp/JIT).  Para ASCII / UTF-8 de 1 byte coincide con el codepoint.
- **Inferencia del lambda emitido**: el lambda `() => { ...; return tape[p]; }`
  que emite el macro no lleva tipo de retorno declarado; se infiere del `return`.
- **`expr` capture**: el fuente Brainfuck se pasa sin comillas.  Debe tener los
  corchetes `[ ]` balanceados y no contener `,` (colisiona con el separador de
  argumentos); por eso el ejemplo cubre el nucleo `> < + - [ ] .`.

---

## 9. Limitaciones y workarounds

### Estructuras runtime NO disponibles en compile-time

`ArrayList`, `HashMap`, `HashSet`, `Queue`, `Deque`, `TreeMap`, `TreeSet`
viven en el plugin nativo `vesta_collections.dll` y NO funcionan en macros.

**Workaround**: usar arrays nativos `T[N]` y/o pares de arrays paralelos
(ver seccion 5). Para datasets pequenos (< 1000 entries) es perfectamente
eficiente porque la busqueda lineal ocurre solo en compile-time.

### Forward references

Un `@Macro` debe declararse **antes** de su uso en el archivo. Single-pass
parser sin forward-decl scanner.

**Workaround**: ordenar las decls top-to-bottom.

### Macros que devuelven inicializadores de array/struct

`u64 arr[4] = my_macro(...)` no funciona si el macro emite `{1, 2, 3, 4}`.
Los init lists solo son validos en el path estatico del var-decl.

**Workaround**: emitir una expresion escalar (suma, primer elemento, etc.) o
usar el macro para emitir cada elemento individualmente.

### Builtins `comptime_compile` / `comptime_emit_expr` / `comptime_type`

Estos builtins son comptime-only y NO se lowean a IR. Macros que los usen
caen al AST evaluator (mas lento que VM eval pero funcional).

### Macros con args de tipos compuestos (struct, array)

Los args de un macro deben ser primitivos (`i64`, `string`, `bool`, etc.)
o literales encodables. Pasar `Punto p = {...}; my_macro(p)` no esta
soportado en la VM eval -- cae al AST evaluator.

**Workaround**: pasar los fields del struct como args separados, o usar
`expr` para capturar la expresion del call site como texto.

---

## Ver tambien

- [TiposDatos.md](./TiposDatos.md) -- tipos primitivos, structs, arrays.
- [Strings.md](./Strings.md) -- builtins de string runtime (compatibles
 con los comptime).
- [FFI.md](./FFI.md) -- FFI runtime tradicional.
- [ReflexionAOP.md](./ReflexionAOP.md) -- reflexion runtime (`forName`,
 `getClass`, AOP advice).
- [examples_codes_vx/159_macro_expr_capture.vx](../../../examples_codes_vx/159_macro_expr_capture.vx)
 -- demo de `expr` capture.
- [examples_codes_vx/160_macro_walk_pchase.vx](../../../examples_codes_vx/160_macro_walk_pchase.vx)
 -- macro `walk` real con DSL.
- [examples_codes_vx/161_macro_ffi_compile_time.vx](../../../examples_codes_vx/161_macro_ffi_compile_time.vx)
 -- FFI a kernel32 + virtual lib.
- [examples_codes_vx/162_macro_comptime_data.vx](../../../examples_codes_vx/162_macro_comptime_data.vx)
 -- arrays, structs, dict via arrays paralelos.
