# GC Generacional - GcHeap

Gestiona objetos del lenguaje con ciclo de vida automatico. Cada proceso VM tiene su propio
`GcHeap`, lo que elimina pausas inter-proceso y contention de bloqueos.

> Si no conoces que es un GC ni por que se necesita, empieza por [[GC]] antes de leer
> esta pagina.

---

## Como sabe el GC que esta "vivo"?

Un objeto esta **vivo** si existe alguna forma de llegar a el desde el codigo que se esta
ejecutando. Esa "forma de llegar" se llama **raiz**: normalmente un registro de la CPU
virtual (R0-R15), una variable local en la pila, o un campo de otro objeto ya vivo.

```c
// Ejemplo de grafo de alcanzabilidad:
//
//   Registros del proceso (R0..R15)
//     |-- handle 3  -->  Objeto A  -->  (campo)  -->  handle 7  -->  Objeto B
//                                                      |
//                                                      `--> handle 12  -->  Objeto C
//
//   Objeto D  (ninguna ruta desde los registros llega hasta el)  --> BASURA
```

El GC recorre el grafo de referencias partiendo de las raices. Todo lo que no se puede
alcanzar es basura y su memoria se puede reutilizar.

> **Importante:** un objeto sigue vivo aunque su handle haya sido liberado con `DROP`,
> siempre que otro objeto vivo tenga su handle embebido en el payload. El GC hace un
> recorrido transitivo del grafo, no solo mira los handles directos.

---

## Por que dos generaciones?

Repasar todo el heap cada vez que hay que limpiar es caro, especialmente si el heap es
grande. La **hipotesis generacional** dice que la mayoria de los objetos mueren pronto,
asi que podemos limpiar solo la zona de objetos jovenes (que suele estar casi vacia de
supervivientes) muy rapido, y dejar la zona de objetos viejos para revisiones mucho menos
frecuentes.

```c
// Ejemplo con 10 000 objetos creados en un ciclo tipico:
//   8 000 mueren antes del primer GC  ->  limpiar Nursery es baratisimo
//     800 sobreviven y van a OldGen
//      50 siguen vivos 10 ciclos despues
//
// Limpiar solo la Nursery (1 200 vivos + 8 000 muertos en un espacio pequeno)
// es mucho mas rapido que revisar los 10 000 objetos completos del heap.
```

---

## Arquitectura interna

```
ProcessVM
  |-- GcHeap
        |-- Nursery  (bump-pointer, tamano fijo, ej. 1 MB)
        |     |-- objetos jovenes  -> GcHandle
        |
        |-- OldGen   (bloques de arena, mark-and-sweep)
        |     |-- objetos promovidos  -> GcHandle
        |
        |-- HandleTable[]
        |     GcHandle  -> direccion actual del objeto
        |
        |-- RememberedSet  (unordered_set de GcHandle)
              handles OLD que contienen referencias a objetos YOUNG
```

### Nursery y la asignacion por bump-pointer

La Nursery es una zona de tamano fijo. Asignar un objeto es tan sencillo como avanzar un
puntero (`bump`) en N bytes. No hay que buscar huecos libres: O(1) garantizado.

```
Antes:  [LLENO][LLENO][bump->           ][ libre...     ][ fin ]
         obj1   obj2

Despues NEWOBJ 64 bytes:
        [LLENO][LLENO][obj3 64B][bump->  ][ libre...     ][ fin ]
```

Cuando la Nursery se llena, se lanza el **minor GC**. Los objetos vivos se copian a OldGen
y el bump pointer vuelve al inicio: toda la Nursery queda libre de golpe, sin necesidad de
rastrear individualmente que esta libre y que no.

### OldGen y el mark-and-sweep

OldGen crece dinamicamente en bloques de arena. No usa bump-pointer porque los objetos
tienen distintos tamanos y tiempos de vida. En su lugar usa el algoritmo **mark-and-sweep**:

1. **Marcar** todos los objetos alcanzables (incluyendo los alcanzables transitivamente
   a traves del grafo de referencias).
2. **Barrer** (sweep): los no marcados son basura y se liberan.

OldGen solo se limpia cuando su ocupacion supera el umbral configurado con `GCCONFIG`.

---

## Cabecera interna de objeto (GcHeader)

Cada objeto en el heap GC lleva una cabecera interna de 8 bytes **antes del payload**:

```
Offset 0                                           Offset 8
+----------+-------+-----------+------+----------+
| size     | color | generacion| _pad | reserved |
| 32 bits  | 2 bit | 1 bit     | 5 b  | 3 bytes  |
+----------+-------+-----------+------+----------+
              ^       ^
              |       YOUNG=0 o OLD=1
              estado tri-color del GC (ver abajo)
```

- `size` - bytes del payload (sin la cabecera). El escaner lo usa para saltar al siguiente
  objeto en el bloque.
- `color` - estado del objeto durante el GC (WHITE/GREY/BLACK/DEAD).
- `generacion` - en que generacion vive el objeto (YOUNG o OLD).

> **Nota:** Esta GcHeader es la cabecera interna del GC, distinta del `ObjectHeader` de
> 24 bytes que forma parte del payload de los objetos OOP (creados con NEWOBJ). El payload
> de NEWOBJ empieza 8 bytes despues de la GcHeader, y el ObjectHeader de 24 bytes ocupa
> los primeros 24 bytes de ese payload. Los campos del usuario empiezan en offset +32 desde
> la GcHeader, o en offset +24 desde el inicio del payload.

---

## Tri-color mark: como el GC distingue vivos de muertos

El algoritmo de marcado usa cuatro estados (codificados en 2 bits):

| Color   | Valor | Significado                                               |
| :------ | :---: | :-------------------------------------------------------- |
| `WHITE` |   0   | No alcanzado en este ciclo; candidato a liberar           |
| `GREY`  |   1   | Alcanzado, pero sus hijos aun no han sido inspeccionados  |
| `BLACK` |   2   | Alcanzado y todos sus hijos inspeccionados; esta vivo     |
| `DEAD`  |   3   | Slot liberado por sweep; el espacio es reutilizable       |

El estado `DEAD` es especial: cuando el sweep libera un objeto, preserva el campo `size`
y solo cambia el color a `DEAD`. Asi el escaner de bloques puede seguir avanzando
correctamente (sabe cuanto mide el slot) y `alloc_in_old` puede reutilizar ese hueco si
el proximo objeto que llega cabe en el.

### Por que hace falta el paso PRE-MARK

Los objetos llegan a OldGen con color `BLACK` (recien copiados del minor GC). Si el handle
se libera con `DROP`, el objeto pierde su raiz directa pero sigue con `BLACK`. Sin un paso
previo de reset, el SWEEP lo veria `BLACK` y pensaria que esta vivo; nunca se liberaria.

La solucion es que **antes de cada major GC** todos los objetos no-`DEAD` de OldGen se
restablecen a `WHITE`. Asi el MARK solo pone `BLACK` los que realmente tienen raiz (directa
o transitiva), y el SWEEP puede liberar el resto.

---

## Por que se usan handles y no punteros directos

Cuando se mueve un objeto (por ejemplo, al copiarlo de Nursery a OldGen), su direccion en
memoria cambia. Si el bytecode guardase la direccion directa y el GC moviera el objeto,
esa direccion quedaria invalida.

La solucion es que el bytecode nunca vea la direccion real. En su lugar recibe un **handle**
(un entero de 32 bits, indice en la `HandleTable`). Cuando el GC mueve un objeto, solo
actualiza la entrada correspondiente en la tabla. Todo el bytecode que tenga ese handle
sigue funcionando sin cambios.

```c
// Antes del minor GC:
//   Handle 5  ->  HandleTable[5] = 0x10000020  (en Nursery)
//
// Despues del minor GC (el objeto se copio a OldGen):
//   Handle 5  ->  HandleTable[5] = 0x30000080  (en OldGen)
//   (el bytecode no sabe ni que paso nada)
```

`GC_NULL_HANDLE = 0xFFFFFFFF` es el handle invalido. Si `NEWOBJ` o `GCALLOC` retornan
este valor, la asignacion fallo (sin memoria disponible).

### Handles embebidos en payloads

Los handles (valores de 4 bytes) se pueden escribir dentro del payload de otro objeto.
Esto forma el **grafo de objetos** que el GC recorre transitivamente.

```c
// Payload de Objeto A (16 bytes):
//   offset 0: [handle_B = 0x00000002]  // 4 bytes: indice en HandleTable
//   offset 4: [42       = 0x0000002A]  // dato numerico
//   offset 8: [handle_C = 0x00000005]  // otra referencia GC
//   offset 12:[0x00000000            ]  // campo vacio (null)
```

> Este escaneo es **conservativo**: el GC trata cualquier valor de 4 bytes que encaje como
> indice valido de handle como una posible referencia. La consecuencia es que algun dato
> numerico podria coincidir con un handle y mantener vivo un objeto innecesariamente
> (memory leak menor), pero nunca se liberara un objeto alcanzable por error.

---

## Ciclo de vida completo de un objeto

```c
// NEWOBJ/GCALLOC
//   |
//   |-- espacio en Nursery? --> asignar en Nursery (bump, O(1))
//   |
//   |-- Nursery llena --> minor_gc()
//   |     |
//   |     |-- objeto alcanzable? --> do_evacuate() --> copiar a OldGen (color=BLACK)
//   |     |    |
//   |     |    `-> escaneo transitivo del payload (Cheney BFS):
//   |     |        si el payload apunta a otro objeto YOUNG no evacuado,
//   |     |        tambien se evacua aunque su handle este soltado
//   |     |
//   |     |-- objeto no alcanzable? --> abandonado; bump se resetea a base (O(1))
//   |     |
//   |     `-- old_used >= old_threshold? --> major_gc()
//   |           |
//   |           |-- PRE-MARK: todos los objetos no-DEAD --> WHITE
//   |           |-- MARK:     BFS transitivo desde handles vivos --> BLACK
//   |           |             (sigue handles embebidos en payloads)
//   |           `-- SWEEP:    WHITE --> DEAD, old_used--, stats++
//   |
//   `-- sigue sin espacio --> asignar directamente en OldGen
//
// DROP handle
//   |
//   `-> HandleTable[handle].live = false
//       (addr se preserva hasta que el GC confirme la muerte del objeto;
//        el slot fisico en OldGen lo recoge el siguiente major_gc)
```

---

## Minor GC: Cheney-style transitivo (en detalle)

El minor GC realiza una evacuacion completa en dos fases:

**Fase 1 - Semilla de raices:**

1. Escanea todos los handles vivos (`live == true`) cuya direccion cae dentro de la Nursery
   y evacua cada objeto joven (`gen == YOUNG`) a OldGen.
2. Escanea los handles del **remembered set** (objetos OLD que contienen referencias a
   YOUNG) y los usa como raices adicionales.

**Fase 2 - BFS transitivo (Cheney):**

Por cada objeto recien promovido a OldGen, escanea su payload buscando handles que apunten
a objetos YOUNG todavia en Nursery. Si los encuentra, los evacua tambien, aunque su handle
directo haya sido soltado con `DROP`. Continua hasta que no queden referencias YOUNG
pendientes.

```c
// Nursery antes del minor GC:
//   [A vivo]  [B solo referenciado por A]  [C muerto - nadie lo referencia]
//
// Fase 1: evacua A (tiene handle vivo)
// Fase 2: escanea payload de A, encuentra handle B --> evacua B tambien
// Reset: toda la Nursery queda libre; C es basura recuperada en O(1)
```

Al terminar, el bump pointer se resetea a la base de la Nursery: toda la Nursery queda
libre instantaneamente. Los handles no-vivos que todavia apuntaban a Nursery tienen su
`addr` borrada.

Si tras la evacuacion `old_used >= old_threshold`, se lanza `major_gc` automaticamente.

> **Pausa del minor GC:** proporcional al numero de objetos **vivos** en Nursery, no al
> tamano total. Cuanto mas corta sea la vida media de los objetos, menor es la pausa.

---

## Major GC: mark-and-sweep transitivo (en detalle)

Cuatro fases consecutivas sobre OldGen:

**Fase 1 - PRE-MARK:**

Recorre todos los bloques de OldGen y pone `WHITE` a todos los objetos que no esten `DEAD`.
Necesario porque los objetos llegan con `BLACK` y los handles soltados con `DROP` no tienen
mecanismo de reset automatico.

**Fase 2 - MARK (BFS transitivo):**

Recorre la `HandleTable`. Por cada handle vivo (`live == true`) cuya direccion cae en
OldGen, pone el objeto a `BLACK` y lo anade a un worklist. Luego procesa el worklist: por
cada objeto `BLACK`, escanea su payload en busca de handles que apunten a objetos OLD
`WHITE`, los marca `BLACK` y los anade al worklist. Continua hasta vaciar el worklist.

```c
// Ejemplo de recorrido BFS:
//   Worklist = [A]           // A tiene handle vivo
//   Procesar A -> payload tiene handle B -> B marcado BLACK, anadir B
//   Worklist = [B]
//   Procesar B -> payload tiene handle C -> C marcado BLACK, anadir C
//   Worklist = [C]
//   Procesar C -> payload vacio -> worklist vacio -> MARK terminado
```

Esto garantiza que el grafo completo de objetos alcanzables desde cualquier handle vivo
se marca como `BLACK`, aunque los handles intermedios hayan sido soltados.

**Fase 3 - SWEEP:**

Recorre de nuevo los bloques de OldGen. Cada objeto `WHITE` se marca `DEAD`, se decrementa
`old_used_` y se actualizan las estadisticas. Los objetos `BLACK` se dejan intactos. Los
slots `DEAD` pueden ser reutilizados por `alloc_in_old` si el proximo objeto cabe en el
hueco.

**Fase 4 - Limpieza de handles:**

Borra el campo `addr` de todos los handles no-vivos (`live == false`) que apuntaban a
objetos ahora `DEAD`. Esto libera la memoria y evita que futuros escaneos conservativos
los sigan innecesariamente.

---

## Write barrier: referencias entre generaciones

Un problema especial de los GC generacionales: si un objeto de OldGen tiene un campo que
apunta a un objeto de la Nursery, el minor GC podria no verlo como alcanzable.

```c
// Problema:
//   OldGen: Objeto A  -->  campo  -->  Handle B (en Nursery)
//
// Si nadie en los registros apunta a B directamente, y A no esta en el
// remembered set, el minor GC pensaria que B esta muerto aunque A lo referencia.
```

La solucion es el **remembered set**: un `unordered_set` de handles OLD que contienen
referencias a objetos YOUNG. El minor GC los trata como raices adicionales. La insercion
es O(1) y sin duplicados (a diferencia de un vector).

### Instruccion GCWB

El bytecode debe emitir `GCWB old_handle` **cada vez que escribe un handle dentro del
payload de un objeto OLD**:

```c
gcderef  cur0, r_handle_A   // cur0 = payload de A (objeto OLD)
writecur cur0, r_handle_B   // escribir handle de B (YOUNG) en el campo
gcwb     r_handle_A         // registrar que A ahora referencia algo YOUNG
```

`GCWB` es idempotente: llamarlo varias veces con el mismo handle no tiene coste adicional
(el `unordered_set` descarta duplicados automaticamente).

### Cuando llamar GCWB

| Situacion                             | GCWB necesario? | Razon                                    |
| :------------------------------------ | :-------------: | :--------------------------------------- |
| Objeto YOUNG escribe ref a YOUNG      | No              | El minor GC escanea toda la Nursery      |
| Objeto YOUNG escribe ref a OLD        | No              | El minor GC evacua el YOUNG; ref seguida |
| **Objeto OLD escribe ref a YOUNG**    | **Si**          | El minor GC no escanea OldGen sin RS     |
| Objeto OLD escribe ref a OLD          | No              | El major GC hace mark completo de OldGen |

> Si no puedes saber en tiempo de compilacion si el objeto es OLD o YOUNG, **emite siempre
> `GCWB`**. El coste extra es minimo (hash set con deduplicacion automatica).

---

## Instrucciones bytecode

| Instruccion          | Opcode0 | Opcode1 | Descripcion                                           |
| :------------------- | :-----: | :-----: | :---------------------------------------------------- |
| `newobj size`        |  0x00   |  0xA0   | Reserva `size` bytes; retorna GcHandle en R0          |
| `gcrun`              |  0x00   |  0xA1   | Dispara minor + (si procede) major GC manualmente     |
| `gcconfig threshold` |  0x00   |  0xA2   | Configura el umbral de OldGen para major GC           |
| `drop handle`        |  0x00   |  0xA3   | Libera el handle directo; el objeto puede morir en GC |
| `gcwb old_handle`    |  0x00   |  0xA4   | Registra referencia OLD->YOUNG en el remembered set   |

---

## Ejemplos en bytecode Vesta

### Objeto simple con un campo

```c
// Crear un objeto de 64 bytes y escribir un valor en el
mov    r1, 64
newobj r1              // R0 = GcHandle del nuevo objeto

gcderef cur0, r0       // cur0 apunta al inicio del payload
mov     r5, 42
writecur cur0, r5      // payload[0] = 42 (primer qword)

// Leer el valor de vuelta
readcur r6, cur0       // r6 = 42

// Liberar el handle cuando ya no se necesite
drop   r0
```

### Grafo de objetos: A referencia a B

```c
// Crear objeto A (contenedor, 16 bytes) y objeto B (hijo, 8 bytes)
mov    r1, 16
newobj r1              // R0 = handle A
mov    r8, r0

mov    r1, 8
newobj r1              // R0 = handle B
mov    r9, r0

// Escribir el handle de B en el primer campo del payload de A
gcderef cur0, r8       // cur0 = payload de A
writecur cur0, r9      // A.campo[0] = handle B (4 bytes del handle)
gcwb    r8             // OBLIGATORIO: A (puede ser OLD) ahora referencia a B (YOUNG)

// Soltar el handle directo de B: A lo mantiene vivo a traves del grafo
drop   r9

// B sobrevive aunque su handle este soltado porque A lo referencia
gcrun

// Para acceder a B mas tarde: leer el handle desde el payload de A
gcderef cur0, r8
readcur r9, cur0       // r9 = handle B recuperado
gcderef cur1, r9       // cur1 = payload de B

mov     r5, 99
writecur cur1, r5      // escribir en B a traves del handle recuperado
```

### Ajustar el umbral y forzar GC

```c
// Aumentar el umbral antes de una rafaga de asignaciones
mov    r1, 16777216   // 16 MB
gcconfig r1

// ... muchas asignaciones ...

// Forzar un ciclo cuando convenga (por ejemplo, entre etapas de trabajo)
gcrun

// Reducir el umbral para liberar memoria mas agresivamente al terminar
mov    r1, 1048576    // 1 MB
gcconfig r1
```

---

## Estadisticas disponibles

Accesibles en C++ via `GcHeap::stats()` (tipo `GcStats`):

| Campo              | Descripcion                                            |
| :----------------- | :----------------------------------------------------- |
| `alloc_count`      | Llamadas exitosas a `alloc()`                          |
| `alloc_bytes`      | Bytes totales asignados                                |
| `freed_count`      | Objetos recogidos por sweep                            |
| `freed_bytes`      | Bytes recuperados por sweep                            |
| `promoted_count`   | Objetos evacuados de Nursery a OldGen                  |
| `promoted_bytes`   | Bytes promovidos                                       |
| `minor_gc_count`   | Veces que se ejecuto minor GC                          |
| `major_gc_count`   | Veces que se ejecuto major GC                          |
| `peak_nursery`     | Maximo de bytes vivos en Nursery en cualquier momento  |
| `peak_old`         | Maximo de bytes vivos en OldGen en cualquier momento   |

Las estadisticas son acumulativas desde que el proceso arranca. Son utiles para diagnosticar
presion de memoria, ajustar el threshold de OldGen, y verificar que la hipotesis generacional
se cumple: si `promoted_count / alloc_count` es bajo, los objetos mueren jovenes como se
espera.

---

## Seguridad de hilos

`GcHeap` es **per-proceso** y no thread-safe. La sincronizacion la aporta el planificador
(`Scheduler`): un `ProcessVM` solo corre en un hilo nativo a la vez, por lo que el GC
nunca compite con el codigo de usuario del mismo proceso.

---

Ver tambien: [[GC]], [[Allocator crudo para FFI y memoria manual]], [[GCALLOC]], [[cursor]]
