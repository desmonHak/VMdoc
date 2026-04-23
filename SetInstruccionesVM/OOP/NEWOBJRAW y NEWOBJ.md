# NEWOBJ y NEWOBJRAW - Creación de objetos OOP

Estas dos instrucciones asignan memoria para objetos OOP y escriben el `ObjectHeader` al inicio del bloque. La diferencia clave está en el gestor de memoria que usan y en el tipo de puntero que devuelven.

> **Ver también:** [[GCALLOC]] para asignación GC sin estructura OOP, [[Generacional (para objetos OOP)|GcHeap]] para el ciclo de vida del GC.

---

## `NEWOBJ` - objeto bajo GC generacional

```asm
newobj  reg_classinfo    ; R0 = GcHandle
```

| Campo     | Valor              |
| :-------: | :----------------- |
| Opcode1   | `0x00`             |
| Opcode2   | `0xA0`             |
| Tamaño    | 4 bytes (FIXED_4)  |
| Resultado | `R0` = `GcHandle` |

`reg_classinfo` contiene un **puntero host** a un `ClassInfo` cargado por el loader. La instrucción:

1. Lee `classinfo->instance_size` para determinar cuántos bytes reservar.
2. Llama a `GcHeap::alloc(instance_size)`.
3. Si la asignación tiene éxito, escribe el `ObjectHeader` al inicio del payload:
   - `class_ptr` = el `ClassInfo*` que se pasó.
   - `flags`     = `OBJ_FLAG_GC_OWNED` (bit 0 = 1).
   - `hash_code` = los 32 bits bajos del `GcHandle`.
4. Devuelve el `GcHandle` en `R0`.

Si `reg_classinfo` es cero o la asignación falla, `R0` = `GC_NULL_HANDLE` (`0xFFFFFFFF`).

### Uso típico

```asm
; cls_ptr está en R1 (host ptr a ClassInfo, normalmente cargado por el loader)
newobj  r1              ; R0 = GcHandle al nuevo objeto

; Para acceder al payload (ObjectHeader + campos del objeto):
gcderef cur0, r0        ; cur0 = puntero host al ObjectHeader

; Leer class_ptr del ObjectHeader (offset 0):
readcur r2, cur0        ; r2 = ClassInfo*
```

### ObjectHeader escrito

```
Payload GC (después de GcHeader de 8 bytes interno):
  offset 0  [8B] class_ptr  = ClassInfo*
  offset 8  [4B] flags      = OBJ_FLAG_GC_OWNED (0x00000001)
  offset 12 [4B] hash_code  = GcHandle & 0xFFFFFFFF
```

---

## `NEWOBJRAW` - objeto bajo allocator crudo (sin GC)

```asm
newobjraw  reg_classinfo, reg_size    ; R0 = host_ptr
```

| Campo     | Valor              |
| :-------: | :----------------- |
| Opcode1   | `0x00`             |
| Opcode2   | `0xD0`             |
| Tamaño    | 4 bytes (FIXED_4)  |
| Resultado | `R0` = host pointer al `ObjectHeader` |

`reg_classinfo` = puntero host a `ClassInfo`.  
`reg_size` = número de bytes a reservar (si es `R0`, se usa `classinfo->instance_size`).

La instrucción:
1. Si `reg_classinfo` es cero: `R0 = 0`, sin asignación.
2. Si `reg_size != R00`: usa el valor del registro como tamaño.
3. Si `reg_size == R00` (índice cero): usa `classinfo->instance_size`.
4. Reserva memoria con `RawAllocator::alloc(size)`.
5. Escribe el `ObjectHeader`:
   - `class_ptr` = `ClassInfo*`
   - `flags`     = `OBJ_FLAG_RAW_OWNED` (bit 1 = 2)
   - `hash_code` = los 32 bits bajos del puntero host
6. Devuelve el puntero host en `R0`.

El objeto resultante **no está bajo GC**: debe liberarse manualmente con `FREE`.

### Uso típico

```asm
; Crear objeto con tamaño explícito
mov       r1, my_class_ptr   ; ClassInfo host ptr
mov       r2, 64             ; tamaño total (ObjectHeader 16B + 48B payload)
newobjraw r1, r2             ; R0 = host ptr al ObjectHeader

; Acceder a campos del objeto via cursor:
xchg  cur0, r0               ; cur0 = inicio del ObjectHeader
readcur r3, cur0             ; r3 = class_ptr (offset 0)

; Acceder a payload (offset 16, después del ObjectHeader):
mov   r14, r0
addu  r14, 16
xchg  cur1, r14              ; cur1 = inicio del payload de usuario

; Liberar cuando ya no se necesite:
free  r0
```

### Cuándo usar NEWOBJ vs NEWOBJRAW

| Criterio              | `NEWOBJ`                         | `NEWOBJRAW`                        |
| :-------------------- | :------------------------------- | :--------------------------------- |
| Ciclo de vida         | Gestionado por GC                | Manual (`FREE`)                    |
| Resultado             | `GcHandle` (indirecto)           | Host pointer (directo)             |
| Acceso al payload     | `GCDEREF` -> cursor               | Cursor directo                     |
| Flag en ObjectHeader  | `OBJ_FLAG_GC_OWNED` (1)          | `OBJ_FLAG_RAW_OWNED` (2)           |
| Uso recomendado       | Objetos de usuario normales      | FFI, objetos de vida muy corta     |

---

## `ObjectHeader` - layout de cabecera

Todos los objetos OOP (tanto GC como raw) tienen los primeros 16 bytes en este formato:

```
Offset 0                                Offset 16
+─────────────────────────┬────────┬──────────+
│   class_ptr  (8 bytes)  │ flags  │ hash_code│
│   ClassInfo host ptr    │ 4 B    │ 4 B      │
+─────────────────────────┴────────┴──────────+
```

| Campo       | Tipo        | Descripción                                        |
| :---------- | :---------- | :------------------------------------------------- |
| `class_ptr` | `ClassInfo*`| Puntero host a la clase del objeto                 |
| `flags`     | `uint32_t`  | Bitfield: bit 0 = GC_OWNED, bit 1 = RAW_OWNED      |
| `hash_code` | `uint32_t`  | Hash inicial (handle o puntero truncado a 32 bits) |

Los flags definidos son:

| Constante            | Valor | Significado                                   |
| :------------------- | :---: | :-------------------------------------------- |
| `OBJ_FLAG_GC_OWNED`  |  `1`  | Objeto gestionado por GC (`NEWOBJ`)            |
| `OBJ_FLAG_RAW_OWNED` |  `2`  | Objeto asignado con raw allocator (`NEWOBJRAW`)|
| `OBJ_FLAG_LOCKED`    |  `4`  | Monitor del objeto adquirido (reservado)       |

---

## Codificación binaria

### NEWOBJ

```
┌────────┬────────┬────────────────────┬────────────────────┐
│ 0x00   │ 0xA0   │  mode<<6 | 0x00    │  0000 | reg_cls    │
└────────┴────────┴────────────────────┴────────────────────┘
```

### NEWOBJRAW

```
┌────────┬────────┬────────────────────┬────────────────────┐
│ 0x00   │ 0xD0   │  mode<<6 | 0x00    │  reg_sz<<4 | reg_cls│
└────────┴────────┴────────────────────┴────────────────────┘
```

- `reg_cls` (4 bits) = índice del registro con `ClassInfo*`
- `reg_sz`  (4 bits) = índice del registro con el tamaño; `0` = usar `instance_size`

---

Ver también: [[GCALLOC]], [[Generacional (para objetos OOP)|GcHeap]], [[Allocator crudo para FFI  y memoria manual|RawAllocator]], [[cursor]], [[GETCLASS]], [[CALLVIRT y CALLSUPER]]
