# Control de flujo en Vex

Vex soporta el set completo de control de flujo estilo C/Java + extensiones modernas
(foreach Python-like, goto/labels, pattern matching `match`).

---

## Indice

1. [Condicionales: if / else / else if](#1-condicionales-if--else--else-if)
2. [Bucles: while / do-while / for / foreach](#2-bucles-while--do-while--for--foreach)
3. [break / continue](#3-break--continue)
4. [goto y labels](#4-goto-y-labels)
5. [Operador ternario](#5-operador-ternario)
6. [Pattern matching: match / case](#6-pattern-matching-match--case)
7. [Operadores cortocircuito: && y ||](#7-operadores-cortocircuito--y-)

---

## 1. Condicionales: if / else / else if

```vex
if (n > 0) {
    println("positivo");
} else if (n < 0) {
    println("negativo");
} else {
    println("cero");
}
```

- Las llaves `{ ... }` son **obligatorias** (no se permite `if (x) statement;`).
- La condicion debe ser `bool` (Vex no convierte implícitamente int -> bool).
- `else if` es azúcar para `else { if ... }`.

**Lowering**: cada branch crea bloques basicos separados; el merge inserta PHIs si las
variables se modifican en ambas ramas.

---

## 2. Bucles: while / do-while / for / foreach

### while

```vex
i32 i = 0;
while (i < 10) {
    println("i = ${i}");
    i = i + 1;
}
```

Comprueba la condición ANTES del body. Si `i < 10` ya es falso al entrar, el body no
se ejecuta.

### do-while

```vex
i32 i = 0;
do {
    println("i = ${i}");
    i = i + 1;
} while (i < 10);
```

Comprueba la condición DESPUÉS del body. Garantiza al menos una iteración. El IR
optimizer detecta este patrón y aplica fusion `cmpjmp` (1 instr VM vs 2).

### for (C-style)

```vex
for (i32 i = 0; i < 10; i = i + 1) {
    println("i = ${i}");
}
```

Equivalente a `{ i32 i = 0; while (i < 10) { ...; i = i + 1; } }`. La variable
declarada en el init solo vive dentro del for.

### foreach (Java-style)

```vex
for (i32 x : array) {
    println("x = ${x}");
}
```

Itera sobre arrays nativos `T[N]` o `T[]`. Sintaxis equivalente Python-style:

```vex
for (x in array) {
    println("x = ${x}");
}
```

Ambas formas son intercambiables; los parentesis son obligatorios.

---

## 3. break / continue

```vex
i32 i = 0;
while (i < 100) {
    if (i == 42) break; // sale del while
    if (i % 2 == 0) {
        i = i + 1;
        continue; // salta al header
    }
    println("impar: ${i}");
    i = i + 1;
}
```

- `break;` sale del loop actual (while/for/do-while/foreach).
- `continue;` salta al header del loop actual sin ejecutar el resto del body.
- Funcionan correctamente con loops anidados (afectan al loop MAS interno).
- `continue` contribuye PHI args al header del loop para variables que se modifican
 en branches (cerrado en A.34).

**Limitación**: no hay `break loop_label;` (labeled break). Usar `goto` para salir
de loops anidados.

---

## 4. goto y labels

```vex
i32 main() {
    i32 i = 0;
    outer:
    while (i < 10) {
        i32 j = 0;
        while (j < 10) {
            if (i * j > 42) goto done;
            j = j + 1;
        }
        i = i + 1;
    }
    done:
    return 0;
}
```

- `goto label;` salta al label declarado (forward o backward).
- Un label es `nombre:` ANTES de cualquier statement.
- Útil para early-exit de loops anidados (reemplaza labeled break) y patrones retry.
- **Forward goto**: el label puede declararse DESPUÉS del goto; el lowering resuelve
 las referencias pendientes cuando encuentra el label.

**Limitaciones documentadas**:
- Backward goto que forma ciclo NO inserta PHI nodes (a diferencia de loops). Usar
 `while`/`for` cuando se necesite back-edge cíclico.
- Goto a label en scope anidado con cleanup actions (destructores RAII, monexit, etc.)
 NO ejecuta los cleanups al saltar.

---

## 5. Operador ternario

```vex
i32 abs = (n < 0) ? -n : n;
string label = (active) ? "ON" : "OFF";
```

- Sintaxis estándar C: `cond ? expr_si_true : expr_si_false`.
- Ambas ramas deben tener tipos compatibles (se aplica coerción si es posible).
- Lowering: crea PHI en el merge (mismo patrón que if/else con asignación).

---

## 6. Pattern matching: match / case

Sintaxis para destructuring de ADTs (`enum`) con bindings y exhaustividad obligatoria.

### Declaración de enum

```vex
enum Color {
    Red, // variante sin payload
    Green(i32), // variante con payload i32
    Blue(i32, i32) // variante con payload (i32, i32)
}
```

### Construcción

```vex
Color c1 = Color.Red;
Color c2 = Color.Green(42);
Color c3 = Color.Blue(10, 20);
```

### Pattern matching

```vex
i32 describe(Color c) {
    match c {
        case Red => return 0;
        case Green(n) => return n;
        case Blue(x, y) => return x + y;
    }
}
```

- **Exhaustividad obligatoria**: el type checker rechaza el match si faltan variantes.
 Usar `case _ => default;` para catch-all si no se quieren enumerar todas.
- Los **bindings** (`n`, `x`, `y`) tienen el tipo del payload de la variante.
- El `return` dentro de un arm sale de la función envolvente.
- Lowering: cadena de `cmp_eq tag + br_cond` (O(N)). Optimización futura:
  uso de `jumptable` (0x27) para N >= 4 variantes.

**Layout runtime** del enum: `[+0 i64 tag][+8 payload[0]][+16 payload[1]]...` en stack.
Tag indica qué variante; los payloads se promueven a i64 para acceso uniforme.

Ver [[TiposDatos]] sección "Enumeraciones" para más detalles del modelo de memoria.

---

## 7. Operadores cortocircuito: && y ||

```vex
if (p != null && p.valid()) {
    p.use();
}

if (cache_hit || lookup_expensive()) {
    do_thing();
}
```

- `&&` evalúa el lado derecho SOLO si el izquierdo es `true`.
- `||` evalúa el lado derecho SOLO si el izquierdo es `false`.
- Ambos producen `bool`.
- Lowering: crea bloques basicos con BR_COND para evitar evaluar el lado derecho
 cuando no es necesario (semántica estándar C/Java/JavaScript).

**Importante**: no confundir con `&` / `|` (AND/OR bitwise sobre enteros, evalúan
SIEMPRE ambos lados).

---

Ver también: [[TiposDatos]] (sistema de tipos), [[OptionalResult]] (`isPresent`/`isOk`
para control de flujo con Optional/Result), [[Closures]] (lambdas como argumentos),
[[Operadores]] (referencia completa de operadores).
