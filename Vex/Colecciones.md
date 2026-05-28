# Colecciones en Vex

Vex trata ocho tipos de coleccion como **tipos primitivos del lenguaje** (keywords). No
son clases genericas en el heap GC; son handles opacos a estructuras nativas del plugin
`stdlib/native/collections/vesta_collections`.

El dispatch de metodos es CALLN directo (sin vtable, sin CALLVIRT). Las colecciones se
**liberan automaticamente** al salir del scope donde fueron declaradas.

---

## Tipos de coleccion

| Tipo | Estructura subyacente | Operaciones principales |
| :--------- | :-------------------------- | :--------------------------------------- |
| `ArrayList`| Array dinamico con grow 2x | push, pop, get, set, remove_at, size |
| `HashMap` | SwissTable (wyhash64 + SSE2)| put, get, remove, contains_key, size |
| `HashSet` | Wrapper sobre HashMap | add, contains, remove, size |
| `Queue` | Ring buffer potencia de 2 | push (enqueue), pop (dequeue), peek, size|
| `Deque` | Ring buffer doble extremo | push_front, push_back, pop_front, pop_back|
| `TreeMap` | Red-Black tree O(log n) | put, get, remove, first, last, floor, ceiling |
| `TreeSet` | Wrapper sobre TreeMap | add, contains, remove, first, last |
| `Stack` | Array dinamico LIFO | push, pop, peek, size |

---

## Sintaxis basica

```java
// Crear con capacidad inicial:
ArrayList lista = arraylist(16); // 16 elementos iniciales
HashMap mapa = hashmap(32); // 32 slots iniciales
HashSet conj = hashset(8);
Queue cola = queue(8);
Deque deque = deque(8);
TreeMap arbol = treemap(0); // treemap no usa capacidad inicial
TreeSet set = treeset(0);
Stack pila = stack(8);

// Las colecciones se liberan automaticamente al salir del scope.
// No hay free() manual.
```

---

## ArrayList

```java
ArrayList xs = arraylist(8);

// Agregar elementos:
xs.push(10);
xs.push(20);
xs.push(30);

// Acceder por indice (0-indexed):
i64 primer = xs.get(0); // 10
xs.set(1, 99); // cambiar el elemento en indice 1

// Insertar en posicion (desplaza el resto):
xs.insert(1, 55); // xs = [10, 55, 99, 30]

// Eliminar por indice:
xs.remove_at(2); // xs = [10, 55, 30]

// Quitar el ultimo elemento:
i64 ultimo = xs.pop(); // ultimo = 30, xs = [10, 55]

// Tamanio:
i32 n = xs.size(); // 2

// Vaciar:
xs.clear();

println("size = ${xs.size()}"); // 0
```

### GC-aware: cuando los elementos son GcHandles

Si los elementos son handles GC (por ejemplo, instancias de clase), usar las variantes
`*_gc` que notifican al GC:

```java
ArrayList animales = arraylist(8);

Animal a = new Animal("Rex", 5);
animales.push_gc(a); // registra el handle en el GC para que no se recolecte

// ... mas adelante:
Animal recuperado = animales.get_gc(0); // devuelve handle GC
animales.pop_gc(); // libera el handle GC del GC
```

---

## HashMap

```java
HashMap m = hashmap(16);

// Insertar clave-valor (i64 -> i64):
m.put(1, 100);
m.put(2, 200);
m.put(3, 300);

// Obtener valor por clave:
i64 val = m.get(2); // 200
bool existe = m.contains_key(3); // true

// Eliminar clave:
m.remove(1);

// Iterar:
i32 n = m.size();

// Obtener todas las claves y valores:
ArrayList claves = m.keys();
ArrayList valores = m.values();
```

El HashMap usa **wyhash64** (una multiplicacion de 128 bits + XOR) y **sondeo por grupos
de 16 slots con SSE2** para maxima velocidad en CPU modernas. Factor de carga 87.5%.

---

## HashSet

```java
HashSet s = hashset(8);

s.add(10);
s.add(20);
s.add(10); // elemento duplicado (no se agrega dos veces)

bool tiene = s.contains(20); // true
bool no = s.contains(99); // false

s.remove(10);
i32 n = s.size(); // 1 (solo 20)
```

---

## Queue (FIFO)

```java
Queue q = queue(4);

q.push(1); // enqueue
q.push(2);
q.push(3);

i64 primero = q.pop(); // dequeue: 1 (FIFO)
i64 segundo = q.pop(); // 2
i64 cabeza = q.peek(); // mira sin extraer: 3

println("size = ${q.size()}"); // 1
```

---

## Deque (cola doble)

```java
Deque d = deque(8);

d.push_back(10);
d.push_back(20);
d.push_front(5); // d = [5, 10, 20]

i64 frente = d.pop_front(); // 5, d = [10, 20]
i64 atras = d.pop_back(); // 20, d = [10]
i64 pico = d.peek_front(); // 10 (sin extraer)
```

---

## TreeMap (mapa ordenado)

```java
TreeMap t = treemap(0);

t.put(30, 300);
t.put(10, 100);
t.put(20, 200);

// Acceso O(log n):
i64 val = t.get(20); // 200
bool ok = t.contains_key(10); // true

// Extremos en O(log n):
i64 min_key = t.first_key(); // 10
i64 max_key = t.last_key(); // 30

// Operaciones de rango:
i64 floor = t.floor_key(25); // 20 (mayor clave <= 25)
i64 ceiling = t.ceiling_key(15); // 20 (menor clave >= 15)

// Eliminar:
t.remove(20);
```

---

## TreeSet (conjunto ordenado)

```java
TreeSet s = treeset(0);

s.add(30);
s.add(10);
s.add(20);

i64 min = s.first(); // 10
i64 max = s.last(); // 30

bool tiene = s.contains(20); // true
s.remove(20);
```

---

## Stack (pila LIFO)

```java
Stack st = stack(8);

st.push(1);
st.push(2);
st.push(3);

i64 top = st.peek(); // 3 (sin extraer)
i64 val = st.pop(); // 3
i64 n = st.size(); // 2
```

---

## Operaciones de cadena con colecciones

El plugin `vesta_collections` incluye operaciones de cadena implementadas con `memmem`
(deteccion SIMD interna de libc):

```java
// Buscar subcadena en un buffer:
char* pos = vstr_indexof(haystack_ptr, h_len, needle_ptr, n_len);

// Verificar prefijos/sufijos:
bool empieza = vstr_starts_with(s_ptr, s_len, prefix_ptr, p_len);
bool termina = vstr_ends_with(s_ptr, s_len, suffix_ptr, suf_len);

// Convertir a mayusculas/minusculas (ASCII):
vstr_lower_inplace(buf_ptr, len);
vstr_upper_inplace(buf_ptr, len);

// Split por delimitador (devuelve ArrayList de (offset<<32)|len):
ArrayList partes = arraylist(8);
i32 n = vstr_split_offsets(texto_ptr, t_len, delim_ptr, d_len, partes);
```

---

## Operaciones de array de enteros

```java
// Ordenar array de u64 ascendente:
varr_sort_u64(arr_ptr, len);

// Ordenar descendente:
varr_sort_u64_desc(arr_ptr, len);

// Busqueda binaria branchless (~20-40% mas rapido que busqueda con saltos):
i64 idx = varr_bsearch_u64(arr_ptr, len, valor);

// Busqueda lineal:
i64 idx2 = varr_indexof_u64(arr_ptr, len, valor);

// Invertir:
varr_reverse_u64(arr_ptr, len);
```

---

## Liberacion automatica y escape

Las colecciones se liberan automaticamente al salir del scope. Si se necesita pasar
una coleccion fuera del scope donde se declaro, el compilador detecta el escape y
omite el cleanup:

```java
ArrayList construir() {
    ArrayList xs = arraylist(8);
    xs.push(1); xs.push(2); xs.push(3);
    return xs; // xs escapa: no se libera al salir de construir()
} // el llamante es el dueno

ArrayList resultado = construir();
println("${resultado.size()}"); // 3
// resultado se libera aqui
```

Para liberacion adelantada dentro de loops:

```java
for (i32 i = 0; i < 1000; i++) {
    ArrayList tmp = arraylist(4);
    // ... usar tmp ...
    dispose(tmp); // libera ahora en lugar de esperar al exit del for
}
```

---

## Declaracion extern para variantes _gc

```java
extern import "stdlib/native/collections/vesta_collections";

// Declarar las variantes GC-aware que necesites:
extern "stdlib/native/collections/vesta_collections" {
    fn vcol_alist_push_gc(handle: i64, gc_handle: i64) -> void;
    fn vcol_alist_get_gc(handle: i64, idx: i32) -> i64;
    fn vcol_alist_pop_gc(handle: i64) -> void;
}
```

---

Ver tambien: [[TiposDatos]], [[FFI]], [[SetInstruccionesVM/GC/GC]]
