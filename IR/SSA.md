# IR SSA de VestaVM

Modulos: `include/ir/ssa_ir.h`, `src/ir/ssa_ir.cpp`, `src/ir/ir_emitter.cpp`,
`src/ir/ir_optimizer.cpp`, `src/ir/liveness.cpp`, `src/ir/regalloc.cpp`

---

## Indice

1. [Que es SSA y por que existe](#1-que-es-ssa-y-por-que-existe)
2. [El problema: variables que se reasignan](#2-el-problema-variables-que-se-reasignan)
3. [La solucion: renombrar cada asignacion](#3-la-solucion-renombrar-cada-asignacion)
4. [Bloques basicos y grafo de flujo de control](#4-bloques-basicos-y-grafo-de-flujo-de-control)
5. [La funcion phi: unificar ramas de control](#5-la-funcion-phi-unificar-ramas-de-control)
6. [Convertir codigo real a SSA paso a paso](#6-convertir-codigo-real-a-ssa-paso-a-paso)
7. [Bucles en SSA](#7-bucles-en-ssa)
8. [Invariantes y reglas de SSA](#8-invariantes-y-reglas-de-ssa)
9. [Optimizaciones habilitadas por SSA](#9-optimizaciones-habilitadas-por-ssa)
   - 9.1 Dead Code Elimination
   - 9.2 Copy Propagation
   - 9.3 Constant Folding
   - 9.4 CSE
   - 9.5 TCO (Tail Call Optimization)
   - 9.6 Eliminacion de bloques inalcanzables
10. [Salir de SSA: destruccion de phi nodes](#10-salir-de-ssa-destruccion-de-phi-nodes)
11. [Pipeline de VestaVM](#11-pipeline-de-vestavm)
12. [Sintaxis del formato .ir](#12-sintaxis-del-formato-ir)
13. [Conjunto de instrucciones IR](#13-conjunto-de-instrucciones-ir)
14. [Directivas de fichero](#14-directivas-de-fichero)
15. [Asignacion de registros y spilling](#15-asignacion-de-registros-y-spilling)
16. [Preprocesador](#16-preprocesador)
17. [API de C++](#17-api-de-c)
18. [Ejemplos incluidos](#18-ejemplos-incluidos)
19. [Notas de implementacion](#19-notas-de-implementacion)

---

## 1. Que es SSA y por que existe

**SSA** (Static Single Assignment, "asignacion unica estatica") es una propiedad de
una representacion intermedia (IR) en la que **cada variable se define exactamente
una vez**. El nombre "estatica" significa que esta unicidad es visible en el texto
del programa, no solo en tiempo de ejecucion.

### La analogia del cuaderno de contabilidad

Imagina un cuaderno de contabilidad en el que cada linea registra un dato: cuando
el contador anota un valor nuevo para una cuenta, tacha el anterior y escribe uno
nuevo encima. Esto complica saber cual era el valor de una cuenta en un momento dado.

En SSA el cuaderno funciona de otra manera: nunca se borra nada. Cada asignacion
crea una **linea nueva** con un nombre nuevo. Si quieres saber que valor tenia una
variable en cualquier punto del programa, basta con mirar la unica linea donde se
definio ese nombre.

### Para que sirve

Los compiladores usan SSA como representacion interna porque simplifica enormemente
tres tareas fundamentales:

1. **Analisis de uso-definicion**: en SSA es trivial saber donde se define y donde
   se usa cada valor. Si `x3` solo se define una vez, toda referencia a `x3` se
   refiere a ese mismo valor.

2. **Optimizaciones**: muchas optimizaciones clasicas (DCE, CSE, constant folding...)
   son casi triviales de implementar sobre SSA pero complicadas sobre codigo normal.

3. **Asignacion de registros**: el compilador puede calcular con precision cuando
   un valor "esta vivo" (necesita un registro) y cuando ya no lo esta (el registro
   se puede reutilizar).

---

## 2. El problema: variables que se reasignan

Considera este fragmento de pseudocodigo:

```
x = 1
x = x + 2
x = x * 3
print(x)
```

La variable `x` se reasigna tres veces. Un compilador que analice este codigo
tiene que rastrear todas las asignaciones para saber cual valor llega a `print`.
Esto se complica mucho en presencia de condicionales y bucles.

Otro ejemplo con bifurcacion:

```
x = 1
if condicion:
    x = 10
y = x + 5
```

Cuando llegamos a `y = x + 5`, el valor de `x` puede ser `1` o `10` segun si
se ejecuto el `if`. Un compilador tiene que propagar ambas posibilidades por
todo el codigo que siga. Esto se llama **analisis de flujo de datos** y es
costoso de calcular correctamente para variables que se reasignan muchas veces.

---

## 3. La solucion: renombrar cada asignacion

La idea de SSA es simple: **cada asignacion crea un nombre nuevo**. En lugar de
reasignar `x`, creamos `x1`, `x2`, `x3`... donde cada nombre se asigna solo una vez.

El ejemplo anterior se convierte en:

```
x1 = 1
x2 = x1 + 2
x3 = x2 * 3
print(x3)
```

Ahora es inmediatamente visible que:
- `print` usa `x3`
- `x3` fue calculado a partir de `x2`
- `x2` fue calculado a partir de `x1`
- `x1` es la constante `1`

No hay ambiguedad: cada nombre solo puede tener un valor.

El compilador puede ver de inmediato que `x1` solo se usa para calcular `x2`,
que `x2` solo se usa para calcular `x3`, y que `x3` solo se usa en `print`.
Si eliminasemos `print`, todo el calculo seria **codigo muerto** (dead code).

---

## 4. Bloques basicos y grafo de flujo de control

Antes de explicar como SSA maneja los condicionales y bucles, necesitamos dos
conceptos fundamentales: el **bloque basico** y el **grafo de flujo de control** (CFG).

### Bloque basico

Un **bloque basico** es una secuencia maximal de instrucciones que siempre se
ejecutan en orden, sin bifurcaciones intermedias. Tiene exactamente:
- Una **entrada**: la primera instruccion del bloque (a la que se salta desde fuera).
- Una **salida**: la ultima instruccion del bloque, que es un **terminador**:
  un salto incondicional (`br`), un salto condicional (`br.cond`) o un retorno (`ret`).

```
entry:                     <-- unica entrada del bloque
    x1 = 1
    x2 = x1 + 2
    cmp x2, 10             <-- no es un salto, sigue dentro del bloque
    br.cond cond, si, no   <-- terminador: bifurca a dos bloques diferentes
```

### Grafo de flujo de control (CFG)

El **CFG** es un grafo dirigido donde:
- Los **nodos** son bloques basicos.
- Las **aristas** representan saltos: si el bloque A puede transferir control al
  bloque B, hay una arista A -> B.

Ejemplo para `if (cond) { y = 1; } else { y = 2; } return y`:

```
        entry
          |
    br.cond cond
       /       \
    si_bloque  no_bloque
    y_si = 1   y_no = 2
       \       /
        merge
       return y
```

Los bloques `si_bloque` y `no_bloque` son **predecesores** de `merge`. El bloque
`merge` tiene dos predecesores.

### Terminador

Cada bloque termina en exactamente uno de estos:
- `br bloque` — salto incondicional al bloque destino.
- `br.cond cond, bloque_si, bloque_no` — salto condicional a dos destinos posibles.
- `ret.tipo valor` — retorno de la funcion.

---

## 5. La funcion phi: unificar ramas de control

Ahora viene el punto mas importante de SSA: como manejar los **puntos de union**,
es decir, los bloques que tienen mas de un predecesor.

### El problema de la union

Siguiendo el ejemplo del `if`:

```
entry:
    x1 = 1
    br.cond cond, si_bloque, no_bloque

si_bloque:
    x_si = 10
    br merge

no_bloque:
    x_no = 1
    br merge

merge:
    // aqui, ¿que valor tiene x?
    // puede ser x_si=10 o x_no=1
    y = x + 5
```

En el bloque `merge`, necesitamos referirnos a "el x que venga de donde sea que
llegamos". Pero SSA dice que cada nombre se define una vez. Necesitamos un mecanismo
especial para unificar los dos posibles valores.

### La funcion phi

La **funcion phi** es exactamente ese mecanismo. Se escribe:

```
x_merged = phi.i64 [x_si, si_bloque], [x_no, no_bloque]
```

Se lee: "el valor de `x_merged` es `x_si` si llegamos desde `si_bloque`,
o `x_no` si llegamos desde `no_bloque`".

La phi no es una instruccion ejecutable en el sentido tradicional: no hay una
instruccion de maquina llamada `phi`. Es una notacion para decir "en este punto del
programa, la variable puede tener distintos valores segun el camino de ejecucion
que se siguio". El compilador la transforma en copias de registro cuando genera codigo.

### Ejemplo completo

```ir
@function valor_absoluto(x: i64) -> i64 {
entry:
    zero  = const.i64 0
    // Comparar x con 0
    isneg = cmp.lt.i64 x, zero    // isneg = 1 si x < 0
    br.cond isneg, rama_neg, rama_pos

rama_neg:
    // x es negativo: calcular -x
    neg_x = sub.i64 zero, x       // neg_x = 0 - x
    br merge

rama_pos:
    // x es positivo: usar x tal cual
    br merge

merge:
    // resultado es neg_x si venimos de rama_neg, o x si venimos de rama_pos
    resultado = phi.i64 [neg_x, rama_neg], [x, rama_pos]
    ret.i64 resultado
}
```

El CFG de esta funcion tiene esta forma:

```
      entry
        |
   br.cond isneg
     /          \
 rama_neg    rama_pos
 neg_x=0-x   (usa x)
     \          /
      merge
   resultado = phi [neg_x, neg] [x, pos]
      |
    ret resultado
```

La phi en `merge` resuelve la ambiguedad: despues del if-else, `resultado` tiene
el valor correcto independientemente del camino tomado.

### Propiedad clave de phi

**La phi se evalua "en paralelo"**: todos los phi nodes al inicio de un bloque
leen sus argumentos del estado justo antes del salto, no unos a otros. Esto es
importante cuando los phi nodes intercambian valores entre si (lo veremos en bucles).

---

## 6. Convertir codigo real a SSA paso a paso

Veamos como se transforma codigo ordinario a SSA de forma sistematica.

### Codigo de partida

```
// Pseudocodigo: max de dos enteros
function max(a, b):
    if a > b:
        result = a
    else:
        result = b
    return result
```

### Paso 1: Identificar bloques basicos

```
entry:
    if a > b goto then_block else else_block

then_block:
    result = a
    goto merge

else_block:
    result = b
    goto merge

merge:
    return result
```

### Paso 2: Renombrar asignaciones

Cada asignacion a `result` crea un nombre nuevo. El resultado en `then_block` es
`result_1`, en `else_block` es `result_2`. En `merge`, tenemos que unificar ambos:

```
entry:
    cond = cmp_gt a, b
    br.cond cond, then_block, else_block

then_block:
    result_1 = mov a              // result viene de a
    br merge

else_block:
    result_2 = mov b              // result viene de b
    br merge

merge:
    result_3 = phi [result_1, then_block], [result_2, else_block]
    ret result_3
```

### Paso 3: En formato .ir de VestaVM

```ir
@module ejemplo_max
@function max(a: i64, b: i64) -> i64 {
entry:
    cond = cmp.gt.i64 a, b
    br.cond cond, then_block, else_block

then_block:
    result_1 = mov.i64 a       // en O1, copy_prop elimina este mov
    br merge

else_block:
    result_2 = mov.i64 b
    br merge

merge:
    result_3 = phi.i64 [result_1, then_block], [result_2, else_block]
    ret.i64 result_3
}
```

Con optimizacion O1, el optimizador aplica copy propagation y simplifica esto a:

```ir
@function max(a: i64, b: i64) -> i64 {
entry:
    cond = cmp.gt.i64 a, b
    br.cond cond, then_block, else_block

then_block:
    br merge

else_block:
    br merge

merge:
    result = phi.i64 [a, then_block], [b, else_block]
    ret.i64 result
}
```

Incluso los bloques vacios `then_block` y `else_block` son oportunidades para
que una optimizacion posterior (jump threading) los elimine si no hacen nada.

---

## 7. Bucles en SSA

Los bucles son el caso mas complejo de SSA porque una variable cambia de valor
en cada iteracion. SSA los resuelve con phi nodes cuyo segundo argumento es el
propio resultado del cuerpo del bucle.

### Suma 1 + 2 + ... + n

```
// Pseudocodigo
function sum(n):
    total = 0
    i = 1
    while i <= n:
        total = total + i
        i = i + 1
    return total
```

### Convertir a SSA

Al entrar al bloque del bucle por primera vez, `total` vale `0` e `i` vale `1`.
En iteraciones posteriores, `total` vale el resultado de la iteracion anterior
y `i` es el valor anterior incrementado.

Los phi nodes unen el estado inicial (del bloque `entry`) con el estado actualizado
(del propio cuerpo del bucle):

```ir
@function sum(n: i64) -> i64 {
entry:
    total_init = const.i64 0
    i_init     = const.i64 1
    br loop_check

loop_check:
    /* total_cur = total_init  si venimos de entry (primera iteracion)
       total_cur = total_next  si venimos de loop_body (resto de iteraciones) */
    total_cur = phi.i64 [total_init, entry],     [total_next, loop_body]
    i_cur     = phi.i64 [i_init,     entry],     [i_next,     loop_body]

    // Condicion de salida: i > n
    done = cmp.gt.i64 i_cur, n
    br.cond done, loop_exit, loop_body

loop_body:
    total_next = add.i64 total_cur, i_cur   // total += i
    one        = const.i64 1
    i_next     = add.i64 i_cur, one          // i++
    br loop_check

loop_exit:
    ret.i64 total_cur
}
```

Observa el ciclo en el CFG:

```
entry --> loop_check <--> loop_body
               |
           loop_exit
```

Los phi nodes en `loop_check` tienen un argumento que viene del propio bucle
(`loop_body`). Esto es lo que hace que SSA sea "circular" para los bucles, y
es perfectamente valido: la propiedad "cada valor se define una vez" se mantiene
porque `total_cur` e `i_cur` son nombres diferentes a `total_next` e `i_next`.

### Trazar una ejecucion

Para `sum(3)`:

| Iteracion | `i_cur` | `total_cur` | `total_next` | `i_next` |
| :-------: | :-----: | :---------: | :----------: | :------: |
| 1a        | 1       | 0           | 0+1 = 1      | 2        |
| 2a        | 2       | 1           | 1+2 = 3      | 3        |
| 3a        | 3       | 3           | 3+3 = 6      | 4        |
| salida    | 4 > 3   | 6           | —            | —        |

`sum(3)` retorna `6`. Correcto.

---

## 8. Invariantes y reglas de SSA

### Regla 1: Definicion unica

Cada nombre de valor se asigna en exactamente una instruccion de todo el programa.
No importa cuantas veces se "ejecute" esa instruccion en tiempo de ejecucion:
estaticamente (en el texto del programa) hay solo una asignacion.

**Valido:**
```ir
x = const.i64 5         // x se define aqui, y solo aqui
y = add.i64 x, x        // x se usa dos veces, pero solo hay una definicion
```

**Invalido (viola SSA):**
```ir
x = const.i64 5
x = add.i64 x, 1        // ERROR: x ya estaba definido
```

### Regla 2: Uso antes de definicion

Un valor solo puede usarse si ya ha sido definido (en el sentido del CFG, siguiendo
todos los caminos de ejecucion posibles hacia ese punto).

**Valido:**
```ir
entry:
    a = const.i64 10
    br bloque

bloque:
    b = add.i64 a, 1    // a esta definido en entry, que siempre precede a bloque
```

**Invalido:**
```ir
bloque:
    b = add.i64 a, 1    // ERROR: a no esta definido antes de llegar a bloque
```

### Regla 3: Los phi nodes van primero

Los phi nodes deben ser las primeras instrucciones de su bloque, antes de cualquier
operacion normal. El motivo es conceptual: el phi "recibe" el valor en el momento
del salto, antes de que el bloque empiece a calcular.

```ir
loop_check:
    total = phi.i64 [t0, entry], [t_next, loop_body]   // phi primero
    i     = phi.i64 [i0, entry], [i_next, loop_body]   // phi primero
    done  = cmp.gt.i64 i, n                              // calculo despues
    br.cond done, exit, loop_body
```

### Regla 4: Numero de argumentos del phi

El phi de un bloque B debe tener exactamente un par `[valor, predecesor]` por
cada predecesor de B en el CFG. Si B tiene 3 predecesores, todos sus phi nodes
tienen 3 argumentos.

---

## 9. Optimizaciones habilitadas por SSA

VestaVM implementa un pipeline de optimizaciones SSA activadas segun el nivel `-O`:

| Nivel | Pases activos |
| :---- | :------------ |
| O0    | Ninguno (bajar IR directamente) |
| O1    | DCE + Copy Propagation |
| O2    | O1 + Constant Folding + Unreachable Block Elimination + **TCO** |
| O3    | O2 + CSE |

### Resumen de pases



La propiedad de definicion unica hace que muchas optimizaciones sean casi triviales.
Aqui explicamos las cuatro que implementa VestaVM, con ejemplos.

### 9.1 Dead Code Elimination (DCE) — Eliminacion de codigo muerto

**Idea**: si el resultado de una instruccion nunca se usa, la instruccion no tiene
ningun efecto observable y puede eliminarse.

En SSA esto es especialmente sencillo: como cada valor tiene una unica definicion
y podemos rastrear facilmente cuantas veces se "usa" ese nombre, si el contador de
usos es cero, la instruccion es muerta.

**Ejemplo antes de DCE:**

```ir
@function ejemplo(a: i64, b: i64) -> i64 {
entry:
    resultado_real  = add.i64 a, b      // esto se usa
    muerto_1        = mul.i64 a, b      // nadie usa muerto_1
    muerto_2        = add.i64 muerto_1, 5  // nadie usa muerto_2 tampoco
    ret.i64 resultado_real
}
```

DCE detecta que `muerto_1` no tiene usos → elimina `mul.i64 a, b`.
Despues, `muerto_2` tampoco tiene usos → elimina su instruccion.
El codigo resultante:

```ir
@function ejemplo(a: i64, b: i64) -> i64 {
entry:
    resultado_real = add.i64 a, b
    ret.i64 resultado_real
}
```

**Algoritmo de DCE en SSA:**
1. Para cada valor, contar cuantas veces aparece como operando en otras instrucciones.
2. Las instrucciones cuyo resultado tiene `uso_count == 0` y no tienen efectos de
   lado (no son `store`, `ret`, `call`, ...) pueden eliminarse.
3. Al eliminar una instruccion, decrementar el `uso_count` de sus operandos.
   Si alguno de ellos llega a 0, puede eliminarse tambien (propagacion).
4. Repetir hasta convergencia.

### 9.2 Copy Propagation — Propagacion de copias

**Idea**: si una instruccion solo copia un valor (`x = mov y`), reemplazar todos
los usos de `x` directamente con `y` y eliminar el `mov`.

```ir
// Antes
a   = const.i64 42
tmp = mov.i64 a        // tmp es solo una copia de a
b   = add.i64 tmp, 1   // usa tmp

// Despues de copy propagation
a   = const.i64 42
b   = add.i64 a, 1     // tmp eliminado, usamos a directamente
```

En SSA esto es especialmente potente porque al ver `tmp = mov a`, sabemos con
certeza que `tmp` siempre tiene el mismo valor que `a`, sin importar el flujo
de control (porque `tmp` solo se define en ese unico punto).

La copy propagation funciona tambien con phi nodes que colapsan a un solo valor.
Si un phi tiene todos sus argumentos iguales:

```ir
// Ambas ramas pasan x sin modificar
x2 = phi.i64 [x, bloque_a], [x, bloque_b]  // x2 siempre es x
```

Entonces `x2` puede reemplazarse por `x` en todos sus usos.

### 9.3 Constant Folding — Plegado de constantes

**Idea**: evaluar en tiempo de compilacion operaciones cuyos operandos son
constantes conocidas, sustituyendo la instruccion por su resultado.

```ir
// Antes
a    = const.i64 3
b    = const.i64 4
suma = add.i64 a, b     // 3 + 4 = 7, evaluable en compilacion
ret.i64 suma

// Despues de constant folding
siete = const.i64 7     // calculado por el compilador
ret.i64 siete
```

El plegado de constantes se propaga en cascada con copy propagation y DCE.
Ejemplo mas complejo:

```ir
// Antes
zero  = const.i64 0
x     = add.i64 a, zero   // a + 0 = a

// Constant folding reconoce que x + 0 = x
// Copy propagation: x es una copia de a
// DCE: la instruccion add se elimina
// Resultado: se usa a directamente
```

Por eso en el ejemplo de la demo (`ejemplo_opt_demo.ir`) la linea
`suma2 = add.i64 suma, zero` desaparece completamente con O2.

### 9.4 Common Subexpression Elimination (CSE)

**Idea**: si la misma expresion (mismo opcode, mismos operandos) aparece calculada
dos o mas veces, calcularla solo una vez y reutilizar el resultado.

```ir
// Antes: x * 2 se calcula dos veces
dos   = const.i64 2
x2_a  = mul.i64 x, dos     // primera vez
x2_b  = mul.i64 x, dos     // segunda vez, identica!
tmp   = add.i64 suma, x2_a
result = add.i64 tmp, x2_b

// Despues de CSE: segunda mul eliminada, x2_b = x2_a
dos   = const.i64 2
x2_a  = mul.i64 x, dos
tmp   = add.i64 suma, x2_a
result = add.i64 tmp, x2_a   // reutiliza x2_a
```

En SSA el CSE es facil porque cada valor se define una vez: si dos instrucciones
tienen exactamente el mismo opcode y los mismos operandos (mismos nombres SSA),
producen el mismo resultado. Podemos reemplazar una por un `mov` del resultado
de la otra, y la copy propagation hace el resto.

### 9.5 Tail Call Optimization (TCO) — Eliminacion de marcos de cola

**Idea**: si el resultado de una llamada se retorna inmediatamente sin ningun
calculo intermedio, el marco actual no necesita existir despues de la llamada.
Se puede reutilizar descartandolo antes de saltar.

El pase `ir_pass_tailcall` detecta el patron:
```ir
%r = call.T @f(args...)
ret.T %r
```
y lo convierte en:
```ir
tailcall @f(args...)
```
eliminando el `ret`. Para llamadas `void`:
```ir
call.void @g(args...)
ret.void
// -> tailcall @g(args...)
```

**El pase NO convierte una llamada si el resultado se usa antes de retornar:**
```ir
r  = call.i64 @f(x)
r2 = add.i64 r, 1    // uso intermedio: no es tail
ret.i64 r2
```

**Activacion**: automatica en O2 y superiores via `ir_optimize(O2+)`.
En la CLI: `--ir-opt 2` o `--ir-opt 3`.

**Beneficios**:
- Elimina la creacion de un nuevo `enter`/`leave`.
- Hace posible la recursion mutua y de cola sin desbordamiento de pila.
- Se combina con constant folding y DCE para maximizar la reduccion.

```ir
// Antes (O0):
r = call.i64 @fact_acc(n_dec, new_acc)
ret.i64 r

// Despues (O2):
tailcall @fact_acc(n_dec, new_acc)
```

Ver `examples_ir/ejemplo_tco.ir` para los cinco casos documentados.

### 9.6 Eliminacion de bloques inalcanzables

Un bloque basico es **inalcanzable** si no existe ningun camino en el CFG desde
el bloque de entrada hasta el. Sus instrucciones nunca se ejecutan.

```ir
entry:
    one = const.i64 1
    br  done              // salto INCONDICIONAL: dead_block nunca se alcanza

dead_block:               // nunca se llega aqui
    unused = mul.i64 n, n  // esta instruccion nunca ejecuta
    br done

done:
    result = add.i64 n, one
    ret.i64 result
```

Con O2, `dead_block` y su contenido se eliminan completamente. El CFG queda:

```
entry --> done --> ret
```

---

## 10. Salir de SSA: destruccion de phi nodes

Cuando el compilador quiere generar codigo maquina (o bytecode de VestaVM),
los phi nodes deben desaparecer: no hay instruccion `phi` en ningun ISA real.

### El proceso de destruccion de phi

La tecnica estandar es **insertar copias en los predecesores**:

Para un phi node `v = phi [v1, blk1], [v2, blk2]`:
- Al final de `blk1`, antes del salto, insertar `v = copy(v1)`.
- Al final de `blk2`, antes del salto, insertar `v = copy(v2)`.

El bloque `merge` ya no tiene phi; en su lugar, todos los predecesores colocan
el valor correcto en `v` antes de saltar.

**Antes (SSA):**
```
blk1:            blk2:
  v1 = ...        v2 = ...
  br merge        br merge

merge:
  v = phi [v1, blk1], [v2, blk2]
  ... usa v ...
```

**Despues (sin phi):**
```
blk1:               blk2:
  v1 = ...           v2 = ...
  v = copy v1        v = copy v2
  br merge           br merge

merge:
  ... usa v ...      (phi eliminado)
```

En VestaVM, `copy` se traduce a `mov rV, rV1` o `mov rV, rV2` en bytecode `.vel`.

### El problema del swap: por que las copias deben ser paralelas

Aqui surge un problema sutil. Considera un bucle de Fibonacci donde dos phi
nodes intercambian valores:

```ir
loop:
    a = phi.i64 [a_init, entry], [b_prev, loop]   // a toma el valor anterior de b
    b = phi.i64 [b_init, entry], [a_plus_b, loop] // b toma a+b
```

Al destruir el phi del bloque `loop` (desde su predecesor `loop`), necesitamos:
- `a = copy b_prev`
- `b = copy a_plus_b`

Supongamos que `a` esta en r1, `b` en r2, `b_prev` en r3, `a_plus_b` en r4.
Las copias son:
- `mov r1, r3`  (a = b_prev)
- `mov r2, r4`  (b = a_plus_b)

Esto es correcto porque r3 y r4 son registros diferentes. Pero considera el caso
en que el propio valor de `a` se reutiliza como argumento del phi de `b`:

```ir
loop:
    a_cur = phi.i64 [a0, entry], [b_cur,        loop]  // a_cur = b_cur anterior
    b_cur = phi.i64 [b0, entry], [a_cur_plus_b, loop]  // b_cur = a_cur + b_cur

    a_cur_plus_b = add.i64 a_cur, b_cur
    br loop
```

Supongamos que el allocador asigna `a_cur` -> r1, `b_cur` -> r2.
En el bloque `loop` (predecesor de si mismo), antes del salto, necesitamos:
- `a_cur_new = copy b_cur`   →   `mov r1, r2`
- `b_cur_new = copy a_cur_plus_b`   →   `mov r2, r4`

**Si lo emitimos en orden secuencial:**
```
mov r1, r2   // r1 <- r2 (correcto hasta aqui)
mov r2, r4   // r2 <- r4 (correcto)
```

Esto es correcto en este caso. Pero supongamos un intercambio directo:
`a = phi [b, loop]` y `b = phi [a, loop]`, con `a` en r1 y `b` en r2:

```
// Copias secuenciales (INCORRECTO):
mov r1, r2   // r1 = r2  (bien, a toma el valor de b)
mov r2, r1   // r2 = r1  (MAL: r1 ya tiene el nuevo valor, no el antiguo)
// Resultado: r1 = r2_antiguo, r2 = r2_antiguo  (ambos tienen el valor de b!)
```

El problema es que la segunda copia lee `r1` despues de que ya fue sobreescrito.

**Solucion: copia paralela con registro temporal:**

```
// Copia paralela usando r14 como scratch (CORRECTO):
mov r14, r1   // guardar el valor antiguo de r1 (a) en scratch
mov r1, r2    // r1 = r2  (a toma el valor de b)
mov r2, r14   // r2 = r14 = r1_antiguo  (b toma el valor antiguo de a)
```

Ahora si: r1 = b_antiguo, r2 = a_antiguo. El intercambio es correcto.

### El algoritmo de copia paralela (VestaVM)

El emitter de VestaVM implementa el siguiente algoritmo para destruir phi nodes:

1. **Construir el grafo de copias**: lista de pares `(dst_reg, src_reg)`.
2. **Eliminar copias triviales**: si `dst == src`, no hace falta emitirlas.
3. **Emitir copias "seguras"** (sin conflictos): una copia `a -> b` es segura si
   `b` no es el origen de ninguna otra copia pendiente. Se emiten en cualquier orden.
4. **Detectar ciclos**: los pares restantes forman ciclos (cada destino es fuente
   de otro). Para cada ciclo, romperlo con r14 como temporal:
   ```
   mov r14, inicio_del_ciclo
   mov inicio, siguiente
   mov siguiente, siguiente_siguiente
   ...
   mov penultimo, r14   // restaurar desde el temporal
   ```

Este algoritmo garantiza que todas las copias son correctas independientemente
del numero de phi nodes y de sus interdependencias.

---

## 11. Pipeline de VestaVM

```
fichero .ir
    |
    v  [si VESTA_HAS_PREPROCESSOR]
vpp::Preprocessor
    |
    v
ir_parse()         -- texto .ir  ->  IrModule en memoria
    |
    v
ir_optimize()      -- O0: sin cambios
    |                  O1: DCE + copy propagation
    |                  O2: O1 + constant folding + unreachable block elimination + TCO
    |                  O3: O2 + CSE
    v
ir_emit_module()   -- IrModule  ->  texto .vel
    |                  [liveness analysis + linear scan register allocation]
    |                  [phi destruction con copia paralela]
    |                  [emit_debug: "// @line N" por instruccion si opts.emit_debug]
    v
Ensamblador .vel   -- texto .vel  ->  .velb bytecode ejecutable
```

Desde la CLI:

```bash
# Compilar .ir a .velb con nivel de optimizacion 2
vm --ir-file examples_ir/ejemplo_add.ir --ir-opt 2 -o build/add

# Solo emitir el .vel generado (sin ensamblar a .velb)
vm --ir-file examples_ir/ejemplo_add.ir --ir-emit-only

# Emitir .vel con numeros de linea de fuente (debug info)
vm --ir-file examples_ir/ejemplo_add.ir --ir-emit-only --emit-debug

# Comparar O0 vs O3
vm --ir-file examples_ir/ejemplo_opt_demo.ir --ir-opt 0 --ir-emit-only
vm --ir-file examples_ir/ejemplo_opt_demo.ir --ir-opt 3 --ir-emit-only

# Comparar con y sin TCO
vm --ir-file examples_ir/ejemplo_tco.ir --ir-opt 0 --ir-emit-only
vm --ir-file examples_ir/ejemplo_tco.ir --ir-opt 2 --ir-emit-only

# Solo ejecutar el preprocesador y mostrar la salida expandida
vm --ir-file examples_ir/ejemplo_preprocessor.ir --preprocess-only
```

### Debug info (emit_debug)

Cuando `EmitOptions::emit_debug = true` (flag `--emit-debug` en CLI), el emitter
inyecta un comentario de linea de fuente antes de cada instruccion que tiene
`IrInstr::source_line > 0`:

```vel
// @line 47
gcderef cur0, r1
addcur  cur0, 24
readcur r2, cur0
// @line 48
...
```

Esto permite depuradores y herramientas de analisis correlacionar instrucciones
`.vel` con lineas del fichero `.ir` original. El campo `source_line` se rellena
por el parser de `.ir` y por el frontend HHL; si vale `0`, no se emite comentario.

`emit_debug` es independiente de `emit_comments` (que emite notas de registro).
Ambas opciones pueden activarse simultaneamente.

---

## 12. Sintaxis del formato .ir

### Comentarios

```ir
// comentario de linea

/* comentario
   de varias lineas */

; comentario de linea (estilo legado, soportado por compatibilidad)
```

### Estructura de un fichero .ir

```ir
// --- Cabecera (opciones globales) ---

// Nombre del modulo (obligatorio si se exportan simbolos)
@module nombre_modulo

// Formato binario de salida (por defecto "velb")
@format velb

// Espacio de direcciones personalizado (opcional)
@space  nombre_espacio 0x0000000000400000 0x00000000007FFFFF

// Seccion dentro del espacio (opcional)
@section code nombre_espacio 0x1000

// Libreria nativa FFI (cargada en tiempo de ejecucion)
@native_lib "stdlib/native/io/vesta_io"

// Importacion de funcion de otro modulo
@import funcion_externa

// --- Funciones ---

@function nombre(param1: tipo, param2: tipo) -> tipo_ret {
bloque_entrada:
    valor = opcode.tipo operando1, operando2
    br.cond cond, bloque_si, bloque_no

bloque_si:
    resultado = ...
    ret.tipo resultado

bloque_no:
    otro_resultado = ...
    ret.tipo otro_resultado
}
```

### Tipos disponibles

| Token    | Descripcion                               |
| :------- | :---------------------------------------- |
| `bool`   | Booleano (0 o 1); alias de `i1`           |
| `i1`     | Entero de 1 bit / booleano                |
| `i8`     | Entero de 8 bits con signo                |
| `u8`     | Entero de 8 bits sin signo                |
| `i16`    | Entero de 16 bits con signo               |
| `u16`    | Entero de 16 bits sin signo               |
| `i32`    | Entero de 32 bits con signo               |
| `u32`    | Entero de 32 bits sin signo               |
| `i64`    | Entero de 64 bits con signo **(comun)**   |
| `u64`    | Entero de 64 bits sin signo               |
| `f32`    | Flotante IEEE 754 de 32 bits              |
| `f64`    | Flotante IEEE 754 de 64 bits              |
| `ptr`    | Puntero opaco de VM (64 bits)             |
| `handle` | GcHandle a objeto GC (32 bits opaco)      |
| `gcref`  | Alias de `handle` (compatibilidad)        |
| `vec128` | Vector de 128 bits                        |
| `vec256` | Vector de 256 bits                        |
| `vec512` | Vector de 512 bits                        |
| `void`   | Sin valor (retorno de procedimientos)     |

### Nombres de valores

Los valores pueden escribirse con o sin prefijo `%`:

```ir
result = add.i64 a, b    // sin prefijo (recomendado)
%r     = add.i64 %a, %b  // con prefijo % (compatible con LLVM-IR)
```

---

## 13. Conjunto de instrucciones IR

### Movimiento y constantes

| Instruccion | Sintaxis                               | Semantica                         |
| :---------- | :------------------------------------- | :-------------------------------- |
| `const`     | `dst = const.tipo literal`             | `dst = literal`                   |
| `mov`       | `dst = mov.tipo src`                   | `dst = src` (copia)               |
| `phi`       | `dst = phi.tipo [v1,blk1], [v2,blk2]` | Selector segun bloque predecesor  |

### Aritmetica entera

| Instruccion | Semantica          | Ejemplo                |
| :---------- | :----------------- | :--------------------- |
| `add`       | `dst = a + b`      | `r = add.i64 x, y`     |
| `sub`       | `dst = a - b`      | `r = sub.i64 x, y`     |
| `mul`       | `dst = a * b`      | `r = mul.i64 x, y`     |
| `div`       | `dst = a / b`      | `r = div.i64 x, y`     |
| `mod`       | `dst = a % b`      | `r = mod.i64 x, y`     |
| `neg`       | `dst = -a`         | `r = neg.i64 x`        |

### Aritmetica flotante

`fadd`, `fsub`, `fmul`, `fdiv`, `fneg` — misma sintaxis con tipo `f32` o `f64`.

```ir
r = fadd.f64 x, y
```

### Operaciones de bits

| Instruccion | Semantica                    |
| :---------- | :--------------------------- |
| `and`       | `dst = a & b`                |
| `or`        | `dst = a \| b`               |
| `xor`       | `dst = a ^ b`                |
| `not`       | `dst = ~a`                   |
| `shl`       | `dst = a << b` (logico)      |
| `shr`       | `dst = a >> b` (logico)      |
| `sar`       | `dst = a >> b` (aritmetico)  |

### Comparacion

La instruccion `cmp` produce un valor `bool`/`i1` (0 o 1).

| Forma          | Condicion                              |
| :------------- | :------------------------------------- |
| `cmp.eq.tipo`  | igual (`a == b`)                       |
| `cmp.ne.tipo`  | distinto (`a != b`)                    |
| `cmp.lt.tipo`  | menor con signo (`a < b`)              |
| `cmp.le.tipo`  | menor o igual con signo (`a <= b`)     |
| `cmp.gt.tipo`  | mayor con signo (`a > b`)              |
| `cmp.ge.tipo`  | mayor o igual con signo (`a >= b`)     |
| `cmp.ult.tipo` | menor sin signo                        |
| `cmp.uge.tipo` | mayor o igual sin signo                |

```ir
cond = cmp.ge.i64 i, n     // cond = (i >= n) ? 1 : 0
br.cond cond, salir, continuar
```

### Conversion de tipos

| Instruccion | Semantica                              |
| :---------- | :------------------------------------- |
| `zext`      | Zero-extend a tipo mas ancho           |
| `sext`      | Sign-extend a tipo mas ancho           |
| `trunc`     | Truncar a tipo mas estrecho            |
| `bitcast`   | Reinterpretar bits sin conversion      |
| `i2f`       | Entero con signo a flotante            |
| `f2i`       | Flotante a entero (truncado hacia 0)   |

### Acceso a memoria VM (raw)

| Instruccion | Sintaxis                  | Semantica                       |
| :---------- | :------------------------ | :------------------------------ |
| `load`      | `dst = load.tipo ptr`     | Leer de memoria VM              |
| `store`     | `store.tipo val, ptr`     | Escribir en memoria VM          |
| `alloca`    | `dst = alloca.tipo n`     | Reservar N palabras en la pila  |

### Flujo de control

| Instruccion  | Sintaxis                              | Semantica                               |
| :----------- | :------------------------------------ | :-------------------------------------- |
| `br`         | `br bloque`                           | Salto incondicional                     |
| `br.cond`    | `br.cond cond, blk_si, blk_no`        | Salto condicional                       |
| `ret`        | `ret.tipo valor`                      | Retorno con valor                       |
| `ret`        | `ret.void`                            | Retorno sin valor                       |
| `call`       | `dst = call.tipo @fn(args...)`        | Llamada directa                         |
| `tailcall`   | `tailcall @fn(args...)`               | Tail call — generado por TCO (O2+)      |
| `unreachable`| `unreachable`                         | Marca codigo imposible de alcanzar      |

---

### OOP — Acceso a campos de objetos GC

Los objetos GC tienen un `ObjectHeader` de 24 bytes seguido de sus campos.
El acceso a campos se realiza por desplazamiento en bytes desde el inicio del objeto.

| Instruccion      | Sintaxis                        | Semantica                                                             |
| :--------------- | :------------------------------ | :-------------------------------------------------------------------- |
| `getfield`       | `dst = getfield.tipo obj, off`  | Leer campo en `obj + off` bytes; emite `gcderef+addcur+readcur`       |
| `setfield`       | `setfield.tipo obj, off, val`   | Escribir campo; si tipo=`handle` emite `gcwb` automaticamente         |
| `gep`            | `gep.ptr obj, off`              | Cursor `cur0` apunta a `obj + off`; usar `readcur`/`writecur` a cont. |
| `gcwb_ir`        | `gcwb_ir obj`                   | Emite `gcwb obj` (write barrier manual); no eliminado por DCE         |
| `gcderef_ir`     | `gcderef_ir obj`                | Emite `gcderef cur0, obj`; `cur0` queda apuntando al host ptr del obj |

**Nota sobre `gep`:** el cursor `cur0` no puede exportarse a un registro general.
Las instrucciones `readcur`/`writecur` deben seguir inmediatamente al `gep` en el
mismo bloque basico; el compilador HHL puede emitir ensamblado crudo con `raw_asm`
para accesos mas elaborados.

**Layout tipico de un objeto:**
```
offset  0..23  ObjectHeader { class_ptr:8, flags:4, hash_code:4, owner_pid:4,
                              lock_depth:2, _pad:2 }
offset 24+     campos propios del tipo
```

Ejemplo:
```ir
// Leer campo i64 en offset 24
x = getfield.i64 punto, 24

// Escribir campo handle en offset 40 (genera gcwb automatico)
setfield.handle punto, 40, s
```

Ver `examples_ir/ejemplo_getfield_setfield.ir` para ejemplos completos.

---

### Arrays

Los arrays residen en memoria VM (no en el heap GC directamente).
Layout: `[u64 length][T data[0]][T data[1]]...`

| Instruccion    | Sintaxis                             | Semantica                                                           |
| :------------- | :----------------------------------- | :------------------------------------------------------------------ |
| `array_alloc`  | `dst = array_alloc.tipo n`           | Asignar array de n elementos via helper nativo `va_alloc`           |
| `array_len`    | `dst = array_len.tipo arr`           | Leer campo `length` (u64 en offset 0)                               |
| `array_load`   | `dst = array_load.tipo arr, idx`     | Leer `arr[idx]`; usa MOVC SIB con stride de `sizeof(tipo)`          |
| `array_store`  | `array_store.tipo arr, idx, val`     | Escribir `arr[idx]`; si tipo=`handle` emite `gcwb` automaticamente  |

**Strides por tipo:**

| Tipo(s)                   | Stride |
| :------------------------ | :----- |
| `i8`, `u8`, `bool`        | 1      |
| `i16`, `u16`              | 2      |
| `i32`, `u32`, `f32`       | 4      |
| `i64`, `u64`, `f64`, `ptr`, `handle` | 8 |

Ejemplo:
```ir
n   = array_len.i64 arr
v   = array_load.i64 arr, idx
array_store.handle arr, idx, h    // gcwb emitido
```

Ver `examples_ir/ejemplo_array_ops.ir` para bucles de suma, maximo y copia.

---

### Strings

Las instrucciones de cadena operan sobre `StringObject` (GcHandle).
Todas retornan `handle` salvo `strlen`/`strcmp` (`i64`), `strhash` (`u64`) y
`strraw` (`ptr`).

| Instruccion    | Sintaxis                          | Semantica                                                      |
| :------------- | :-------------------------------- | :------------------------------------------------------------- |
| `strmake`      | `dst = strmake.handle buf, len, enc` | Crear `StringObject` desde buffer VM con encoding `enc`     |
| `strlen`       | `dst = strlen.i64 s`              | Contar code-points                                             |
| `strcat`       | `dst = strcat.handle a, b`        | Concatenar; retorna `StringObject` de tipo ROPE                |
| `strcmp`       | `dst = strcmp.i64 a, b`           | Comparar lexicograficamente: -1/0/1; actualiza ZF/SF           |
| `strflat`      | `dst = strflat.handle s`          | Materializar ROPE/SLICE a FLAT; identidad si ya es FLAT        |
| `strhash`      | `dst = strhash.u64 s`             | Hash FNV-1a (calculado y cacheado)                             |
| `strintern`    | `dst = strintern.handle s`        | Deduplicar en pool de internamiento; util para comparacion     |
| `strraw`       | `dst = strraw.ptr s`              | Host pointer nulo-terminado (para FFI Win32 `*A`/`*W`)         |
| `strslice`     | `dst = strslice.handle s, rng`    | Vista SLICE sin copia; `rng = (cp_start<<32) \| cp_len`         |
| `strconv`      | `dst = strconv.handle s, enc`     | Convertir encoding; `enc`: 0=ASCII 1=ANSI 2=UTF-8 3=UTF-16 4=UTF-32 |
| `strreserve`   | `dst = strreserve.handle cap`     | FLAT con capacidad `cap` bytes, `byte_len=0` inicial            |
| `strfinalize`  | `strfinalize buf, byte_len`       | Confirmar escritura manual en FLAT (actualiza metadatos)        |
| `strgetenc`    | `dst = strgetenc.handle s`        | Obtener byte de encoding                                       |
| `strgetbytes`  | `dst = strgetbytes.handle s`      | Obtener `byte_len`                                             |
| `strgetkind`   | `dst = strgetkind.handle s`       | Obtener kind (0=FLAT 1=ROPE 2=SLICE)                           |

Ejemplo:
```ir
joined = strcat.handle a, b
flat   = strflat.handle joined     // materializar antes de acceso byte a byte
p      = strraw.ptr flat           // puntero para FFI
```

Ver `examples_ir/ejemplo_string_ops.ir` para los 13 casos documentados.

---

### Excepciones

| Instruccion   | Sintaxis                           | Semantica                                                               |
| :------------ | :--------------------------------- | :---------------------------------------------------------------------- |
| `tryenter`    | `tryenter handler, type`           | Empujar `ExceptionFrame{handler_pc, type}` en la pila de excepciones    |
| `tryleave`    | `tryleave`                         | Pop del `ExceptionFrame` en salida normal del bloque try                |
| `throw`       | `throw exc_obj`                    | Lanzar objeto; `do_throw` busca el frame mas cercano cuyo tipo coincide |
| `landingpad`  | `exc = landingpad.handle`          | Recibir el objeto lanzado en r0 (generado por el emisor)                |
| `unwrap`      | `dst = unwrap src`                 | `dst = src` si no nulo; lanza `NullPointerException` si nulo            |
| `isnull`      | `dst = isnull.bool src`            | `dst = (src == 0) ? 1 : 0`; nunca lanza                                |
| `unreachable` | `unreachable`                      | Marca codigo imposible; permite a DCE eliminar bloques posteriores      |

**Patron try/catch:**
```ir
    catch_addr = const.i64 0     // compilador HHL resuelve la direccion real
    class_info = const.i64 0     // 0 = catch-all
    tryenter catch_addr, class_info

    // ... cuerpo del try ...
    tryleave
    br after_try

catch_block:
    exc = landingpad.handle       // objeto de excepcion en r0
    // ... manejo del error ...
    br after_try

after_try:
    final = phi.i64 [result, entry], [minus_one, catch_block]
    ret.i64 final
```

Ver `examples_ir/ejemplo_excepciones.ir` para los cinco patrones documentados.

---

### Async / Futures

| Instruccion | Sintaxis                  | Semantica                                                                     |
| :---------- | :------------------------ | :---------------------------------------------------------------------------- |
| `future`    | `dst = future`            | Crear `FutureObject` en estado PENDING; `dst = GcHandle`                      |
| `await`     | `dst = await fut`         | Bloquear hasta que `fut` se resuelva; `dst = valor` del future. Re-ejecuta tras wakeup |
| `fulfill`   | `fulfill fut, val`        | Resolver future (RESOLVED) con `val`; despertar al proceso en await si hay    |
| `reject`    | `reject fut, err`         | Rechazar future (REJECTED) con codigo de error; despertar al waiter           |

Ejemplo:
```ir
    fut  = future              // r0 = GcHandle (PENDING)
    // ... en otro proceso: fulfill fut, 42
    result = await fut         // bloquea; cuando resuelto: result = 42
```

---

### Monitores (sincronizacion entre procesos)

| Instruccion | Sintaxis          | Semantica                                                                       |
| :---------- | :---------------- | :------------------------------------------------------------------------------ |
| `monenter`  | `monenter obj`    | Adquirir monitor reentrante del objeto GC; bloquea si otro proceso lo posee     |
| `monexit`   | `monexit obj`     | Liberar monitor (decrementa `lock_depth`); despertar al siguiente en espera     |
| `monwait`   | `monwait obj`     | Liberar completamente el monitor y suspender; re-adquirir tras `monnoti/nota`   |
| `monnoti`   | `monnoti obj`     | Despertar a un proceso en espera del monitor                                    |
| `monnota`   | `monnota obj`     | Despertar a todos los procesos en espera del monitor                            |

---

### Corutinas / Fibras

| Instruccion | Sintaxis              | Semantica                                                                              |
| :---------- | :-------------------- | :------------------------------------------------------------------------------------- |
| `yield`     | `yield`               | Ceder quantum al scheduler; proceso pasa a READY y se reencola                        |
| `resume`    | `resume pid`          | Despertar proceso con PID codificado en `pid` (`sched_id<<32 \| local_pid`)            |
| `spawn`     | `spawn fn_addr`       | Crear proceso en el mismo scheduler; PC = `fn_addr`; r0 = PID codificado del hijo     |
| `swapctx`   | `swapctx dst, src`    | Cambio cooperativo de contexto de fibra: guarda en `src`, carga desde `dst`           |

**`yield` vs `swapctx`:** `yield` es consciente del scheduler (el proceso vuelve
a la cola de listos con otros procesos). `swapctx` es puramente cooperativo entre
dos fibras del mismo `ProcessVM`, sin involucrar al scheduler.

---

### Distribucion

| Instruccion | Sintaxis                    | Semantica                                                                  |
| :---------- | :-------------------------- | :------------------------------------------------------------------------- |
| `rspawn`    | `rspawn node, fn`           | Crear proceso en nodo remoto; r0 = GcHandle de future (0xFFFFFFFF si error)|
| `msgsend`   | `msgsend pid, addr, len`    | Enviar mensaje a proceso (local o remoto); r0 = 1 OK / 0 error             |
| `msgrecv`   | `msgrecv buf, max`          | Recibir de la propia mailbox; r0 = bytes copiados; bloquea si vacia        |
| `memsync`   | `memsync params`            | Sincronizar region de VM a nodo remoto; async si `future_handle != 0`      |

---

### Referencias debiles (GC)

| Instruccion  | Sintaxis               | Semantica                                                             |
| :----------- | :--------------------- | :-------------------------------------------------------------------- |
| `weakref`    | `dst = weakref obj`    | Registrar `obj` en tabla de debiles; `dst = indice opaco`             |
| `deref_weak` | `dst = deref_weak idx` | `dst = GcHandle` si el objeto sigue vivo; `GC_NULL_HANDLE` si fue GC |
| `free_weak`  | `free_weak idx`        | Liberar entrada de la tabla de debiles                                |

---

### Genericos

| Instruccion  | Sintaxis                      | Semantica                                                               |
| :----------- | :---------------------------- | :---------------------------------------------------------------------- |
| `specialize` | `dst = specialize cls, types` | Instanciar clase generica en tiempo de ejecucion; `dst = ClassInfo*`    |

---

### FFI / Ensamblado en linea

| Instruccion | Sintaxis             | Semantica                                                              |
| :---------- | :------------------- | :--------------------------------------------------------------------- |
| `raw_asm`   | `raw_asm "mnemonic"` | Emitir instruccion `.vel` literal; no analizado por el IR              |

Ejemplo:
```ir
raw_asm "mov r3, r4"
raw_asm "readcur r5, cur0"
```

---

### Contexto de proceso

| Instruccion | Sintaxis           | Semantica                            |
| :---------- | :----------------- | :----------------------------------- |
| `getproc`   | `dst = getproc`    | `dst = ProcessVM*` del proceso actual |
| `getvm`     | `dst = getvm`      | `dst = VM*` de la VM propietaria     |
| `getmgr`    | `dst = getmgr`     | `dst = ManageVM*` del gestor global  |

---

## 14. Directivas de fichero

### @module

Define el nombre del modulo. Se emite como `@Module(nombre)` en el `.vel` generado.
Obligatorio si se van a exportar simbolos con `@Export`.

```ir
@module mi_modulo
```

### @format

Especifica el formato binario de salida. Por defecto `velb`.

```ir
@format velb
```

### @space

Define un espacio de direcciones personalizado. Si se omite, el emitter genera
el espacio `anonymous` (0x0 a 0xFFFFFFFFFFFFFFFF).

```ir
// Sintaxis: @space nombre hex_inicio hex_fin
@space user_code 0x0000000000400000 0x00000000007FFFFF
```

### @section

Asocia una seccion al espacio de direcciones e indica la alineacion.
Si se omite, se genera la seccion `code` con alineacion `0x1000`.

```ir
// Sintaxis: @section nombre espacio alineacion
@section code user_code 0x1000
```

### @native_lib

Declara una libreria nativa que el loader buscara y cargara al ejecutar.

```ir
@native_lib "stdlib/native/io/vesta_io"
```

### @import

Declara una funcion importada de otro modulo.

```ir
@import nombre_funcion
```

---

## 15. Asignacion de registros y spilling

### Analisis de liveness (vivacidad)

Antes de asignar registros, el emitter calcula el **rango de vida** de cada valor SSA:
desde la instruccion donde se define hasta la ultima instruccion donde se usa.

Dos valores cuyas vidas se solapan necesitan registros diferentes; dos valores
cuyas vidas no se solapan pueden compartir el mismo registro.

**Ejemplo:**
```ir
entry:
    a = const.i64 5     // a vive desde aqui
    b = const.i64 3     // b vive desde aqui
    c = add.i64 a, b    // a y b se usan aqui -> dejan de vivir
    d = mul.i64 c, c    // c vive desde add hasta aqui
    ret.i64 d           // d vive desde mul hasta aqui
```

Rangos de vida:
- `a`: desde linea 1 hasta linea 3 (usada en `add`)
- `b`: desde linea 2 hasta linea 3 (usada en `add`)
- `c`: desde linea 3 hasta linea 4 (usada en `mul`)
- `d`: desde linea 4 hasta linea 5 (usada en `ret`)

En cada punto hay como maximo 2 valores vivos simultaneamente. Con los registros
r1-r12 disponibles, hay mas que suficiente.

### Linear scan register allocation

El algoritmo de **barrido lineal** de Poletto y Sarkar (1999) es el que usa VestaVM:

1. Ordenar los intervalos de vida por inicio.
2. Mantener un conjunto "activo" de intervalos en vuelo.
3. Para cada nuevo intervalo:
   a. Expirar los intervalos activos cuyo fin ya paso (liberar sus registros).
   b. Si hay un registro libre: asignarlo al nuevo intervalo.
   c. Si no hay registros libres: **spill** (derramar) el intervalo con el mayor
      extremo final (la heuristica "spill the farthest" minimiza spills futuros).
4. Los parametros se pre-asignan a r1, r2, ..., r12 (convencion de llamada de VestaVM).

### Spilling

Cuando el linear scan no tiene registros suficientes (mas de 13 valores vivos
simultaneamente), los valores menos prioritarios se **derraman (spill) a la pila**.

El emitter reserva `N` slots de pila con la instruccion `enter N` al inicio de la
funcion. Cada slot ocupa 8 bytes. El acceso a slots usa `movc` con base `rbp`:

```vel
// Guardar en slot 0:
movc [rbp, r0, 0, 0], r14    // *(rbp + 0*8) = r14

// Cargar desde slot 1:
movc r14, [rbp, r0, 0, 8]    // r14 = *(rbp + 1*8)
```

r14 actua como scratch primario y r13 como scratch secundario cuando hay dos
operandos derramados en la misma instruccion.

### Convencion de llamada

| Uso           | Registro(s)   |
| :------------ | :------------ |
| Parametro 1   | r1            |
| Parametro 2   | r2            |
| ...           | ...           |
| Parametro 12  | r12           |
| Valor de retorno | r0         |
| Scratch primario (spill) | r14 |
| Scratch secundario (spill) | r13 |
| argc en llamadas | r15      |

### Prologo y epilogo generados

```vel
mi_funcion:
    enter 2         // reservar 2 slots de spill (2 * 8 = 16 bytes)
    // parametros: a=r1, b=r2
mi_funcion_entry:
    // ... instrucciones del cuerpo ...
mi_funcion_ret:
    leave
    ret
```

---

## 16. Preprocesador

Cuando el ejecutable se compila con `VESTA_HAS_PREPROCESSOR`, los ficheros `.ir`
pasan automaticamente por `vpp::Preprocessor` antes de ser parseados.

```ir
// Definir una constante usada en el codigo
#define MAX_ITER 100
#define DEBUG_MODE

// Condicional de compilacion
#ifdef DEBUG_MODE
    // Solo incluido en builds de debug
#endif

// Inclusion de ficheros de macros
#include "common_defs.ir"
#import <vel/registers>

// Macro parametrica
#define SWAP(a, b, tmp)  tmp = a; a = b; b = tmp;
```

Ver `doc/VMdoc/Preprocesador/` para la referencia completa del preprocesador.

Para ver la expansion sin compilar:

```bash
vm --ir-file mi_fichero.ir --preprocess-only
```

---

## 17. API de C++

### Construccion programatica

```cpp
#include "ir/ssa_ir.h"
#include "ir/ir_emitter.h"

// Crear un modulo con una funcion suma
ir::IrModule mod;
mod.name = "mi_modulo";

ir::IrFunction fn;
fn.name        = "add";
fn.return_type = ir::IrType::I64;
fn.param_names = {"a", "b"};
fn.param_types = {ir::IrType::I64, ir::IrType::I64};

ir::IrBlock entry;
entry.name = "entry";

// Instruccion: result = add.i64 a, b
ir::IrInstr add_ins;
add_ins.op   = ir::IrOp::ADD;
add_ins.type = ir::IrType::I64;
add_ins.dst  = fn.get_or_create_val("result", ir::IrType::I64);
add_ins.operands = {
    fn.get_or_create_val("a", ir::IrType::I64, true /*is_param*/),
    fn.get_or_create_val("b", ir::IrType::I64, true)
};
entry.instrs.push_back(add_ins);

// Instruccion: ret.i64 result
ir::IrInstr ret_ins;
ret_ins.op      = ir::IrOp::RET;
ret_ins.type    = ir::IrType::I64;
ret_ins.operands = {fn.get_or_create_val("result", ir::IrType::I64)};
entry.instrs.push_back(ret_ins);

fn.blocks.push_back(entry);
mod.add_function(fn);
```

### Serializar / deserializar

```cpp
// IrModule -> texto .ir
std::string text = ir::ir_print(mod);

// Texto .ir -> IrModule
ir::IrModule mod2;
std::string err;
if (!ir::ir_parse(text, mod2, err)) {
    std::cerr << "Error: " << err << "\n";
}

// Verificar propiedad SSA (cada valor definido exactamente una vez)
std::vector<std::string> errors;
if (!ir::ir_verify(mod2, errors)) {
    for (auto &e : errors) std::cerr << e << "\n";
}
```

### Optimizar y emitir a .vel

```cpp
#include "ir/ir_emitter.h"

ir::EmitOptions opts;
opts.opt_level     = ir::OptLevel::O2;   // O0, O1, O2, O3
opts.export_all    = true;
opts.emit_comments = true;               // notas de registro en .vel
opts.emit_debug    = true;               // comentarios "// @line N" por instruccion
opts.module_name   = "mi_modulo";

ir::EmitResult res = ir::ir_emit_module(mod, opts);
if (!res.ok) {
    std::cerr << "Error de emision: " << res.error << "\n";
} else {
    // res.vel_text contiene el codigo .vel listo para el ensamblador
    std::cout << res.vel_text;
}
```

### Construccion de instrucciones avanzadas

```cpp
// GETFIELD: leer campo i64 en offset 24
ir::IrInstr gf;
gf.op       = ir::IrOp::GETFIELD;
gf.type     = ir::IrType::I64;
gf.dst      = fn.get_or_create_val("x", ir::IrType::I64);
gf.operands = { fn.get_or_create_val("punto", ir::IrType::HANDLE) };
gf.imm      = 24;   // offset en bytes
entry.instrs.push_back(gf);

// ARRAY_LOAD: leer elemento i64 del array
ir::IrInstr al;
al.op       = ir::IrOp::ARRAY_LOAD;
al.type     = ir::IrType::I64;
al.dst      = fn.get_or_create_val("v", ir::IrType::I64);
al.operands = { fn.get_or_create_val("arr", ir::IrType::I64),
                fn.get_or_create_val("idx", ir::IrType::I64) };
entry.instrs.push_back(al);

// THROW: lanzar objeto de excepcion
ir::IrInstr th;
th.op       = ir::IrOp::THROW;
th.type     = ir::IrType::VOID;
th.dst      = IR_NO_VALUE;
th.operands = { fn.get_or_create_val("exc_obj", ir::IrType::HANDLE) };
entry.instrs.push_back(th);

// Ejecutar pase TCO manualmente (normalmente lo hace ir_optimize a O2+)
ir::ir_pass_tailcall(fn);
```

---

## 18. Ejemplos incluidos

Los ficheros de ejemplo estan en `examples_ir/`. Se recomienda leerlos en el
orden siguiente para un aprendizaje progresivo:

### Serie basica (SSA, flujo de control, optimizaciones)

| Nivel      | Fichero                         | Conceptos                                             |
| :--------- | :------------------------------ | :---------------------------------------------------- |
| Basico     | `ejemplo_add.ir`                | Funcion minima, parametros, `add.i64`, `ret`          |
| Basico     | `ejemplo_abs.ir`                | `cmp.lt` + `br.cond`, dos bloques de retorno          |
| Basico     | `ejemplo_clamp.ir`              | Anidamiento de condicionales, retorno multiple        |
| Basico     | `ejemplo_max3.ir`               | Diamante con phi de dos ramas                         |
| Bucles     | `ejemplo_sum_range.ir`          | Primer bucle con phi, acumulador simple               |
| Bucles     | `ejemplo_factorial.ir`          | Bucle acumulador con `mul`, caso base early-return    |
| Bucles     | `ejemplo_fibonacci.ir`          | Tres phi nodes, intercambio a<->b (swap paralelo)     |
| Avanzado   | `ejemplo_gcd.ir`                | Euclides iterativo, `mod.i64`                         |
| Avanzado   | `ejemplo_power.ir`              | Exponenciacion rapida, phi en rama condicional        |
| Bitops     | `ejemplo_bitops.ir`             | `and`, `or`, `shl`, `shr`, multiples funciones        |
| Optimiz.   | `ejemplo_opt_demo.ir`           | DCE, const folding, CSE, unreachable block (O0-O3)    |
| Avanzado   | `ejemplo_custom_section.ir`     | `@format`, `@space`, `@section` personalizados        |
| Avanzado   | `ejemplo_preprocessor.ir`       | `#define`, `#ifdef`, macros de preprocesador          |
| Avanzado   | `ejemplo_raw_asm.ir`            | `raw_asm`, ensamblado inline, acceso a cur0           |

### Serie HHL (nuevas instrucciones)

| Nivel      | Fichero                           | Conceptos principales                                              |
| :--------- | :-------------------------------- | :----------------------------------------------------------------- |
| OOP        | `ejemplo_getfield_setfield.ir`    | `getfield`, `setfield`, `gcwb` automatico, layout de objeto        |
| Strings    | `ejemplo_string_ops.ir`           | `strcat`, `strcmp`, `strlen`, `strflat`, `strraw`, `strintern`, ...  |
| Arrays     | `ejemplo_array_ops.ir`            | `array_len`, `array_load`, `array_store`, `gep`, bucles con phi    |
| TCO        | `ejemplo_tco.ir`                  | TCO O0 vs O2, recursion simple, recursion mutua, void tail, no-TCO |
| Excepciones| `ejemplo_excepciones.ir`          | `tryenter`, `tryleave`, `throw`, `landingpad`, `unwrap`, `isnull`  |
| Integral   | `ejemplo_hhl_completo.ir`         | OOP + strings + arrays + excepciones + async (future/await)        |

### Ejercicio recomendado: observar las optimizaciones

```bash
# Sin optimizacion: todo el IR se baja directamente
vm --ir-file examples_ir/ejemplo_opt_demo.ir --ir-opt 0 --ir-emit-only

# Con O3: DCE, copy_prop, const_folding, unreachable, CSE aplicados
vm --ir-file examples_ir/ejemplo_opt_demo.ir --ir-opt 3 --ir-emit-only
```

Con O0 veras instrucciones muertas y calculos redundantes en el `.vel`.
Con O3 el codigo estara limpio y compacto.

### Ejercicio recomendado: ver el efecto del TCO

```bash
# Sin TCO: call + ret presentes
vm --ir-file examples_ir/ejemplo_tco.ir --ir-opt 0 --ir-emit-only

# Con TCO: call + ret -> tailcall
vm --ir-file examples_ir/ejemplo_tco.ir --ir-opt 2 --ir-emit-only
```

### Ejercicio recomendado: debug info

```bash
# .vel con comentarios "// @line N"
vm --ir-file examples_ir/ejemplo_hhl_completo.ir --ir-opt 2 --ir-emit-only --emit-debug
```

---

## 19. Notas de implementacion

### Copia paralela de phi nodes

El emitter (`src/ir/ir_emitter.cpp`, funcion `emit_phi_copies`) implementa el
algoritmo de copia paralela de Briggs/Chaitin para la destruccion de phi nodes.

**Algoritmo:**
1. Construir la lista de pares `(dst_reg, src_reg)` de los phi del bloque sucesor.
2. Separar valores en registro de valores derramados; los derramados se copian
   secuencialmente (sus slots de pila son siempre distintos, no hay alias).
3. Para los valores en registro: emitir en orden topologico las copias "seguras"
   (cuyo destino no es fuente de ninguna otra copia pendiente).
4. Para los ciclos restantes: romper cada ciclo con `r14` como temporal.

Los ciclos (el caso `a, b = b, a+b` de fibonacci) generan:
```vel
mov r14, r3     // guardar a en scratch
mov r3, r4      // a = b
mov r4, r14     // b = a_antiguo
```

### Spilling

Cuando el linear scan necesita derramar valores (mas de 13 vivos simultaneamente):
- `enter N` reserva `N` slots de 8 bytes en la pila (`rbp` apunta a la base).
- `load_src(vid)` emite `movc r14, [rbp, r0, 0, offset]` antes de usar el valor.
- `store_spilled(vid)` emite `movc [rbp, r0, 0, offset], r14` despues de calcularlo.
- `r13` es el scratch secundario cuando dos operandos estan derramados en la misma
  instruccion binaria (evita que ambas cargas sobreescriban `r14`).

### Tipos flotantes

`fadd`, `fsub`, `fmul`, `fdiv`, `fneg`, `fabs`, `fsqrt` se bajan a las instrucciones
`.vel` del mismo nombre. Las conversiones `i2f`/`f2i` se bajan a `fcvt`/`fmowi`.
`floor`/`ceil`/`round` se copian como identidad y emiten un comentario indicando
que deben delegarse a `vesta_math` via `calln`.
