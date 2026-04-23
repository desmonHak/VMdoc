# Reflexión OOP - Introspección de clases y objetos

Las instrucciones de reflexión permiten consultar en tiempo de ejecución la clase de un objeto, su jerarquía, sus campos y sus métodos. Todas devuelven el resultado en `R0`.

> **Ver también:** [[NEWOBJRAW y NEWOBJ]], [[CALLVIRT y CALLSUPER]], [[THROW y RETHROW]]

---

## Resumen rápido

| Instrucción              | Opcode     | Entrada                    | R0 resultado              |
| :----------------------- | :--------: | :------------------------- | :------------------------ |
| `getclass  reg_obj`      | `0x00 0xD5`| host ptr a ObjectHeader    | host ptr a ClassInfo      |
| `instanceof reg_obj, reg_cls` | `0x00 0xD6` | objeto + ClassInfo    | 1 si es instancia, 0 si no|
| `checkcast reg_obj, reg_cls`  | `0x00 0xD7` | objeto + ClassInfo    | obj ptr o EVT_ERROR       |
| `getfield  reg_cls, idx` | `0x00 0xD8`| ClassInfo + índice (imm8)  | host ptr a FieldInfo      |
| `getmethod reg_cls, idx` | `0x00 0xD9`| ClassInfo + índice (imm8)  | host ptr a MethodInfo     |
| `fieldcount  reg_cls`    | `0x00 0xDA`| host ptr a ClassInfo       | número de campos          |
| `methodcount reg_cls`    | `0x00 0xDB`| host ptr a ClassInfo       | número de métodos         |
| `classname   reg_cls`    | `0x00 0xDC`| host ptr a ClassInfo       | host ptr a `char*` nombre |

En todos los casos: si el puntero de entrada es `null` (0), `R0 = 0`.

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
fieldcount r2        ; R0 = número de campos de la clase
```

---

## `INSTANCEOF` - comprobar tipo en jerarquía

```asm
instanceof  reg_obj, reg_cls    ; R0 = 1 si obj es instancia de cls (o subclase)
```

Implementa una búsqueda BFS recursiva sobre `ClassInfo.supers[]` e `ClassInfo.interfaces[]`. Devuelve `1` si:
- `obj->class_ptr == cls`, o
- algún super o interfaz de `obj->class_ptr` es compatible con `cls` (transitivo).

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

## `CHECKCAST` - cast con comprobación de tipo

```asm
checkcast  reg_obj, reg_cls    ; R0 = obj_ptr si OK, o EVT_ERROR si falla
```

- Si `reg_obj == 0` (null): `R0 = 0` (null pasa cualquier cast, convención JVM).
- Si el objeto es instancia de `reg_cls`: `R0 = obj_ptr`.
- Si no: el proceso termina con `EVT_ERROR` (`THREAD_ILLEGAL_INSTRUCTION`).

```asm
checkcast r4, r2        ; r4 debe ser instancia de r2
mov       r5, r0        ; r5 = puntero tipado (ya verificado)
```

---

## `GETFIELD` - obtener descriptor de campo por índice

```asm
getfield  reg_cls, field_idx    ; R0 = host ptr a FieldInfo
```

`field_idx` es un inmediato de 8 bits (0–255). Si `field_idx >= field_count`: `R0 = 0`.

### Layout de `FieldInfo` (40 bytes)

```
+0   name.data     (8B)  puntero host al nombre del campo
+8   name.length   (8B)  longitud del nombre
+16  flags         (8B)  bits de modificadores
+24  offset        (4B)  offset del campo dentro del payload del objeto
+28  size          (4B)  tamaño del campo en bytes
+32  access        (4B)  enum: PUBLIC=0, PRIVATE=1, PROTECTED=2
+36  kind          (4B)  enum: INT=0, FLOAT=1, REF=2, RAW=3
```

```asm
; Obtener el primer campo de la clase en r2
getfield r2, 0          ; R0 = &ClassInfo.fields[0]
mov      r5, r0         ; r5 = FieldInfo*

; Leer offset del campo (a offset +24 en FieldInfo)
xchg     cur0, r5
mov      r14, r5 ; addu r14, 24 ; xchg cur0, r14
readcur  r6, cur0       ; r6 = FieldInfo.offset (dword, en los 32 bits bajos)
```

---

## `GETMETHOD` - obtener descriptor de método por índice

```asm
getmethod  reg_cls, method_idx    ; R0 = host ptr a MethodInfo
```

`method_idx` es un inmediato de 8 bits. Devuelve `&ClassInfo.methods[method_idx]`. Si el índice está fuera de rango: `R0 = 0`.

### Layout de `MethodInfo` (56 bytes)

```
+0   name.data       (8B)  puntero host al nombre del método
+8   name.length     (8B)  longitud del nombre
+16  flags           (8B)  bits de modificadores
+24  code_vaddr      (8B)  dirección virtual del código del método
+32  param_count     (4B)  número de parámetros
+36  local_count     (4B)  número de variables locales
+40  handlers*       (8B)  puntero host a tabla HandlerException[]
+48  handler_count   (8B)  número de entradas en la tabla
```

```asm
; Obtener código virtual del método 0
getmethod r2, 0         ; R0 = MethodInfo*
mov       r5, r0

; Leer code_vaddr (offset 24)
mov       r14, r5 ; addu r14, 24 ; xchg cur0, r14
readcur   r6, cur0      ; r6 = dirección virtual del código

; Saltar directamente (CALLVM al código virtual)
callvm r6
```

---

## `FIELDCOUNT` y `METHODCOUNT`

```asm
fieldcount   reg_cls    ; R0 = ClassInfo.field_count
methodcount  reg_cls    ; R0 = ClassInfo.method_count
```

Acceso directo a los contadores. Útil para iterar sobre campos o métodos:

```asm
fieldcount r2           ; R0 = N campos
mov        r12, r0      ; r12 = contador del loop

mov        r11, 0       ; índice actual
field_loop:
    cmpu   r11, r12
    jmp.je @Absolute("code.field_loop_end")

    getfield r2, r11    ; R0 = &FieldInfo[r11]  -- NOTA: r11 debe ser imm8
    ; ...procesar campo...

    addu   r11, 1
    jmp    @Absolute("code.field_loop")
field_loop_end:
```

> **Nota:** `getfield` y `getmethod` usan un índice **inmediato de 8 bits** codificado en la instrucción. Para iterar dinámicamente sobre todos los campos se necesita una llamada con índice fijo por cada campo, o implementar la iteración en código nativo a través de `ENC`.

---

## `CLASSNAME` - obtener el nombre de una clase

```asm
classname  reg_cls    ; R0 = host ptr a char* (ClassInfo.name.data)
```

Devuelve un puntero host directo a la cadena de nombre de la clase. Si `name.data == null`: `R0 = 0`.

```asm
classname r2            ; R0 = puntero host al nombre de la clase
mov       r5, r0        ; r5 = char* nombre

; Leer el primer carácter del nombre:
xchg      cur0, r5
readcur   r6b, cur0     ; r6 = primer byte del nombre (char)
```

---

## Codificación binaria

### Formato de dos registros (GETCLASS, INSTANCEOF, CHECKCAST, FIELDCOUNT, METHODCOUNT, CLASSNAME)

```
┌────────┬────────┬────────────────────┬────────────────────┐
│ 0x00   │ opcode │  mode<<6 | 0x00    │  reg2<<4 | reg1    │
└────────┴────────┴────────────────────┴────────────────────┘
```

### Formato registro + imm8 (GETFIELD, GETMETHOD)

```
┌────────┬────────┬──────────────────┬──────────────────────┐
│ 0x00   │ opcode │  reg_cls & 0x0F  │  field/method_idx    │
└────────┴────────┴──────────────────┴──────────────────────┘
  byte 0   byte 1       byte 2               byte 3 (imm8)
```

---

Ver también: [[NEWOBJRAW y NEWOBJ]], [[CALLVIRT y CALLSUPER]], [[THROW y RETHROW]], [[cursor]], [[GCALLOC]]
