# Strings en Vesta

Vesta tiene tres maneras de manejar texto:

1. **`string`** — tipo dedicado, GC-managed (`StringObject` runtime).
2. **`char*` / `char[]`** — punteros raw estilo C, para FFI con APIs nativas.
3. **String literals** — `"..."`, `r"..."`, `"""..."""` con interpolación `${expr}`.

---

## Indice

- [Strings en Vesta](#strings-en-vesta)
 - [Indice](#indice)
 - [1. El tipo `string`](#1-el-tipo-string)
 - [2. Literales: estándar, raw, triple-quoted](#2-literales-estándar-raw-triple-quoted)
 - [Literal estándar `"..."`](#literal-estándar-)
 - [Literal raw `r"..."`](#literal-raw-r)
 - [Triple-quoted `"""..."""`](#triple-quoted-)
 - [3. Interpolación `${expr}`](#3-interpolación-expr)
 - [4. Format specifiers `${expr:fmt}`](#4-format-specifiers-exprfmt)
 - [Format kinds disponibles](#format-kinds-disponibles)
 - [Format alignment](#format-alignment)
 - [5. Operadores `+`, `==`, `!=`](#5-operadores---)
 - [6. Métodos sobre string](#6-métodos-sobre-string)
 - [Las funciones libres `str_*` están retiradas](#las-funciones-libres-str_-están-retiradas)
 - [7. Indexado por byte y slices](#7-indexado-por-byte-y-slices)
 - [8. Cstring (`char*`) para FFI](#8-cstring-char-para-ffi)
 - [9. Codificación: sólo en la frontera nativa](#9-codificación-sólo-en-la-frontera-nativa)
 - [Por qué no existe `str_convert`](#por-qué-no-existe-str_convert)
 - [Limitaciones conocidas](#limitaciones-conocidas)

---

## 1. El tipo `string`

`string` es un GcHandle (i64 opaco) a un `StringObject` GC-managed. Layout interno:

```
StringObject (40 bytes header + data[]):
    ObjectHeader (24 bytes: class_ptr, flags, hash, owner_pid, lock_depth)
    encoding (1 byte: ASCII=0, ANSI=1, UTF8=2, UTF16=3, UTF32=4)
    pad[3]
    length (u32: code-point count)
    byte_len (u32: byte count)
    str_hash (u32: FNV-1a cache, 0 si no computado)
    data[] (bytes, +1 NUL extra para FFI con APIs *A)
```

**3 kinds** internamente:
- **FLAT**: bytes contiguos en data[].
- **ROPE**: nodo + dos hijos (resultado de concatenaciones O(1)).
- **SLICE**: vista (offset, len) sobre otro StringObject.

Los tres kinds son un detalle interno: la materialización de ROPE/SLICE a FLAT
la hace el runtime cuando hace falta (por ejemplo al pedir `cstr()`), y no se
expone como operación del lenguaje.

---

## 2. Literales: estándar, raw, triple-quoted

### Literal estándar `"..."`

```vx
string greeting = "Hola Mundo";
string with_escapes = "Linea1\nLinea2\tTabulado";
string interp = "User: ${name}";
```

Procesa escapes: `\n`, `\t`, `\r`, `\\`, `\"`, `\$`, `\xHH`, `\uHHHH`. Permite
interpolación `${expr}`.

`\$` emite un `$` **literal** sin disparar la interpolación.  Es lo que permite
generar codigo (via `comptime` / `@Macro`) que a su vez contenga
interpolaciones que deben re-interpretarse mas tarde:

```vx
// Este string contiene el TEXTO ${x}, no interpola x aqui.
string codigo = "print(\"\${x}\");";   // -> print("${x}");
```

### Literal raw `r"..."`

```vx
string regex = r"\d{3}-\d{4}"; // sin procesar \d como escape
string path = r"C:\Users\name"; // sin escapar backslash
```

NO procesa escapes ni interpolación. Útil para regex, rutas Windows.

### Triple-quoted `"""..."""`

```vx
string multilinea = """Linea 1
Linea 2
con saltos literales""";

string html = """
<html>
<body>${content}</body>
</html>
""";
```

Permite saltos de línea literales dentro del bloque. Procesa escapes
(`\t`, `\n`, etc.). Soporta interpolación `${expr}` (incluido con format
specifiers `${expr:fmt}`) dentro del triple-quoted.

Modo raw triple-quoted: `r"""..."""` (sin interpolación, sin escapes).

---

## 3. Interpolación `${expr}`

```vx
i32 count = 42;
string name = "World";
println("Hello ${name}, count is ${count}!");
// -> "Hello World, count is 42!"
```

**Tipos soportados** en interpolación:
- `string` — concatena directo.
- `i8`/`i16`/`i32`/`i64` — formato decimal signed.
- `u8`/`u16`/`u32`/`u64` — formato decimal unsigned.
- `bool` — `"true"` o `"false"`.
- `char` — emite el codepoint UTF-8.
- `ptr` / `array` — `0x<hex>` compacto (sin ceros líder).
- Clases (con `gchandle`): `<gc:N>` (N = índice del GcHandle).

**Tipos NO soportados** (emite error claro): `float`/`f64`, struct, class sin
`toString()`, enum. Para estos, construir el mensaje con `print` directo o
toString explícito.

**Implementación**: `lower_string_literal_to_string_object` detecta
`is_interpolated()` y construye el `StringObject` como cadena:

```
STRMAKE(parts[0])
+ STRCAT(stringify(expr[0]))
+ STRCAT(parts[1])
+ ... + STRCAT(parts[N])
```

Para tipos primitivos, hay 5 helpers nativos en `vesta_io` (`vio_int_to_vmbuf`,
`vio_uint_to_vmbuf`, `vio_bool_to_vmbuf`, `vio_char_to_vmbuf`, `vio_ptr_to_vmbuf`)
que escriben representación ASCII al buffer VM, seguidos de STRMAKE para
materializar el fragment.

---

## 4. Format specifiers `${expr:fmt}`

Sintaxis estilo Python/Rust dentro de `${...}` para evitar 40 builtins discretos:

```vx
i32 n = 255;
println("${n}"); // "255"
println("${n:hex}"); // "0x00000000000000FF"
println("${n:bin}"); // "0b11111111"
println("${n:oct}"); // "0o377"
println("${n:>10}"); // " 255" (right-align, width 10)
println("${n:<10}"); // "255 " (left-align)
println("${n:>10*}"); // "*******255" (right-align, fill char '*')
println("${n:hex:>20}"); // " 0x00000000000000FF" (combina kind + align)
```

### Format kinds disponibles

| Kind | Significado |
| :----: | :------------------------------------------------------- |
| `hex` | `0x<hex>` con ancho fijo (i64 = 16 dígitos) |
| `bin` | `0b<bin>` compacto |
| `oct` | `0o<oct>` compacto |
| `dec` | decimal explícito (default) |
| `ptr` | puntero `0x<hex>` compacto sin ceros líder |
| `gc` | GcHandle como `<gc:N>` (clases) |
| `char` | codepoint UTF-32 -> UTF-8 |
| `bool` | `"true"` / `"false"` |

### Format alignment

- `>N` — right-align, ancho mínimo N bytes.
- `<N` — left-align, ancho mínimo N bytes.
- Después del `N` opcional un fill char (no-dígito): `>10*`, `<10.`, `>16=`.

Múltiples specs se separan con `:`: `${n:hex:>20=}` = hex + right-align + width 20
+ fill `=`.

**Limitaciones**:

- El fill char debe ser ASCII 1-byte (multi-byte fill no soportado).
- Format kind `string` no soportado (ignora el spec y usa el camino normal).

`width` se mide en **columnas de terminal**, no en bytes: el texto de doble
ancho (CJK, hangul, emoji) cuenta 2 y las marcas combinantes 0, así que las
tablas alinean bien con texto no latino.

---

## 5. Operadores `+`, `==`, `!=`

```vx
string s = "hola";
string t = " mundo";
string u = s + t; // "hola mundo" - bytecode strcat (ROPE O(1))

bool eq = (s == "hola"); // true - bytecode strcmp byte-a-byte
bool ne = (s != t); // true

// Auto-coerce literal + string:
string r = "prefix " + dynamic_name; // literal se promociona a StringObject
```

- `+` con dos `string` -> `STRCAT` (crea ROPE O(1)).
- `==` y `!=` -> `STRCMP` (compara byte-a-byte, valor 0/1).
- **Auto-coerce**: si un lado es `StringLitExpr` (sin interpolación) y el otro es
 `string`, el literal se promociona automáticamente vía `STRMAKE` inline.

### Lo que se conoce al compilar no cuesta nada al ejecutar

Un `+` entre cadenas que el compilador ya conoce **no llega a ejecutarse**: se
resuelve al compilar y queda un literal.

```vx
string saludo = "hola" + " " + "mundo";   // un literal, 0 STRCAT

string a = "aaa";
string b = "bbb";
string c = a + b;                          // tambien: "aaabbb" al compilar
```

El segundo caso importa porque es el que se escribe sin pensar. Funciona aunque
las partes lleguen por variables, o desde otra función que se haya inlineado, y
encadena solo (`a + b + c`).

Además, la parte que **sólo existía para el concat** deja de construirse: arriba,
`b` no llega a crearse en tiempo de ejecución. Si se usa en algún otro sitio, se
queda. (Sus bytes siguen en la sección de datos: hoy no se podan los literales
que nadie referencia. No cuesta un ciclo, sólo unos bytes.)

Y para partir un mensaje largo, dos literales pegados son uno solo, como en C —
sin operadores:

```vx
string aviso = "esto es una sola cadena, "
               "aunque este escrita en dos lineas";
```

En cuanto una parte se conoce sólo al ejecutar, el `STRCAT` es real y debe serlo:

```vx
string dinamico = "n=" + to_str(n) + "!";   // aqui si hay concatenacion
```

Ejemplo completo: `321_strings_comptime.vx`.

---

## 6. Métodos sobre string

Las operaciones de cadena son **métodos**. Hay una sola forma de escribir cada
cosa:

| Método | Devuelve | Descripción |
| :------------- | :------- | :------------------------------------------------- |
| `s.length()` | `i64` | número de code points |
| `s.bytes()` | `i64` | número de bytes |
| `s.cstr()` | `u8*` | buffer UTF-8 NUL-terminado (APIs `*A` / `const char*`) |
| `s.wstr()` | `u8*` | buffer UTF-16 NUL-terminado (APIs `*W` / `wchar_t*`) |
| `s.hash()` | `u64` | FNV-1a, cacheado en el objeto |
| `s.intern()` | `string` | representante canónico del pool de internados |
| `s.concat(o)` | `string` | igual que `s + o` (ROPE, O(1)) |
| `s.equals(o)` | `bool` | igual que `s == o` (compara contenido) |

```vx
string s = "Hello World";

i64 len   = s.length();      // 11
i64 nb    = s.bytes();       // 11
u8* ptr   = s.cstr();        // host_ptr, NUL-terminado
u64 h     = s.hash();
string cn = s.intern();

string g  = s.concat(" Foo");  // = s + " Foo"
bool   eq = s.equals(t);       // = (s == t)
```

Los métodos funcionan también sobre un literal, que se promociona solo:

```vx
i64 n = "hola".length();     // 4
```

### Las funciones libres `str_*` están retiradas

Escribir `str_length(s)`, `str_cstr(s)`, `str_concat(a, b)`… es un **error de
compilación** que sugiere el método equivalente:

```
str_cstr no existe como funcion libre: las operaciones de cadena son metodos.
Usa `s.cstr(...)` sobre la propia cadena
```

El motivo es que existían las dos formas para lo mismo, y no eran
intercambiables: dependiendo de cuál se escribiera, el código bajaba distinto en
intérprete y en compilación nativa. Los nombres siguen existiendo *dentro* del
compilador —el despacho de métodos reescribe `s.length()` a `str_length(s)` al
bajar a IR— pero eso ocurre después del chequeo de tipos, así que el usuario
nunca los ve.

`str_make(ptr, len)` **no** se retiró: construye una cadena a partir de un
buffer crudo, no es una operación *sobre* una cadena ya existente.

---

## 7. Indexado por byte y slices

`s[i]` devuelve el **byte i-esimo** del `string` como `char` (funciona en
interp, JIT y AOT, y tambien en compile-time dentro de un `comptime fn` /
`@Macro`).  Para ASCII o UTF-8 de 1 byte coincide con el codepoint:

```vx
string s = "GET";
char c = s[0];          // 'G' (0x47)
for (i64 i = 0; i < strlen(s); i++) {
    match s[i] { case 'G' => ...; case _ => ...; }
}
```

Para texto multi-byte, `s[i]` es el byte crudo (no el codepoint completo); usa
`substr(s, i, 1)` si necesitas un `string` de un caracter.  El slice
`s[a..b]` de momento solo esta en compilacion nativa (AOT); usa
`substr(s, start, len)` en interp/JIT.

---

## 8. Cstring (`char*`) para FFI

Para llamar APIs nativas C que esperan `const char*`, usar `s.cstr()`, que
devuelve un `host_ptr` al buffer NUL-terminado:

```vx
extern "kernel32.dll" {
    fn GetFileAttributesA(u8* path) -> u32;
}

string path = "C:\\file.txt";
u32 attrs = GetFileAttributesA(path.cstr());
```

**Alias `cstring`**: es equivalente a `char*`, útil para legibilidad
en signatures FFI. La sintaxis de `typedef` en Vesta es estilo C (tipo primero,
alias después):

```vx
typedef char* cstring; // alias; tipo a la izquierda, nombre nuevo a la derecha
extern fn fopen(cstring path, cstring mode) -> i64;
```

---

## 9. Codificación: sólo en la frontera nativa

Un `string` es **siempre una secuencia de code points**, guardada internamente
como UTF-8. No lleva etiqueta de codificación observable: *"una cadena en
UTF-16"* no es un valor del lenguaje.

La codificación aparece únicamente al **cruzar a código nativo**, donde sí hay
que decidir en qué bytes se entrega el texto:

```vx
extern "kernel32.dll" {
    fn GetFileAttributesA(u8* path) -> u32;    // API *A  -> UTF-8 / ANSI
    fn GetFileAttributesW(u8* path) -> u32;    // API *W  -> UTF-16
}

string ruta = "C:\\datos\\informe.txt";

u32 a = GetFileAttributesA(ruta.cstr());   // buffer UTF-8,  NUL-terminado
u32 w = GetFileAttributesW(ruta.wstr());   // buffer UTF-16, NUL-terminado
```

### Por qué no existe `str_convert`

Antes había `str_convert(s, ENC_*)`, que devolvía otro `string` con una etiqueta
de codificación distinta. Escribirlo hoy da un error explícito:

```
str_convert no existe: un `string` es siempre una secuencia de code points.
La codificacion se elige al cruzar a codigo nativo: usa `s.cstr()` para UTF-8
o `s.wstr()` para UTF-16
```

Se retiró por dos razones:

1. **No significaba nada dentro del lenguaje.** `s.length()` cuenta code points
   y `s == t` compara contenido, así que dos cadenas con el mismo texto y
   distinta etiqueta eran indistinguibles… salvo por `bytes()`, que devolvía un
   número distinto sin que nada más cambiara.
2. **Divergía entre modos.** En compilación nativa la etiqueta era meramente
   informativa, así que el mismo programa daba resultados distintos en
   intérprete y en AOT, en silencio.

El transcodificador sigue existiendo por debajo: el opcode `strconv` decodifica
a code points y recodifica, cubriendo ASCII, ANSI (Latin-1), UTF-8, UTF-16 (con
pares suplentes) y UTF-32 en cualquier combinación. Es lo que usan `cstr()` y
`wstr()`. Lo que desapareció es la posibilidad de dejar una cadena "a medio
convertir" dentro del lenguaje.

Si necesitas los bytes en una codificación concreta distinta de UTF-8/UTF-16,
pide el buffer y trátalo como memoria nativa (`u8*`), que es lo que es.

---

## Limitaciones conocidas

1. **Interpolación con `float`/`f64`/`struct`/`class`/`enum`** en contexto STRING
 (construir un `string` con `${expr}`): todavía no soportada para esos tipos
 (emite error). Los tipos primitivos soportados en interpolación de string son
 `string`, `i8`..`i64`, `u8`..`u64`, `bool`, `char` y `ptr`/`array`. Workaround
 para float: `print`/`println` directo (que sí soporta floats) o stringify
 explícito. Se planea añadir soporte de float y un mecanismo `toString()` virtual
 para clases.

2. **Interpolación `${...}` dentro de triple-quoted**: SI soportada, incluido con
 format specifiers `${expr:fmt}`.

3. **Format kind en STRING**: `${str_var:hex}` ignora el *kind* (sólo aplica a
 tipos numéricos). La **alineación** sí funciona sobre cadenas:
 `${nombre:<10.}` alinea a la izquierda en ancho 10 rellenando con `.`.

---

Ver también: [[TiposDatos]] (modelo de memoria del StringObject), [[FFI]] (uso de
`cstring` con extern), [[Operadores]] (operadores `+`/`==`/`!=` en strings),
[[ControlFlow]] (use de strings en match).
