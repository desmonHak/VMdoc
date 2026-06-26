# Vectorizacion automatica (SSE2 / AVX2 / AVX512)

VestaVM auto-vectoriza bucles aritmeticos sobre memoria HOST: el compilador
reconoce el patron del bucle y emite instrucciones SIMD empaquetadas (SSE2,
AVX2 o AVX512) que procesan varios elementos por iteracion, mas un bucle de
cola escalar para el resto.

**No hay anotaciones**: se aplica automaticamente cuando el bucle encaja en uno
de los patrones reconocidos. El mismo mecanismo funciona en el **interprete**
(emulado escalar por lane, como oraculo de correccion), en el **JIT** (SIMD
nativo) y en el **AOT** (binarios nativos con SIMD real).

```vex
// Esto:
for (i64 i = 0; i < n; i = i + 1) { c[i] = a[i] + b[i]; }

// El JIT/AOT lo bajan a (AVX2, 4 f64 por iteracion):
//   vmovupd ymm0, [a+i*8]
//   vaddpd  ymm0, ymm0, [b+i*8]   (conceptual)
//   vmovupd [c+i*8], ymm0
// + un bucle de cola para los N % 4 elementos finales.
```

---

## Patrones reconocidos

| Patron | Forma | Operaciones |
| :----- | :---- | :---------- |
| **Element-wise** | `c[i] = a[i] OP b[i]` | `+ - * /` |
| **Scalar broadcast** | `c[i] = a[i] OP k` (k invariante) | `+ - * /` |
| **Reduccion** | `acc = acc OP a[i]` | `+` (suma) |
| **Dot-product / FMA** | `acc = acc + a[i] * b[i]` | multiplica-acumula |
| **Unario** | `b[i] = OP a[i]` | `-a[i]`, `sqrt(a[i])`, `fabs(a[i])` |
| **Copia (memcpy)** | `c[i] = a[i]` | `rep movsb` / SIMD |

Todos funcionan en su forma `for` **y** `while`:

```vex
i64 i = 0;
while (i < n) { c[i] = a[i] * 2.0; i = i + 1; }   // tambien se vectoriza
```

### Variantes del scalar broadcast

```vex
c[i] = a[i] * k;     // escalar a la derecha (cualquier op)
c[i] = k + a[i];     // escalar a la izquierda (solo conmutativo: + *)
c[i] += k;           // compound assignment
arr[i] *= 3;         // in-place
```

---

## Tipos soportados

| Tipo | Anchos por chunk (SSE2 / AVX2 / AVX512) | Operaciones |
| :--- | :-------------------------------------- | :---------- |
| `f64` | 2 / 4 / 8 | `+ - * /`, neg/abs/sqrt, FMA |
| `f32` | 4 / 8 / 16 | `+ - * /`, FMA |
| `i64` / `u64` | 2 / 4 / 8 | `+ -` |
| `i32` / `u32` | 4 / 8 / 16 | `+ - *` |
| `i16` / `u16` | 8 / 16 / 32 | `+ - *` |
| `i8` / `u8` | 16 / 32 / 64 | `+ -` |

Notas:

- **Multiplicacion entera**: solo `i16`/`u16` (PMULLW) e `i32`/`u32` (PMULLD)
  tienen multiplicacion empaquetada. `i8`/`i64` se multiplican escalar.
- **Division entera**: nunca se vectoriza (no existe division SIMD empaquetada
  de enteros); cae al bucle escalar, pero el resultado es correcto.
- **Cuanto mas estrecho el tipo, mas lanes**: `i16` con AVX2 procesa 16
  elementos por iteracion (vs 4 para `f64`) -> mayores aceleraciones.

---

## Como escribir codigo que se vectorice

### 1. Usa punteros HOST (`malloc` / FFI), no arrays de la VM

La vectorizacion **solo** se aplica sobre memoria HOST: punteros obtenidos de
`malloc`, de FFI, o campos `host_ptr`. Los arrays virtuales de la VM (`T[N]`,
`&local`) **no** se vectorizan (viven en el espacio de direcciones de la VM y
el SIMD nativo opera sobre direcciones host).

```vex
// SI se vectoriza (host)
f64* a = (f64*) malloc(n * 8);
f64* b = (f64*) malloc(n * 8);
f64* c = (f64*) malloc(n * 8);
for (i64 i = 0; i < n; i = i + 1) { c[i] = a[i] + b[i]; }   // SIMD

// NO se vectoriza (array virtual de la VM) -> corre escalar (correcto, lento)
f64 v[256];
for (i64 i = 0; i < 256; i = i + 1) { v[i] = v[i] * 2.0; }
```

### 2. Bucle contador con paso 1 (stride-1)

El indice debe avanzar de 1 en 1 (`i++`, `i = i + 1`, `i += 1`) y los accesos
ser `arr[i]` (sin strides ni indices calculados). Un cuerpo de **una sola
sentencia** vectorizable.

### 3. Reducciones: usa un acumulador LOCAL (register-resident)

En una reduccion o dot-product, el acumulador vectorial vive en un registro
SIMD **a traves del bucle** (sin round-trip a memoria por iteracion) **solo si
el acumulador es una variable local del bucle**. Si compartes el acumulador con
algo externo (lo vuelve address-taken), pierde la residencia en registro.

```vex
// OPTIMO: acc local -> el acumulador vectorial vive en un XMM/YMM
f64 sumar(f64* a, i64 n) {
    f64 acc = 0.0;
    for (i64 i = 0; i < n; i = i + 1) { acc = acc + a[i]; }
    return acc;
}

// Dot-product (FMA): acc local
f64 punto(f64* a, f64* b, i64 n) {
    f64 acc = 0.0;
    for (i64 i = 0; i < n; i = i + 1) { acc = acc + a[i] * b[i]; }
    return acc;
}
```

### 4. Mete el bucle caliente en una funcion

El JIT compila funciones. Un bucle caliente dentro de una funcion hoja se
compila a nativo y se vectoriza; el mismo bucle en `main` mezclado con
`malloc`/`free` puede ejecutarse por el interprete. Encapsula el nucleo
numerico en una funcion:

```vex
void escalar_vec(f64* c, f64* a, f64 k, i64 n) {
    for (i64 i = 0; i < n; i = i + 1) { c[i] = a[i] * k; }
}
```

---

## ISA: SSE2 / AVX2 / AVX512 automatico y portable

El `.velb`/`.exe` generado es **portable**: el ancho SIMD se elige por CPU en
tiempo de ejecucion (auto-cpuid). El compilador hornea un "chunk" (16/32/64
bytes) y el backend lo descompone al ancho del host:

- **SSE2** (baseline, siempre presente en x86-64): 128 bits.
- **AVX2** (si la CPU lo tiene): 256 bits.
- **AVX512** (si la CPU lo tiene): 512 bits.

Un chunk de 64 bytes (AVX512) se ejecuta como 1 operacion ZMM en AVX512, 2 YMM
en AVX2, o 4 XMM en SSE2 -> el mismo binario corre optimo en cualquier x86-64.

### Variables de entorno (control y diagnostico)

| Variable | Efecto |
| :------- | :----- |
| `VESTA_VEC_ISA=sse2\|avx2\|avx512` | Fuerza el ancho del CHUNK horneado en compilacion. |
| `VESTA_JIT_VEC_ISA=sse2\|avx2\|avx512` | Fuerza el ancho de EMISION del JIT/AOT. |
| `VESTA_NO_VECTORIZE=1` | Desactiva la vectorizacion (linea base escalar, util para medir/depurar). |

Por defecto ambos detectan la CPU automaticamente. `VESTA_VEC_ISA=avx512` con
`VESTA_JIT_VEC_ISA=avx2` permite hornear un chunk ancho y emitirlo descompuesto
(verificacion de portabilidad).

---

## Rendimiento

Aceleraciones medidas frente a la version escalar (`VESTA_NO_VECTORIZE=1`),
mismo programa, JIT, ~300M operaciones internas:

| Kernel | SSE2 | AVX2 |
| :----- | :--: | :--: |
| element-wise `c=a+b` (f64) | 3.4x | 3.4x |
| reduccion `acc+=a[i]` (f64) | 6.6x | 8.8x |
| dot-product / FMA (f64) | 4.3x | 4.2x |
| scalar broadcast `a*k` (f64) | 3.8x | 4.6x |
| element-wise `c=a+b` (i32) | 5.9x | 6.9x |
| element-wise `c=a*b` (i32) | 5.9x | 6.4x |
| element-wise `c=a+b` (i16) | 8.4x | **10.3x** |
| scalar broadcast `a*k` (i32) | 5.8x | 8.9x |

Los tipos estrechos (`i16`, `i8`) y las reducciones obtienen las mayores
aceleraciones (mas lanes / menos round-trips). Los element-wise de `f64` son
memory-bound: el cuello de botella es el ancho de banda de memoria, no el
calculo, por eso AVX2 apenas mejora a SSE2 ahi.

---

## Detalles que importan para exprimir el maximo

### Bucle de cola automatico

Si `N` no es multiplo del numero de lanes, los ultimos `N % W` elementos se
procesan en un bucle escalar de cola. Para `N` grandes el coste relativo de la
cola es despreciable. No hace falta alinear `N` a multiplos de nada.

### Difusion del escalar fuera del bucle (hoist)

En el scalar broadcast el valor escalar se difunde a todos los lanes **una sola
vez** antes del bucle (no en cada iteracion). El cuerpo del bucle es VEX puro,
sin penalizacion de transicion AVX/SSE. No tienes que hacer nada: es
automatico.

### Multiplicacion entera estrecha es muy rentable

```vex
// i16 multiply: 16 multiplicaciones por iteracion en AVX2 (PMULLW)
i16* c; i16* a; i16* b;
for (i64 i = 0; i < n; i = i + 1) { c[i] = a[i] * b[i]; }
```

### FMA preserva la semantica IEEE

El dot-product usa una instruccion fusionada multiplica-acumula (un solo
redondeo). El interprete usa la misma fusion (`fma`) para que el resultado
coincida bit-a-bit con el JIT/AOT.

---

## Limitaciones

- Solo punteros HOST (no arrays virtuales de la VM).
- Cuerpo de **una sola** sentencia vectorizable por bucle (cuerpos
  multi-sentencia no se fusionan; se ejecutan escalar).
- Division entera: escalar (sin SIMD entero de division).
- Multiplicacion `i8`/`i64`: escalar (sin packed mul en SSE2 para esos anchos).
- Reduccion: solo suma (`acc += a[i]`); otras reducciones (max/min) son
  candidatas futuras.
- `f32` en scalar broadcast usa el ancho de `f64` (difusion de 64 bits); el
  resto de patrones `f32` usan el doble de lanes que `f64`.

Cuando un bucle no encaja en ningun patron, se ejecuta de forma **escalar
correcta** (nunca produce un resultado incorrecto; simplemente no acelera).

---

## Verificacion

La correccion se valida comparando tres modos sobre el mismo programa:

- **interprete** (`-m vm`): oraculo, ejecuta cada lane escalar.
- **JIT** (`-m jit`): SIMD nativo.
- **escalar** (`VESTA_NO_VECTORIZE=1`): bucle sin vectorizar.

Los tres deben dar el mismo resultado. Para depurar el rendimiento de tu propio
kernel, compara con y sin `VESTA_NO_VECTORIZE=1`.

---

Ver tambien: [[Matematicas]] (funciones numericas), [[TiposDatos]] (punteros
HOST vs arrays virtuales), [[JIT]] y [[AOT]] (donde se materializa el SIMD).
