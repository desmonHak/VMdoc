# Sistema de Cadenas de VestaVM (StringObject)

Este documento describe la arquitectura interna del sistema de cadenas de VestaVM:
como se almacenan los strings en memoria, como se gestionan con el GC, como
funciona la compactacion de codificacion, los ropes, los slices, el hashing y el
pool de interning.

Para la referencia de instrucciones (STRMAKE, STRLEN, etc.) ver:
`doc/VMdoc/SetInstruccionesVM/STRINGS.md`

---

## Nivel basico: que es un string en VestaVM

Un string en VestaVM es un objeto inmutable que almacena una secuencia de texto.
"Inmutable" significa que una vez creado, su contenido no cambia.  Si quieres
concatenar dos strings, no modificas ninguno de los dos: obtienes un nuevo objeto
que representa la concatenacion.

Los strings son gestionados por el recolector de basura (GC) de la VM.  El
programador no necesita reservar ni liberar memoria manualmente: basta con dejar
de usar el handle y el GC limpiara el objeto cuando sea necesario.

**GcHandle:** cada string se identifica por un numero entero de 32 bits llamado
GcHandle.  Es como el numero de un casillero: el programador guarda el numero en
un registro y la VM sabe donde encontrar el objeto real en memoria.

---

## Tres modos de almacenamiento

VestaVM usa tres representaciones internas distintas para un string.  Todas
comparten la misma interfaz de instrucciones; el programador no necesita saber
cual es la representacion activa en cada momento (salvo cuando usa `strgetkind`).

### FLAT (kind=0): el string normal

El modo FLAT es la representacion canonica.  Todos los bytes del texto estan
en un bloque de memoria contiguo, justo despues de la cabecera del objeto.

```
[ObjectHeader 24 bytes][encoding 1][kind 1][_pad 2][length 4][byte_len 4][hash 4]
[byte_0][byte_1]...[byte_N][0x00]
  ^-- offset 40
```

El byte extra al final (0x00) siempre esta presente aunque byte_len sea 0.
Esto garantiza que el puntero devuelto por `strraw` sea un C-string valido
para las funciones Win32 *A y *W.

Los strings FLAT cortos (hasta 255 bytes) actuan como "SSO implicito" (Small
String Optimization): todo el objeto, cabecera y datos, cabe en un solo bloque
GC sin punteros adicionales.

### ROPE (kind=1): concatenacion perezosa

Cuando concatenas dos strings con `strcat`, la VM no copia ninguno de los dos.
En cambio, crea un nodo ROPE: un objeto que dice "este string es A seguido de B",
guardando solo los GcHandles de A y B.

```
[ObjectHeader 24 bytes][encoding][kind=1][_pad 2][length 4][byte_len 4][hash 4]
[left_handle 4][right_handle 4][depth 4][_pad 4]
  ^-- RopeData en offset 40
```

El campo `depth` mide la profundidad del arbol.  Cuando la profundidad supera
STR_ROPE_MAX_DEPTH (48), la proxima operacion `strcat` materializa el rope a
FLAT en lugar de crear otro nivel.

Para obtener los bytes del rope (por ejemplo para pasarlo a una funcion nativa)
se usa `strflat` o `strraw`, que recorren el arbol recursivamente y copian todos
los bytes a un nuevo FLAT.

**Por que es util:** si un compilador genera codigo que construye un string
concatenando 100 fragmentos, con ROPE solo se crean 100 nodos de 40+16 bytes
cada uno.  Sin ROPE, cada concatenacion copiaria el string acumulado entero, con
un coste cuadratico O(n^2) en total.

### SLICE (kind=2): vista sin copia

`strslice` crea un objeto que apunta a una parte de otro string FLAT existente,
sin copiar ningun byte.

```
[ObjectHeader 24 bytes][encoding][kind=2][_pad 2][length 4][byte_len 4][hash 4]
[parent_handle 4][byte_offset 4]
  ^-- SliceData en offset 40
```

El `parent_handle` es el GcHandle del string FLAT padre.  `byte_offset` es el
numero de bytes desde el inicio del buffer del padre hasta donde comienza el slice.

Si el string fuente es un ROPE, `strslice` lo materializa primero (llama a
`flatten_string` internamente) y luego crea el slice sobre el FLAT resultante.

**Por que es util:** tokenizar, parsear, extraer subcadenas de un buffer grande.
El slice no tiene coste de memoria por los datos (solo 24+16 bytes de cabecera)
y no tiene coste de tiempo de copia.  Solo paga cuando se materializa (strflat)
o cuando se necesita el puntero host (strraw).

---

## Compact Strings: reduccion automatica de codificacion

Cuando se crea un string con `strmake`, la VM examina el contenido del buffer
y aplica la codificacion mas compacta posible.  Esto se llama "Compact Strings"
porque es el mismo mecanismo que usa la JVM de HotSpot desde Java 9.

Las reglas son:

```
Entrada UTF-8:
  todos los bytes <= 0x7F  -->  ASCII  (mitad de metadata, mismo puntero Win32)

Entrada UTF-16:
  todos los code units <= 0x7F  -->  ASCII  (1 byte/char en lugar de 2)
  todos los code units <= 0xFF  -->  ANSI   (1 byte/char, compatible Latin-1)
```

El resultado es que "Hello" (5 bytes UTF-8, todos ASCII) se almacena como
StringEncoding::ASCII = 0, y su buffer es identico byte a byte al texto original.
Una funcion Win32 que reciba el puntero via `strraw` puede usarlo directamente
con las funciones *A (MessageBoxA, WriteFile, etc.) sin ninguna conversion.

Si el texto contiene caracteres Unicode que no caben en ASCII (por ejemplo, "Hola
mundo con tilde"), la codificacion se mantiene como UTF-8 o UTF-16 segun la
entrada.

---

## Tabla de codificaciones

| Valor | Nombre  | Bytes/char | Rango                 | Win32 API  |
| :---: | :-----: | :--------: | :-------------------- | :--------: |
|   0   | ASCII   | 1          | U+0000..U+007F        | *A         |
|   1   | ANSI    | 1          | U+0000..U+00FF        | *A         |
|   2   | UTF8    | 1-4        | U+0000..U+10FFFF      | MultiByteToWideChar |
|   3   | UTF16   | 2 o 4      | U+0000..U+10FFFF      | *W         |
|   4   | UTF32   | 4          | U+0000..U+10FFFF      | (manual)   |

`strconv` convierte entre codificaciones.  Los casos implementados son
UTF-8 <-> UTF-16LE con soporte completo de surrogates.

---

## Hash FNV-1a: calculo perezoso y cache

El campo `str_hash` del StringObject almacena el hash del buffer de bytes.
El valor 0 significa "todavia no calculado".

El hash se calcula usando el algoritmo FNV-1a (Fowler-Noll-Vo):

```
hash = FNV_OFFSET_BASIS   (= 0xcbf29ce484222325 para FNV-1a de 64 bits)
para cada byte b en el buffer:
    hash = hash XOR b
    hash = hash * FNV_PRIME  (= 0x100000001b3)
```

VestaVM usa una variante de 32 bits almacenada en el campo `str_hash`.
El calculo se hace automaticamente en:
- `strmake`: si byte_len <= STR_INTERN_THRESHOLD (64 bytes)
- `strhash`: explicitamente, con cache posterior
- `strflat`: para el nuevo FLAT materializado

Una vez calculado, el hash se reutiliza sin recalcular.  Las comparaciones de
strings iguales en el pool de interning aprovechan este cache.

---

## Pool de interning por proceso

Cada `ProcessVM` tiene un puntero `str_intern_pool` a un `StringInternPool`,
creado de forma lazy la primera vez que se usa `strintern` o `strmake` con un
string corto.

```cpp
// include/runtime/string_intern.h
using StringInternPool = std::unordered_map<std::string, gc::GcHandle>;
```

La clave del mapa es el contenido bruto del string (bytes) concatenado con un
byte que representa la codificacion.  Esto garantiza que dos strings con el mismo
texto pero distinta codificacion se traten como distintos.

Cuando `strintern r_dst, r_src` se ejecuta:
1. Se materializa r_src a FLAT si es ROPE o SLICE.
2. Se construye la clave: raw_bytes + encoding_byte.
3. Se busca en el mapa del proceso.
4. Si existe: r_dst = el GcHandle almacenado (el canonico).
5. Si no existe: se inserta r_src como canonico y r_dst = r_src.

Tras el interning, dos variables que apuntan al mismo string logico tienen el
mismo GcHandle numerico.  Esto permite comparar strings con igualdad de puntero
en O(1) usando `cmpu` + `setcc`, sin leer los bytes.

`strmake` llama a `auto_intern` automaticamente para strings de <= 64 bytes.
Para strings largos se debe llamar a `strintern` explicitamente si se desea el
comportamiento de interning.

---

## Integracion con el GC

Los StringObjects son objetos GC normales (flag OBJ_FLAG_GC_OWNED en la cabecera).
El GC de VestaVM es generacional con una Nursery y un Old Space.

**Raices GC de los strings:**
- Los GcHandles en registros del proceso son raices directas.
- Los GcHandles en el intern pool son raices permanentes (mientras el proceso
  viva, los strings internados nunca se recolectan).
- Los hijos de un nodo ROPE son raices mantenidas por el nodo ROPE mismo.
- El padre de un SLICE es una raiz mantenida por el SLICE.

**Estabilidad de punteros:** los GcHandles son indices estables a traves de la
HandleTable del GC.  Aunque el GC mueva objetos (evacuacion de la Nursery),
el handle permanece valido y apunta al nuevo destino automaticamente.

**strraw y GC:** el puntero host devuelto por `strraw` apunta directamente al
buffer del StringObject en el heap GC.  Si el GC mueve el objeto en la siguiente
coleccion, el puntero se invalida.  Por eso no debe guardarse en memoria VM ni
en registros que persistan a traves de puntos de safepoint (yield, spawn, calln
que puede provocar GC).

---

## Resumen de costes de cada operacion

| Operacion    | Coste de tiempo   | Coste de memoria                        |
| :----------: | :---------------: | :-------------------------------------- |
| strmake      | O(n)              | 40 + n + 1 bytes (un bloque GC)         |
| strlen/kind  | O(1)              | Sin aloc                                |
| strcat       | O(1) normal       | 40 + 16 bytes (nodo ROPE)               |
| strcat deep  | O(n) si depth>48  | 40 + n + 1 bytes (materializa)          |
| strflat      | O(n) si ROPE/SLICE| 40 + n + 1 bytes (nuevo FLAT)           |
| strflat FLAT | O(1)              | Sin aloc (identidad)                    |
| strslice     | O(1) si padre FLAT| 40 + 8 bytes (nodo SLICE)               |
| strslice ROPE| O(n)              | Materializa primero + 40+8 slice        |
| strcmp       | O(n) worst case   | Puede materializar temporalmente        |
| strhash      | O(n) primera vez  | Sin aloc (escribe en str_hash del obj)  |
| strhash      | O(1) cacheado     | Sin aloc                                |
| strintern    | O(n) primera vez  | Sin aloc extra si ya existe en el pool  |
| strraw       | O(1) si FLAT      | Sin aloc                                |
| strraw ROPE  | O(n)              | Materializa: 40 + n + 1 bytes           |

---

## Diagrama de relaciones entre nodos

```
Ejemplo: strcat(strcat("Hello", " "), "World")

ROPE [depth=2, len=11]
 +-- left:  ROPE [depth=1, len=6]
 |    +-- left:  FLAT "Hello" [len=5]
 |    +-- right: FLAT " "     [len=1]
 +-- right: FLAT "World" [len=5]

Despues de strflat:
FLAT "Hello World" [len=11]
  (el arbol ROPE puede ser recolectado por el GC si no hay mas referencias)
```

```
Ejemplo: strslice(flat_hello_world, range=(6,5))

FLAT "Hello World" [parent]
      ^           ^
      0           10

SLICE [parent=flat_hello_world, byte_offset=6, len=5]
  -> apunta a los bytes 6..10 de "Hello World"
  -> leer "World" sin copiar ningun byte
```

---

## Archivos de implementacion

| Archivo                                        | Contenido                                  |
| :--------------------------------------------- | :----------------------------------------- |
| `include/loader/string_object.h`               | Struct StringObject, enums, helpers inline |
| `include/runtime/string_intern.h`              | Typedef StringInternPool                   |
| `src/runtime/exec_instruction_string.cpp`      | Implementacion de las 15 instrucciones     |
| `src/runtime/decode_table.cpp` (0x46-0x54)    | Entradas en la tabla de decode extendida   |
| `src/runtime/decode_instruction.cpp`           | decode_instr_raw_bytes (Convention B)      |
| `include/emmit/emmit_decl.h`                   | Declaraciones emit_str_two_reg, etc.       |
| `src/emmit/emmit_decl.cpp`                     | Implementaciones emit_str_two_reg, etc.    |
| `include/emmit/parser_to_bytecode.h`           | InstrTable: mapeo mnemonic -> emit fn      |
| `src/parser/parser.cpp`                        | InstrSet: registro de mnemonicos           |
