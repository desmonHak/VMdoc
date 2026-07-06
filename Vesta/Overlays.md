# Overlays: vistas tipadas sobre memoria binaria

Un `@overlay struct` es una **vista tipada sobre un puntero ajeno**: no reserva
un buffer propio ni copia nada -- un valor overlay *es* el puntero (8 bytes).
Describes el formato una vez, con campos y offsets, y a partir de ahi accedes a
cada campo por su nombre; el overlay traduce ese nombre a la direccion correcta y
hace el load/store tipado. La misma vista **lee, escribe y toma direcciones**
(`&campo`), asi que sirve tanto para *parsear* un formato binario existente como
para *construir* uno nuevo sobre un buffer propio.

Los overlays existen para eliminar la aritmetica de punteros a mano cuando
trabajas con memoria estructurada que no controlas: la cabecera de un PE o un
ELF en disco, el PEB de tu propio proceso, un bloque de registros MMIO, una
instruccion x86-64, una cabecera de red. En C escribirias `*(u32*)(buf + 0x3C)` y
cuidarias tu mismo cada offset, cada stride, cada byte-swap y cada ancho de
puntero. Con un overlay declaras la estructura y el compilador resuelve el
"donde"; tu solo dices "quiero `dos.e_lfanew`". Los offsets constantes se
pliegan a `[base + const]` (coste cero) y los que dependen de datos se resuelven
en runtime. Todo funciona identico en interprete, JIT y compilacion nativa (AOT).

Documentos hermanos: [[TiposDatos]] (structs value-type, punteros, newtypes),
[[InlineAsm]] (el `asm {}` que obtiene el puntero raiz, p.ej. el PEB via
`gs:[0x60]`), [[OptionalResult]] (como envolver los accesos comprobados) y
[[FFI]] (leer memoria de librerias externas).

---

## Tabla-resumen de anotaciones y builtins

| Construccion | Significado |
| :----------- | :---------- |
| `@overlay struct N { ... }` | Declara una vista tipada; un valor `N` es un puntero de 8 bytes |
| `N v = N(ptr)` | Construye la vista sobre el puntero base `ptr` (no aloca) |
| `campo @offset(N)` / `campo @0xNN` | Offset CONSTANTE (se pliega a `[base + N]`) |
| `campo @offset(expr)` | Offset DINAMICO: expresion sobre campos hermanos + `sizeof<T>()` |
| `campo @offset { ...; return dir; }` | Resolver de BLOQUE: logica arbitraria que devuelve la DIRECCION del campo |
| `T name[count] @offset(pos) stride(s)` | Array de overlay: `name[i]` en `base + pos + i*stride` (count/pos/stride runtime) |
| `T name[count] @element { ...; return dir; }` | Resolver POR-ELEMENTO (stride variable / TLV); `index` en scope |
| `u8 campo : N @offset(...)` | Bitfield de N bits (empaquetado, con read-modify-write al escribir) |
| `campo @endian(expr)` | Orden de bytes: `expr` distinto de cero = big-endian |
| `parent<T>()` | Dentro de un resolver: la vista RAIZ tipada `T` de la cadena |
| `offsetof(v.campo)` | `u64`: offset resuelto del campo dentro de la vista |
| `in_bounds(v.campo, len)` | `bool`: `offsetof(v.campo) + sizeof(campo) <= len` |
| `extent(v)` | `u64`: span TOTAL en runtime que ocupa la instancia |
| `sizeof<T>()` | `u64`: huella ESTATICA (constante de compilacion) del overlay `T` |

En scope dentro de un resolver: `base` (el puntero de la vista), los campos
hermanos por su nombre, `this`/`self` (la propia vista), `index` (solo en
`@element`) y `parent<T>()`.

---

## Declaracion y construccion

Un overlay se distingue de un `struct` normal por el marcador `@overlay` delante.
Un `struct` normal es un value-type que *posees*: reserva un buffer, se
zero-inicializa, se copia, tiene destructor. Un `@overlay struct` es lo contrario:
una vista de 8 bytes sobre un puntero que ya existe. Cada campo declara su offset
con `@offset(N)` o el atajo `@0xNN`; los huecos entre campos son padding
implicito (no hace falta declarar toda la estructura, solo los campos que te
interesan).

```vx
// Una cabecera DOS de un PE (simplificada): 'MZ' al inicio y el offset al
// header PE en 0x3C.  Solo declaramos los campos que nos interesan.
@overlay struct DosHeader {
    u16 e_magic  @0x00;   // 'MZ' = 0x5A4D
    i32 e_lfanew @0x3C;   // offset (relativo al inicio) al PE header
}

i32 main() {
    u8* buf = (u8*) malloc(64);
    *(u16*)(buf + 0x00) = 0x5A4D;      // 'MZ'
    *(i32*)(buf + 0x3C) = 0x00000100;  // e_lfanew = 256

    // Overlay-eamos la cabecera sobre el buffer: cero aritmetica a mano.
    DosHeader dos = DosHeader(buf);

    println("MZ ok, e_lfanew=${dos.e_lfanew}");   // 256

    // Escritura tipada a traves de la vista: modifica el buffer real.
    dos.e_lfanew = 0x200;
    println("nuevo e_lfanew=${dos.e_lfanew}");     // 512

    // El cambio se ve en el buffer subyacente (la vista NO copia).
    i32 raw = *(i32*)(buf + 0x3C);
    println("en el buffer = ${raw}");              // 512

    free(buf);
    return 0;
}
```

Puntos clave:

- **Un valor overlay es un puntero.** `DosHeader(buf)` no aloca ni copia: liga la
  vista al valor del puntero `buf`. Pasarlo a una funcion pasa 8 bytes.
- **`v.campo` es un load/store tipado** en `base + offset`, ancho segun el tipo
  del campo (`u16` lee 2 bytes, `i32` lee 4, `u64` lee 8, etc.).
- **Lectura y escritura son simetricas.** Como todo pasa por el mismo camino de
  resolucion de direccion, `dos.e_lfanew = x` es un STORE en la direccion que el
  overlay resuelve; escribir es "crear" a traves de la vista.
- **`&v.campo` toma la direccion** del campo (util para pasarla a `print_cstr`,
  memcpy, o cualquier funcion que espere un puntero).

---

## Offsets constantes: `@offset(N)` y `@0xNN`

La forma mas simple. El offset es una constante y el acceso se pliega a
`[base + const]` en tiempo de compilacion, sin coste en runtime. `@0xNN` es azucar
de `@offset(0xNN)`; ambos son intercambiables.

```vx
@overlay struct Elf64_Phdr {
    u32 p_type   @0x00;
    u32 p_flags  @0x04;
    u64 p_offset @0x08;
    u64 p_vaddr  @0x10;
    u64 p_filesz @0x20;
    u64 p_memsz  @0x28;
    u64 p_align  @0x30;
}
```

---

## Offsets dinamicos: `@offset(expr)`

El offset de un campo puede ser una *expresion* que referencia campos hermanos de
la misma vista, mas `sizeof<T>()`. El resolver lee esos hermanos en el momento del
acceso, asi que encadenas cabeceras (DOS -> NT -> ...) sin escribir aritmetica de
punteros a mano. Los nombres desnudos en la expresion son campos hermanos; el
compilador exige que la expresion evalue a un entero.

```vx
// Un "peek" de un PE: la cabecera DOS lleva en 0x3C el offset al NT header
// (e_lfanew); la firma del NT header ('PE\0\0') esta en base + ese offset.
@overlay struct PePeek {
    u16 magic  @0x00;            // 'MZ' = 0x5A4D
    i32 nt_off @0x3C;            // e_lfanew: offset (relativo a base) al NT hdr
    u32 nt_sig @offset(nt_off);  // DINAMICO: firma NT en base + nt_off
}

i32 main() {
    u8* buf = (u8*) malloc(128);
    PePeek pe = PePeek(buf);

    // Orden topologico: primero el campo del que depende el offset dinamico.
    pe.magic  = (u16)0x5A4D;      // STORE [base + 0x00]
    pe.nt_off = 0x40;             // STORE [base + 0x3C]  (NT hdr tras lo fijo)
    pe.nt_sig = (u32)0x00004550;  // STORE [base + nt_off] = base+0x40 ('PE\0\0')

    // Releer: el offset dinamico se resuelve solo leyendo nt_off.
    println("magic=${pe.magic} nt_off=${pe.nt_off} nt_sig=${pe.nt_sig}");
    free(buf);
    return 0;
}
```

Al **crear** una estructura con offsets dinamicos, escribe primero el campo del
que depende el offset (aqui `nt_off` antes que `nt_sig`): el compilador conoce el
grafo de dependencias entre offsets y detecta ciclos (`A @offset(B); B
@offset(A)`) como error de compilacion.

---

## El resolver de bloque: `@offset { ...; return <direccion>; }`

Cuando la posicion de un campo requiere *logica* (no solo una expresion), se usa
un resolver de bloque. Es la primitiva general de los overlays: todo lo demas
(offset constante, offset expresion, arrays con stride) desazucara a esto.

El punto conceptual mas importante: **el bloque resuelve la DIRECCION del campo,
no un offset.** `return base + hdr_len` devuelve una direccion absoluta. En scope
tiene `base` (el puntero de la vista) y los campos hermanos por su nombre, y admite
**cualquier construccion del lenguaje**: variables locales, `if`/`else`, varios
`return`, bucles. Internamente el bloque se compila como una funcion interna
`__ovl_resolve_<Struct>_<campo>(this)` que reutiliza todo el control de flujo del
lenguaje; el optimizador la puede inlinar, y sirve igual para leer y para escribir.

### Union etiquetada (ternario)

```vx
// `kind` decide donde vive `payload`.
@overlay struct TaggedRec {
    u32 kind    @0x00;   // 0 = inline, !=0 = con desplazamiento extra
    u32 hdr_len @0x04;   // longitud de la cabecera (variable)
    u32 extra   @0x08;   // desplazamiento extra (solo si kind!=0)
    u32 payload @offset {
        u64 hdr = base + hdr_len;              // intermedio nombrado
        return kind == 0 ? hdr : hdr + extra;  // ternario -> direccion condicional
    };
}
```

### Logica completa (if/else + multiples return)

```vx
// Formato con 3 variantes segun `tag`, cada una con el dato en un sitio distinto.
@overlay struct MultiRec {
    u8  tag     @0x00;
    u32 hdr_len @0x04;
    u32 value @offset {
        if (tag == 1) {
            return base + 0x10;          // ranura corta
        }
        if (tag == 2) {
            u64 h = base + hdr_len;      // tras la cabecera variable
            return h;
        }
        return base + 0x30;              // por defecto: al final
    };
}
```

### Bucles en el resolver

Como el resolver es codigo normal, admite bucles. Patron real: un campo que vive
tras un bloque de N ranuras, donde N es otro campo (longitud dada por los datos).

```vx
// El campo `tail` vive justo tras la ultima de `count` ranuras de 8 bytes; su
// direccion no es un offset fijo, se halla recorriendo el array en un bucle.
@overlay struct SlotRec {
    u32 count @0x00;
    u32 tail @offset {
        u64 addr = base + 0x10;      // primera ranura
        u64 i = 0;
        while (i < count) {          // recorre las `count` ranuras
            addr = addr + 8;
            i = i + 1;
        }
        return addr;                 // la direccion del tail
    };
}
```

> Nota: dentro de un resolver, declara los locales con tipo explicito
> (`u64 hdr = ...`), no con inferencia (`let`), para que el tipo se resuelva
> correctamente en el contexto del resolver. Vesta no tiene `switch`; el control
> de flujo del resolver es `if`/`else` + bucles (ver [[ControlFlow]]).

---

## Arrays de overlay: `[count] @offset(pos) stride(s)`

Un campo de overlay puede ser un ARRAY de elementos espaciados por un *stride*
(no por `sizeof`). Es exactamente lo que necesitan las tablas de los formatos
binarios: la tabla de secciones de un PE, los program headers de un ELF.

```
T name[count] @offset(pos) stride(s);
```

- `pos` = offset (relativo a base) de la primera entrada (const o hermano).
- `stride` = bytes entre elementos (const o hermano -> **stride runtime**).
- `count` = numero de entradas (para bounds; puede ser un hermano -> **count runtime**).

`v.name[i]` resuelve a `base + pos + i*stride`, y sirve para leer y escribir. El
stride runtime (leido de un campo) es lo que distingue una tabla ELF de una
tabla C fija:

```vx
// Tabla estilo ELF: offset Y stride LEIDOS de campos (runtime).
@overlay struct ELFish {
    u32 tab_off       @0x00;   // offset de la tabla
    u32 ent_size      @0x04;   // stride RUNTIME (bytes por entrada)
    u32 n             @0x08;
    u32 entry[n] @offset(tab_off) stride(ent_size);
}
```

### Elementos que son otro overlay

El elemento de un array puede ser otro `@overlay struct`. Entonces `v.Tabla[i]` es
una vista tipada del elemento i (en `base + pos + i*stride`) y
`v.Tabla[i].campo` lee/escribe un campo de ese elemento. Es el patron para
parsear la tabla de secciones de un PE:

```vx
@overlay struct SectionHeader {
    u32 VirtualSize      @0x00;
    u32 VirtualAddress   @0x04;
    u32 SizeOfRawData    @0x08;
    u32 PointerToRawData @0x0C;
}

@overlay struct PeImage {
    u16 magic         @0x00;   // 'MZ'
    u32 num_sections  @0x04;   // count RUNTIME
    u32 sec_off       @0x08;   // offset a la tabla RUNTIME
    // Cada SectionHeader ocupa 40 bytes en un PE real.
    SectionHeader Sections[num_sections] @offset(sec_off) stride(40);
}

// ... recorrerla es un bucle normal:
//   while ((u32)i < pe.num_sections) {
//       u32 va = pe.Sections[i].VirtualAddress;
//       ...
//   }
```

### Arrays sin count

Si el formato no da un count (la tabla termina en una entrada nula, como los
descriptores de import de un PE), declara el array sin count y termina el bucle tu
mismo al leer el centinela:

```vx
ImportDesc Imports[] @offset { ... } stride(20);
// ... while (pe.Imports[i].name_rva != 0 && i < 64) { ... }
```

Fijate que la **base del array la puede dar un resolver de bloque** (aqui traduce
un RVA a offset de fichero recorriendo las secciones), no solo un offset fijo.

---

## Resolver por-elemento: `@element { ... }`

Los arrays con `stride(s)` tienen paso fijo. Para formatos donde cada elemento
mide algo distinto -- TLV (tag/length/value), tablas de simbolos con nombres
inline, registros de longitud variable -- se usa `@element { ... }`: un resolver
POR-ELEMENTO con `index` en scope que devuelve la DIRECCION del elemento i. Puede
leer el elemento ANTERIOR (`this.Recs[index-1]`) para encadenar; esa recursion se
resuelve como un CALL en runtime, guardado por el caso base `if (index == 0)`.

```vx
@overlay struct Rec {
    u8 tag @0x00;
    u8 len @0x01;   // longitud del payload (el record ocupa 2 + len bytes)
}

@overlay struct Doc {
    u32 count @0x00;
    // Cada record va justo tras el anterior; el primero en base+8.  El paso NO
    // es constante: depende del `len` del record previo.
    Rec Recs[count] @element {
        if (index == 0) { return base + 8; }
        u64 prev = (u64) this.Recs[index - 1];   // direccion del anterior
        u8  plen = this.Recs[index - 1].len;      // su longitud
        return prev + 2 + (u64)plen;              // stride VARIABLE
    };
}
```

`stride(s)` es azucar de `@element { return table_base + index * s; }`, y el
default (un array sin nada) es `@element { return table_base + index * sizeof(T); }`.
Con el resolver por-elemento no hay ninguna limitacion de formato.

---

## Bitfields

Un campo puede ocupar solo N bits dentro de un byte: `u8 campo : N @offset(...)`.
Varios bitfields al mismo offset se empaquetan de bit bajo a bit alto en el orden
en que se declaran. Al escribir un bitfield se hace read-modify-write del byte
subyacente, preservando los demas bits. Los bitfields soportan `@offset` dinamico
igual que los campos normales.

Ejemplo del byte `ModR/M` de x86-64 (`mod:2 reg:3 rm:3`) con offset dinamico
resuelto por un metodo del propio overlay:

```vx
@overlay struct X86 {
    u8 rm  : 3 @offset { return base + this.modrm_off_rel(); };
    u8 reg : 3 @offset { return base + this.modrm_off_rel(); };
    u8 mod : 2 @offset { return base + this.modrm_off_rel(); };
    // ... this.modrm_off_rel() calcula la posicion del ModR/M segun REX y opcode
}
```

Y el `BitField` del PEB, decodificado en flags individuales al mismo offset:

```vx
@overlay struct PEB {
    u8 BitField                  @0x003;
    u8 ImageUsesLargePages   : 1 @0x003;
    u8 IsProtectedProcess    : 1 @0x003;
    u8 IsImageDynamicallyReloc : 1 @0x003;
    u8 IsAppContainer        : 1 @0x003;
    // ...
}
```

> Los bitfields no admiten `@endian(...)` (solo campos enteros de 2, 4 u 8 bytes
> tienen orden de bytes).

---

## Endianness: `@endian(expr)`

El orden de bytes de un campo lo declara `@endian(expr)`: si `expr` es distinto de
cero, el campo es **big-endian**. En un host little-endian (x86-64) un campo
big-endian emite un byteswap en cada read/write, asi que tu codigo trabaja siempre
en orden nativo pero la memoria queda en el orden del formato. Es simetrico: la
misma vista parsea Y construye. Solo aplica a enteros de 2, 4 u 8 bytes.

### Endianness fija (cabeceras de red)

```vx
// Cabecera tipo TCP: todo big-endian ("network byte order").
@overlay struct NetHdr {
    u16 src_port @endian(true) @0x00;
    u16 dst_port @endian(true) @0x02;
    u32 seq      @endian(true) @0x04;
    u16 window   @endian(true) @0x0C;
    u16 local    @endian(false) @0x0E;   // un campo little-endian explicito
}

// h.dst_port = 80;  ->  en memoria (base+2..+3) quedan los bytes 00 50 (big-endian)
// println(h.dst_port);  ->  80 (orden nativo, releido por la misma vista)
```

### Endianness segun el contexto (formatos bi-endian, tipo ELF)

Muchos formatos declaran su propio orden: ELF lo dice en `e_ident[EI_DATA]`. Con
`@endian(expr)` el orden lo decide una expresion en el momento del acceso -- puede
leer un campo hermano o un `comptime` const. Si la expresion es runtime, el swap
es un select **sin ramas** (`v ^ ((v ^ bswap(v)) & -(big))`); si es comptime, se
pliega a coste cero.

```vx
// Mini-cabecera bi-endian: ei_data (1=LE, 2=BE) manda sobre el resto.
@overlay struct BiHdr {
    u32 magic   @0x00;
    u8  ei_data @0x04;                          // 1 = little, 2 = big
    u16 version @endian(ei_data == 2) @0x06;    // orden segun ei_data
    u32 entry   @endian(ei_data == 2) @0x08;    // idem
}
```

El VALOR logico releido es el mismo sea cual sea el orden; solo cambian los bytes
en memoria. Una sola forma (`@endian(expr)`) cubre el caso fijo y el dinamico.

---

## Metodos en overlays

Un `@overlay struct` puede tener metodos. Dentro de un metodo, `this` es el puntero
HOST de la vista (en los tres modos de ejecucion), y `(u64) this` es la direccion
base. Los metodos encapsulan la logica del formato una sola vez y son llamables
desde los resolvers via `this.metodo(...)` o `parent<T>().metodo(...)`.

El caso canonico: la traduccion RVA -> offset de fichero de un PE es un algoritmo
que recorre la tabla de secciones. En vez de repetirlo en cada resolver, se
escribe una vez como metodo:

```vx
@overlay struct PeImage {
    u32 num_sections  @0x04;
    Section Sections[num_sections] @offset(...) stride(40);

    // La abstraccion: traducir un RVA a direccion de fichero recorriendo las
    // secciones.  Un metodo del overlay; lo usan todos los resolvers.
    public u64 translate(u32 rva) {
        u64 rb = (u64) this;
        i32 i = 0;
        while ((u32)i < (u32)this.num_sections) {
            u32 va = this.Sections[i].virt_addr;
            u32 vs = this.Sections[i].virt_size;
            u32 rs = this.Sections[i].raw_size;
            if (vs < rs) { vs = rs; }
            if (rva >= va && rva < va + vs) {
                return rb + (u64)this.Sections[i].raw_ptr + (u64)(rva - va);
            }
            i = i + 1;
        }
        return rb + (u64)rva;   // fallback: dentro de las cabeceras
    }

    // El offset del directorio de imports = un campo-resolver que llama al metodo.
    u8 import_dir @offset { return this.translate(this.Dirs[1].rva); };
}
```

Un metodo `to_string()` en un overlay convierte a la vista en imprimible: cuando
haces `println(v)`, se invoca `to_string()` (ver [[Strings]] para el protocolo de
impresion). El desensamblador de x86-64 (mas abajo) lo usa para imprimir cada
instruccion decodificada.

> `asm` es palabra reservada; no la uses como nombre de metodo.

---

## `parent<T>()`: alcanzar la vista raiz

Un sub-overlay (por ejemplo un `ImportDesc` obtenido de `pe.Imports[i]`) a veces
necesita datos del contenedor: el nombre de una DLL vive en su propio RVA, y para
traducirlo hace falta la tabla de secciones del `PeImage` raiz. `parent<T>()`
devuelve, desde dentro de un resolver, la vista RAIZ tipada `T` de la cadena de
accesos.

Lo importante es el modelo de implementacion, el **root-threading**: `parent<T>()`
NO cambia la representacion del overlay (sigue siendo 8 bytes). El puntero raiz se
enhebra en el *call site*: cuando escribes `pe.Imports[i].thunks[k].fname`, el
compilador camina la cadena de accesos hasta la raiz (`pe`) y le pasa esa raiz al
resolver como un parametro extra. No se guarda ningun puntero-a-padre en runtime.

```vx
// Entrada de la ILT (import lookup table) de un PE.  El nombre de la funcion
// vive en un RVA; para traducirlo hay que alcanzar el PeImage RAIZ (a dos
// niveles: Thunk -> ImportDesc -> PeImage) via parent<PeImage>().
@overlay struct Thunk {
    u64 raw @0x00;
    u8 fname @offset {
        u32 rva = (u32)(this.raw & 0x7FFFFFFF) + 2;   // +2 salta el hint u16
        return parent<PeImage>().translate(rva);       // metodo del padre raiz
    };
}

@overlay struct ImportDesc {
    u32 name_rva @0x0C;   // RVA al nombre de la DLL
    // El nombre de la DLL, traducido por las secciones del PADRE raiz.
    u8 dll_name @offset {
        return parent<PeImage>().translate(this.name_rva);
    };
}
```

Regla de uso: dentro de un resolver, `parent<T>()` solo es valido si el overlay se
construyo como SUB-overlay (via `pe.Imports[i]`, donde el compilador conoce el
padre); un overlay construido suelto (`ImportDesc(ptr)`) no tiene padre y usar
`parent<T>()` es un error de compilacion.

---

## Builtins de introspeccion y seguridad

### `sizeof<T>()` -- huella estatica

`sizeof<T>()` de un overlay devuelve su **huella estatica**: `max(offset + size)`
sobre los campos de offset CONSTANTE, redondeada a 8. Es una constante de
compilacion (los campos de offset dinamico no cuentan, porque su posicion depende
de los datos). Sirve para reservar el buffer de respaldo exacto al crear una
estructura:

```vx
u64 fixed = sizeof<PePeek>();   // 64 (constante de compilacion)
u8* buf = (u8*) malloc(128);
```

La forma es `sizeof<T>()` (con angulos). `sizeof(overlay_var)` sobre una variable
overlay tambien esta disponible.

### `offsetof(v.campo)` -- offset resuelto

`offsetof(v.campo)` devuelve, como `u64`, el offset resuelto del campo dentro de la
vista -- incluyendo `@offset(expr)` dinamico y `arr[i]*stride`. Es azucar de
`(u64)(&v.campo) - (u64)v`, pero legible:

```vx
println("OFF_S1_SZ=${offsetof(h.secs[1].sz)}");   // 20
println("IMPORT_DIR_OFF=${offsetof(pe.import_dir)}");   // offset traducido por el resolver
```

### `in_bounds(v.campo, len)` -- comprobacion de rango

`in_bounds(v.campo, len)` devuelve `bool`, azucar de
`offsetof(v.campo) + sizeof(campo) <= len`. Comprueba que el campo cae completo
dentro de un buffer de `len` bytes:

```vx
if (!in_bounds(eh.Phdrs[i].p_align, size)) {
    // fuera de rango: politica del usuario (contar, avisar, saltar...)
}
```

### `extent(v)` -- span total en runtime

`extent(v)` devuelve el span REAL de una INSTANCIA: `max(fin de campo) - base`,
evaluando los counts dinamicos y los resolvers con los datos de `v`. Es el
complemento runtime de `sizeof<T>()` (estatico):

```vx
@overlay struct Img {
    u32 magic  @0x00;
    u16 count  @0x04;
    u32 tail   @offset(count + 0x08);      // campo de offset DINAMICO
    Sec secs[count] @offset(0x10) stride(8);
}
// ... con count=3:  secs 0x10 + 3*8 = 40  <- el mayor  ->  extent = 40
u64 ext = extent(im);                   // 40
println("fits256=${extent(im) <= 256}");   // true
```

`extent` cubre escalares (offset const / `@offset(expr)` / `@offset{}`) + arrays de
stride CON count. Para arrays sin count o `@element` (paso variable), mide el
ultimo elemento con `offsetof`.

---

## Overlays anidados

Un overlay puede contener campos que son otros overlays -- inline (la sub-vista
vive en `base + offset`, sin dereferencia) o via puntero seguido. Es como se
componen los parsers de PE/ELF: un `PeImage` contiene un array de `SectionHeader`,
cada `ImportDesc` contiene un array de `Thunk`. La composicion no requiere codigo
especial: `v.Tabla[i]` devuelve la sub-vista, y `.campo` es un acceso overlay
normal sobre ella.

Cuando el sub-overlay vive en otro puntero (no inline), lo construyes desde el
puntero leido -- la unica operacion "cruda" permitida es obtener ese puntero raiz:

```vx
// Caminar la lista de modulos del PEB: Ldr -> InLoadOrderModuleList
LdrData ld = LdrData((u8*) this.Ldr);        // sub-vista sobre el puntero Ldr
u64 head = this.Ldr + 0x10;
u64 cur  = ld.InLoadOrderFlink;
while (cur != head && cur != 0 && count < 64) {
    LdrEntry m = LdrEntry((u8*) cur);        // sub-vista sobre cada nodo
    // ... m.DllBase, m.NameLen, m.NameBuf ...
    cur = m.InLoadFlink;
    count = count + 1;
}
```

---

## Offsets dependientes de version

El offset de un campo puede cambiar entre versiones de una estructura. Con un
resolver de bloque que ramifica sobre un campo de contexto (leido de la propia
estructura, de un global runtime, o de un `comptime` const), el overlay
discrimina la version por dentro -- `main` no sabe nada.

```vx
@overlay struct PEB {
    u16 OSBuildNumber @0x120;   // el CONTEXTO de version, leido de si mismo
    // ...
    // SessionId: su OFFSET cambia entre versiones -> resolver que lee el propio
    // build del PEB para elegir el offset.
    u32 SessionId @offset {
        if (this.OSBuildNumber >= 10240) { return base + 0x2C0; }   // Win10/11
        return base + 0x2A0;                                         // 8.1 y antes
    };
}
```

El mismo mecanismo cubre campos que solo EXISTEN a partir de cierta version:
dentro de un metodo `dump()` del overlay, se lee `this.OSBuildNumber` y solo se
accede a los campos nuevos donde son validos, evitando leer basura en un Windows
antiguo. La discriminacion vive DENTRO del overlay.

Si el contexto es un `comptime` const (target fijo), el resolver se pliega en
compilacion a la rama que corresponde -- coste cero.

---

## Acceso correcto: componer las comprobaciones en el lenguaje

Un overlay lee memoria ajena; un campo cuyo offset (dinamico + stride) cae fuera
del buffer devolveria basura silenciosa. La filosofia de Vesta es **no imponer
maquinaria**: no hay 0-por-defecto (no distingue un 0 real), ni abort (no todo
entorno tiene stderr), ni excepciones obligatorias, ni flags implicitos. En su
lugar, el lenguaje expone primitivas legibles (`offsetof`, `in_bounds`,
`extent`, `sizeof<T>()`) para que **compongas tu propia validacion, donde
quieras, eligiendo el coste y decidiendo que devolver** (un `Optional`, un
`Result`, un centinela, un `panic`... ver [[OptionalResult]]).

```vx
// POLITICA DEL USUARIO (una entre muchas): devolver el tamano de la seccion i,
// o -1 si el indice o el campo se salen del buffer real de `len` bytes.  Otro
// programador podria devolver un Optional, un Result, o hacer panic.
i64 sec_size_checked(Hdr h, i32 i, u64 len) {
    if ((u32)i >= (u32)h.count) { return -1; }         // indice fuera de count
    if (!in_bounds(h.secs[i].sz, len)) { return -1; }  // campo fuera del buffer
    return (i64) h.secs[i].sz;                           // solo aqui, ya probado
}
```

Son azucar sobre `&`, resta y `sizeof`; no son keywords ni maquinaria oculta. Tu
decides el punto de comprobacion y la respuesta.

---

## Garantia: interprete = JIT = AOT

Todo lo descrito -- offsets constantes y dinamicos, resolvers de bloque y
por-elemento, arrays con stride runtime, bitfields, `@endian`, metodos,
`parent<T>()`, los builtins de introspeccion -- funciona identico en los tres modos
de ejecucion: el interprete, el JIT y la compilacion nativa (AOT). Un overlay es un
puntero host de 8 bytes en todos ellos; los resolvers se compilan a funciones
normales; los offsets constantes se pliegan; los swaps de `@endian` se pliegan o se
vuelven un select sin ramas. Esto hace los overlays idoneos para codigo de bajo
nivel sin runtime (ver [[CompilacionNativa]]): parsear estructuras del sistema
operativo o del kernel sin depender de librerias.

---

## Casos de uso

### Parseo de PE y ELF

Los parsers de PE (PE32+, tipo `dumpbin`) y ELF64 se escriben con SOLO `main` +
`@overlay struct`s: cero funciones auxiliares y cero aritmetica de punteros
(`*(T*)(buf+off)`). Todo el direccionamiento -- incluida la logica del formato,
como traducir un RVA a offset de fichero recorriendo la tabla de secciones -- vive
DENTRO de los overlays, como campos-resolver y metodos. `main` solo abre el
fichero, itera y lee campos. Se parsean cabeceras completas, los 16 data
directories de un PE, las tablas de secciones y program/section headers, y se
recorren los imports (descriptores + funciones de la ILT) resolviendo cada nombre
por su RVA via `parent<PeImage>()`.

### Dump del PEB de Windows

El PEB del propio proceso se obtiene con inline asm (`mov rax, gs:[0x60]` dentro de
una funcion; ver [[InlineAsm]]) y se disecciona entero con overlays: ~45 campos
x64, el `BitField` decodificado con bitfields, `SessionId` con offset
version-dependiente, y el walk de la lista de modulos cargados (`Ldr ->
PEB_LDR_DATA -> InLoadOrderModuleList`) con sub-overlays sobre punteros reales,
imprimiendo cada DLL con su nombre UTF-16, base y tamano.

### Ensamblador / desensamblador x86-64

Una instruccion x86-64 es un struct DINAMICO: sus componentes (prefijo REX, opcode
de 1 o 2 bytes, ModR/M, SIB, desplazamiento, inmediato) estan presentes o no y en
posiciones que dependen de los bytes previos. **Un solo overlay expresa toda la
instruccion**: cada campo declara su direccion con un resolver `@offset { return
base + <metodo>(); }` que calcula la posicion mirando los campos hermanos, y los
metodos (`rex_len`, `op_bytes`, `modrm_off_rel`, `disp_len`, `imm_len`, `len`)
encapsulan las reglas de codificacion una sola vez. La MISMA vista ENSAMBLA
(metodos `enc_*` que escriben los campos), DESENSAMBLA (`to_string`, que
`println(ins)` invoca solo) y CAMINA la secuencia (`len()` avanza al siguiente):

```vx
// ... construir:
X86 b = X86(code + pos);
b.enc_alu_rr(Op.MOV, Reg.RBP, Reg.RSP);   // 48 89 e5
pos += b.len();

// ... y desensamblar caminando: la misma vista decodifica y len() avanza:
u64 p = 0;
while (p < pos) {
    X86 ins = X86(code + p);
    println(ins);              // to_string() se invoca sola
    p += ins.len();
}
```

Los registros y opcodes se declaran como enums con valor (`enum Op : u8 { MOV =
0x89 }`), donde el valor de la variante ES el byte del encoding (ver
[[TiposDatos]]).

### Cabeceras de red big-endian y MMIO

`@endian(true)` da cabeceras de protocolo en network byte order construidas y
parseadas por la misma vista, sin byteswaps a mano. Y como un overlay es solo una
vista sobre una direccion, sirve igual sobre un bloque de registros MMIO: declaras
los registros con sus offsets y accedes por nombre, dejando que el compilador
emita el load/store con el ancho correcto.
