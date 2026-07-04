# Hooks de runtime y control de excepciones (Vesta)

Vesta sigue el modelo "sin runtime obligatorio": **cada servicio de runtime es
un hook que el programador puede reemplazar**, y el codegen prefiere SIEMPRE
codigo inline al runtime (el `CALL` al runtime es el ultimo recurso y solo en
el camino frio).  Este documento describe (1) la convencion de override de los
hooks y (2) el control de excepciones (`@NoExcept` / `@NoExceptions`).

## 1. Convencion de override de hooks

Un servicio de runtime se expone como un simbolo `__vex_<servicio>` con un ABI
documentado.  Para reemplazar la implementacion por defecto del lenguaje basta
**definir una funcion Vesta con ese nombre**: el lowering detecta el override y
emite la llamada a tu version en lugar de la del runtime.  Es el mismo
mecanismo que ya usan la I/O nativa, `@AllocatorOverride` y `@PanicHandler`.

```vex
// Override del sink de stdout: el lenguaje usara esta en vez del default libc.
void __vex_write(char* buf, u64 len) {
    // tu impl: a un UART/MMIO en un kernel, a un buffer propio, etc.
}
```

Reglas:

- Si **no** defines el hook, se usa el default del *tier*:
  - **JIT / Full**: la entry in-process del runtime (`vrt_*`).
  - **AOT bare / embed**: el simbolo `__vex_*` de la bare-lib (redefinible al
    enlazar; en freestanding lo DEBE proveer el usuario).
- El codegen **nunca** codifica `vrt_*` a mano: emite la op, y el lowering ya
  decidio si va a tu override o al default.  Esto unifica JIT y AOT.

### Servicios disponibles (hooks)

| Hook | ABI | Default | Notas |
| :--- | :--- | :--- | :--- |
| `__vex_write(buf, len)` | `(char*, u64)` | libc `fwrite` | sink de stdout |
| `__vex_panic(msg, len)` | `(char*, u64)` | mensaje + abort | fallo fatal |
| `__vex_panic_null()` | `()` | mensaje + abort | unwrap sobre null (AOT) |
| `__vex_terminate()` | `()` | abort | excepcion no capturada en `@NoExcept` |
| `__vex_alloc(size)` / `__vex_free(ptr)` | `(u64)->ptr` / `(ptr)` | libc malloc/free | via `@AllocatorOverride` |
| `__vex_print_*` (int/hex/bool/...) | varios | bare-lib | formato runtime de `${...}` |

Las partes COMPTIME (secuencias ANSI, constantes, format-specs sobre valores
comptime) las resuelve el compilador a bytes en `.rodata` y se emiten via
`__vex_write` -- no pasan por un hook de formato.

## 2. Control de excepciones

Las excepciones se pueden **desactivar** total o parcialmente.  Esto es
necesario en contextos que no pueden tenerlas (kernels, drivers, firmware,
hot-paths) y, ademas, **elimina todo el bookkeeping de excepciones** (sin
frames `tryenter`, sin landing pads) -> codigo mas pequeno y rapido.

### `@NoExcept` (por funcion)

```vex
@NoExcept
i32 hot(i32 x) {
    // throw / try / catch aqui -> error de compilacion
    return x * 2;
}
```

La funcion promete no propagar excepciones.  El compilador:

- **Rechaza** `throw`, `try` y `catch` dentro de su cuerpo (error claro).
- Omite el bookkeeping de excepciones en su codegen.
- Si en runtime escapara una excepcion (p.ej. un `unwrap` null), termina el
  proceso via `__vex_terminate` (no es capturable).

### `@NoExceptions` (por modulo)

```vex
@NoExceptions   // a nivel de fichero: TODO el modulo queda sin excepciones

i32 main() {
    // ningun throw/try/catch en este modulo
    return 0;
}
```

Equivale a marcar `@NoExcept` en todas las funciones del modulo (incluidos los
metodos).  Pensado para targets bare/embed/freestanding.

### Comportamiento de `unwrap` / Optional / Result sin excepciones

- `Optional<T>` y `Result<V,E>` siguen funcionando: son *value-types*, no
  dependen de excepciones.
- `unwrap(x)` / `!!x` sobre un valor null en un scope `@NoExcept`:
  - Con excepciones ON (default Full/JIT): lanza un `FatalError` capturable.
  - En `@NoExcept` / `@NoExceptions`: **no es capturable** -> termina el
    proceso (panic).  Equivale al `unwrap()` de Rust en un binario `panic=abort`.
- `unwrap_unchecked(x)`: SIEMPRE baja a identidad (cero chequeo, UB si null),
  independientemente del modo.  Es el opt-out per-sitio cuando TU garantizas
  el non-null.

### Elision del chequeo de `unwrap` (cero coste cuando se puede probar)

Con excepciones ON o OFF, el chequeo de `unwrap` se **elimina en compilacion**
cuando el operando es provably non-null: constante, `new`, `&local`,
`Some(...)`, o dentro de la rama de un null-check (`if (x != null) { ... !!x }`).
En esos casos no se emite ni chequeo ni llamada -> cero codigo.  Ver el pase
`ir_pass_elide_unwrap`.

## 3. Defaults por tier (resumen)

| Aspecto | Bare / Embed (pre-AOT.7) | Full / JIT |
| :--- | :--- | :--- |
| Excepciones | OFF (recomendado; usar `@NoExceptions`) | ON |
| Hooks | `__vex_*` (bare-lib / freestanding) | `vrt_*` in-process |
| `unwrap` null | `__vex_panic_null` (abort) | throw catchable |

El tier se elige con `--target bare|embed|full`; el control fino por modulo/
funcion es via `@NoExceptions` / `@NoExcept`, sin flags de compilacion.
