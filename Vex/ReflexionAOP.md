# Reflexion y AOP en Vex

Vex expone el `ClassRegistry` del runtime como un API de reflexion de primera clase.
Las clases registradas durante `__module_init` son accesibles en tiempo de ejecucion
sin ninguna generacion de codigo adicional.

La misma infraestructura sustenta AOP (Programacion Orientada a Aspectos): los advices
se instalan en el `MethodInfo::advice_chain` antes del primer `CALLVIRT`, con coste
cero cuando no hay aspectos activos.

---

## Reflexion basica

### `forName(nombre)` — obtener una clase por nombre

```java
Class cls = forName("Animal");

if (cls == null) {
    println("Clase no encontrada");
} else {
    println("Clase encontrada");
}
```

Baja a la instruccion `findclass` (0xCC). O(1) por lookup en `unordered_map<string, ClassInfo*>`
del `ClassRegistry`.

### `getClass(obj)` — obtener la clase de un objeto

```java
Animal a = new Animal("Rex", 5);
Class cls = getClass(a);    // equivalente a a.getClass() en Java
```

Baja a un `LOAD i64` desde `obj + 0` (campo `class_ptr` del `ObjectHeader`). Cero overhead.

### Verificar que dos clases son la misma

```java
Animal a = new Animal("Rex", 5);
Class cls1 = forName("Animal");
Class cls2 = getClass(a);

if (cls1 == cls2) {
    println("Mismo ClassInfo*");    // true: un solo objeto ClassInfo por clase
}
```

Los `ClassInfo*` son canonicos: hay exactamente uno por clase registrada en el `ClassRegistry`.

---

## Reflexion de campos

### `getField(cls, nombre)` — obtener un campo por nombre

```java
Class cls = forName("Animal");
Field edad_field = getField(cls, "edad");

if (edad_field != null) {
    println("Campo 'edad' encontrado");
}

Field noexiste = getField(cls, "xyz");
// noexiste == null (campo no encontrado)
```

Baja a la instruccion `findfield` (0xCF). O(1) amortizado via tabla hash open-addressing
en `ClassInfo::field_lookup_table` (FNV-1a + linear probing, factor de carga 0.5).

El valor devuelto es un `FieldInfo*` opaco (representado como `i64` en el IR). Se puede
comparar con `null` para detectar campos inexistentes.

### Uso combinado forName + getField

```java
Class cls1 = forName("Animal");
Class cls2 = getClass(new Animal("Fido", 3));

// Verificar que es el mismo ClassInfo
bool mismo_tipo = (cls1 == cls2);

// Verificar existencia de campos
Field f1 = getField(cls1, "edad");
Field f2 = getField(cls1, "nombre");
Field f3 = getField(cls1, "noexiste");

// f1 != null, f2 != null, f3 == null
```

---

## Reflexion de metodos

### `getMethod(cls, nombre)` — obtener un metodo por nombre

```java
Class cls = forName("Calculadora");
Method m = getMethod(cls, "doblar");

if (m != null) {
    println("Metodo 'doblar' encontrado");
}
```

Baja a la instruccion `findmethod` (0xCD). O(1) amortizado via tabla hash en
`ClassInfo::method_lookup_table`.

### `invoke(method, this, args...)` — invocar dinamicamente

```java
Class cls    = forName("Calculadora");
Object obj   = newInstance(cls);     // crear instancia sin constructor
Method m     = getMethod(cls, "doblar");
i64 resultado = invoke(m, obj, 21);   // doblar(21) = 42
println("${resultado}");              // 42
```

Baja a la instruccion `callm` (0xFD). Recorre `MethodInfo::advice_chain` igual que
`CALLVIRT` — compatible con metodos que tienen aspectos AOP.

**Firma de `invoke`**: variadic, 1 + N argumentos donde el primero es `this` y los
siguientes son los argumentos del metodo. El tipo de retorno es siempre `i64` (sin
coercion automatica al tipo logico del metodo en el MVP actual).

### Ciclo completo de reflexion

```java
class Calculadora {
    public i32 doblar(i32 x) {
        return x * 2;
    }
}

i32 main(string[] args) {
    // Todo dinamico: sin conocer el tipo en compile time
    Class cls     = forName("Calculadora");
    Object obj    = newInstance(cls);
    Method m      = getMethod(cls, "doblar");
    i64 resultado = invoke(m, obj, 21);
    println("resultado = ${resultado}");   // 42
    return (i32)resultado;
}
```

---

## `newInstance(cls)` — instanciacion sin constructor

```java
Class cls   = forName("Animal");
Object inst = newInstance(cls);    // tipo declarado: i64 (GcHandle/host_ptr)
```

Equivale a `Class.newInstance()` en Java: aloca el objeto en el GC heap con todos los
campos inicializados a cero, SIN invocar ningun constructor. Util cuando:

- El tipo se conoce solo en runtime.
- Se quiere inicializar los campos manualmente via reflexion.
- Se usa como factory en arquitecturas de plugin/hot-reload.

Baja a la instruccion `NEWOBJ` (mismo que `new ClassName()`), pero omitiendo el
`CALLVIRT` al constructor.

---

## Tabla de builtins de reflexion

| Builtin                    | Instruccion bytecode | Coste          | Descripcion                         |
| :------------------------- | :------------------- | :------------- | :---------------------------------- |
| `forName("X")`             | `findclass` 0xCC     | O(1) hash      | ClassInfo* por nombre               |
| `getClass(obj)`            | `LOAD i64 [obj+0]`   | O(1) puntero   | ClassInfo* desde ObjectHeader       |
| `getField(cls, "name")`    | `findfield` 0xCF     | O(1) hash      | FieldInfo* por nombre               |
| `getMethod(cls, "name")`   | `findmethod` 0xCD    | O(1) hash      | MethodInfo* por nombre              |
| `newInstance(cls)`         | `NEWOBJ`             | GC alloc       | Objeto sin constructor              |
| `invoke(m, this, args...)` | `callm` 0xFD         | vtable + chain | Invoke dinamico con AOP             |

---

## AOP — Programacion Orientada a Aspectos

### Declarar un aspecto

```java
@Aspect class Logging {
    // Consejo BEFORE: se ejecuta antes que el metodo objetivo
    @Before("Servicio.calcular")
    public void antes_calcular() {
        println("Entrando a calcular...");
    }

    // Consejo AFTER: se ejecuta despues (valor ya retornado)
    @After("Servicio.calcular")
    public void despues_calcular() {
        println("Saliendo de calcular.");
    }
}

class Servicio {
    public i64 calcular(i64 x) {
        return x * 2;
    }
}

i32 main(string[] args) {
    Servicio s = new Servicio();
    i64 r = s.calcular(21);   // imprime "Entrando..." + ejecuta + imprime "Saliendo..."
    println("${r}");          // 42
    return 0;
}
```

### Consejo AROUND — interceptar y controlar la ejecucion

```java
@Aspect class Cache {
    @Around("Servicio.costoso")
    public i64 cache_costoso() {
        // proceed() invoca el metodo original con los argumentos actuales
        i64 resultado = proceed();
        return resultado;
    }
}
```

El builtin `proceed()` dentro de un consejo `@Around` invoca el metodo objetivo original
y devuelve su resultado. Permite:

- Short-circuit: retornar sin llamar al objetivo.
- Pre/post-procesamiento del valor de retorno.
- Decoradores (memoizacion, timing, logging, rate-limiting).

### Como funciona internamente

```
CALLVIRT r_obj, vtable_slot
    |
    +-- AdviceEntry* chain = method->advice_chain
    |
    +-- chain == NULL?  --> dispatch directo (fast path, overhead CERO)
    |
    +-- chain != NULL?  --> ejecutar cadena:
         1. BEFOREs en orden de registro
         2. Metodo target (o AROUND si existe)
         3. AFTERs en orden inverso de registro
```

El `MethodInfo::advice_chain` es `NULL` por defecto. Solo cuando se registra un aspecto
via `addadvice` (0xCE) se enlaza el primer `AdviceEntry`. Esto garantiza que el codigo
SIN aspectos tiene exactamente el mismo coste que antes de introducir AOP.

### Anotaciones de pointcut

| Anotacion                 | Efecto                                               |
| :------------------------ | :--------------------------------------------------- |
| `@Before("Cls.met")`      | Consejo ejecutado ANTES del metodo objetivo          |
| `@After("Cls.met")`       | Consejo ejecutado DESPUES del metodo objetivo        |
| `@Around("Cls.met")`      | Intercepta; usa `proceed()` para invocar el objetivo |

El pointcut es un string `"NombreClase.nombreMetodo"` exacto (no es glob en el MVP
actual; coincidencia exacta via `findclass` + `findmethod`).

### Ejemplo completo BEFORE + AFTER + AROUND

```java
@Aspect class Instrumentacion {
    @Before("BD.query")
    public void antes_query() {
        println("[Antes] query iniciado");
    }

    @After("BD.query")
    public void despues_query() {
        println("[Despues] query completado");
    }

    @Around("BD.query")
    public i64 medir_query() {
        println("[Around] midiendo...");
        i64 r = proceed();    // invocar BD.query original
        println("[Around] resultado = ${r}");
        return r;
    }
}

class BD {
    public i64 query(i64 param) {
        println("  ejecutando query con ${param}");
        return param * 3;
    }
}

i32 main(string[] args) {
    BD bd = new BD();
    i64 r = bd.query(14);    // salida ordenada: Before -> Around -> query -> Around -> After
    println("resultado final: ${r}");   // 42
    return 0;
}
```

Salida esperada:
```
[Antes] query iniciado
[Around] midiendo...
  ejecutando query con 14
[Around] resultado = 42
[Despues] query completado
resultado final: 42
```

### Lowering de aspectos en `__module_init`

El compiler Vex genera en `__module_init` la instalacion de cada aspecto:

```c
// Para @Before("BD.query"):
findclass r1, params_BD          // r1 = ClassInfo* de BD
findmethod r2, params_query      // r2 = MethodInfo* de BD.query
findclass r3, params_Instr       // r3 = ClassInfo* de Instrumentacion
findmethod r4, params_antes      // r4 = MethodInfo* de antes_query
addadvice r2, r4, 0              // kind=0 (BEFORE)

// Para @After("BD.query"):
addadvice r2, r5, 1              // kind=1 (AFTER)

// Para @Around("BD.query"):
addadvice r2, r6, 2              // kind=2 (AROUND)
```

La instruccion `addadvice` (0xCE) enlaza el `AdviceEntry` al final del `advice_chain`
del metodo objetivo. El orden de registro determina el orden de ejecucion.

---

## Polimorfismo via tipo interfaz (A.5.2.b)

Cuando el receptor de un metodo esta tipado como interfaz, el lowering emite
`getclass + findmethod + callm` en lugar de `callvirt vtable_idx`:

```java
interface ICalculadora {
    i64 calcular(i64 x);
}

class ImplA : ICalculadora {
    @Override
    public i64 calcular(i64 x) { return x + 10; }
}

class ImplB : ICalculadora {
    @Override
    public i64 calcular(i64 x) { return x * 2; }
}

void usar(ICalculadora calc) {
    println("${calc.calcular(5)}");   // callm: dispatch correcto segun tipo real
}

usar(new ImplA());   // 15
usar(new ImplB());   // 10
```

El `callm` recorre el `advice_chain` igual que `CALLVIRT`, por lo que los aspectos
aplicados a los metodos concretos siguen funcionando aunque el tipo estatico sea interfaz.

---

## Consideraciones de rendimiento

| Patron                      | Coste                                    |
| :-------------------------- | :--------------------------------------- |
| `obj.method()` tipo estatico| CALLVIRT: 1 lookup vtable + jmp          |
| `invoke(m, obj, args)`      | CALLM: mismo que CALLVIRT (via chain)    |
| `forName("X")`              | findclass: O(1) en unordered_map         |
| `getClass(obj)`             | LOAD obj+0: 1 indireccion               |
| `getField(cls, "f")`        | findfield: O(1) tabla hash              |
| Sin advices (chain==NULL)   | 1 cmp + jmp predicted: overhead cero    |
| Con N advices               | N CALLVIRT adicionales en cadena lineal |

Para hot paths con muchos objetos donde la clase sea conocida en compile time, usar
CALLVIRT directo (`obj.method()`) en lugar de `invoke`.  La reflexion es para casos
donde el tipo se conoce solo en runtime (plugins, frameworks, configuracion dinamica).

---

## Limitaciones del MVP

- El pointcut `@Before("Cls.met")` es una cadena exacta; no hay soporte de glob
  (e.g. `"Servicio.*"` NO funcionaria).
- Solo el primer `@Around` en la cadena tiene efecto (multi-AROUND nesting requiere
  stack de `proceed_targets`, deferido).
- `invoke` devuelve `i64` sin coercion automatica al tipo logico del metodo.
- `getInt(field, obj)`/`setInt(field, obj, val)` no estan disponibles en el MVP
  (requeririan opcodes `freadi`/`fwritei` o exposicion de `FieldInfo->offset`).

---

Ver tambien: [[OOP]], [[FFI]], [[SetInstruccionesVM/META_OOP]],
[[SetInstruccionesVM/OOP/CALLVIRT y CALLSUPER]]
