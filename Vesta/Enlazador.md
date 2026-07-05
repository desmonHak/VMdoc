# Enlazador y archivador propios

Vesta trae su **propio enlazador** (linker) y su **propio archivador** (para
crear librerias estaticas `.a`).  No dependes de `ld`, `gcc` ni `ar` del
sistema para producir un binario nativo: el mismo `vm` compila a objeto,
archiva librerias estaticas, y enlaza el ejecutable final.

Esto cubre el ciclo completo de compilacion nativa "sin toolchain externo":

    fuente .vx  --(compilar)-->  objeto .o/.obj  --(archivar)-->  libreria .a
                                        |                              |
                                        +--------(enlazar)-------------+
                                                    |
                                                    v
                                            ejecutable / .so / .dll

Aun asi los objetos que produce Vesta son **estandar** (ELF64 y COFF AMD64),
de modo que tambien puedes enlazarlos con `ld`/`gcc`/`link.exe` si lo prefieres,
y a la inversa: el enlazador de Vesta acepta objetos y librerias generados por
esas herramientas.

La compilacion nativa en si (extern a librerias del sistema, GC opt-in con
`gc<T>`, librerias precompiladas) esta cubierta en
[Compilacion nativa](CompilacionNativa.md); este documento se centra en las
**flags de salida del compilador nativo** y en los comandos `--link` y `--ar`.

La colocacion de secciones en direcciones fijas y el script de enlace escrito
en Vesta (`--link-script`) tienen su propio documento:
[Disposicion de secciones](DisposicionSecciones.md).

---

## Enlazar: `vm --link`

Fusiona uno o mas objetos relocatables (y, opcionalmente, librerias estaticas y
compartidas) en un ejecutable nativo, resolviendo simbolos cross-file y
relocaciones, sin `ld`/`gcc`.

Sintaxis:

    vm --link a.o b.o [lib.a] [lib.dll] -o prog [--format elf|pe] [--entry sym] [--link-base 0xADDR]

### Que acepta como entrada

El enlazador **autodetecta el formato de cada entrada por su cabecera** (magic),
asi que puedes mezclar libremente:

- **Objetos relocatables**: `.o` (ELF64 `ET_REL`) y `.obj` (COFF AMD64).  Da
  igual si los produjo Vesta (`--emit obj`) o `gcc`/`cl`/MSVC.
- **Librerias estaticas** `.a` (formato `ar`): se extraen solo los miembros
  necesarios (ver "pull perezoso" abajo).
- **Librerias compartidas** `.so` (ELF) y `.dll` (PE): se usan para resolver
  simbolos importados (ver "imports" abajo).

Todos los objetos de un enlace deben coincidir en tamano de palabra (32 vs 64
bits); el enlazador tambien reconoce objetos de 32 bits (ELF32 / COFF i386) y
produce el ejecutable de 32 bits correspondiente.

### Resolucion de simbolos y pull perezoso de `.a`

El enlazador construye una tabla global de simbolos con las definiciones de
todos los objetos sueltos y resuelve cada referencia cross-file.  De las
librerias estaticas `.a` **solo extrae los miembros que definen un simbolo
referenciado-y-aun-no-definido**, iterando hasta punto fijo.  Asi enlazar
contra una `.a` grande no arrastra codigo muerto: entra solo lo que hace falta.

Los duplicados de tipo "inline"/plantilla que aparecen en varios objetos (marca
COMDAT del COFF) se **pliegan**: la primera definicion gana y las demas se
descartan, evitando errores de simbolo duplicado.

### Imports: se leen los exports REALES de la libreria

Cuando un simbolo lo aporta una libreria del sistema, el enlazador **NO usa
listas de simbolos embebidas**: lee la tabla de exports real de cada `.dll`/`.so`
candidata y toma de ahi que biblioteca exporta cada simbolo.  Lo que ninguna
libreria exporta simplemente **no se auto-importa** y produce un error claro de
simbolo ausente (no un binario silenciosamente roto).

- **PE**: por cada import se sintetiza un thunk (`FF 25`) y su entrada en la
  IAT; para simbolos `__imp_X` se usa el slot directo.  Las candidatas son las
  DLL del sistema mas cualquier `.dll` que pases como entrada.
- **ELF**: por cada import se emite un thunk y una entrada en la GOT dinamica
  (`GLOB_DAT`) mas su `DT_NEEDED`.  El enlazador soporta las relocaciones que
  emite un `gcc` real (`GOTPCREL`/`GOTPCRELX`/`REX_GOTPCRELX` -> `.got`;
  `R_X86_64_RELATIVE` en datos PIE; secciones `NOBITS` -> `.bss` con
  `memsz > filesz`).

### Modo hosted vs entrada propia

- **Hosted** (por defecto, sin `--entry`): el enlazador **sintetiza un `_start`**
  que llama a `main` y termina el proceso con su valor de retorno.  Requiere un
  `main` global.
- **Entrada propia** (`--entry <sym>`): usa ese simbolo como punto de entrada
  **sin stub** (kernel, bootloader, driver cuyo entry es propio).  No hace falta
  `main`.

### Flags de `--link`

Verificadas en `main.cpp`:

| Flag | Efecto |
|:-----|:-------|
| `--format elf` \| `pe` | Formato del ejecutable de salida. |
| `--entry <sym>` | Simbolo de entrada sin stub (kernel/bootloader). Vacio => `_start` sintetico que llama a `main`. |
| `--link-base 0xADDR` | Base de carga del ejecutable (hex, p.ej. `0x100000` para un kernel). Por defecto segun el formato. |
| `--link-script <archivo.vx>` | Script de enlace escrito en Vesta (ver [Disposicion de secciones](DisposicionSecciones.md)). Los CLI `--link-base`/`--entry` tienen prioridad sobre el script. |
| `--link-debug` | Con `--link-script`: hace que el builtin `debug_build()` del script devuelva `true`. |
| `--sysroot <dir>` | Raiz donde buscar las librerias del sistema (p.ej. `libc.so.6`) al cross-compilar ELF desde otro SO. En Linux nativo no hace falta. |

Ejemplo (kernel enlazado con un runtime en C, sin `ld`):

    vm --vx kernel.vx -m aot --emit obj --format elf -o kernel.o
    gcc -c -ffreestanding rt.c -o rt.o          # runtime/driver en C
    vm --link kernel.o rt.o -o kernel.elf --format elf --entry _kstart

---

## Archivar: `vm --ar`

Crea una libreria estatica `.a` a partir de objetos, **sin el `ar` del sistema**.

Sintaxis:

    vm --ar libfoo.a a.o b.o ...

Produce un archivo en formato `ar` GNU **con indice de simbolos** (la tabla `/`
que mapea cada simbolo exportado a su miembro).  El resultado es interoperable:

- Lo consume el enlazador de Vesta (`vm --link libfoo.a ...`), que hace pull
  perezoso de sus miembros.
- Lo consumen tambien `ar`, `ld`, `gcc` y `nm` del sistema.

Ejemplo (empaquetar dos objetos en una libreria y enlazarla despues):

    vm --vx mat.vx  -m aot --emit obj --format elf -o mat.o
    vm --vx util.vx -m aot --emit obj --format elf -o util.o
    vm --ar libmate.a mat.o util.o
    vm --link app.o libmate.a -o app.elf --format elf

---

## Flags de salida del compilador nativo (`-m aot`)

Estas flags controlan **que artefacto** produce el compilador nativo y **como**.
Todas verificadas en `main.cpp`.

### `--emit` : tipo de artefacto

| Valor | Artefacto |
|:------|:----------|
| `exe` (por defecto) | Ejecutable standalone. |
| `obj` | Objeto relocatable linkable: `.o` (ELF) o `.obj` (COFF) segun `--format`. Sin `_start`; `main` como simbolo global/external. |
| `shared` | Libreria compartida: `.so` (ELF, `dlopen`/`dlsym`) o `.dll` (PE, `LoadLibrary`/`GetProcAddress`). Exporta sus funciones; sin `main`. |
| `bin` | Binario plano **sin cabecera**: bootloader / ROM / firmware. La entrada esta en el offset 0. |

### `--format` : formato de contenedor

| Valor | Formato |
|:------|:--------|
| `pe` | PE (Windows). Por defecto en Windows. |
| `elf` | ELF (Linux/POSIX). Por defecto fuera de Windows. |

El ABI del codigo lo decide el **formato objetivo**, no el host: `--format elf`
usa la convencion SysV y `--format pe` la de Windows.  Puedes generar ELF en
Windows y PE en Linux (cross-target).

### `--no-pie` : referencias absolutas

Por defecto las referencias a datos son **RIP-relativas** (position-independent,
listas para PIE/`.so`).  Con `--no-pie` se emiten **absolutas** (`mov reg,imm64`,
requieren base de imagen fija; el emisor PE limpia `DYNAMIC_BASE` para fijarla).
Es el analogo a `gcc`/`clang -no-pie`.

### `--bin-base 0xADDR` : base del binario plano

Solo con `--emit bin`: fija la base de carga del binario plano (hex, p.ej.
`0x7C00` para un sector de arranque).  Afecta unicamente a las referencias
absolutas (`--no-pie`); las RIP-relativas son invariantes.  Por defecto `0`.

### `--target` : tier de runtime

| Valor | Runtime |
|:------|:--------|
| `bare` (por defecto) | Minimo: enteros/float, punteros, structs, funciones, FFI. Sin GC ni objetos gestionados. |
| `embed` | Runtime minimo embebido: mas mecanismos del lenguaje en un binario standalone. |
| `full` | Runtime completo. |

### `--freestanding` : bare sin libc

Modo `bare` **sin libc**, para kernels y bootloaders.  Las operaciones de
memoria y de aborto (`RAW_ALLOC`/`RAW_FREE`/`PANIC`) no se resuelven a `malloc`/
`free`/`abort`, sino a **hooks que provee el programador** (`@AllocatorOverride`,
`@PanicHandler`).  El binario no referencia ningun simbolo de libc.

### `--no-exceptions`, `--no-io`, `--no-mem` : quitar mecanismos/runtime

Para kernels y binarios minimos, se puede recortar lo que se auto-incluye:

| Flag | Efecto |
|:-----|:-------|
| `--no-exceptions` | Desactiva el mecanismo de excepciones nativo. Un `try`/`catch`/`throw` da error de compilacion. |
| `--no-io` | No auto-incluye el runtime de entrada/salida. El usuario aporta `__vx_write` y los `__vx_print_*` (p.ej. enlazando un objeto de I/O propio, o en freestanding). |
| `--no-mem` | No auto-incluye el slab allocator. El allocator cae a `malloc`/`free` de libc (o al `@AllocatorOverride` del usuario). |

### `--aot-arch` : arquitectura objetivo

| Valor | Arquitectura |
|:------|:-------------|
| `x86-64` (por defecto) | 64 bits. |
| `x86-32` | Modo protegido de 32 bits (kernels): 8 registros GP `eax`-`edi`, convencion `regparm(3)`, subset entero de 32 bits. |

### `--float-isa` : backend de punto flotante / ancho SIMD

| Valor | Backend |
|:------|:--------|
| `sse2` (por defecto) | 128 bits; corre en **cualquier** x86-64. |
| `x87` | Legacy. |
| `avx` | AVX2 256 bits; requiere AVX2 en la CPU. |
| `avx512f` | 512 bits; requiere AVX-512 en la CPU. |
| `auto` | Multiversion: emite las variantes y elige la optima en runtime por CPUID. Lo mejor para distribuir un solo binario. |

Nota: un binario `avx`/`avx512f` **fijo** provoca instruccion ilegal (SIGILL) en
una CPU sin ese soporte.  Usa `auto` para portabilidad en un binario unico.

---

## Ejemplos completos de principio a fin

### Ejecutable directo

    # PE (Windows)
    vm --vx examples_codes_vx/aot/01_call.vx -m aot -o 01_call.exe
    ./01_call.exe; echo $?        # 42

    # ELF (Linux)
    vm --vx examples_codes_vx/aot/01_call.vx -m aot --format elf -o 01_call.elf
    ./01_call.elf; echo $?        # 42

### Compilar a objeto, archivar, enlazar y ejecutar (todo con Vesta)

    # 1) libreria .vx SIN main -> objeto que exporta sus funciones
    vm --vx lib.vx -m aot --emit obj --format elf -o lib.o

    # 2) empaquetar la libreria en un .a con indice de simbolos
    vm --ar liblib.a lib.o

    # 3) aplicacion con main que referencia la libreria (extern)
    vm --vx app.vx -m aot --emit obj --format elf -o app.o

    # 4) enlazar SIN ld/gcc; pull perezoso de la .a
    vm --link app.o liblib.a -o app.elf --format elf

    # 5) ejecutar
    ./app.elf; echo $?

### Enlazar objetos de Vesta con objetos de C

    # objeto Vesta (COFF) + objeto de C (COFF de MinGW/TDM-GCC)
    vm --vx kernel.vx -m aot --emit obj --format pe -o kernel.obj
    gcc -c rt.c -o rt.obj
    vm --link kernel.obj rt.obj -o kernel.exe --format pe

### Objeto para enlazar con la toolchain del sistema

Los objetos que produce Vesta tambien los enlaza `gcc`/`link.exe`:

    vm --vx 01_call.vx -m aot --format elf --emit obj -o 01_call.o
    gcc 01_call.o -o prog        # el crt de gcc llama a main
    ./prog; echo $?              # 42

### Libreria compartida

    # .so ELF (dlopen/dlsym)
    vm --vx 09_shared_lib.vx -m aot --format elf --emit shared -o libfoo.so

    # .dll PE (LoadLibrary/GetProcAddress)
    vm --vx 10_shared_dll.vx -m aot --format pe --emit shared -o foo.dll

### Binario plano para un sector de arranque

    vm --vx boot.vx -m aot --emit bin --no-pie --bin-base 0x7C00 -o boot.bin

### Kernel de 32 bits con entrada propia

    vm --vx kernel.vx -m aot --aot-arch x86-32 --emit obj --format elf -o k.o
    vm --link k.o -o k.elf --format elf --entry _kstart --link-base 0x100000

---

## GC opt-in en un binario standalone

Un programa que usa `gc<T>` se enlaza automaticamente con la libreria del
recolector (libc-only) al producir un ejecutable con `--emit exe`: el enlazador
detecta las llamadas al GC y agrega la libreria, localizandola junto al binario
`vm`.  El resultado es un `.exe`/ELF autonomo con el recolector dentro, tanto en
PE como en ELF.  El detalle de `gc<T>` esta en
[Compilacion nativa](CompilacionNativa.md).

---

## Notas de plataforma (PE vs ELF)

Diferencias que el enlazador maneja de forma transparente, utiles al depurar:

- **Addend de relocacion**: en COFF (PE) el addend vive **en el campo** que se
  parchea; en ELF `RELA` va en un registro aparte.  El enlazador escribe el
  addend en el sitio para COFF y lo lee del registro para ELF.
- **Nombres agrupados COFF**: `.text$mn`, `.rdata$zzz`, etc. se pliegan a su
  seccion base (`.text`, `.rdata`).  Las secciones vacias que emite `gcc`
  (`.data`/`.bss` de tamano 0) se descartan.
- **Tablas SEH** (`.pdata`/`.xdata`, relocs `ADDR32NB`/RVA) se descartan: los
  frames nativos AOT no llevan unwinding nativo.
- **Imports**: PE via thunk `FF 25` + IAT; ELF via thunk + GOT dinamica
  (`GLOB_DAT`) + `DT_NEEDED`.  En ambos casos la biblioteca de cada simbolo se
  determina leyendo los exports reales de las DLL/`.so` candidatas.
- **Secciones sin inicializar** (`.bss`, `NOBITS`): en ELF EXEC van en el
  segmento R+W con `p_memsz > p_filesz` y el cargador las pone a cero.
- **Cross-compilar ELF desde otro SO**: usa `--sysroot` para indicar donde estan
  las librerias del sistema (p.ej. `libc.so.6`); en Linux nativo no es necesario.

### Limitaciones conocidas

- `--sysroot` cubre la busqueda de librerias del sistema al cross-compilar ELF;
  el `DT_NEEDED` por soname para enlazar contra `.so` propias en ELF esta en
  curso (en PE las `.dll` propias ya funcionan).
- Los objetos de un mismo enlace deben coincidir en 32 vs 64 bits.
