# GC Generacional - GcHeap

Gestiona objetos del lenguaje con ciclo de vida automático. Cada proceso VM tiene su propio `GcHeap`, lo que elimina pausas inter-proceso y contención de bloqueos.

> Si no conoces qué es un GC ni por qué se necesita, empieza por [[GC]] antes de leer esta página.

---

## ¿Cómo sabe el GC qué está "vivo"?

Un objeto está **vivo** si existe alguna forma de llegar a él desde el código que se está ejecutando. Esa "forma de llegar" se llama **raíz**: normalmente un registro de la CPU virtual, una variable local en la pila, o un campo de otro objeto ya vivo.

```
Registros del proceso (R0..R15)
  └── handle 3  ->  Objeto A  ->  (campo)  ->  handle 7  ->  Objeto B
                                                              └── (campo)  ->  handle 12  ->  Objeto C

Objeto D  (no hay ninguna ruta desde los registros hasta él)  ->  BASURA
```

El GC recorre el grafo de referencias partiendo de las raíces. Todo lo que no se puede alcanzar es basura y su memoria se puede reutilizar.

> **Importante:** un objeto sigue vivo aunque su handle haya sido liberado con `DROP`, siempre que otro objeto vivo tenga su handle embebido en el payload. El GC hace un recorrido transitivo del grafo, no solo mira los handles directos.

---

## ¿Por qué dos generaciones?

Repasar todo el heap cada vez que hay que limpiar es caro, especialmente si el heap es grande. La **hipótesis generacional** dice que la mayoría de los objetos mueren pronto, así que podemos limpiar solo la zona de objetos jóvenes (que suele estar casi vacía de supervivientes) muy rápido, y dejar la zona de objetos viejos para revisiones mucho menos frecuentes.

```
Ejemplo con 10 000 objetos creados:
  - 8 000 mueren antes del primer GC  ->  limpiar Nursery es baratísimo
  -   800 sobreviven y van a OldGen
  -    50 siguen vivos 10 ciclos después
```

Limpiar esos 8 000 objetos muertos junto con los 800 vivos costaría revisar los 10 000. Limpiar solo la Nursery (1 200 objetos vivos + 8 000 muertos, todos en un espacio pequeño) cuesta mucho menos.

---

## Arquitectura interna

```
ProcessVM
  └── GcHeap
        ├── Nursery  (bump-pointer, tamaño fijo, ej. 1 MB)
        │     └── objetos jóvenes  ->  GcHandle
        │
        ├── OldGen   (bloques de arena, mark-and-sweep)
        │     └── objetos promovidos  ->  GcHandle
        │
        ├── HandleTable[]
        │     GcHandle  ->  dirección actual del objeto
        │
        └── RememberedSet  (unordered_set<GcHandle>)
              handles OLD que contienen referencias a objetos YOUNG
```

### Nursery y la asignación por bump-pointer

La Nursery es una zona de tamaño fijo. Asignar un objeto es tan sencillo como avanzar un puntero (`bump`) en N bytes. No hay que buscar huecos libres: O(1) garantizado.

```
Antes:  [LLENO][LLENO][bump->           ][ libre...     ][ fin ]
         obj1   obj2                      ...

Después NEWOBJ 64:
        [LLENO][LLENO][obj3 64B][bump->  ][ libre...     ][ fin ]
```

Cuando la Nursery se llena, se lanza el **minor GC**. Los objetos vivos se copian a OldGen y el bump pointer vuelve al inicio: toda la Nursery queda libre de golpe, sin necesidad de rastrear individualmente qué está libre y qué no.

### OldGen y el mark-and-sweep

OldGen crece dinámicamente en bloques de arena. No usa bump-pointer porque los objetos tienen distintos tamaños y tiempos de vida. En su lugar usa el algoritmo **mark-and-sweep**:

1. Marcar todos los objetos alcanzables (incluyendo los alcanzables transitivamente a través del grafo).
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
         |       generación: YOUNG o OLD
         estado tri-color (ver abajo)
```

- `size` - bytes del payload (sin contar la cabecera). El escáner lo usa para saltar al siguiente objeto en el bloque.
- `color` - estado del objeto durante el GC (ver más abajo).
- `gen` - en qué generación vive el objeto.

El payload (los datos reales del objeto) viene justo después de la cabecera. Cuando el bytecode llama a `NEWOBJ`, el runtime devuelve un `GcHandle`; cuando accede al objeto, el runtime calcula `dirección_header + 8` para llegar al payload.

---

## Tri-color mark - cómo el GC distingue vivos de muertos

El algoritmo de marcado usa cuatro estados (codificados en 2 bits de la cabecera):

| Color   | Valor | Significado                                              |
| :------ | :---: | :------------------------------------------------------- |
| `WHITE` |   0   | No alcanzado en este ciclo - candidato a liberar         |
| `GREY`  |   1   | Alcanzado, pero sus hijos aún no han sido inspeccionados |
| `BLACK` |   2   | Alcanzado y todos sus hijos inspeccionados - está vivo   |
| `DEAD`  |   3   | Slot liberado por sweep; el espacio es reutilizable      |

El estado `DEAD` es especial: cuando el sweep libera un objeto, preserva el campo `size` y solo cambia el color a `DEAD`. Así el escáner de bloques puede seguir avanzando correctamente (sabe cuánto mide el slot) y `alloc_in_old` puede reutilizar ese hueco si el próximo objeto que llega cabe en él.

### Por qué es necesario el paso PRE-MARK

Los objetos llegan a OldGen con color `BLACK` (recién copiados del minor GC). Si el handle es liberado con `DROP`, el objeto pierde su raíz directa pero sigue con `BLACK`. Sin un paso previo de reset, el SWEEP lo vería `BLACK` y pensaría que está vivo - nunca se liberaría.

La solución es que **antes de cada major GC** todos los objetos no-`DEAD` de OldGen se restablecen a `WHITE`. Así el MARK solo pone `BLACK` los que realmente tienen raíz (directa o transitiva), y el SWEEP puede liberar el resto.

---

## ¿Qué es un handle y por qué no se usan punteros directos?

Cuando se mueve un objeto (por ejemplo, al copiarlo de Nursery a OldGen), su dirección en memoria cambia. Si el bytecode guardase la dirección directa y el GC moviera el objeto, esa dirección quedaría inválida.

La solución es que el bytecode nunca vea la dirección real. En su lugar recibe un **handle** (un entero de 32 bits, índice en la `HandleTable`). Cuando el GC mueve un objeto, solo actualiza la entrada correspondiente en la tabla. Todo el bytecode que tenga ese handle sigue funcionando sin cambios.

```
Antes del minor GC:
  Handle 5  ->  HandleTable[5] = 0x10000020  (en Nursery)

Después del minor GC (el objeto se copió a OldGen):
  Handle 5  ->  HandleTable[5] = 0x30000080  (en OldGen)
  (el bytecode no sabe ni que pasó nada)
```

`GC_NULL_HANDLE = 0xFFFFFFFF` es el handle inválido. Si `NEWOBJ` retorna este valor, la asignación falló (sin memoria disponible).

### Handles embebidos en payloads

Los handles se pueden escribir como valores de 4 bytes dentro del payload de otro objeto. Esto forma el **grafo de objetos** que el GC recorre transitivamente. El GC reconoce automáticamente que un valor de 4 bytes en el payload es un handle si cae dentro del rango válido de la `HandleTable`.

```
Payload de Objeto A (16 bytes):
  offset 0: [handle_B = 0x00000002]  <- 4 bytes, índice en HandleTable
  offset 4: [42       = 0x0000002A]  <- dato numérico
  offset 8: [handle_C = 0x00000005]  <- otra referencia GC
  offset 12:[0x00000000            ]  <- campo vacío
```

> Este escaneo es **conservativo**: el GC trata cualquier valor de 4 bytes que encaje como índice válido de handle como una posible referencia. La consecuencia es que algún dato numérico podría coincidir con un handle y mantener vivo un objeto innecesariamente (memory leak menor), pero nunca se liberará un objeto alcanzable por error.

---

## Ciclo de vida completo de un objeto

```
NEWOBJ size
   |
   |-- espacio en Nursery? --> asignar en Nursery (bump, O(1))
   |
   |-- Nursery llena --> minor_gc()
   |     |
   |     |- objeto alcanzable? --> do_evacuate() --> copiar a OldGen (color=BLACK)
   |     |    |
   |     |    `-> escaneo transitivo del payload (Cheney BFS):
   |     |        si el payload apunta a otro objeto YOUNG no evacuado,
   |     |        también se evacua aunque su handle esté soltado
   |     |
   |     |- objeto no alcanzable? --> abandonado; bump se resetea a base (O(1))
   |     |
   |     `- old_used >= old_threshold? --> major_gc()
   |           |
   |           |- PRE-MARK: todos los objetos no-DEAD --> WHITE
   |           |- MARK:     BFS transitivo desde handles vivos --> BLACK
   |           |            (sigue handles embebidos en payloads)
   |           `- SWEEP:    WHITE --> DEAD, old_used--, stats++
   |
   `-- sigue sin espacio --> asignar directamente en OldGen

DROP handle
   |
   `-> HandleTable[handle].live = false
       (addr se preserva hasta que el GC confirme la muerte del objeto;
        el slot fisico en OldGen lo recoge el siguiente major_gc)
```

---

## Minor GC - Cheney-style transitivo (en detalle)

El minor GC realiza una evacuación completa en dos fases:

**Fase 1 - Semilla de raíces**

1. Escanea todos los handles vivos (`live == true`) cuya dirección cae dentro de la Nursery y evacúa cada objeto joven (`gen == YOUNG`) a OldGen.
2. Escanea los handles del **remembered set** (objetos OLD que contienen referencias a YOUNG) y los usa como raíces adicionales.

**Fase 2 - BFS transitivo (Cheney)**

Por cada objeto recién promovido a OldGen, escanea su payload buscando handles que apunten a objetos YOUNG todavía en Nursery. Si los encuentra, los evacúa también - aunque su handle directo haya sido soltado con `DROP`. Continúa hasta que no queden referencias YOUNG pendientes.

```
Nursery antes:
  [A vivo]  [B solo referenciado por A]  [C muerto]

Fase 1: evacúa A (handle vivo)
Fase 2: escanea payload de A, encuentra handle B -> evacúa B también
Reset: toda la Nursery libre; C y sus vecinos recuperados en O(1)
```

Al terminar, el bump pointer se resetea a la base de la Nursery - toda la Nursery queda libre instantáneamente. Los handles no-vivos que todavía apuntaban a Nursery tienen su `addr` borrada.

Si tras la evacuación `old_used >= old_threshold`, se lanza `major_gc` automáticamente.

> **Pausa:** proporcional al número de objetos **vivos** en Nursery, no al tamaño total. Cuanto más corta sea la vida media de los objetos, menor es la pausa.

---

## Major GC - mark-and-sweep transitivo (en detalle)

Cuatro fases consecutivas sobre OldGen:

**Fase 1 - PRE-MARK**

Recorre todos los bloques de OldGen y pone `WHITE` a todos los objetos que no estén `DEAD`. Necesario porque los objetos llegan con `BLACK` y los handles soltados con `DROP` no tienen mecanismo de reset automático.

**Fase 2 - MARK (BFS transitivo)**

Recorre la `HandleTable`. Por cada handle vivo (`live == true`) cuya dirección cae en OldGen, pone el objeto a `BLACK` y lo añade a un worklist. Luego procesa el worklist: por cada objeto `BLACK`, escanea su payload en busca de handles que apunten a objetos OLD `WHITE` con `addr != nullptr`, los marca `BLACK` y los añade al worklist. Continúa hasta vaciar el worklist.

```
Worklist = [A]           ; A tiene handle vivo
Procesar A -> payload tiene handle B -> B marcado BLACK, añadir B
Worklist = [B]
Procesar B -> payload tiene handle C -> C marcado BLACK, añadir C
Worklist = [C]
Procesar C -> payload vacío -> worklist vacío -> MARK terminado
```

Esto garantiza que el grafo completo de objetos alcanzables desde cualquier handle vivo se marca como `BLACK`, aunque los handles intermedios hayan sido soltados.

**Fase 3 - SWEEP**

Recorre de nuevo los bloques de OldGen. Cada objeto `WHITE` se marca `DEAD`, se decrementa `old_used_` y se actualizan las estadísticas. Los objetos `BLACK` se dejan intactos. Los slots `DEAD` pueden ser reutilizados por `alloc_in_old` si el próximo objeto cabe en el hueco.

**Fase 4 - Limpieza de handles**

Borra el campo `addr` de todos los handles no-vivos (`live == false`) que apuntaban a objetos ahora `DEAD`. Esto libera la memoria y evita que futuros escaneos conservativos los sigan innecesariamente.

---

## Write barrier - referencias entre generaciones

Un problema especial de los GC generacionales: si un objeto de OldGen tiene un campo que apunta a un objeto de la Nursery, el minor GC podría no verlo como alcanzable.

```
OldGen: Objeto A  -->  campo  -->  Handle B (en Nursery)
```

Cuando el minor GC escanea la Nursery, parte de los handles vivos como raíces. Si nadie en los registros apunta a B directamente, y A no está en el remembered set, el minor GC pensaría que B está muerto - aunque A lo referencia.

La solución es el **remembered set**: un `unordered_set` de handles OLD que contienen referencias a objetos YOUNG. El minor GC los trata como raíces adicionales. La inserción es **O(1)** y sin duplicados (a diferencia de un vector).

### Instrucción GCWB

El bytecode debe emitir `GCWB old_handle` **cada vez que escribe un handle dentro del payload de un objeto OLD**:

```asm
gcderef cur0, r_handle_A   ; cur0 = payload de A (objeto OLD)
writecur cur0, r_handle_B  ; escribir handle de B (YOUNG) en el campo
gcwb    r_handle_A         ; registrar que A ahora referencia algo YOUNG
```

`GCWB` es idempotente: llamarlo varias veces con el mismo handle no tiene coste adicional (el `unordered_set` descarta duplicados automáticamente).

### Cuándo llamar GCWB

| Situación | ¿`GCWB` necesario? | Razón |
|:---|:---:|:---|
| Objeto YOUNG escribe ref a YOUNG | No | El minor GC escanea toda la Nursery |
| Objeto YOUNG escribe ref a OLD | No | El minor GC evacúa el YOUNG; la ref se sigue desde OldGen |
| **Objeto OLD escribe ref a YOUNG** | **Sí** | El minor GC no escanea OldGen sin el remembered set |
| Objeto OLD escribe ref a OLD | No | El major GC hace mark completo de OldGen |

Si no puedes saber en tiempo de compilación si el objeto es OLD o YOUNG, **emite siempre `GCWB`**. El coste extra es mínimo.

---

## Instrucciones bytecode

| Instrucción          | Opcode1 | Opcode2 | Descripción                                           |
| :------------------- | :-----: | :-----: | :---------------------------------------------------- |
| `NEWOBJ size`        |  `0x00` |  `0xA0` | Reserva `size` bytes; retorna `GcHandle` en `R0`      |
| `GCRUN`              |  `0x00` |  `0xA1` | Dispara minor + (si procede) major GC manualmente     |
| `GCCONFIG threshold` |  `0x00` |  `0xA2` | Configura el umbral de OldGen para major GC           |
| `DROP handle`        |  `0x00` |  `0xA3` | Libera el handle directo; el objeto puede morir en GC |
| `GCWB old_handle`    |  `0x00` |  `0xA4` | Registra referencia OLD->YOUNG en el remembered set    |

---

## Ejemplos en bytecode Vesta

### Objeto simple

```c
// crear un objeto de 64 bytes y escribir un valor
mov    r1, 64
newobj r1              // R0 = GcHandle

gcderef cur0, r0       // cur0 apunta al payload
mov     r5, 42
writecur cur0, r5      // payload[0] = 42

// leer el valor de vuelta
readcur r6, cur0       // r6 = 42

// liberar cuando ya no se necesite
drop   r0
```

### Grafo de objetos: A referencia a B

```c
// crear objeto A (contenedor, 16 bytes) y objeto B (hijo, 8 bytes)
mov    r1, 16
newobj r1              // R0 = handle A
mov    r8, r0

mov    r1, 8
newobj r1              // R0 = handle B
mov    r9, r0

// escribir el handle de B en el primer campo del payload de A
gcderef cur0, r8       // cur0 = payload de A
writecur cur0, r9      // A.campo[0] = handle B (uint32_t de 4 bytes)
gcwb    r8             // OBLIGATORIO: A (OLD posible) ahora referencia a B (YOUNG)

// soltar el handle directo de B: A lo mantiene vivo a través del grafo
drop   r9

// B sobrevive aunque su handle esté soltado porque A lo referencia
gcrun

// para acceder a B más tarde: leer el handle desde el payload de A
gcderef cur0, r8
readcur r9, cur0       // r9 = handle B recuperado
gcderef cur1, r9       // cur1 = payload de B

// escribir en B a través del handle recuperado
mov     r5, 99
writecur cur1, r5
```

### Cadena de objetos (lista enlazada)

```c
// crear tres nodos: nodo1 -> nodo2 -> nodo3
// cada nodo tiene 8 bytes: [handle_siguiente (4B)][valor (4B)]

mov    r1, 8
newobj r1             // nodo3
mov    r3, r0
gcderef cur0, r3
mov    r5, 0xFFFFFFFF // GC_NULL_HANDLE (sin siguiente)
writecur cur0, r5
mov    r5, 300
// escribir valor en offset 4
// (avanzar cursor manualmente no está soportado directamente;
//  usar dos objetos o un campo separado)

newobj r1             // nodo2
mov    r2, r0
gcderef cur0, r2
writecur cur0, r3     // nodo2.siguiente = nodo3
gcwb   r2             // nodo2 puede ser OLD cuando se enlace

newobj r1             // nodo1
mov    r1_save, r0
gcderef cur0, r1_save
writecur cur0, r2     // nodo1.siguiente = nodo2
gcwb   r1_save

// soltar handles intermedios: la cadena completa sobrevive
// porque nodo1 la mantiene transitivamente viva
drop   r2
drop   r3
gcrun
// nodo2 y nodo3 siguen vivos porque nodo1 -> nodo2 -> nodo3
```

### Ajustar el umbral y forzar GC

```c
// aumentar el umbral antes de una ráfaga de asignaciones
mov    r1, 16777216   // 16 MB
gcconfig r1

// ... muchas asignaciones ...

// forzar un ciclo cuando convenga
gcrun

// reducir el umbral para liberar memoria más agresivamente
mov    r1, 1048576    // 1 MB
gcconfig r1
```

---

## Estadísticas disponibles

Accesibles en C++ via `GcHeap::stats()` (tipo `GcStats`):

| Campo              | Descripción                                           |
| :----------------- | :---------------------------------------------------- |
| `alloc_count`      | Llamadas exitosas a `alloc()`                         |
| `alloc_bytes`      | Bytes totales asignados                               |
| `freed_count`      | Objetos recogidos por sweep                           |
| `freed_bytes`      | Bytes recuperados por sweep                           |
| `promoted_count`   | Objetos evacuados de Nursery a OldGen                 |
| `promoted_bytes`   | Bytes promovidos                                      |
| `minor_gc_count`   | Veces que se ejecutó minor GC                         |
| `major_gc_count`   | Veces que se ejecutó major GC                         |
| `peak_nursery`     | Máximo de bytes vivos en Nursery en cualquier momento |
| `peak_old`         | Máximo de bytes vivos en OldGen en cualquier momento  |

Las estadísticas son acumulativas desde que el proceso arranca. Son útiles para diagnosticar presión de memoria, ajustar el threshold de OldGen y verificar que la hipótesis generacional se cumple en el programa concreto (si `promoted_count / alloc_count` es bajo, los objetos mueren jóvenes como se espera).

---

## Seguridad de hilos

`GcHeap` es **per-proceso** y no thread-safe. La sincronización la aporta el planificador (`Scheduler`): un `ProcessVM` solo corre en un hilo nativo a la vez, por lo que el GC nunca compite con el código de usuario del mismo proceso.

Ver también: [[GC]], [[Allocator crudo para FFI  y memoria manual|RawAllocator]], [[VmInstance]], [[VmManager]]
