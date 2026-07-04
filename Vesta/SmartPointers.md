# Smart pointers: `unique<T>` y `shared<T>`

Vesta ofrece dos punteros inteligentes con gestion de recursos **determinista**
(RAII), independientes del recolector de basura: `unique<T>` para ownership
exclusivo y `shared<T>` para ownership compartido por conteo de referencias.
Ambos liberan su recurso en un momento preciso -- al destruirse el ultimo dueno,
no "cuando el GC decida" -- lo que los hace validos tanto en codigo gestionado
como en binarios nativos sin runtime (AOT).

Documento hermano: [`BorrowChecker.md`](BorrowChecker.md) (`borrow<T>` para acceso
temporal sin transferir ownership) e [`InteropC.md`](InteropC.md) (ownership de
structs y campos).

## El modelo de ownership

Un smart pointer es un value-type que **posee** un recurso (memoria, un handle
del SO, una conexion...) y garantiza su liberacion.  La diferencia entre los dos
es como se comparte ese ownership:

- **`unique<T>` -- un solo dueno.**  No se puede copiar; solo *mover*.  Al salir
  de scope, libera el recurso.  Es la opcion por defecto: ownership claro y coste
  cero.
- **`shared<T>` -- varios duenos.**  Se puede copiar; cada copia incrementa un
  conteo de referencias.  El recurso se libera cuando se destruye la *ultima*
  copia (el conteo llega a cero).

Ambos liberan de forma deterministica, sin GC.  En el caso de `shared<T>` la
liberacion es por **conteo de referencias puro**: el bloque se libera
inmediatamente cuando el contador cae a cero.

## `unique<T>` -- ownership exclusivo

```vex
unique<i32> p = unique_box(42);   // construye + posee
i32 v = *ptr_of(p);                // 42
// al salir de scope: el recurso se libera automaticamente
```

`unique<T>` es **move-only**: copiarlo es un error de compilacion; hay que
transferir el ownership explicitamente con `move`.

```vex
unique<i32> a = unique_box(42);
unique<i32> b = move(a);   // ownership transferido; `a` queda invalidado
i32 v = *ptr_of(b);        // 42
// al salir: solo `b` libera (un unico free)
```

Retornar un `unique<T>` por valor es un *move* (transfiere el ownership al
llamante, sin coste):

```vex
unique<i64> abrir(string ruta) {
    return unique_with(fopen(...), cerrar);   // el llamante recibe el ownership
}
```

## `shared<T>` -- ownership compartido (refcount)

```vex
shared<i32> a = shared_box(78);   // conteo = 1
shared<i32> b = a;                // COPIA: conteo = 2 (a y b)
i64 n = use_count(a);             // 2
// al salir de scope b: conteo = 1; al salir a: conteo = 0 -> libera
```

Copiar un `shared<T>` (`b = a`, pasarlo por valor, o guardarlo en un campo)
incrementa el conteo; destruir cada copia lo decrementa.  Cuando llega a cero, el
recurso se libera al instante.  El conteo es **no atomico**: el modelo es
intra-hilo (para compartir entre hilos, usar futures/mailbox).

## Construccion y deleters

```vex
unique<i32>  u = unique_box(42);         // deleter por defecto: free
shared<i64>  s = shared_box(100);

// Deleter personalizado para adoptar cualquier recurso del SO.  El segundo
// argumento es un NOMBRE de funcion de aridad 1 (Vesta o extern), no una llamada.
unique<i64>  g = unique_with(VirtualAlloc(0, 4096, 0x3000, 0x04), liberar_vmem);
```

```vex
extern "kernel32.dll" { fn VirtualFree(u64 a, u64 s, u32 t) -> u32; }
void liberar_vmem(i64 p) { VirtualFree(p, 0, 0x8000); }   // wrapper de 1 arg
```

El deleter por defecto libera la memoria del recurso; uno personalizado puede
cerrar un fichero, liberar una pagina, cerrar un socket o un handle del SO.

## Acceso e inspeccion

```vex
i32* raw = ptr_of(p);    // T*: el puntero crudo, sin consumir ni mover
i32 v = *ptr_of(p);
i64 n = use_count(s);    // conteo de referencias actual de un shared<T>
```

`ptr_of` no transfiere ownership ni modifica el smart pointer.  Para acceso
temporal de solo-lectura o lectura/escritura sin mover, usa `borrow<T>` /
`borrow_mut<T>` (ver [`BorrowChecker.md`](BorrowChecker.md)).

## Smart pointers en campos y contenedores

Un contenedor (clase o struct) con un campo `unique<T>` o `shared<T>` se vuelve
destructible: al destruirse, libera el recurso del campo.  Para `unique<T>` el
campo posee el unico recurso (lo libera el destructor del contenedor; ver
[`InteropC.md`](InteropC.md)); para `shared<T>` el campo es un dueno mas (al
guardar incrementa el conteo, al destruir el contenedor lo decrementa).

## Como funciona por dentro

El modelo es el de los lenguajes con destructores (C++, Rust, Swift): el smart
pointer es un value-type cuya **destruccion** y **copia** las gestiona el
compilador llamando a dos operaciones del tipo, sin recolector de basura:

- **Destruccion (drop).**  Al salir de scope, al destruirse un contenedor que lo
  tiene como campo, o tras un *move*, el compilador invoca la liberacion del
  tipo.  Para `unique<T>` ejecuta el deleter (free, o el personalizado) sobre el
  recurso.  Para `shared<T>` decrementa el conteo de referencias y, si llega a
  cero, ejecuta el deleter y libera el bloque de control.

- **Copia.**  `unique<T>` no define copia: copiarlo es error (move-only).
  `shared<T>` define una copia que incrementa el conteo de referencias, asi que
  asignar o pasar por valor un `shared<T>` produce un dueno adicional del mismo
  bloque.

- **Move.**  Transferir el ownership (`move`, `return`) invalida el origen y
  suprime su destruccion, de modo que solo el destino libera.  Coste cero (sin
  copia ni ajuste de conteo).

El bloque de control de `shared<T>` (`[conteo | deleter | recurso]`) vive en el
heap nativo y se libera explicitamente cuando el conteo cae a cero -- sin
intervencion del recolector.  Por eso `shared<T>` es deterministico y funciona en
binarios nativos sin runtime.

El comportamiento es identico en los tres backends: interprete, JIT y AOT nativo.

## Limitaciones

- **Ciclos.**  El conteo de referencias no recupera ciclos (`A` referencia a `B`
  y `B` a `A` via `shared<T>`): ese grupo nunca llega a cero y se fuga.  Es la
  limitacion estandar del conteo de referencias.  Para romper ciclos, usa
  `unique<T>`/`borrow<T>`, o un puntero crudo `T*` que no cuenta (cuidando el
  dangling).  No existe todavia un `weak<T>`.
- **Conteo no atomico.**  El refcount de `shared<T>` no usa operaciones atomicas;
  el modelo es intra-hilo.  Para compartir entre hilos, usa futures/mailbox.
- **Deleters personalizados null-safe.**  Tras un *move* el origen queda
  invalidado; el deleter por defecto (free) es null-safe.  Un deleter
  personalizado debe tolerar recibir un recurso ya invalidado (o el cleanup
  comprueba null antes de invocarlo).
