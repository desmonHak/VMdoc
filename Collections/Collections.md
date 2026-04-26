# Colecciones Nativas de VestaVM

Plugin: `stdlib/native/collections/vesta_collections`
Fuente: `stdlib/native/collections/vesta_collections.c`

## Vision general

El plugin `vesta_collections` proporciona cuatro tipos de coleccion nativas que
viven en memoria del host (gestionadas con `malloc/realloc/free`). Los handles
son punteros de host casteados a `uint64_t` y se pasan entre el bytecode VM
y las funciones nativas usando la convencion de llamada estandar de VestaVM.

> **Nota de GC:** Los `GcHandle` almacenados dentro de colecciones nativas NO
> son raices del recolector de basura. Si el unico origen de un `GcHandle` es
> una coleccion nativa, el GC puede recolectar el objeto. El bytecode debe
> mantener una copia del handle en un registro VM o en memoria VM.

### Convencion de llamada

```
r1..r12 = argumentos
r15     = argc
r0      = valor de retorno
calln @Method("stdlib/native/collections/vesta_collections:funcion")
```

---

## VestaList — array dinamico de GcHandle

Almacena elementos de 32 bits (`uint32_t` por slot). Optimizado para
referencias a objetos GC de VestaVM cuyos handles son de 32 bits.

**Capacidad inicial minima:** 8 elementos.  
**Factor de crecimiento:** x2 mediante `realloc`.

### Funciones

| Funcion                       | Args (r1, r2, r3)    | Retorno                          |
| :---------------------------- | :------------------- | :------------------------------- |
| `vcol_list_new(cap)`          | cap                  | handle, o `0` si falla           |
| `vcol_list_free(h)`           | h                    | —                                |
| `vcol_list_push(h, elem)`     | h, elem              | nuevo size, o `UINT64_MAX` error |
| `vcol_list_pop(h)`            | h                    | ultimo elem, o `UINT64_MAX`      |
| `vcol_list_get(h, idx)`       | h, idx               | elem, o `UINT64_MAX` fuera rango |
| `vcol_list_set(h, idx, elem)` | h, idx, elem         | valor anterior                   |
| `vcol_list_size(h)`           | h                    | numero de elementos              |
| `vcol_list_cap(h)`            | h                    | slots asignados                  |
| `vcol_list_clear(h)`          | h                    | —                                |
| `vcol_list_remove_at(h, idx)` | h, idx               | elem eliminado                   |
| `vcol_list_insert(h, idx, e)` | h, idx, elem         | nuevo size                       |
| `vcol_list_indexof(h, elem)`  | h, elem              | indice, o `UINT64_MAX` no hallado|
| `vcol_list_clone(h)`          | h                    | handle del clon                  |

### Ejemplo basico

```asm
mov  r1, 8
mov  r15, 1
calln @Method("stdlib/native/collections/vesta_collections:vcol_list_new")
mov  r5, r0                ; r5 = handle

mov  r1, r5
mov  r2, 42
mov  r15, 2
calln @Method("stdlib/native/collections/vesta_collections:vcol_list_push")

mov  r1, r5
mov  r2, 0
mov  r15, 2
calln @Method("stdlib/native/collections/vesta_collections:vcol_list_get")
; r0 = 42

mov  r1, r5
mov  r15, 1
calln @Method("stdlib/native/collections/vesta_collections:vcol_list_free")
```

Ver: `examples_codes_vm/ejemplo_collections_list.vel`

---

## VestaArrayList — array dinamico de uint64_t

Almacena valores de 64 bits por slot. Adecuado para valores primitivos, punteros
de host, o mezcla de valores raw y GcHandles de 64 bits.

API identica a VestaList pero con prefijo `vcol_alist_*` y valores de 64 bits.
Adicionalmente expone `vcol_alist_sort(h)` para ordenacion ascendente con `qsort`.

---

## VestaHashMap — tabla hash uint64_t -> uint64_t

Implementacion: direccionamiento abierto, sondeo lineal, factor de carga max 0.75.
Hash de clave: FNV-1a de 64 bits sobre los 8 bytes de la clave.
Capacidad siempre potencia de 2 para usar mascara en lugar de modulo.

**Capacidad inicial minima:** 16 slots.

### Sentinels internos

| Constante        | Valor          | Uso                              |
| :--------------- | :------------- | :------------------------------- |
| `VCOL_NOT_FOUND` | `UINT64_MAX`   | slot vacio en la tabla           |
| `VMAP_DELETED_KEY` | `UINT64_MAX-1` | tombstone para borrado en sondeo |

### Funciones

| Funcion                          | Args              | Retorno                           |
| :------------------------------- | :---------------- | :-------------------------------- |
| `vcol_map_new(cap)`              | cap               | handle, o `0` si falla            |
| `vcol_map_free(h)`               | h                 | —                                 |
| `vcol_map_put(h, key, val)`      | h, key, val       | —                                 |
| `vcol_map_get(h, key)`           | h, key            | valor, o `0` si no existe         |
| `vcol_map_get_or(h, key, def)`   | h, key, default   | valor, o `default` si no existe   |
| `vcol_map_contains(h, key)`      | h, key            | `1` si existe, `0` si no          |
| `vcol_map_remove(h, key)`        | h, key            | valor eliminado, o `UINT64_MAX`   |
| `vcol_map_size(h)`               | h                 | numero de pares                   |
| `vcol_map_clear(h)`              | h                 | —                                 |
| `vcol_map_keys(h)`               | h                 | handle VestaArrayList con claves  |
| `vcol_map_values(h)`             | h                 | handle VestaArrayList con valores |

> `vcol_map_get` devuelve `0` tanto si la clave no existe como si su valor es `0`.
> Usar `vcol_map_contains` primero, o `vcol_map_get_or` con un sentinel propio.

### Algoritmo de rehash

El mapa se rehashea cuando `(size + tombstones + 1) * 4 > capacity * 3`.
Esto garantiza que los tombstones no degraden el rendimiento del sondeo lineal.

Ver: `examples_codes_vm/ejemplo_collections_hashmap.vel`

---

## VestaHashSet — conjunto de uint64_t unicos

Implementado internamente como `VestaHashMap` con valor sentinel `UINT64_MAX-2`.
Proporciona semantica de conjunto: insercion, pertenencia, eliminacion, union,
interseccion y subconjunto.

### Funciones

| Funcion                       | Args         | Retorno                                |
| :---------------------------- | :----------- | :------------------------------------- |
| `vcol_set_new(cap)`           | cap          | handle, o `0` si falla                 |
| `vcol_set_free(h)`            | h            | —                                      |
| `vcol_set_add(h, elem)`       | h, elem      | `1` nuevo, `0` ya existia              |
| `vcol_set_contains(h, elem)`  | h, elem      | `1` si existe, `0` si no               |
| `vcol_set_remove(h, elem)`    | h, elem      | `1` eliminado, `0` no existia          |
| `vcol_set_size(h)`            | h            | numero de elementos                    |
| `vcol_set_clear(h)`           | h            | —                                      |
| `vcol_set_to_list(h)`         | h            | handle VestaArrayList con todos elems  |
| `vcol_set_is_subset(a, b)`    | set_a, set_b | `1` si A subconjunto de B              |
| `vcol_set_union(a, b)`        | set_a, set_b | nuevo handle set (A union B)           |
| `vcol_set_intersect(a, b)`    | set_a, set_b | nuevo handle set (A interseccion B)    |

Ver: `examples_codes_vm/ejemplo_collections_set.vel`

---

## Construccion del plugin

```cmake
# stdlib/native/CMakeLists.txt
add_subdirectory(collections)

# stdlib/native/collections/CMakeLists.txt
include(VestaPlugin)
add_vesta_plugin(vesta_collections
    SOURCES vesta_collections.c
    SDK_DIR "${VESTA_SDK_DIR}"
    C_STANDARD 11)
```

El modulo `VestaPlugin.cmake` configura automaticamente los flags de visibilidad
y la macro `VESTA_PLUGIN_EXPORT` segun la plataforma (Windows `__declspec(dllexport)`
vs GCC/Clang `__attribute__((visibility("default")))`).
