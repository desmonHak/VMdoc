# Interop con C y ownership de structs

Vesta se integra con C en las dos direcciones: el codigo Vesta puede llamar a C
(via `extern "lib"`) y, sobre todo, el codigo Vesta compilado a nativo (AOT) puede
ser **llamado desde C** como una libreria mas, sin runtime de la VM en la
frontera.  Esta integracion descansa en dos propiedades del lenguaje: el layout
C de los structs y un modelo de ownership con liberacion determinista (RAII +
move semantics, sin GC).  Este documento explica ambas y como usarlas.

Documento hermano: [`ClosuresEnCampos.md`](ClosuresEnCampos.md) (cfn vs lambda,
closures almacenados en campos).

## Las dos categorias de tipos

Cada tipo cae en una de dos categorias, que el compilador **infiere** de su
forma (no hay anotaciones que ponerlas):

- **C-representable**: cruza la frontera C por valor, con ABI C, y se copia como
  bits (memcpy).  Son los primitivos (`i32`, `f64`, `bool`, `char`...), los
  punteros host (`T*`), los punteros a funcion crudos (`cfn(...)->R`), los
  `borrow<T>`, y los structs cuyos campos son todos C-representables.

- **Gestionado**: posee un recurso de lifetime no-C y por tanto es move-only con
  liberacion determinista (RAII).  Son las lambdas con captura (`fn(...)->R`),
  `string`, las clases, las colecciones, `unique<T>`/`shared<T>`, y los structs
  que contienen algun campo gestionado o que declaran un destructor `~Struct()`.

Las dos categorias **no son complementarias**: un `VirtualPtr<T>` (direccion del
espacio de la VM) no es C-representable pero tampoco es gestionado.

### El layout C de los structs es el unico layout

Todos los structs Vesta usan layout C: campos en orden de declaracion y alineacion
de plataforma (padding al campo mas grande).  No existe un layout alternativo,
asi que no hay nada que activar: un struct sin campos gestionados es C-compatible
por defecto.

## Structs que poseen recursos

Un struct value-type puede declarar un destructor `~Struct()`, igual que una
clase.  Con eso pasa a ser **gestionado**: el compilador llama a su destructor de
forma determinista cuando la instancia sale de scope (RAII), y aplica move
semantics para que el recurso tenga siempre un unico dueno.

### Destructor y RAII al salir del scope

```vx
struct Recurso {
    i64* p;
    ~Recurso() { free(this.p); }
}

i32 usar() {
    Recurso r;
    r.p = (i64*) malloc(8);
    *r.p = 42;
    return (i32)(*r.p);   // al hacer return, r sale de scope -> ~Recurso() -> free(r.p)
}
```

El destructor se invoca con un CALL directo a `Recurso____dtor(&r)` insertado al
final del scope.  Es dispatch estatico (sin vtable) e inlineable, asi que un
destructor trivial cuesta practicamente cero.

### Move al retornar (move-on-return)

Retornar un struct gestionado **transfiere** su recurso al llamante en vez de
copiarlo: el destructor del local que se retorna se **suprime**, y el llamante
recibe la responsabilidad de liberar (su propia copia ejecutara el destructor al
salir de su scope).  El resultado es un unico `free`, sin coste de copia
profunda ni de refcount.

```vx
Recurso crear(i64 v) {
    Recurso r;
    r.p = (i64*) malloc(8);
    *r.p = v;
    return r;            // move: ~Recurso() de r NO corre aqui; el dueno pasa al caller
}
```

### Move al almacenar en un campo (move-on-store)

Asignar un struct gestionado a un campo de un contenedor que tambien es
destructible **mueve** el recurso al campo: copia los bytes del struct al campo
y suprime el destructor del origen.  Solo el destructor del contenedor libera.

```vx
class Caja {
    Recurso r;     // el campo struct hace a Caja destructible
}

i32 demo() {
    Recurso tmp;
    tmp.p = (i64*) malloc(8);
    *tmp.p = 41;
    Caja c = new Caja();
    c.r = tmp;            // move: tmp queda invalidado; Caja posee el recurso
    i64 v = *c.r.p;       // 41
    return (i32)v;        // al salir, ~Caja() libera el Recurso una sola vez
}
```

Almacenar un struct gestionado en un destino que **no** asume su ownership (un
slot de array, un `*ptr = ...`, o un campo de un contenedor no destructible) se
rechaza en tiempo de compilacion, porque dejaria el recurso sin dueno (fuga).

### Recursos `unique<T>` en un campo

Un contenedor puede tener un campo `unique<T>`; al destruirse, libera el recurso
del campo invocando su deleter (el de por defecto, `free`, o el personalizado
dado a `unique_with`).

```vx
class Conexion {
    unique<i64> handle;     // hace a Conexion destructible
}

void usar(i64 fd) {
    Conexion c = new Conexion();
    c.handle = unique_with(fd, cerrar);   // adopta el recurso
}   // al salir: ~Conexion() -> cerrar(fd), un unico cierre deterministico
```

El slot del `unique<T>` vive en heap para sobrevivir junto al contenedor.

### Composicion y RAII recursivo

Un contenedor (struct o clase) que tiene un campo de un tipo destructible se
vuelve destructible automaticamente: gana un destructor implicito que invoca el
destructor de cada campo gestionado, en orden de declaracion.  Esto encadena la
liberacion sin escribir codigo manual, a cualquier profundidad.

```vx
struct Inner { i64* p;  ~Inner() { free(this.p); } }
struct Mid   { Inner inner;  i64 tag; }     // gana ~Mid() implicito -> ~Inner()
struct Outer { Mid mid; }                   // gana ~Outer() implicito -> ~Mid() -> ~Inner()

i64 usar(i64 v) {
    Outer o;
    o.mid.inner.p = (i64*) malloc(8);   // acceso a un campo struct anidado
    *o.mid.inner.p = v;
    return *o.mid.inner.p;              // al salir: ~Outer -> ~Mid -> ~Inner -> free
}
```

El mismo mecanismo aplica cuando el contenedor es una clase con un campo struct
gestionado: el destructor de la clase libera el campo struct con un CALL directo
a su destructor.

## Como funciona por dentro

- **Inferencia de destructibilidad.**  Durante el analisis semantico se calcula,
  por punto-fijo, que structs y clases son destructibles (tienen `~T()` propio o
  un campo destructible).  El calculo de structs corre antes que el de clases,
  para que una clase vea el destructor sintetizado de un struct que contiene.
  A los tipos destructibles sin `~T()` propio se les sintetiza uno implicito.

- **Augmentacion del destructor.**  Al bajar el cuerpo de un destructor (propio o
  sintetizado), el compilador inserta, antes del retorno, la liberacion de cada
  campo gestionado: para un campo struct, un CALL directo a `<Struct>____dtor`
  con la direccion del campo; para un campo clase, un CALLVIRT con guarda de
  null; para un campo lambda, la liberacion del bloque env (ver
  [`ClosuresEnCampos.md`](ClosuresEnCampos.md)).

- **El campo struct es inline.**  Un campo de tipo struct vive **dentro** del
  payload del contenedor; su "valor" es su direccion, no un puntero a otro
  bloque.  Por eso acceder a `obj.campo.x` o `obj.campo.metodo()` computa la
  direccion del campo (base + offset) y opera ahi, sin un load intermedio.

- **Move = supresion del destructor del origen.**  El analisis de escape detecta
  cuando un local se retorna o se almacena en un campo/slot/deref y lo marca como
  "escapante"; su destructor de salida de scope no se emite.  Combinado con la
  copia de bytes al destino, eso da move semantics de coste cero: un unico dueno,
  un unico `free`.

- **Copia memberwise en el store.**  Asignar un struct a un campo copia sus bytes
  qword a qword desde la direccion del origen a la del campo (respetando si cada
  lado es memoria de la VM o del host).  No es un store escalar: este pisaria el
  primer qword del campo con la direccion del origen.

Todo lo anterior es identico en los tres backends (interprete, JIT y AOT nativo).

## Usar codigo Vesta desde C

El frontend puede emitir una cabecera C tipada con `--emit-header`, junto al
modo de transpilacion a C:

```sh
vesta --port c --emit-header mi_lib.vx -o mi_lib
# genera  mi_lib.c  (codigo)  +  mi_lib.h  (tipos y prototipos)
```

La cabecera contiene los typedefs de los structs C-compatibles, los prototipos
de las funciones cuya firma es C-representable, y los punteros a funcion `cfn`.
Un programa C la incluye y enlaza el `.c`:

```c
#include "mi_lib.h"

int main(void) {
    return (int) mi_funcion_vx(20, 22);   // 42
}
```

Para que el `.vx` sea consumible como libreria no debe tener `main` (si lo
tiene, el transpilador emite un `main` envoltorio que colisiona con el del
consumidor).

### Closures en la frontera C

Una lambda gestionada `fn(...)->R` es internamente `{code_ptr, env_ptr}` y su
env la posee el compilador, algo que C no expresa.  Para cruzar a C se usa el par
crudo de la convencion universal de callbacks: un puntero a funcion `cfn(...)->R`
mas un `void*` de contexto, que C maneja y libera manualmente.

## Limitaciones actuales

- El move al almacenar cubre los campos; almacenar un struct gestionado en un
  **slot de array** o tras un **deref de puntero** sigue rechazandose.
- Pasar un struct **por valor** con copy-hook `__clone__` hace una copia (se
  invoca `__clone__` y el `~dtor` de la copia corre tras la llamada).  Para un
  struct gestionado **sin** copy-hook (move-only), el paso por valor todavia es
  un alias; usa `borrow<T>` o un puntero si necesitas pasarlo sin copiar.
- La cabecera generada (`--emit-header`) describe los structs por puntero
  (compatible con el `.c` del transpilador); las firmas tipadas con structs
  nombrados estan en curso.
