# STATIC_FIELDS — Instrucciones getstatic y setstatic

Las instrucciones `getstatic` y `setstatic` leen y escriben campos estaticos de clases
registradas en el `ClassRegistry`. Los campos estaticos se almacenan en un buffer de
memoria HOST (`ClassInfo::static_data`) separado del heap GC, compartido por todas las
instancias de la clase.

Ambas instrucciones usan el prefijo extendido `[0x00]` y encoding **FIXED_8** (8 bytes).

---

## Tabla resumen

| Instruccion | opcode2 | Encoding | Descripcion |
| :---------- | :-----: | :------- | :------------------------------------------- |
| `getstatic` | 0x60 | FIXED_8 | Leer i64 de `cls->static_data + offset` |
| `setstatic` | 0x61 | FIXED_8 | Escribir i64 a `cls->static_data + offset` |

---

## Encoding FIXED_8

Ambas instrucciones tienen un encoding de 8 bytes:

```
[0x00][opcode2][regs_byte][_pad8][offset_u32_LE (4 bytes)]
```

- `regs_byte`:
 - Para `getstatic`: `(r_class << 4) | r_dst`
 - Para `setstatic`: `(r_class << 4) | r_value`
- `_pad8`: byte de relleno, debe ser 0.
- `offset_u32_LE`: offset en bytes dentro de `ClassInfo::static_data`, codificado en
 little-endian de 4 bytes. El offset es un literal calculado en compile-time por el
 frontend Vesta.

---

## `getstatic` (0x60) — Leer campo estatico

### Encoding detallado

```
byte 0: 0x00
byte 1: 0x60
byte 2: (r_class << 4) | r_dst
byte 3: 0x00 (_pad)
bytes 4-7: offset_u32 en little-endian
```

- `r_class`: registro con el `ClassInfo*` de la clase.
- `r_dst`: registro destino donde se escribe el valor i64 leido.
- `offset`: offset en bytes dentro del bloque `static_data` del campo.

### Comportamiento

```cpp
// Pseudocodigo del exec_instr_getstatic:
ClassInfo* cls = (ClassInfo*)vm->registers[r_class];
if (cls == nullptr || cls->static_data == nullptr) {
    throw_fatal(vm, FATAL_NULL_POINTER, "getstatic: ClassInfo* nulo");
}
int64_t value;
memcpy(&value, cls->static_data + offset, 8);
vm->registers[r_dst] = value;
```

Para tipos mas pequenos que i64 (e.g. i32), el frontend Vesta emite un truncado despues
del load: el valor leido como i64 se enmascara o sign-extiende al tipo logico.

### Ejemplo

```
// Leer Animal.contador (campo estatico u64) en offset 0:
mov r12, @Absolute("data.fc_Animal_params")
findclass r1, r12 // r1 = ClassInfo* de Animal
getstatic r2, r1, 0 // r2 = Animal.contador (leido del static_data[0])
```

En Vesta:

```vex
u64 n = Animal.contador; // baja a: findclass + getstatic offset
```

---

## `setstatic` (0x61) — Escribir campo estatico

### Encoding detallado

```
byte 0: 0x00
byte 1: 0x61
byte 2: (r_class << 4) | r_value
byte 3: 0x00 (_pad)
bytes 4-7: offset_u32 en little-endian
```

- `r_class`: registro con el `ClassInfo*` de la clase.
- `r_value`: registro con el valor i64 a escribir.
- `offset`: offset en bytes dentro del bloque `static_data`.

### Comportamiento

```cpp
// Pseudocodigo del exec_instr_setstatic:
ClassInfo* cls = (ClassInfo*)vm->registers[r_class];
if (cls == nullptr || cls->static_data == nullptr) {
    throw_fatal(vm, FATAL_NULL_POINTER, "setstatic: ClassInfo* nulo");
}
int64_t value = vm->registers[r_value];
memcpy(cls->static_data + offset, &value, 8);
```

Para tipos mas pequenos que i64 (e.g. i32), el frontend Vesta sign-extiende el valor
antes del store si el campo es signed, o lo zero-extiende si es unsigned.

### Ejemplo

```
// Escribir Animal.contador += 1:
getstatic r2, r1, 0 // r2 = Animal.contador
add r2, 1
setstatic r2, r1, 0 // Animal.contador = r2
```

En Vesta:

```vex
Animal.contador += 1; // baja a: findclass + getstatic + add + setstatic
```

---

## Layout de static_data en ClassInfo

El `ClassInfo::static_data` es un buffer de memoria HOST (no VM) alocado una vez durante
`defclass`. Su tamaño es la suma de los campos estaticos declarados con `deffield` con
`is_static = 1`.

```
ClassInfo::static_data
    +--[ offset 0 ]--[ 8 bytes ] campo_estatico_1
    +--[ offset 8 ]--[ 8 bytes ] campo_estatico_2
    +--[ offset 16 ]--[ 4 bytes ] campo_estatico_3 (i32, 4 bytes pero alineado a 8)
    ...
```

El offset de cada campo se calcula en `deffield` de forma acumulativa (igual que los
campos de instancia). El frontend Vesta conoce el offset en compile-time y lo incrusta
directamente en el encoding de `getstatic`/`setstatic` como `offset_u32`.

---

## Cache del ClassInfo*

Para evitar el costo de `findclass` en cada acceso a un campo estatico, el frontend
Vesta reserva un slot de 8 bytes en `static_data` con clave `__class_cache_<Class>`.
El `__module_init` escribe el `ClassInfo*` en ese slot justo despues del `defclass`,
y el helper `__new_<Class>` lo lee en lugar de invocar `findclass`.

Los accesos a campos estaticos desde `main` o funciones normales SIEMPRE invocan
`findclass` inline (busqueda O(1) en el ClassRegistry). El cache solo aplica al
helper `__new_<Class>`.

---

## Inicializacion de campos estaticos con valor por defecto

El frontend Vesta emite codigo en `__module_init` para inicializar los campos estaticos
que tienen valor por defecto:

```vex
class Contador {
    public static i64 total = 0; // valor por defecto: 0
}
```

Genera:

```
defclass r1, r3 // crear la clase
// (deffield para el campo total, is_static=1)
// inicializar total = 0:
mov r5, 0
setstatic r5, r1, <offset_total> // Contador.total = 0
```

Para valores por defecto distintos de cero, el cuerpo de `__module_init` emite los
`setstatic` correspondientes.

---

## Ejemplo completo: contador estatico

```vex
class Logger {
    public static i64 mensajes = 0;

    public static void log(string msg) {
        Logger.mensajes += 1;
        println("[${Logger.mensajes}] ${msg}");
    }
}

i32 main(string[] args) {
    Logger.log("inicio"); // [1] inicio
    Logger.log("proceso"); // [2] proceso
    Logger.log("fin"); // [3] fin
    println("Total: ${Logger.mensajes}"); // Total: 3
    return 0;
}
```

Bytecode generado (esquematico para `Logger.mensajes += 1`):

```
// findclass de Logger (inlineado como cache lookup)
mov r12, @Absolute("code.s_class_cache_Logger")
mov r12, [r12] // r12 = ClassInfo* de Logger (del cache)

// getstatic + add + setstatic
getstatic r2, r12, <offset_mensajes>
add r2, 1
setstatic r2, r12, <offset_mensajes>
```

---

## Rendimiento

| Operacion | Coste |
| :----------------------------------- | :----------------------------------- |
| `getstatic` / `setstatic` | 1 instruccion VM + 1 memcpy de 8 B |
| findclass + getstatic (sin cache) | O(1) hash + 1 instruccion |
| cache + getstatic (con cache) | 2 instrucciones (2 loads) + 1 getstatic |

El coste es equivalente al acceso a un campo de instancia ordinario. No hay overhead
de vtable ni de GC.

---

Implementacion: `src/runtime/exec_instruction_meta.cpp`
(funciones `exec_instr_getstatic` y `exec_instr_setstatic`).

Ver tambien: [[META_OOP]], [[Vesta/OOP]], [[GC/GC]]
