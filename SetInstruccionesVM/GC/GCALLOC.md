# GCALLOC - Asignación GC de tamaño arbitrario

`GCALLOC` reserva un bloque de memoria de tamaño arbitrario en el GC heap **sin escribir ningún `ObjectHeader`**. Es la contraparte "cruda" de `NEWOBJ`: el GC gestiona el ciclo de vida del bloque (incluidos minor y major GC, evacuación y liberación), pero el contenido es completamente libre -no hay estructura OOP implícita.

> **Ver también:** [[Generacional (para objetos OOP)|GcHeap]], [[NEWOBJ y NEWOBJRAW]], [[Allocator crudo para FFI  y memoria manual|RawAllocator]]

---

## Instrucción

```asm
gcalloc  reg_size    ; R0 = GcHandle
```

| Campo      | Valor              |
| :--------: | :----------------- |
| Opcode1    | `0x00`             |
| Opcode2    | `0xA5`             |
| Tamaño     | 4 bytes (FIXED_4)  |
| `reg_size` | registro con el número de bytes a reservar |
| Resultado  | `R0` = `GcHandle` (o `GC_NULL_HANDLE = 0xFFFFFFFF` si falla) |

La instrucción:
1. Lee `size = regs[reg_size]`.
2. Llama a `GcHeap::alloc(size)`.
3. **No escribe** ningún `ObjectHeader` ni metadatos en el bloque.
4. Devuelve el `GcHandle` en `R0`.

---

## Diferencias respecto a `NEWOBJ` y `ALLOC`

| Criterio            | `NEWOBJ`                           | `GCALLOC`                       | `ALLOC`                        |
| :------------------ | :--------------------------------- | :------------------------------ | :----------------------------- |
| Gestor              | GC generacional                    | GC generacional                 | Raw allocator (malloc)         |
| Ciclo de vida       | Automático (GC)                    | Automático (GC)                 | Manual (`FREE`)                |
| Resultado           | `GcHandle`                         | `GcHandle`                      | Host pointer                   |
| Requiere ClassInfo  | Sí (`instance_size`)               | **No** (tamaño libre)           | No                             |
| Escribe ObjectHeader| Sí (class_ptr, flags, hash)        | **No**                          | No                             |
| Caso de uso         | Objetos OOP con clase              | Buffers, arrays, datos genéricos| FFI, objetos raw sin GC        |

---

## Cuándo usar GCALLOC

- **Buffers temporales con vida GC**: datos que deben vivir mientras el proceso los use pero sin necesidad de ser un objeto OOP (p.ej. una cadena de bytes, un buffer de serialización).
- **Arrays**: colecciones de elementos del mismo tipo sin campo de clase.
- **Memoria compartida entre procesos GC-aware**: un bloque que debe ser rastreado por el GC pero cuya estructura interna la gestiona el bytecode.
- **Prototipado**: cuando aún no se tiene la `ClassInfo` disponible pero se quiere el ciclo de vida del GC.

Para datos que necesitan clase y dispatch virtual, usa `NEWOBJ`. Para datos completamente fuera del GC, usa `ALLOC`.

---

## Uso típico

### Buffer de bytes bajo GC

```asm
; Reservar 256 bytes bajo GC
mov     r1, 256
gcalloc r1              ; R0 = GcHandle

; Obtener puntero al payload
gcderef cur0, r0        ; cur0 = puntero host al inicio del bloque

; Escribir datos
mov     r5, 0xDEADBEEF
writecur cur0, r5       ; bloque[0..7] = 0xDEADBEEF

; El GC libera el bloque automáticamente cuando no quedan handles vivos.
; Para liberar explícitamente:
drop    r0
```

### Array de enteros bajo GC (8 elementos × 8 bytes = 64 bytes)

```asm
mov     r1, 64
gcalloc r1              ; R0 = handle del array

gcderef cur0, r0
mov     r8, r0          ; guardar handle

; Escribir elemento[0] = 42
mov     r5, 42
writecur cur0, r5

; Escribir elemento[1] = 100 (avanzar cursor 8 bytes)
xchg  r14, cur0 ; addu r14, 8 ; xchg cur0, r14
mov     r5, 100
writecur cur0, r5

; ...

; Leer elemento[0] de vuelta
gcderef cur1, r8        ; re-deref por si el GC movio el objeto
readcur r9, cur1        ; r9 = 42
```

### Grafo de buffers GC (GCALLOC + GCWB)

```asm
; Buffer padre (16 bytes)
mov     r1, 16
gcalloc r1
mov     r8, r0

; Buffer hijo (8 bytes)
mov     r1, 8
gcalloc r1
mov     r9, r0

; Guardar handle del hijo en el payload del padre (offset 0)
gcderef cur0, r8
writecur cur0, r9       ; padre.slot[0] = handle_hijo

; GCWB obligatorio: padre puede ser OLD, hijo puede ser YOUNG
gcwb r8

; Soltar el handle directo del hijo; el padre lo mantiene vivo
drop r9
gcrun                   ; el hijo sobrevive por ser alcanzable desde el padre
```

---

## Codificación binaria

```
┌────────┬────────┬────────────────────┬────────────────────┐
│ 0x00   │ 0xA5   │  mode<<6 | 0x00    │  0000 | reg_size   │
└────────┴────────┴────────────────────┴────────────────────┘
```

- `mode` (2 bits) = tamaño de lectura del registro: `00`=byte, `01`=word, `10`=dword, `11`=qword
- `reg_size` (4 bits) = índice del registro con el tamaño en bytes

---

## Advertencias

- El puntero obtenido con `GCDEREF` puede **invalidarse** tras cualquier ciclo de GC. Llama a `GCDEREF` de nuevo después de cualquier asignación (`NEWOBJ`, `GCALLOC`) que pueda disparar un minor GC.
- Si `R0 == GC_NULL_HANDLE (0xFFFFFFFF)`: la asignación falló (sin memoria disponible).
- El contenido del bloque recién asignado **no está inicializado a cero**: puede contener basura del heap.

---

Ver también: [[Generacional (para objetos OOP)|GcHeap]], [[NEWOBJ y NEWOBJRAW]], [[Allocator crudo para FFI  y memoria manual|RawAllocator]], [[cursor]], [[GC]]
