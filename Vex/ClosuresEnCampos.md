# Closures en campos y metodos ligados

Una lambda en Vex es un valor de 16 bytes: el par `{codigo, env}`, donde `env`
es el bloque que guarda las variables capturadas.  Cuando una lambda con captura
se almacena en un **campo** de un objeto, su `env` debe vivir exactamente lo que
viva ese objeto.  Vex lo resuelve con ownership determinista (RAII), sin
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
devuelve memoria cruda del host.  En lugar de hacerlo rastreable por el GC, Vex
lo modela como un **recurso propiedad del objeto contenedor**:

- El `env` y el par `{codigo, env}` se alocan en el heap del host cuando la
  lambda se guarda en un campo de clase.
- El objeto contenedor es su dueno, como si fuera un campo `unique<env>`.
- El destructor del contenedor libera el `env` (y reasignar el campo libera el
  anterior antes de guardar el nuevo).

Esto reutiliza el mismo mecanismo de RAII que libera los campos `unique<T>` y los
campos struct gestionados: un contenedor con un campo lambda se vuelve
destructible y gana la liberacion automatica.

## Reglas segun el contenedor

| Contenedor del campo | Ubicacion del env | Liberacion |
|:---|:---|:---|
| Clase (`obj.f = lambda`) | heap del host, propiedad del objeto | el destructor del contenedor y la reasignacion del campo lo liberan |
| Struct (`s.f = lambda`) | stack, en el scope del struct | implicita: muere con el frame, sin coste |

**Clase.**  El campo guarda un puntero a un bloque heap que no cuelga aunque el
objeto escape el scope donde se creo.  Al destruirse el objeto, su destructor
libera el bloque; al reasignar el campo, se libera el bloque anterior antes de
guardar el nuevo.

**Struct.**  Un struct es value-type y vive en el scope actual, asi que el `env`
puede quedarse en stack y morir con el.  Es correcto para el caso comun de un
callback usado localmente.  Si un struct que guarda una lambda con captura
**escapa** su scope (se retorna por valor, o se almacena en algo que le
sobrevive), su copia apuntaria a un `env` de stack ya liberado; el compilador lo
rechaza con un error que sugiere usar una clase, cuyo `env` vive en heap y lo
libera el destructor.

**Capturas por referencia.**  Una captura por referencia (una variable mutada
dentro de la lambda) que escapa a un campo de clase es un error de compilacion:
el `env` guardaria un puntero a un slot de stack que muere.  Solo se permiten
capturas por valor, que se copian al `env` y son seguras de liberar con el
contenedor.  Esto es coherente con el borrow checker.

## Ciclo de vida

```vex
o.f = (x) => x + base;   // env = {base} en heap; el tipo de 'o' pasa a destructible
// ... uso de o ...
// al destruirse 'o' (fin de scope o transferencia de ownership):
//   por cada campo lambda: se libera su env del heap
```

## Metodos ligados: `&obj.metodo`

`&obj.metodo` (cuando `obj` es una variable de tipo clase o struct y `metodo` es
un metodo) es una lambda que **captura el receptor**:

```vex
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

- Un campo lambda con captura en un **struct** que escapa su scope no esta
  soportado (requeriria move semantics de value-types con captura); se rechaza en
  compilacion.  Para ese caso usa una clase.
- Las capturas por referencia que escapan a un campo se rechazan; usa captura por
  valor.
- Los holders que el compilador puede probar que no escapan siguen alocando el
  `env` en heap; la promocion a stack de esos casos es una optimizacion futura.
