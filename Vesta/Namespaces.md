# Namespaces

Los **namespaces** dan direccionamiento lógico a los símbolos públicos de
Vesta, desacoplado del fichero físico. Complementan al
[sistema de módulos](Modulos.md): un módulo es la unidad de compilación
(fichero/paquete); un namespace es el nombre lógico bajo el que viven sus
símbolos y por el que otros módulos los importan.

> Regla de oro: un namespace es para **librerías** (código reutilizable). El
> top-level de un ejecutable (el fichero con `main`) no necesita namespace —
> aunque puede tenerlo.

---

## 1. Declarar un namespace

Dos formas, semánticamente equivalentes:

```vesta
// (a) STATEMENT: agrupa el RESTO del fichero.
namespace geo.shapes;

public struct Rect { i64 w; i64 h; }
public i64 area(i64 w, i64 h) { return w * h; }
```

```vesta
// (b) BLOQUE: acota con llaves; permite varios namespaces por fichero.
namespace geo.shapes {
    public struct Rect { i64 w; i64 h; }
}
namespace geo.color {
    public i64 rgb(i64 r, i64 g, i64 b) { return (r << 16) | (g << 8) | b; }
}
```

El path es punteado (`a.b.c`). El mangling físico une los segmentos con `__`
(`geo.shapes.area` → `geo__shapes__area`), pero eso es un detalle interno: en el
código escribes siempre la notación con puntos.

Dentro del namespace, los **hermanos** se llaman por su nombre desnudo (incluso
en interpolación `${...}`, lambdas, `match`, etc.):

```vesta
namespace demo;
i64 doble(i64 x) { return x * 2; }
i64 main() {
    println("r=${doble(21)}");   // 42 -- `doble` es hermano del namespace
    return doble(21);
}
```

---

## 2. Acceso cualificado

Desde fuera del namespace, se accede con el path punteado:

```vesta
geo.shapes.Rect r = { .w = 3, .h = 7 };
i64 a = geo.shapes.area(r.w, r.h);
```

Funciona para todo: funciones, structs, clases, enums, typedefs, globales,
`comptime` const/fn, macros, concepts y genéricos.

---

## 3. Importar por namespace

Además del import por-path (`import "ruta/fichero";`), puedes importar por
**namespace**. El compilador resuelve el namespace a fichero mediante un índice
que escanea las cabeceras `namespace` de las *source roots* (proyecto + stdlib
+ `vex_modules`):

```vesta
import geo.shapes;              // por namespace (no por ruta de fichero)
import geo.shapes as g;         // alias del namespace
import geo.shapes.{ area, Rect }; // selectivo: trae area/Rect al scope local

i64 main() {
    i64 a = geo.shapes.area(6, 7);   // cualificado completo
    i64 b = g.area(1, 8);            // vía alias
    Rect r = { .w = 3, .h = 0 };      // vía selectivo (sin cualificar)
    return a + b;
}
```

---

## 4. Visibilidad: `public`, `internal`, `private`

Tres ejes, ortogonales al namespace (el mangling es siempre por namespace; la
visibilidad es una regla aparte):

| Modificador | Alcance |
| :---------- | :------ |
| `public`    | Exportado del paquete (visible para consumidores externos). |
| `internal`  | Visible en TODO el paquete (mismo *PackageId*), pero NO exportado a paquetes distintos. Equivale a `internal` de C# / `pub(crate)` de Rust. |
| `private` / sin modificador | Privado al fichero/tipo. |

```vesta
namespace lib;
public   i64 publica(i64 x) { return x + 1; }   // la ven otros paquetes
internal i64 interna(i64 x) { return x + 100; } // solo dentro de este paquete
```

`internal` se apoya en el *PackageId*: un consumidor de OTRO paquete no ve los
símbolos `internal` del dependiente.

---

## 5. Identidad de paquete: PackageId y `@id`

Cada `.vxi` (interfaz binaria de un módulo) lleva un **PackageId** que identifica
el paquete que lo produjo. Sirve para (a) desambiguar namespaces homónimos de
paquetes distintos y (b) marcar la frontera de `internal`.

- **Derivado** por defecto del manifiesto `vx.toml` (`[package] name` +
  `version`) → `pkg:<hash>`. Sin manifiesto, el paquete es *anónimo*.
- **Explícito** en el manifiesto con `[package] id = "..."`.
- **Override por namespace** con `@id(...)`, para renombrar el namespace
  manteniendo la identidad ABI:

```vesta
namespace geo.shapes @id("geo-stable-v1");
```

Puedes inspeccionar el PackageId de un `.vxi`:

```bash
vesta pkg inspect geo.vxi
# ... package_id: geo-stable-v1
```

---

## 6. `extension` e `impl`

Añaden métodos a un tipo **ya existente** (struct o clase), posiblemente de otro
módulo, estilo Swift extensions / C# extension methods / Rust `impl`. Dispatch
**estático** (llamada directa, *inline*-able, coste cero).

```vesta
struct Punto { i64 x; i64 y; }

// extension: anyade metodos sueltos a un tipo.
extension Punto {
    i64 suma() { return this.x + this.y; }
}

// impl: implementa un concept para un tipo (conformance estructural).
concept Sumable<T> { i64 suma(); }
impl Sumable for Punto {
    // (si el concept exige mas metodos, se anyaden aqui)
}

i64 main() {
    Punto p = { .x = 40, .y = 2 };
    return p.suma();   // 42
}
```

Funciona **cross-módulo**: puedes extender un tipo importado de otro módulo, y
el consumidor ve los métodos añadidos (viajan en el `.vxi`).

### Regla de coherencia

Vesta es **permisiva** (a diferencia de la *orphan rule* de Rust): puedes
extender cualquier tipo con cualquier método. El único error duro es la
**colisión real**: dos extensiones visibles que añaden el mismo método+aridad al
mismo tipo. En ese caso el compilador aborta y debes desambiguar (renombrar o
usar la forma función-libre).

---

## 7. Buenas prácticas

- **Namespacea las librerías, no los ejecutables.** El fichero con `main` no
  necesita namespace.
- **Un namespace por área lógica**, no por fichero: varios ficheros pueden
  contribuir al mismo namespace (namespaces parciales).
- **Prefiere `import a.b.c;` (por namespace)** sobre el import por-ruta cuando el
  código es una librería reubicable: reorganizar carpetas no rompe los imports.
- **Marca `internal`** lo que es detalle de implementación del paquete pero debe
  ser visible entre sus módulos.
- **Usa `@id(...)`** solo si necesitas renombrar un namespace público sin romper
  la ABI de consumidores ya compilados.
- **La stdlib** expone una capa importable namespaced (`std.math`, `std.text`,
  ...); el runtime interno del AOT (`vx_io` etc.) no lleva namespace por diseño.

---

## Ver también

- [Sistema de módulos](Modulos.md) — imports, paquetes, caché, `.vxi`.
- [Gestor de paquetes](PackageManager.md) — `vx.toml`, PackageId, `vex_modules`.
- [Genéricos](Generics.md) — monomorfización, concepts.
