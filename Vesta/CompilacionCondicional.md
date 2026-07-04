# Compilación condicional con `@Target`

Vesta soporta compilación condicional mediante la anotación `@Target("spec")`.
La anotación evalúa un spec (expresión booleana sobre tags del binario)
en tiempo de compilación; si el spec no coincide, la declaración se
descarta del AST sin generar código ni diagnósticos espurios.

Equivalencias en otros lenguajes:

- C/C++: `#if defined(_WIN32) && defined(__x86_64__)`
- Rust: `#[cfg(all(target_os = "windows", target_arch = "x86_64"))]`
- D: `static if (...)`
- Zig: `if (builtin.os.tag == .windows)` (evaluado en comptime)

## Sintaxis

```java
@Target("spec_expr")
<declaracion>
```

Donde `spec_expr` es una expresión sobre tags del binario:

```text
atom ::= IDENT(":" IDENT)? ; "os:windows", "cpu:avx2"
    | IDENT op VERSION ; "compiler>=1.0", "vm<2.0"
unary ::= "!"? atom | "!"? "(" expr ")"
and ::= unary ("&&" unary)*
expr ::= and ("||" and)*
```

## Tags disponibles

### Sistema operativo

| Tag | Significado |
|:---|:---|
| `os:windows` | Build en Windows (cualquier x86/x64/ARM). |
| `os:linux` | Build en Linux. |
| `os:macos` | Build en macOS. |
| `os:posix` | Linux o macOS (heredado automáticamente). |

### Arquitectura

| Tag | Significado |
|:---|:---|
| `arch:x86_64` | AMD64 / x86-64. |
| `arch:arm64` | ARM64 / AArch64. |
| `arch:x86` | x86 32 bits. |

### Modo de build

| Tag | Significado |
|:---|:---|
| `debug` | Build sin `-DNDEBUG`. |
| `release` | Build con `-DNDEBUG`. |

### CPU features (autodetección)

En x86/x86_64 se detectan automáticamente vía `__cpuid` / `__cpuid_count`
al compilar el binario:

| Tag | Detección |
|:---|:---|
| `cpu:sse` | CPUID leaf 1, EDX bit 25 |
| `cpu:sse2` | EDX bit 26 |
| `cpu:sse3` | ECX bit 0 |
| `cpu:ssse3` | ECX bit 9 |
| `cpu:sse4_1` | ECX bit 19 |
| `cpu:sse4_2` | ECX bit 20 |
| `cpu:avx` | ECX bit 28 |
| `cpu:fma` | ECX bit 12 |
| `cpu:bmi1` | CPUID 7.0 EBX bit 3 |
| `cpu:avx2` | CPUID 7.0 EBX bit 5 |
| `cpu:bmi2` | CPUID 7.0 EBX bit 8 |
| `cpu:avx512f` | CPUID 7.0 EBX bit 16 |

En ARM64 (siempre presente):

| Tag | Detección |
|:---|:---|
| `cpu:neon` | Inherente a ARM64. |

### Versión semver

Tags con prefijo `compiler:` y `vm:` que soportan comparación semver:

| Tag | Significado |
|:---|:---|
| `compiler:M.N` | Versión del compilador (de `project(VERSION X.Y.Z)` en `CMakeLists.txt`). |
| `vm:M.N` | Versión de la VM (misma fuente). |

Operadores: `>=`, `>`, `<=`, `<`, `==`, `!=`. Ejemplo:

```java
@Target("compiler>=1.0")
i32 needs_modern_compiler() { return 1; }
```

### Modo de ejecución (variable de entorno)

Tag con prefijo `mode:` controlable mediante la variable
`VEX_TARGET_MODE` al compilar:

| Valor (env) | Tag emitido | Uso |
|:---|:---|:---|
| (sin definir) | `mode:auto` | Por defecto. Código agnóstico al JIT. |
| `jit` | `mode:jit` | Build optimizado para ejecución JIT. |
| `vm` | `mode:vm` | Build solo intérprete (sin JIT). |
| `jit-required` | `mode:jit-required` | Código que solo funciona con JIT (por ejemplo, asm inline). |

```bash
VEX_TARGET_MODE=jit vm --vex prog.vx -o prog
```

## Ejemplos

### Atómico simple

```java
@Target("os:windows")
i32 win_specific() { return 1; }

@Target("os:linux")
i32 linux_specific() { return 2; }
```

### Negación

```java
@Target("!os:windows")
i32 not_for_windows() { return 100; }
```

### OR (alternativas)

```java
@Target("os:linux || os:macos")
i32 posix_only() { return 200; }
```

### AND (intersección)

```java
@Target("os:windows && arch:x86_64")
i32 win_x64_only() { return 11; }
```

### Paréntesis (precedencia)

```java
@Target("(os:windows || os:linux) && arch:x86_64")
i32 win_or_linux_x64() { return 4; }
```

### CPU features

```java
@Target("cpu:avx2")
i32 vectorized_sum(i32[] arr) { /* impl con intrínsecos AVX2 */ }

@Target("cpu:sse2 && !cpu:avx2")
i32 vectorized_sum(i32[] arr) { /* fallback SSE2 */ }

@Target("!cpu:sse2 && !cpu:neon")
i32 vectorized_sum(i32[] arr) { /* implementación escalar */ }
```

### Semver

```java
@Target("compiler>=1.0 && vm>=1.0")
i32 modern_only() { return 4; }

@Target("compiler<99.0")
i32 not_future() { return 0; }
```

### Modo de ejecución

```java
@Target("mode:jit || mode:jit-required")
@Asm
i32 hot_path_inline_asm() { /* asm directo */ }

@Target("mode:vm")
i32 hot_path_fallback() { /* implementación portable interp */ }
```

### Combinaciones

```java
@Target("os:windows && cpu:avx2 && compiler>=1.0")
i32 highly_specific() { return 42; }

@Target("!(os:windows && arch:x86)")
i32 not_legacy_32bit() { return 100; }
```

## `@Target` sobre imports

Permite cargar dependencias platform-specific. La anotación debe
preceder al `import`:

```java
// Para Windows: carga del wrapper Win32.
@Target("os:windows") import "win_specific";

// Para POSIX: wrapper distinto con la misma API pública.
@Target("os:posix") import "posix_specific";

// Si ninguna condición coincide, el import se descarta del AST.
// La dependencia no se resuelve ni se compila, lo que permite tener
// .vx exclusivos de cada plataforma sin necesidad de stub vacíos en
// las demás.

i32 main() {
    return platform_value();
}
```

Mapeo directo a otros lenguajes:

- C: `#if defined(_WIN32)\n#include "win_specific.h"\n#endif`
- Rust: `#[cfg(target_os = "windows")] use win_specific;`

## Limitaciones

### Sintaxis inválida implica skip silente

Si el spec está mal formado (paréntesis desbalanceados, operador
suelto, tag desconocido), la declaración se descarta como si no
coincidiera. No hay diagnóstico explícito de "spec inválido".

Workaround: probar el spec aislado en un programa pequeño para
verificar que el output coincide con lo esperado.

### `@Target` solo en declaraciones top-level

```java
i32 main() {
    @Target("os:windows") // ERROR: el parser no lo reconoce aquí
    i32 x = 1;
    return x;
}
```

Workaround: extraer la lógica platform-specific a funciones top-level
con guard y llamarlas desde el cuerpo común.

### Versiones leídas en tiempo de compilación

Los tags `compiler:` y `vm:` se calculan al construir el binario del
compilador (desde `project(VMProject VERSION X.Y.Z)` de CMake). Para
incrementar:

1. Editar `CMakeLists.txt`: `project(VMProject VERSION 1.1.0 ...)`.
2. Rebuild: `cmake --build cmake-build-windows --target vm`.
3. El header `version_generated.h` se regenera automáticamente y los
 programas Vesta compilados con el nuevo binario emiten los nuevos
 tags.

### Features de CPU son estáticos al binario

Los tags `cpu:*` se detectan al CONSTRUIR el compilador, no al
compilar cada programa. Si el binario del compilador corre en una
máquina con CPU diferente al target final, los tags emitidos podrían
no coincidir.

Para cross-compilation, el usuario debe poder forzar los tags
manualmente (no implementado todavía).

## Detalles internos

El parser del spec vive en `src/vex/parser.cpp`:

- `target_matches_(spec)` evalúa el spec contra `build_tags_()`.
- `build_tags_()` enumera todos los tags del binario actual.
- `TargetSpecParser` es un mini-parser recursivo descendente del spec.

El skip de la declaración usa `skip_target_skipped_decl()`, que avanza
el lexer hasta el cierre del bloque o el punto y coma sin consumir
tokens de la declaración siguiente.

Tests integradores en `tests/bugs/m_l24_test/`,
`tests/bugs/m_condcomp_test/` y `tests/bugs/m_condcomp_import_test/`.
