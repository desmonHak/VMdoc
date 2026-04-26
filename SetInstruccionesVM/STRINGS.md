# Instrucciones de Strings (Cadenas de Texto)

Una **cadena de texto** es una secuencia de caracteres: la palabra "Hola", el mensaje
"Error al abrir archivo", o cualquier fragmento de texto que necesite un programa.
En VestaVM las cadenas se gestionan como libros en una biblioteca: el bibliotecario
(el GC) lleva el inventario, y el programador trabaja con el numero de ficha del
libro (el `GcHandle`) sin preocuparse de en que estante fisico esta guardado ni de
devolver el libro cuando termina.  El GC libera automaticamente los objetos cadena
cuando nadie los referencia.

Las cadenas de VestaVM son **inmutables por contrato**.  Una vez creada una cadena
no se modifica su contenido; las operaciones como concatenacion o slice producen
nuevos objetos cadena en lugar de alterar el original.  La unica excepcion
controlada es el ciclo `strreserve` + escritura directa + `strfinalize`, que
permite construir cadenas de forma eficiente caracter a caracter antes de
congelarlas.

---

## Tabla de instrucciones

Todas las instrucciones de cadena son instrucciones extendidas (prefijo `0x00`) e
implementadas en `src/runtime/exec_instruction_string.cpp`.

| Instruccion    | opcode0 | opcode1 | Arity | Descripcion breve                                          |
| :------------: | :-----: | :-----: | :---: | :--------------------------------------------------------- |
| `strmake`      |  0x00   |  0x46   | THREE | Crear StringObject desde un buffer en memoria VM           |
| `strlen`       |  0x00   |  0x47   | TWO   | r_dst = numero de code-points de la cadena                 |
| `strcat`       |  0x00   |  0x48   | THREE | r_dst = nuevo nodo ROPE perezoso (A ++ B); O(1)            |
| `strcmp`       |  0x00   |  0x49   | THREE | r_dst = -1 / 0 / 1; comparacion lexicografica              |
| `strconv`      |  0x00   |  0x4A   | THREE | r_dst = nueva cadena con codificacion destino              |
| `strraw`       |  0x00   |  0x4B   | TWO   | r_dst = puntero host al buffer de bytes (para FFI Win32)   |
| `strslice`     |  0x00   |  0x4C   | THREE | r_dst = vista SLICE sin copia; r_range=(cp_start<<32)|cp_len |
| `strflat`      |  0x00   |  0x4D   | TWO   | Materializar ROPE o SLICE a FLAT; identidad si ya es FLAT  |
| `strhash`      |  0x00   |  0x4E   | TWO   | r_dst = hash FNV-1a (calculado y cacheado en el objeto)    |
| `strintern`    |  0x00   |  0x4F   | TWO   | r_dst = handle canonico del intern pool                    |
| `strgetenc`    |  0x00   |  0x50   | TWO   | r_dst = byte de codificacion (0=ASCII..4=UTF32)            |
| `strgetbytes`  |  0x00   |  0x51   | TWO   | r_dst = byte_len (numero de bytes del buffer)              |
| `strgetkind`   |  0x00   |  0x52   | TWO   | r_dst = kind (0=FLAT 1=ROPE 2=SLICE)                       |
| `strreserve`   |  0x00   |  0x53   | TWO   | Crear FLAT mutable con capacidad r_src bytes, byte_len=0   |
| `strfinalize`  |  0x00   |  0x54   | TWO   | Fijar byte_len, length y hash tras escritura directa       |

---

## Los tres modos de almacenamiento: FLAT, ROPE y SLICE

Una cadena de VestaVM puede existir en tres formas internas.  El campo `kind` del
objeto indica cual es la forma actual.  El programador no suele necesitar preocuparse
por esto: las instrucciones de cadena manejan la conversion automaticamente cuando
es necesario.

### FLAT: la cadena clasica

Analogia: un libro con todas sus paginas impresas y encuadernadas de forma
consecutiva.  Los bytes estan en un bloque contiguo de memoria inmediatamente
despues del encabezado del objeto.  Es la forma mas eficiente para lectura y para
pasar punteros a funciones nativas (FFI).

Caracteristicas:
- Un solo bloque de memoria: cabecera + datos.
- Siempre termina con un byte nulo extra para compatibilidad con Win32.
- Las cadenas cortas (byte_len <= 64 bytes) se crean como FLAT automaticamente.
- `strmake` siempre crea una cadena FLAT.

### ROPE: concatenacion perezosa en O(1)

Analogia: en lugar de copiar dos libros en uno nuevo, se crea una "tabla de
contenidos" que dice "primero lee el libro A, luego lee el libro B".  No se
copian datos hasta que alguien necesita leer la cadena entera como un bloque
continuo.

Caracteristicas:
- Creado por `strcat`.  El nodo ROPE contiene solo dos handles (izquierdo y
  derecho) y un campo de profundidad del arbol.
- La concatenacion es O(1) independientemente del tamano de las cadenas.
- Para obtener un puntero de bytes usable (por ejemplo para FFI) hay que
  llamar `strflat` primero, lo que materializa todos los nodos del arbol.
- Limite de profundidad: `STR_ROPE_MAX_DEPTH = 48`.  Si se supera ese limite,
  `strcat` materializa el arbol inmediatamente en lugar de crear otro nodo ROPE.

### SLICE: substring sin copia en O(1)

Analogia: un separador de paginas que dice "lee solo las paginas 6 a 10 del libro
A".  No se copian bytes: el SLICE referencia al padre y registra el desplazamiento
y la longitud en code-points.

Caracteristicas:
- Creado por `strslice`.
- El campo `data[]` del SLICE contiene `{ parent_handle, byte_offset }`.
- Si el padre es un ROPE, `strslice` lo materializa primero (O(n)).
- `strraw` sobre un SLICE materializa el padre y devuelve el puntero correcto
  dentro del buffer del padre.

---

## Layout de StringObject en memoria VM

```
offset   0.. 7  class_ptr    (ObjectHeader: 8 bytes) -- puntero a ClassInfo
offset   8..11  flags        (ObjectHeader: 4 bytes) -- OBJ_FLAG_GC_OWNED, etc.
offset  12..15  hash_code    (ObjectHeader: 4 bytes) -- identidad del objeto
offset  16..19  owner_pid    (ObjectHeader: 4 bytes) -- propietario del monitor (0=libre)
offset  20..21  lock_depth   (ObjectHeader: 2 bytes) -- contador reentrante del monitor
offset  22..23  _mon_pad     (ObjectHeader: 2 bytes) -- relleno de alineacion
--- campo propio de StringObject ---
offset  24      encoding     (uint8_t)  -- codificacion: ASCII=0 ANSI=1 UTF8=2 UTF16=3 UTF32=4
offset  25      kind         (uint8_t)  -- bits[1:0]=StringKind, bit7=is_interned
offset  26..27  _pad[2]      (2 bytes)  -- alineacion
offset  28..31  length       (uint32_t) -- numero de code-points
offset  32..35  byte_len     (uint32_t) -- numero de bytes en data[]
offset  36..39  str_hash     (uint32_t) -- cache FNV-1a (0 = no calculado aun)
offset  40+     data[]       -- contenido segun kind:
                    FLAT:  bytes crudos + 1 byte nulo extra al final
                    ROPE:  RopeData  { left_handle(4), right_handle(4), depth(4), _pad(4) }
                    SLICE: SliceData { parent_handle(4), byte_offset(4) }
```

En C++:

```cpp
struct alignas(8) StringObject {
    ObjectHeader header;   // 24 bytes (ABI v2)
    uint8_t      encoding; // StringEncoding
    uint8_t      kind;     // bits[1:0] = StringKind; bit7 = is_interned
    uint8_t      _pad[2];
    uint32_t     length;   // code-point count
    uint32_t     byte_len; // byte count in data[]
    uint32_t     str_hash; // FNV-1a cache (0 = not computed)
    // data[] sigue a offset 40; FLAT tiene siempre 1 byte nulo extra
};
```

Valores de `StringEncoding`:

| Valor | Nombre  | Descripcion                               |
| :---: | :-----: | :---------------------------------------- |
|   0   | ASCII   | Solo caracteres US-ASCII (<= 0x7F)        |
|   1   | ANSI    | Un byte por caracter (<= 0xFF), Latin-1   |
|   2   | UTF8    | Codificacion variable multibyte           |
|   3   | UTF16   | Dos bytes por code-unit (little-endian)   |
|   4   | UTF32   | Cuatro bytes por code-point               |

`GcHandle` es un `uint32_t` con indice en la HandleTable del GC.
`GC_NULL_HANDLE = 0xFFFFFFFF` indica ausencia de objeto.

---

## Compactacion automatica de codificacion

Cuando se crea una cadena UTF-8 cuyos bytes son todos <= 0x7F, VestaVM la
almacena como ASCII en lugar de UTF-8.  Esto es similar a la optimizacion
"compact strings" de HotSpot JVM: se elige la codificacion mas estrecha posible
que pueda representar todos los caracteres sin perdida.

Reglas aplicadas en `strmake` y `strconv`:

- UTF-8 con todos los bytes <= 0x7F -> se almacena como ASCII.
- UTF-16 con todos los code-units <= 0x7F -> se almacena como ASCII.
- UTF-16 con todos los code-units <= 0xFF -> se almacena como ANSI.

Ventajas:
- Menor uso de memoria (ASCII usa un byte por caracter frente a dos de UTF-16).
- Comparaciones mas rapidas (no hay que descodificar secuencias multibyte).
- El puntero devuelto por `strraw` sobre una cadena ASCII es directamente
  compatible con funciones Win32 `*A` (MessageBoxA, WriteFile, etc.) sin
  conversion adicional.

---

## Cache de hash

El campo `str_hash` almacena el hash FNV-1a de la cadena.  El valor `0` significa
"no calculado todavia".

- `strhash`: calcula el hash si aun no esta cacheado, lo almacena en `str_hash`
  y lo devuelve en `r_dst`.  Las llamadas siguientes a `strhash` sobre la misma
  cadena son O(1) (solo leen el campo).
- `strmake`: precalcula el hash para cadenas con `byte_len <= 64` bytes.
- `strflat`: recalcula el hash sobre el buffer materializado de la nueva cadena FLAT.

El algoritmo es FNV-1a de 32 bits aplicado sobre el buffer de bytes crudo
(`data[]`), incluyendo todos los bytes segun la codificacion almacenada.

---

## Intern pool

La instruccion `strintern` busca la cadena en un pool de interning por proceso
(`StringInternPool`), implementado como un mapa cuya clave son los bytes crudos
mas el byte de codificacion.

- Si ya existe una cadena con el mismo contenido y codificacion en el pool,
  `strintern` devuelve el handle canonico existente.
- Si no existe, la registra y devuelve el handle de la cadena recibida.

Consecuencia: dos cadenas iguales internadas con `strintern` producen el **mismo
handle numerico**.  Eso permite comparar cadenas internadas con `cmpu` (comparacion
entera de handles) en lugar de `strcmp` (comparacion de bytes), lo que es O(1).

`strmake` interna automaticamente las cadenas con `byte_len <= 64` bytes, por lo
que las cadenas literales cortas del codigo fuente suelen estar ya internadas sin
necesidad de llamar `strintern` explicitamente.

---

## Semantica detallada de cada instruccion

### `strmake r_dst, r_src, r_len` - crear cadena desde buffer VM

```asm
strmake r12, r1, r2   ; r12 = GcHandle del nuevo StringObject
```

| Parametro | Descripcion                                                           |
| :-------: | :-------------------------------------------------------------------- |
| `r_dst`   | Registro destino: recibe el GcHandle del nuevo StringObject.          |
| `r_src`   | Direccion VM del buffer de bytes de entrada.                          |
| `r_len`   | Numero de bytes a copiar del buffer.                                  |

Algoritmo:
1. Lee `r_len` bytes desde la direccion VM `r_src`.
2. Aplica la heuristica de compactacion de codificacion (UTF-8 -> ASCII si procede).
3. Aloca un `StringObject` en el GC heap con `byte_len = r_len` (o menos si hubo
   compactacion) y copia los bytes al campo `data[]`.
4. Anade un byte nulo extra al final del buffer para compatibilidad Win32.
5. Precalcula `str_hash` si `byte_len <= 64`.
6. Si `byte_len <= 64`, interna la cadena automaticamente.
7. Almacena el `GcHandle` en `r_dst`.

### `strlen r_dst, r_src` - longitud en code-points

```asm
strlen r1, r12   ; r1 = numero de code-points de la cadena en r12
```

Devuelve el campo `length` del `StringObject`, que es el numero de code-points
(caracteres abstractos), no el numero de bytes.  Para UTF-8 puede diferir de
`byte_len`; para ASCII son iguales.

### `strcat r_dst, r_a, r_b` - concatenacion perezosa

```asm
strcat r6, r12, r5   ; r6 = ROPE("A" ++ "B")
```

Crea un nodo ROPE cuyo `left_handle` apunta a `r_a` y `right_handle` apunta a
`r_b`.  La profundidad del nodo es `max(depth_a, depth_b) + 1`.
Si la profundidad resultante supera `STR_ROPE_MAX_DEPTH = 48`, se materializa
inmediatamente a FLAT para evitar arboles excesivamente profundos.

La operacion es O(1) en el caso normal.

### `strcmp r_dst, r_a, r_b` - comparacion lexicografica

```asm
strcmp r1, r12, r11   ; r1 = -1 si A < B, 0 si A == B, 1 si A > B
```

Compara byte a byte el contenido de las dos cadenas.  Si tienen codificaciones
distintas, convierte internamente antes de comparar.  Actualiza ZF y SF del
registro de flags del proceso.

### `strconv r_dst, r_src, r_enc` - conversion de codificacion

```asm
strconv r9, r12, r3   ; r9 = nueva cadena con encoding = r3
```

`r_enc` es un entero con el valor de `StringEncoding` destino (0-4).
Aplica la misma heuristica de compactacion: si todos los bytes del resultado
caben en ASCII, se almacena como ASCII aunque se haya pedido UTF-8.

### `strraw r_dst, r_src` - puntero host al buffer

```asm
strraw r9, r8   ; r9 = puntero host (void*) al inicio de data[]
```

Devuelve el puntero de memoria del host (no una direccion VM) al buffer de datos
del `StringObject`.  Si la cadena es ROPE, la materializa primero.  Si es SLICE,
calcula el puntero correcto dentro del padre materializado.

**ADVERTENCIA:** el puntero queda invalidado en la proxima coleccion del GC.
No almacenar el puntero mas alla de la llamada FFI inmediata.

### `strslice r_dst, r_src, r_range` - substring sin copia

```asm
; r_range = (cp_start << 32) | cp_len
mov      r1, 0x0000000600000005   ; cp_start=6, cp_len=5
strslice r11, r13, r1             ; r11 = SLICE view
```

`r_range` empaqueta el inicio y la longitud en code-points en un solo registro
de 64 bits: los 32 bits superiores contienen `cp_start` y los 32 bits inferiores
contienen `cp_len`.

Si `r_src` es ROPE, se materializa primero a FLAT.

### `strflat r_dst, r_src` - materializar a FLAT

```asm
strflat r13, r6   ; r13 = FLAT con todos los bytes del ROPE r6
```

Si `r_src` ya es FLAT, `strflat` es una identidad: devuelve el mismo handle.
Si es ROPE, recorre el arbol y copia todos los bytes en un nuevo objeto FLAT.
Si es SLICE, extrae el rango indicado del padre en un nuevo FLAT.

### `strhash r_dst, r_src` - obtener hash

```asm
strhash r2, r12   ; r2 = hash FNV-1a de la cadena en r12
```

El hash es un `uint32_t`.  Si el campo `str_hash` del objeto es `0`, se calcula
sobre el buffer de bytes y se cachea antes de devolverlo.

### `strintern r_dst, r_src` - internar cadena

```asm
strintern r5, r4   ; r5 = handle canonico del intern pool
```

Si la cadena ya esta en el pool (mismo contenido y codificacion), devuelve el
handle del objeto existente.  En otro caso, registra `r_src` como canonico.

### `strgetenc r_dst, r_src` - leer codificacion

```asm
strgetenc r1, r12   ; r1 = 0 (ASCII), 2 (UTF8), etc.
```

### `strgetbytes r_dst, r_src` - leer byte_len

```asm
strgetbytes r1, r12   ; r1 = numero de bytes del buffer (sin el nulo extra)
```

### `strgetkind r_dst, r_src` - leer kind

```asm
strgetkind r1, r6   ; r1 = 0 (FLAT), 1 (ROPE), 2 (SLICE)
```

### `strreserve r_dst, r_src` - reservar buffer mutable

```asm
mov      r2, 128
strreserve r8, r2   ; r8 = FLAT con capacidad 128 bytes, byte_len=0
```

Crea un `StringObject` FLAT con un buffer de `r_src` bytes pero con `byte_len=0`
y `length=0`.  El contenido del buffer es indefinido.  El programador escribe
directamente en el buffer (obtenido con `strraw`) y luego llama `strfinalize`
para registrar la longitud real.

### `strfinalize r_dst, r_src` - fijar longitud tras escritura directa

```asm
strfinalize r8, r2   ; r2 = nuevo byte_len; actualiza length y str_hash
```

Fija `byte_len = r_src` en el `StringObject r_dst`, recalcula `length`
(recorriendo los bytes para contar code-points segun la codificacion) y
recalcula `str_hash`.  Debe llamarse exactamente una vez despues de terminar
la escritura directa en el buffer de un `strreserve`.

---

## Codificacion binaria (Convencion B)

Todas las instrucciones de cadena usan la **Convencion B**, que es opuesta a la
usada por las instrucciones ALU (0x05-0x40).

```
+--------+--------+--------+--------+
| 0x00   | opcode |  byte2 |  byte3 |
+--------+--------+--------+--------+
  byte0    byte1    byte2    byte3

byte2 = (r_dst << 4) | r_src
byte3 = (r3   << 4) | extra    (extra=0 para instrucciones de dos operandos)
```

Funcion de decodificacion: `decode_instr_raw_bytes` (almacena byte2 -> reg1,
byte3 -> reg2 sin interpretar como campo de modo/registro).

Extraccion en exec:
- `r_dst = (reg1 >> 4) & 0xF`
- `r_src = reg1 & 0xF`
- `r3   = (reg2 >> 4) & 0xF`

**IMPORTANTE:** Las instrucciones ALU (Convencion A) usan `decode_instr_two_op_reg`
donde byte2 es el campo de control (modo de direccionamiento) y byte3 contiene los
numeros de registro.  Nunca usar `decode_instr_two_op_reg` para instrucciones de
cadena ni viceversa; la mezcla de funciones de decodificacion produce decodificaciones
incorrectas silenciosas que son dificiles de depurar.

---

## Ejemplos completos en .vel

### Basico: crear, medir y comparar

```asm
; Suponer que "all.str_hello" contiene los bytes de "Hello" en memoria VM
mov     r1, @Absolute("all.str_hello")  ; direccion VM del buffer
mov     r2, 5                            ; longitud en bytes
strmake r12, r1, r2                     ; r12 = GcHandle de "Hello"

; Medir la longitud en code-points
strlen  r1, r12                          ; r1 = 5
mov     r15, 1
calln   @Method("stdlib/native/io/vesta_io:vio_print_uint")  ; imprime 5

; Crear una segunda cadena con el mismo contenido
mov     r1, @Absolute("all.str_hello")
mov     r2, 5
strmake r11, r1, r2                     ; r11 = otra instancia de "Hello"

; Comparar lexicograficamente
strcmp   r1, r12, r11                    ; r1 = 0 (iguales)
mov      r15, 1
calln    @Method("stdlib/native/io/vesta_io:vio_print_uint")  ; imprime 0

; Comprobar codificacion (ASCII automatica por compactacion)
strgetenc r1, r12                        ; r1 = 0 (ASCII)
mov       r15, 1
calln     @Method("stdlib/native/io/vesta_io:vio_print_uint")  ; imprime 0
```

### Concatenacion perezosa y materializacion

```asm
; Crear "Hello"
mov     r1, @Absolute("all.str_hello")
mov     r2, 5
strmake r12, r1, r2                     ; r12 = "Hello"

; Crear " World"
mov     r1, @Absolute("all.str_world")
mov     r2, 6
strmake r5, r1, r2                      ; r5 = " World"

; Concatenar en O(1): solo crea un nodo ROPE, no copia bytes
strcat  r6, r12, r5                     ; r6 = ROPE("Hello" ++ " World")

; Verificar que es un ROPE
strgetkind r1, r6                       ; r1 = 1 (ROPE)

; Materializar a FLAT cuando se necesite un bloque contiguo
strflat r13, r6                         ; r13 = FLAT "Hello World"

; Medir la cadena materializada
strlen  r1, r13                         ; r1 = 11
strgetkind r2, r13                      ; r2 = 0 (FLAT)
```

### Substring sin copia con strslice

```asm
; Extraer "World" de "Hello World" (inicio=6, longitud=5 en code-points)
; r_range = (cp_start << 32) | cp_len = (6 << 32) | 5 = 0x0000000600000005
mov      r1, 0x0000000600000005
strslice r11, r13, r1                   ; r11 = SLICE -> "World"

; Verificar
strgetkind r1, r11                      ; r1 = 2 (SLICE)
strlen     r2, r11                      ; r2 = 5
```

### Interning y comparacion por handle

```asm
; Crear dos instancias con el mismo contenido
mov     r1, @Absolute("all.str_hello")
mov     r2, 5
strmake   r4, r1, r2
strintern r5, r4                        ; r5 = handle canonico

strmake   r6, r1, r2
strintern r7, r6                        ; r7 = el mismo handle canonico

; Comparar handles como enteros (O(1), sin strcmp)
cmpu   r5, r7
setcc  r1, 4                            ; cond 4 = JE; r1 = 1 si iguales
mov    r15, 1
calln  @Method("stdlib/native/io/vesta_io:vio_print_uint")  ; imprime 1
```

### FFI Win32 con strraw

```asm
; Obtener puntero host nulo-terminado para API de sistema operativo
strflat r8, r6                          ; asegurar FLAT antes de strraw
strraw  r9, r8                          ; r9 = char* (ASCII/UTF-8) o WCHAR* (UTF-16LE)

; Pasar directamente a una funcion C como puts o MessageBoxA
mov     r1, r9
mov     r15, 1
calln   @Method("msvcrt.dll:puts")      ; muestra la cadena en consola
```

### Construccion incremental con strreserve y strfinalize

```asm
; Reservar un buffer de 32 bytes para construir una cadena manualmente
mov        r2, 32
strreserve r8, r2                       ; r8 = FLAT vacio con capacidad 32

; Obtener puntero host y escribir bytes directamente (ejemplo: codigo nativo FFI)
strraw  r9, r8                          ; r9 = char* al buffer interno

; ... el codigo nativo escribe bytes en r9 y devuelve la longitud real en r0 ...

; Fijar la longitud real (suponer que el codigo nativo escribio 7 bytes)
mov         r2, 7
strfinalize r8, r2                      ; actualiza byte_len, length y str_hash

strlen r1, r8                           ; r1 = longitud en code-points
```

---

## Advertencias y buenas practicas

### 1. Guardar GcHandles en registros que no se usan como argumentos

Las instrucciones `calln` utilizan `r1-r15` como argumentos y cuentan de
argumentos.  Un `GcHandle` almacenado en `r1` se sobreescribira si hay una
llamada nativa antes de usarlo.  Guardar los handles en `r8-r13` cuando sea
necesario preservarlos a traves de llamadas.

```asm
strmake r12, r1, r2    ; r12 esta fuera del rango de argumentos tipicos
strlen  r1,  r12       ; r1 = longitud
mov     r15, 1
calln   @Method("stdlib/native/io/vesta_io:vio_print_uint")
; r12 sigue valido despues del calln
strlen  r1, r12        ; correcto: r12 no fue modificado por calln
```

### 2. El puntero de strraw se invalida tras una coleccion del GC

El puntero host devuelto por `strraw` apunta directamente al interior del heap
del GC.  Si ocurre una coleccion (compactacion) despues de obtener el puntero,
el objeto puede haberse movido y el puntero ya no es valido.

No almacenar el resultado de `strraw` mas alla de la llamada FFI inmediata.
No llamar a `yield`, `spawn`, `calln` u otras instrucciones que puedan
desencadenar GC mientras se use un puntero de `strraw`.

### 3. strslice requiere padre FLAT

Si el padre de un `strslice` es un ROPE, la instruccion lo materializa primero
en O(n) antes de crear el SLICE.  Para extraer multiples substrings de un mismo
ROPE, es mas eficiente llamar `strflat` una vez y luego usar `strslice` varias
veces sobre el FLAT resultante.

### 4. Limite de profundidad del arbol ROPE

`STR_ROPE_MAX_DEPTH = 48`.  Al concatenar muchas cadenas en cadena (A ++ B ++ C
++ ... ++ N) se va incrementando la profundidad del arbol ROPE.  Cuando se
supera el limite, `strcat` materializa el arbol en ese punto, lo que tiene un
coste O(n).  Si se sabe de antemano que hay mas de 48 concatenaciones, usar
`strreserve` + escritura directa + `strfinalize` es mas eficiente.

### 5. strfinalize debe llamarse exactamente una vez por strreserve

Llamar a `strfinalize` mas de una vez sobre el mismo objeto o no llamarlo en
absoluto deja el `StringObject` en un estado inconsistente (longitud incorrecta
o hash no actualizado).  Tratar el ciclo `strreserve` / escritura / `strfinalize`
como una transaccion atomica.

---

## Interaccion con el GC y el sistema de tipos

Los `StringObject` son objetos GC de primera clase.  Su `ObjectHeader` tiene el
flag `OBJ_FLAG_GC_OWNED` activado.  El GC escanea los handles contenidos en nodos
ROPE (`left_handle`, `right_handle`) y en nodos SLICE (`parent_handle`) como raices
adicionales durante la fase de marcado, garantizando que el padre no sea recolectado
mientras haya un SLICE o ROPE que lo referencie.

Las cadenas internadas en el `StringInternPool` son raices permanentes mientras
el pool exista; no se recolectan aunque no haya otras referencias.

Ver tambien: [[NativeCall (CallN)]], [[WEAKREF]], [[GC]]
