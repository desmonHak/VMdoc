# Closures y lambdas en Vex

Vex soporta funciones de primera clase: lambdas inline con captura léxica, paso de
funciones como argumentos (HOF), y promoción automática de funciones top-level a
function values.

---

## Indice

1. [Sintaxis de lambdas](#1-sintaxis-de-lambdas)
2. [Tipo `fn(...)->R`](#2-tipo-fn-r)
3. [Captura léxica (by-value)](#3-captura-lexica-by-value)
4. [Captura by-reference (mutable)](#4-captura-by-reference-mutable)
5. [Funciones top-level como function values](#5-funciones-top-level-como-function-values)
6. [HOF: pasar funciones como argumentos](#6-hof-pasar-funciones-como-argumentos)
7. [Closures que escapan (factory functions)](#7-closures-que-escapan-factory-functions)
8. [Modelo de runtime: function value + env block](#8-modelo-de-runtime-function-value--env-block)
9. [Limitaciones](#9-limitaciones)

---

## 1. Sintaxis de lambdas

```vex
// Expression-bodied: 1 sola expresion
(i32 x, i32 y) => x + y

// Block-bodied: cuerpo completo con statements
(i32 n) => { 
    if (n < 0) return 0;
    return n * 2;
}

// Sin parametros
() => 42

// Parametros sin tipo (inferidos del contexto)
fn(i32, i32) -> i32 sum = (a, b) => a + b;
```

- **Expression body**: el valor de la expresion es el return.
- **Block body**: usar `return` explicito.
- **Parametros sin tipo**: el compilador infiere del tipo declarado de la variable
  destino (`fn(T1, T2) -> R` declarada -> los params son T1, T2).

---

## 2. Tipo `fn(...)->R`

```vex
fn(i32, i32) -> i32 op;        // funcion que toma 2 i32, devuelve i32
fn() -> void noop;              // funcion sin params ni retorno
fn(string) -> bool predicate;  // funcion que toma string, devuelve bool
```

`fn(...)` es un primitive kind dedicado (`PrimitiveKind::FUNCTION`). Internamente
se representa como un slot de **16 bytes** en stack:

```
+-------------+-------------+
|  fn_addr    |  env_addr   |
| (8 bytes)   | (8 bytes)   |
+-------------+-------------+
```

- `fn_addr`: dirección del código (label en `.vel`).
- `env_addr`: dirección del environment block con las capturas (0 si no hay).

---

## 3. Captura léxica (by-value)

```vex
i32 main() {
    i32 y = 25;
    fn(i32) -> i32 add_y = (i32 x) => x + y;  // captura `y` por valor
    return add_y(5);                            // = 30
}
```

El compilador detecta los identificadores libres en el body de la lambda y los
captura en un environment block. La captura es por valor: el snapshot de `y` se
toma en el momento de crear la lambda.

```vex
i32 y = 10;
fn() -> i32 get = () => y;
y = 999;                       // mutar y DESPUES de crear la lambda
println("${get()}");           // imprime 10 (NO 999) - by-value snapshot
```

**Lowering**: cada lambda con captura genera un helper sintético `__lambda_<N>`
con prologue que carga las capturas desde `r14 + 8*i` (donde r14 = env_addr).
El cuerpo de la lambda se compila como función separada.

---

## 4. Captura by-reference (mutable)

Cuando la lambda **modifica** una variable capturada (asignación al identifier),
el compilador automáticamente la captura por referencia:

```vex
i32 main() {
    i32 sum = 0;                              // se promociona a address-taken
    fn(i32) -> void add = (i32 x) => { sum = sum + x; };
    add(10);
    add(20);
    return sum;                                // = 30
}
```

**Detección**: el type checker (`LambdaExpr::mutable_captures`) marca las
variables que aparecen como lhs de `=` dentro del body. El lowering:

1. Marca esas variables como address-taken en el outer scope (ALLOCA estable +
   LOAD/STORE para todos los accesos).
2. En `lower_lambda_expr`, el slot del env guarda el PUNTERO de la celda (no el
   valor).
3. En `generate_lambda_helper`, el prologue hace LOAD del puntero y bindea el
   nombre como `address_taken_local` del helper.
4. Reads/writes en el body del helper se traducen a LOAD/STORE indirectos via
   el puntero.

Resultado: capture-by-reference puro, las modificaciones desde la lambda se ven
fuera de ella.

---

## 5. Funciones top-level como function values

Funciones declaradas a nivel de módulo se promocionan automáticamente a function
values cuando se pasan como argumento a un parametro `fn(...)`:

```vex
i32 add2(i32 a, i32 b) { return a + b; }
i32 mul2(i32 a, i32 b) { return a * b; }

i32 reduce(fn(i32, i32) -> i32 op, i32 init, i32[] arr) {
    i32 acc = init;
    for (i32 x : arr) { acc = op(acc, x); }
    return acc;
}

i32 main() {
    i32[] data = {1, 2, 3, 4};
    i32 sum = reduce(add2, 0, data);     // add2 promovido a fn(i32,i32)->i32
    i32 prod = reduce(mul2, 1, data);    // idem mul2
    return sum + prod;
}
```

**Lowering (A.14)**: el type checker detecta `parametro tipo FUNCTION + arg IdentExpr
resolviendo a Symbol::Function` y sintetiza el tipo `fn(T1...) -> R` del argumento
para validar firma con `types_assignable`. El lowering emite un slot de 16 bytes
con `fn_addr = @Absolute("code.<fn_name>")` y `env_addr = 0` (sin captura).

El callee (helper sintético de lambda o `CALLCLOSURE`) ignora env_addr cuando es 0
(no toca r14). Cero overhead adicional vs llamada directa.

---

## 6. HOF: pasar funciones como argumentos

```vex
i32 each(i32[] arr, fn(i32) -> void f) {
    for (i32 x : arr) { f(x); }
    return arr.length;
}

i32 map_sum(i32[] arr, fn(i32) -> i32 transform) {
    i32 sum = 0;
    for (i32 x : arr) { sum = sum + transform(x); }
    return sum;
}

i32 main() {
    i32[] xs = {1, 2, 3, 4, 5};
    each(xs, (x) => println("${x}"));          // lambda inline
    i32 doubles_sum = map_sum(xs, (x) => x * 2);  // = 30
    return doubles_sum;
}
```

Patrón funcional clásico. El runtime invoca via opcode `callclosure` (env_addr en
R14, args en R1..R12, return en R0).

---

## 7. Closures que escapan (factory functions)

Una función que **retorna** una closure (factory):

```vex
fn(i32) -> i32 make_adder(i32 n) {
    return (i32 x) => x + n;     // captura `n`
}

i32 main() {
    fn(i32) -> i32 add5 = make_adder(5);
    fn(i32) -> i32 add10 = make_adder(10);
    return add5(20) + add10(30);              // = 25 + 40 = 65
}
```

**Implementación** (A.15, gap O cerrado):

1. **SRET para tipo FUNCTION**: `make_adder` se transforma a
   `void make_adder(retbuf: ptr, n: i32)`. El caller aloca 16 bytes en stack y
   pasa el puntero como primer arg hidden.
2. **Env block en HEAP RAW** (no stack): cuando la lambda captura variables y la
   función contenedora retorna FUNCTION, el env se aloca via `RAW_ALLOC` (host
   heap) en lugar de ALLOCA. Sobrevive al `ret` de make_adder.
3. **Bug fix regalloc colateral**: el evacuation de fn_addr a r13 en CALLCLOSURE
   se extendió para incluir registros de argumento (r1..r12) que el
   parallel-move podría destruir.

Coste: leak por env (no hay free automático). Solucionable con escape analysis +
promoción al GC heap; deferido a Phase B.

---

## 8. Modelo de runtime: function value + env block

### Function value (16 bytes en stack)

```
+-------------+-------------+
|  fn_addr    |  env_addr   |
| (8 bytes)   | (8 bytes)   |
+-------------+-------------+
   |              |
   v              v
 código       env block:
 del          [captura_0][captura_1]...
 helper       (N * 8 bytes, en stack o heap)
```

### Call via `callclosure` (opcode IR 0x86)

El emisor IR para `CALLCLOSURE`:

1. Evacúa fn_addr a r13 si va a quedar clobbeado por el parallel-move.
2. Parallel-move args a R1..R12.
3. `mov r14, env_addr`.
4. `mov r15, nargs`.
5. `callvmr fn_addr` (call indirecto al helper).

El helper sintético `__lambda_<N>`:

1. Lee capturas desde `[r14 + 0]`, `[r14 + 8]`, etc., y las bindea a los
   nombres del closure.
2. Ejecuta el body original como función normal.
3. Return en r0 (mismo ABI que cualquier función).

---

## 9. Limitaciones

1. **Env leak en factory functions** (gap O remanente): los env blocks alocados
   en heap RAW para closures que retornan no tienen free automático. Deferido a
   Phase B con escape analysis + promoción al GC heap.

2. **`++`/`--` dentro de lambdas**: no soportados directamente (Vex en general).
   Usar `x = x + 1` explícito.

3. **Recursión directa en lambda**: una lambda no puede referenciarse a si misma
   por nombre (no tiene nombre). Workaround: declarar función top-level con
   nombre y usar la promoción automática del paso 5.

4. **Function values en colecciones**: técnicamente se puede `fn(i32)->i32[]`
   pero el storage es 16 bytes/elemento (no 8). Las APIs de `ArrayList<T>` etc.
   asumen 8 bytes/elemento — usar wrappers manuales.

---

Ver también: [[TiposDatos]] (tipos de función), [[OOP]] (métodos vs lambdas),
[[ControlFlow]] (HOF con `for x in arr`), [[Async]] (function values en spawn).
