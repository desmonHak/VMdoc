# Vex - El lenguaje de alto nivel de VestaVM

**Vex** es el lenguaje de alto nivel que compila a bytecode de VestaVM. Es un lenguaje
multi-paradigma estaticamente tipado disenado con tres principios:

1. **Expresivo**: sintaxis C/Java/Python combinada. Sin burocracia de clase obligatoria.
2. **Seguro**: tipos estrictos, nullability explicita, manejo de errores sin excepciones
 implicitas.
3. **Performante**: baja directamente al IR SSA sin capas intermedias; comparte backend
 con el JIT y el futuro AOT nativo.

---

## Indice de documentacion Vex

### Sintaxis y semantica del lenguaje

| Documento | Contenido |
| :------------------------------ | :----------------------------------------------------------- |
| [[TiposDatos]] | Tipos primitivos, punteros, struct, enum, aliases, newtypes (@opaque/@align/explicit) |
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

### Sistema de modulos (Phase M)

| Documento | Contenido |
| :------------------------------ | :----------------------------------------------------------- |
| [[Modulos]] | `import`, public/private, paquete-dir, reexport, cache, paralelismo M8 |
| [[CompilacionCondicional]] | `@Target(...)` con OS/arch/CPU/semver/mode + AND/OR/NOT/parens + sobre imports |
| [[CargaDinamica]] | `loadmodule`/`unloadmodule`, hot-reload, transitividad caps |
| [[Sandbox]] | Capability-based sandbox (10 caps + whitelists), `--vex-caps`, zero overhead |
| [[PackageManager]] | `vm pkg` (Phase PM): vex.toml/json, vex.lock, firmas ed25519, anti-malware by design |

---

## Pipeline de compilacion

```
.vex fuente
    |
    v VPP preprocesador (#define #include #if #foreach)
    |
    v Lexer Vex (tokens, strings interpolados, literales)
    |
    v Parser Vex (AST: decls, stmts, exprs, tipos)
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

El flag `--vex-emit-ir` guarda el SSA IR en `<modulo>.ir` antes de emitir `.velb`.
El flag `--diagram-all` ademas genera diagramas Mermaid (`.ast.mmd`, `.ir.pre.mmd`,
`.ir.post.mmd`, `.vel.mmd`) utiles para depurar el pipeline.

---

## Primer programa

```java
// hola.vex
i32 main(string[] args) {
    println("Hola desde Vex ${1 + 1}!");
    return 0;
}
```

Compilar y ejecutar:

```bash
./vm --vex hola.vex -o hola
./vm --run hola.velb
```

Salida esperada:

```
Hola desde Vex 2!
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
string nombre = "Vex";
string msg = "Hola ${nombre}!"; // interpolacion: "Hola Vex!"
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
import std.io; // modulo Vex estandar (dot-separated)
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
string nombre = "Vex";

// Tipos soportados en ${...}:
println("Entero: ${x}"); // "Entero: 42"
println("Float: ${pi}"); // "Float: 3.141590"
println("String: ${nombre}"); // "String: Vex"
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

El preprocesador VPP corre como primera etapa antes del lexer Vex:

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

El `asm { }` inline dentro de funciones normales esta reservado para Phase D (MachineIR).

---

## Anotaciones del compilador

| Anotacion | Efecto |
| :---------------- | :------------------------------------------------------------ |
| `@Override` | Obligatorio al sobrescribir un metodo; error si esta ausente |
| `@Inline` | Expande el cuerpo en el call site (sin CALLVIRT) |
| `@Async` | La funcion retorna un Future implicito |
| `@Sync` | El cuerpo de un metodo se ejecuta con `synchronized(this)` |
| `@Aspect` | La clase define consejos AOP (BEFORE/AFTER/AROUND) |
| `@Final` | Alias del keyword `final` sobre clase o metodo |
| `@Asm` | El cuerpo de la funcion es ensamblador .vel verbatim |
| `@Naked` | Sin prologo/epilogo (placeholder, efecto completo en Phase D) |
| `@Module(pkg.x)` | Declara el modulo activo para el archivo |
| `@Export(Symbol)` | Marca el simbolo como publicamente accesible |
| `@Generic(T)` | Registra parametro de tipo para monomorphizacion |

---

Ver tambien: [[IR/SSA]], [[SetInstruccionesVM/STRINGS]], [[SetInstruccionesVM/GC/GC]]
