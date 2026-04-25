# GCALLOC - Asignacion GC de tamano arbitrario

`GCALLOC` reserva un bloque de memoria de tamano arbitrario en el GC heap **sin escribir
ningun `ObjectHeader`**. Es la contraparte "cruda" de `NEWOBJ`: el GC gestiona el ciclo
de vida del bloque automaticamente, pero el contenido es completamente libre. No hay
estructura OOP implicita: puedes usar el bloque como un array, un buffer de bytes, o
cualquier estructura de datos que quieras definir tu mismo.

> **Ver tambien:** [[Generacional (para objetos OOP)|GcHeap]], [[NEWOBJRAW y NEWOBJ]],
> [[Allocator crudo para FFI y memoria manual|RawAllocator]]

---

## Para que sirve

Usa `GCALLOC` cuando necesitas:
- **Buffers temporales con ciclo de vida GC**: datos que deben vivir mientras el proceso
  los use pero sin necesidad de ser un objeto OOP (por ejemplo, una cadena de bytes o un
  buffer de serializacion).
- **Arrays**: colecciones de elementos del mismo tipo sin un ObjectHeader.
- **Memoria gestionada automaticamente sin clase**: cuando aun no tienes la `ClassInfo`
  disponible pero quieres que el GC controle la vida del bloque.

Para objetos con clase y dispatch virtual, usa `NEWOBJ`. Para memoria completamente fuera
del GC, usa `ALLOC`.

---

## Instruccion

```c
gcalloc  reg_size    // R0 = GcHandle (o GC_NULL_HANDLE = 0xFFFFFFFF si falla)
```

| Campo      | Valor                                                                |
| :--------: | :------------------------------------------------------------------- |
| Opcode1    | `0x00`                                                               |
| Opcode2    | `0xA5`                                                               | 
| Tamano     | 4 bytes (FIXED_4)                                                    |
| `reg_size` | registro con el numero de bytes a reservar                           |
| Resultado  | `R0` = `GcHandle` (o `GC_NULL_HANDLE = 0xFFFFFFFF` si falla)        |

### Que hace exactamente

1. Lee `size = regs[reg_size]`.
2. Llama a `GcHeap::alloc(size)`.
3. **No escribe** ningun `ObjectHeader` ni metadatos en el bloque.
4. Devuelve el `GcHandle` en `R0`.

El contenido del bloque **no esta inicializado**: puede contener datos arbitrarios del heap
anterior. Inicializa siempre antes de leer.

---

## Diferencias respecto a `NEWOBJ` y `ALLOC`

| Criterio             | `NEWOBJ`                       | `GCALLOC`                       | `ALLOC`                      |
| :------------------- | :----------------------------- | :------------------------------ | :--------------------------- |
| Gestor               | GC generacional                | GC generacional                 | Raw allocator (malloc)       |
| Ciclo de vida        | Automatico (GC)                | Automatico (GC)                 | Manual (`FREE`)              |
| Resultado            | `GcHandle`                     | `GcHandle`                      | Host pointer                 |
| Requiere ClassInfo   | Si (`instance_size`)           | **No** (tamano libre)           | No                           |
| Escribe ObjectHeader | Si (class_ptr, flags, hash...) | **No**                          | No                           |
| Caso de uso          | Objetos OOP con clase          | Buffers, arrays, datos genericos| FFI, objetos raw sin GC      |

---

## Uso tipico

### Buffer de bytes bajo GC

```c
// Reservar 256 bytes bajo GC
mov     r1, 256
gcalloc r1              // R0 = GcHandle

// Obtener puntero al payload para leer/escribir:
gcderef cur0, r0        // cur0 = puntero host al inicio del bloque

// Escribir datos en el buffer:
mov     r5, 0xDEADBEEF
writecur cur0, r5       // bloque[0..7] = 0xDEADBEEF

// El GC libera el bloque automaticamente cuando no quedan handles que lo referencien.
// Para liberar explicitamente (opcional):
drop    r0
```

### Array de enteros bajo GC (8 elementos de 8 bytes cada uno = 64 bytes)

```c
mov     r1, 64
gcalloc r1              // R0 = handle del array
mov     r8, r0          // guardar el handle en r8

gcderef cur0, r0        // cur0 = puntero al inicio del array

// Escribir elemento[0] = 42:
mov     r5, 42
writecur cur0, r5       // array[0] = 42

// Para elemento[1] necesitamos avanzar el cursor 8 bytes.
// Como los cursores no tienen un "avanzar", usamos un registro auxiliar:
gcderef cur0, r8        // re-obtener puntero al inicio
mov     r14, cur0       // r14 = puntero al inicio
// (problema: no podemos usar cur0 como reg general directamente)
// La forma correcta es re-hacer el gcderef con offset manual:

// Para acceder a array[1] (offset +8), necesitamos usar la instruccion de cursor
// con offset relativo. Esto requiere instrucciones MOVC o acceso via host ptr.

// Leer elemento[0] de vuelta:
gcderef cur1, r8        // IMPORTANTE: re-deref por si el GC movio el objeto
readcur r9, cur1        // r9 = 42
```

> **ADVERTENCIA CRITICA**: El puntero obtenido con `GCDEREF` puede **invalidarse** tras
> cualquier ciclo de GC. Siempre llama a `GCDEREF` de nuevo despues de cualquier
> asignacion (`NEWOBJ`, `GCALLOC`) que pueda disparar un minor GC.

### Grafo de buffers GC (GCALLOC + GCWB)

```c
// Buffer padre (16 bytes: espacio para 2 handles de 8 bytes)
mov     r1, 16
gcalloc r1
mov     r8, r0          // r8 = handle del padre

// Buffer hijo (8 bytes)
mov     r1, 8
gcalloc r1
mov     r9, r0          // r9 = handle del hijo

// Guardar el handle del hijo en el slot 0 del padre:
gcderef cur0, r8        // cur0 = inicio del padre
writecur cur0, r9       // padre.slot[0] = handle_hijo

// WRITE BARRIER OBLIGATORIO:
// Si el padre esta en OldGen y el hijo esta en Nursery, el GC no veria esta
// referencia durante un minor GC a menos que la registremos explicitamente.
gcwb r8                 // notificar al GC: padre referencia a hijo

// Ahora podemos soltar el handle directo del hijo:
// el padre lo mantiene vivo a traves de su slot[0]
drop r9

// Despues de un GC, el hijo sigue vivo (el padre lo referencia):
gcrun
// r8 sigue siendo un handle valido; el hijo esta vivo
```

---

## La write barrier (gcwb) explicada

Cuando el heap tiene objetos en dos zonas (Nursery y OldGen), el minor GC solo analiza
la Nursery. Si un objeto en OldGen referencia a uno en Nursery, el GC no lo vera durante
un minor GC y podria liberar el objeto joven pensando que nadie lo referencia.

La **write barrier** (`GCWB`) resuelve esto: cuando escribes un GcHandle de un objeto
joven dentro del payload de un objeto viejo, debes llamar a `gcwb` con el handle del
objeto viejo. El GC lo agrega al **remembered set** y lo escanea durante el proximo
minor GC.

Regla: **siempre que escribas un GcHandle en el payload de otro objeto GC, llama a gcwb**.

---

## Codificacion binaria

```
+--------+--------+--------------------+--------------------+
| 0x00   | 0xA5   |  mode<<6 | 0x00   |  0000 | reg_size   |
+--------+--------+--------------------+--------------------+
  byte0    byte1       byte2                 byte3
```

- `mode` (2 bits) = tamano de lectura del registro: `00`=byte, `01`=word, `10`=dword, `11`=qword
- `reg_size` (4 bits) = indice del registro con el tamano en bytes a reservar

---

## Resumen de advertencias

| Situacion                                      | Que hacer                          |
| :--------------------------------------------- | :--------------------------------- |
| R0 == 0xFFFFFFFF despues de gcalloc            | Sin memoria; manejar el error      |
| Despues de cualquier asignacion GC             | Re-hacer gcderef antes de acceder  |
| Escribir un handle joven en payload de un viejo| Llamar gcwb con el handle padre    |
| Bloque no inicializado                         | Escribir antes de leer             |

---

Ver tambien: [[Generacional (para objetos OOP)|GcHeap]], [[NEWOBJRAW y NEWOBJ]], [[Allocator crudo para FFI y memoria manual|RawAllocator]], [[cursor]], [[GC]]
