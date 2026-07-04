# Strings en Vesta

Vesta tiene tres maneras de manejar texto:

1. **`string`** — tipo dedicado, GC-managed (`StringObject` runtime).
2. **`char*` / `char[]`** — punteros raw estilo C, para FFI con APIs nativas.
3. **String literals** — `"..."`, `r"..."`, `"""..."""` con interpolación `${expr}`.

---

## Indice

- [Strings en Vesta](#strings-en-vx)
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
 - [6. Métodos OO sobre string](#6-métodos-oo-sobre-string)
 - [7. Builtins libres](#7-builtins-libres)
 - [8. Cstring (`char*`) para FFI](#8-cstring-char-para-ffi)
 - [9. Encodings y conversion UTF-8 / UTF-16](#9-encodings-y-conversion-utf-8--utf-16)
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

`str_flat(s)` materializa ROPE/SLICE a FLAT (identidad si ya es FLAT).

---

## 2. Literales: estándar, raw, triple-quoted

### Literal estándar `"..."`

```vx
string greeting = "Hola Mundo";
string with_escapes = "Linea1\nLinea2\tTabulado";
string interp = "User: ${name}";
```

Procesa escapes: `\n`, `\t`, `\r`, `\\`, `\"`, `\xHH`, `\uHHHH`. Permite
interpolación `${expr}`.

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
(`\t`, `\n`, etc.). Soporta interpolación `${expr}` desde .

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

Sintaxis estilo Python/Rust dentro de `${...}` para evitar 40 builtins discretos
(añadido ):

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
- `width` se cuenta en **bytes** UTF-8 emitidos, no en columnas visuales (los
 caracteres CJK/emoji multi-byte pueden dar columnas erróneas).
- El fill char debe ser ASCII 1-byte (multi-byte fill no soportado).
- Format kind `string` no soportado (ignora el spec y usa el camino normal).

---

## 5. Operadores `+`, `==`, `!=`

```vx
string s = "hola";
string t = " mundo";
string u = s + t; // "hola mundo" - bytecode strcat (ROPE O(1))

bool eq = (s == "hola"); // true - bytecode strcmp byte-a-byte
bool ne = (s != t); // true

// Auto-coerce literal + string (A.18 fase B):
string r = "prefix " + dynamic_name; // literal se promociona a StringObject
```

- `+` con dos `string` -> `STRCAT` (crea ROPE O(1)).
- `==` y `!=` -> `STRCMP` (compara byte-a-byte, valor 0/1).
- **Auto-coerce**: si un lado es `StringLitExpr` (sin interpolación) y el otro es
 `string`, el literal se promociona automáticamente vía `STRMAKE` inline.

---

## 6. Métodos OO sobre string

Sintaxis natural `s.method()` (azúcar para builtins libres):

```vx
string s = "Hello World";

i32 len = s.length(); // 11 - code-point count
i32 bytes = s.bytes(); // 11 - byte count
u8* ptr = s.cstr(); // host_ptr a buffer NUL-terminated
u32 hash = s.hash(); // FNV-1a cached
string canon = s.intern(); // canonical (intern pool)

string g = s.concat(" Foo"); // = s + " Foo"
bool eq = s.equals(t); // = (s == t)
```

Internamente, los métodos despachan a builtins libres (`str_length(s)`,
`str_concat(s, t)`, etc.) — mismo bytecode emitido, sólo cambia la sintaxis.

---

## 7. Builtins libres

| Builtin | Equivalente método | Descripción |
| :------------------- | :----------------- | :--------------------------------------- |
| `str_length(s)` | `s.length()` | code-point count |
| `str_bytes(s)` | `s.bytes()` | byte count |
| `str_cstr(s)` | `s.cstr()` | host_ptr al buffer NUL-terminated |
| `str_wstr(s)` | `s.wstr()` | convierte a UTF-16, devuelve host_ptr a `wchar_t*` (Win32 *W) |
| `str_hash(s)` | `s.hash()` | FNV-1a cacheado |
| `str_intern(s)` | `s.intern()` | canonical pool (dedup runtime) |
| `str_concat(a, b)` | `a.concat(b)` | concatenación (ROPE) |
| `str_equals(a, b)` | `a.equals(b)` | comparación contenido |
| `str_make(ptr, len)` | | crea StringObject desde buffer raw |
| `str_convert(s, enc)`| | convierte a otro encoding |

Constantes de encoding registradas globalmente:

```vx
ENC_ASCII = 0
ENC_ANSI = 1
ENC_UTF8 = 2
ENC_UTF16 = 3
ENC_UTF32 = 4
```

Uso:

```vx
string utf16 = str_convert(s, ENC_UTF16);
u8* w_ptr = str_cstr(utf16); // wchar_t* listo para Win32 *W APIs
```

---

## 8. Cstring (`char*`) para FFI

Para llamar APIs nativas C que esperan `const char*`, usar `str_cstr()` que devuelve
un `host_ptr` al buffer NUL-terminated:

```vx
extern "kernel32.dll" {
    fn GetFileAttributesA(u8* path) -> u32;
}

string path = "C:\\file.txt";
u32 attrs = GetFileAttributesA(str_cstr(path));
```

**Alias `cstring`** (A.18 fase B): es equivalente a `char*`, útil para legibilidad
en signatures FFI.

```vx
typedef cstring = char*; // alias declarado en el preludio (no hace falta hacer typedef)
extern fn fopen(cstring path, cstring mode) -> i64;
```

---

## 9. Encodings y conversion UTF-8 / UTF-16

El runtime de strings (`stdlib/native/string` + bytecode 0x46-0x54) soporta los
5 encodings listados arriba. El default al crear strings (literales sin sufijo,
`str_make` sin encoding explícito) es **UTF-8**.

```vx
string utf8_s = "Hola µndo"; // UTF-8 default
string utf16 = str_convert(utf8_s, ENC_UTF16);
// utf16.bytes() != utf8_s.bytes() (UTF-16 usa 2-4 bytes/char vs 1-4 UTF-8)
```

**HotSpot-style compaction**: el runtime puede detectar que un string UTF-8 es
puramente ASCII (<= 0x7F) y mantenerlo como ASCII interno (no expande a UTF-16
automaticamente). Para FORZAR un encoding específico, usar `str_convert`
explícito.

**Round-trips**: `str_convert(str_convert(s, UTF16), UTF8)` produce el mismo string
si todos los caracteres están en ambos encodings (Unicode BMP).

---

## Limitaciones conocidas

1. **Interpolación con `float`/`f64`/`struct`/`class`/`enum`**: no soportada
 (emite error). Workaround: `print` directo (que sí soporta floats) o
 stringify explícito. Propuesta futura P1: añadir helper
 `vio_float_to_vmbuf` y mecanismo `toString()` virtual para clases.

2. **Interpolación `${...}` dentro de triple-quoted**: SI soportada desde .

3. **Format spec en STRING**: `${str_var:hex}` ignora el spec (sólo aplica a tipos
 numéricos). Para alinear strings, medir con `str_length(s)` y usar
 `print_pad(' ', N - len)` manualmente.

4. **Width en columnas vs bytes**: `${expr:>N}` mide N bytes UTF-8, no columnas
 terminal. Texto multi-byte (CJK, emoji) puede dar alineación incorrecta.

---

Ver también: [[TiposDatos]] (modelo de memoria del StringObject), [[FFI]] (uso de
`cstring` con extern), [[Operadores]] (operadores `+`/`==`/`!=` en strings),
[[ControlFlow]] (use de strings en match).
