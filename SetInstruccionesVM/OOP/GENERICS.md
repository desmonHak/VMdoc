# SPECIALIZE - Monomorphization de clases genericas en tiempo de ejecucion

La instruccion **SPECIALIZE** instancia una clase generica con tipos concretos
en tiempo de ejecucion.  El resultado es una `ClassInfo*` especializada, lista
para usarse en NEWOBJRAW, CALLVIRT, INSTANCEOF, etc., exactamente igual que
cualquier otra clase no generica.

| Instruccion  | opcode0 | opcode1 | Modo | Tamano  | Descripcion                                          |
| :----------: | :-----: | :-----: | :--: | :-----: | :--------------------------------------------------- |
| `specialize` |  0x00   |  0x3A   | REG  | 4 bytes | Crear/recuperar ClassInfo especializada en el Loader |

Implementacion: `src/runtime/exec_instruction_generic.cpp`

---

## Concepto accesible: que es una clase generica

Imagina una caja que puede guardar cualquier cosa.  En el codigo fuente
escribes `Caja<T>`, donde `T` es el tipo del contenido.  Cuando quieres una
caja de enteros usas `Caja<int>`; cuando quieres una caja de texto usas
`Caja<str>`.

SPECIALIZE es la instruccion que le dice a la VM "necesito una Caja de int
concretamente" y la VM te devuelve una descripcion de clase adaptada para eso.
Si ya habia creado antes una Caja de int, te devuelve la misma (cache);
si no, la crea en ese momento.

Este enfoque se llama **monomorphization**: para cada combinacion de tipos
se genera una clase distinta, igual que en C++ con los templates.  No hay
overhead en tiempo de ejecucion por tipo generico: el codigo especializado
trabaja con tamanos y offsets exactos.

---

## Codificacion binaria (FIXED_4, 4 bytes)

```
+--------+--------+--------+--------+
| 0x00   | 0x3A   | byte2  | byte3  |
+--------+--------+--------+--------+
  byte0    byte1    byte2    byte3

byte2 = (r_dst << 4) | r_class
byte3 = (r_types << 4) | count
```

- `r_dst`   = registro destino (recibira la ClassInfo* especializada)
- `r_class` = registro con la ClassInfo* de la clase generica
- `r_types` = registro con la direccion del array de ClassInfo* (tipos concretos)
- `count`   = numero de parametros de tipo (0-15, cabe en 4 bits)

La instruccion reutiliza la funcion de decodificacion `decode_instr_jumptable`
porque tiene el mismo formato de tres nibbles en byte2/byte3.

---

## Operandos en sintaxis .vel

```c
specialize r_dst, r_class, r_types, count
```

Ejemplo:

```c
specialize r9, r1, r4, 1    // ClassGeneric<ClassInt> -> r9
```

Los cuatro campos se mapean al byte2 y byte3 de la codificacion FIXED_4.
El maximo de `count` es 15 por la restriccion de 4 bits.

---

## Semantica

1. Leer `r_class`: puntero a ClassInfo generica (debe tener `CLASS_FLAG_GENERIC`
   en su campo `flags`).
2. Leer `r_types`: direccion en memoria VM de un array de `count` punteros a
   ClassInfo (los tipos concretos).
3. Construir la clave de cache: `"NombreClase<Tipo0,Tipo1,...>"` usando los
   nombres de cada ClassInfo del array.
4. Buscar la clave en `generic_cache_` del Loader.
   - **Cache hit**: devolver el puntero ya almacenado.  Sin aloc, O(1).
   - **Cache miss**: clonar la ClassInfo generica, rellenar `type_params` con
     los punteros concretos, limpiar `CLASS_FLAG_GENERIC`, establecer
     `generic_parent` apuntando a la clase origen, registrar en `generic_cache_`
     y `generic_store_` (para lifetime management).
5. Escribir el resultado en `r_dst`.

---

## Campos de ClassInfo relevantes para genericos

```
offset  0-15  name          (stringx, 16B) -- nombre de la especializacion
offset 16-23  flags         (uint64_t)     -- CLASS_FLAG_GENERIC = 0x10
offset 24-27  instance_size (uint32_t)     -- tamano de instancia
offset 32-39  supers*       (8B)           -- array de superclases
offset 40-47  super_count   (8B)
offset 48-55  type_params*  (ClassInfo**)  -- parametros de tipo concretos
offset 56-63  type_param_count (size_t)
offset 64-71  generic_parent* (ClassInfo*) -- clase generica origen
```

Tras SPECIALIZE, la especializacion tiene:
- `flags` sin `CLASS_FLAG_GENERIC` (ya no es generica, es concreta).
- `type_params` apuntando a los ClassInfo* concretos.
- `generic_parent` apuntando a la clase generica original.

---

## Gestion de memoria

Las especializaciones las gestiona el Loader:

```cpp
// En Loader:
std::unordered_map<std::string, loader::ClassInfo*> generic_cache_;
std::vector<std::unique_ptr<loader::ClassInfo>> generic_store_;
```

`generic_store_` es el propietario real (via unique_ptr) y garantiza que la
memoria se libera al destruir el Loader.  `generic_cache_` almacena punteros
crudos como indice de busqueda rapida.

El bytecode **no debe liberar** manualmente los punteros obtenidos con
SPECIALIZE; su vida util esta ligada al Loader.

---

## Interaccion con otras instrucciones OOP

Una vez obtenida la ClassInfo especializada, se puede usar en cualquier
instruccion OOP que acepte una ClassInfo*:

```c
// Obtener la especializacion
specialize r9, r1, r4, 1        // r9 = ClassInfo* de Lista<int>

// Crear un objeto de ese tipo
mov        r6, 48
newobjraw  r9, r6               // R0 = ObjectHeader* del nuevo objeto

// Comprobar si otro objeto es de ese tipo
instanceof r4, r9               // R0 = 1 si r4 es instancia de Lista<int>

// Despacho dinamico a un metodo de la especializacion
callvirt   r4, 0                // vtable[0] de la clase especializada
```

---

## Ejemplo en .vel

```c
// Paso 1: preparar el array de tipos (1 elemento: ClassInt)
mov   r6, 8
alloc r6
mov   r4, r0                    // r4 = types_array (8 bytes)

mov   r14, r4
xchg  cur0, r14
writecur cur0, r2               // types_array[0] = ClassInt (puntero en r2)

// Paso 2: especializar
// r1 = ClassInfo* de la clase generica (CLASS_FLAG_GENERIC en flags)
// r4 = types_array con un puntero a ClassInt
// r9 = destino: recibe la ClassInfo* especializada
specialize r9, r1, r4, 1       // r9 = ClassGeneric<ClassInt>

// Paso 3: crear instancia de la clase especializada
mov       r6, 32
newobjraw r9, r6               // R0 = puntero al nuevo objeto
```

Vease `examples_codes_vm/ejemplo_generics.vel` para el ejemplo ejecutable
completo que demuestra cache hit, cache miss y verificacion de punteros.

---

## Diferencias con la monomorphization en el emitter

El plan de implementacion contempla dos niveles de monomorphization:

| Nivel           | Cuando ocurre         | Como funciona                         | Overhead    |
| :-------------- | :-------------------- | :------------------------------------ | :---------- |
| Emitter (emit)  | En tiempo de compilacion | El emitter clona el AST por combinacion de tipos | Ninguno en ejecucion |
| Runtime (specialize) | En tiempo de ejecucion | El Loader clona ClassInfo al primer uso, cachea el resto | O(1) tras primera ejecucion |

SPECIALIZE cubre el caso de uso donde los tipos no se conocen hasta la
ejecucion (p.ej. tipos proporcionados por el usuario, cargados dinamicamente,
o construidos mediante reflexion).

---

## Consideraciones de rendimiento

- La primera llamada con una combinacion nueva de tipos requiere clonar la
  ClassInfo y construir la clave de cache: costo O(k) donde k es el numero de
  parametros de tipo.
- Las llamadas siguientes con la misma combinacion son O(1) (lookup en el
  mapa hash).
- No hay overhead por boxing ni por type erasure: la ClassInfo especializada
  tiene `instance_size`, offsets de campos y entradas de vtable exactas.
- El numero maximo de tipos por llamada es 15 (limitacion de 4 bits en la
  codificacion FIXED_4).  Para mas parametros, usar especializaciones anidadas.

---

## Errores comunes

1. **r_class no tiene CLASS_FLAG_GENERIC**: el Loader puede tratar la clase
   como no generica y devolver una copia identica o un error.  Asegurarse de
   que el bit 0x10 esta activo en ClassInfo.flags.

2. **types_array con punteros invalidos**: si alguno de los count punteros
   en el array apunta a memoria invalida, la construccion de la clave fallara
   o producira un nombre corrupto.  Verificar que todos los punteros a ClassInfo
   del array son validos antes de llamar SPECIALIZE.

3. **count = 0**: equivale a no especializar.  El Loader puede devolver la
   clase generica original o una copia sin type_params.  Evitar este caso.

4. **Usar el puntero despues de destruir el Loader**: los punteros devueltos
   por SPECIALIZE solo son validos mientras el Loader vive.
