# GC Generacional — GcHeap

Gestiona objetos del lenguaje con ciclo de vida automático. Cada proceso VM tiene su propio `GcHeap`, lo que elimina pausas inter-proceso y contención de bloqueos.

> Si no conoces qué es un GC ni por qué se necesita, empieza por [[GC]] antes de leer esta página.

---

## ¿Cómo sabe el GC qué está "vivo"?

Un objeto está **vivo** si existe alguna forma de llegar a él desde el código que se está ejecutando. Esa "forma de llegar" se llama **raíz**: normalmente un registro de la CPU virtual, una variable local en la pila, o un campo de otro objeto ya vivo.

```
Registros del proceso (R0..R15)
  └── handle 3  →  Objeto A  →  (campo)  →  handle 7  →  Objeto B
                                                              └── (campo)  →  handle 12  →  Objeto C

Objeto D  (no hay ninguna ruta desde los registros hasta él)  →  BASURA
```

El GC recorre el grafo de referencias partiendo de las raíces. Todo lo que no se puede alcanzar es basura y su memoria se puede reutilizar.

---

## ¿Por qué dos generaciones?

Repasar todo el heap cada vez que hay que limpiar es caro, especialmente si el heap es grande. La **hipótesis generacional** dice que la mayoría de los objetos mueren pronto, así que podemos limpiar solo la zona de objetos jóvenes (que suele estar casi vacía de supervivientes) muy rápido, y dejar la zona de objetos viejos para revisiones mucho menos frecuentes.

```
Ejemplo con 10 000 objetos creados:
  - 8 000 mueren antes del primer GC  →  limpiar Nursery es baratísimo
  -   800 sobreviven y van a OldGen
  -    50 siguen vivos 10 ciclos despues
```

Limpiar esos 8 000 objetos muertos junto con los 800 vivos costaría revisar los 10 000. Limpiar solo la Nursery (1 200 objetos vivos + 8 000 muertos, todos en un espacio pequeño) cuesta mucho menos.

---

## Arquitectura interna

```
ProcessVM
  └── GcHeap
        ├── Nursery  (bump-pointer, tamano fijo, ej. 2 MB)
        │     └── objetos jovenes  ->  GcHandle
        │
        ├── OldGen   (bloques de arena, mark-and-sweep)
        │     └── objetos promovidos  ->  GcHandle
        │
        └── HandleTable[]
              GcHandle  ->  direccion actual del objeto
```

### Nursery y la asignación por bump-pointer

La Nursery es una zona de tamaño fijo. Asignar un objeto es tan sencillo como avanzar un puntero (`bump`) en N bytes. No hay que buscar huecos libres: O(1) garantizado.

```
Antes:  [LLENO][LLENO][bump→           ][ libre...     ][ fin ]
         obj1   obj2                      ...

Despues NEWOBJ 64:
        [LLENO][LLENO][obj3 64B][bump→  ][ libre...     ][ fin ]
```

Cuando la Nursery se llena, se lanza el **minor GC**. Los objetos vivos se copian a OldGen y el bump pointer vuelve al inicio: toda la Nursery queda libre de golpe, sin necesidad de rastrear individualmente qué está libre y qué no.

### OldGen y el mark-and-sweep

OldGen crece dinámicamente en bloques de arena. No usa bump-pointer porque los objetos tienen distintos tamaños y tiempos de vida. En su lugar usa el algoritmo **mark-and-sweep**:

1. Marcar todos los objetos alcanzables.
2. Barrer (sweep): los no marcados son basura y se liberan.

OldGen solo se limpia cuando su ocupación supera el umbral configurado con `GCCONFIG`.

---

## Cabecera de objeto (`GcHeader`)

Cada objeto en heap lleva una cabecera de 8 bytes antes del payload de datos:

```
Offset 0                             Offset 8
+--------+-------+-----+------+----------+
| size   | color | gen | _pad | reserved |
| 32 bit | 2 bit | 1b  | 5 b  |  3 bytes |
+--------+-------+-----+------+----------+
         ^       ^
         |       generacion: YOUNG o OLD
         estado tri-color (ver abajo)
```

- `size` — bytes del payload (sin contar la cabecera). El escáner lo usa para saltar al siguiente objeto en el bloque.
- `color` — estado del objeto durante el GC (ver más abajo).
- `gen` — en qué generación vive el objeto.

El payload (los datos reales del objeto) viene justo después de la cabecera. Cuando el bytecode llama a `NEWOBJ`, el runtime devuelve un `GcHandle`; cuando accede al objeto, el runtime calcula `dirección_header + 8` para llegar al payload.

---

## Tri-color mark — cómo el GC distingue vivos de muertos

El algoritmo de marcado usa cuatro estados (codificados en 2 bits de la cabecera):

| Color   | Valor | Significado                                              |
| :------ | :---: | :------------------------------------------------------- |
| `WHITE` |   0   | No alcanzado en este ciclo — candidato a liberar         |
| `GREY`  |   1   | Alcanzado, pero sus hijos aún no han sido inspeccionados |
| `BLACK` |   2   | Alcanzado y todos sus hijos inspeccionados — está vivo   |
| `DEAD`  |   3   | Slot liberado por sweep; el espacio es reutilizable      |

El estado `DEAD` es especial: cuando el sweep libera un objeto, preserva el campo `size` y solo cambia el color a `DEAD`. Así el escáner de bloques puede seguir avanzando correctamente (sabe cuánto mide el slot) y `alloc_in_old` puede reutilizar ese hueco si el próximo objeto que llega cabe en él.

### Por qué es necesario el paso PRE-MARK

Los objetos llegan a OldGen con color `BLACK` (recién copiados del minor GC). Si el handle es liberado con `DROP`, el objeto pierde su raíz pero sigue con `BLACK`. Sin un paso previo de reset, el SWEEP lo vería `BLACK` y pensaría que está vivo — nunca se liberaría.

La solución es que **antes de cada major GC** todos los objetos no-`DEAD` de OldGen se restablecen a `WHITE`. Así el MARK solo pone `BLACK` los que realmente tienen raíz, y el SWEEP puede liberar el resto.

---

## ¿Qué es un handle y por qué no se usan punteros directos?

Cuando se mueve un objeto (por ejemplo, al copiarlo de Nursery a OldGen), su dirección en memoria cambia. Si el bytecode guardase la dirección directa y el GC moviera el objeto, esa dirección quedaría inválida.

La solución es que el bytecode nunca vea la dirección real. En su lugar recibe un **handle** (un entero de 32 bits, índice en la `HandleTable`). Cuando el GC mueve un objeto, solo actualiza la entrada correspondiente en la tabla. Todo el bytecode que tenga ese handle sigue funcionando sin cambios.

```
Antes del minor GC:
  Handle 5  →  HandleTable[5] = 0x10000020  (en Nursery)

Despues del minor GC (el objeto se copio a OldGen):
  Handle 5  →  HandleTable[5] = 0x30000080  (en OldGen)
  (el bytecode no sabe ni que paso nada)
```

`GC_NULL_HANDLE = 0xFFFFFFFF` es el handle inválido. Si `NEWOBJ` retorna este valor, la asignación falló (sin memoria disponible).

---

## Ciclo de vida completo de un objeto

```
NEWOBJ size
   |
   |-- espacio en Nursery? --> asignar en Nursery (bump, O(1))
   |
   |-- Nursery llena --> minor_gc()
   |     |
   |     |- objeto vivo? --> evacuate_object() --> copiar a OldGen (color=BLACK)
   |     |- objeto muerto? --> abandonado; bump se resetea a base (O(1))
   |     |
   |     `- old_used >= old_threshold? --> major_gc()
   |           |
   |           |- PRE-MARK: todos los objetos no-DEAD --> WHITE
   |           |- MARK:     handles vivos en OldGen --> BLACK
   |           `- SWEEP:    WHITE --> DEAD, old_used--, stats++
   |
   `-- sigue sin espacio --> asignar directamente en OldGen

DROP handle
   |
   `-> HandleTable[handle] = {nullptr, false}
       (el slot fisico en OldGen lo recoge el siguiente major_gc)
```

---

## Minor GC — Cheney-style copy (en detalle)

El minor GC escanea todos los handles vivos cuya dirección cae dentro de la Nursery:

1. Por cada objeto joven (`gen == YOUNG`): llama a `evacuate_object(handle)`.
2. `evacuate_object` copia el objeto completo (cabecera + payload) a OldGen, actualiza el handle, y deja un **forward pointer** en el payload original para detectar doble evacuación.
3. Al terminar, el bump pointer se resetea a la base de la Nursery — toda la Nursery queda libre instantáneamente.
4. Si tras la evacuación `old_used >= old_threshold`, se lanza `major_gc`.

> **Pausa:** proporcional al número de objetos **vivos** en Nursery, no al tamaño total. Cuanto más corta sea la vida media de los objetos, menor es la pausa.

---

## Major GC — mark-and-sweep (en detalle)

Tres fases consecutivas sobre OldGen:

**Fase 1 — PRE-MARK**

Recorre todos los bloques de OldGen y pone `WHITE` a todos los objetos que no estén `DEAD`. Esto es necesario porque los objetos llegan con `BLACK` y los handles soltados con `DROP` no tienen mecanismo de reset automático.

**Fase 2 — MARK**

Recorre la `HandleTable`. Por cada handle vivo (`live == true`) cuya dirección cae en OldGen, pone el objeto a `BLACK`. Estos son los objetos alcanzables.

**Fase 3 — SWEEP**

Recorre de nuevo los bloques de OldGen. Cada objeto `WHITE` (alcanzable en el MARK anterior = nadie lo tiene) se marca `DEAD`, se decrementa `old_used_` y se actualizan las estadísticas. Los objetos `BLACK` se dejan intactos. Los objetos `DEAD` anteriores también se dejan: el escáner los salta usando su campo `size`.

Los slots `DEAD` pueden ser reutilizados por `alloc_in_old` si el próximo objeto que llega cabe en el hueco.

---

## Write barrier — referencias entre generaciones

Un problema especial de los GC generacionales: ¿qué pasa si un objeto de OldGen tiene un campo que apunta a un objeto de la Nursery?

```
OldGen: Objeto A  -->  campo  -->  Handle joven B (en Nursery)
```

Cuando el minor GC escanea la Nursery, parte de los registros del proceso como raíces. Pero si nadie en los registros apunta a B directamente, el minor GC pensaría que B está muerto — aunque A (en OldGen) lo tiene referenciado.

La solución es el **remembered set**: una lista de handles de OldGen que contienen referencias a objetos jóvenes. El minor GC también los trata como raíces.

El **write barrier** es el mecanismo que mantiene actualizado el remembered set: cada vez que el bytecode escribe un handle joven en un campo de un objeto viejo, el ejecutor llama automáticamente a `write_barrier(handle_viejo)`.

```vesta
; El ejecutor llama write_barrier(handle_A) automaticamente
STFIELD handle_A, campo_idx, handle_B_joven
```

---

## Instrucciones bytecode

| Instrucción          | Opcode1 | Opcode2 | Descripción                                         |
| :------------------- | :-----: | :-----: | :-------------------------------------------------- |
| `NEWOBJ size`        |  `0x00` |  `0xA0` | Reserva `size` bytes; retorna `GcHandle` en `R0`    |
| `DROP handle`        |  `0x00` |  `0xA3` | Libera el handle; el objeto es recogido en major GC |
| `GCRUN`              |  `0x00` |  `0xA1` | Dispara minor + (si procede) major GC manualmente   |
| `GCCONFIG threshold` |  `0x00` |  `0xA2` | Configura el umbral de OldGen para major GC         |

### Codificación — NEWOBJ

| opcode1 | opcode2 | bytes 3-6 (size, 32 bits) | total |
| :-----: | :-----: | :-----------------------: | :---: |
|  `0x00` |  `0xA0` |       `0xFFFFFFFF`        |   6   |

Retorna el `GcHandle` en `R0`. Si la asignación falla, retorna `GC_NULL_HANDLE = 0xFFFFFFFF`.

### Codificación — DROP

| opcode1 | opcode2 |      byte3      | total |
| :-----: | :-----: | :-------------: | :---: |
|  `0x00` |  `0xA3` | `0b0000`\|`reg` |   3   |

El registro indicado contiene el `GcHandle` a liberar. Tras `DROP`, el handle es inválido.

### Codificación — GCRUN

| opcode1 | opcode2 | total |
| :-----: | :-----: | :---: |
|  `0x00` |  `0xA1` |   2   |

Sin operandos. Ejecuta el GC del proceso actual de forma síncrona.

### Codificación — GCCONFIG

| opcode1 | opcode2 | bytes 3-10 (threshold, 64 bits) | total |
| :-----: | :-----: | :-----------------------------: | :---: |
|  `0x00` |  `0xA2` |    `0xFFFFFFFFFFFFFFFF`         |  10   |

El threshold es el número de bytes de OldGen a partir del cual se dispara el major GC automáticamente tras cada minor GC.

---

## Ejemplos en bytecode Vesta

```vesta
; Crear un objeto de 64 bytes
NEWOBJ 64          ; R0 = handle (ej. 3)
mov    r1, r0      ; guardar handle en r1

; Disparar GC manualmente cuando el programador sabe que hay mucha basura
GCRUN

; Cambiar umbral a 8 MB antes de una rafaga de asignaciones
GCCONFIG 8388608   ; 8 * 1024 * 1024

; Liberar el objeto (el GC lo recogerá en el siguiente major GC)
DROP r1
```

```vesta
; Patron: crear muchos objetos temporales (la Nursery los recoge solos)
loop_inicio:
    NEWOBJ 32       ; objeto temporal de 32 bytes
    mov    r2, r0
    ; ... usar r2 ...
    ; no hace falta DROP: si r2 se sobreescribe, el objeto muere en el proximo minor GC
    jmp loop_inicio
```

---

## Estadísticas disponibles

Accesibles en C++ via `GcHeap::stats()` (tipo `GcStats`):

| Campo              | Descripción                                       |
| :----------------- | :------------------------------------------------ |
| `alloc_count`      | Llamadas exitosas a `alloc()`                     |
| `alloc_bytes`      | Bytes totales asignados                           |
| `freed_count`      | Objetos recogidos por sweep                       |
| `freed_bytes`      | Bytes recuperados por sweep                       |
| `promoted_count`   | Objetos evacuados de Nursery a OldGen             |
| `promoted_bytes`   | Bytes promovidos                                  |
| `minor_gc_count`   | Veces que se ejecutó minor GC                     |
| `major_gc_count`   | Veces que se ejecutó major GC                     |
| `peak_nursery`     | Máximo de bytes vivos en Nursery en cualquier momento |
| `peak_old`         | Máximo de bytes vivos en OldGen en cualquier momento  |

Las estadísticas son acumulativas desde que el proceso arranca. Son útiles para diagnosticar presión de memoria, ajustar el threshold de OldGen y verificar que la hipótesis generacional se cumple en el programa concreto (si `promoted_count / alloc_count` es bajo, los objetos mueren jóvenes como se espera).

---

## Seguridad de hilos

`GcHeap` es **per-proceso** y no thread-safe. La sincronización la aporta el planificador (`Scheduler`): un `ProcessVM` solo corre en un hilo nativo a la vez, por lo que el GC nunca compite con el código de usuario del mismo proceso.

Ver también: [[GC]], [[Allocator crudo para FFI  y memoria manual|RawAllocator]], [[VmInstance]], [[VmManager]]
