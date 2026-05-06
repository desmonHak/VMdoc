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

    println("b1 = ${b1.get()}");   // 42
    println("b2 = ${b2.get()}");   // 100
    return 0;
}
```

### Nombre mangled generado

| Instanciacion   | Nombre interno   |
| :-------------- | :--------------- |
| `Box<i32>`      | `Box_i32`        |
| `Box<i64>`      | `Box_i64`        |
| `Box<string>`   | `Box_string`     |
| `Pair<i32,i64>` | `Pair_i32_i64`   |
| `List<Box<i32>>`| `List_Box_i32`   |

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
println(c.toString());   // "respuesta: 42"
```

---

## Multiples parametros de tipo

```java
class Pair<A, B> {
    public A first;
    public B second;

    public Pair(A a, B b) {
        this.first  = a;
        this.second = b;
    }

    public string toString() {
        return "(${first}, ${second})";
    }
}

Pair<i32, string> p = new Pair<i32, string>(1, "uno");
println("${p.first}");   // 1
println(p.toString());   // (1, uno)
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
println("${d.transform(21)}");   // 42
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
class Box<T> { ... }    // <-- template, se ignora en codegen
```

### Paso 2: Detectar instanciaciones

Un pre-pase del type checker recorre todos los `TypeNode` y `NewExpr` buscando
referencias `Cls<T>`:

```java
Box<i32> b = new Box<i32>(42);   // <-- detectado: (Box, i32) -> monomorphizar
```

### Paso 3: Generar clases concretas

Para cada `(template, args)` unico se genera una nueva `ClassDecl` con:

- **Nombre mangled**: `Box_i32`
- **AST clonado**: copia profunda del AST del template con todos los type params
  sustituidos por los args concretos.
- **Registro normal**: la clase concreta sigue el pipeline normal de type checking
  y lowering.

```java
Box<T>  +  T=i32  -->  Box_i32 { i32 value; Box_i32(i32 v); i32 get(); void set(i32 v); }
Box<T>  +  T=i64  -->  Box_i64 { i64 value; Box_i64(i64 v); i64 get(); void set(i64 v); }
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

println("size = ${s.length()}");   // 3
println("pop  = ${s.pop()}");      // 30
println("pop  = ${s.pop()}");      // 20
println("size = ${s.length()}");   // 1
```

---

Ver tambien: [[TiposDatos]], [[OOP]], [[SetInstruccionesVM/GENERIC_SPECIALIZE]]
