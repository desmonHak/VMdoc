# Reflexion OOP - Introspeccion de clases y objetos

Las instrucciones de reflexion permiten que un programa examine en **tiempo de ejecucion**
la clase de un objeto, su jerarquia, sus campos y sus metodos. Es la capacidad de un programa
de "mirarse a si mismo" para saber con que tipo de datos esta trabajando.

> **Ver tambien:** [[NEWOBJRAW y NEWOBJ]], [[CALLVIRT y CALLSUPER]], [[THROW y RETHROW]], [[Doc y Atributos OOP]]

---

## Para que sirve la reflexion

Imagina que tienes una funcion que recibe un objeto de tipo desconocido. Sin reflexion, no
puedes saber que clase es ni que campos tiene. Con reflexion puedes:

- Comprobar si un objeto es de una clase concreta (o subclase de ella).
- Obtener el nombre de la clase en tiempo de ejecucion.
- Iterar sobre todos los campos de un objeto y leerlos.
- Llamar a metodos por indice (despacho dinamico completo).

---

## Resumen rapido

| Instruccion                        | Opcode       | Entrada                          | R0 resultado               |
| :--------------------------------- | :----------: | :------------------------------- | :------------------------- |
| `getclass  reg_obj`                | `0x00 0xD5`  | host ptr a ObjectHeader          | host ptr a ClassInfo       |
| `instanceof reg_obj, reg_cls`      | `0x00 0xD6`  | objeto + ClassInfo               | 1 si es instancia, 0 si no |
| `checkcast  reg_obj, reg_cls`      | `0x00 0xD7`  | objeto + ClassInfo               | obj ptr o EVT_ERROR        |
| `getfield   reg_cls, idx`          | `0x00 0xD8`  | ClassInfo + indice (imm8)        | host ptr a FieldInfo       |
| `getmethod  reg_cls, idx`          | `0x00 0xD9`  | ClassInfo + indice (imm8)        | host ptr a MethodInfo      |
| `fieldcount   reg_cls`             | `0x00 0xDA`  | host ptr a ClassInfo             | numero de campos           |
| `methodcount  reg_cls`             | `0x00 0xDB`  | host ptr a ClassInfo             | numero de metodos          |
| `classname    reg_cls`             | `0x00 0xDC`  | host ptr a ClassInfo             | host ptr a char* nombre    |

En todos los casos: si el puntero de entrada es `null` (0), `R0 = 0`.

Para las instrucciones de documentacion y atributos (`classdoc`, `classattrcount`, etc.)
ver [[Doc y Atributos OOP]].

---

## Tipos de datos internos

### `stringx` (16 bytes)

Cadena de texto usada internamente por ClassInfo, MethodInfo y FieldInfo:

```
+0   data   (8B)  puntero host a uint8_t[] (el texto)
+8   size   (4B)  longitud en bytes (uint32_t)
+12  --     (4B)  padding de alineacion
```

### `AttrEntry` (32 bytes)

Par clave/valor para anotaciones. Ver [[Doc y Atributos OOP]].

```
+0   key.data    (8B)  puntero host al nombre de la anotacion
+8   key.size    (4B)  longitud del nombre
+12  --          (4B)  padding
+16  value.data  (8B)  puntero host al valor de la anotacion
+24  value.size  (4B)  longitud del valor
+28  --          (4B)  padding
```

---

## `GETCLASS` - obtener la clase de un objeto

```c
getclass  reg_obj    // R0 = ClassInfo* del objeto
```

Lee el campo `ObjectHeader.class_ptr` (offset +0 del objeto) y lo devuelve en R0.
Esto te dice exactamente de que clase es el objeto.

```c
// obj_ptr en r1 (host ptr al ObjectHeader del objeto)
getclass r1          // R0 = ClassInfo* (que clase es este objeto)

// R0 tiene la ClassInfo; se puede pasar a otras instrucciones de reflexion
mov      r2, r0
fieldcount r2        // R0 = cuantos campos tiene la clase
```

---

## `INSTANCEOF` - comprobar tipo en la jerarquia de herencia

```c
instanceof  reg_obj, reg_cls    // R0 = 1 si obj es instancia de cls (o subclase)
```

Esta instruccion responde a la pregunta: "?es este objeto de la clase X, o de alguna
subclase de X?"

Realiza una busqueda recursiva sobre `ClassInfo.supers[]` e `ClassInfo.interfaces[]`.
Devuelve `1` si:
- `obj->class_ptr == cls` (mismo tipo exacto), o
- algun super o interfaz de `obj->class_ptr` coincide con `cls` (herencia transitiva).

Devuelve `0` si no hay relacion, o si cualquier puntero es nulo.

```c
// r4 = objeto, r1 = ClassInfo de la clase que queremos comprobar
instanceof r4, r1       // R0 = 1 si r4 es instancia de r1 o subclase de r1

// Uso en una condicion:
cmpu      r0, 1
jmp.je    es_instancia  // si R0 == 1, es instancia
// ... no es instancia, manejar el caso ...
jmp       fin
es_instancia:
// ... el objeto es del tipo esperado ...
fin:
```

---

## `CHECKCAST` - cast verificado (crash si falla)

```c
checkcast  reg_obj, reg_cls    // R0 = obj_ptr si OK; falla con EVT_ERROR si no
```

Similar a `instanceof` pero con consecuencias: si el objeto NO es del tipo `reg_cls`,
el proceso termina con error. Si es del tipo correcto, `R0 = obj_ptr`.

Convencion especial: si `reg_obj == 0` (puntero nulo), `R0 = 0` sin error (un null
puede castearse a cualquier tipo, convencion similar a JVM).

```c
// Uso tipico: "se que este objeto debe ser de tipo MiClase, si no hay un bug"
checkcast r4, r2        // r4 debe ser instancia de r2; si no, crash
mov       r5, r0        // r5 = obj_ptr verificado (ya sabemos que el tipo es correcto)
```

Diferencia con `instanceof`:
- `instanceof` solo informa (devuelve 0 o 1).
- `checkcast` fuerza el tipo (falla si no coincide).

---

## `GETFIELD` - obtener descriptor de un campo por indice

```c
getfield  reg_cls, field_idx    // R0 = host ptr a FieldInfo
```

`field_idx` es un **inmediato de 8 bits** en la instruccion (0-255). Devuelve un puntero
a la estructura `FieldInfo` que describe ese campo: su nombre, tipo, offset, etc.

Si `field_idx >= field_count`: `R0 = 0`.

### Layout de `FieldInfo` (80 bytes)

```
+0   name.data      (8B)  puntero host al nombre del campo
+8   name.size      (4B)  longitud del nombre
+12  --             (4B)  padding
+16  access         (4B)  enum: PUBLIC=0, PRIVATE=1, PROTECTED=2, DEFAULT=3
+20  kind           (4B)  enum: PRIMITIVE=0, CLASS=1, STRUCT=2, TYPEDEF=3, ENUM=4, ASPECT=5
+24  type_class     (8B)  ClassInfo* (si kind == CLASS o STRUCT; null si no)
+32  size           (4B)  bytes que ocupa el campo en el payload del objeto
+36  offset         (4B)  offset del campo dentro del payload (desde el inicio del objeto)
+40  is_static      (1B)  bool: 1 si el campo es estatico
+41  --             (7B)  padding de alineacion
+48  doc.data       (8B)  puntero host a la cadena de documentacion del campo
+56  doc.size       (4B)  longitud del doc
+60  --             (4B)  padding
+64  attrs          (8B)  AttrEntry* (tabla de anotaciones; null si attr_count == 0)
+72  attr_count     (8B)  size_t - numero de entradas en attrs[]
```

Total: 80 bytes por campo.

```c
// Obtener el primer campo (indice 0) de la clase en r2
getfield r2, 0          // R0 = &ClassInfo.fields[0] (FieldInfo*)
mov      r5, r0         // r5 = FieldInfo*

// Leer el offset del campo dentro del objeto (campo 'offset' en FieldInfo a +36):
mov      r14, r5
addu     r14, 36
xchg     cur0, r14
readcur  r6d, cur0      // r6 = offset del campo (dword)
// Ahora r6 contiene en que byte del objeto empieza este campo
```

---

## `GETMETHOD` - obtener descriptor de un metodo por indice

```c
getmethod  reg_cls, method_idx    // R0 = host ptr a MethodInfo
```

`method_idx` es un inmediato de 8 bits. Devuelve `&ClassInfo.methods[method_idx]`.
Si el indice esta fuera de rango: `R0 = 0`.

### Layout de `MethodInfo` (144 bytes)

```
+0    name.data       (8B)  puntero host al nombre del metodo
+8    name.size       (4B)  longitud del nombre
+12   --              (4B)  padding
+16   descriptor.data (8B)  puntero a la firma, p.ej. "(int,String)int"
+24   descriptor.size (4B)  longitud de la firma
+28   --              (4B)  padding
+32   flags           (8B)  METHOD_FLAG_* (ver abajo)
+40   owner_class     (8B)  ClassInfo* al que pertenece el metodo
+48   args            (8B)  FieldInfo* descripcion de parametros (para reflexion)
+56   arg_count       (8B)  size_t
+64   return_type     (8B)  FieldInfo* tipo de retorno (null = void)
+72   code_vaddr      (8B)  direccion virtual del inicio del bytecode
+80   code_size       (8B)  bytes de bytecode
+88   handlers        (8B)  HandlerException* tabla de handlers de excepcion
+96   handler_count   (8B)  size_t
+104  jit_code        (8B)  void* (null si no esta compilado JIT todavia)
+112  doc.data        (8B)  puntero host a la documentacion del metodo
+120  doc.size        (4B)  longitud del doc
+124  --              (4B)  padding
+128  attrs           (8B)  AttrEntry* (null si attr_count == 0)
+136  attr_count      (8B)  size_t
```

Total: 144 bytes por metodo.

```c
// Obtener la direccion de codigo del metodo 0 de la clase en r2
getmethod r2, 0         // R0 = MethodInfo*
mov       r5, r0

// Leer code_vaddr (offset +72 en MethodInfo):
mov       r14, r5
addu      r14, 72
xchg      cur0, r14
readcur   r6, cur0      // r6 = direccion virtual del codigo del metodo

// Llamar al metodo directamente (saltandose el dispatch virtual):
callvmr   r6
```

### Flags de metodo (`METHOD_FLAG_*`)

| Bit | Flag                       | Significado                      |
| :-: | :------------------------- | :------------------------------- |
|  2  | `METHOD_FLAG_STATIC`       | metodo estatico                  |
|  3  | `METHOD_FLAG_ABSTRACT`     | metodo abstracto (sin cuerpo)    |
|  4  | `METHOD_FLAG_FINAL`        | no puede ser sobreescrito        |
|  8  | `METHOD_FLAG_NATIVE`       | implementado en C++ del runtime  |
|  9  | `METHOD_FLAG_CONSTRUCTOR`  | constructor de clase             |
| 10  | `METHOD_FLAG_VIRTUAL`      | dispatch virtual (polimorfismo)  |
| 11  | `METHOD_FLAG_OVERRIDE`     | sobreescribe un metodo padre     |
| 12  | `METHOD_FLAG_SYNCHRONIZED` | seccion critica (monenter/monexit)|

---

## `FIELDCOUNT` y `METHODCOUNT` - contar miembros de una clase

```c
fieldcount   reg_cls    // R0 = ClassInfo.field_count  (cuantos campos tiene)
methodcount  reg_cls    // R0 = ClassInfo.method_count (cuantos metodos tiene)
```

Devuelven directamente los contadores de la ClassInfo. Util para iterar:

```c
// Imprimir cuantos campos y metodos tiene la clase en r2
fieldcount  r2           // R0 = numero de campos
mov         r10, r0

methodcount r2           // R0 = numero de metodos
mov         r11, r0

// R10 = campos, R11 = metodos
```

> **Nota importante:** `getfield` y `getmethod` usan un indice **inmediato de 8 bits**
> codificado en la instruccion. Esto significa que el indice debe ser una constante conocida
> en tiempo de compilacion. No se puede usar una variable como indice dinamico.
> Para iterar sobre todos los campos con un indice variable, necesitas implementar la
> iteracion en codigo nativo a traves de `ENC` o `CALLN`.

---

## `CLASSNAME` - obtener el nombre de una clase

```c
classname  reg_cls    // R0 = host ptr a char* (ClassInfo.name.data)
```

Devuelve un puntero host directo al primer byte del nombre de la clase (como una cadena C).
Si `name.data == null`: `R0 = 0`.

```c
classname r2            // R0 = puntero host al nombre de la clase (char*)
mov       r5, r0        // r5 = char*

// Leer el primer caracter del nombre:
xchg      cur0, r5      // ATENCION: xchg destruye r5; guarda el ptr antes
readcur   r6b, cur0     // r6 = primer byte del nombre (char)
```

---

## Layout de `ClassInfo` (168 bytes)

La estructura que describe una clase completa:

```
+0    name.data            (8B)  puntero host al nombre de la clase
+8    name.size            (4B)  longitud del nombre
+12   --                   (4B)  padding
+16   flags                (8B)  CLASS_FLAG_* (ver abajo)
+24   instance_size        (4B)  bytes totales de cada objeto (incluye ObjectHeader de 24B)
+28   --                   (4B)  padding
+32   supers               (8B)  ClassInfo** (array de clases padre)
+40   super_count          (8B)  size_t
+48   interfaces           (8B)  ClassInfo** (array de interfaces implementadas)
+56   interface_count      (8B)  size_t
+64   fields               (8B)  FieldInfo* (array de campos)
+72   field_count          (8B)  size_t
+80   vtable               (8B)  MethodInfo** (tabla de metodos virtuales)
+88   vtable_size          (8B)  size_t
+96   static_data          (8B)  uint8_t* (datos estaticos de la clase)
+104  static_fields        (8B)  FieldInfo* (campos estaticos)
+112  static_field_count   (8B)  size_t
+120  methods              (8B)  MethodInfo* (array de metodos)
+128  method_count         (8B)  size_t
+136  doc.data             (8B)  puntero host a la documentacion de la clase
+144  doc.size             (4B)  longitud del doc
+148  --                   (4B)  padding
+152  attrs                (8B)  AttrEntry* (null si attr_count == 0)
+160  attr_count           (8B)  size_t
```

Total: 168 bytes.

### Flags de clase (`CLASS_FLAG_*`)

| Bits | Significado                                                   |
| :--: | :------------------------------------------------------------ |
| 0-1  | Visibilidad: 0=default, 1=public, 2=private, 3=protected      |
|  2   | `CLASS_FLAG_ABSTRACT`  - clase abstracta (no instanciable)    |
|  3   | `CLASS_FLAG_INTERFACE` - es una interfaz                      |
|  4   | `CLASS_FLAG_GENERIC`   - clase generica (plantilla)           |
|  5   | `CLASS_FLAG_ENUM`      - es una enumeracion                   |
|  6   | `CLASS_FLAG_STATIC`    - clase estatica (anidada)             |
|  7   | `CLASS_FLAG_EXCEPTION` - hereda de Throwable                  |
|  8   | `CLASS_FLAG_NATIVE`    - implementada en C++ del runtime      |
|  9   | `CLASS_FLAG_CLOSURE`   - es un ClosureObject                  |

---

## Codificacion binaria

### Formato de dos registros (GETCLASS, INSTANCEOF, CHECKCAST, FIELDCOUNT, METHODCOUNT, CLASSNAME)

```
+-------+--------+--------------------+--------------------+
| 0x00  | opcode |  mode<<6 | 0x00   |  reg2<<4 | reg1   |
+-------+--------+--------------------+--------------------+
  byte 0  byte 1      byte 2               byte 3
```

### Formato registro + imm8 (GETFIELD, GETMETHOD)

```
+-------+--------+------------------+----------------------+
| 0x00  | opcode |  reg_cls & 0x0F  |  field/method_idx    |
+-------+--------+------------------+----------------------+
  byte 0  byte 1      byte 2               byte 3 (imm8)
```

---

Ver tambien: [[NEWOBJRAW y NEWOBJ]], [[CALLVIRT y CALLSUPER]], [[THROW y RETHROW]], [[Doc y Atributos OOP]], [[cursor]], [[GCALLOC]]
