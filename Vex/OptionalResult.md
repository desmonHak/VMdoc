# Optional y Result en Vex

`Optional<T>` y `Result<V, E>` son **builtins del compilador** (NO templates):
PrimitiveKinds dedicados con layout fijo en stack del caller (cero heap),
ABI SRET, y sin overhead de GC.

---

## Indice

1. [Optional<T>: valor que puede no existir](#1-optionalt-valor-que-puede-no-existir)
2. [Result<V, E>: éxito o error](#2-resultv-e-exito-o-error)
3. [Builtins: Some, None, Ok, Err](#3-builtins-some-none-ok-err)
4. [Builtins de inspección y unwrap](#4-builtins-de-inspeccion-y-unwrap)
5. [Implicit Some](#5-implicit-some)
6. [Must-handle Result (`#[must_use]`)](#6-must-handle-result-must_use)
7. [Operador `!!` (unwrap-or-fail)](#7-operador--unwrap-or-fail)
8. [`nonnull T` y `T !!name`](#8-nonnull-t-y-t-name)
9. [Optional vs nullable (`T?`)](#9-optional-vs-nullable-t)
10. [Layout runtime](#10-layout-runtime)

---

## 1. Optional<T>: valor que puede no existir

`Optional<T>` representa "puede haber valor de tipo T, o puede no haberlo". Es
una alternativa segura a punteros nullable cuando no necesitas indirection.

```vex
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

**T puede ser cualquier tipo**: primitivo, puntero, struct, clase. El layout es
fijo (16 bytes) independiente del tipo de T.

---

## 2. Result<V, E>: éxito o error

`Result<V, E>` representa "operación que puede devolver un valor V o un error E".
Es la manera estándar de modelar errores sin excepciones implícitas.

```vex
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
struct (info del error), clase. Layout fijo (24 bytes).

---

## 3. Builtins: Some, None, Ok, Err

Constructores. Cero alocaciones en heap: escriben el slot stack del caller via SRET.

| Builtin | Retorno | Significado |
| :-------- | :-------------- | :--------------------------- |
| `Some(v)` | `Optional<T>` | Optional con valor v |
| `None()` | `Optional<T>` | Optional sin valor |
| `Ok(v)` | `Result<V, E>` | Result éxito con valor V |
| `Err(e)` | `Result<V, E>` | Result error con valor E |

```vex
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
| `unwrap(opt)` | `T` | extrae valor; FATAL si None |

### Result

| Builtin | Retorno | Descripción |
| :---------- | :-----: | :----------------------------------------- |
| `isOk(r)` | `bool` | true si es Ok |
| `value(r)` | `V` | extrae valor; FATAL si Err |
| `error(r)` | `E` | extrae error; FATAL si Ok |

```vex
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

**FATAL en unwrap**: si haces `unwrap(None())` o `value(Err(...))` el runtime lanza
`FatalError(FATAL_NULL_POINTER)` que es capturable con `try/catch (FatalError e)`.

---

## 5. Implicit Some

Cuando asignas un valor directo a una variable de tipo `Optional<T>`, el compilador
inserta `Some(...)` automáticamente:

```vex
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

```vex
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

Sintaxis postfix azúcar para `unwrap()`:

```vex
Optional<i32> opt = Some(42);
i32 v = !!opt; // = unwrap(opt) = 42

i32? maybe = nullable_call();
i32 v = !!maybe; // unwrap nullable reference (lanza FATAL si null)
```

Funciona sobre:
- `Optional<T>` -> equivalente a `unwrap(opt)`.
- Referencias `T?` (nullable) -> equivalente al opcode `unwrap` runtime (lanza
 `FATAL_NULL_POINTER` si null).

---

## 8. `nonnull T` y `T !!name`

### `nonnull T` (keyword)

Marca una referencia como NO nullable. El compilador rechaza asignar `null` literal:

```vex
nonnull MyClass obj = new MyClass(); // OK
nonnull MyClass bad = null; // ERROR de compilación
```

Las clases son **nullable por defecto** (modelo legacy compatible). `nonnull` solo
restringe al compile time; no hay coste runtime.

### `T !!name` en var-decl y params

Sintaxis para combinar `nonnull` + auto-unwrap del init:

```vex
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

## 9. Optional vs nullable (`T?`)

Vex tiene DOS modelos para "valor que puede no existir":

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

```vex
Optional<i32> idx = find(arr, x); // valor primitivo, mejor Optional
Person? owner = item.owner; // referencia opcional, mejor nullable
```

---

## 10. Layout runtime

### Optional<T>

```
+---------+---------+
| tag | payload |
| (8 b) | (8 b) |
+---------+---------+
    0=None
    1=Some
```

Total: 16 bytes en stack. El payload se promueve a i64 (cualquier T <= 8 bytes
cabe directo; tipos más grandes están restringidos por el ABI actual).

### Result<V, E>

```
+---------+---------+---------+
| tag | value_v | error_e |
| (8 b) | (8 b) | (8 b) |
+---------+---------+---------+
    0=Err
    1=Ok
```

Total: 24 bytes en stack. Tanto V como E se promueven a i64.

### ABI SRET

Funciones que retornan `Optional<T>` o `Result<V, E>` se transforman internamente:

```vex
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
