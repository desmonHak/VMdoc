# Vesta - El lenguaje de alto nivel de VestaVM

**Vesta** es el lenguaje de alto nivel que compila a bytecode de VestaVM. Es un lenguaje
multi-paradigma estaticamente tipado disenado con tres principios:

1. **Expresivo**: sintaxis C/Java/Python combinada. Sin burocracia de clase obligatoria.
2. **Seguro**: tipos estrictos, nullability explicita, manejo de errores sin excepciones
 implicitas.
3. **Performante**: baja directamente al IR SSA sin capas intermedias; comparte backend
 con el JIT y el futuro AOT nativo.

---

## Indice de documentacion Vesta

### Sintaxis y semantica del lenguaje

| Documento | Contenido |
| :------------------------------ | :----------------------------------------------------------- |
| [[TiposDatos]] | Tipos primitivos, punteros, struct, enum, aliases, newtypes (@opaque/@align/explicit) |
| [[Enums]] | Uniones etiquetadas (ADT) y enums con valor C-style (backing int/float/string/struct/clase), concepts (is_enum/Enum/ValuedEnum), comptime |
| [[Operadores]] | Aritmeticos, comparacion, logicos, bitwise, compound, cast |
| [[Matematicas]] | Funciones integradas: sqrt/pow/trig/log/abs/min/max/clamp + bit ops + rotaciones |
| [[Vectorizacion]] | Auto-vectorizacion SSE2/AVX2/AVX512: patrones, tipos, como exprimir el SIMD |
| [[ControlFlow]] | if/while/do-while/for/foreach, break/continue/goto, match |
| [[Strings]] | Tipo string, interpolacion `${expr:fmt}`, triple-quoted, FFI |
| [[OptionalResult]] | `Optional<T>`, `Result<V,E>`, `!!`, `nonnull`, `T !!name` |
| [[Closures]] | Lambdas, captura lexica, HOF, top-level fn promotion |

### Modelo de programacion

| Documento | Contenido |
| :------------------------------ | :----------------------------------------------------------- |
| [[OOP]] | Clases, herencia, interfaces, propiedades, modificadores |
| [[Generics]] | `class Box<T>`, monomorphizacion compile-time, specialize |
| [[ReflexionAOP]] | forName, getClass, getField, getMethod, invoke, @Aspect |
| [[Metaprogramacion]] | @Macro, expr capture, introspeccion, FFI compile-time |
| [[Colecciones]] | ArrayList, HashMap, HashSet, Queue, Deque, TreeMap, Stack |
| [[Excepciones]] | try/catch/finally, FatalError, panic, throw |

### Memoria y seguridad

| Documento | Contenido |
| :------------------------------ | :----------------------------------------------------------- |
| [[SmartPointers]] | `unique<T>`/`shared<T>`, `move`, RAII, deleters custom |
| [[BorrowChecker]] | `borrow<T>`/`borrow_mut<T>`, 4 reglas + F1-F4 (NLL/reborrow) |

### Concurrencia y FFI

| Documento | Contenido |
| :------------------------------ | :----------------------------------------------------------- |
| [[Async]] | spawn, await, Future, IPC (msgsend/msgrecv), distribuido |
| [[Sincronizacion]] | `synchronized (obj)`, monitores, wait/notify/notifyAll |
| [[FFI]] | extern declarativo, ffi_open/sym/call, plugins nativos |
| [[CallbacksNativos]] | `as_native_callback`, thunks x86-64, WndProc, qsort, audio |

### Sistema de modulos

| Documento | Contenido |
| :------------------------------ | :----------------------------------------------------------- |
| [[Modulos]] | `import`, public/private, paquete-dir, reexport, cache, paralelismo M8 |
| [[Namespaces]] | `namespace a.b.c;`, `import a.b.c;`, alias, `internal`, PackageId `@id`, `extension`/`impl` |
| [[CompilacionCondicional]] | `@Target(...)` con OS/arch/CPU/semver/mode + AND/OR/NOT/parens + sobre imports |
| [[CargaDinamica]] | `loadmodule`/`unloadmodule`, hot-reload, transitividad caps |
| [[Sandbox]] | Capability-based sandbox (10 caps + whitelists), `--vx-caps`, zero overhead |
| [[PackageManager]] | `vm pkg`: vx.toml/json, vx.lock, firmas ed25519, anti-malware by design |

### Compilacion nativa

| Documento | Contenido |
| :------------------------------ | :----------------------------------------------------------- |
| [[CompilacionNativa]] | Ejecutables/objetos/`.a`/`.so`/`.dll` nativos, GC en AOT, usar y crear librerias, enlazado estatico sin dependencias |
| [[Enlazador]] | Linker y archivador propios (`vm --link`/`vm --ar`), flags de salida (`--emit`/`--format`/`--no-pie`/`--bin-base`/`--target`/`--freestanding`/`--float-isa`) |
| [[DisposicionSecciones]] | Layout en el propio lenguaje (sustituto de linker scripts): `@section`/`@at`/`@order`, bloques `bytes {}`, `@bits` asm, `.bin` plano, script de enlace en Vesta (`fn link()`) |
| [[InlineAsm]] | Ensamblador inline (`asm {}` + `@Naked`), `register()`, calificadores `volatile`/`nomem`/`preserves_flags`/`pure`, `clobbers`, sustitucion comptime |
| [[Overlays]] | Vistas tipadas sobre memoria binaria: `@offset`/`@element`/`@endian`, bitfields, arrays con stride, `parent<T>`, `offsetof`/`in_bounds`/`extent` |

---

## Pipeline de compilacion

```
.vx fuente
    |
    v VPP preprocesador (#define #include #if #foreach)
    |
    v Lexer Vesta (tokens, strings interpolados, literales)
    |
    v Parser Vesta (AST: decls, stmts, exprs, tipos)
    |
    v Type Checker (inference, aliases, generics, nullability)
    |
    v Lowering (AST -> SSA IR)
    |
    v IR Optimizer (~15 pasadas O2: DCE, copy-prop, TCO, const-fold, CSE,
    | strength reduction, LICM, dead-alloc-elim, inline-loop-header,
    | DSE+SLF, devirt+inline, const-cse, load_narrow, schedule -
    | ver [[SSA]] seccion 9 para detalles)
    v ir_emitter (SSA IR -> .vel ensamblador virtual; emite super-instrucciones
    | cuando aplica: alu3, loadz/loadzh, cmpjmp/cmpjmpu, gcallocp,
    | spawnargs, fulfillhlt - ver [[SUPER_INSTRUCCIONES]])
    v Assembler + Linker (-> .velb bytecode)
    |
    v VM (intérprete threaded computed-goto, ~340 MIPS promedio)
```

El flag `--vx-emit-ir` guarda el SSA IR en `<modulo>.ir` antes de emitir `.velb`.
El flag `--diagram-all` ademas genera diagramas Mermaid (`.ast.mmd`, `.ir.pre.mmd`,
`.ir.post.mmd`, `.vel.mmd`) utiles para depurar el pipeline.

---

## Primer programa

```java
// hola.vx
i32 main(string[] args) {
    println("Hola desde Vesta ${1 + 1}!");
    return 0;
}
```

Compilar y ejecutar:

```bash
./vm --vx hola.vx -o hola
./vm --run hola.velb
```

Salida esperada:

```
Hola desde Vesta 2!
```

---

## Reglas lexicas

### Comentarios

```java
// comentario de linea

/* comentario
de bloque */
```

### Identificadores

Patron: `[A-Za-z_][A-Za-z0-9_]*`

### Literales enteros

| Forma | Ejemplo |
| :----------- | :-------------- |
| Decimal | `42`, `-7` |
| Hexadecimal | `0xFF`, `0x2A` |
| Binario | `0b101010` |
| Octal | `0o52` |

### Literales flotantes

```java
3.14 // f64 implicito
1e-9 // notacion cientifica
0x1.8p+1 // hexadecimal IEEE 754
```

### Literales de caracter

```java
'a' // caracter ASCII
'\n' // escape
'é' // unicode
```

### Literales de cadena

| Tipo | Sintaxis | Interpolacion |
| :------------ | :--------------------- | :-----------: |
| Estandar | `"hola ${expr}"` | Si |
| Raw | `r"sin \escape"` | No |
| Triple-quoted | `"""multi\nlinea"""` | No |

```java
string nombre = "Vesta";
string msg = "Hola ${nombre}!"; // interpolacion: "Hola Vesta!"
string raw = r"ruta\sin\escape"; // sin procesar escapes
string multi = """
Primera linea
Segunda linea
""";
```

---

## Punto de entrada

```java
i32 main(string[] args) {
    // args[0] es el nombre del ejecutable
    return 0;
}
```

`main` es opcional: los archivos sin `main` son importables como modulos pero no son
ejecutables directamente.

---

## Imports

```java
import std.io; // modulo Vesta estandar (dot-separated)
import std.collections.List; // importar tipo especifico
extern import "stdlib/native/io/vesta_io"; // plugin nativo compilado
```

---

## Variables y constantes

```java
i32 x = 10; // variable mutable
const f64 PI = 3.14159; // constante (error de compilacion si se reasigna)

i32 y; // valor indefinido (solo tipos primitivos)
```

La inferencia de tipos se aplica en la mayoria de contextos donde el tipo se puede deducir
del inicializador. El tipo se escribe explicitamente cuando es ambiguo.

---

## Funciones de primer nivel

```java
// Funcion simple
u64 fibonacci(u64 n) {
    if (n < 2) return n;
    return fibonacci(n - 1) + fibonacci(n - 2);
}

// Funcion con multiples tipos de retorno (Result)
Result<i32, string> dividir(i32 a, i32 b) {
    if (b == 0) return Err("division por cero");
    return Ok(a / b);
}

// Expresion bodied (shorthand para funciones de una linea)
i32 doble(i32 x) => x * 2;
```

---

## Condicionales

```java
if (condicion) {
    // rama verdadera
} else if (otra) {
    // rama alternativa
} else {
    // rama default
}
```

---

## Bucles

```java
// while
while (condicion) { ... }

// do-while
do { ... } while (condicion);

// for C-style
for (i32 i = 0; i < 10; i++) { ... }

// foreach Java-style (sobre array)
for (i32 x : arr) { ... }

// foreach Python-style
for (x in col) { ... }

// break y continue estan soportados
for (i32 i = 0; i < 100; i++) {
    if (i == 50) break;
    if (i % 2 == 0) continue;
    println("${i}");
}

// goto y labels (patrones de early-exit)
retry:
i32 resultado = intentar();
if (resultado < 0) goto retry;
```

---

## Operadores

### Aritmeticos

| Op | Descripcion |
| :-- | :----------------- |
| `+` | Suma |
| `-` | Resta |
| `*` | Multiplicacion |
| `/` | Division |
| `%` | Modulo |

### Relacionales y logicos

| Op | Descripcion |
| :--- | :----------------- |
| `==` | Igualdad |
| `!=` | Desigualdad |
| `<` | Menor que |
| `<=` | Menor o igual |
| `>` | Mayor que |
| `>=` | Mayor o igual |
| `&&` | AND logico |
| `\|\|` | OR logico |
| `!` | NOT logico |

### Bit a bit

| Op | Descripcion |
| :--- | :----------------- |
| `&` | AND bit a bit |
| `\|` | OR bit a bit |
| `^` | XOR bit a bit |
| `~` | NOT bit a bit |
| `<<` | Desplazamiento izq |
| `>>` | Desplazamiento der |

### Asignacion compuesta

```java
x += 1; x -= 1; x *= 2; x /= 2;
x %= 3; x &= 0xFF; x |= 0x01; x ^= 0x10;
x <<= 2; x >>= 1;
```

Todos funcionan sobre lvalues no triviales: `obj.campo += v`, `arr[i] *= 2`, `*p &= mask`.

### Operadores de puntero y nulidad

| Op | Descripcion |
| :------- | :-------------------------------------------------- |
| `&x` | Direccion de x (`VirtualPtr<T>` o `T*` segun contexto) |
| `*p` | Desreferencia de puntero |
| `p[i]` | Subscript con escalado sizeof |
| `!!a` | Unwrap de Optional/nullable (lanza si null) |
| `p->f` | Desreferencia + acceso a campo (azucar para `(*p).f`) |

---

## Interpolacion de cadenas

```java
i32 x = 42;
f64 pi = 3.14159;
string nombre = "Vesta";

// Tipos soportados en ${...}:
println("Entero: ${x}"); // "Entero: 42"
println("Float: ${pi}"); // "Float: 3.141590"
println("String: ${nombre}"); // "String: Vesta"
println("Expr: ${x * 2 + 1}"); // "Expr: 85"
println("Bool: ${x > 0}"); // "Bool: true"
```

---

## Codigos ANSI integrados

Identificadores magicos disponibles directamente en el scope global:

```java
println("${RED}Error:${RESET} algo salio mal");
println("${GREEN}OK${RESET} - operacion exitosa");
println("${BOLD}Titulo${RESET}");
println("${UNDERLINE}subrayado${RESET}");
```

| Identificador | Secuencia ANSI |
| :------------ | :---------------------- |
| `BLACK` | `\x1b[30m` |
| `RED` | `\x1b[31m` |
| `GREEN` | `\x1b[32m` |
| `YELLOW` | `\x1b[33m` |
| `BLUE` | `\x1b[34m` |
| `MAGENTA` | `\x1b[35m` |
| `CYAN` | `\x1b[36m` |
| `WHITE` | `\x1b[37m` |
| `BOLD` | `\x1b[1m` |
| `DIM` | `\x1b[2m` |
| `ITALIC` | `\x1b[3m` |
| `UNDERLINE` | `\x1b[4m` |
| `BLINK` | `\x1b[5m` |
| `REVERSE` | `\x1b[7m` |
| `RESET` | `\x1b[0m` |
| `BG_RED` | `\x1b[41m` |
| `BG_GREEN` | `\x1b[42m` |
| `BG_BLUE` | `\x1b[44m` |
| `CLEAR_SCREEN`| `\x1b[2J` |
| `CURSOR_HOME` | `\x1b[H` |

---

## Builtins de I/O

```java
print("texto sin newline");
println("texto con newline");
flush(); // vacia el buffer de salida (64 KB)
print_uint(u64_val);
print_hex(u64_val);
print_float(f64_bits);
print_bool(b);
print_char(cp); // codepoint UTF-32
print_color(codigo_ansi);
```

---

## Preprocesador VPP

El preprocesador VPP corre como primera etapa antes del lexer Vesta:

```java
#define MAX_SIZE 1024
#define ASSERT(cond) if (!(cond)) panic("assertion failed: " #cond)

#include "common.vhx"

#if defined(WINDOWS)
// codigo especifico de Windows
#endif

#foreach(i, 0, 8)
println("linea ${i}");
#end
```

Ver [[Preprocesador/Macros parametricas]] para la documentacion completa del preprocesador.

---

## Inline assembly

```java
// Whole-function assembly con @Asm:
@Asm
u64 fast_add(u64 a, u64 b) {
    add r0, r1 // r0 = a + b (r0 es el registro de retorno)
    ret
}
```

El cuerpo se emite verbatim al flujo `.vel`. Util para operaciones que el compilador no
puede expresar eficientemente o para acceder a instrucciones VM avanzadas.

El `asm { }` inline dentro de funciones normales tambien esta soportado, en
sintaxis NASM Intel + storage-class `register("reg")` para ligar variables a
registros.  El cuerpo se ensambla a codigo nativo host y corre en los tres modos
de ejecucion (por defecto, `-m jit` y `-m vm` interprete puro, este ultimo via un
trampolin con marshalling de los `register()`).  Los registros y flags que el
bloque pisa se INFIEREN automaticamente (incluidos callee-saved y los reservados
por el runtime); declarar `clobbers(...)` es opcional.  Las `comptime` consts se
sustituyen por su literal en el cuerpo antes de ensamblar.  Ver los ejemplos en
`examples_codes_vx/asm/`.

---

## Anotaciones del compilador

Las anotaciones llevan prefijo `@`. Por convencion las que estan en PascalCase
(`@Override`, `@Async`, ...) declaran comportamiento, y las que estan en minuscula
(`@align`, `@hot`, `@pure`, ...) son atributos y contratos. Una anotacion no
reconocida se ignora silenciosamente (no es error, pero tampoco tiene efecto).

### OOP y aspectos (AOP)

| Anotacion | Ambito | Efecto |
| :-------- | :----- | :----- |
| `@Override` | metodo | Obligatorio al sobrescribir un metodo; error de compilacion si esta ausente |
| `@Aspect` | clase | La clase define consejos AOP; ver [[ReflexionAOP]] |
| `@Before("Cls.metodo")` | metodo (en `@Aspect`) | Registra un consejo BEFORE sobre el metodo objetivo |
| `@After("Cls.metodo")` | metodo (en `@Aspect`) | Registra un consejo AFTER |
| `@AfterReturning("Cls.metodo")` | metodo (en `@Aspect`) | Registra un consejo tras el retorno normal |
| `@Around("Cls.metodo")` | metodo (en `@Aspect`) | Registra un consejo AROUND (envuelve la llamada) |

`final` (clase no heredable / metodo no sobrescribible) es una **palabra clave**,
no una anotacion; se escribe `final class C { ... }`. Ver [[OOP]].

### Codegen e inline assembly

| Anotacion | Ambito | Efecto |
| :-------- | :----- | :----- |
| `@Inline` | metodo | Marca el metodo para expandirse en el call site (sin CALLVIRT) |
| `@Asm` | funcion | El cuerpo es ensamblador `.vel` verbatim (ver "Inline assembly" arriba) |
| `@Naked` | funcion | Sin prologo ni epilogo; para rutinas de interrupcion (ISRs) y stubs nativos |

### Concurrencia

| Anotacion | Ambito | Efecto |
| :-------- | :----- | :----- |
| `@Async` | funcion | La funcion retorna un `Future` implicito (envuelta en future + spawn); ver [[Async]] |

### Metaprogramacion y reflexion

| Anotacion | Ambito | Efecto |
| :-------- | :----- | :----- |
| `@Macro` | funcion | Funcion macro evaluada en tiempo de compilacion; ver [[Metaprogramacion]] |
| `@Pure` | macro | Marca un `@Macro` como memoizable: mismos argumentos devuelven el resultado cacheado. Distinto del contrato `@pure` en minuscula (ver seccion de contratos) |
| `@Introspect` | clase/struct/enum | Genera metadata de introspeccion en runtime (reflexion sobre campos y metodos del tipo) |

### Propiedades y boilerplate de clases (estilo Lombok)

Se declaran a nivel de clase (o de campo cuando aplica). Un pre-pase del
compilador genera los metodos correspondientes en el AST, como si se hubieran
escrito a mano.

| Anotacion | Ambito | Efecto |
| :-------- | :----- | :----- |
| `@Getter` / `@Setter` | clase o campo | Genera `get_X()` / `set_X(v)` para los campos |
| `@Getter(lazy=true)` | campo | Getter perezoso: calcula el valor una vez y lo cachea |
| `@NonNull` | campo | Reescribe el tipo del campo a `nonnull T` (rechazo de null en compile-time) |
| `@With` | clase o campo | Genera `with_X(v)`, que devuelve una copia de la instancia con `X = v` |
| `@ToString` | clase | Genera `string toString()` |
| `@EqualsAndHashCode` | clase | Genera `bool equals(Object o)` + `u64 hashCode()` |
| `@NoArgsConstructor` | clase | Constructor sin argumentos |
| `@AllArgsConstructor` | clase | Constructor con todos los campos |
| `@RequiredArgsConstructor` | clase | Constructor con los campos `final` / `nonnull` |
| `@Data` | clase | `= @Getter + @Setter + @ToString + @EqualsAndHashCode + @RequiredArgsConstructor` |
| `@Value` | clase | `= @Data` inmutable (todos los campos `final`) |
| `@Builder` | clase | Genera una clase auxiliar `XBuilder` con metodos encadenables |
| `@Synchronized` | clase | Envuelve cada metodo no-static en `synchronized(this) { ... }` |
| `@Log` | clase | Anade un logger estatico a la clase |

### Excepciones

| Anotacion | Ambito | Efecto |
| :-------- | :----- | :----- |
| `@NoExcept` | funcion | La funcion no usa excepciones; un fallo no capturable termina el proceso. Ver [[Excepciones]] y [[RuntimeHooks]] |
| `@NoExceptions` | modulo | Sticky: todo el fichero queda sin excepciones (equivale a marcar `@NoExcept` en cada funcion) |

### Hooks de runtime (compilacion sin runtime)

El programador provee la implementacion de un servicio del runtime. La funcion
anotada reemplaza el default. Ver [[RuntimeHooks]] para el detalle y las firmas.

| Anotacion | Ambito | Efecto |
| :-------- | :----- | :----- |
| `@AllocatorOverride` | funcion | Reemplaza el allocator (equivalentes a `malloc` / `free`) |
| `@PanicHandler` | funcion | Reemplaza el handler de `panic` |
| `@SyncImpl` | funcion | Reemplaza las primitivas de monitor (enter/exit) |
| `@StringConcat` | funcion | Reemplaza la concatenacion de strings |
| `@StringEq` | funcion | Reemplaza la comparacion de igualdad de strings |
| `@HelperOverride(nombre)` | funcion | Reemplaza un helper multi-versionado del build (p.ej. `memcpy`) |

### Atributos de layout y seccion

Se aplican a funciones, constantes `comptime`, bloques `bytes` y bloques `asm`.
Controlan como se coloca el simbolo en la imagen nativa.

| Anotacion | Efecto |
| :-------- | :----- |
| `@align(N)` | Alineacion del simbolo; `N` potencia de 2 en `[1,4096]` |
| `@hot` / `@cold` | Hint de layout: codigo/datos calientes vs frios |
| `@section("nombre"[, "rwx"])` | Coloca el simbolo en una seccion nativa; 2do argumento opcional = permisos de pagina |
| `@at(N)` | Offset/direccion fija de la seccion en la imagen (binarios planos) |
| `@order(N)` | Orden relativo de la seccion en la imagen (menor primero) |
| `@bits(16\|32\|64)` | Bitness de un bloque `asm` ensamblado (modo real/protegido/largo) |

### Compilacion condicional

| Anotacion | Efecto |
| :-------- | :----- |
| `@Target("...")` | Descarta la declaracion (o el `import`) si el target actual no cumple la condicion (OS/arch/CPU/semver/mode, con `&&` / `\|\|` / `!` / parentesis). Ver [[CompilacionCondicional]] |

### Otras anotaciones contextuales

| Anotacion | Contexto | Efecto |
| :-------- | :------- | :----- |
| `@id("...")` | tras `namespace` | Fija el PackageId del namespace manteniendo la identidad ABI; ver [[Namespaces]] |
| `@opaque` | `typedef ... new` | Newtype opaco (no cruza la barrera de conversion sin `explicit`); ver [[TiposDatos]] |
| `@align(N)` | `typedef ... new` | Alineacion del newtype; ver [[TiposDatos]] |

> Modulos, exports y genericos NO se declaran con anotaciones: se usan las
> construcciones del lenguaje `namespace` / `public` / `class Box<T>`
> respectivamente (ver [[Namespaces]], [[Modulos]], [[Generics]]).

---

## Contratos comprobables de recurso y efecto (huella computacional)

Vesta permite adjuntar a una funcion una **huella computacional**: propiedades
de recurso y efecto que forman parte de su "tipo" y que el compilador
**verifica**. No son comentarios ni documentacion pasiva. El compilador infiere
del codigo la huella REAL de la funcion (cuantas allocaciones hace, cuanto stack
usa, si es pura, si puede lanzar o hacer panic, si es recursiva) y la compara
contra lo declarado.

Los contratos son anotaciones en minuscula:

| Contrato | Significado |
| :------- | :---------- |
| `@pure` | La funcion no tiene efectos de dato observables |
| `@nothrow` | No hay ningun `throw` alcanzable |
| `@nopanic` | No hay ningun `panic` alcanzable |
| `@alloc(N)` | A lo sumo `N` sitios de allocacion en heap (GC / raw / `new` / closure) |
| `@stack(N)` | A lo sumo `N` bytes de pila (ver dimensiones abajo) |
| `@complexity(O(...))` | Contrato de coste asintotico (Big-O); ver mas abajo |

Se declaran sobre funciones libres y sobre los **metodos** de un struct o una
clase, y admiten un `when:` para condicionarlos a la arquitectura, al sistema o
al parametro de tipo (ver mas abajo).

**`@alloc` y `@stack` tienen dos dimensiones**, como `@complexity`: **parcial**
(lo que la funcion hace por si misma) y **total** (incluyendo lo que llama).

- `@stack` parcial = el marco de pila PROPIO de la funcion; total = la
  profundidad de pila PEOR CASO del arbol de llamadas (el propio marco mas el
  camino de llamadas mas hondo, no la suma de todos).
- `@alloc` parcial = los sitios de allocacion PROPIOS; total = los del cierre
  transitivo alcanzable.

La forma corta `@stack(N)` / `@alloc(N)` fija el **total** (el peor caso que
importa desde fuera, como `@complexity(O(..))` es azucar de `total_post`).  La
forma nombrada declara cualquiera de las dos, o ambas:

```java
@stack(0)                       // total 0 (no usa pila ni la de sus callees)
@stack(partial: 32, total: 160) // 32 propio, 160 con la cadena de llamadas
@alloc(partial: 0, total: 2)    // no aloca propio, 2 en lo que llama
```

El parcial es EXACTO (siempre verificable).  El total de `@stack` puede quedar
**inverificable** (`?`) si la profundidad no es acotable -- hay recursion en el
callgraph o un callee externo cuyo marco no se ve -- y entonces no se afirma
ninguna cota (nunca un falso VIOLATED).

### Verificacion: OK, VIOLATED, UNVERIFIABLE

Al compilar, cada contrato declarado se resuelve a uno de tres estados:

- **OK** — el compilador PRUEBA que el contrato se cumple.
- **VIOLATED** — el compilador PRUEBA que el contrato NO se cumple. Es un error
  de compilacion: el build se aborta con un mensaje del tipo
  `contrato @alloc incumplido en 'crea': esperado <=0, inferido 1`.
- **UNVERIFIABLE** — el compilador no puede decidir (hay una llamada dinamica o
  externa opaca en el cierre de la funcion). No se afirma ni se niega el
  contrato: el compilador nunca miente.

La verificacion es **sound y asimetrica**. Las propiedades exactas
(allocaciones, stack, throw/panic, pureza) son un contrato duro: solo se marca
`VIOLATED` cuando la violacion es demostrable, y nunca hay un falso `VIOLATED`.
La complejidad `@complexity` es asimetrica: solo se rechaza cuando el coste
inferido es demostrablemente de otra clase que el declarado.

### Composicion por el grafo de llamadas

Las propiedades se **componen** de forma interprocedural: se agregan la funcion
mas su cierre transitivo por el grafo de llamadas estatico. Una funcion es pura
solo si TODO lo que alcanza es puro; sus allocaciones totales son las propias
mas las de sus callees; etc. Si en ese cierre aparece una llamada dinamica
(metodo virtual, closure) o a un simbolo externo no resuelto, los efectos dejan
de conocerse del todo y los totales se vuelven conservadores: nunca se afirma
`@nothrow` o `@alloc(0)` sin poder demostrarlo.

```java
@pure @nothrow @nopanic @alloc(0)
i64 cuadrado(i64 x) {
    return x * x;                 // sin efectos: las cuatro propiedades se prueban
}

// PUREZA POR COMPOSICION: 'combina' no tiene efectos propios y solo llama a
// 'cuadrado', que es pura -> el compilador deduce que 'combina' es pura.
@pure @alloc(0) @nothrow
i64 combina(i64 a, i64 b) {
    return cuadrado(a) + cuadrado(b);
}

// @alloc(0): garantiza que el bucle no reserva heap (hot path sin GC).
@alloc(0) @nothrow @nopanic
i64 suma_hasta(i64 n) {
    i64 acc = 0;
    i64 i = 0;
    while (i < n) { acc = acc + i; i = i + 1; }
    return acc;
}

// @stack(N): acota los bytes de marco propio.
@stack(256) @alloc(0)
i64 poligono(i64 lado, i64 n) => lado * lado * n;
```

Ejemplos de violacion (abortan la compilacion a proposito):

```java
@alloc(0)
Punto crea() { return new Punto(); }
//  -> error: contrato @alloc incumplido en 'crea': esperado <=0, inferido 1

@pure
void escribe(i64* p) { *p = 5; }
//  -> error: contrato @pure incumplido en 'escribe': la funcion tiene efectos

@nopanic
i64 fuerza(i64 x) { if (x < 0) { panic("negativo"); } return x; }
//  -> error: contrato @nopanic incumplido en 'fuerza': puede hacer panic
```

### El contrato de coste `@complexity`

`@complexity(O(...))` declara el coste asintotico (Big-O) de una funcion. Es
azucar de la dimension "total tras retorno" del subsistema de coste. Dos formas:

```java
@complexity(O(n^2))
void ordena(i32[] a) { ... }

// Nombrar la variable del tamano cuando no es evidente:
@complexity(O(n), n = len(arg0))
i64 suma(i64[] xs) { ... }
```

El compilador infiere la clase de coste de la funcion y valida cada dimension
declarada; si la clase inferida es demostrablemente distinta de la declarada,
lo reporta como discrepancia.

`O(...)` a secas es azucar de una de las **cuatro** dimensiones, `total_post`.
Las cuatro salen de cruzar PARCIAL (el cuerpo, contando cada llamada como O(1))
con TOTAL (interprocedural), y PRE-opt (la complejidad del fuente) con POST-opt
(la del codigo final, tras inline y demas). Se declaran por separado:

```java
@complexity(partial_pre: O(1), partial_post: O(n),
            total_pre: O(n), total_post: O(n))
i64 f(...) { ... }
```

Que difieran no es raro ni es un fallo: `partial_post` sale O(n) donde
`partial_pre` es O(1) cuando el optimizador inlinea un callee que tiene un
bucle -- post-inline ese bucle esta en el cuerpo, no detras de una llamada.

### Contratos en metodos

Valen igual sobre los metodos de un struct o una clase, que es donde mas falta
hacen: la API de un tipo son sus metodos, y sus propiedades ("no aloca", "no
lanza", "es O(1)") son justo lo que decide si se puede usar desde un ISR, un
allocator o codigo freestanding.

```java
struct Contador {
    i64 n;

    @pure
    @nothrow
    @nopanic
    @alloc(0)
    @stack(0)
    @complexity(partial_pre: O(1), partial_post: O(1),
                total_pre: O(1), total_post: O(1))
    public i64 leer() { return this.n; }
}
```

En un tipo generico, el contrato de la plantilla es una promesa para **cada**
instanciacion: se verifica una vez por cada una, y el error nombra la culpable
(`contrato @alloc incumplido en 'Caja_i64__desc'`).

### `when:` -- el contrato que depende del target o del tipo

Un mismo codigo no siempre cuesta lo mismo:

- El coste **total** cambia con la arquitectura si algun callee tiene cuerpos
  por-arch distintos. `atomic<T>::swap` de la stdlib llama a
  `vx_atomic_swap64`, que en x86-64 es un bucle CAS escrito en Vesta (no hay
  instruccion de exchange) y en arm64 el LL/SC nativo: O(n) contra O(1).
- El coste de un metodo generico cambia con el **parametro de tipo** si el
  cuerpo tiene un `if` que la introspeccion resuelve en comptime.
  `atomic<i64>::fetch_add` es un `lock xadd` (O(1)) y `atomic<f64>::fetch_add`
  un bucle CAS (O(n)), del mismo fuente.

Con un solo contrato no se puede decir la verdad de los dos casos a la vez.
`when:` condiciona la anotacion:

```java
// Por arquitectura.
@complexity(total_post: O(n), when: arch:x86_64)
@complexity(total_post: O(1), when: arch:arm64)
public T swap(T v) { ... }

// Por el parametro de tipo.
@complexity(total_post: O(1), when: is_integer<T>())
@complexity(total_post: O(n), when: is_float<T>())
public T fetch_add(T d) { ... }

// Los dos ejes se combinan: es una sola expresion.
@complexity(total_post: O(n), when: arch:x86_64 && is_float<T>())
```

La expresion es la de `@Target` (`os:` / `arch:` / `cpu:` / `mode:` /
`compiler OP M.m` / `vm OP M.m`, con `&&`, `||`, `!` y parentesis) mas los
predicados sobre los type params: `is_float<T>()`, `is_integer<T>()`,
`is_pointer<T>()`, `is_signed<T>()` y `sizeof<T>() OP N`. Va **sin comillas**:
asi un editor la ve como una expresion y puede completarla y validarla, no como
una cadena opaca.

Cada atomo se resuelve donde tiene respuesta: los del target al compilar, los
que hablan de `T` al instanciar el generico. Un atomo que no se entienda es un
error con la lista de los validos, no un contrato descartado en silencio. Sin
`when:`, el contrato aplica siempre.

`when:` sirve en todas: `@alloc(0, when: ...)`, `@nothrow(when: ...)`, etc.

**Varios `when:` que casan a la vez.** Se evaluan TODAS las condiciones y se
aplican todas las que casan. Si dos escriben el mismo campo (p.ej. dos
`@complexity` con `total_post`), gana la **mas especifica**: A es mas especifica
que B si A implica B y B no implica A (se comprueba por tabla de verdad sobre la
union de sus atomos). Si el compilador no puede decidir cual es mas especifica,
es un **error de ambiguedad**, no una eleccion arbitraria. Asi un
`when: arch:x86_64 && is_float<T>()` gana sobre un `when: arch:x86_64` para la
instancia `f64` en x86-64, sin que haga falta ordenarlos a mano.

### Inspeccion: modo analisis

Estos contratos y la huella inferida se muestran con el modo de analisis, sin
generar codigo, y estan pensados tambien para el hover de un editor:

```bash
vm --analyze fichero.vx          # salida legible: huella + estado de cada contrato
vm --analyze fichero.vx --analyze-json   # el mismo analisis como JSON
vm --analyze fichero.vx --analyze-write  # escribe las anotaciones sugeridas al fichero
```

Por cada funcion imprime una linea `Huella:` con las propiedades exactas
(`allocs`, `stack`, `pure`, `throws`, `panics`, `recursion`) y, debajo, el
estado de cada contrato declarado (`OK` / `FALLA` / `?`). Al compilar de verdad
(`vm --vesta fichero.vx -o ...`), un contrato en estado `VIOLATED` aborta el
build. El ejemplo completo esta en `examples_codes_vx/analyze/contracts.vx`.

**`--analyze` genera las anotaciones.** Ademas de verificar, el analisis emite
el bloque de contratos que cada funcion deberia llevar, medido, para copiar y
pegar. El valor ES lo que se mide (no una suposicion): mide en TODAS las
arquitecturas soportadas y, si una instancia generica cambia con `T`, en todas
las clases de tipo, y agrupa las instancias bajo su plantilla
(`atomic<T>::fetch_add`). Donde el valor coincide en todo, sale una linea sin
`when:`; donde difiere, el analisis pone el `when:` MiNIMO que lo describe
(`arch:x86_64`, `is_float<T>()`, o los dos), sin perder la informacion por-arch
ni por-tipo. Las cuatro dimensiones de `@complexity` se miden por separado:

- `partial` = **el cuerpo escrito**, contando cada llamada como O(1). Es una
  propiedad de la funcion: el inline NO lo altera (si lo hiciera, dependeria del
  optimizador y `-O0` daria un valor distinto de `-O3`).
- `total` = interprocedural: el coste propio mas el de lo que llama.
- `pre` = el fuente tal cual; `post` = el codigo final (tras optimizar).

Con `--analyze-write` esas anotaciones se escriben **en el propio fichero**,
sustituyendo las de contrato que ya tuviera cada funcion o metodo definido ahi
(respeta `@Target`/`@Override`/comentarios). Las funciones que vienen de un
`import` no se tocan: se anotan analizando su propio fichero. El reescritor se
aplica sobre su salida antes de guardar y, si no es estable, no toca el fichero.

---

Ver tambien: [[IR/SSA]], [[SetInstruccionesVM/STRINGS]], [[SetInstruccionesVM/GC/GC]]
