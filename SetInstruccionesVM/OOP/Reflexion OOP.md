# Reflexion OOP - Introspeccion de clases y objetos

Las instrucciones de reflexion permiten consultar en tiempo de ejecucion la clase de un objeto, su jerarquia, sus campos y sus metodos. Todas devuelven el resultado en `R0`.

> **Ver tambien:** [[NEWOBJRAW y NEWOBJ]], [[CALLVIRT y CALLSUPER]], [[THROW y RETHROW]], [[Doc y Atributos OOP]]

---

## Resumen rapido

| Instruccion                       | Opcode      | Entrada                         | R0 resultado               |
| :-------------------------------- | :---------: | :------------------------------ | :------------------------- |
| `getclass  reg_obj`               | `0x00 0xD5` | host ptr a ObjectHeader         | host ptr a ClassInfo       |
| `instanceof reg_obj, reg_cls`     | `0x00 0xD6` | objeto + ClassInfo              | 1 si es instancia, 0 si no |
| `checkcast  reg_obj, reg_cls`     | `0x00 0xD7` | objeto + ClassInfo              | obj ptr o EVT_ERROR        |
| `getfield   reg_cls, idx`         | `0x00 0xD8` | ClassInfo + indice (imm8)       | host ptr a FieldInfo       |
| `getmethod  reg_cls, idx`         | `0x00 0xD9` | ClassInfo + indice (imm8)       | host ptr a MethodInfo      |
| `fieldcount   reg_cls`            | `0x00 0xDA` | host ptr a ClassInfo            | numero de campos           |
| `methodcount  reg_cls`            | `0x00 0xDB` | host ptr a ClassInfo            | numero de metodos          |
| `classname    reg_cls`            | `0x00 0xDC` | host ptr a ClassInfo            | host ptr a char* nombre    |

En todos los casos: si el puntero de entrada es `null` (0), `R0 = 0`.

Para las instrucciones de documentacion y atributos (`classdoc`, `classattrcount`, `classattrkey`, etc.) ver [[Doc y Atributos OOP]].

---

## Tipos de datos en memoria

### `stringx` (16 bytes)

```
+0   data   (8B)  puntero host a uint8_t[]
+8   size   (4B)  longitud en bytes (uint32_t)
+12  --     (4B)  padding de alineacion
```

### `AttrEntry` (32 bytes)

Entrada clave/valor para anotaciones extensibles. Ver [[Doc y Atributos OOP]].

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

```asm
getclass  reg_obj    ; R0 = ClassInfo* del objeto
```

Lee `ObjectHeader.class_ptr` del objeto en `reg_obj` y lo devuelve en `R0`.

```asm
; obj_ptr en r1 (host ptr al ObjectHeader)
getclass r1          ; R0 = ClassInfo* del objeto

; Ahora R0 tiene la ClassInfo; se puede pasar a fieldcount, classname, etc.
mov      r2, r0
fieldcount r2        ; R0 = numero de campos de la clase
```

---

## `INSTANCEOF` - comprobar tipo en jerarquia

```asm
instanceof  reg_obj, reg_cls    ; R0 = 1 si obj es instancia de cls (o subclase)
```

Implementa una busqueda BFS recursiva sobre `ClassInfo.supers[]` e `ClassInfo.interfaces[]`. Devuelve `1` si:
- `obj->class_ptr == cls`, o
- algun super o interfaz de `obj->class_ptr` es compatible con `cls` (transitivo).

Devuelve `0` en caso contrario, o si cualquier puntero es nulo.

```asm
; r4 = objeto, r1 = ClassParent
instanceof r4, r1       ; R0 = 1 si r4 es instancia de ClassParent o subclase

; Uso en condicional:
cmpu      r0, 1
jmp.je    @Absolute("code.es_instancia")
; ... no es instancia ...
jmp       @Absolute("code.fin")
es_instancia:
; ...
fin:
```

---

## `CHECKCAST` - cast con comprobacion de tipo

```asm
checkcast  reg_obj, reg_cls    ; R0 = obj_ptr si OK, o EVT_ERROR si falla
```

- Si `reg_obj == 0` (null): `R0 = 0` (null pasa cualquier cast, convencion JVM).
- Si el objeto es instancia de `reg_cls`: `R0 = obj_ptr`.
- Si no: el proceso termina con `EVT_ERROR` (`THREAD_ILLEGAL_INSTRUCTION`).

```asm
checkcast r4, r2        ; r4 debe ser instancia de r2
mov       r5, r0        ; r5 = puntero tipado (ya verificado)
```

---

## `GETFIELD` - obtener descriptor de campo por indice

```asm
getfield  reg_cls, field_idx    ; R0 = host ptr a FieldInfo
```

`field_idx` es un inmediato de 8 bits (0-255). Si `field_idx >= field_count`: `R0 = 0`.

### Layout de `FieldInfo` (80 bytes)

```
+0   name.data      (8B)  puntero host al nombre del campo
+8   name.size      (4B)  longitud del nombre (uint32_t)
+12  --             (4B)  padding
+16  access         (4B)  enum FieldAccess: PUBLIC=0, PRIVATE=1, PROTECTED=2, DEFAULT=3
+20  kind           (4B)  enum FieldKind: PRIMITIVE=0, CLASS=1, STRUCT=2, TYPEDEF=3, ENUM=4, ASPECT=5
+24  type_class     (8B)  ClassInfo* (valido si kind == CLASS o STRUCT; null si no)
+32  size           (4B)  bytes que ocupa el campo en el payload del objeto
+36  offset         (4B)  offset del campo dentro del payload del objeto
+40  is_static      (1B)  bool
+41  --             (7B)  padding de alineacion
+48  doc.data       (8B)  puntero host a la cadena de documentacion del campo
+56  doc.size       (4B)  longitud del doc (uint32_t)
+60  --             (4B)  padding
+64  attrs          (8B)  AttrEntry* (tabla de anotaciones; null si attr_count == 0)
+72  attr_count     (8B)  size_t - numero de entradas en attrs[]
```

Total: 80 bytes.

```asm
; Obtener el primer campo de la clase en r2
getfield r2, 0          ; R0 = &ClassInfo.fields[0]
mov      r5, r0         ; r5 = FieldInfo*

; Leer offset del campo (a offset +36 en FieldInfo)
mov      r14, r5
addu     r14, 36
xchg     cur0, r14
readcur  r6, cur0       ; r6 = FieldInfo.offset (dword en bits bajos)
```

---

## `GETMETHOD` - obtener descriptor de metodo por indice

```asm
getmethod  reg_cls, method_idx    ; R0 = host ptr a MethodInfo
```

`method_idx` es un inmediato de 8 bits. Devuelve `&ClassInfo.methods[method_idx]`. Si el indice esta fuera de rango: `R0 = 0`.

### Layout de `MethodInfo` (144 bytes)

```
+0    name.data       (8B)  puntero host al nombre del metodo
+8    name.size       (4B)  longitud del nombre (uint32_t)
+12   --              (4B)  padding
+16   descriptor.data (8B)  puntero host a la firma legible, p.ej. "(int,String)int"
+24   descriptor.size (4B)  longitud de la firma (uint32_t)
+28   --              (4B)  padding
+32   flags           (8B)  METHOD_FLAG_* (uint64_t)
+40   owner_class     (8B)  ClassInfo* al que pertenece el metodo
+48   args            (8B)  FieldInfo* descripcion de parametros (reflexion)
+56   arg_count       (8B)  size_t
+64   return_type     (8B)  FieldInfo* tipo de retorno (null = void)
+72   code_vaddr      (8B)  direccion virtual del inicio del bytecode
+80   code_size       (8B)  bytes de bytecode
+88   handlers        (8B)  HandlerException* tabla de handlers de excepcion
+96   handler_count   (8B)  size_t
+104  jit_code        (8B)  void* (null si no esta compilado JIT)
+112  doc.data        (8B)  puntero host a la cadena de documentacion del metodo
+120  doc.size        (4B)  longitud del doc (uint32_t)
+124  --              (4B)  padding
+128  attrs           (8B)  AttrEntry* (null si attr_count == 0)
+136  attr_count      (8B)  size_t
```

Total: 144 bytes.

```asm
; Obtener la direccion virtual del codigo del metodo 0
getmethod r2, 0         ; R0 = MethodInfo*
mov       r5, r0

; Leer code_vaddr (offset +72)
mov       r14, r5
addu      r14, 72
xchg      cur0, r14
readcur   r6, cur0      ; r6 = direccion virtual del codigo

; Saltar directamente
callvm r6
```

### Flags de metodo (`METHOD_FLAG_*`)

| Bit | Flag                    | Significado                     |
| :-: | :---------------------- | :------------------------------ |
| 2   | `METHOD_FLAG_STATIC`    | metodo estatico                 |
| 3   | `METHOD_FLAG_ABSTRACT`  | metodo abstracto                |
| 4   | `METHOD_FLAG_FINAL`     | no puede ser sobreescrito       |
| 8   | `METHOD_FLAG_NATIVE`    | implementado en C++ del runtime |
| 9   | `METHOD_FLAG_CONSTRUCTOR` | constructor de clase           |
| 10  | `METHOD_FLAG_VIRTUAL`   | dispatch virtual                |
| 11  | `METHOD_FLAG_OVERRIDE`  | sobreescribe un metodo padre    |
| 12  | `METHOD_FLAG_SYNCHRONIZED` | seccion critica               |

---

## `FIELDCOUNT` y `METHODCOUNT`

```asm
fieldcount   reg_cls    ; R0 = ClassInfo.field_count
methodcount  reg_cls    ; R0 = ClassInfo.method_count
```

Acceso directo a los contadores. Util para iterar sobre campos o metodos:

```asm
fieldcount r2           ; R0 = N campos
mov        r12, r0      ; r12 = contador del loop

mov        r11, 0       ; indice actual
field_loop:
    cmpu   r11, r12
    jmp.je @Absolute("code.field_loop_end")

    getfield r2, r11    ; R0 = &FieldInfo[r11]  -- NOTA: r11 debe ser imm8
    ; ...procesar campo...

    addu   r11, 1
    jmp    @Absolute("code.field_loop")
field_loop_end:
```

> **Nota:** `getfield` y `getmethod` usan un indice **inmediato de 8 bits** codificado en la instruccion. Para iterar dinamicamente sobre todos los campos se necesita una llamada con indice fijo por cada campo, o implementar la iteracion en codigo nativo a traves de `ENC`.

---

## `CLASSNAME` - obtener el nombre de una clase

```asm
classname  reg_cls    ; R0 = host ptr a char* (ClassInfo.name.data)
```

Devuelve un puntero host directo a la cadena de nombre de la clase. Si `name.data == null`: `R0 = 0`.

```asm
classname r2            ; R0 = puntero host al nombre de la clase
mov       r5, r0        ; r5 = char* nombre

; Leer el primer caracter del nombre:
xchg      cur0, r5
readcur   r6b, cur0     ; r6 = primer byte del nombre (char)
```

### Layout de `ClassInfo` (168 bytes)

```
+0    name.data            (8B)  puntero host al nombre de la clase
+8    name.size            (4B)  longitud del nombre (uint32_t)
+12   --                   (4B)  padding
+16   flags                (8B)  CLASS_FLAG_* (uint64_t)
+24   instance_size        (4B)  bytes totales del objeto (incluye ObjectHeader)
+28   --                   (4B)  padding
+32   supers               (8B)  ClassInfo**
+40   super_count          (8B)  size_t
+48   interfaces           (8B)  ClassInfo**
+56   interface_count      (8B)  size_t
+64   fields               (8B)  FieldInfo*
+72   field_count          (8B)  size_t
+80   vtable               (8B)  MethodInfo**
+88   vtable_size          (8B)  size_t
+96   static_data          (8B)  uint8_t*
+104  static_fields        (8B)  FieldInfo*
+112  static_field_count   (8B)  size_t
+120  methods              (8B)  MethodInfo*
+128  method_count         (8B)  size_t
+136  doc.data             (8B)  puntero host a la cadena de documentacion de la clase
+144  doc.size             (4B)  longitud del doc (uint32_t)
+148  --                   (4B)  padding
+152  attrs                (8B)  AttrEntry* (null si attr_count == 0)
+160  attr_count           (8B)  size_t
```

Total: 168 bytes.

### Flags de clase (`CLASS_FLAG_*`)

| Bits 0-1 | Visibilidad: 0=default, 1=public, 2=private, 3=protected |
| :------- | :-------------------------------------------------------- |
| Bit 2    | `CLASS_FLAG_ABSTRACT`  - clase abstracta                  |
| Bit 3    | `CLASS_FLAG_INTERFACE` - es interfaz                      |
| Bit 4    | `CLASS_FLAG_FINAL`     - no puede ser extendida           |
| Bit 5    | `CLASS_FLAG_ENUM`      - es enumeracion                   |
| Bit 6    | `CLASS_FLAG_STATIC`    - clase estatica (anidada)         |
| Bit 7    | `CLASS_FLAG_EXCEPTION` - hereda de Throwable              |
| Bit 8    | `CLASS_FLAG_NATIVE`    - implementada en C++ del runtime  |

---

## Codificacion binaria

### Formato de dos registros (GETCLASS, INSTANCEOF, CHECKCAST, FIELDCOUNT, METHODCOUNT, CLASSNAME)

```
+-------+--------+--------------------+--------------------+
| 0x00  | opcode |  mode<<6 | 0x00   |  reg2<<4 | reg1   |
+-------+--------+--------------------+--------------------+
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
