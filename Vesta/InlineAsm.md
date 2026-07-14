# Ensamblador en linea (inline asm)

Vesta permite intercalar codigo ensamblador **x86-64 nativo** dentro de una
funcion escrita en Vesta.  Es la valvula de escape para lo que el lenguaje no
expresa directamente: intrinsecos de la CPU (`popcnt`, `bswap`, `rdtsc`,
`cpuid`), instrucciones SIMD (`paddd`, `vpaddd`, `pext`), llamadas al sistema
(`syscall`) y rutinas de muy bajo nivel (ISRs, cambio de modo, stubs de
arranque).

El bloque se escribe en **sintaxis NASM Intel pura**: puedes copiar y pegar el
codigo directamente de los manuales de Intel/AMD.  Las variables Vesta se ligan
a registros fisicos concretos con la clase de almacenamiento `register("reg")`.

El ensamblador es codigo **nativo del anfitrion** (host).  Se ejecuta en los
tres modos de VestaVM (por defecto, `-m jit` y `-m vm`), se puede portar a C
(`--port c`) y se compila a binarios nativos (`-m aot`).


## Indice

1. [Un primer ejemplo](#un-primer-ejemplo)
2. [Formas de escribir ensamblador](#formas-de-escribir-ensamblador)
3. [El bloque `asm { ... }`](#el-bloque-asm----)
4. [Ligar variables a registros: `register("reg")`](#ligar-variables-a-registros-registerreg)
5. [Operandos con la lista `( ... )` (el compilador elige el registro)](#operandos-con-la-lista----el-compilador-elige-el-registro)
6. [Calificadores (qualifiers)](#calificadores-qualifiers)
7. [La clausula `clobbers(...)`](#la-clausula-clobbers)
8. [Sustitucion de constantes `comptime`](#sustitucion-de-constantes-comptime)
9. [Modos de ejecucion y backends](#modos-de-ejecucion-y-backends)
10. [Referenciar simbolos propios](#referenciar-simbolos-propios)
11. [Funciones sin marco: `@Naked`](#funciones-sin-marco-naked)
12. [Bloques `asm` a nivel de modulo (16/32/64 bits)](#bloques-asm-a-nivel-de-modulo-163264-bits)
13. [Mas ejemplos](#mas-ejemplos)
14. [Limitaciones actuales](#limitaciones-actuales)


## Un primer ejemplo

Cuenta los bits a 1 de un valor con la instruccion `popcnt`:

```vesta
u64 main() {
    register("rdi") u64 valor = 0xFF;   // 8 bits a 1
    register("rax") u64 cuenta;          // solo salida (sin inicializar)
    asm volatile {
        popcnt rax, rdi                  // cuenta = popcount(valor)
    };
    return cuenta;                        // -> 8
}
```

- `register("rdi") u64 valor = 0xFF;` liga la variable `valor` al registro
  `rdi` y la inicializa.  El asm la lee ahi.
- `register("rax") u64 cuenta;` liga `cuenta` a `rax`.  No tiene inicializador,
  asi que es una **salida**: el asm la escribe.
- Dentro del bloque escribes NASM puro usando los nombres de registro
  directamente (`rax`, `rdi`), **no** los nombres de las variables Vesta.

Se compila y ejecuta sin ninguna opcion extra:

```sh
vm --vx 01_popcount.vx -o popcount
vm --run popcount.velb
```


## Formas de escribir ensamblador

Vesta ofrece tres puntos de entrada, cada uno para un caso distinto:

| Forma | Donde | Para que |
|:------|:------|:---------|
| `asm { ... }` | Sentencia dentro de una funcion | Intrinsecos e instrucciones sueltas mezclados con codigo Vesta normal. |
| `@Naked` + `asm { ... }` | Funcion entera sin marco | ISRs, stubs de entrada, rutinas con convencion de llamada manual: el cuerpo es asm puro y el compilador no anade prologo/epilogo/`ret`. |
| `asm nombre { ... }` | Declaracion a nivel de modulo | Bloques NASM crudos con control de seccion/offset y ancho de bits (16/32/64), para sectores de arranque y codigo bare-metal. |

> Nota: no existe una anotacion `@Asm` de "funcion entera".  El equivalente es
> una funcion `@Naked` cuyo cuerpo es un unico bloque `asm { ... }` (ver mas
> abajo).

Las secciones siguientes cubren la forma principal (`asm { ... }`) en detalle y
luego describen `@Naked` y los bloques a nivel de modulo.


## El bloque `asm { ... }`

El bloque es una **sentencia** que puedes colocar en cualquier punto del cuerpo
de una funcion, entre otras sentencias Vesta.  Estructura general:

```vesta
asm [calificadores] {
    ... instrucciones NASM Intel ...
} [clobbers("...", ...)] ;
```

Los calificadores y la clausula `clobbers(...)` son opcionales.  El `;` final
tambien es opcional.

### Contenido del cuerpo

El cuerpo se captura **verbatim** hasta la llave de cierre y se pasa al
ensamblador tal cual.  Reglas:

- **Sintaxis NASM Intel**: `mov rax, rdi`, `add rax, 10`, `paddd xmm0, [rsi]`.
- **Comentarios**: puedes usar `;` (estilo NASM) al final de una instruccion.
- **Una instruccion por linea** (los saltos de linea separan instrucciones,
  como en NASM).
- **Etiquetas locales**: escribe `.bucle:` para marcar un punto de salto dentro
  del bloque; el ensamblador las resuelve dentro del propio bloque.
- **Directivas de datos**: `db`, `dw`, `dd`, `dq` para emitir bytes/palabras
  en linea, y `times N ...` para repetir.

### Bases numericas

Los enteros del cuerpo se interpretan segun su prefijo: `0x` (hex), `0b`
(binario), `0o` (octal) y **decimal** para el resto (por ejemplo `shl rdx, 32`
desplaza 32 posiciones, en decimal).  El separador `_` se admite dentro de un
literal.  Escribe las constantes con la base que quieras; el compilador las
normaliza antes de ensamblar.


## Ligar variables a registros: `register("reg")`

Para que el asm intercambie datos con el codigo Vesta, declara una variable con
la clase de almacenamiento `register("reg")`, indicando el registro fisico:

```vesta
register("rdi") u64 valor = 0xFF;   // entrada: se lee en rdi
register("rax") u64 cuenta;          // salida: se escribe desde rax
```

- El tipo debe ser **primitivo** (enteros `i8..i64`/`u8..u64`, `bool`, `char`
  o un puntero `T*`).
- El nombre del registro es una cadena.  Se aceptan los registros de proposito
  general de 64 bits (`rax`, `rbx`, `rcx`, `rdx`, `rsi`, `rdi`, `rbp`, `rsp`,
  `r8`..`r15`) y sus alias de menor ancho (`eax`/`ax`/`al`, `r8d`/`r8w`/`r8b`,
  etc.), que se resuelven al registro fisico correspondiente.

### Entrada, salida e inout

El papel de cada variable ligada lo determina su uso, no una palabra clave:

| Patron | Papel |
|:-------|:------|
| `register("rdi") u64 x = expr;` y el asm la **lee** | Entrada. |
| `register("rax") u64 y;` (sin init) y el asm la **escribe** | Salida. |
| `register("rax") u64 z = expr;` y el asm la **lee y reescribe** | Inout. |

Ejemplo de inout con `bswap`, que invierte los bytes de `rax` in-place:

```vesta
register("rax") u64 v = 0x1122334455667788;
asm volatile {
    bswap rax                        // v = byteswap(v)
};
// v == 0x8877665544332211
```

### Reglas y garantias

- **Un registro fisico por variable viva**: si dos variables `register(...)`
  vivas a la vez piden el mismo registro fisico, es un error de compilacion.
- **No se puede tomar la direccion** de una variable ligada a registro
  (`&x`): vive en un registro, no en memoria.  El compilador lo rechaza.
- El asignador de registros **respeta el registro pedido** y protege el valor a
  traves del bloque: variables Vesta ordinarias que sigan vivas despues del asm
  no se corrompen aunque el asm pise registros (ver `clobbers`).


## Operandos con la lista `( ... )` (el compilador elige el registro)

Ademas de ligar variables con `register("reg")`, un bloque `asm` puede declarar
sus operandos en una **lista entre parentesis** justo antes del `{`.  Asi el
`{ ... }` queda con **solo asm real** (nada de declaraciones mezcladas), y la
mayor ventaja: puedes pedir que **el compilador elija el registro** en vez de
fijarlo tu.

```vesta
i64 cas(i64* addr, i64 expected, i64 desired) {
    asm volatile (
        reg p = addr,             // in: 'reg' = el COMPILADOR elige el registro
        reg d = desired,          // in
        rax a = expected,         // rax (lo exige cmpxchg); tras el bloque 'a' = resultado
    ) clobber(flags, memory) {
        lock cmpxchg [p], d       // aqui SOLO asm; 'p','d','a' son los operandos
    }
    return a;                     // read-back: 'a' contiene el valor viejo
}
```

Cada enlace se lee como una **declaracion**: `<clase> <nombre> [= <init>]`.

- **La clase es el "tipo" del enlace**:
  - `reg` — cualquier registro entero; **el compilador elige el mejor libre**.
  - `rax`, `rcx`, ..., `r15` — un registro concreto (cuando la instruccion lo
    exige, p.ej. `cmpxchg` usa `rax`).
  - `xmm`/`ymm`/`zmm` — registros vectoriales.
  - `mem` — operando en memoria (reservado; aun no disponible).
- **La direccion se infiere del uso** (no hay que escribir `in`/`out`):
  - con inicializador (`= expr`) el operando **entra** con ese valor;
  - si **lees el nombre despues** del bloque, ese es su valor de **salida**
    (*read-back*);
  - `reg t,` a secas (sin init y sin leerse despues) = **scratch** que el
    compilador reserva.
- Los nombres del enlace se usan **tal cual** dentro del asm.  `clobber(...)`
  (o `clobbers(...)`) va antes del `{` y acepta identificadores desnudos
  (`clobber(flags, memory)`).

Ejemplos por combinacion (ver `examples_codes_vesta/asm/10..13`):

```vesta
// 'reg' auto + scratch + read-back
asm volatile ( reg x = 15, reg y = 27, reg s, ) clobber(flags) {
    mov s, x
    add s, y
}
// s = 42

// inout via read-back (xadd deja el valor viejo en el registro)
asm volatile ( reg p = addr, reg v = delta, ) clobber(memory) {
    lock xadd [p], v
}
// v = valor viejo
```

Comparado con `register("reg")`: el modelo de lista es equivalente pero mas
legible (interfaz separada del cuerpo) y **permite `reg` = eleccion automatica
del registro**, evitando gestionar registros a mano y los errores de portar
codigo entre convenciones de llamada (un registro que es *callee-saved* en una
ABI y *caller-saved* en otra).  Funciona en los cuatro modos: interprete, JIT y
AOT (PE/ELF).


## Calificadores (qualifiers)

Los calificadores van entre `asm` y `{`, en cualquier orden.  Describen que
**no** hace el bloque, para que el optimizador pueda ser mas agresivo alrededor
de el.  Por defecto un bloque es conservador (`volatile`): no se elimina ni se
reordena.

### `volatile`  (por defecto)

El bloque tiene efectos observables: **no se elimina** aunque su salida no se
use, **no se reordena** respecto al codigo vecino y **no se fusiona** con otro
bloque identico.  Es el comportamiento por defecto; escribir `volatile`
explicitamente solo lo documenta.  Usalo para instrucciones con efectos
laterales (E/S, `rdtsc`, `syscall`, escrituras a memoria).

### `nomem`

Afirmas que el bloque **no accede a memoria** observable (solo opera sobre
registros).  Permite al optimizador mover cargas y almacenamientos vecinos con
mas libertad y, si ademas las entradas son invariantes de un bucle, sacar el
bloque fuera del bucle.  No lo pongas si el asm lee o escribe memoria.

### `preserves_flags`

Afirmas que el bloque **no altera los flags de la CPU** (RFLAGS).  Sin esto, el
compilador asume que el asm puede pisar los flags y no confia en su valor
previo a traves del bloque.  Con `preserves_flags` puede seguir usando una
comparacion hecha antes del asm.

### `pure`

El bloque **no tiene efectos**: implica `nomem` **y** `preserves_flags`.  Es el
mas permisivo.  Habilita eliminar el bloque si su salida no se usa, fusionar
dos bloques identicos con las mismas entradas y sacarlo de un bucle.  Reservalo
para calculos puros registro->registro (por ejemplo un `popcnt` o un `pext`).

### `noinfer`

Desactiva la **inferencia automatica** de `clobbers` (ver la seccion
siguiente).  Con `noinfer`, el compilador **no** analiza el cuerpo para deducir
que registros/flags pisa el bloque; asume que la clausula `clobbers(...)` que tu
escribes es la lista **completa y exacta**.  Es una valvula de experto: solo la
necesitas si la inferencia no encaja con lo que quieres.

### Tabla resumen

Efecto de cada calificador sobre las optimizaciones (eliminar codigo muerto,
fusionar expresiones comunes, sacar codigo de bucles, reordenar):

| Calificador | Eliminar | Fusionar | Sacar de bucle | Reordenar |
|:------------|:--------:|:--------:|:--------------:|:---------:|
| `volatile` (por defecto) | no | no | no | no |
| `nomem` | no | no | si (si entradas invariantes) | con cargas/almacenamientos vecinos |
| `preserves_flags` | no | no | no | con usuarios de flags previos |
| `pure` (= `nomem` + `preserves_flags`) | si (si salida muerta) | si (si mismas entradas) | si | si |

`noinfer` es ortogonal a la tabla: no cambia las optimizaciones, solo apaga la
inferencia de `clobbers`.


## La clausula `clobbers(...)`

Un "clobber" es un registro o efecto que el bloque **pisa** sin haberlo ligado
con `register()`.  El asignador de registros necesita saberlo para no confiar
valores vivos a esos registros a traves del asm.

### La inferencia es automatica: `clobbers` es opcional

En el caso comun **no escribes `clobbers`**.  El compilador **analiza el cuerpo
del asm** y deduce que registros y flags pisa, y los preserva por ti.  Esto
incluye:

- Registros escritos por las instrucciones (por ejemplo `mov r12, ...` marca
  `r12`).
- Los flags, si alguna instruccion los modifica (aritmetica, `cmp`, etc.).
- Registros reservados por el runtime (`rbx`, `rbp`): si el asm los pisa (por
  ejemplo `cpuid`, que escribe `ebx`), el compilador los salva y restaura
  alrededor del bloque automaticamente.

Ejemplo: este bloque pisa `r12`/`r13` y los flags, todo **inferido**, y la
variable `keep` sigue intacta a traves de el:

```vesta
u64 medir(u64 a, u64 b) {
    u64 keep = a * b;                    // viva a traves del asm
    register("rdi") u64 valor = 0xFF;
    register("rax") u64 cuenta;
    asm volatile {
        mov r12, 111                     // pisa r12/r13 SIN ligarlos;
        mov r13, 222                     // el compilador lo infiere
        popcnt rax, rdi
    };
    return cuenta + keep;                // keep no se corrompe
}
```

### Cuando declararlos explicitamente

`clobbers(...)` **anade** informacion a lo que el compilador ya infiere.  Es
util para lo que el analisis del texto no puede saber, sobre todo **llamadas
externas** (`call`), cuya convencion de llamada pisa registros que no aparecen
literalmente en el cuerpo.  Sintaxis:

```vesta
asm volatile {
    syscall
} clobbers("rcx", "r11", "memory");
```

Cada entrada es una cadena.  Ademas de nombres de registro, hay dos efectos
especiales:

- **`"memory"`**: barrera de memoria.  El compilador no reordena cargas ni
  almacenamientos alrededor del bloque.  Usalo si el asm accede a memoria que el
  codigo Vesta vecino tambien toca.
- **`"flags"`** (o su alias **`"cc"`**): el bloque pisa los flags de la CPU.

Si escribes `clobbers(...)` **sin** `noinfer`, la lista se **une** a la
inferida.  Con `noinfer`, la lista es exactamente la que escribes.  Los
calificadores `nomem` y `preserves_flags` actuan en sentido inverso:
**quitan** `memory`/`flags` del conjunto inferido (afirmas que el bloque no los
toca aunque lo parezca).


## Sustitucion de constantes `comptime`

Las constantes `comptime` se **sustituyen textualmente** por su literal dentro
del cuerpo del asm antes de ensamblar, como una macro.  Asi evitas incrustar el
numero a mano y mantienes el codigo legible:

```vesta
comptime u32 MASK = 0x00FF00FF;          // cabe en imm32

u64 main() {
    register("rax") u64 v = 0x12345678;
    asm volatile {
        and rax, MASK                    // MASK -> 0x00FF00FF (comptime)
    };
    return v;                            // 0x12345678 & 0x00FF00FF = 0x340078
}
```

El identificador `MASK` del cuerpo se reemplaza por su valor hexadecimal antes
de que el ensamblador vea el texto.


## Modos de ejecucion y backends

El bloque asm siempre acaba ejecutandose como **codigo nativo del anfitrion**.
Funciona en los tres modos de VestaVM:

- **Por defecto** y **`-m jit`**: el compilador JIT traduce la funcion a codigo
  nativo; el asm se ensambla e integra directamente, y el asignador de registros
  honra los `register()` pedidos.
- **`-m vm`** (interprete puro, sin compilador JIT): el bloque se ensambla a un
  pequeno trampolin nativo al **cargar** el `.velb`, y el interprete lo invoca
  a traves de un ayudante que copia las variables `register()` hacia y desde el
  trampolin.

Ademas:

- **`--port c`**: el bloque se traduce a `__asm__ __volatile__` de GCC/Clang con
  `.intel_syntax noprefix`, ligando cada `register()` a una variable de registro
  de C.  El compilador de C ensambla el asm.
- **`-m aot`**: compilacion nativa a un binario independiente (PE/ELF); el asm
  forma parte del codigo maquina generado.

En todos los casos el resultado observable es el mismo.


## Referenciar simbolos propios

Solo en los backends **nativos** (JIT y AOT), el asm puede referenciar
**funciones y variables globales** del propio modulo.  El programador elige la
**forma** de la instruccion segun lo que necesite:

| Forma | Significado |
|:------|:------------|
| `call sym` / `jmp sym` | Llamada/salto directo relativo a una funcion del modulo. |
| `mov r64, sym` | Carga la **direccion absoluta** de `sym` en un registro. |
| `lea r64, [rel sym]` | Carga la direccion en forma **rip-relativa** (PIC). |
| `mov r64, [global]` | Carga el **valor** (8 bytes) de una variable global. |

Ejemplo (compilado a nativo con `-m aot`):

```vesta
i64 add(i64 a, i64 b) { return a + b; }

i64 g_fp = 0;

@Naked i64 via_call(i64 a, i64 b) {
    asm volatile {
        call add                 // llamada directa a la funcion del modulo
        ret
    }
}

@Naked i64 via_lea(i64 a, i64 b) {
    asm volatile {
        lea rax, [rel add]       // direccion rip-relativa + llamada indirecta
        call rax
        ret
    }
}
```

El compilador se encarga de emitir la relocacion correcta para cada forma, de
modo que el enlazador resuelva el simbolo.


## Funciones sin marco: `@Naked`

Una funcion marcada `@Naked` se emite **sin prologo, sin epilogo y sin el `ret`
implicito** que el compilador anade normalmente.  El cuerpo (tipicamente un
unico bloque `asm { ... }`) se emite tal cual y **tu** provees la salida real
(`ret`, `iretq`, `iret`).  Es el equivalente de `__attribute__((naked))` de
GCC.

Garantias del codegen:

- Sin `push rbp` / `mov rbp, rsp` / `sub rsp` (nada de prologo).
- Sin epilogo (nada de restaurar el marco).
- Sin el `ret` implicito de cierre.
- La funcion **no se elimina** aunque ninguna llamada la referencie (un ISR se
  referencia desde la tabla de interrupciones, invisible al compilador): siempre
  se emite, con simbolo global.

Es el primitivo para escribir **manejadores de interrupcion (ISR)**, stubs de
entrada y rutinas con convencion de llamada manual:

```vesta
// ISR de timer: envia EOI al controlador de interrupciones y retorna con iretq.
@Naked void isr_timer() {
    asm volatile {
        push rax
        mov al, 0x20
        out 0x20, al
        pop rax
        iretq
    }
}

// Helper de bajo nivel con ABI manual (SysV: args en rdi/rsi, resultado en rax).
@Naked i64 add_naked(i64 a, i64 b) {
    asm volatile {
        mov rax, rdi
        add rax, rsi
        ret
    }
}
```

Una funcion `@Naked` puede llamar a otra funcion del modulo por su nombre
(`call add`), como se vio en la seccion de simbolos.

### Recibir N argumentos crudos: `...`

Una funcion `@Naked` puede recibir un numero **variable** de argumentos de
**cualquier tipo** con un `...` **pelado** (sin tipo ni nombre) como ultimo
parametro -- igual que una `F(a, ...)` de C.  A diferencia del variadico normal
`T... args` (que empaqueta los args en un array y ofrece `vacount()`), el `...`
crudo **no empaqueta nada**: cada argumento aterriza en su registro de argumento
del ABI segun su tipo, y el cuerpo `asm` los lee de ahi directamente.

```vesta
// Wrapper de syscall: sysno + N args crudos.  El cuerpo baraja los registros
// del ABI a la convencion del syscall.
@Naked
u64 syscall_exec(u32 sysno, ...) {
    register("eax") u32 nr = sysno;   // arg fijo -> su arg-reg
    asm {
        // los args crudos estan en los arg-regs RESTANTES del ABI
        mov r10, rcx        // (ejemplo) primer arg variadico
        syscall
        ret
    }
}

// El caller coloca cada arg segun su tipo en el call site:
syscall_exec(60, ptr, valor);
```

Como los registros dependen del ABI, un wrapper portable selecciona el cuerpo con
`@Target`:

| Posicion | Win64 (entero/puntero) | SysV / Linux (entero/puntero) |
| :------- | :--------------------- | :---------------------------- |
| arg 0    | `rcx`                  | `rdi`                         |
| arg 1    | `rdx`                  | `rsi`                         |
| arg 2    | `r8`                   | `rdx`                         |
| arg 3    | `r9`                   | `rcx`                         |
| arg 4+   | pila                   | `r8`, `r9`, pila              |

```vesta
@Target("os:windows")
@Naked
u64 add_raw(u64 a, ...) {
    asm { mov rax, rcx; add rax, rdx; ret }   // a=rcx, vararg0=rdx
}

@Target("!os:windows")
@Naked
u64 add_raw(u64 a, ...) {
    asm { mov rax, rdi; add rax, rsi; ret }   // a=rdi, vararg0=rsi
}

u64 r = add_raw(40, 2);   // 42
```

Reglas del `...` crudo:

- **Solo en `@Naked`**: fuera de una funcion `@Naked` es un error de compilacion
  (el cuerpo debe acceder a los registros de argumento por su cuenta).
- **Debe ser el ultimo** parametro.
- **No hay `vacount()` ni indexado**: `...` no introduce un nombre ni un array.
  Para eso usa el variadico empaquetado `T... args`.
- **Type-agnostico**: acepta enteros, punteros, floats, etc.  El compilador
  coloca cada argumento segun su **clase**, con conteo **separado por clase**:
  - enteros y punteros llenan los arg-regs **GP** en orden (tabla de arriba:
    Win64 `rcx, rdx, r8, r9`; SysV `rdi, rsi, rdx, rcx, r8, r9`);
  - los floats llenan los arg-regs **XMM** en orden -- el **primer** float va a
    `xmm0`, el segundo a `xmm1`, etc. (el conteo XMM es independiente del GP, no
    de la posicion global del argumento).

  Ejemplo: en `f(u64 a, f64 x, u64 b)` con `...`, `a`->GP0, `b`->GP1, y `x`->xmm0.

```vesta
@Naked
u64 with_float(u64 a, ...) {
    asm {
        cvttsd2si rdx, xmm0   // primer float variadico -> xmm0
        mov rax, rcx          // (Win64) arg fijo a
        add rax, rdx
        ret
    }
}
u64 r = with_float(40, 2.9);   // 40 + trunc(2.9) = 42
```


## Bloques `asm` a nivel de modulo (16/32/64 bits)

Para codigo bare-metal (sectores de arranque, cambio de modo real->protegido->
largo) existe una forma de **declaracion** a nivel de modulo: un bloque NASM
crudo con nombre, con control de **ancho de bits** y de **ubicacion**:

```vesta
@bits(16) @section(".boot", "rx")
asm boot {
    cli
    xor ax, ax
    hang: hlt
          jmp hang        // etiquetas dentro del bloque (el ensamblador las resuelve)
}
```

- `@bits(16 | 32 | 64)` selecciona el modo del ensamblador para ese bloque
  (16 para el arranque BIOS en modo real, 32 para modo protegido, 64 por
  defecto).
- `@section("nombre", "permisos")`, `@at(offset)` y `@order(N)` controlan en
  que seccion y en que posicion se coloca el bloque (util para firmas de boot,
  padding, tablas en offsets fijos).  Para relleno y firmas se combina con
  bloques de datos crudos.
- El cuerpo se ensambla con el ensamblador integrado; admite instrucciones,
  etiquetas internas y directivas de datos (`db`/`dw`/`dd`/`dq`).

Esta forma es independiente del `asm { ... }` en linea: no liga variables Vesta
(no usa `register()`), es codigo maquina que se emite en su seccion.


## Mas ejemplos

Los siguientes ejemplos completos estan en `examples_codes_vx/asm/`.  Todos se
compilan y ejecutan sin opciones adicionales.

### Time-Stamp Counter (`rdtsc`)

Lee el contador de ciclos de 64 bits, devuelto en el par `edx:eax`:

```vesta
u64 main() {
    register("rax") u64 tsc;             // recibe el TSC de 64 bits
    asm volatile {
        rdtsc                            // edx:eax = TSC
        shl rdx, 32                      // parte alta << 32
        or  rax, rdx                     // (alta << 32) | baja
    };                                   // clobber de rdx y flags: inferidos
    return tsc;
}
```

### `cpuid` (pisa un registro reservado)

`cpuid` escribe `eax`/`ebx`/`ecx`/`edx`.  El registro `rbx` lo reserva el
runtime, pero el compilador lo salva y restaura automaticamente alrededor del
bloque:

```vesta
u64 main() {
    register("rax") u64 a = 0;   // eax=0 (entrada) -> max leaf (salida): inout
    asm volatile {
        cpuid
    };
    return a;
}
```

### SIMD sobre memoria (SSE2 `paddd`)

Patron recomendado para SIMD: los datos viven en memoria (host, via `malloc`) y
las variables `register()` ligan **punteros** (registros de proposito general);
el registro vectorial (`xmm0`) es scratch interno del asm:

```vesta
i32 main() {
    i32* a = malloc(16);                 // 4 x i32
    i32* b = malloc(16);
    a[0] = 1;  a[1] = 2;  a[2] = 3;  a[3] = 4;
    b[0] = 10; b[1] = 20; b[2] = 30; b[3] = 40;

    register("rdi") i32* pa = a;
    register("rsi") i32* pb = b;
    asm volatile {
        movdqu xmm0, [rdi]               // xmm0 = a[0..3]
        paddd  xmm0, [rsi]               // xmm0 += b[0..3] (4 sumas en 1 instr)
        movdqu [rdi], xmm0               // a[0..3] = resultado
    };
    return a[0] + a[1] + a[2] + a[3];     // 11+22+33+44 = 110
}
```

La version AVX2 (`vpaddd` sobre `ymm`, 8 sumas en paralelo) sigue el mismo
patron con el doble de ancho, terminando con `vzeroupper` para evitar la
penalizacion de transicion AVX<->SSE.

### BMI2 `pext` (bloque puro)

Extraccion paralela de bits.  Como es un calculo puro registro->registro,
podria marcarse `pure`:

```vesta
u64 main() {
    register("rdi") u64 src  = 0b1011;   // bits 3,1,0
    register("rsi") u64 mask = 0b1010;   // selecciona bits 3 y 1
    register("rax") u64 out;
    asm volatile {
        pext rax, rdi, rsi
    };
    return out;                          // -> 3
}
```


## Limitaciones actuales

- **Registros vectoriales como binding**: ligar una variable directamente a un
  registro vectorial (`register("xmm0") ...`) **no** esta soportado todavia (el
  asignador solo maneja el banco de proposito general).  Para SIMD, usa el
  patron de memoria: liga **punteros** con `register()` y usa los registros
  vectoriales como scratch interno del asm (ejemplos SSE2/AVX2 de arriba).
- **Interprete puro (`-m vm`)**: el bloque se ensambla al cargar el `.velb`
  usando el ensamblador integrado en `vm`.  Es portable a plataformas sin JIT,
  pero necesita ese ensamblador en tiempo de ejecucion.  Ademas, los bytes
  generados son especificos de **x86-64**: un `.velb` con inline asm x86 no
  corre en ARM.
- **Maximo de bindings en `-m vm`**: en el interprete puro, un bloque admite
  hasta **8** variables `register()` (el ayudante de marshalling las pasa como
  argumentos).  El backend JIT no tiene este limite.
- **Pisar `rsp`**: un asm que modifica el puntero de pila no se puede envolver
  de forma segura (el propio envoltorio usa la pila), asi que se rechaza.
- **Referenciar simbolos propios** (`call sym`, `mov r64, sym`, etc.) solo
  funciona en los backends nativos (JIT/AOT), no al portar a C.
