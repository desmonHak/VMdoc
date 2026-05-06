# Manejo de excepciones en Vex

Vex ofrece un modelo de excepciones estructurado con tres capas:

1. **Excepciones de usuario** (`throw` / `try-catch`): control de flujo de alto nivel.
2. **FatalError predefinido**: errores del runtime (NPE, illegal opcode, etc.) que se
   pueden capturar.
3. **`panic`**: error fatal iniciado por el programador, capturable con `FatalError`.

---

## try / catch / finally

```java
try {
    i32 resultado = dividir(10, 0);
    println("Resultado: ${resultado}");
} catch (MiExcepcion e) {
    println("Capturada MiExcepcion: ${e.mensaje}");
} catch (FatalError e) {
    println("Error fatal: ${e.kind}");
    print_cstr(e.message);
} finally {
    println("Este bloque siempre se ejecuta");
}
```

### Multi-catch

```java
try {
    operacion_riesgosa();
} catch (ErrorDeRed e) {
    println("Error de red: ${e.codigo}");
} catch (ErrorDeDisco e) {
    println("Error de disco");
} catch (FatalError e) {
    println("Error fatal del runtime");
}
```

---

## throw

```java
class MiExcepcion {
    public string mensaje;
    public i32    codigo;

    public MiExcepcion(string msg, i32 cod) {
        this.mensaje = msg;
        this.codigo  = cod;
    }
}

void validar(i32 valor) {
    if (valor < 0) {
        throw new MiExcepcion("Valor negativo", -1);
    }
}

try {
    validar(-5);
} catch (MiExcepcion e) {
    println("Error: ${e.mensaje} (codigo ${e.codigo})");
}
```

---

## FatalError (clase predefinida del runtime)

El runtime de VestaVM lanza `FatalError` cuando ocurre un error irrecuperable. Desde
Vex se puede capturar como cualquier otra excepcion:

```java
class Servicio {
    public i32 metodo(Object obj) {
        return obj.obtenerValor();    // puede lanzar FatalError si obj es null
    }
}

try {
    Servicio s = new Servicio();
    i32 r = s.metodo(null);          // NPE: obj es null
} catch (FatalError e) {
    // Campos de FatalError:
    i32    kind       = e.kind;       // codigo de error (FatalKind)
    u64    pc         = e.pc;         // PC donde ocurrio el error
    char*  message    = e.message;    // descripcion textual (host pointer)
    char*  stack_trace = e.stack_trace; // stack trace (host pointer)

    println("Error ${e.kind} en PC ${e.pc}");
    print_cstr(e.message);            // imprimir la descripcion
    print_cstr(e.stack_trace);        // imprimir el stack trace
}
```

### Codigos de FatalKind

| Codigo | Constante                  | Significado                                |
| :----- | :------------------------- | :----------------------------------------- |
| 0      | `FATAL_NULL_POINTER`       | Desreferencia de puntero/referencia nula   |
| 1      | `FATAL_ILLEGAL_INSTRUCTION`| Opcode invalido o metodo abstracto         |
| 2      | `FATAL_STACK_OVERFLOW`     | Desbordamiento de la pila                  |
| 3      | `FATAL_OUT_OF_MEMORY`      | Sin memoria disponible                     |
| 4      | `FATAL_NATIVE_EXCEPTION`   | Excepcion C++ escapo de un plugin nativo   |
| 5      | `FATAL_NATIVE_CRASH`       | Crash del sistema en plugin nativo (SEH)   |
| 6      | `FATAL_USER_ABORT`         | El programador llamo `panic()`             |
| 7      | `FATAL_CLASS_CAST`         | Cast a tipo incompatible                   |

### Stack trace con file:line

Cuando `FatalError` incluye informacion de debug, el stack trace contiene:

```
Stack trace (Vesta):
  at metodo [Servicio] (src/servicio.vex:42)
  at principal [App] (src/app.vex:10)
  in process pid=0 sched=0
```

El formato por frame es:
```
  at <nombre_metodo> [<nombre_clase>] (<archivo>:<linea>)
```

---

## panic()

El builtin `panic` lanza un `FatalError` con kind `FATAL_USER_ABORT` y el mensaje
proporcionado. Es capturable con `try/catch (FatalError)`.

```java
void verificarIndice(i32 idx, i32 len) {
    if (idx < 0 || idx >= len) {
        panic("Indice fuera de rango: ${idx} (len=${len})");
    }
}

try {
    verificarIndice(100, 10);
} catch (FatalError e) {
    println("Capturado: ${e.kind}");    // FATAL_USER_ABORT = 6
    print_cstr(e.message);              // "Indice fuera de rango: 100 (len=10)"
}
```

---

## assert (macro VPP)

```java
#define ASSERT(cond) if (!(cond)) panic("assertion failed: " #cond)

ASSERT(x > 0);              // lanza panic si x <= 0
ASSERT(ptr != null);        // lanza panic si ptr es null
```

---

## Excepciones en contextos async

Las excepciones dentro de un bloque `spawn { }` no se propagan al proceso padre
automaticamente. Usar `fulfill`/`reject` para comunicar errores:

```java
i64 fut = future_alloc();
i64 fh  = fut;

spawn {
    try {
        i64 r = operacion_riesgosa();
        fulfill(fh, r);
    } catch (FatalError e) {
        // Rechazar el future con un codigo de error:
        reject(fh, (i64)e.kind);
    }
};

i64 resultado = await fut;

// Distinguir exito de error por convencion (bit 63):
if ((resultado >> 63) != 0) {
    println("El hijo fallo con codigo ${resultado & 0x7FFFFFFFFFFFFFFF}");
} else {
    println("Exito: ${resultado}");
}
```

---

## Excepciones con synchronized

`synchronized` garantiza que el monitor se libere incluso si se lanza una excepcion:

```java
// try/finally implicito: el compilador emite tryleave + monexit antes de cualquier salida
synchronized (obj) {
    operacion_que_puede_fallar();   // si lanza, el monitor se libera igualmente
}
```

Si se necesita capturar la excepcion dentro del synchronized:

```java
synchronized (obj) {
    try {
        operacion_riesgosa();
    } catch (MiExcepcion e) {
        manejar_error(e);
        // El monitor se libera al salir del bloque synchronized
    }
}
```

---

## ExceptionFrame: estructura interna

Cada `tryenter` empuja un `ExceptionFrame` en la pila de excepciones del proceso:

```java
struct ExceptionFrame {
    uint64_t   handler_pc;  // PC del bloque catch
    ClassInfo *type;         // tipo a capturar (nullptr = catch-all)
    ExceptionFrame *prev;    // frame anterior
};
```

`tryleave` desapila el frame al salir del bloque try normalmente.
`do_throw` busca el primer frame cuyo tipo sea compatible y salta a `handler_pc`.

---

## Restricciones conocidas

- Las variables de tipo CLASS declaradas dentro de un bloque `try/catch` no tienen
  liberacion automatica de GcHandle (RAII no aplica en presencia de handlers de
  excepcion). Workaround: declarar la variable fuera del try o usar `~Class()` explicito.
- `@Async` + excepcion dentro del cuerpo: la excepcion no llega al llamante. Usar
  `fulfill`/`reject` para comunicar el error via future.

---

Ver tambien: [[OOP]], [[Async]], [[SetInstruccionesVM/TRYENTER_TRYLEAVE]],
[[SetInstruccionesVM/OOP/THROW y RETHROW]]
