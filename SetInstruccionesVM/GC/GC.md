# Gestión de memoria — Sistema GC de VestaVM

## ¿Qué es la memoria y por qué hay que gestionarla?

Cuando un programa crea una variable, un objeto o un buffer de datos, el sistema operativo le presta un trozo de memoria RAM para almacenarlo. Esa memoria es un recurso finito: si el programa pide más de la que hay disponible, falla.

El problema es que los programas crean y destruyen datos constantemente. Si cada trozo de memoria pedido nunca se devuelve, la memoria disponible se agota aunque los datos ya no sirvan para nada. Esto se llama **fuga de memoria** (*memory leak*), y es uno de los errores más comunes y difíciles de detectar en programación.

Hay dos grandes enfoques para resolver esto:

| Enfoque                    | Quién libera la memoria          | Ejemplo                        |
| :------------------------- | :------------------------------- | :----------------------------- |
| **Manual**                 | El programador, explícitamente   | C (`malloc` / `free`)          |
| **Automático (GC)**        | El runtime, en segundo plano     | Java, Python, Go, C#           |

VestaVM ofrece **ambos enfoques** a la vez, porque cada uno tiene su lugar según el caso de uso.

---

## ¿Qué es un Garbage Collector?

Un **Garbage Collector** (GC, recolector de basura) es un componente del runtime que observa qué objetos del programa siguen siendo accesibles (tienen al menos una referencia activa) y cuáles ya no lo son (son "basura"). Los objetos basura se destruyen automáticamente para liberar memoria.

### Analogía: la habitación y los juguetes

Imagina una habitación donde los niños sacan juguetes para jugar. Algunos juguetes se siguen usando; otros quedan abandonados en el suelo. El GC es como alguien que entra periódicamente, comprueba qué juguetes nadie está tocando y los recoge. Los niños no tienen que preocuparse por "devolver" los juguetes; simplemente dejan de usarlos y en algún momento desaparecen solos.

### ¿Cuándo se ejecuta?

El GC no corre constantemente (sería demasiado lento). Se activa cuando:

1. Se intenta asignar memoria y no hay espacio libre suficiente.
2. El programador lo solicita explícitamente con `GCRUN`.
3. El volumen de objetos viejos supera un umbral configurable (`GCCONFIG`).

### ¿Tiene coste?

Sí. Mientras el GC analiza qué objetos están vivos, el programa hace una pequeña pausa. VestaVM minimiza esto con un diseño **generacional** (ver más abajo).

---

## ¿Qué es la gestión manual?

En la gestión manual el programador pide memoria con `ALLOC` y la devuelve con `FREE` cuando ya no la necesita. Es más rápido porque no hay análisis automático, pero requiere disciplina: olvidar un `FREE` produce una fuga; hacer `FREE` dos veces sobre el mismo bloque produce corrupción.

Este modo es imprescindible cuando el programa necesita pasar punteros de memoria a funciones externas escritas en C (llamadas **FFI** — *Foreign Function Interface*), porque esas funciones esperan la dirección real de un bloque de bytes en la RAM del host, no un identificador abstracto.

---

## Los dos sistemas de VestaVM

VestaVM separa los dos enfoques en componentes distintos, uno por cada proceso VM:

```
ProcessVM
  ├── GcHeap        -- GC automatico generacional  (objetos OOP, ciclo de vida automatico)
  └── RawAllocator  -- Allocator manual             (buffers FFI, control explicito)
```

Cada proceso tiene sus propias instancias. No se comparten entre procesos, lo que significa que el GC de un proceso nunca bloquea a otro.

### Resumen comparativo

| Característica           | GcHeap (generacional)                | RawAllocator (manual)                 |
| :----------------------- | :----------------------------------- | :------------------------------------ |
| Instrucción de reserva   | `NEWOBJ size`  → `GcHandle`          | `ALLOC size`  → ptr host real         |
| Instrucción de liberación | `DROP handle` (opcional)            | `FREE reg` (obligatorio)              |
| GC automático            | Sí — minor + major                   | No                                    |
| Tipo de retorno          | Handle opaco (no es un puntero)      | Puntero host real (`uint64_t`)        |
| Compatible con FFI       | No                                   | Sí                                    |
| Uso ideal                | Objetos OOP, grafos, closures        | Buffers C, strings, llamadas nativas  |

---

## ¿Qué es un GC generacional?

La observación clave detrás del GC de VestaVM es la **hipótesis generacional**:

> La mayoría de los objetos mueren jóvenes.

Es decir, la mayor parte de los objetos que un programa crea tienen una vida muy corta: se crean para hacer un cálculo, se usan y ya no se necesitan. Solo una minoría vive durante mucho tiempo (configuración global, cachés, estructuras persistentes).

Aprovechando esto, el heap se divide en dos zonas:

```
+---------------------------+------------------------------------+
|         Nursery           |             OldGen                 |
|   (objetos jovenes)       |   (objetos que sobrevivieron)      |
|   asignacion rapida O(1)  |   mark-and-sweep cuando se llena   |
+---------------------------+------------------------------------+
```

- **Nursery**: zona pequeña y rápida donde nacen todos los objetos. Limpiarla es barato porque casi todo está muerto; no hace falta analizar los objetos supervivientes en detalle.
- **OldGen**: zona donde van los objetos que sobrevivieron al menos un ciclo de limpieza de la Nursery. Se limpia con un algoritmo más completo pero se activa con mucha menos frecuencia.

Resultado: **pausas cortas y frecuentes** (Nursery) en lugar de una pausa larga y rara (todo el heap).

---

## ¿Qué es un handle?

En el GcHeap, el programador nunca recibe la dirección real del objeto en memoria. En su lugar recibe un **handle**: un número entero opaco que actúa como identificador.

```
Bytecode:   NEWOBJ 64   →  R0 = 3       (handle, no dirección)
                             ↓
HandleTable:   [3]  →  0x7f3a0000       (dirección real, interna al GC)
```

¿Por qué? Porque el GC puede necesitar **mover** objetos (por ejemplo, al compactar el heap o al copiar un objeto joven a OldGen). Si el bytecode guardase la dirección directa y el GC mueve el objeto, esa dirección quedaría inválida. Con handles, solo hay que actualizar la entrada en la `HandleTable`; el bytecode no cambia nada.

En el `RawAllocator` no existe este problema porque el bloque nunca se mueve: la dirección retornada es permanente mientras no se llame a `FREE` o `REALLOC`.

---

## Opcodes de referencia rápida

### GcHeap

| Instrucción          | Opcode1 | Opcode2 | Descripción                              |
| :------------------- | :-----: | :-----: | :--------------------------------------- |
| `NEWOBJ size`        |  `0x00` |  `0xA0` | Crea objeto; retorna `GcHandle` en `R0`  |
| `GCRUN`              |  `0x00` |  `0xA1` | Fuerza un ciclo de GC ahora              |
| `GCCONFIG threshold` |  `0x00` |  `0xA2` | Ajusta el umbral de OldGen               |
| `DROP handle`        |  `0x00` |  `0xA3` | Libera el handle (el GC recogerá el objeto) |

### RawAllocator

| Instrucción         | Opcode1 | Opcode2 | Descripción                              |
| :------------------ | :-----: | :-----: | :--------------------------------------- |
| `ALLOC size`        |  `0x00` |  `0xB0` | Reserva bytes; retorna ptr real en `R0`  |
| `FREE reg`          |  `0x00` |  `0xB1` | Libera el bloque en `reg` inmediatamente |
| `REALLOC reg, size` |  `0x00` |  `0xB2` | Redimensiona; nuevo ptr en `R0`          |

---

## Documentación detallada

- [[Generacional (para objetos OOP)]] — Cómo funciona el GcHeap internamente: Nursery, minor GC, major GC, tri-color mark, write barrier, handles, estadísticas.
- [[Allocator crudo para FFI  y memoria manual]] — Cómo funciona el RawAllocator: contiguidad, FFI, ALLOC/FREE/REALLOC, estadísticas.

Ver también: [[VmInstance]], [[NativeCall (CallN)]], [[VmManager]]
