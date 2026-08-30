# Optional y Result en Vesta

`Optional<T>` y `Result<V, E>` son **builtins del compilador** (NO templates):
PrimitiveKinds dedicados con layout fijo en stack del caller (cero heap),
ABI SRET, y sin overhead de GC.

---

## Indice

1. [``Optional<T>``: valor que puede no existir](#1-optionalt-valor-que-puede-no-existir)
2. [Result<V, E>: éxito o error](#2-resultv-e-exito-o-error)
3. [Builtins: Some, None, Ok, Err](#3-builtins-some-none-ok-err)
4. [Builtins de inspección y unwrap](#4-builtins-de-inspeccion-y-unwrap)
5. [Implicit Some](#5-implicit-some)
6. [Must-handle Result](#6-must-handle-result)
7. [Operador `!!` (unwrap-or-fail)](#7-operador--unwrap-or-fail)
8. [Operador `?` (propagación de error en Result)](#8-operador--propagacion-de-error-en-result)
9. [`nonnull T` y `T !!name`](#9-nonnull-t-y-t-name)
10. [Optional vs nullable (`T?`)](#10-optional-vs-nullable-t)
11. [Layout runtime](#11-layout-runtime)

---

## 1. ``Optional<T>``: valor que puede no existir

`Optional<T>` representa "puede haber valor de tipo T, o puede no haberlo". Es
una alternativa segura a punteros nullable cuando no necesitas indirection.

```vx
Optional<i32> find_index(string[] arr, string target) {
    for (i32 i = 0; i < arr.length; i = i + 1) {
        if (arr[i] == target) return Some(i);
    }
    return None();
}

Optional<i32> result = find_index(names, "Alice");
if (isPresent(result)) {
    i32 idx = unwrap(result);
    println("Found at ${idx}");
} else {
    println("Not found");
}
```

**T puede ser cualquier tipo**: primitivo, puntero, struct, clase. Lo que ocupa
**depende de T** — ver «Layout en memoria» al final: dieciséis con un escalar,
más si envuelve un struct por valor, y **ocho** cuando T no puede valer cero.

---

## 2. Result<V, E>: éxito o error

`Result<V, E>` representa "operación que puede devolver un valor V o un error E".
Es la manera estándar de modelar errores sin excepciones implícitas.

```vx
Result<i32, i64> divide(i32 a, i32 b) {
    if (b == 0) return Err(1); // E = código de error
    return Ok(a / b);
}

Result<i32, i64> r = divide(10, 0);
if (isOk(r)) {
    i32 v = value(r);
    println("Quotient: ${v}");
} else {
    i64 e = error(r);
    println("Error code: ${e}");
}
```

**E puede ser**: primitivo (i32/i64 código de error), puntero (a string descripción),
struct (info del error), clase. Lo que ocupa **depende de V y de E**: veinticuatro
cuando los dos son escalares, más cuando alguno es un struct por valor.

---

## 3. Builtins: Some, None, Ok, Err

Constructores. Cero alocaciones en heap: escriben el slot stack del caller via SRET.

| Builtin | Retorno | Significado |
| :-------- | :-------------- | :--------------------------- |
| `Some(v)` | `Optional<T>` | Optional con valor v |
| `None()` | `Optional<T>` | Optional sin valor |
| `Ok(v)` | `Result<V, E>` | Result éxito con valor V |
| `Err(e)` | `Result<V, E>` | Result error con valor E |

```vx
Optional<i32> a = Some(42);
Optional<i32> b = None();
Result<i32, string> c = Ok(100);
Result<i32, string> d = Err("file not found");
```

El compilador infiere los tipos genéricos `T`/`V`/`E` desde el contexto (tipo
declarado de la variable destino, signature de la función que retorna, etc.).

---

## 4. Builtins de inspección y unwrap

### Optional

| Builtin | Retorno | Descripción |
| :---------------- | :-----: | :------------------------------------------- |
| `isPresent(opt)` | `bool` | true si tiene valor (Some) |
| `unwrap(opt)` | `T` | extrae el valor; **mata el proceso** si no hay |
| `unwrap_or(opt, def)` | `T` | el valor si lo hay, y si no `def`. **No puede fallar** |
| `expect(opt, "msg")` | `T` | como `unwrap`, pero deja dicho qué se dio por hecho |
| `unwrap_unchecked(opt)` | `T` | sin comprobar nada. Leer algo que no está da basura |

### Result

| Builtin | Retorno | Descripción |
| :---------- | :-----: | :----------------------------------------- |
| `isOk(r)` | `bool` | true si es Ok |
| `value(r)` | `V` | extrae valor; FATAL si Err |
| `error(r)` | `E` | extrae error; FATAL si Ok |

```vx
Optional<i32> opt = ...;
if (isPresent(opt)) {
    i32 v = unwrap(opt);
}

Result<i32, i64> r = ...;
if (isOk(r)) {
    process(value(r));
} else {
    log_error(error(r));
}
```

### Fallar una afirmación mata el proceso, y no se captura

`unwrap(x)`, `!!x` y asignar a un `nonnull` son **la misma operación**: afirmar
que hay algo. Si no lo hay, el que se equivocó fue quien lo afirmó — es un bug
del programa, no una condición que el programa se encuentre —, y capturarlo sólo
serviría para seguir corriendo con la suposición ya rota. Es lo que hacen Swift
(trampa) y Rust (pánico); Kotlin lanza porque vive en una máquina donde siempre
hay excepciones.

Y hay una razón más fuerte que la teoría: en nativo **no hay desenrollado de
excepciones**, así que allí siempre fue fatal. Mientras el intérprete lo dejaba
capturar, el mismo programa hacía una cosa al interpretarlo y otra al
compilarlo, que es peor que cualquiera de las dos.

Se sigue imprimiendo el mensaje del catálogo (`VX7001`) y la cadena de llamadas;
lo único que cambia es que ningún `catch` lo intercepta.

```vx
try {
    i32 v = unwrap(vacio);      // el catch NO corre: el proceso muere aqui
} catch (FatalError e) { }
```

### La forma recuperable

Preguntar antes de afirmar. Es la única, y por eso está a mano:

```vx
const i32 v = unwrap_or(opt, 0);            // sin comprobar nada a mano
if (isPresent(opt) != 0) { ... }            // preguntar y decidir
match (opt) { case Some(x) => ..., case None => ... }
const i32 w = parse(s)?;                    // Result: propagar el error
```

`expect(opt, "el limite venia de la configuracion")` sí afirma, pero deja
escrito **qué** se dio por hecho: cuando falla, es lo único que queda. El mensaje
se emite sólo en la rama de que no hay nada, así que el camino bueno no paga, y
tiene que ser una cadena escrita en el sitio (se resuelve al compilar).

---

## 5. Implicit Some

Cuando asignas un valor directo a una variable de tipo `Optional<T>`, el compilador
inserta `Some(...)` automáticamente:

```vx
Optional<i32> a = 50; // = Some(50)
Optional<i32> b = null; // = None() (null literal -> None)

// Equivalente explícito:
Optional<i32> a_explicit = Some(50);
Optional<i32> b_explicit = None();
```

Reduce el ruido visual. Sólo aplica en asignación directa; en argumentos de función
hay que usar el constructor explícito si el tipo del parámetro es `Optional<T>`.

---

## 6. Must-handle Result (`#[must_use]`)

El compilador rechaza en compile-time las `ExprStmt` cuyo `CallExpr` retorna un
`Result<V, E>` y el valor de retorno se descarta:

```vx
Result<i32, i64> divide(i32 a, i32 b) { ... }

i32 main() {
    divide(10, 2); // ERROR de compilación: Result no consumido
 
    Result<i32, i64> r = divide(10, 2); // OK: capturado en variable
    if (isOk(r)) { ... } // OK: inspeccionado
 
    i32 v = value(divide(10, 2)); // OK: extraído inline
    return 0;
}
```

Mismo comportamiento que `#[must_use]` en Rust: forzar al programador a manejar el
caso de error explícitamente. Evita silently ignored errors.

`Optional<T>` NO tiene must-handle (es legítimo descartar opcionales).

---

## 7. Operador `!!` (unwrap-or-fail)

`!!x` **es** `unwrap(x)`: la misma operación, la misma semántica y el mismo
fallo. Sólo cambia dónde se escribe.

```vx
Optional<i32> opt = Some(42);
i32 v = !!opt; // = unwrap(opt) = 42

i32? maybe = nullable_call();
i32 v = !!maybe; // afirma que no es nulo; si lo es, muere
```

Funciona sobre `Optional<T>` y sobre referencias nulables, que son dos formas de
guardar la misma pregunta.

### `nonnull` y `!!` son las dos mitades de una cosa

| | `nonnull T` | `!!x` |
| :-- | :-- | :-- |
| Qué es | un **calificador de tipo** | un **operador de expresión** |
| Dónde va | en una declaración: variable, parámetro, campo | en cualquier expresión |
| Qué dice | «esta ranura nunca guarda nulo» | «este valor de aquí no es nulo» |
| Qué deja detrás | el tipo lleva la promesa a donde vaya | nada: el resultado es `T` a secas |

`nonnull` **enuncia** la promesa y se comprueba en **cada** asignación; `!!` es
el **acto** de comprobarla, y la garantía se gasta ahí. De ahí que escribir `!!`
al asignar a un `nonnull` sea **redundante**: la asignación ya comprueba. Y no
cuesta nada escribirlo, porque la comprobación de más la borra el optimizador —
el resultado de una comprobación es demostrablemente no nulo.

Donde `!!` gana su sitio es donde no hay declaración `nonnull` de por medio:
`f(!!p)`, `*!!q`, `(!!obj).campo`.

### Lo que cuesta

Un `test` y un salto que el predictor acierta siempre. Y a menudo ni eso: el
optimizador borra la comprobación cuando puede demostrar que el valor no es
nulo — la dirección de algo, un objeto recién creado, una constante — y también
usa **análisis de flujo**, así que dentro de un `if (p != null) { ... }` no queda
ninguna.

---

## 8. Operador `?` (propagación de error en Result)

El postfix `?` propaga errores de `Result<V, E>` con early-return. Aplica SOLO a
`Result<V, E>` (no a `Optional<T>`), y sólo dentro de una función que también
retorna un `Result<V, E>` con el mismo tipo de error `E`.

Semántica de `expr?` cuando `expr` es de tipo `Result<V, E>`:

- Si es `Err(e)`, copia el error al buffer de retorno de la función contenedora y
  hace `return` inmediato (propaga el `Err` al caller).
- Si es `Ok(v)`, la expresión evalúa al valor `v` desempaquetado.

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

El compilador valida en compile-time que la expresión sea `Result`, que la función
contenedora retorne `Result`, y que ambos tipos de error `E` coincidan. Cero
overhead runtime frente al patrón `if`-`return` manual (reusa el mismo `LOAD`/`CMP`/
`BR_COND`/`RET`; sin nuevos opcodes).

---

## 9. `nonnull T` y `T !!name`

### `nonnull T` (keyword)

Marca una referencia como NO nullable. El compilador rechaza asignar `null` literal:

```vx
nonnull MyClass obj = new MyClass(); // OK
nonnull MyClass bad = null; // ERROR de compilación
```

Las clases son **nullable por defecto** (modelo legacy compatible). `nonnull` solo
restringe al compile time; no hay coste runtime.

### `T !!name` en var-decl y params

Sintaxis para combinar `nonnull` + auto-unwrap del init:

```vx
// Var-decl: inyecta unwrap automático si maybe es nullable
i32 !!v = maybe_value(); // si maybe_value() retorna null/None, FATAL al entry

// Param: el callee garantiza non-null antes de usar
void process(MyClass !!obj) {
    obj.method(); // sin null-check necesario
}
```

Patrón fail-fast: si esperas no-null pero recibes null, fallas inmediatamente con
mensaje claro en lugar de propagar el null hacia adentro del código.

---

## 10. Optional vs nullable (`T?`)

Vesta tiene DOS modelos para "valor que puede no existir":

| Característica | `Optional<T>` | `T?` (nullable) |
| :------------- | :-------------------------- | :----------------------- |
| Tipo aplicable | cualquier T (val o ref) | sólo referencias (CLASS) |
| Layout | 16 bytes en stack | 8 bytes (puntero) |
| Coste | cero heap | cero heap |
| Inspección | `isPresent(opt)` | `obj == null` |
| Unwrap | `unwrap(opt)` o `!!opt` | `obj!!` o `unwrap(obj)` |
| Implicit Some | sí | no aplicable |
| Construcción | `Some(v)` / `None()` | asignación directa o `null` |

**Cuándo usar cuál**:
- **`Optional<T>`** para tipos value (i32, struct, etc.) donde no quieres heap allocations
 ni indirection.
- **`T?`** para referencias a clases cuando es semánticamente "puede no apuntar a nada".

```vx
Optional<i32> idx = find(arr, x); // valor primitivo, mejor Optional
Person? owner = item.owner; // referencia opcional, mejor nullable
```

---

## 11. Layout runtime

### ``Optional<T>``

```
+---------+---------+
| tag | payload |
| (8 b) | (8 b) |
+---------+---------+
    0=None
    1=Some
```

Dieciséis bytes cuando T es un escalar. **No es un número fijo**: el hueco del
valor es una palabra para un escalar y el tamaño real redondeado a palabra para
un struct por valor, así que un `Optional<StructDeTreintaYDos>` mide cuarenta.

### Cuando la marca sobra: ocho bytes

Si el valor **no puede valer cero**, el cero sobra como valor y sirve de marca
por sí solo: no hace falta la palabra aparte y el conjunto se queda en **ocho**.

Se aplica sólo donde el tipo lo **promete**. Un préstamo (`borrow<T>`) es la
dirección de algo vivo — sólo se obtiene prestando algo que existe —, así que
nunca es cero:

```vx
Optional<borrow<i64>>  ->  8 bytes   (el valor es su propia marca)
Optional<i64>          -> 16 bytes
Optional<i64*>         -> 16 bytes
```

Un `T*` crudo **sí** puede ser nulo, y ahí `Some(nulo)` y «no hay nada» pasarían
a ser el mismo valor: por eso queda fuera. Es la misma idea que la *niche
optimization* de Rust, que por el mismo motivo la aplica a `&T` y no a `*const T`.

Nada de esto se nota desde fuera: construir, preguntar, sacar el valor y el
`match` funcionan igual, porque todos preguntan la disposición en vez de darla
por sabida.

### Result<V, E>

```
+---------+---------+---------+
| tag | value_v | error_e |
| (8 b) | (8 b) | (8 b) |
+---------+---------+---------+
    0=Err
    1=Ok
```

Veinticuatro bytes cuando V y E son escalares. **Tampoco es un número fijo**: la
marca en el cero, el valor detrás y el error detrás del valor, y cada payload
ocupa una palabra si es escalar o su tamaño real redondeado a palabra si es un
struct por valor. Así que un `Result<StructDeTreintaYDos, i64>` mide cuarenta y
ocho, con el error en el desplazamiento cuarenta.

Los dos payloads se guardan **uno al lado del otro** aunque sólo uno esté vivo
cada vez.

### ABI SRET

Funciones que retornan `Optional<T>` o `Result<V, E>` se transforman internamente:

```vx
// Tu código:
Optional<i32> find(...) { return Some(42); }

// Lo que el compilador emite:
void find(ptr retbuf, ...) {
    *retbuf = { tag=1, payload=42 };
}
```

El caller aloca 16/24 bytes en su stack (`ALLOCA`) y pasa el puntero como primer
argumento hidden. Cero heap, cero leaks, cero overhead vs un retorno normal.

---

Ver también: [[ControlFlow]] (uso de `match` con Optional/Result),
[[Excepciones]] (FatalError lanzado por unwrap-fail),
[[Operadores]] (operador `!!`),
[[TiposDatos]] (modelo de tipos: nullable vs Optional).
