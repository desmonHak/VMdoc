# Gestion de memoria - Sistema GC de VestaVM

## Por que la memoria necesita gestion

Cuando un programa crea una variable, un objeto o un buffer de datos, el sistema operativo
le presta un trozo de memoria RAM para almacenarlo. Esa memoria es un recurso finito: si
el programa pide mas de la que hay disponible, falla.

El problema es que los programas crean y destruyen datos constantemente. Si cada trozo de
memoria pedido nunca se devuelve, la memoria disponible se agota aunque los datos ya no
sirvan para nada. Esto se llama **fuga de memoria** (memory leak), y es uno de los errores
mas comunes y dificiles de detectar en programacion.

Hay dos grandes enfoques para resolver esto:

| Enfoque                | Quien libera la memoria        | Ejemplo                    |
| :--------------------- | :----------------------------- | :------------------------- |
| **Manual**             | El programador, explicitamente | C (`malloc` / `free`)      |
| **Automatico (GC)**    | El runtime, en segundo plano   | Java, Python, Go, C#       |

VestaVM ofrece **ambos enfoques** a la vez, porque cada uno tiene su lugar segun el caso.

---

## Que es un Garbage Collector

Un **Garbage Collector** (GC, recolector de basura) es un componente del runtime que
observa que objetos del programa siguen siendo accesibles (tienen al menos una referencia
activa) y cuales ya no lo son (son "basura"). Los objetos basura se destruyen
automaticamente para liberar memoria.

### Analogia: la habitacion y los juguetes

Imagina una habitacion donde los ninos sacan juguetes para jugar. Algunos juguetes se
siguen usando; otros quedan abandonados en el suelo. El GC es como alguien que entra
periodicamente, comprueba que juguetes nadie esta tocando y los recoge. Los ninos no
tienen que preocuparse por "devolver" los juguetes; simplemente dejan de usarlos y en
algun momento desaparecen solos.

### Cuando se ejecuta el GC

El GC no corre constantemente (seria demasiado lento). Se activa cuando:

1. Se intenta asignar memoria y no hay espacio libre suficiente en el heap.
2. El programador lo solicita explicitamente con `GCRUN`.
3. El volumen de objetos "viejos" supera un umbral configurable (`GCCONFIG`).

### Tiene coste

Si. Mientras el GC analiza que objetos estan vivos, el programa hace una pequena pausa.
VestaVM minimiza esto con un diseno **generacional** (ver mas abajo).

---

## Que es la gestion manual

En la gestion manual el programador pide memoria con `ALLOC` y la devuelve con `FREE`
cuando ya no la necesita. Es mas rapido porque no hay analisis automatico, pero requiere
disciplina:
- Olvidar un `FREE` produce una fuga de memoria.
- Hacer `FREE` dos veces sobre el mismo bloque produce corrupcion de memoria.

Este modo es imprescindible cuando el programa necesita pasar punteros de memoria a
funciones externas escritas en C (llamadas **FFI**, Foreign Function Interface), porque
esas funciones esperan la direccion real de un bloque de bytes en la RAM, no un
identificador abstracto del GC.

---

## Los dos sistemas de VestaVM

VestaVM separa los dos enfoques en componentes distintos, uno por cada proceso VM:

```
ProcessVM
  +-- GcHeap        -- GC automatico generacional  (objetos OOP, ciclo de vida automatico)
  +-- RawAllocator  -- Allocator manual             (buffers FFI, control explicito)

VM (compartido entre todos los procesos)
  +-- shared_futures[]  -- FutureObject fuera del GcHeap per-proceso
```

Cada proceso tiene sus propias instancias de `GcHeap` y `RawAllocator`. No se comparten
entre procesos, lo que significa que el GC de un proceso nunca bloquea a otro.

Los `FutureObject` viven en `vm_ref.shared_futures` (vector protegido por mutex por VM),
no en el GcHeap per-proceso, para que distintos procesos puedan cumplir (`fulfill`) y
esperar (`await`) el mismo future sin problemas de aliasing.

### Resumen comparativo

| Caracteristica           | GcHeap (generacional)             | RawAllocator (manual)               |
| :----------------------- | :-------------------------------- | :---------------------------------- |
| Instruccion de reserva   | `NEWOBJ r` -> `GcHandle`          | `ALLOC r` -> ptr host real          |
| Instruccion de liberacion| `DROP handle` (opcional)          | `FREE reg` (obligatorio)            |
| GC automatico            | Si - minor + major                | No                                  |
| Tipo de retorno          | Handle opaco (no es una direccion)| Puntero host real (`uint64_t`)      |
| Compatible con FFI       | No (el handle no es una dir. real)| Si                                  |
| Uso ideal                | Objetos OOP, grafos, closures     | Buffers C, strings, llamadas nativas|

> **Nota:** Los `StringObject` son una excepcion: se alocan con `alloc_pinned()` directamente
> en OldGen (sin pasar por Nursery) para que el puntero `data[]` retornado por `STRRAW` sea
> estable durante toda la vida del handle. Los `StringObject` en OldGen no se mueven durante
> el mark-and-sweep, por lo que el puntero host es valido mientras el handle este vivo.

---

## ObjectHeader ABI v2 (objetos OOP)

Los objetos creados con `NEWOBJ` llevan un `ObjectHeader` de **24 bytes** al inicio de su
payload (inmediatamente despues de la GcHeader interna de 8 bytes):

```cpp
struct alignas(8) ObjectHeader {
    ClassInfo *class_ptr;  // 8 bytes - puntero al ClassInfo del ClassRegistry
    uint32_t   flags;      // 4 bytes - OBJ_FLAG_* (GC_OWNED, etc.)
    uint32_t   hash_code;  // 4 bytes - hash identidad (lazy)
    uint32_t   owner_pid;  // 4 bytes - local_pid del proceso que tiene el monitor (0=libre)
    uint16_t   lock_depth; // 2 bytes - profundidad de bloqueo reentrante
    uint16_t   _mon_pad;   // 2 bytes - padding de alineacion
};
```

Esto es un **cambio breaking** respecto a la v1 (que tenia 16 bytes). Cualquier codigo que
acceda a campos del ObjectHeader en offsets fijos >= 16 debe actualizarse.

Los campos del usuario empiezan en `offset +32` desde la GcHeader (8 GcHeader + 24 ObjectHeader).
El campo `class_ptr` en `offset 0` del ObjectHeader es lo que lee `getClass(obj)` (instruccion
`LOAD i64 [obj+0]` con `is_host_ptr=true`).

---

## GC generacional: por que los objetos "jovenes" mueren pronto

La observacion clave detras del GC de VestaVM es la **hipotesis generacional**:

> La mayoria de los objetos mueren jovenes.

Es decir, la mayor parte de los objetos que un programa crea tienen una vida muy corta:
se crean para hacer un calculo, se usan y ya no se necesitan. Solo una minoria vive
durante mucho tiempo (configuracion global, caches, estructuras persistentes).

Aprovechando esto, el heap se divide en dos zonas:

```
+---------------------------+------------------------------------+
|         Nursery           |             OldGen                 |
|   (objetos jovenes)       |   (objetos que sobrevivieron)      |
|   asignacion rapida O(1)  |   mark-and-sweep cuando se llena   |
+---------------------------+------------------------------------+
```

- **Nursery**: zona pequena y rapida donde nacen todos los objetos. Limpiarla es barato
  porque casi todo esta muerto; no hace falta analizar los objetos supervivientes en detalle.
  Este ciclo de limpieza se llama **minor GC**.
- **OldGen**: zona donde van los objetos que sobrevivieron al menos un ciclo de limpieza
  de la Nursery. Se limpia con un algoritmo mas completo (**major GC**) pero se activa
  con mucha menos frecuencia.

Resultado: **pausas cortas y frecuentes** (minor GC) en lugar de una pausa larga y rara.

---

## Que es un GcHandle

En el GcHeap, el programador nunca recibe la direccion real del objeto en memoria. En su
lugar recibe un **GcHandle**: un numero entero de 32 bits que actua como identificador.

```
Bytecode:   NEWOBJ r1    ->  R0 = 3          (handle, no es una direccion)

HandleTable:   [3]  ->  0x7f3a0000       (direccion real, interna al GC)
```

Por que un handle y no una direccion directa? Porque el GC puede necesitar **mover**
objetos (por ejemplo, al copiar un objeto joven a OldGen). Si el bytecode guardase la
direccion directa y el GC mueve el objeto, esa direccion quedaria invalida. Con handles,
solo hay que actualizar la entrada en la `HandleTable`; el bytecode no cambia nada.

El valor especial `GC_NULL_HANDLE = 0xFFFFFFFF` indica fallo de asignacion o handle nulo.

En el `RawAllocator` no existe este problema porque el bloque nunca se mueve: la direccion
retornada es permanente mientras no se llame a `FREE` o `REALLOC`.

---

## Roots externos para plugins (A.30)

Los plugins nativos escritos en C pueden retener handles GC en estructuras de datos propias
(colecciones, caches) que el escaner conservativo del GC no visita. Para que el GC no
libere esos objetos, la `VestaPluginAPI v2` expone dos funciones:

```c
// Incrementar refcount externo del handle (el GC no lo recolecta mientras refcount > 0)
g_api->gc_addref(proc_ptr, handle);

// Decrementar refcount externo; si llega a 0 el GC puede liberarlo
g_api->gc_release(proc_ptr, handle);
```

Internamente, `GcHeap` mantiene un `unordered_map<GcHandle, uint32_t> external_refs_`.
Durante el major GC, la fase MARK itera `external_refs_` y trata como BLACK cualquier
handle con refcount > 0, aunque el stack scan no lo haya encontrado.

`release_handle` respeta el refcount: si el bytecode hace `DROP h` pero el handle sigue
con refcount externo > 0, el handle NO se libera hasta que el ultimo `gc_release` lo
desregistre.

---

## Opcodes de referencia rapida

### GcHeap

| Instruccion          | Opcode1 | Opcode2 | Descripcion                                             |
| :------------------- | :-----: | :-----: | :------------------------------------------------------ |
| `newobj r`           | `0x00`  | `0xA0`  | Crea objeto OOP; retorna `GcHandle` en R0               |
| `gcalloc r`          | `0x00`  | `0xA5`  | Crea bloque GC sin ObjectHeader; retorna GcHandle en R0 |
| `gcrun`              | `0x00`  | `0xA1`  | Fuerza un ciclo de GC ahora                             |
| `gcconfig r`         | `0x00`  | `0xA2`  | Ajusta el umbral de OldGen                              |
| `drop r`             | `0x00`  | `0xA3`  | Libera el handle (el GC recogera el objeto)             |
| `gcwb r`             | `0x00`  | `0xA4`  | Registra referencia OLD->YOUNG en el remembered set     |
| `gchandle r_dst, r_src` | `0x00` | `0x56` | host_ptr -> GcHandle via lookup O(1) en ptr_to_handle_ |

### RawAllocator

| Instruccion          | Opcode1 | Opcode2 | Descripcion                               |
| :------------------- | :-----: | :-----: | :---------------------------------------- |
| `alloc r`            | `0x00`  | `0xB0`  | Reserva bytes; retorna ptr real en R0     |
| `free r`             | `0x00`  | `0xB1`  | Libera el bloque en `r` inmediatamente    |
| `realloc r, r`       | `0x00`  | `0xB2`  | Redimensiona; nuevo ptr en R0             |

---

## Sintaxis en ensamblador Vesta

### GcHeap

```c
// Crear un bloque de 64 bytes bajo GC - el handle retorna en R0
mov    r1, 64
gcalloc r1           // R0 = GcHandle

// Para leer y escribir el payload, primero obtenemos el puntero real con gcderef:
gcderef cur0, r0     // cur0 = puntero real al payload en memoria host
mov     r5, 42
writecur cur0, r5    // escribe 42 en los primeros 8 bytes del bloque
readcur  r6, cur0    // r6 = lee los primeros 8 bytes (debe ser 42)

// Forzar un ciclo de GC ahora mismo
gcrun

// Ajustar el umbral de OldGen (en bytes):
mov    r2, 16777216   // 16 MB
gcconfig r2

// Liberar el handle cuando ya no se necesite:
mov    r3, r0          // guardar el handle en r3
drop   r3              // el GC recogera el objeto en el proximo ciclo

// Write barrier: OBLIGATORIO cuando un objeto OLD referencia a uno YOUNG
// (Si A esta en OldGen y B esta en Nursery, y escribimos handle_B en A):
gcderef cur0, r_handle_A
writecur cur0, r_handle_B   // A.campo = handle_B
gcwb r_handle_A             // notificar al GC de esta referencia OLD->YOUNG
```

### RawAllocator

```c
// Reservar un buffer de 256 bytes - se obtiene el puntero real directamente
mov    r1, 256
alloc  r1             // R0 = puntero host real (no un handle)

// Guardar el puntero y acceder al buffer con cursor:
mov    r14, r0
xchg   cur1, r14      // cur1 = inicio del buffer
mov    r5, 0xFF
writecur cur1, r5     // buffer[0..7] = 0xFF

// Redimensionar el buffer (puede cambiar de direccion):
alloc  r1             // Reservar el original de nuevo para este ejemplo
mov    r2, 1024
realloc r0, r2        // R0 = nuevo puntero (puede ser diferente del original)

// Liberar OBLIGATORIAMENTE cuando ya no se necesite:
free   r0
```

---

## Documentacion detallada

- [[Generacional (para objetos OOP)]] - Como funciona el GcHeap internamente: Nursery,
  minor GC, major GC, algoritmo tri-color mark, write barrier, handles, estadisticas,
  stack scanning conservativo.
- [[Allocator crudo para FFI y memoria manual]] - Como funciona el RawAllocator:
  contiguidad, FFI, ALLOC/FREE/REALLOC, estadisticas.
- [[cursor]] - Instrucciones `readcur`, `writecur` y `gcderef` para acceder a memoria
  host desde bytecode.

Ver tambien: [[NativeCall (CallN)]], [[NEWOBJRAW y NEWOBJ]], [[GCALLOC]],
[[SetInstruccionesVM/FFI_RUNTIME]] (gchandle 0x56), [[SetInstruccionesVM/STATIC_FIELDS]]
