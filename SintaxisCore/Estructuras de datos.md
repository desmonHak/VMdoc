# Estructuras de datos en Vesta

Una **estructura de datos** es una forma de organizar y almacenar multiples valores
relacionados en memoria. En lugar de tener 100 variables separadas para guardar 100
numeros, usas un array. En lugar de buscar uno por uno, usas un mapa que te da la
respuesta en tiempo constante.

VestaVM provee soporte nativo para las estructuras mas comunes a traves de la
libreria estandar y del sistema de clases OOP.

---

## Arrays

Un **array** es una secuencia de elementos del mismo tipo almacenados de forma contigua
en memoria. Es la estructura mas simple y directa.

**Analogia:** una caja de huevos tiene espacios numerados del 0 al 11. Puedes acceder
directamente al huevo numero 5 sin pasar por los anteriores.

```c
// Declarar e inicializar un array de 5 enteros
uint64_t numeros[5] = { 10, 20, 30, 40, 50 };

// Acceder por indice (empezando en 0)
uint64_t primero = numeros[0];  // 10
uint64_t tercero = numeros[2];  // 30

// Modificar un elemento
numeros[1] = 99;  // ahora el array es { 10, 99, 30, 40, 50 }

// Recorrer con for
for (uint8_t i = 0; i < 5; i++) {
    print(numeros[i]);
}
```

### Array dinamico

Para arrays cuyo tamano no se conoce en tiempo de compilacion:

```c
// Allocar un array de tamano variable
uint64_t n = 100;
uint64_t[] arr = new uint64_t[n];  // array de 100 enteros en el heap GC

// Liberar (si no se usa GC automatico)
// El GC lo libera automaticamente cuando ya no hay referencias
```

### Complejidades del array

| Operacion            | Complejidad | Descripcion                          |
| :------------------- | :---------: | :----------------------------------- |
| Acceso por indice    | O(1)        | Directo por posicion en memoria      |
| Busqueda lineal      | O(n)        | Comparar elemento por elemento       |
| Insercion al final   | O(1)        | Si hay espacio disponible            |
| Insercion al inicio  | O(n)        | Hay que desplazar todos los elementos|

---

## Listas enlazadas

Una **lista enlazada** es una secuencia donde cada elemento (nodo) guarda su valor
y un puntero al siguiente elemento. No es contigua en memoria: cada nodo puede estar
en cualquier lugar del heap.

**Analogia:** como una cadena de personas donde cada una sabe quien sigue despues de
ella, pero no quien esta en la posicion 7 sin seguir la cadena desde el principio.

```c
// Definicion de un nodo de lista enlazada
struct Nodo {
    uint64_t valor;
    Nodo*    siguiente;   // puntero al proximo nodo (null si es el ultimo)
}

// Crear una lista: 10 -> 20 -> 30 -> null
Nodo* n3 = new Nodo { 30, null };
Nodo* n2 = new Nodo { 20, n3   };
Nodo* n1 = new Nodo { 10, n2   };  // n1 es la cabeza de la lista

// Recorrer la lista
Nodo* actual = n1;
while (actual != null) {
    print(actual.valor);
    actual = actual.siguiente;
}
```

### Complejidades de la lista enlazada

| Operacion             | Complejidad | Descripcion                              |
| :-------------------- | :---------: | :--------------------------------------- |
| Acceso por indice     | O(n)        | Hay que seguir los punteros uno a uno    |
| Insercion al inicio   | O(1)        | Solo cambiar el puntero cabeza           |
| Insercion al final    | O(n)        | Hay que llegar hasta el ultimo nodo      |
| Insercion en medio    | O(1)*       | Si ya tienes el puntero al nodo previo   |
| Busqueda por valor    | O(n)        | Comparar cada nodo                       |

*Una vez que tienes el puntero al nodo anterior, insertar es O(1).

---

## HashMaps (Mapas / Diccionarios)

Un **hashmap** (o mapa, o diccionario) asocia **claves** a **valores**. Dada una clave,
te devuelve el valor asociado en tiempo O(1) de media.

**Analogia:** es como un diccionario de papel. Dada la palabra (clave) "gato",
encuentras directamente la definicion (valor) sin leer todas las paginas.

```c
// Crear un mapa que asocia String -> uint64_t
HashMap<String, uint64_t> edades = new HashMap();

// Insertar pares clave-valor
edades.put("Ana",   25);
edades.put("Luis",  30);
edades.put("Maria", 28);

// Obtener un valor por clave
uint64_t edad_ana = edades.get("Ana");   // 25

// Comprobar si existe una clave
if (edades.containsKey("Luis")) {
    print("Luis esta en el mapa");
}

// Eliminar una entrada
edades.remove("Luis");

// Recorrer todas las entradas
for (String clave : edades.keys()) {
    print(clave);
    print(edades.get(clave));
}
```

### Como funciona internamente

Cuando insertas `edades.put("Ana", 25)`:
1. Se calcula un **hash** de la clave `"Ana"` (un numero entero).
2. Ese numero determina en que "cubo" (bucket) del array interno se guarda.
3. Al buscar con `edades.get("Ana")`, se recalcula el hash y se va directamente
   al cubo correcto.

Por eso el acceso es O(1): no hay que comparar con todas las claves.

### Complejidades del hashmap

| Operacion    | Complejidad media | Descripcion                              |
| :----------- | :---------------: | :--------------------------------------- |
| get(k)       | O(1)              | Calcular hash y acceder al bucket        |
| put(k, v)    | O(1)              | Calcular hash e insertar                 |
| remove(k)    | O(1)              | Calcular hash y eliminar                 |
| containsKey  | O(1)              | Calcular hash y comprobar                |
| Iteracion    | O(n)              | Recorrer todos los buckets               |

---

## Tabla comparativa

| Estructura      | Acceso | Busqueda | Insercion | Memoria      | Uso tipico                        |
| :-------------- | :----: | :------: | :-------: | :----------- | :-------------------------------- |
| Array           | O(1)   | O(n)     | O(n)*     | Contigua     | Secuencias de tamano fijo         |
| Lista enlazada  | O(n)   | O(n)     | O(1)**    | Dispersa     | Inserciones/eliminaciones frecuentes |
| HashMap         | O(1)   | O(1)     | O(1)      | Extra para buckets | Busqueda rapida por clave   |

*Insercion en el array requiere desplazar elementos.
**Insercion en lista es O(1) si se tiene el puntero al lugar de insercion.

---

## Nivel de bytecode

A nivel de bytecode VestaVM, todas estas estructuras son objetos en el heap GC
(a menos que se usen con el allocator raw para FFI). El acceso a campos usa
instrucciones de memoria con offsets calculados por el compilador:

```c
// Acceso al campo 0 de un nodo en bytecode:
// readcur cur0, r1q   // leer uint64_t en cur0 (apunta al nodo) -> r1
```

---

Ver tambien:
- [[POO.md]] - clases y objetos
- [[Estructuras (struct).md]] - structs de bajo nivel
- [[SetInstruccionesVM/GC/Generacional (para objetos OOP).md]] - gestion de memoria
- [[SetInstruccionesVM/JUMPTABLE_TYPESWITCH.md]] - despacho eficiente
