# Doc y Atributos OOP - Documentacion y anotaciones en tiempo de ejecucion

Las instrucciones de documentacion permiten acceder en tiempo de ejecucion a las cadenas de documentacion (`doc`) y a la tabla de anotaciones clave/valor (`attrs`) almacenadas en `ClassInfo`, `MethodInfo` y `FieldInfo`. Todas devuelven el resultado en `R0`.

> **Ver tambien:** [[Reflexion OOP]], [[NEWOBJRAW y NEWOBJ]], [[CALLVIRT y CALLSUPER]]

---

## Resumen rapido

| Instruccion                         | Opcode      | Entrada                              | R0 resultado                        |
| :---------------------------------- | :---------: | :----------------------------------- | :---------------------------------- |
| `classdoc       reg_cls`            | `0x00 0xDD` | host ptr a ClassInfo                 | host ptr a `char*` (doc.data)       |
| `classattrcount reg_cls`            | `0x00 0xDE` | host ptr a ClassInfo                 | numero de entradas en attrs[]       |
| `classattrkey   reg_cls, idx`       | `0x00 0xDF` | ClassInfo + indice imm8              | host ptr al nombre de la anotacion  |
| `classattrval   reg_cls, idx`       | `0x00 0xE0` | ClassInfo + indice imm8              | host ptr al valor de la anotacion   |
| `methodname     reg_meth`           | `0x00 0xE1` | host ptr a MethodInfo                | host ptr a `char*` (name.data)      |
| `methoddoc      reg_meth`           | `0x00 0xE2` | host ptr a MethodInfo                | host ptr a `char*` (doc.data)       |
| `methoddesc     reg_meth`           | `0x00 0xE3` | host ptr a MethodInfo                | host ptr a `char*` (descriptor.data)|
| `methodattrcount reg_meth`          | `0x00 0xE4` | host ptr a MethodInfo                | numero de entradas en attrs[]       |
| `methodattrkey  reg_meth, idx`      | `0x00 0xE5` | MethodInfo + indice imm8             | host ptr al nombre de la anotacion  |
| `methodattrval  reg_meth, idx`      | `0x00 0xE6` | MethodInfo + indice imm8             | host ptr al valor de la anotacion   |
| `fieldname      reg_fld`            | `0x00 0xE7` | host ptr a FieldInfo                 | host ptr a `char*` (name.data)      |
| `fielddoc       reg_fld`            | `0x00 0xE8` | host ptr a FieldInfo                 | host ptr a `char*` (doc.data)       |
| `fieldattrcount reg_fld`            | `0x00 0xE9` | host ptr a FieldInfo                 | numero de entradas en attrs[]       |
| `fieldattrkey   reg_fld, idx`       | `0x00 0xEA` | FieldInfo + indice imm8              | host ptr al nombre de la anotacion  |
| `fieldattrval   reg_fld, idx`       | `0x00 0xEB` | FieldInfo + indice imm8              | host ptr al valor de la anotacion   |

En todos los casos: si el puntero de entrada es `null` (0), o el indice esta fuera de rango, `R0 = 0`.

---

## Estructura `AttrEntry` (32 bytes)

Cada anotacion es un par clave/valor almacenado en un `AttrEntry`:

```cpp
typedef struct AttrEntry {
    stringx key;    // nombre de la anotacion ("Deprecated", "Since", "Author", ...)
    stringx value;  // valor de la anotacion  ("true", "1.4", "David", ...)
} AttrEntry;
```

Layout en memoria:

```
+0   key.data    (8B)  puntero host al nombre de la anotacion
+8   key.size    (4B)  longitud del nombre (uint32_t)
+12  --          (4B)  padding
+16  value.data  (8B)  puntero host al valor de la anotacion
+24  value.size  (4B)  longitud del valor (uint32_t)
+28  --          (4B)  padding
```

Total: 32 bytes. La tabla `attrs` es un array contiguo de `AttrEntry` en memoria host.

---

## Instrucciones de clase

### `CLASSDOC` - docstring de una clase

```asm
classdoc  reg_cls    ; R0 = ClassInfo.doc.data (host ptr a char*)
```

Lee `ClassInfo.doc.data` (offset +136) y lo devuelve en `R0`. Si `doc.data == null`: `R0 = 0`.

### `CLASSATTRCOUNT` - numero de anotaciones de una clase

```asm
classattrcount  reg_cls    ; R0 = ClassInfo.attr_count
```

Lee `ClassInfo.attr_count` (offset +160). Devuelve 0 si el puntero es null.

### `CLASSATTRKEY` - clave de una anotacion de clase

```asm
classattrkey  reg_cls, idx    ; R0 = ClassInfo.attrs[idx].key.data
```

`idx` es un inmediato de 8 bits (0-255). Si `idx >= attr_count` o `attrs == null`: `R0 = 0`.

### `CLASSATTRVAL` - valor de una anotacion de clase

```asm
classattrval  reg_cls, idx    ; R0 = ClassInfo.attrs[idx].value.data
```

Misma semantica que `classattrkey` pero devuelve `value.data` en lugar de `key.data`.

```asm
; Ejemplo: leer la primera anotacion de la clase en r2
classattrcount r2       ; R0 = numero de anotaciones
mov   r10, r0

classattrkey   r2, 0    ; R0 = puntero al nombre de la anotacion[0]
mov   r11, r0

classattrval   r2, 0    ; R0 = puntero al valor de la anotacion[0]
mov   r12, r0

classdoc       r2       ; R0 = puntero al docstring de la clase
mov   r13, r0
```

---

## Instrucciones de metodo

### `METHODNAME` - nombre de un metodo

```asm
methodname  reg_meth    ; R0 = MethodInfo.name.data
```

Lee `MethodInfo.name.data` (offset +0). Equivalente a leer el nombre del metodo directamente.

### `METHODDOC` - docstring de un metodo

```asm
methoddoc  reg_meth    ; R0 = MethodInfo.doc.data
```

Lee `MethodInfo.doc.data` (offset +112).

### `METHODDESC` - descriptor/firma de un metodo

```asm
methoddesc  reg_meth    ; R0 = MethodInfo.descriptor.data
```

Lee `MethodInfo.descriptor.data` (offset +16). La firma es una cadena legible como `"(int,String)int"`.

### `METHODATTRCOUNT` - numero de anotaciones de un metodo

```asm
methodattrcount  reg_meth    ; R0 = MethodInfo.attr_count
```

### `METHODATTRKEY` y `METHODATTRVAL`

```asm
methodattrkey  reg_meth, idx    ; R0 = MethodInfo.attrs[idx].key.data
methodattrval  reg_meth, idx    ; R0 = MethodInfo.attrs[idx].value.data
```

```asm
; Ejemplo: leer nombre, firma y primera anotacion del metodo en r5
methodname      r5      ; R0 = nombre
mov  r6, r0
methoddesc      r5      ; R0 = firma
mov  r7, r0
methoddoc       r5      ; R0 = docstring
mov  r8, r0
methodattrcount r5      ; R0 = numero de anotaciones
mov  r9, r0
methodattrkey   r5, 0   ; R0 = clave de anotacion[0]
mov  r10, r0
methodattrval   r5, 0   ; R0 = valor de anotacion[0]
mov  r11, r0
```

---

## Instrucciones de campo

### `FIELDNAME` - nombre de un campo

```asm
fieldname  reg_fld    ; R0 = FieldInfo.name.data
```

Lee `FieldInfo.name.data` (offset +0).

### `FIELDDOC` - docstring de un campo

```asm
fielddoc  reg_fld    ; R0 = FieldInfo.doc.data
```

Lee `FieldInfo.doc.data` (offset +48).

### `FIELDATTRCOUNT` - numero de anotaciones de un campo

```asm
fieldattrcount  reg_fld    ; R0 = FieldInfo.attr_count
```

Lee `FieldInfo.attr_count` (offset +72). Si el campo no tiene anotaciones: `R0 = 0`.

### `FIELDATTRKEY` y `FIELDATTRVAL`

```asm
fieldattrkey  reg_fld, idx    ; R0 = FieldInfo.attrs[idx].key.data
fieldattrval  reg_fld, idx    ; R0 = FieldInfo.attrs[idx].value.data
```

```asm
; Ejemplo: leer nombre y doc de un campo
getfield   r2, 0        ; R0 = FieldInfo* del primer campo
mov        r5, r0

fieldname  r5           ; R0 = nombre del campo
mov        r6, r0

fielddoc   r5           ; R0 = docstring del campo
mov        r7, r0

fieldattrcount r5       ; R0 = numero de anotaciones (0 si ninguna)
mov        r8, r0
```

---

## Patron de uso: preparar estructuras manualmente

Al construir `ClassInfo`, `MethodInfo` o `FieldInfo` en bytecode (con `alloc` + `writecur`), los campos doc y attrs deben inicializarse explicitamente. `alloc` devuelve memoria zeroeada, por lo que `doc.data == null` y `attr_count == 0` por defecto si no se escriben.

```asm
; Crear un MethodInfo con doc y un AttrEntry
mov  r1, 144
alloc r1
mov  r14, r0        ; r14 = MethodInfo*

; Escribir name.data (offset 0)
mov  r8, r14
xchg cur0, r8
writecur cur0, r7   ; r7 = puntero al string pool

; Escribir doc.data (offset 112)
mov  r8, r14
addu r8, 112
xchg cur0, r8
writecur cur0, r7

; Crear un AttrEntry (32B)
mov  r1, 32
alloc r1
mov  r4, r0         ; r4 = AttrEntry*

; AttrEntry.key.data = r7
mov  r8, r4
xchg cur0, r8
writecur cur0, r7

; AttrEntry.value.data = r7 (offset +16)
mov  r8, r4
addu r8, 16
xchg cur0, r8
writecur cur0, r7

; MethodInfo.attrs (offset 128) = r4
mov  r8, r14
addu r8, 128
xchg cur0, r8
writecur cur0, r4

; MethodInfo.attr_count (offset 136) = 1
mov  r8, r14
addu r8, 136
xchg cur0, r8
mov  r1, 1
writecur cur0, r1

; Leer de vuelta:
methodattrcount r14     ; R0 = 1
methodattrkey   r14, 0  ; R0 = key.data != 0
methodattrval   r14, 0  ; R0 = value.data != 0
```

> **Nota sobre xchg:** `xchg cur0, rN` es un swap destructivo: intercambia el valor del cursor con `rN`, destruyendo `rN`. Para preservar un puntero base, usar el patron `mov r8, rN / addu r8, offset / xchg cur0, r8` antes de cada acceso.

---

## Codificacion binaria

### Instrucciones de un registro (ONE_REG)

`classdoc`, `classattrcount`, `methodname`, `methoddoc`, `methoddesc`, `methodattrcount`, `fieldname`, `fielddoc`, `fieldattrcount`

```
+-------+--------+--------------------+--------------------+
| 0x00  | opcode |  mode<<6 | 0x00   |  0000 | reg        |
+-------+--------+--------------------+--------------------+
  byte 0  byte 1      byte 2               byte 3
```

### Instrucciones registro + imm8 (REG_IMM8)

`classattrkey`, `classattrval`, `methodattrkey`, `methodattrval`, `fieldattrkey`, `fieldattrval`

```
+-------+--------+------------------+----------------------+
| 0x00  | opcode |  reg & 0x0F      |  attr_idx (imm8)     |
+-------+--------+------------------+----------------------+
  byte 0  byte 1      byte 2               byte 3
```

---

Ver tambien: [[Reflexion OOP]], [[NEWOBJRAW y NEWOBJ]], [[CALLVIRT y CALLSUPER]], [[cursor]]
