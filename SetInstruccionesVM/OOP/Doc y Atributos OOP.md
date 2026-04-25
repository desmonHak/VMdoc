# Doc y Atributos OOP - Documentacion y anotaciones en tiempo de ejecucion

Las instrucciones de documentacion permiten acceder en tiempo de ejecucion a las cadenas
de documentacion (`doc`) y a la tabla de anotaciones clave/valor (`attrs`) almacenadas en
`ClassInfo`, `MethodInfo` y `FieldInfo`. Son el equivalente en VestaVM de la reflexion de
anotaciones en Java o los atributos en C#.

> **Ver tambien:** [[Reflexion OOP]], [[NEWOBJRAW y NEWOBJ]], [[CALLVIRT y CALLSUPER]]

---

## Para que sirven

Las anotaciones (`attrs`) son pares clave/valor que se adjuntan a clases, metodos o campos.
Permiten que el codigo que usa una biblioteca consulte metadata adicional: si un metodo
esta obsoleto, desde que version existe, quien lo escribio, restricciones de uso, etc.

Ejemplo de anotaciones en un lenguaje de alto nivel:
```c
// @Deprecated("Usar nuevo_metodo() en su lugar")
// @Since("2.0")
// metodo_viejo(...)
```

Estas instrucciones permiten leer esa metadata en tiempo de ejecucion desde bytecode Vesta.

---

## Resumen rapido

| Instruccion                         | Opcode       | Entrada                              | R0 resultado                         |
| :---------------------------------- | :----------: | :----------------------------------- | :----------------------------------- |
| `classdoc       reg_cls`            | `0x00 0xDD`  | host ptr a ClassInfo                 | host ptr a `char*` (doc.data)        |
| `classattrcount reg_cls`            | `0x00 0xDE`  | host ptr a ClassInfo                 | numero de entradas en attrs[]        |
| `classattrkey   reg_cls, idx`       | `0x00 0xDF`  | ClassInfo + indice imm8              | host ptr al nombre de la anotacion   |
| `classattrval   reg_cls, idx`       | `0x00 0xE0`  | ClassInfo + indice imm8              | host ptr al valor de la anotacion    |
| `methodname     reg_meth`           | `0x00 0xE1`  | host ptr a MethodInfo                | host ptr a `char*` (name.data)       |
| `methoddoc      reg_meth`           | `0x00 0xE2`  | host ptr a MethodInfo                | host ptr a `char*` (doc.data)        |
| `methoddesc     reg_meth`           | `0x00 0xE3`  | host ptr a MethodInfo                | host ptr a `char*` (descriptor.data) |
| `methodattrcount reg_meth`          | `0x00 0xE4`  | host ptr a MethodInfo                | numero de entradas en attrs[]        |
| `methodattrkey  reg_meth, idx`      | `0x00 0xE5`  | MethodInfo + indice imm8             | host ptr al nombre de la anotacion   |
| `methodattrval  reg_meth, idx`      | `0x00 0xE6`  | MethodInfo + indice imm8             | host ptr al valor de la anotacion    |
| `fieldname      reg_fld`            | `0x00 0xE7`  | host ptr a FieldInfo                 | host ptr a `char*` (name.data)       |
| `fielddoc       reg_fld`            | `0x00 0xE8`  | host ptr a FieldInfo                 | host ptr a `char*` (doc.data)        |
| `fieldattrcount reg_fld`            | `0x00 0xE9`  | host ptr a FieldInfo                 | numero de entradas en attrs[]        |
| `fieldattrkey   reg_fld, idx`       | `0x00 0xEA`  | FieldInfo + indice imm8              | host ptr al nombre de la anotacion   |
| `fieldattrval   reg_fld, idx`       | `0x00 0xEB`  | FieldInfo + indice imm8              | host ptr al valor de la anotacion    |

En todos los casos: si el puntero de entrada es `null` (0), o el indice esta fuera de
rango, `R0 = 0`.

---

## Estructura `AttrEntry` (32 bytes)

Cada anotacion es un par clave/valor almacenado en un `AttrEntry`:

```cpp
// Definicion en C++:
typedef struct AttrEntry {
    stringx key;    // nombre de la anotacion ("Deprecated", "Since", "Author", ...)
    stringx value;  // valor de la anotacion  ("true", "1.4", "David", ...)
} AttrEntry;
```

Layout en memoria:

```
+0   key.data    (8B)  puntero host al nombre de la anotacion (char*)
+8   key.size    (4B)  longitud del nombre en bytes (uint32_t)
+12  --          (4B)  padding de alineacion
+16  value.data  (8B)  puntero host al valor de la anotacion (char*)
+24  value.size  (4B)  longitud del valor en bytes (uint32_t)
+28  --          (4B)  padding de alineacion
```

Total: 32 bytes por entrada. La tabla `attrs` es un array contiguo de `AttrEntry` en
memoria host; `attrs[i]` esta en `attrs + i * 32`.

---

## Instrucciones de clase

### `CLASSDOC` - docstring de una clase

```c
classdoc  reg_cls    // R0 = ClassInfo.doc.data (host ptr a char*)
```

Lee `ClassInfo.doc.data` (offset +136 en ClassInfo) y lo devuelve en R0. Es el texto de
documentacion de la clase. Si `doc.data == null`: R0 = 0 (sin documentacion).

### `CLASSATTRCOUNT` - numero de anotaciones de una clase

```c
classattrcount  reg_cls    // R0 = ClassInfo.attr_count
```

Lee `ClassInfo.attr_count` (offset +160 en ClassInfo). Devuelve 0 si no hay anotaciones.

### `CLASSATTRKEY` - clave de una anotacion de clase

```c
classattrkey  reg_cls, idx    // R0 = ClassInfo.attrs[idx].key.data
```

`idx` es un inmediato de 8 bits (0-255). Devuelve el puntero al nombre de la anotacion
numero `idx`. Si `idx >= attr_count` o `attrs == null`: R0 = 0.

### `CLASSATTRVAL` - valor de una anotacion de clase

```c
classattrval  reg_cls, idx    // R0 = ClassInfo.attrs[idx].value.data
```

Misma semantica que `classattrkey` pero devuelve `value.data` (el valor) en lugar de
`key.data` (el nombre).

```c
// Ejemplo: leer la primera anotacion de la clase en r2
classattrcount r2       // R0 = numero de anotaciones

classattrkey   r2, 0    // R0 = puntero al nombre de la anotacion[0]
mov   r11, r0           // r11 = char* nombre (p.ej. "Deprecated")

classattrval   r2, 0    // R0 = puntero al valor de la anotacion[0]
mov   r12, r0           // r12 = char* valor (p.ej. "true")

classdoc       r2       // R0 = puntero al docstring de la clase
mov   r13, r0           // r13 = char* documentacion
```

---

## Instrucciones de metodo

### `METHODNAME` - nombre de un metodo

```c
methodname  reg_meth    // R0 = MethodInfo.name.data (char*)
```

Lee `MethodInfo.name.data` (offset +0 en MethodInfo). El nombre simple del metodo.

### `METHODDOC` - docstring de un metodo

```c
methoddoc  reg_meth    // R0 = MethodInfo.doc.data (char*)
```

Lee `MethodInfo.doc.data` (offset +112 en MethodInfo). La documentacion del metodo.

### `METHODDESC` - descriptor/firma de un metodo

```c
methoddesc  reg_meth    // R0 = MethodInfo.descriptor.data (char*)
```

Lee `MethodInfo.descriptor.data` (offset +16 en MethodInfo). La firma es una cadena
legible que describe parametros y tipo de retorno, por ejemplo `"(int,String)int"`.

### `METHODATTRCOUNT`, `METHODATTRKEY`, `METHODATTRVAL`

```c
methodattrcount  reg_meth    // R0 = MethodInfo.attr_count
methodattrkey    reg_meth, idx    // R0 = MethodInfo.attrs[idx].key.data
methodattrval    reg_meth, idx    // R0 = MethodInfo.attrs[idx].value.data
```

```c
// Ejemplo: leer nombre, firma y primera anotacion del metodo en r5
methodname      r5      // R0 = nombre
mov  r6, r0
methoddesc      r5      // R0 = firma (p.ej. "(int,int)int")
mov  r7, r0
methoddoc       r5      // R0 = docstring
mov  r8, r0
methodattrcount r5      // R0 = numero de anotaciones
mov  r9, r0
methodattrkey   r5, 0   // R0 = nombre de la anotacion[0]
mov  r10, r0
methodattrval   r5, 0   // R0 = valor de la anotacion[0]
mov  r11, r0
```

---

## Instrucciones de campo

### `FIELDNAME` - nombre de un campo

```c
fieldname  reg_fld    // R0 = FieldInfo.name.data (char*)
```

Lee `FieldInfo.name.data` (offset +0 en FieldInfo).

### `FIELDDOC` - docstring de un campo

```c
fielddoc  reg_fld    // R0 = FieldInfo.doc.data (char*)
```

Lee `FieldInfo.doc.data` (offset +48 en FieldInfo).

### `FIELDATTRCOUNT`, `FIELDATTRKEY`, `FIELDATTRVAL`

```c
fieldattrcount  reg_fld    // R0 = FieldInfo.attr_count
fieldattrkey    reg_fld, idx    // R0 = FieldInfo.attrs[idx].key.data
fieldattrval    reg_fld, idx    // R0 = FieldInfo.attrs[idx].value.data
```

```c
// Ejemplo: leer nombre y doc de un campo
getfield   r2, 0        // R0 = FieldInfo* del primer campo
mov        r5, r0

fieldname  r5           // R0 = nombre del campo
mov        r6, r0

fielddoc   r5           // R0 = docstring del campo
mov        r7, r0

fieldattrcount r5       // R0 = numero de anotaciones (0 si ninguna)
mov        r8, r0
```

---

## Patron de uso: construir estructuras con doc y attrs manualmente

Al construir `ClassInfo`, `MethodInfo` o `FieldInfo` en bytecode (con `alloc` + `writecur`),
los campos doc y attrs deben inicializarse explicitamente. `alloc` devuelve memoria sin
inicializar, por lo que es necesario escribir todos los campos.

```c
// Crear un MethodInfo con doc y un AttrEntry
mov  r1, 144
alloc r1
mov  r14, r0        // r14 = MethodInfo* (host ptr)

// Escribir name.data (offset 0): puntero al nombre del metodo
mov  r8, r14
xchg cur0, r8
writecur cur0, r7   // r7 = puntero al string pool con el nombre

// Escribir doc.data (offset 112): puntero a la documentacion
mov  r8, r14
addu r8, 112
xchg cur0, r8
writecur cur0, r7   // r7 = puntero a la cadena de documentacion

// Crear un AttrEntry (32B) para una anotacion
mov  r1, 32
alloc r1
mov  r4, r0         // r4 = AttrEntry*

// AttrEntry.key.data (offset 0 del AttrEntry)
mov  r8, r4
xchg cur0, r8
writecur cur0, r7   // r7 = puntero al nombre de la anotacion ("Deprecated")

// AttrEntry.value.data (offset 16 del AttrEntry)
mov  r8, r4
addu r8, 16
xchg cur0, r8
writecur cur0, r7   // r7 = puntero al valor de la anotacion ("true")

// MethodInfo.attrs (offset 128) = puntero al array de AttrEntry
mov  r8, r14
addu r8, 128
xchg cur0, r8
writecur cur0, r4   // r4 = AttrEntry*

// MethodInfo.attr_count (offset 136) = 1
mov  r8, r14
addu r8, 136
xchg cur0, r8
mov  r1, 1
writecur cur0, r1

// Verificar leyendo de vuelta:
methodattrcount r14     // R0 = 1 (correcto)
methodattrkey   r14, 0  // R0 = key.data (puntero al nombre, != 0)
methodattrval   r14, 0  // R0 = value.data (puntero al valor, != 0)
```

> **Nota sobre xchg:** `xchg cur0, rN` es un swap: intercambia el valor del cursor con rN,
> destruyendo el valor de rN. Para preservar un puntero base, usa el patron
> `mov r8, base / addu r8, offset / xchg cur0, r8` antes de cada acceso.

---

## Codificacion binaria

### Instrucciones de un registro (ONE_REG)

`classdoc`, `classattrcount`, `methodname`, `methoddoc`, `methoddesc`, `methodattrcount`,
`fieldname`, `fielddoc`, `fieldattrcount`

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
