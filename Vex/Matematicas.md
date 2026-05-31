# Funciones matematicas en Vex

Vex incluye un conjunto de **funciones matematicas integradas** que estan
disponibles sin necesidad de importar nada: se llaman como cualquier
funcion normal, son comprobadas por tipos en compile-time y bajan
directamente a la operacion mas eficiente que la plataforma puede
ofrecer.

Este documento describe **todas** las funciones disponibles, agrupadas
por categoria, con dos niveles de explicacion:

- **Que hace** -- explicacion breve para cualquiera.
- **Detalles tecnicos** -- semantica precisa, tipo, casos limite, y
  como esta optimizada por dentro.

---

## Indice

- [1. Vision general](#1-vision-general)
- [2. Aritmetica de coma flotante](#2-aritmetica-de-coma-flotante)
- [3. Funciones trigonometricas y logaritmicas](#3-funciones-trigonometricas-y-logaritmicas)
- [4. Redondeo y truncamiento](#4-redondeo-y-truncamiento)
- [5. Comparaciones (min/max/clamp)](#5-comparaciones-minmaxclamp)
- [6. Valor absoluto](#6-valor-absoluto)
- [7. Operaciones bit a bit](#7-operaciones-bit-a-bit)
- [8. Rotaciones de bits](#8-rotaciones-de-bits)
- [9. Como se optimizan](#9-como-se-optimizan)
- [10. Ejemplos completos](#10-ejemplos-completos)

---

## 1. Vision general

| Funcion | Aridad | Tipo entrada -> salida | Categoria |
| :------ | :----: | :--------------------- | :-------- |
| `sqrt(x)`        | 1 | `f64 -> f64` | Aritmetica FP |
| `pow(x, y)`      | 2 | `f64, f64 -> f64` | Aritmetica FP |
| `fabs(x)`        | 1 | `f64 -> f64` | Aritmetica FP |
| `fmin(a, b)`     | 2 | `f64, f64 -> f64` | Comparacion FP |
| `fmax(a, b)`     | 2 | `f64, f64 -> f64` | Comparacion FP |
| `floor(x)`       | 1 | `f64 -> f64` | Redondeo |
| `ceil(x)`        | 1 | `f64 -> f64` | Redondeo |
| `round(x)`       | 1 | `f64 -> f64` | Redondeo |
| `trunc(x)`       | 1 | `f64 -> f64` | Redondeo |
| `log(x)`         | 1 | `f64 -> f64` | Logaritmo |
| `log2(x)`        | 1 | `f64 -> f64` | Logaritmo |
| `log10(x)`       | 1 | `f64 -> f64` | Logaritmo |
| `sin(x)`         | 1 | `f64 -> f64` | Trigonometrica |
| `cos(x)`         | 1 | `f64 -> f64` | Trigonometrica |
| `tan(x)`         | 1 | `f64 -> f64` | Trigonometrica |
| `abs(x)`         | 1 | `i64 -> i64` | Valor absoluto |
| `imin(a, b)`     | 2 | `i64, i64 -> i64` | Comparacion entera |
| `imax(a, b)`     | 2 | `i64, i64 -> i64` | Comparacion entera |
| `iminu(a, b)`    | 2 | `u64, u64 -> u64` | Comparacion entera |
| `imaxu(a, b)`    | 2 | `u64, u64 -> u64` | Comparacion entera |
| `clamp(x, lo, hi)` | 3 | `i64, i64, i64 -> i64` | Acotar |
| `ilog2(x)`       | 1 | `u64 -> u64` | Log entero |
| `popcount(x)`    | 1 | `u64 -> u64` | Bits |
| `clz(x)`         | 1 | `u64 -> u64` | Bits |
| `ctz(x)`         | 1 | `u64 -> u64` | Bits |
| `bswap(x)`       | 1 | `u64 -> u64` | Bits |
| `rotl(x, n)`     | 2 | `u64, u64 -> u64` | Rotacion |
| `rotr(x, n)`     | 2 | `u64, u64 -> u64` | Rotacion |

**Tipos**: todas las funciones FP aceptan `f64` (numeros con decimales
de doble precision IEEE 754).  Si pasas un `f32`, se promueve
automaticamente.  Las funciones enteras estan tipadas por separado
para signed (`i64`) y unsigned (`u64`).

---

## 2. Aritmetica de coma flotante

### `sqrt(x: f64) -> f64`

**Que hace**: calcula la raiz cuadrada de un numero.

**Tecnico**: equivalente a `sqrt` de la libreria estandar C.  Para
entradas negativas devuelve `NaN`.  Para `0.0` devuelve `0.0`.  Para
infinito positivo devuelve infinito.

```vex
f64 r = sqrt(25.0);   // r = 5.0
f64 d = sqrt(2.0);    // d = 1.4142135...
```

### `pow(x: f64, y: f64) -> f64`

**Que hace**: eleva `x` a la potencia `y`.

**Tecnico**: equivalente a `pow` en C.  `pow(0, 0)` devuelve `1.0` por
convencion IEEE.  Exponentes fraccionarios o negativos son validos:
`pow(8.0, 1.0/3.0)` devuelve la raiz cubica.

```vex
f64 r1 = pow(2.0, 10.0);    // 1024.0
f64 r2 = pow(9.0, 0.5);     // 3.0  (raiz cuadrada)
f64 r3 = pow(2.0, -1.0);    // 0.5  (inverso)
```

### `fabs(x: f64) -> f64`

**Que hace**: valor absoluto de un float (siempre positivo).

**Tecnico**: implementado SIN salto (branchless) limpiando el bit de
signo del IEEE 754.  Mas rapido que un `if (x < 0) return -x`
tradicional porque no rompe la prediccion de saltos.

```vex
f64 a = fabs(-3.14);   // 3.14
f64 b = fabs(7.5);     // 7.5
f64 c = fabs(-0.0);    // 0.0
```

---

## 3. Funciones trigonometricas y logaritmicas

Las funciones `sin`, `cos`, `tan`, `log`, `log2`, `log10` se mapean a la
implementacion de alta precision de la libreria matematica del sistema.

### `sin(x: f64) -> f64`, `cos(x: f64) -> f64`, `tan(x: f64) -> f64`

**Que hace**: las funciones trigonometricas clasicas, con el argumento
en **radianes** (no grados).

**Tecnico**: para convertir grados a radianes multiplica por
`3.14159265358979 / 180.0`.  El periodo se gestiona internamente; no
hay que normalizar el argumento.

```vex
f64 s = sin(0.0);                   // 0.0
f64 c = cos(3.14159);               // ~-1.0
f64 t = tan(0.7854);                // ~1.0 (pi/4 -> 45 grados)
```

### `log(x: f64) -> f64`

**Que hace**: logaritmo natural (base `e` ~= 2.71828).

**Tecnico**: para `x <= 0` devuelve `NaN` (no es error en runtime; se
puede comprobar con un `if`).

### `log2(x: f64) -> f64`, `log10(x: f64) -> f64`

**Que hace**: logaritmo en base 2 y base 10 respectivamente.

```vex
f64 a = log(2.71828);     // ~1.0
f64 b = log2(1024.0);     // 10.0
f64 c = log10(100.0);     // 2.0
```

---

## 4. Redondeo y truncamiento

| Funcion | Politica |
| :------ | :------- |
| `floor(x)` | Hacia menos infinito.  `floor(3.7) = 3.0`, `floor(-3.7) = -4.0` |
| `ceil(x)`  | Hacia mas infinito.  `ceil(3.2) = 4.0`, `ceil(-3.2) = -3.0` |
| `round(x)` | Al entero mas cercano, con desempate al par (banker's rounding) |
| `trunc(x)` | Hacia cero (descarta la parte fraccionaria) |

```vex
f64 a = floor(3.7);    // 3.0
f64 b = ceil(3.2);     // 4.0
f64 c = round(2.5);    // 2.0  (banker's: redondea al par)
f64 d = round(3.5);    // 4.0
f64 e = trunc(-2.9);   // -2.0
```

**Detalle tecnico**: las cuatro se mapean a la instruccion
`roundsd` cuando esta disponible (procesadores con SSE 4.1, lo que
incluye practicamente toda CPU x86-64 producida en los ultimos 15
anos).  Una sola instruccion maquina en menos de 5 ciclos.

---

## 5. Comparaciones (min/max/clamp)

### `fmin(a, b)` / `fmax(a, b)` -- floats

**Que hace**: devuelve el menor o mayor de dos numeros, respetando
las reglas IEEE 754 para `NaN` (si uno es NaN, devuelve el otro).

```vex
f64 lo = fmin(3.5, 7.2);   // 3.5
f64 hi = fmax(3.5, 7.2);   // 7.2
```

### `imin(a, b)` / `imax(a, b)` -- enteros con signo

**Que hace**: como las anteriores pero para `i64` con signo.

```vex
i64 lo = imin(-5, 10);    // -5
i64 hi = imax(-5, 10);    // 10
```

### `iminu(a, b)` / `imaxu(a, b)` -- enteros sin signo

**Que hace**: version unsigned.  Util cuando trabajas con cantidades
que nunca son negativas (tamanos, contadores, hashes, mascaras).

```vex
u64 lo = iminu(0xFFFF, 0x1234);   // 0x1234
u64 hi = imaxu(100, 50);          // 100
```

### `clamp(x, lo, hi)` -- acotar al rango

**Que hace**: si `x < lo`, devuelve `lo`; si `x > hi`, devuelve `hi`;
en cualquier otro caso, devuelve `x` sin cambios.  Util para limitar
un valor a un rango valido (volumen, color, indice).

```vex
i64 vol = clamp(150, 0, 100);   // 100  (recortado)
i64 cnt = clamp(-3, 0, 10);     // 0    (recortado)
i64 ok  = clamp(42, 0, 100);    // 42   (dentro de rango)
```

---

## 6. Valor absoluto

### `abs(x: i64) -> i64`

**Que hace**: valor absoluto de un entero.

**Tecnico**: implementado branchless con tres operaciones
(`xor`/`sub`/`sar`) en lugar de un `if (x < 0) return -x`.  El
comportamiento para `INT_MIN` es indefinido (no representable).

```vex
i64 a = abs(-42);   // 42
i64 b = abs(7);     // 7
i64 c = abs(0);     // 0
```

Para `f64` usa `fabs` (ver seccion 2).

---

## 7. Operaciones bit a bit

Estas funciones tratan al numero como una secuencia de 64 bits.
Cada una corresponde a una **instruccion maquina directa** en los
procesadores modernos, por lo que su coste es de 1 a 3 ciclos.

### `popcount(x: u64) -> u64`

**Que hace**: cuenta cuantos bits estan a 1 en `x` (peso de Hamming).

**Uso tipico**: contar elementos en un bitset, calcular paridad,
medir cardinalidad de un conjunto representado por bits.

```vex
u64 n1 = popcount(0xFF);              // 8   (8 bits a 1)
u64 n2 = popcount(0xF0F0F0F0F0F0F0F0); // 32
u64 n3 = popcount(0);                 // 0
```

### `clz(x: u64) -> u64`

**Que hace**: cuenta cuantos ceros hay a la **izquierda** del primer
bit a 1.  "Count Leading Zeros".

**Uso tipico**: encontrar la posicion del bit mas significativo,
calcular `log2(x)` rapido.

```vex
u64 a = clz(1);              // 63  (todo ceros excepto el bit 0)
u64 b = clz(0x8000000000000000); // 0   (el bit 63 esta a 1)
u64 c = clz(256);            // 55  (bit 8 a 1, 55 ceros antes)
```

> El comportamiento para `clz(0)` no esta definido: el numero no
> tiene ningun bit a 1.  Comprueba antes con `if (x != 0)`.

### `ctz(x: u64) -> u64`

**Que hace**: cuenta cuantos ceros hay a la **derecha** del primer
bit a 1.  "Count Trailing Zeros".

**Uso tipico**: encontrar la posicion del bit menos significativo,
calcular si un numero es multiplo de potencia de 2.

```vex
u64 a = ctz(1);              // 0   (bit 0 esta a 1)
u64 b = ctz(256);            // 8   (bit 8 a 1, 8 ceros antes)
u64 c = ctz(0x80);           // 7
```

> `ctz(0)` no esta definido.  Comprueba antes con `if (x != 0)`.

### `ilog2(x: u64) -> u64`

**Que hace**: logaritmo entero en base 2.  Es decir, la posicion del
bit mas significativo (mismo concepto que `63 - clz(x)`).

**Uso tipico**: calcular el numero de bits necesarios para
representar un valor.

```vex
u64 a = ilog2(1);    // 0
u64 b = ilog2(8);    // 3   (2^3 = 8)
u64 c = ilog2(1023); // 9   (2^9 = 512 <= 1023 < 1024 = 2^10)
u64 d = ilog2(1024); // 10
```

> `ilog2(0)` no esta definido.

### `bswap(x: u64) -> u64`

**Que hace**: invierte el orden de los bytes (endianness swap).  El
byte 0 pasa a la posicion 7, el 1 al 6, etc.

**Uso tipico**: convertir entre little-endian y big-endian para
protocolos de red, formatos binarios, hashes.

```vex
u64 le = 0x0102030405060708;
u64 be = bswap(le);        // 0x0807060504030201
```

---

## 8. Rotaciones de bits

### `rotl(x: u64, n: u64) -> u64`

**Que hace**: rotacion circular a la izquierda.  Los bits que
"salen" por el extremo izquierdo entran por el derecho.  No se
pierden bits (a diferencia de un shift).

### `rotr(x: u64, n: u64) -> u64`

**Que hace**: rotacion circular a la derecha.

**Uso tipico**: implementacion de hashes (SipHash, MurmurHash),
cifrados (ChaCha, Salsa20), generadores pseudoaleatorios (PCG,
xorshift).  La rotacion es la operacion fundamental porque mezcla
bits sin perder informacion.

```vex
u64 x = 0x1;
u64 a = rotl(x, 1);    // 0x2     (bit 0 -> bit 1)
u64 b = rotl(x, 63);   // 0x8000000000000000  (vuelve al bit 63)
u64 c = rotl(x, 64);   // 0x1     (rotacion completa = identidad)

u64 y = 0x8000000000000000;
u64 d = rotr(y, 1);    // 0x4000000000000000
u64 e = rotr(y, 63);   // 0x1     (el bit 63 da la vuelta al 0)
```

> La cuenta `n` se interpreta modulo 64.  `rotl(x, 65)` equivale a
> `rotl(x, 1)`.

---

## 9. Como se optimizan

Vex aplica varias capas de optimizacion sobre las funciones
matematicas.  No tienes que hacer nada para beneficiarte: ocurre
automaticamente al compilar.

### 9.1 Resolucion en compile time

Cuando todos los argumentos son **constantes literales**, la funcion
se evalua durante la compilacion y se reemplaza por el resultado
directo.  No queda ninguna llamada en el codigo final.

```vex
f64 a = sqrt(25.0);          // se compila como  f64 a = 5.0;
f64 b = fmin(3.0, 7.0);      // se compila como  f64 b = 3.0;
u64 c = popcount(0xFF);      // se compila como  u64 c = 8;
u64 d = bswap(0x01020304);   // se compila como  u64 d = 0x0403020100000000;
i64 e = abs(-42);            // se compila como  i64 e = 42;
i64 f = clamp(150, 0, 100);  // se compila como  i64 f = 100;
```

Esto se propaga a traves de cadenas:

```vex
f64 r = sqrt(sqrt(256.0)) + fmin(3.0, 7.0);
// se compila como:  f64 r = 7.0;
```

**Consecuencia practica**: usa libremente las funciones matematicas
en constantes y configuraciones.  Si el compilador puede evaluar la
expresion, lo hara, y el ejecutable final no tendra ningun coste de
runtime para esos calculos.

### 9.2 Instruccion maquina directa

Cuando los argumentos NO son constantes (se conocen solo en runtime),
Vex elige la instruccion del procesador mas eficiente para cada
funcion.

| Funcion | Instruccion x86-64 | Coste tipico |
| :------ | :----------------- | :----------- |
| `sqrt(x)`     | `sqrtsd` | 4-6 ciclos |
| `fabs(x)`     | `andpd` con mascara | 1 ciclo |
| `floor/ceil/round/trunc` | `roundsd` | 3-5 ciclos |
| `fmin/fmax`   | `minsd` / `maxsd` | 3 ciclos |
| `abs(x)`      | branchless `sar`/`xor`/`sub` | 3 ciclos |
| `imin/imax/iminu/imaxu` | `cmp` + `cmov<cc>` | 2 ciclos |
| `popcount(x)` | `popcnt` | 3 ciclos |
| `clz(x)`      | `lzcnt` | 3 ciclos |
| `ctz(x)`      | `tzcnt` | 3 ciclos |
| `bswap(x)`    | `bswap` | 1 ciclo |
| `rotl/rotr`   | `rol cl` / `ror cl` | 1 ciclo |
| `ilog2(x)`    | `lzcnt` + `neg` + `add` | ~5 ciclos |

Comparado con una llamada a libreria tradicional (~30-50
nanosegundos), esto es entre 10 y 50 veces mas rapido.

### 9.3 Cuando no hay instruccion nativa

Las funciones trigonometricas (`sin`, `cos`, `tan`), logaritmicas
(`log`, `log2`, `log10`) y `pow` no tienen una instruccion maquina
unica.  Para estas, Vex llama a la implementacion de la libreria
matematica del sistema, que usa polinomios ajustados a la
arquitectura.  El coste es de 10 a 50 nanosegundos, dependiendo de
la funcion y el procesador.

Si necesitas un caso especifico mas rapido (por ejemplo, `sin` para
angulos siempre en `[0, 2*PI]`), puedes implementar tu propia
aproximacion con polinomios.  Vex no impone su implementacion.

### 9.4 Tipos pequenos se promueven

Si pasas un `f32`, un `i32`, un `u8` o cualquier tipo mas pequeno
que `i64`/`f64`, Vex lo convierte automaticamente al tipo esperado
por la funcion.  La conversion se aplica antes de la llamada, asi
que no hay coste en runtime mas alla de la promocion misma.

```vex
i32 small = 7;
i64 r = imax(small, 10);   // small se promueve a i64 antes
```

---

## 10. Ejemplos completos

### Calcular la distancia entre dos puntos

```vex
f64 dist(f64 x1, f64 y1, f64 x2, f64 y2) {
    f64 dx = x2 - x1;
    f64 dy = y2 - y1;
    return sqrt(dx * dx + dy * dy);
}

i32 main() {
    f64 d = dist(0.0, 0.0, 3.0, 4.0);
    return (i32) d;   // 5
}
```

### Normalizar un valor a un rango

```vex
// Mapea x de [a, b] a [c, d].
f64 normalize(f64 x, f64 a, f64 b, f64 c, f64 d) {
    f64 t = (x - a) / (b - a);
    return c + t * (d - c);
}

// Acotar el resultado al rango [0, 100].
i32 main() {
    f64 r = normalize(75.0, 0.0, 200.0, 0.0, 100.0);  // 37.5
    i64 v = clamp((i64) r, 0, 100);                    // 37
    return (i32) v;
}
```

### Contar bits y rotar (hash simple)

```vex
u64 mini_hash(u64 x) {
    u64 h = x;
    h = h * 0xff51afd7ed558ccd;
    h = h ^ (h >> 33);
    h = rotl(h, 13);
    h = h ^ popcount(h);
    return h;
}

i32 main() {
    u64 h = mini_hash(42);
    return (i32)(h & 0xFFFF);
}
```

### Aproximacion del seno con polinomio (sin libreria)

```vex
// Aproximacion suficiente para muchos juegos / shaders.
// Asume x en el rango [-pi, pi].
f64 fast_sin(f64 x) {
    f64 x2 = x * x;
    f64 x3 = x2 * x;
    f64 x5 = x3 * x2;
    f64 x7 = x5 * x2;
    return x - x3 / 6.0 + x5 / 120.0 - x7 / 5040.0;
}
```

### Logaritmo entero y potencia mas cercana de dos

```vex
u64 next_power_of_two(u64 x) {
    if (x <= 1) return 1;
    return rotl(1, ilog2(x - 1) + 1);
}

i32 main() {
    u64 a = next_power_of_two(5);     // 8
    u64 b = next_power_of_two(1024);  // 1024
    u64 c = next_power_of_two(1025);  // 2048
    return (i32) a;
}
```

### Bitswap para convertir entre orden de bytes

```vex
// Lectura de un entero big-endian desde memoria.
u64 read_be64(u64 little_endian_value) {
    return bswap(little_endian_value);
}
```

---

## Resumen rapido

- **Todas** las funciones estan disponibles globalmente sin imports.
- Cuando los argumentos son **literales**, el resultado se calcula al
  compilar (cero coste en runtime).
- Cuando los argumentos son **variables**, se elige la instruccion
  mas eficiente que el procesador puede ejecutar.
- Las funciones FP usan IEEE 754 estandar (resultados deterministas
  cross-plataforma).
- Las funciones de bits operan sobre 64 bits y tienen comportamiento
  indefinido para entradas `0` en `clz`/`ctz`/`ilog2`.
- Las trigonometricas (`sin`/`cos`/`tan`) reciben radianes.

Si necesitas funciones que no estan en esta lista (por ejemplo
`asin`, `atan2`, `exp`, `expm1`), revisa la libreria
[stdlib/native/math](#) o impone tu propia implementacion: Vex no te
limita a las funciones builtin.
