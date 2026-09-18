# Closures en campos y metodos ligados

Una lambda en Vesta es un valor de 16 bytes: el par `{codigo, env}`, donde `env`
es el bloque que guarda las variables capturadas.  Cuando una lambda con captura
se almacena en un **campo** de un objeto, su `env` debe vivir exactamente lo que
viva ese objeto.  Vesta lo resuelve con ownership determinista (RAII), sin
intervencion del recolector de basura: el `env` es propiedad del objeto que lo
contiene y se libera con el, igual que un campo `unique<T>`.

Documento hermano: [`InteropC.md`](InteropC.md) (categorias de tipos C-compatible
vs gestionado, ownership de structs).

## El problema del lifetime

Para una lambda en una variable local que no escapa, el `env` vive en el stack
del scope y se libera al salir, sin coste.  Pero si la lambda se guarda en el
campo de un objeto que puede sobrevivir al frame actual, un `env` en stack
quedaria colgando al retornar.  El bloque de capturas necesita un lifetime
ligado al del objeto contenedor.

## El env como propiedad del contenedor (sin GC)

El `env` de una lambda no es un objeto del recolector: sale de un allocator que
devuelve memoria cruda del host.  En lugar de hacerlo rastreable por el GC, Vesta
lo modela como un **recurso propiedad del objeto contenedor**:

- **El par `{codigo, env}` vive INLINE en el campo**, que por eso mide 16 bytes:
  no es un puntero a otro sitio, es el par mismo.  Guardar una lambda en un
  campo copia los dos qwords.
- Lo unico reservado aparte es el **`env`**, y de eso el objeto contenedor es
  dueno, como si fuera un campo `unique<env>`.
- El destructor del contenedor libera el `env` (y reasignar el campo libera el
  anterior antes de guardar el nuevo).  El par no se libera: no es una reserva
  propia, vive dentro del contenedor.

Esto reutiliza el mismo mecanismo de RAII que libera los campos `unique<T>` y los
campos struct gestionados: un contenedor con un campo lambda se vuelve
destructible y gana la liberacion automatica.

## Reglas segun el contenedor

| Contenedor del campo | Ubicacion del env | Liberacion |
|:---|:---|:---|
| Clase (`obj.f = lambda`) | heap del host, propiedad del objeto | el destructor del contenedor y la reasignacion del campo lo liberan |
| Struct local que no escapa (`s.f = lambda`) | stack, en el scope del struct | implicita: muere con el frame, sin coste |
| Struct que se retorna por valor | heap del host, transferido por move | el consumidor (quien recibe el struct) lo libera al salir de su scope |

**Clase.**  El par vive en el campo y el `env` en un bloque del heap que no
cuelga aunque el objeto escape el scope donde se creo.  Al destruirse el objeto,
su destructor libera ese bloque; al reasignar el campo, se libera el anterior
antes de guardar el nuevo.

**Struct.**  Un struct es value-type y se mueve por valor (SRET).  Para el caso
comun de un callback usado localmente, su `env` se queda en stack y muere con el
scope, sin coste.  Cuando el struct **se retorna por valor**, el compilador
detecta el escape y aloca su `env` en heap: el struct se mueve al llamante (sus
bytes, el par `{codigo, env}` incluido, se copian al buffer de retorno) y el
ownership del `env` se transfiere — el productor no lo libera y el consumidor que
recibe el struct lo libera al salir de su scope.  Un unico free, sin GC.
Almacenar ese struct en un **campo** de otro que le sobrevive sigue la misma
regla: el ownership del `env` viaja con la copia, asi que quien acabe teniendo
el valor es quien lo libera.

```vesta
struct Holder { fn(i64) -> i64 f; }
struct Outer  { Holder h; }

Holder make(i64 base) {
	Holder h = { .f = (i64 x) => x + base };
	return h;                       // el env se va con el valor devuelto
}

i32 main() {
	Outer o = { .h = make(10) };     // y sobrevive a `make`
	return (i32) o.h.f(32);          // -> 42
}
```

**Capturas por referencia.**  Una captura por referencia (una variable mutada
dentro de la lambda) que escapa a un campo de clase es un error de compilacion:
el `env` guardaria un puntero a un slot de stack que muere.  Solo se permiten
capturas por valor, que se copian al `env` y son seguras de liberar con el
contenedor.  Esto es coherente con el borrow checker.

## Ciclo de vida

```vx
o.f = (x) => x + base;   // env = {base} en heap; el tipo de 'o' pasa a destructible
// ... uso de o ...
// al destruirse 'o' (fin de scope o transferencia de ownership):
//   por cada campo lambda: se libera su env del heap
```

## Metodos ligados: `&obj.metodo`

`&obj.metodo` (cuando `obj` es una variable de tipo clase o struct y `metodo` es
un metodo) es una lambda que **captura el receptor**:

```vx
&c.inc   ===   (args) => c.inc(args)
```

El metodo ligado opera siempre sobre el mismo objeto, se puede guardar en
variables y campos, y pasar como callback, igual que cualquier lambda.  Si se
guarda en un campo de clase, su `env` (que contiene el receptor) es propiedad del
campo y lo libera el destructor del contenedor.

Para una clase, la captura es la referencia al objeto; para un struct, es una
copia del struct por valor.  La base puede ser una variable (`&var.metodo`) o una
expresion compuesta (`&obtener().metodo`); en este ultimo caso la base se evalua
**una sola vez**, al crear el puntero ligado, y el receptor resultante se captura
para todas las llamadas posteriores.

Esto contrasta con `&Tipo.metodo` (no ligado), que es un puntero a funcion crudo
(`cfn`, 8 bytes) que recibe el receptor como primer parametro explicito;
`&obj.metodo` es una lambda (16 bytes) que ya lleva el receptor dentro.

## Como funciona por dentro

- Cuando una lambda se almacena en un campo de clase, su `env` y su par
  `{codigo, env}` se alocan en el heap del host (no en stack).  El campo guarda el
  puntero al par.
- Un campo lambda hace al tipo contenedor destructible; el destructor (propio o
  sintetizado) libera el `env` de cada campo lambda antes de retornar, con guarda
  de null.  La reasignacion del campo libera el `env` anterior.
- Al cargar el campo para llamarlo, las lecturas de `codigo` y `env` salen del
  heap del host, lo que en el interprete y el JIT usa el modo de acceso a memoria
  del host; en AOT todo el espacio es del host.
- Un metodo ligado `&obj.metodo` se traduce a la lambda `(args) => obj.metodo(args)`
  y entra por exactamente el mismo flujo que cualquier otra lambda.

El comportamiento es identico en los tres backends: interprete, JIT y AOT nativo.

## Limitaciones

- Las capturas por referencia que escapan a un campo se rechazan; usa captura por
  valor.  El mensaje lo dice entero: *"captura por referencia en un closure que
  se almacena en un campo: 'acc' se mutaria fuera de su scope.  Captura por
  valor (no la modifiques dentro del lambda)."*
- En el caso de clase, el `env` se aloca en heap aunque el holder no escape; la
  promocion a stack de los holders provablemente locales es una optimizacion
  futura del analisis de escape.
