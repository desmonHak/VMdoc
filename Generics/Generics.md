# Generics en VestaVM

## Vision general

VestaVM implementa genericos mediante **monomorphization dinamica con reificacion**.
A diferencia de la borrado de tipos (type erasure), cada especializacion genera una
`ClassInfo` independiente en tiempo de ejecucion que contiene los tipos concretos
como metadatos. Esto permite reflexion completa sobre los parametros de tipo.

---

## Instruccion SPECIALIZE (0x3A)

### Descripcion

`SPECIALIZE r_dst, r_class, r_types, count`

Instancia una clase generica en tiempo de ejecucion. Si ya existe una especializacion
con los mismos tipos concretos, devuelve el puntero cacheado (O(1) lookup por clave de nombre).

### Codificacion binaria (FIXED_4)

```
[0x00][0x3A][byte2][byte3]
  byte2 = (r_dst << 4) | r_class
  byte3 = (r_types << 4) | count
```

- `r_class`  = registro con el host-pointer a la `ClassInfo` generica (`CLASS_FLAG_GENERIC = 0x10`)
- `r_types`  = registro con el VM address de un array de `count` punteros a `ClassInfo`
- `count`    = numero de parametros de tipo (0..15, cabe en 4 bits)
- `r_dst`    = registro destino, recibe el `ClassInfo*` de la especializacion

### Semantica

1. Construye la clave de cache: `"ClassName<T1.name,T2.name,...>"` (formato sin espacios).
2. Busca en `Loader::generic_cache_` (tabla hash por clave de cadena).
3. Si existe: devuelve el `ClassInfo*` cacheado directamente.
4. Si no existe:
   a. Clona la `ClassInfo` generica (nombre, campos, metodos, parametros).
   b. Para cada campo con `is_type_param=true`, sustituye `type_class` por el tipo concreto.
   c. Limpia `CLASS_FLAG_GENERIC` en el clon.
   d. Establece `generic_parent` apuntando a la `ClassInfo` original.
   e. Rellena `type_params[i].concrete` con los `ClassInfo*` concretos.
   f. Almacena en la cache.
5. Escribe el puntero en `r_dst`.

---

## Estructuras de soporte

### GenericParam (include/loader/oop_types.h)

```cpp
struct GenericParam {
    const char *name;       // nombre del parametro (p.ej. "T", "K", "V")
    ClassInfo  *constraint; // bound del parametro (nullptr = sin restriccion)
    ClassInfo  *concrete;   // tipo concreto en una especializacion (nullptr en plantilla)
};
```

La separacion `constraint` / `concrete` es deliberada:
- `constraint` es el tipo del bound (`T extends Comparable`) — no se modifica al especializar.
- `concrete` es el tipo real instanciado — se rellena por `specialize_class()`.

### Campos genericos en FieldInfo

```cpp
typedef struct FieldInfo {
    // ... campos normales ...
    bool        is_type_param;   // true si el tipo del campo es un parametro T
    uint16_t    type_param_idx;  // indice en ClassInfo::type_params
    uint8_t     _field_pad[4];   // relleno de alineacion
} FieldInfo;
```

Cuando `is_type_param = true`, el campo no tiene un tipo estatico; `specialize_class()`
lo resuelve a `type_params[type_param_idx].concrete` durante la monomorphization.

### Flags de ClassInfo relevantes

| Flag                | Valor  | Significado                                    |
| :------------------ | :----- | :--------------------------------------------- |
| `CLASS_FLAG_GENERIC`| `0x10` | Clase plantilla, no instanciable directamente  |

---

## Ciclo de vida de la especializacion

```
ClassGeneric<T>  (CLASS_FLAG_GENERIC=1, concrete=nullptr)
      |
      | specialize(ClassInt)
      v
ClassGeneric<ClassInt>  (CLASS_FLAG_GENERIC=0, generic_parent -> ClassGeneric<T>)
      |
      | [cacheada en Loader::generic_cache_]
      v
reutilizada en llamadas futuras con el mismo tipo
```

---

## Ejemplo de uso en bytecode

```asm
; r1 = ClassInfo* de la clase generica (CLASS_FLAG_GENERIC=1)
; r2 = ClassInfo* del tipo concreto "int"
; r4 = tipos_array (VM addr de un array de 1 puntero: [r2])

; Escribir r2 en el array de tipos (en VM memory)
mov   r14, r4
xchg  cur0, r14
writecur cur0, r2          ; tipos_array[0] = ClassInt

; Especializar: r9 = ClassGeneric<ClassInt>
specialize r9, r1, r4, 1

; Segunda llamada con el mismo tipo -> cache hit
specialize r10, r1, r4, 1 ; r10 == r9

; Verificar que r9 == r10 (cache funciona)
cmp   r9, r10
jmp.jne fallo
mov   r0, 1
hlt
fallo:
mov   r0, 0
hlt
```

Ver: `examples_codes_vm/ejemplo_generics.vel`

---

## Almacenamiento de especializaciones en el Loader

El `Loader` gestiona la vida de los metadatos clonados mediante vectores de
`unique_ptr`:

```cpp
// include/loader/loader.h (privado)
std::vector<std::unique_ptr<char[]>>              generic_store_names_;
std::vector<std::unique_ptr<loader::GenericParam[]>> generic_store_params_;
std::vector<std::unique_ptr<loader::FieldInfo[]>>    generic_store_fields_;
std::vector<std::unique_ptr<loader::MethodInfo[]>>   generic_store_methods_;
```

Estos vectores evitan la gestion manual de `new/delete` y garantizan que los
punteros almacenados en `ClassInfo` sean validos durante toda la vida del `Loader`.

---

## Reflexion sobre genericos

Dado que la especializacion es reificada (no borrada), es posible obtener en
tiempo de ejecucion los tipos concretos de una especializacion:

```cpp
// pseudo-C++ de un HLL compilando a VestaVM bytecode:
ClassInfo *spec = ...; // puntero a ClassGeneric<ClassInt>
spec->type_param_count;          // 1
spec->type_params[0].concrete;   // -> ClassInt
spec->generic_parent;            // -> ClassGeneric<T>
```

El HLL puede exponer esto como `typeof(obj).genericArgs[0]` en su reflexion.
