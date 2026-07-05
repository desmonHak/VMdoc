# Closures y lambdas en Vesta

Vesta soporta funciones de primera clase: lambdas inline con captura lexica, paso de
funciones como argumentos (HOF), promocion automatica de funciones top-level a
function values, closures que escapan de su scope, closures almacenados en campos
y punteros a funcion crudos estilo C (`cfn`).

---

## Indice

1. [Sintaxis de lambdas](#1-sintaxis-de-lambdas)
2. [Tipo `fn(...)->R`](#2-tipo-fn-r)
3. [Captura lexica (by-value)](#3-captura-lexica-by-value)
4. [Captura by-reference (mutable)](#4-captura-by-reference-mutable)
5. [Funciones top-level como function values](#5-funciones-top-level-como-function-values)
6. [HOF: pasar funciones como argumentos](#6-hof-pasar-funciones-como-argumentos)
7. [Closures que escapan (factory functions)](#7-closures-que-escapan-factory-functions)
8. [Modelo de runtime: function value + env block](#8-modelo-de-runtime-function-value--env-block)
9. [Closures almacenados en campos](#9-closures-almacenados-en-campos)
10. [Valores de funcion: `fn(...)->R` vs `cfn(...)->R`](#10-valores-de-funcion-fn-r-vs-cfn-r)
11. [Limitaciones](#11-limitaciones)

---

## 1. Sintaxis de lambdas

```vx
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

```vx
fn(i32, i32) -> i32 op; // funcion que toma 2 i32, devuelve i32
fn() -> void noop; // funcion sin params ni retorno
fn(string) -> bool predicate; // funcion que toma string, devuelve bool
```

`fn(...)` es un primitive kind dedicado (`PrimitiveKind::FUNCTION`). Internamente
se representa como un slot de **16 bytes** (fat-pointer) en stack:

```
+-------------+-------------+
| fn_addr | env_addr |
| (8 bytes) | (8 bytes) |
+-------------+-------------+
```

- `fn_addr`: direccion del codigo (label en `.vel`).
- `env_addr`: direccion del environment block con las capturas (0 si no hay).

Un lambda (`fn`) NO es lo mismo que un puntero a funcion crudo. Para el puntero
crudo de 8 bytes estilo C, ver la seccion [`cfn`](#10-valores-de-funcion-fn-r-vs-cfn-r).

---

## 3. Captura lexica (by-value)

```vx
i32 main() {
    i32 y = 25;
    fn(i32) -> i32 add_y = (i32 x) => x + y; // captura `y` por valor
    return add_y(5); // = 30
}
```

El compilador detecta los identificadores libres en el body de la lambda y los
captura en un environment block. La captura es por valor: el snapshot de `y` se
toma en el momento de crear la lambda.

```vx
i32 y = 10;
fn() -> i32 get = => y;
y = 999; // mutar y DESPUES de crear la lambda
println("${get()}"); // imprime 10 (NO 999) - by-value snapshot
```

**Lowering**: cada lambda con captura genera un helper sintetico `__lambda_<N>`
con prologue que carga las capturas desde `r14 + 8*i` (donde r14 = env_addr).
El cuerpo de la lambda se compila como funcion separada.

---

## 4. Captura by-reference (mutable)

Cuando la lambda **modifica** una variable capturada (asignacion al identifier),
el compilador automaticamente la captura por referencia:

```vx
i32 main() {
    i32 sum = 0; // se promociona a address-taken
    fn(i32) -> void add = (i32 x) => { sum = sum + x; };
    add(10);
    add(20);
    return sum; // = 30
}
```

**Deteccion**: el type checker (`LambdaExpr::mutable_captures`) marca las
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

Funciones declaradas a nivel de modulo se promocionan automaticamente a function
values cuando se pasan como argumento a un parametro `fn(...)`:

```vx
i32 add2(i32 a, i32 b) { return a + b; }
i32 mul2(i32 a, i32 b) { return a * b; }

i32 reduce(fn(i32, i32) -> i32 op, i32 init, i32[] arr) {
    i32 acc = init;
    for (i32 x : arr) { acc = op(acc, x); }
    return acc;
}

i32 main() {
    i32[] data = {1, 2, 3, 4};
    i32 sum = reduce(add2, 0, data); // add2 promovido a fn(i32,i32)->i32
    i32 prod = reduce(mul2, 1, data); // idem mul2
    return sum + prod;
}
```

**Lowering**: el type checker detecta `parametro tipo FUNCTION + arg IdentExpr
resolviendo a Symbol::Function` y sintetiza el tipo `fn(T1...) -> R` del argumento
para validar firma con `types_assignable`. El lowering emite un slot de 16 bytes
con `fn_addr = @Absolute("code.<fn_name>")` y `env_addr = 0` (sin captura).

El callee (helper sintetico de lambda o `CALLCLOSURE`) ignora env_addr cuando es 0
(no toca r14). Cero overhead adicional vs llamada directa.

La promocion tambien funciona al asignar el nombre desnudo de una funcion a una
variable o campo de tipo `fn(...)`/`cfn(...)` (`cfn c = doblar; o.f = triplicar;`).

---

## 6. HOF: pasar funciones como argumentos

```vx
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
    each(xs, (x) => println("${x}")); // lambda inline
    i32 doubles_sum = map_sum(xs, (x) => x * 2); // = 30
    return doubles_sum;
}
```

Patron funcional clasico. El runtime invoca via opcode `callclosure` (env_addr en
R14, args en R1..R12, return en R0).

---

## 7. Closures que escapan (factory functions)

Una funcion que **retorna** una closure (factory):

```vx
fn(i32) -> i32 make_adder(i32 n) {
    return (i32 x) => x + n; // captura `n`
}

i32 main() {
    fn(i32) -> i32 add5 = make_adder(5);
    fn(i32) -> i32 add10 = make_adder(10);
    return add5(20) + add10(30); // = 25 + 40 = 65
}
```

**Implementacion**:

1. **SRET para tipo FUNCTION**: `make_adder` se transforma a
 `void make_adder(retbuf: ptr, n: i32)`. El caller aloca 16 bytes en stack y
 pasa el puntero como primer arg hidden.
2. **Env block en el heap gestionado (GC)**: cuando la lambda captura variables y
 la funcion contenedora retorna un valor de funcion, el env NO se aloca en el
 stack sino en el heap gestionado por el recolector de basura. Asi sobrevive al
 `ret` de `make_adder` y la closure devuelta sigue siendo valida en el caller.

**Sin fuga de memoria**: el env entra en la tabla de handles del GC como cualquier
otro objeto gestionado; el recolector lo libera automaticamente cuando ningun root
lo alcanza. Verificado creando 1M de closures que escapan sin agotar la memoria.
No hace falta liberacion manual.

---

## 8. Modelo de runtime: function value + env block

### Function value (16 bytes en stack)

```
+-------------+-------------+
| fn_addr | env_addr |
| (8 bytes) | (8 bytes) |
+-------------+-------------+
    | |
    v v
    codigo env block:
    del [captura_0][captura_1]...
    helper (N * 8 bytes; stack, heap gestionado o campo segun el caso)
```

### Call via `callclosure`

El emisor IR para `CALLCLOSURE`:

1. Evacua fn_addr a r13 si va a quedar clobbeado por el parallel-move.
2. Parallel-move args a R1..R12.
3. `mov r14, env_addr`.
4. `mov r15, nargs`.
5. `callvmr fn_addr` (call indirecto al helper).

El helper sintetico `__lambda_<N>`:

1. Lee capturas desde `[r14 + 0]`, `[r14 + 8]`, etc., y las bindea a los
 nombres del closure.
2. Ejecuta el body original como funcion normal.
3. Return en r0 (mismo ABI que cualquier funcion).

---

## 9. Closures almacenados en campos

Un closure con captura puede guardarse en un campo de una clase o de un struct. El
modelo de ownership es determinista y no depende del GC: el env es propiedad del
objeto contenedor y se libera cuando este se destruye.

```vx
class Contador {
    fn(i32) -> i32 op; // campo de tipo funcion

    public Contador(i32 base) {
        this.op = (i32 x) => x + base; // captura `base`
    }

    public i32 aplicar(i32 x) {
        return this.op(x); // llamada indirecta al closure del campo
    }
}
```

- **Campo de clase**: el env del closure se aloca en el heap y su ownership
 pertenece al objeto contenedor. El destructor del objeto (sintetico o explicito)
 libera el env. Reasignar el campo (`o.op = otro;`) libera primero el env anterior,
 igual que reasignar un `unique<T>`. No hay fuga ni doble liberacion; verificado
 con valgrind (0 bytes en uso al salir) en un stress de 100k iteraciones de crear,
 reasignar, llamar y destruir.
- **Campo de struct**: como el struct es un value-type sin destructor de campos, el
 env vive en el stack junto con el struct y muere con su scope (coste cero). Si el
 struct escapa de su scope llevandose un closure capturador (por ejemplo, un
 `return` del struct completo), el compilador promociona el env al heap para que
 siga siendo valido, o rechaza el caso con un error claro cuando no puede
 garantizar la validez.
- **Captura por referencia que escapa a un campo de clase**: es un error de
 compilacion (la celda referenciada moriria antes que el objeto contenedor).

---

## 10. Valores de funcion: `fn(...)->R` vs `cfn(...)->R`

Vesta tiene **dos** tipos de valor de funcion, que NO son intercambiables. La
eleccion importa mucho en structs/clases, tablas de despacho, FFI y codigo
bare-metal/kernel.

```
lambda (fn)   !=   puntero a funcion crudo (cfn)   !=   metodo
```

### 10.1 Los dos tipos

**`fn(...)->R` -- lambda / closure.** Un **fat-pointer de 16 bytes**
`{fn_addr, env_addr}`: la direccion del codigo MAS un puntero al entorno con las
variables capturadas. Es el tipo natural para funciones anonimas con estado
(closures). Se invoca por `CALLCLOSURE` (el env viaja en R14). Ver secciones 2-9.

**`cfn(...)->R` -- puntero a funcion crudo estilo C.** Un valor de **8 bytes**:
SOLO la direccion de una funcion, sin entorno. Equivale a `R (*)(args)` en C. Se
invoca por llamada indirecta directa (`CALLIND`). No captura nada. Es lo que
espera un callback de C, una api-table de un kernel/driver, o una tabla de
despacho, donde un fat-pointer de 16 bytes no es aceptable.

```vx
i64 doblar(i64 x)    { return x * 2; }
i64 triplicar(i64 x) { return x * 3; }

i32 main() {
    // cfn: puntero crudo de 8 bytes; llamada indirecta directa.
    cfn(i64) -> i64 f = &doblar;
    if (f(21) != 42) { return 1; }        // CALLIND -> doblar(21) = 42

    // fn: closure de 16 bytes que captura estado.
    i64 factor = 7;
    fn(i64) -> i64 escalar = (x) => x * factor;   // captura `factor`
    if (escalar(6) != 42) { return 2; }   // CALLCLOSURE -> 42
    return 42;
}
```

### 10.2 Tabla comparativa

| Aspecto            | `fn(...)->R` (lambda)          | `cfn(...)->R` (puntero crudo) |
|:-------------------|:------------------------------|:------------------------------|
| Tamano             | 16 bytes `{fn_addr, env_addr}`| 8 bytes (solo la direccion)   |
| Entorno (capturas) | Si (env block)                | No                            |
| Invocacion         | `CALLCLOSURE` (env en R14)     | `CALLIND` directo             |
| Estado             | Con estado (closure)          | Sin estado                    |
| Apto para FFI/C    | No (C espera puntero crudo)   | Si                            |
| Apto para tablas   | No (16 B/elem)                | Si (8 B/elem, como C)         |
| `(i64) valor`      | ERROR (16 no cabe en 8)       | OK -- la direccion tal cual   |
| Caso natural       | variable local / campo        | tabla, callback, kernel       |

Un lambda son 16 bytes y por eso `(i64)(fn(...))f` es un **error de compilacion**
(no cabe en un entero de 8 bytes). Para obtener una direccion cruda usa
`cfn` / `&`; para deconstruir el `{fn_addr, env_addr}` de un lambda, castea a un
struct de dos `i64` (16 == 16).

### 10.3 Tomar la direccion de una funcion o metodo

- **`&funcion`** produce un `cfn` que apunta a una funcion libre.
- **`nombre` desnudo** (sin `&`) se promociona automaticamente a la direccion de
  la funcion cuando el destino es de tipo `cfn(...)` o `fn(...)`: al declarar una
  variable, asignar a una variable o asignar a un campo. Ejemplo real del canonico
  199: `o.op = triplicar;` equivale a `o.op = &triplicar;`.
- **`&Tipo.metodo`** produce un `cfn` **NO ligado**: apunta a la funcion libre del
  metodo (`Tipo__metodo`), que toma el receptor como PRIMER parametro explicito.
  Para un struct value-type la firma es `cfn(Tipo*, ...)`; para una clase es
  `cfn(Tipo, ...)` (la referencia ya es un puntero). El `cfn` NO lleva el objeto:
  hay que pasarlo en cada llamada.
- **`&obj.metodo`** produce un `fn` **LIGADO**: azucar de un closure que CAPTURA el
  receptor. Opera siempre sobre ese objeto; NO es un `cfn`.

```vx
struct Calculadora {
    i64 base;
    i64 sumar(i64 x) { return this.base + x; }
}

// &Tipo.metodo -- cfn NO ligado: el receptor va como primer arg explicito.
cfn(Calculadora*, i64) -> i64 pm = &Calculadora.sumar;
Calculadora c2; c2.base = 37;
i64 r = pm(&c2, 5);              // sumar(c2, 5) = 42
```

```vx
// &obj.metodo -- fn LIGADO: captura el receptor (es un closure, 16 bytes).
Contador c = new Contador();
fn() -> i64 paso = &c.inc;      // opera siempre sobre c
paso(); paso();                 // c.n = 2
```

Contraste directo: `&Tipo.metodo` es un `cfn` de 8 bytes que exige pasar el
receptor cada vez; `&obj.metodo` es un `fn` de 16 bytes que ya lleva el receptor
dentro. Ver `examples_codes_vx/199_cfn_vs_lambda.vx` (seccion 2h) y
`examples_codes_vx/201_metodo_ligado.vx`.

### 10.4 Un metodo NO es ninguno de los dos

Un metodo de struct/clase es codigo asociado al TIPO, no a la instancia: no ocupa
bytes en cada objeto. Su dispatch es DIRECTO (`CALL`, inlineable), decidido en
compile-time y no reasignable. Solo cuando pones un puntero a funcion como CAMPO
(`cfn(...)` o `fn(...)`) obtienes un valor reasignable e indirecto por instancia.
El compilador distingue "metodo" de "campo puntero-a-funcion": el campo solo
entra si NO existe un metodo con ese nombre.

### 10.5 Tablas de despacho (patron kernel / api-table)

Un `cfn` es lo que se guarda en tablas de funciones, como en una api-table de C o
de un kernel. Las direcciones se pueden guardar como `u64` y castear a `cfn` al
invocar:

```vx
u64[3] tabla;
tabla[0] = (u64) &doblar;
tabla[1] = (u64) &triplicar;
tabla[2] = (u64) &negar;
i64 suma = 0;
for (i64 i = 0; i < 3; i = i + 1) {
    suma = suma + ((cfn(i64) -> i64) tabla[i])(10);   // CALLIND por indice
}
```

Como campo de struct, un `cfn` ocupa 8 bytes y es reasignable en runtime:

```vx
struct Operacion {
    i64 valor;
    cfn(i64) -> i64 op;         // puntero a funcion crudo (8 B), reasignable
}

Operacion o; o.valor = 10;
o.op = &doblar;                 // guarda la DIRECCION (8 B)
i64 r = (o.op)(o.valor);        // CALLIND -> doblar(10) = 20
```

La firma de un `cfn` es solo compile-time: se puede castear entre firmas
distintas (`(cfn(u64)->u64) cfn_i64`). El cast `(cfn(...)) direccion` devuelve la
direccion cruda tal cual (no la envuelve en un slot de lambda).

La propia stdlib usa este patron para poner direcciones de trampolines en una
pila de contexto de fibra, casteando la direccion de la funcion a `cfn` para
tratarla como el `u64` que es (`stdlib/vx/vx_fiber.vx`, `stdlib/vx/vx_async.vx`):

```vx
s[6] = (u64)(cfn() -> void) __fiber_trampoline;   // retaddr que consume el ret
```

### 10.6 FFI: `extern` como `cfn`

Un `cfn` es lo que necesita un callback de C: el header C lo declara como
`R (*f)(args)` y un programa C pasa una funcion suya directamente
(`examples_codes_vx/205_c_interop.vx`):

```vx
// El parametro es un puntero a funcion crudo estilo C.
i64 apply(cfn(i64) -> i64 f, i64 x) { return f(x); }
```

Tambien puedes tomar la direccion de una funcion `extern` (FFI) y usarla como
`cfn`. Como la direccion nativa no es bytecode de la VM (no se puede llamar
directamente por la ruta interna de llamadas), el compilador genera de forma
automatica un thunk que envuelve la llamada nativa; `&extern` apunta a ese thunk,
que se invoca por `CALLIND` y reenvia al FFI:

```vx
extern "msvcrt.dll" { fn toupper(i32 c) -> i32; }

cfn(i32) -> i32 mayus = &toupper;   // genera un thunk automatico
i32 up = mayus(97);                 // toupper('a') = 'A' = 65
```

### 10.7 `cfn` con ownership: `unique<cfn>` / `borrow<cfn>`

Como un `cfn` es un valor de 8 bytes (igual que un primitivo), entra en un
`unique<...>` / `borrow<...>` como cualquier otro valor de 8 bytes; NO se
construye un slot de lambda:

```vx
cfn(i64) -> i64 cc = &doblar;
unique<cfn(i64) -> i64> up = unique_box(cc);   // toma posesion (8 B en heap)
unique<cfn(i64) -> i64> uq = move(up);          // transfiere ownership
cfn(i64) -> i64 desbox = *ptr_of(uq);           // recupera el cfn (1 LOAD)
if (desbox(21) != 42) { return 10; }            // doblar(21) = 42
```

Un `borrow<cfn(...)>` presta el `cfn` sin tomar ownership; se recupera con
`read_borrow`:

```vx
i64 aplicar_borrow(borrow<cfn(i64) -> i64> b, i64 x) {
    cfn(i64) -> i64 f = read_borrow(b);
    return f(x);
}
```

### 10.8 Devirtualizacion de un `cfn` constante

Cuando el `cfn` es una constante conocida en compile-time, el optimizador
reescribe la llamada indirecta (`CALLIND`) a un `CALL` directo a la funcion, y el
inliner puede absorber su cuerpo. En el ejemplo siguiente `(21)*2` se pliega a 42
en compile-time (cero llamada en runtime):

```vx
cfn(i64) -> i64 conocido = &doblar;
if (conocido(21) != 42) { return 12; }   // devirt + inline + const-fold -> 42
```

Ejemplo completo que contrasta metodo / `cfn` / lambda en los tres backends
(interprete, JIT y AOT): `examples_codes_vx/199_cfn_vs_lambda.vx`.

---

## 11. Limitaciones

1. **`++`/`--` dentro de lambdas**: no soportados directamente (Vesta en general).
 Usar `x = x + 1` explicito.

2. **Recursion directa en lambda**: una lambda no puede referenciarse a si misma
 por nombre (no tiene nombre). Workaround: declarar funcion top-level con
 nombre y usar la promocion automatica de la seccion 5.

3. **Function values en colecciones**: tecnicamente se puede `fn(i32)->i32[]`
 pero el storage es 16 bytes/elemento (no 8). Las APIs de `ArrayList<T>` etc.
 asumen 8 bytes/elemento -- usar wrappers manuales, o `cfn(...)` (8 bytes) cuando
 no se necesita captura.

---

Ver tambien: [[TiposDatos]] (tipos de funcion), [[OOP]] (metodos vs lambdas),
[[ControlFlow]] (HOF con `for x in arr`), [[Async]] (function values en spawn).
