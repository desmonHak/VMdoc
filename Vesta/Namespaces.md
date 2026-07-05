# Namespaces

Los **namespaces** dan un nombre logico a los simbolos publicos de Vesta,
desacoplado del fichero en disco. Complementan al
[sistema de modulos](Modulos.md): un modulo es la unidad fisica de compilacion
(un fichero, o un paquete de ficheros); un namespace es el nombre bajo el que
viven sus simbolos y por el que otros modulos los referencian.

Un mismo namespace puede repartirse entre varios ficheros, y varios namespaces
pueden convivir en un fichero. Reorganizar carpetas no cambia el namespace, asi
que el codigo que importa por namespace no se rompe al mover ficheros.

> **Regla practica.** Los namespaces son para **librerias** (codigo
> reutilizable). El fichero de nivel superior de un ejecutable (el que tiene
> `main`) no necesita namespace, aunque puede tenerlo.

---

## 1. Declarar un namespace

Hay dos formas, semanticamente equivalentes. El path es punteado (`a.b.c`) y
puede tener cualquier numero de segmentos.

### 1.1. Forma de sentencia

`namespace a.b.c;` agrupa **todo el resto del fichero** desde la sentencia hasta
el final (o hasta la siguiente sentencia `namespace`).

```vesta
namespace geo.shapes;

public struct Rect { i64 w; i64 h; }
public i64 area(i64 w, i64 h) { return w * h; }
```

### 1.2. Forma de bloque

`namespace a.b.c { ... }` acota con llaves. Permite declarar **varios**
namespaces en el mismo fichero.

```vesta
namespace geo.shapes {
    public struct Rect { i64 w; i64 h; }
    public i64 area(i64 w, i64 h) { return w * h; }
}

namespace geo.color {
    public i64 rgb(i64 r, i64 g, i64 b) {
        return (r << 16) | (g << 8) | b;
    }
}
```

Compila y ejecuta con:

```bash
vm --vx geo.vx -o geo
vm --run geo.velb
```

### 1.3. Llamadas entre hermanos

Dentro de un namespace, los simbolos **hermanos** (declarados en el mismo
namespace) se llaman por su nombre desnudo, sin cualificar. Esto tambien vale
dentro de interpolacion de cadenas `${...}`, lambdas, `match`, `spawn`, `try`,
etc.

```vesta
namespace demo.util;

u64 fact(u64 n) { if (n < 2) { return n; } return n * fact(n - 1); }
i64 doble(i64 x) { return x * 2; }

i64 main() {
    // fact() y doble() son hermanos: nombre desnudo, incluso en ${...}.
    println("fact(5)=${fact(5)} doble(21)=${doble(21)}");
    return doble(21);   // R00 = 42
}
```

---

## 2. Nombre publico y nombre fisico

Cada simbolo tiene dos nombres:

- El **nombre publico**, con el que lo escribes en el codigo:
  `geo.shapes.area`.
- El **nombre fisico** (etiqueta interna del binario), que une los segmentos del
  namespace con `__`: `geo__shapes__area`.

En el codigo Vesta siempre usas el nombre punteado. El nombre fisico solo
importa en la [interoperabilidad de bajo nivel](#12-limitaciones-y-cuando-no-usar-namespace),
donde otro sistema (ensamblador, C) referencia el simbolo por su etiqueta cruda.

---

## 3. Acceso cualificado

Desde fuera del namespace se accede con el path punteado completo. Funciona para
**todo**: funciones, structs, clases, enums, typedefs, globales, `comptime`
const/fn, macros, concepts y genericos.

```vesta
namespace geo;

public class Box {
    public i64 v;
    public Box() { this.v = 0; }
    public i64 get() { return this.v; }
    public void set(i64 x) { this.v = x; }
}

public struct Vec2 { public i64 x; public i64 y; }

public enum Shape { Circle(i64), Square(i64), Point }

public i64 area_hint(geo.Shape s) {
    match (s) {
        case Circle(r) => { return r * 3; }
        case Square(l) => { return l * l; }
        case Point     => { return 0; }
    }
    return -1;
}
```

Un consumidor los usa por su nombre cualificado:

```vesta
import "shapes";   // trae el modulo (y su namespace geo)

i64 main() {
    geo.Box b = new geo.Box();
    b.set(32);                           // clase namespaced

    geo.Shape sq = geo.Shape.Square(3);  // enum namespaced
    i64 a = geo.area_hint(sq);           // 9

    geo.Vec2 v;                          // struct namespaced
    v.x = 1; v.y = 0;

    return b.get() + a + v.x + v.y;      // 32 + 9 + 1 + 0 = R00 42
}
```

---

## 4. Importar por namespace

Ademas del import por-ruta (`import "ruta/fichero";`), puedes importar por
**namespace**. El compilador resuelve el namespace a fichero mediante un indice
que escanea las cabeceras `namespace` de los ficheros `.vx` bajo las *source
roots*: el directorio del fichero raiz de la compilacion, los directorios de la
variable de entorno `VX_PATH`, y la stdlib.

> **Requisito.** Para que `import a.b.c;` funcione, el fichero que declara
> `a.b.c` debe estar accesible bajo alguna source root. En la practica esto
> significa compilar el ejecutable con el fichero raiz situado en (o por debajo
> de) el mismo arbol de directorios que la libreria, o anadir su carpeta a
> `VX_PATH`.

```vesta
// geolib.vx
namespace geo.shapes;
public i64 area(i64 w, i64 h) { return w * h; }
public struct Rect { i64 w; i64 h; }
```

### 4.1. Import simple: acceso cualificado

```vesta
import geo.shapes;

i64 main() {
    i64 a = geo.shapes.area(6, 7);          // cualificado completo
    geo.shapes.Rect r = { .w = 3, .h = 0 }; // tipo cualificado
    return a - r.w;                          // 42 - 0 = R00 42
}
```

El import simple **no** crea un alias corto por el ultimo segmento: escribe
siempre el path completo `geo.shapes.area`, o usa un alias (seccion 4.2) o un
import selectivo (seccion 4.3).

### 4.2. Alias: `as`

Renombra el namespace a un identificador local corto.

```vesta
import geo.shapes as g;

i64 main() {
    return g.area(6, 7);   // via alias -> R00 42
}
```

### 4.3. Import selectivo: `.{ ... }`

Trae simbolos concretos al scope local; se usan **sin cualificar**.

```vesta
import geo.shapes.{ area, Rect };

i64 main() {
    Rect r = { .w = 3, .h = 0 };
    return area(6, 7) - r.w;   // R00 42
}
```

Las tres formas conviven en el mismo fichero sin problema:

```vesta
import geo.shapes;              // cualificado
import geo.shapes as g;         // alias
import geo.shapes.{ area, Rect }; // selectivo

i64 main() {
    i64 a = geo.shapes.area(6, 7);  // 42 (cualificado)
    i64 b = g.area(1, 8);           // 8  (alias)
    Rect r = { .w = 3, .h = 0 };    // (selectivo)
    i64 c = area(1, 1);             // 1  (selectivo)
    return a + b - r.w - c - 4;     // 42 + 8 - 3 - 1 - 4 = R00 42
}
```

### 4.4. Ambiguedad en imports selectivos

Si dos imports selectivos traen simbolos con el **mismo nombre**, la referencia
sin cualificar se resuelve al primero que se importo. Para evitar sorpresas,
cuando dos namespaces exportan nombres iguales prefiere el **acceso cualificado**
(o un alias distinto para cada uno) en lugar del import selectivo.

```vesta
import pa;
import pb;

i64 main() {
    // pa y pb exportan ambos `area`: desambigua cualificando.
    return pa.area(1) + pb.area(1);
}
```

---

## 5. Visibilidad: `public`, `internal`, `private`

La visibilidad es un eje **ortogonal** al namespace: el nombre fisico se manglea
siempre por namespace; la visibilidad es una regla aparte que decide quien puede
ver el simbolo.

| Modificador | Alcance |
| :---------- | :------ |
| `public`    | Exportado del paquete. Visible para cualquier consumidor, incluso de otros paquetes. |
| `internal`  | Visible en **todo el paquete** (mismo *PackageId*), pero **no** exportado a paquetes distintos. Equivale a `internal` de C# / `pub(crate)` de Rust. |
| `private` (por defecto) | Restringido a su fichero. Es el modo por defecto cuando no escribes ningun modificador. |

```vesta
namespace lib;

public   i64 publica(i64 x) { return x + 1; }   // la ven otros paquetes
internal i64 interna(i64 x) { return x + 100; } // solo dentro de este paquete
         i64 privada(i64 x) { return x - 1; }   // solo este fichero
```

### 5.1. `internal` cruza modulos del mismo paquete

Dos ficheros del mismo paquete ven mutuamente sus simbolos `internal`. Si no hay
manifiesto `vx.toml` que fije un *PackageId*, todos los ficheros de la
compilacion comparten un paquete anonimo comun y `internal` es visible entre
ellos:

```vesta
// main.vx  (mismo paquete que lib.vx)
import lib;

i64 main() {
    i64 a = lib.publica(41);   // 42  (public)
    i64 b = lib.interna(1);    // 101 (internal, MISMO paquete -> visible)
    return a + b - 101;        // R00 42
}
```

### 5.2. `internal` esta oculto a otros paquetes

Cuando el consumidor pertenece a un **paquete distinto** (otro *PackageId*), los
simbolos `internal` del dependiente desaparecen: referenciarlos es un error de
compilacion.

```vesta
// dep.vx
namespace pkgdep @id("dep-pkg-v1");
public   i64 publica(i64 x) { return x + 1; }
internal i64 interna(i64 x) { return x + 100; }
```

```vesta
// consumer.vx (paquete distinto)
namespace consumer @id("consumer-pkg-v1");
import pkgdep;

public i64 run() {
    return pkgdep.interna(1);   // error: el namespace 'pkgdep' no tiene
                                //        un simbolo llamado 'interna'
}
```

El mismo consumidor **si** puede usar `pkgdep.publica`.

### 5.3. Alcance real de `private`

`private` (el modo por defecto) restringe un simbolo a su fichero. En la version
actual conviene conocer el matiz exacto:

- Los **datos** (variables globales) `private` **no se exportan**: quedan
  restringidos a su fichero. Referenciar una global privada desde otro modulo es
  un error de compilacion.
- Las **funciones** de primer nivel `private` de un namespace todavia pueden
  alcanzarse por su nombre cualificado desde otros modulos que participen en la
  **misma compilacion**. Por tanto, trata `private` sobre funciones como una
  **declaracion de intencion** (documentas que no forma parte de la API), y usa
  `internal` cuando necesites una frontera **dura** frente a otros paquetes.

---

## 6. Namespaces parciales

Varios ficheros pueden contribuir al **mismo** namespace, sin un "fichero
principal". Los simbolos de todos ellos se combinan bajo el mismo nombre logico:

```vesta
// part_a.vx
namespace bag;
public i64 uno(i64 x) { return x + 1; }
```

```vesta
// part_b.vx
namespace bag;
public i64 cien(i64 x) { return x + 100; }
```

Para que un consumidor vea los simbolos de **todos** los ficheros que forman el
namespace, cada fichero contribuyente debe entrar en la compilacion. La forma
mas directa es importarlos:

```vesta
// main.vx
import "part_a";
import "part_b";

i64 main() {
    return bag.uno(0) + bag.cien(1) - 60;   // 1 + 101 - 60 = R00 42
}
```

> **Nota.** `import bag;` (por namespace) trae el **primer** fichero que declara
> `bag`. Si el namespace esta repartido en varios ficheros, asegurate de que
> todos los contribuyentes entran en el build (por ejemplo importandolos por
> ruta, o importando el namespace y ademas el resto de ficheros). Mientras todos
> esten presentes, sus simbolos se combinan correctamente.

---

## 7. Reexport: `public import`

Un modulo puede **reexportar** lo que importa, para ofrecer una fachada. Con
`public import`, los simbolos del modulo importado quedan visibles a traves del
modulo que hace la fachada.

```vesta
// base.vx
namespace base;
public i64 val(i64 x) { return x + 1; }
```

```vesta
// facade.vx
namespace facade;
public import base;                    // reexporta base
public i64 own(i64 x) { return x + 2; }
```

```vesta
// main.vx
import facade;

i64 main() {
    i64 a = base.val(40);    // 41  (visible via el public import de facade)
    i64 b = facade.own(-1);  // 1
    return a + b;            // R00 42
}
```

Sin `public` delante, un `import` es privado al modulo que lo hace: el
importador ve los simbolos, pero no se propagan a sus propios consumidores.

---

## 8. `extension` e `impl`

`extension` e `impl` anaden metodos a un tipo **ya existente** (struct o clase),
posiblemente definido en otro modulo, al estilo de las extensiones de Swift /
C# o de `impl` en Rust. El *dispatch* es **estatico** (llamada directa,
*inline*-able, coste cero), y funcionan **cross-modulo**: puedes extender un
tipo importado y el consumidor ve los metodos anadidos.

### 8.1. `extension Tipo { ... }`

Anade metodos sueltos a un tipo.

```vesta
struct Punto { i64 x; i64 y; }

extension Punto {
    i64 suma() { return this.x + this.y; }
    i64 escala(i64 k) { return (this.x + this.y) * k; }
}

class Contador {
    i64 n;
    Contador(i64 v) { this.n = v; }
    i64 get() { return this.n; }
}

extension Contador {
    i64 doble() { return this.get() * 2; }
}

i64 main() {
    Punto p = { .x = 15, .y = 5 };
    Contador c = new Contador(6);
    return p.suma() + p.escala(1) + c.doble() - 10;  // 20 + 20 + 12 - 10 = R00 42
}
```

### 8.2. `impl Concept for Tipo { ... }`

Implementa un [concept](Generics.md) para un tipo, anadiendo los metodos que el
concept exige. Como los concepts de Vesta son predicados estructurales de
`comptime`, el `impl` satisface el concept al aportar esos metodos.

```vesta
concept Sumable<T> { i64 suma(); }

struct Par { i64 a; i64 b; }

impl Sumable for Par {
    i64 suma() { return this.a + this.b; }
}

i64 main() {
    Par p = { .a = 40, .b = 2 };
    return p.suma();   // R00 42
}
```

### 8.3. Regla de coherencia

Vesta es **permisivo** (a diferencia de la *orphan rule* de Rust): puedes
extender cualquier tipo con cualquier metodo. El unico error duro es la
**colision real**: dos extensiones visibles que anaden el mismo metodo con la
misma aridad al mismo tipo. En ese caso el compilador aborta y debes desambiguar
(renombrar el metodo o usar la forma de funcion libre).

---

## 9. Paquetes y `PackageId`

El **PackageId** identifica el paquete que produjo un modulo. Cumple dos
funciones:

1. **Desambiguar** namespaces homonimos de paquetes distintos (dos librerias
   pueden declarar `geo` sin colisionar si tienen PackageId diferente).
2. Marcar la **frontera de `internal`** (seccion 5.2).

Como se determina:

- **Derivado** por defecto del manifiesto `vx.toml` a partir de `[package] name`
  y `version`. Sin manifiesto, el paquete es **anonimo**.
- **Explicito** en el manifiesto con `[package] id = "..."`.
- **Sobrescrito por namespace** con `@id(...)`, util para renombrar un namespace
  publico manteniendo su identidad de ABI ante consumidores ya compilados:

```vesta
namespace geo.shapes @id("geo-stable-v1");

public i64 area(i64 w, i64 h) { return w * h; }
```

Puedes inspeccionar el PackageId (y los simbolos publicos) de una interfaz de
modulo (`.vxi`) con:

```bash
vm pkg inspect geo.vxi
# ...
#   package_id: geo-stable-v1
#   symbols: ...
#   fn  area  (i64, i64) -> i64
```

Un namespace sin manifiesto ni `@id` aparece como `package_id: (anonimo)`.

Ver [Gestor de paquetes](PackageManager.md) para el detalle de `vx.toml` y el
ciclo de vida de los paquetes.

---

## 10. Interaccion con el resto del lenguaje

Los namespaces se integran con todas las construcciones del lenguaje. En todos
los casos, el simbolo se referencia por su nombre cualificado desde fuera y por
su nombre desnudo entre hermanos.

### 10.1. Clases, structs y enums

Se declaran `public` dentro del namespace y se consumen por su nombre
cualificado (`geo.Box`, `geo.Vec2`, `geo.Shape`), como en la seccion 3. El
`new`, el acceso a campos, los metodos y el `match` sobre variantes de enum
funcionan cross-modulo.

### 10.2. Genericos

- Las **funciones genericas** en un namespace funcionan cross-modulo, con
  type-args explicitos o inferidos, y con bounds a concepts del namespace:

  ```vesta
  namespace mat;
  concept Numerico<T> = is_numeric<T>();
  public T doble<T: mat.Numerico>(T x) { return x + x; }
  public K primero<K, V>(K a, V b) { return a; }
  ```

  ```vesta
  import "mat_lib";
  i64 main() {
      i64 a = mat.doble<i64>(20);          // 40 (explicito + bound)
      i64 c = mat.primero<i64, i64>(1, 9); // 1  (multi-param)
      return a + c - 41 + 42;              // R00 42
  }
  ```

- Las **clases genericas** (`class Box<T>`) funcionan dentro del **modulo que
  las declara**, referenciadas por su nombre desnudo:

  ```vesta
  namespace col;
  public class Box<T> {
      public T v;
      public Box(T x) { this.v = x; }
      public T get() { return this.v; }
  }
  // dentro del mismo modulo:
  i64 usar() { Box<i64> b = new Box<i64>(42); return b.get(); }
  ```

  Instanciar una **clase generica desde otro modulo** por su nombre cualificado
  (`col.Box<i64>`) es una limitacion actual: para consumir plantillas genericas
  cross-modulo, expon una **funcion generica** o un tipo concreto ya
  monomorfizado.

### 10.3. Concepts y constraints

Un `concept` declarado en un namespace se usa en un bound por su nombre
cualificado, tanto en la forma inline `<T: mat.Numerico>` como en la clausula
`where T: mat.Ordenable`. Las constraints son 100% de `comptime` y desaparecen
del binario.

```vesta
namespace mat;
concept Numerico<T> = is_numeric<T>();
concept Ordenable<T> = Comparable<T>() && Sized<T>();

public T doble<T: mat.Numerico>(T x) { return x + x; }
public T maxv<T>(T a, T b) where T: mat.Ordenable {
    if (a < b) { return b; }
    return a;
}
```

### 10.4. Metaprogramacion `comptime`

Las constantes `comptime`, las funciones `comptime` y los macros pueden vivir en
un namespace y usarse por su nombre cualificado; todo se pliega o expande en
tiempo de compilacion.

```vesta
namespace mat;

comptime i64 BASE = 40;
public comptime i64 sq(i64 n) { return n * n; }

@Macro
public comptime string veces7(i64 n) { return to_str(n * 7); }
```

```vesta
import "mathlib";
i64 main() {
    i64 s = mat.sq(5);       // 25 (fn comptime -> fold)
    i64 m = mat.veces7(3);   // 21 (macro -> literal)
    return s + m - 4;        // 25 + 21 - 4 = R00 42
}
```

### 10.5. Introspeccion

Los builtins de introspeccion (`typename<T>`, y en general los que devuelven el
nombre de un tipo) devuelven el **nombre publico** del tipo, sin el prefijo del
namespace:

```vesta
namespace geom;
public struct Vec3 { i64 x; i64 y; i64 z; }

comptime string N = typename<geom.Vec3>();

i64 main() {
    static_assert(comptime_streq(N, "Vec3"), "nombre publico sin prefijo");
    return 42;
}
```

`typename<geom.Vec3>()` produce `"Vec3"`, no `"geom.Vec3"`.

### 10.6. Reflexion

La reflexion registra las clases por su **nombre publico**. Una clase declarada
dentro de un namespace se busca por su nombre desnudo:

```vesta
namespace zoo;
public class Animal {
    public i32 edad;
    public Animal(i32 e) { this.edad = e; }
}

i32 main() {
    Animal a = new Animal(7);
    i64 by_name = forName("Animal");   // por nombre publico
    i64 of_obj  = getClass(a);
    if (by_name == of_obj && by_name != 0) { return 42; }
    return 0;
}
```

### 10.7. Programacion orientada a aspectos (AOP)

Los *pointcuts* de AOP identifican el metodo objetivo por su nombre publico
`Clase.metodo`, aunque la clase viva en un namespace:

```vesta
namespace ejemplos.aop;

class Service {
    public Service() { }
    public i32 run() { return 1; }
}

@Aspect class Tracer {
    public Tracer() { }
    @Before("Service.run") public i32 pre()  { return 7; }
    @After("Service.run")  public i32 post() { return 99; }
}

i32 main() {
    Service s = new Service();
    return s.run();   // el AFTER pisa el retorno -> R00 99
}
```

---

## 11. Buenas practicas

- **Namespacea las librerias, no los ejecutables.** El fichero con `main` no
  necesita namespace.
- **Un namespace por area logica**, no por fichero: varios ficheros pueden
  contribuir al mismo namespace (namespaces parciales, seccion 6).
- **Prefiere `import a.b.c;` (por namespace)** frente al import por ruta cuando
  el codigo es una libreria reubicable: reorganizar carpetas no rompe los
  imports.
- **Marca `internal`** lo que es detalle de implementacion del paquete pero
  debe compartirse entre sus modulos; reserva `public` para la API real.
- **Desambigua con acceso cualificado** cuando dos namespaces exportan nombres
  iguales, en lugar de confiar en imports selectivos.
- **Usa `@id(...)`** solo si necesitas renombrar un namespace publico sin romper
  la ABI de consumidores ya compilados.
- **La stdlib** expone una capa importable con namespace (`std.math`,
  `std.text`, ...); el runtime interno del AOT no lleva namespace por diseno
  (seccion 12).

```vesta
import std.math;
import std.text;

i64 main() {
    f64 h = std.math.hypot(3.0, 4.0);      // 5.0
    i64 c = std.math.iclamp(100, 0, 37);   // 37
    return (i64)h + c;                      // 5 + 37 = R00 42
}
```

---

## 12. Limitaciones y cuando NO usar namespace

Poner un simbolo en un namespace cambia su **nombre fisico** (`a.b.c.name` pasa a
ser `a__b__c__name`, seccion 2). Cualquier construccion que dependa de un nombre
de simbolo **estable y crudo** debe tenerlo en cuenta. Como regla, **mantenlos en
el namespace global** (sin declaracion `namespace`) cuando:

- **Ensamblador que referencia simbolos propios por nombre.** El codigo
  ensamblador que salta o llama a otro simbolo Vesta por su etiqueta cruda, y las
  funciones `@Naked` que se llaman entre si por nombre en asm, esperan el nombre
  sin manglear. Un namespace cambia esa etiqueta a `ns__nombre` y la referencia
  deja de resolver.

- **Interoperabilidad con C.** Cuando un consumidor `.c` externo referencia una
  funcion Vesta por su nombre C estable, ese nombre debe permanecer sin manglear.
  Declara esos puntos de entrada en el namespace global. Ver
  [Interoperabilidad con C](InteropC.md).

- **Carga dinamica por nombre.** Si buscas un simbolo por su nombre en tiempo de
  ejecucion (carga de modulos por nombre, o resolucion de simbolos crudos), usa
  el nombre que realmente existe: el fisico manglear si esta en un namespace, o
  manten el simbolo en el namespace global para que su nombre coincida con el
  publico.

Otras limitaciones a tener presentes:

- El **import por namespace** (`import a.b.c;`) trae el primer fichero que
  declara el namespace; para namespaces repartidos en varios ficheros, incluye
  todos los contribuyentes en el build (seccion 6).
- El import simple **no** habilita el acceso por el ultimo segmento
  (`shapes.area`); usa el path completo, un alias, o un import selectivo
  (seccion 4).
- Instanciar **clases genericas** por su nombre cualificado **desde otro modulo**
  no esta soportado (seccion 10.2); expon funciones genericas o tipos concretos.

---

## Ver tambien

- [Sistema de modulos](Modulos.md) - imports, paquetes, cache, `.vxi`.
- [Gestor de paquetes](PackageManager.md) - `vx.toml`, PackageId, `vex_modules`.
- [Genericos](Generics.md) - monomorfizacion, concepts, `impl`.
- [Reflexion y AOP](ReflexionAOP.md) - `forName`, `getClass`, `@Aspect`.
- [Metaprogramacion](Metaprogramacion.md) - `comptime`, macros, introspeccion.
- [Interoperabilidad con C](InteropC.md) - nombres de simbolo estables.
