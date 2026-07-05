# Disposicion de secciones y datos crudos en Vesta

## Introduccion: describir el layout en el propio lenguaje

En las cadenas de herramientas clasicas (GNU `ld`), cuando necesitas controlar
donde va cada byte de tu binario -- en que direccion se carga, en que orden van
las secciones, que hay en cada offset -- escribes un fichero aparte: un *linker
script* (`.ld`) con su propia sintaxis, ajena a tu lenguaje.

Vesta toma el enfoque opuesto: **describes la disposicion dentro del propio
lenguaje**. Las anotaciones de colocacion (`@section`, `@at`, `@order`), los
bloques de datos crudos (`bytes`), los bloques de ensamblador por modo (`@bits
... asm`) y el script de enlace (una funcion `link()` en Vesta) viven en tu
codigo `.vx`. No hay un segundo lenguaje que aprender: usas variables, `if`,
constantes de compilacion y toda la logica de Vesta para *calcular* el layout.

### Cuando lo necesitas

Este control fino solo hace falta para software de muy bajo nivel:

- **Kernels**: colocar el punto de entrada del kernel en una direccion fija,
  poner una cabecera (por ejemplo multiboot) en una VA exacta, situar `.text` y
  `.data` en direcciones del mapa de memoria.
- **Bootloaders**: un boot sector BIOS ocupa exactamente 512 bytes con la firma
  `0xAA55` en el offset 510; el codigo arranca en la direccion `0x7C00`.
- **Firmware y ROMs**: imagenes planas sin cabecera que se cargan en una
  direccion fisica fija.
- **Sistemas operativos de juguete (dev-OS)**: mezcla de bloques de 16, 32 y 64
  bits, secciones con permisos concretos, tablas de punteros a funcion (IDT,
  jump tables) construidas a mano.

Para un programa de linea de comandos normal no necesitas nada de esto: el
compilador elige el layout por ti. Las herramientas de este documento son el
escape hacia el metal.

> Las opciones de salida (`--emit exe|obj|shared|bin`, `--format elf|pe`,
> `--aot-arch`) y el enlazador propio (`vm --link`) se documentan en
> [Enlazador.md](Enlazador.md). Aqui nos centramos en como *describir* el
> layout desde el lenguaje.

---

## Anotaciones de colocacion

### `@section("nombre")` -- en que seccion vive un simbolo

`@section` coloca una funcion o un bloque de datos en una seccion con nombre
propio, en lugar de la seccion por defecto (`.text` para codigo, `.rodata` para
datos). Los permisos se derivan por convencion del nombre (`.text*` -> lectura +
ejecucion, `.rodata*` -> lectura, `.data*`/`.bss*` -> lectura + escritura) o se
dan explicitamente como segundo argumento:

```vx
// Funcion colocada en una seccion RWX propia (arranque de un dev-OS).
@section(".boot", "rwx")
i32 boot_entry(i32 x) { return x + 7; }

i32 main() { return boot_entry(35); }   // llamada cross-seccion; exit 42
```

Una funcion con `@section` **no se inlinea**: permanece fisicamente en su
seccion. Las llamadas entre secciones distintas las resuelve el compilador
mediante relocaciones.

`@section` tambien se aplica a bloques de datos crudos (ver mas abajo). Por
defecto un bloque `bytes` va a `.rodata`; con `@section` lo mandas donde quieras:

```vx
@section(".sig", "r")
bytes boot_sig {
    db 0x55, 0xAA
}
```

### `@at(N)` -- offset absoluto fijo

`@at(N)` fija el offset absoluto de la seccion dentro de la imagen: el
compilador rellena con ceros el hueco hasta llegar a `N`, de modo que la seccion
empiece exactamente ahi. Es lo que necesitas para una firma en una posicion fija
o una cabecera que un cargador espera en un offset concreto.

### `@order(N)` -- orden relativo

`@order(N)` controla el orden relativo de las secciones (el numero menor va
primero). Sirve para decidir que va al inicio y que al final de la imagen sin
tener que calcular offsets absolutos.

```vx
// Cabecera primero, cuerpo despues, y una firma fija en el offset 512.
@section(".header") @order(0)
bytes header { db "HDR!", 0x01 }      // 5 bytes en offset 0

@section(".body") @order(1)
bytes body { dd 0xAABBCCDD }          // 4 bytes tras la cabecera (offset 5)

@section(".sig") @at(512)
bytes sig { db 0x55, 0xAA }           // firma fija en el offset 512
```

**Importante:** `@at` y `@order` solo tienen efecto en la salida de **binario
plano** (`--emit bin`). Los formatos con cabecera (ejecutable, objeto,
biblioteca compartida) tienen su propio layout de paginas y no honran estas
anotaciones; ahi el orden y la posicion los decide el emisor del formato. `@at`
y `@order` son para cuando controlas la imagen byte a byte.

---

## Bloques de datos crudos: `bytes name { ... }`

Un bloque `bytes` emite bytes verbatim, con la sintaxis de directivas de datos de
NASM. Es la forma de meter tablas, firmas, cabeceras y cualquier dato binario en
tu programa sin depender de un ensamblador externo.

### Directivas de datos

| Directiva | Ancho | Emite |
| :-------- | :---- | :---- |
| `db` | 1 byte | un byte por operando (little-endian trivial) |
| `dw` | 2 bytes | word little-endian |
| `dd` | 4 bytes | dword little-endian |
| `dq` | 8 bytes | qword little-endian |
| `times N <dir>` | -- | repite la directiva `N` veces |

Los operandos de `db` aceptan enteros, literales de caracter (`'A'`) y cadenas
(`"Hi"`, un byte por caracter):

```vx
bytes tabla {
    db 0x01, 'A', "Hi"        // 1 + 1 + 2 = 4 bytes: 01 41 48 69
    dw 0x1234                  // 34 12
    dd 0xDEADBEEF             // EF BE AD DE
    dq 0x1122334455667788      // 88 77 66 55 44 33 22 11
    times 4 db 0xFF           // FF FF FF FF
}
```

### Operadores `$` y `$$` (posicion y relleno)

Dentro de un bloque `bytes` (y de un bloque `asm`, ver mas abajo) puedes usar:

- **`$`** -- el offset actual dentro del bloque.
- **`$$`** -- el inicio del bloque (siempre `0`).

Combinados con aritmetica (`+ - * / ( )`) permiten el idioma clasico de NASM
para **rellenar hasta** una posicion. El caso arquetipico es un boot sector de
512 bytes: rellenar con ceros desde donde llegue el codigo hasta el offset 510 y
poner la firma en 510..511:

```vx
    times 510 - ($ - $$) db 0  // relleno con ceros hasta el offset 510
    db 0x55, 0xAA              // firma MBR en 510..511
```

`510 - ($ - $$)` es "510 menos cuantos bytes llevo escritos", es decir, exactamente
los bytes de padding que faltan para llegar a 510.

### Operandos: literales, constantes de compilacion y simbolos

Un operando de una directiva de datos puede ser tres cosas:

1. **Un literal** entero, caracter o cadena.
2. **Una constante de compilacion** (`comptime`): un escalar se emite como un
   literal del ancho de la directiva; un array `comptime` se expande elemento a
   elemento.
3. **El nombre de una funcion**: se emite una relocacion. `dq main` coloca la
   direccion absoluta de 64 bits de `main`; el compilador la parchea tras
   calcular el layout. Asi construyes tablas de punteros a funcion (vtables,
   jump tables, la IDT de un kernel).

```vx
comptime u32 MAGIC = 0xDEADBEEF;
comptime u8[] palette = {1, 2, 3, 4};

bytes header {
    dd MAGIC                   // 4 bytes: ef be ad de (comptime const escalar)
    db palette                 // 4 bytes: 01 02 03 04 (comptime array, 1B c/u)
    dq main                    // 8 bytes: reloc -> VA de main
    dw 0x1234                  // 2 bytes literal: 34 12
}
```

Una tabla de punteros a funcion mezclando simbolos y literales:

```vx
bytes jump_table {
    dq main                    // -> VA de main
    dq helper                  // -> VA de helper
    dq 0xDEADBEEFCAFEBABE     // literal, sin reloc
}
```

---

## Bloques de ensamblador por modo: `@bits(16|32|64) asm name { ... }`

Un bloque `asm` con la anotacion `@bits` ensambla codigo nativo en el modo
indicado. Es como Vesta escribe un boot stub BIOS de 16 bits (modo real), la
etapa de 32 bits (modo protegido) de un kernel, o rutinas de 64 bits.

Lo potente es que **dentro de un mismo bloque `asm` puedes mezclar codigo y
datos crudos**: instrucciones (las ensambla el motor de codigo maquina), labels
intra-bloque, y las mismas directivas de datos que en un bloque `bytes`
(`db`/`dw`/`dd`/`dq`/`times` con `$`/`$$`). Esto permite meter en un unico bloque
el codigo, el relleno y la firma de un boot sector completo:

```vx
@bits(16) @section(".text", "rx")
asm boot {
    cli
    xor ax, ax
    mov ds, ax
    mov es, ax
    mov ss, ax
    mov sp, 0x7C00

    mov ah, 0x0E          ; BIOS teletype
    mov al, 0x56          ; 'V'
    int 0x10

hang:
    hlt
    jmp hang              ; label intra-bloque

    times 510-($-$$) db 0 ; relleno hasta 510 (datos, evaluados por el compilador)
    dw 0xAA55             ; firma MBR en 510..511
}
```

Aqui el codigo (`cli`, `mov`, `int 0x10`, el bucle `hang`) lo ensambla el motor
de codigo maquina; el relleno (`times ... db 0`) y la firma (`dw 0xAA55`) los
evalua el compilador con la misma logica de `$`/`$$` de los bloques `bytes`.

Un boot sector de validacion que escribe por el puerto de depuracion de QEMU y
por la BIOS, y luego sale del emulador:

```vx
@bits(16) @section(".text", "rx")
asm boot {
    cli
    xor ax, ax
    mov ds, ax

    mov al, 0x56          ; 'V' -> debugcon QEMU (port 0xE9)
    out 0xE9, al

    mov ah, 0x0E          ; 'V' -> BIOS teletype (path real)
    mov al, 0x56
    int 0x10

    mov al, 0x42          ; salir de QEMU via isa-debug-exit
    out 0xF4, al

hang:
    hlt
    jmp hang

    times 510-($-$$) db 0
    dw 0xAA55
}
```

### Cambios de modo entre bloques

Cuando un dev-OS pasa de 16 a 32 y luego a 64 bits, cada modo va en un bloque
`@bits` separado, referenciados por simbolo. El motor de codigo maquina no
cambia de modo con directivas intra-bloque (`.code32`/`bits 32` se ignoran a
mitad de un mismo bloque), asi que la transicion se hace con un salto lejano por
simbolo a otro bloque:

```vx
jmp 0x08:pm32            // far jump 16->32 a otro bloque, por SIMBOLO
...
bytes gdtr { dw 0x0027
             dd gdt_ents }   // base del GDT por SIMBOLO (cross-bloque)
```

El compilador resuelve estas referencias entre bloques: un `jmp`/`call` cercano
a otro bloque se resuelve como offset relativo; `dd sym` emite la VA absoluta de
32 bits; `dq sym` la de 64 bits; un far jump `jmp SEG:sym` resuelve el offset del
destino.

---

## Binario plano `.bin`

Con `-m aot --emit bin --bin-base 0xADDR` el compilador produce un **binario sin
cabecera**: no hay formato ejecutable, no hay tabla de secciones, no hay punto de
entrada declarado. El primer byte de la imagen es el byte de arranque, y todo se
resuelve como si la imagen se cargara en la direccion `--bin-base`. Es el formato
para bootloaders, ROMs, firmware y payloads que un cargador situa en una
direccion fisica fija.

```sh
vm -m aot --vx 12_flat_bin.vx --emit bin --bin-base 0x7C00 -o boot.bin
```

En un `.bin`:

- El layout es la concatenacion de las secciones (por ejemplo `.text` con `main`
  en el offset 0, luego `.rodata` con los bloques `bytes`).
- **`@at` y `@order` controlan la posicion y el orden** de las secciones (es su
  unico contexto real de uso).
- Los bloques `bytes` y `@bits ... asm` emiten sus bytes verbatim en la imagen.
- Las referencias a simbolos absolutas se resuelven contra `--bin-base`; las
  relativas son invariantes a la base.

```vx
// Binario plano: .text (main en offset 0) ++ .rodata (bytes) verbatim.
bytes firma {
    db "VX", 0x00            // 56 45 58 00
    dw 0xCAFE                  // FE CA
    dd 0x12345678             // 78 56 34 12
}

i32 main() {
    return 0x42;
}
```

---

## El script de enlace escrito en Vesta

Cuando enlazas objetos (`vm --link ...`; ver [Enlazador.md](Enlazador.md)) y
necesitas control sobre la base de carga, el punto de entrada, el tamano de pila
o la VA de una seccion, en lugar de un fichero `.ld` de GNU escribes un `.vx`
normal con una funcion `void link()`. El enlazador compila y ejecuta esa funcion,
que llama a builtins de configuracion para describir el layout. Al ser Vesta
puro, tienes logica, condicionales y constantes de compilacion completas para
*calcular* las direcciones.

```sh
vm --link kernel.o -o kernel.elf --format elf --link-script link_layout.vx
```

```vx
void link() {
    u64 load = 0x100000;          // 1 MB: base tipica de kernel
    if (debug_build()) { load = 0x200000; }
    base(load);
    entry("_kstart");             // entry propio (sin stub _start -> main)
    stack_size(align_up(64 * 1024, 4096));
}
```

### Builtins disponibles dentro de `link()`

| Builtin | Efecto |
| :------ | :----- |
| `base(u64 addr)` | Direccion de carga de la imagen. |
| `entry(string sym)` | Simbolo de entrada (sin generar un `_start` sintetico que llame a `main`). |
| `stack_size(u64 bytes)` | Tamano de la pila. |
| `place_section(string nm, u64 addr)` | Coloca una seccion en una VA fija (cabecera multiboot, MMIO, `.text`/`.data` en direcciones del mapa de memoria del kernel). |
| `section_bytes(string nm)` | Tamano en bytes de una seccion ya fusionada (para calcular direcciones dependientes). |
| `align_up(u64 v, u64 a)` | Redondeo de `v` al siguiente multiplo de `a`. |
| `debug_build()` | Devuelve `true` cuando se pasa `--link-debug`; permite ramas de layout distintas para build normal vs depuracion. |

Las opciones de linea de comandos `--link-base` y `--entry` tienen **prioridad**
sobre lo que fije el script: si las pasas, ganan.

`--link-debug` es lo que hace que `debug_build()` devuelva `true`, de modo que el
mismo script sirve para produccion y para una imagen de depuracion cargada en
otra base:

```vx
void link() {
    u64 load = 0x100000;
    if (debug_build()) { load = 0x200000; }   // logica Vesta real
    base(load);
    entry("_kstart");
    stack_size(align_up(64 * 1024, 4096));
}
```

Para colocar una seccion en una direccion virtual exacta -- por ejemplo la
cabecera de arranque de un kernel -- combinas `base`, `entry` y `place_section`:

```vx
void link() {
    base(0x400000);
    entry("_kstart");
    place_section(".boot", 0x410000);   // .boot en una VA fija
}
```

El emisor rellena el fichero hasta `addr - base` para que la seccion caiga
exactamente en `addr`. Las VAs deben ir en orden ascendente; colocar una seccion
por debajo de la posicion actual es un error claro.

---

## Ejemplo integrador: boot sector BIOS completo

Un boot sector MBR de 512 bytes escrito integramente como datos crudos: codigo
de 16 bits (aqui como bytes literales), relleno con ceros hasta el offset 510 y
firma `0x55AA` en 510..511.

```vx
// Boot sector MBR de 512 bytes: codigo + relleno + firma, todo datos crudos.
@section(".boot", "rx")
bytes boot {
    db 0xFA                     // cli
    db 0xF4                     // hlt
    db 0xEB, 0xFE              // jmp $ (bucle infinito)

    times 510 - ($ - $$) db 0  // relleno con ceros hasta el offset 510
    db 0x55, 0xAA              // firma MBR en 510..511
}
```

```sh
vm -m aot --vx 15_boot_sector.vx --emit bin --bin-base 0x7C00 -o boot.bin
qemu-system-i386 -drive format=raw,file=boot.bin
```

La misma imagen, pero con el codigo real ensamblado (no bytes a mano), va en un
bloque `@bits(16) asm` como en el ejemplo `17_boot16.vx` de la seccion de
bloques `asm`: el bloque mezcla las instrucciones (`cli`, `mov`, `int 0x10`, el
bucle `hang`) con el relleno `times 510-($-$$) db 0` y la firma `dw 0xAA55` en un
unico bloque de 512 bytes.

## Ejemplo integrador: enlazar un kernel con base fija

Un kernel se compila a un objeto y se enlaza fijando su base de carga y su punto
de entrada mediante un script de enlace en Vesta. El kernel tiene su propia
entrada (`_kstart`), sin el `_start` sintetico que llamaria a `main`:

```sh
# Compilar el kernel a un objeto relocatable.
vm --vx kernel.vx -m aot --emit obj --format elf -o kernel.o

# Enlazar con el layout descrito en Vesta.
vm --link kernel.o -o kernel.elf --format elf --link-script link_layout.vx
```

Con `link_layout.vx` fijando `base(0x100000)`, `entry("_kstart")` y el tamano de
pila, obtienes un ejecutable cargado en 1 MB con tu propio punto de entrada, sin
un solo byte de layout descrito fuera de Vesta.

Para un dev-OS completo (bootloader propio real -> protegido -> largo, kernel
modular, drivers, programas cargados de disco) que ejercita `@section`, `@at`,
bloques `@bits` de 16/32/64 bits, tablas de punteros a funcion y binario plano a
distintas bases, mira el mini sistema operativo en
`examples_codes_vx/aot/os/`.

---

## Resumen

- **Describe el layout en el lenguaje**, no en un script externo.
- **`@section("nombre"[, "perms"])`**: coloca funciones o datos en una seccion
  propia (todos los formatos de salida).
- **`@at(N)`** / **`@order(N)`**: posicion absoluta y orden relativo de secciones
  (solo en binario plano `.bin`).
- **`bytes name { db/dw/dd/dq/times ... }`**: datos crudos estilo NASM, con
  `$`/`$$` para relleno y operandos que pueden ser literales, constantes
  `comptime` o simbolos de funcion (`dq main`).
- **`@bits(16|32|64) asm name { ... }`**: codigo por modo, con codigo + datos +
  relleno + firma en un mismo bloque; cambios de modo entre bloques por simbolo.
- **`--emit bin --bin-base 0xADDR`**: binario plano sin cabecera para
  bootloaders, ROMs y firmware.
- **Script de enlace en Vesta** (`fn link()` + `--link-script`): `base`,
  `entry`, `stack_size`, `place_section`, `section_bytes`, `align_up`,
  `debug_build` para controlar base, entrada, pila y VAs de seccion con logica
  Vesta completa.

Para el detalle de las flags del enlazador (`vm --link`, `--emit`, `--format`,
`--entry`, `--link-base`, `--link-debug`) consulta [Enlazador.md](Enlazador.md).
