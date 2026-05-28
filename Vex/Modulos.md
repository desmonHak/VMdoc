# Sistema de módulos

Vex tiene un sistema de módulos completo que permite dividir código en
múltiples archivos, importar símbolos, organizar paquetes (directorios) y
distribuir librerías precompiladas con su interfaz binaria estable.

Documentos relacionados:

- [[CompilacionCondicional]] — `@Target` para dependencias platform-specific
- [[CargaDinamica]] — `loadmodule`/`unloadmodule` en runtime
- [[Sandbox]] — capabilities sobre módulos cargados

## Resumen rápido

```java
// math_utils.vex
public i32 add(i32 a, i32 b) { return a + b; }
public const i32 PI_FIXED = 314;
public class Counter {
    public i32 value;
    public Counter(i32 init) { this.value = init; }
    public i32 inc() { this.value = this.value + 1; return this.value; }
}
private i32 internal_helper() { return 42; } // NO se exporta
```

```java
// main.vex
import "math_utils"; // namespace cualificado
import "math_utils" as mu; // alias
import "math_utils" only add, PI_FIXED, Counter; // al scope local

i32 main() {
    i32 a = math_utils.add(10, 20); // vía namespace
    i32 b = mu.add(1, 2); // vía alias
    i32 c = add(3, 4); // vía only (sin prefijo)
    Counter ctr = new Counter(0);
    return ctr.inc() + c;
}
```

Compilación:

```bash
vm --vex main.vex -o programa
```

El compilador detecta los `import` automáticamente, resuelve las
dependencias en orden topológico, cachea la interfaz binaria de cada
una (archivos `.vexi`) y produce un único `programa.velb` con todo el
código necesario.

## Sintaxis de `import`

### Formas soportadas

```java
import "path"; // (1) plain: namespace = último segmento
import "path" as alias; // (2) plain con alias
import "path" only A, B; // (3) selectivo: A, B al scope local
import "path" only A as A2, B; // (4) selectivo con renombrado
public import "path"; // (5) re-export completo
public import "path" only A, B; // (6) re-export selectivo
@Target("os:windows") import "path"; // (7) condicional (ver CompilacionCondicional)
```

Notas:

- El `path` es siempre un string literal. No se admite variable.
- Los paths usan `/` como separador (multiplataforma).
- No se incluye la extensión `.vex` — el resolvedor la añade automáticamente.
- Para cargar un `.velb` en runtime, ver `loadmodule(path)` en [[CargaDinamica]].

### Resolución de paths

Para `import "a/b/c";` el resolvedor busca en este orden:

1. **Directorio del importador**: `<importer_dir>/a/b/c.vex` (módulo single-file) o `<importer_dir>/a/b/c/mod.vex` (paquete-directorio).
2. **Search paths**: cada entrada de la variable de entorno `VEX_PATH` (separador `:` en POSIX, `;` en Windows) y cualquier `--vex-path /dir` adicional.
3. **Paquetes locales**: `./vex_modules/a/b/c.vex` (reservado para el futuro gestor de paquetes).
4. **Standard library**: `$VEX_HOME/stdlib/a/b/c.vex` (solo si el path empieza con `std/`).

El primer match gana. Si nada coincide, se reporta un error claro con la
lista de paths probados.

### Privacidad: `public` y `private`

Los símbolos son **privados al módulo** por defecto. El programador
marca explícitamente lo que exporta:

```java
public i32 add(i32 a, i32 b) { return a + b; } // exportado
public class Counter { ... } // exportado
public const i32 MAX_USERS = 100; // exportado (con valor inlineable)

private i32 internal_helper() { return 42; } // NO exportado
i32 also_internal() { return 0; } // sin keyword: tampoco exportado
```

Aplicable a: `class`, `struct`, `enum`, `typedef`, `using`, funciones
libres y globales (`const`/`var`).

Si un consumidor escribe `import "lib" only internal_helper;`, el
compilador lo rechaza con un mensaje claro:

```text
error: el modulo 'lib' no exporta 'internal_helper' (es privado o no existe)
```

## Múltiples ficheros: paquetes-directorio

Un paquete es un directorio con un fichero de entrada `mod.vex` más
otros `.vex` adicionales.

```text
proyecto/
├── main.vex
└── mypkg/
    ├── mod.vex ; entry point del paquete
    ├── utils.vex
    └── internal.vex
```

```java
// mypkg/mod.vex
// Re-export plain: TODOS los símbolos públicos de utils son visibles
// como si estuvieran declarados directamente en mod.vex.
public import "utils";

// También se puede hacer re-export selectivo:
public import "internal" only helpful_function;

// Y declaraciones propias del entry:
public i32 mod_value() { return 100; }
```

```java
// mypkg/utils.vex
// El path es relativo al directorio del importador (mypkg/).
// Desde mod.vex, "import 'utils'" resuelve a mypkg/utils.vex.
public i32 utility_a() { return 1; }
public i32 utility_b() { return 2; }
```

```java
// proyecto/main.vex
import "mypkg" only utility_a, utility_b, mod_value, helpful_function;

i32 main() {
    return utility_a() + utility_b() + mod_value() + helpful_function();
}
```

Cuando el resolvedor encuentra `mypkg/mod.vex`, el nombre lógico del
módulo es `mypkg` (no `mod`). El paquete se identifica por su
directorio, no por el fichero entry.

## Re-export: `public import`

Un módulo puede "re-exportar" símbolos de sus dependencias para que
los consumidores no necesiten conocer la estructura interna del
paquete.

### Forma selectiva (`only`)

```java
// mid.vex
public import "base" only foo, bar; // foo y bar visibles como propios de mid
```

Ahora `import "mid" only foo;` funciona aunque `foo` esté declarado en
`base.vex`.

### Forma plana (re-export completo)

```java
// mid.vex
public import "base"; // TODOS los símbolos públicos de base son re-exportados
```

El compilador itera los símbolos públicos de `base`, los inyecta en
`mid` y los marca como re-exportados. El `.vexi` de `mid` los expone
como propios.

**Cuándo usar plain vs only**:

- `public import "base";` — el paquete entero forma parte de la API pública.
- `public import "base" only A, B;` — solo unos símbolos específicos se reexponen.

## Compilación incremental (caché)

El compilador mantiene una caché transparente para evitar recompilar
módulos cuyo source no haya cambiado. Activa por defecto.

### Qué se cachea

Por cada módulo (excepto el root):

- **`<source>.vexi`** — interfaz binaria del módulo (tipos y firmas públicas).
- **`<source>.vexir`** — IR SSA serializado para reutilizar al hacer el merge.
- **`<source>.vel`** — bytecode standalone del módulo (formato distribuible).

Adicionalmente:

- **`.vex_cache/projects/<root_hash>.vpc`** — el `.velb` final completo. Si todos los módulos del proyecto coinciden por source_hash, el `.velb` se copia directamente (omite compile y link, hit instantáneo).

### Invalidación

Tres niveles independientes:

1. **Source hash**: si el `.vex` del módulo cambió (cualquier byte), su caché se invalida y se recompila.
2. **Dependencia transitiva**: si una dependencia DIRECTA del módulo cambió su `abi_hash` (interfaz pública), el módulo también se recompila aunque su source no haya cambiado.
3. **Compiler version hash**: si el binario del compilador cambió (otro build, otra versión), todas las cachés se invalidan automáticamente.

Esto garantiza builds reproducibles bit-perfect: dos compilaciones del
mismo source con el mismo compilador producen el mismo `.velb` byte
por byte.

### Variables de entorno

| Variable | Efecto |
|:---|:---|
| `VEX_NO_CACHE=1` | Desactiva toda la caché (compile y link siempre frescos). |
| `VEX_NO_PROJECT_CACHE=1` | Desactiva solo el `.vpc` (project cache); sigue cacheando `.vexi`/`.vexir`. |
| `VEX_CACHE_DIR=/path` | Redirige la caché de `.vexi`/`.vexir`/`.vel` al directorio dado (por defecto: junto al source). Habilita compartir caché entre proyectos para librerías comunes. |
| `VEX_VERBOSE_CACHE=1` | Imprime `[vex-cache] hit/miss` por módulo. |
| `VEX_VERBOSE_PROJECT_CACHE=1` | Imprime información del project cache. |
| `VEX_VERBOSE_COMPILE=1` | Imprime el plan topológico y el progreso de compilación de cada módulo. |

## Compilación paralela

Por defecto el compilador usa `std::thread::hardware_concurrency()`
limitado a 8 threads como máximo. Los módulos del MISMO nivel
topológico (sin dependencias entre sí) se compilan concurrentemente;
los de niveles distintos respetan el orden mediante un barrier por
nivel.

Override vía `VEX_PARALLEL_COMPILE=N`:

```bash
VEX_PARALLEL_COMPILE=1 vm --vex main.vex -o prog # secuencial forzado
VEX_PARALLEL_COMPILE=4 vm --vex main.vex -o prog # exactamente 4 threads
# Sin la variable: auto (hardware_concurrency limitado a 8)
```

Cuándo usar secuencial: para diagnóstico, o cuando se necesita output
determinista de `VEX_VERBOSE_COMPILE`. En modo paralelo el orden de
las líneas `[Lx]` puede variar, pero un mutex interno garantiza que
cada línea sea atómica (no se entremezclan a nivel byte).

Cuándo subir el conteo: builds grandes con más de 10 módulos en un
mismo nivel topológico. Por encima de 8 threads aparecen retornos
decrecientes por contención.

## Formato `.vexi` (interfaz binaria)

El `.vexi` es un blob binario que describe la API pública de un módulo.
Es lo único que un consumidor necesita conocer de la dependencia — el
source `.vex` puede mantenerse privado, distribuyendo solo `.vexi` y
`.velb` (librería precompilada).

### Layout

```text
[+0 ] u32 magic = 'VEXI'
[+4 ] u16 format_version
[+6 ] u16 reservado
[+8 ] u64 abi_hash (FNV-1a sobre símbolos y firmas públicas)
[+16 ] u64 source_hash (FNV-1a sobre el .vex fuente)
[+24 ] u64 compiler_version_hash (auto-invalidación cross-version)
[+32 ] u32 symbol_count
[+36 ] u32 string_pool_offset
[+40 ] u32 dep_count (dependencias directas para invalidación transitiva)
[+44 ] u32 dep_table_offset
[+48 ] SymbolEntry[symbol_count] (tamaño variable; ver tipos abajo)
[ +N ] DepEntry[dep_count] (16 bytes cada uno: name_off + name_len + abi_hash)
[ +M ] String pool (nombres y payload referenciados por offset)
```

### Tipos de símbolo

| Kind | Payload |
|:---|:---|
| `FUNCTION` | nombre, lib (extern), mangled_label, return_type, param_types[] |
| `STRUCT` | nombre, size, align, fields[] (con bit_offset/bit_width) |
| `CLASS` | nombre, super, interfaces[], size, fields[], methods[] (con vtable_index/flags) |
| `ENUM` | nombre, variants[] (con payload_types[]) |
| `TYPEDEF_ALIAS` | nombre, underlying_type_string |
| `TYPEDEF_NEW` | nombre, underlying, is_opaque, align_override, from_conversions[], to_conversions[] |
| `GLOBAL_VAR` | nombre, type, flags (is_const, is_public), has_init_value, init_value (i64) |

Los símbolos privados se filtran al emitir (no aparecen en el `.vexi`).

### Firma digital

Soporte opcional vía OpenSSL `EVP_DigestSign`/`Verify` (RSA-SHA256 o
ECDSA-SHA256):

```bash
vm --sign-velb prog.velb --sign-key private.pem -o prog.signed.velb
vm --verify-velb prog.signed.velb --verify-key public.pem
# exit 0 si la firma es válida, 1 si es inválida o el archivo no está firmado
```

El footer firmado se añade al final del `.velb` con magic `VSIG`. El
loader lo ignora al cargar; un `.velb` firmado se ejecuta normalmente
sin verificación explícita.

## Namespaces inline

Sintaxis al estilo C++ para agrupar declaraciones dentro de un mismo
fichero. Soporta anidamiento. El mangling automático evita colisiones.

```java
namespace ui {
    public class Button {
        public string label;
        public Button(string l) { this.label = l; }
    }
    namespace internal {
        public i32 layout_helper() { return 0; }
    }
}

namespace audio {
    public class Button { // OK: distinto namespace, no colisiona
        public f64 volume;
    }
}

i32 main() {
    ui.Button b = new ui.Button("OK");
    audio.Button a = new audio.Button();
    i32 x = ui.internal.layout_helper();
    return 0;
}
```

Internamente las clases se manglean a `ui__Button`, `audio__Button`,
`ui__internal__layout_helper`, etc. El acceso siempre se hace con `.`
(nunca con `::`) para mantener consistencia con el acceso a campos.

## Tipos cross-module cualificados

Cuando se hace `import "lib";` (plain), los TIPOS públicos del módulo
quedan accesibles como `lib.TypeName`:

```java
// lib.vex
public class Counter { public i32 value; public Counter(i32 v) { this.value = v; } }
public struct Point { i32 x; i32 y; }
public typedef i64 UserId;
```

```java
// main.vex
import "lib";

i32 main() {
    lib.Counter c = new lib.Counter(42);
    lib.Point p = {.x = 1, .y = 2};
    lib.UserId uid = 0xDEADBEEF;
    return c.value;
}
```

Diferencia con `only Counter`: el cualificado `lib.Counter` permite
acceder al tipo SIN traerlo al scope local. Útil cuando hay potencial
colisión con tipos del consumidor.

## Tree-shaking opt-in

Permite reducir el `.velb` final eliminando dependencias cuyos símbolos
importados no se referencien. Desactivado por defecto.

```bash
VEX_TREE_SHAKE=1 vm --vex main.vex -o prog
```

Reglas:

- Solo se eliminan dependencias importadas con `only` y con TODOS los símbolos sin usar.
- Imports plain (`import "lib";`) nunca se eliminan (referencia opaca).
- Dependencias que declaran clases nunca se eliminan (`__module_init` puede tener efectos colaterales de registro).

Beneficio típico: en una librería con 3 funciones y ninguna usada, el
`.velb` baja del orden de 2.2 KB a 1.2 KB.

## Errores frecuentes

### "modulo 'X' no encontrado"

```text
error: modulo 'mypkg/utils' no encontrado. Paths probados:
    - main.vex/../mypkg/utils.vex
    - main.vex/../mypkg/utils/mod.vex
    - VEX_HOME/stdlib/mypkg/utils.vex
```

Causas posibles:

- Typo en el path. Usa `/` como separador y sin extensión.
- El fichero existe pero está en otro directorio. Añadirlo vía `VEX_PATH=/dir1:/dir2` o `--vex-path /dir`.
- Para stdlib, prefijar con `std/`.

### "el modulo 'X' no exporta 'Y'"

```text
error: el modulo 'lib' no exporta 'helper' (es privado o no existe)
```

El símbolo `helper` está declarado en `lib.vex` pero sin `public`. O no
existe. Verificar con `grep "public.*helper" lib.vex`.

### "ciclo de herencia detectado"

```text
error: ciclo de herencia detectado: la clase 'A' aparece en su propia cadena de superclases
```

Una clase `A : B` cuya cadena de superclases vuelve a `A` (por ejemplo
`A : B`, `B : C`, `C : A`). El detector soporta hasta 256 niveles de
profundidad.

### "el dep 'Y' cambio su abi_hash"

```text
[vex-cache] miss (transitivo): dep 'lib' cambio (abi_hash old=0x... new=0x...)
```

Tras cambiar `lib.vex` modificando su interfaz pública, los
consumidores detectan el desajuste y se recompilan. Solo informativo;
el build continúa.

## Limitación conocida

### Distribución de librerías precompiladas sin source

El linker actual fusiona los IRs de cada módulo en uno solo antes de
emitir el `.velb` final. No puede consumir directamente un `.velb` ya
compilado para incluirlo en otro `.velb`. Consecuencia práctica:

- Para distribuir una librería precompilada sin source, el consumidor
 necesita el `.vex` original. El `.vexi` por sí solo no basta porque
 carece de los cuerpos IR necesarios para el merge.
- **Alternativa**: usar `loadmodule("lib.velb")` en runtime (ver
 [[CargaDinamica]]). El módulo se carga dinámicamente en el proceso
 vivo, sus clases se registran vía `__module_init` y el consumidor las
 usa por reflexión (`forName`, `newInstance`, `invoke`).

Cerrar esta limitación requiere un linker capaz de parsear `.velb`s
ya compilados y resolver símbolos cross-velb. Es trabajo significativo
que no afecta al uso normal del lenguaje.
