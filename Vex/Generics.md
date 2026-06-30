# Generics en Vex

Vex soporta clases genericas con monomorphizacion en compile-time. Cada instanciacion
`Cls<T>` con un tipo concreto genera una clase independiente con su propio layout,
vtable y nombre mangled (`Cls_T`). No hay boxing, no hay overhead de vtable para el
tipo parametrico.

---

## Sintaxis basica

```java
class Box<T> {
    public T value;

    public Box(T v) {
        this.value = v;
    }

    public T get() {
        return this.value;
    }

    public void set(T v) {
        this.value = v;
    }
}

i32 main(string[] args) {
    Box<i32> b1 = new Box<i32>(42);
    Box<i64> b2 = new Box<i64>(100);

    println("b1 = ${b1.get()}"); // 42
    println("b2 = ${b2.get()}"); // 100
    return 0;
}
```

### Nombre mangled generado

| Instanciacion | Nombre interno |
| :-------------- | :--------------- |
| `Box<i32>` | `Box_i32` |
| `Box<i64>` | `Box_i64` |
| `Box<string>` | `Box_string` |
| `Pair<i32,i64>` | `Pair_i32_i64` |
| `List<Box<i32>>`| `List_Box_i32` |

El mangling es recursivo para tipos genericos anidados.

---

## Clases genericas con herencia

```java
class Contenedor<T> {
    protected T dato;

    public Contenedor(T v) {
        this.dato = v;
    }

    public T obtener() {
        return this.dato;
    }
}

class ContenedorNombrado<T> : Contenedor<T> {
    private string nombre;

    public ContenedorNombrado(T v, string n) {
        super(v);
        this.nombre = n;
    }

    @Override
    public string toString() {
        return "${nombre}: ${this.obtener()}";
    }
}

ContenedorNombrado<i32> c = new ContenedorNombrado<i32>(42, "respuesta");
println(c.toString()); // "respuesta: 42"
```

---

## Multiples parametros de tipo

```java
class Pair<A, B> {
    public A first;
    public B second;

    public Pair(A a, B b) {
        this.first = a;
        this.second = b;
    }

    public string toString() {
        return "(${first}, ${second})";
    }
}

Pair<i32, string> p = new Pair<i32, string>(1, "uno");
println("${p.first}"); // 1
println(p.toString()); // (1, uno)
```

---

## Interfaces genericas

```java
interface Transformable<T, R> {
    R transform(T input);
}

class Duplicador : Transformable<i32, i32> {
    @Override
    public i32 transform(i32 input) {
        return input * 2;
    }
}

class Stringificador : Transformable<i32, string> {
    @Override
    public string transform(i32 input) {
        return "valor:${input}";
    }
}

Duplicador d = new Duplicador();
println("${d.transform(21)}"); // 42
```

---

## Restricciones del MVP

Las restricciones siguientes aplican a la implementacion actual. Se levantan
progresivamente en fases posteriores.

1. **Solo monomorphizacion compile-time**: el codigo fuente del template debe estar
 disponible en el momento de la compilacion. No hay generacion de codigo en runtime
 para clases genericas cuya fuente no este disponible (el fallback `specialize` 0x3A
 esta reservado para fases futuras).

2. **Sin constraints de tipo**: no hay `where T : Comparable` ni `T extends Base`. El
 type checker acepta cualquier tipo concreto como argumento. Errores de tipo en metodos
 que usen operadores no soportados por T se detectan al monomorphizar.

3. **Sin metodos genericos sueltos**: no hay `T max<T>(T a, T b) { ... }` a nivel
 de funcion. Los genericos son solo de clase.

4. **Sin wildcards**: no hay `Box<?>` ni `Box<? extends T>` al estilo Java. Las
 referencias son siempre al tipo concreto instanciado.

---

## Como funciona la monomorphizacion

El frontend Vex realiza la monomorphizacion en tres pasos:

### Paso 1: Detectar templates

Las clases con `type_params` no vacios se marcan como templates y se ignoran en la
generacion de bytecode. Nunca se emite `defclass`/`deffield`/`defmethod` para el
template base.

```java
class Box<T> { ... } // <-- template, se ignora en codegen
```

### Paso 2: Detectar instanciaciones

Un pre-pase del type checker recorre todos los `TypeNode` y `NewExpr` buscando
referencias `Cls<T>`:

```java
Box<i32> b = new Box<i32>(42); // <-- detectado: (Box, i32) -> monomorphizar
```

### Paso 3: Generar clases concretas

Para cada `(template, args)` unico se genera una nueva `ClassDecl` con:

- **Nombre mangled**: `Box_i32`
- **AST clonado**: copia profunda del AST del template con todos los type params
 sustituidos por los args concretos.
- **Registro normal**: la clase concreta sigue el pipeline normal de type checking
 y lowering.

```java
Box<T> + T=i32 --> Box_i32 { i32 value; Box_i32(i32 v); i32 get(); void set(i32 v); }
Box<T> + T=i64 --> Box_i64 { i64 value; Box_i64(i64 v); i64 get(); void set(i64 v); }
```

Cada clase concreta tiene su propia vtable, sus propios `ClassInfo*` y `FieldInfo[]`.

---

## `specialize` (0x3A) — fallback runtime

Cuando la fuente del template NO esta disponible en compile-time (por ejemplo, al cargar
un `.velb` externo que uso generics), el runtime puede instanciar clases genericas
dinamicamente via la instruccion `specialize`:

```java
specialize r_dst, r_class, r_types, count
```

- `r_class`: `ClassInfo*` del template base (debe haber sido compilado con `CLASS_FLAG_GENERIC`)
- `r_types`: array de `ClassInfo*` con los argumentos de tipo
- `count`: numero de argumentos de tipo (0..15)
- `r_dst`: `ClassInfo*` de la instanciacion (cacheada en el Loader para reutilizar)

Este mecanismo NO esta disponible desde la sintaxis Vex actual (Phase A). Su uso directo
requiere assembly `.vel` o bytecode manual.

---

## Ejemplo avanzado: pila generica

```java
class Stack<T> {
    private T[] data;
    private i32 size;

    public Stack(i32 cap) {
        this.data = malloc<T>(cap);
        this.size = 0;
    }

    public void push(T v) {
        this.data[this.size] = v;
        this.size += 1;
    }

    public T pop() {
        this.size -= 1;
        return this.data[this.size];
    }

    public i32 length() {
        return this.size;
    }
}

Stack<i32> s = new Stack<i32>(16);
s.push(10);
s.push(20);
s.push(30);

println("size = ${s.length()}"); // 3
println("pop = ${s.pop()}"); // 30
println("pop = ${s.pop()}"); // 20
println("size = ${s.length()}"); // 1
```

---

## Anidamiento profundo de generics

Vex acepta cualquier nivel de anidamiento sin limites artificiales.  Cada
combinacion (`Box<i32>`, `Box<i64>`, `Box<Box<i32>>`, ...) genera una clase
monomorphizada con vtable y layout propios al momento de la compilacion.
La sintaxis usa `>>` que el parser separa internamente como dos `>`
consecutivos para evitar ambiguedad con shift right.

```java
class Box<T> {
    public T value;
    public Box(T v) { this.value = v; }
    public T get() { return this.value; }
}

class Pair<A, B> {
    public A first;
    public B second;
    public Pair(A a, B b) { this.first = a; this.second = b; }
}

i32 main() {
    // 1 nivel: dos instanciaciones distintas coexisten.
    Box<i32> b_i32 = new Box<i32>(10);
    Box<i64> b_i64 = new Box<i64>(20);

    // 2 niveles: Box anidada.
    Box<Box<i32>> b_outer = new Box<Box<i32>>(new Box<i32>(5));
    Box<i32> b_inner = b_outer.get();

    // 3 niveles: Pair con uno de sus type-params siendo otro generic.
    Pair<i32, Box<i64>> p = new Pair<i32, Box<i64>>(7, new Box<i64>(100));

    // Tipos compuestos en getters.
    Box<i64> inner_box = p.get_second();
    return (i32)inner_box.get();
}
```

Cada combinacion produce nombres mangled estables: `Box_i32`, `Box_i64`,
`Box_Box_i32`, `Pair_i32_Box_i64`.  El compilador deduplica por nombre,
asi que multiples menciones de `Box<i32>` en distintas funciones comparten
la misma clase generada.

Ejemplo extenso: `examples_codes_vex/175_generics_deep_nesting.vex` muestra
`Triple<X, Y, Z>` con 3 type-params + `Box<Pair<i32, i64>>` + multiples
niveles combinados.

---

## Metodos genericos `R metodo<U>(...)`

Un metodo puede declarar SUS PROPIOS parametros de tipo, independientes de los
del struct/clase contenedor.  La llamada `obj.metodo<U>(args)` (con `U`
explicito o inferido de los argumentos) genera en compile-time una version
concreta del metodo.  El dispatch es SIEMPRE estatico (como C++/Rust): no
existe un "metodo template virtual".

```java
struct Caja<T> {
    T v;
    // Metodo generico: U es independiente de T.
    i64 mezcla<U>(U x) { return (i64) this.v + (i64) x; }
    // Cast (U)expr funciona con el type-param del metodo.
    U como<U>() { return (U) this.v; }
}

class Contador {
    i64 n;
    Contador(i64 i) { this.n = i; }
    U sumar<U>(U extra) { return (U) this.n + extra; }
}

i32 main() {
    Caja<i32> c = {.v = 30};
    i64 m = c.mezcla<i64>(12);        // explicito  -> 42
    Contador k = new Contador(35);
    i64 s = k.sumar(7);                // inferido i64 -> 42
    return (i32) m;
}
```

Funciona en struct y clase, con type-arg explicito o inferido, varios
type-params (`R m<A, B>(...)`) y en structs genericos (el `U` del metodo es
independiente del `T` del contenedor).  Cero artefacto generico en runtime.

---

## Constraints / conceptos `<T: Concepto>` y `where`

Un CONCEPTO es un predicado COMPTIME sobre un tipo (devuelve bool).  Se usa
como CONSTRAINT (bound) de un type-param: al instanciar el generico, el
compilador evalua el concepto; si no se satisface, error claro.  Las
constraints son 100% compile-time: DESAPARECEN tras el type-check (cero
codigo, cero coste runtime).

### Aplicar bounds

```java
// Inline (1-2 bounds simples):
T maximo<T: Comparable>(T a, T b) { if (a > b) { return a; } return b; }

// Clausula where (varios params o legibilidad):
T sumar<T>(T a, T b) where T: Numeric { return a + b; }

// Multiples conceptos con '+':
T elegir<T>(T a, T b) where T: Numeric + Comparable {
    if (a > b) { return a; } return b;
}
```

Los bounds se admiten en funciones libres, structs, clases, enums y metodos
genericos.

### Conceptos built-in

`Numeric/Integer/Float`, `Signed/Unsigned`, `Bool/Char/String/Pointer`,
`Comparable/Ordered/Eq`, `Sized/Copyable`, `Hashable/Stringable`,
`Default/Callable/Destructible/Iterable/Shareable`, `Primitive/Class/Struct`.

### Conceptos de usuario (tres formas)

```java
// 1. Predicado (sobre la introspeccion comptime):
concept MiNumero<T> = is_numeric<T>();

// 2. Composicion de conceptos:
concept Ordenable<T> = Comparable<T>() && Sized<T>();

// 3. Bloque (logica comptime arbitraria, estilo comptime{}):
concept CabeEnReg<T> {
    if (is_primitive<T>()) { return sizeof<T>() <= 8; }
    return false;
}

// 4. Estructural (exige metodos con FIRMA completa):
concept Dibujable { i64 area(); bool igual(i64 x); }
```

El concepto estructural verifica nombre + aridad + tipo de retorno + tipos de
parametros: un tipo con `bool area()` NO satisface `Dibujable { i64 area(); }`.

Predicados de introspeccion disponibles (ademas de `sizeof`, `field_count`,
`has_method`, ...): `is_integer`, `is_signed`, `is_unsigned`, `is_float`,
`is_numeric`, `is_bool`, `is_char`, `is_pointer`, `is_string`.

---

## Especializacion total + parcial

Tras el template PRIMARIO, se pueden declarar especializaciones para casos
concretos.  Al instanciar `Caja<X>`, se elige la MAS ESPECIFICA que matchee
(exacta > patron > primario) y se clona ESA definicion.  Cada especializacion
puede tener campos y metodos distintos.  Aplica a structs, clases y funciones.

```java
struct Caja<T>   { T v; i64 marca() { return 1; } }          // primario
struct Caja<i64> { i64 v; i64 extra; i64 marca() { return 2; } } // TOTAL exacta
struct Caja<T*>  { T* v; bool propio; i64 marca() { return 3; } } // PARCIAL (puntero)

// Tambien clases y funciones:
class Lista<T>   { ... }
class Lista<i64> { ... }
i64 id<T>(T x)   { return 10; }
i64 id<i64>(i64 x) { return 20; }
i64 id<T*>(T* x) { return 30; }

// Patron generico anidado (T se liga al type-arg interno):
struct Inner<T> { T v; }
struct Bolsa<Inner<T>> { Inner<T> w; ... }
```

Regla de params frescos: un identificador DESNUDO en el patron es un tipo
CONCRETO (total: `Caja<Punto>`); un identificador DENTRO de un patron compuesto
(`T*`, `T[]`, `Inner<T>`) es un param FRESCO (parcial).  La PRIMERA decl con un
nombre es el primario; las siguientes son especializaciones.

---

## Interaccion con typedef / using

Los alias de tipo son transparentes como type-args: `typedef i64 Edad;` hace
que `Caja<Edad>` se comporte exactamente como `Caja<i64>` (mismo metodo
generico, mismo bound, misma especializacion elegida).

## Uso en comptime

Los tipos genericos y especializados son introspectables en comptime:
`sizeof<Caja<i64>>()` refleja la especializacion (sus campos), y los conceptos
(built-in y de usuario) se evaluan como predicados booleanos comptime
(`comptime bool ok = Comparable<i64>();`).

## Genericos cross-module (via imports)

Las plantillas genericas y los conceptos atraviesan el sistema de modulos.  Un
modulo exporta `public struct Caja<T>` / `public i64 fn<T>(...)` /
`public concept C<T> = ...` y otro las instancia importandolas:

```java
// lib.vex
public struct Caja<T> { T v; U id<U>(U x) { return x; } }
public concept Num<T> = is_numeric<T>();
public i64 doblar<T: Num>(T x) { return (i64)x + (i64)x; }

// main.vex
import "lib" only Caja, doblar;       // o:  import "lib";  -> lib.Caja<i64>
i32 main() {
    Caja<i64> c = {.v = 5};
    return (i32)(c.id<i64>(10) + doblar<i64>(11));  // 10 + 22
}
```

Funciona en forma `only Caja` (nombre directo) y namespace `import "lib"`
(`lib.Caja<i64>`), con metodos genericos, bounds/conceptos y especializacion.
Internamente la plantilla viaja como TEXTO FUENTE en el `.vexi`; el importador
la re-parsea y monomorphiza en su propio modulo (las instancias identicas se
deduplican al enlazar).  El cache invalida correctamente: cambiar el cuerpo de
una plantilla recompila los modulos que la instancian.

---

Ejemplos: `examples_codes_vex/222_metodos_genericos.vex` (#4),
`223_conceptos_genericos.vex` + `227_concepts_avanzado.vex` (#6),
`225_especializacion.vex` + `226_especializacion_avanzada.vex` (#7),
`229_typedef_genericos.vex` (typedef).

Ver tambien: [[TiposDatos]], [[OOP]], [[Metaprogramacion]], [[SetInstruccionesVM/GENERIC_SPECIALIZE]]
