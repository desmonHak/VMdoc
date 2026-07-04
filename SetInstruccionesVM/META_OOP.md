# META_OOP — Instrucciones de POO dinamica con reflexion y AOP

Las instrucciones de este grupo implementan el sistema de clases en runtime de VestaVM.
A diferencia de otras VMs donde las clases son metadata estatica en el binario, en VestaVM
las clases se registran **dinamicamente** durante la ejecucion de `__module_init` mediante
estas instrucciones. Esto unifica clases declaradas estaticamente, clases creadas por
metaprogramacion y clases cargadas con `loadmodule`.

Todas las instrucciones usan el prefijo extendido `[0x00]` y encoding FIXED_4 (4 bytes).

---

## Tabla resumen

| Instruccion | opcode2 | Arity | Descripcion |
| :------------ | :-----: | :---: | :----------------------------------------------- |
| `defclass` | 0xC9 | TWO | Crear nueva clase; retorna `ClassInfo*` en r_dst |
| `deffield` | 0xCA | TWO | Anadir campo a una clase |
| `defmethod` | 0xCB | TWO | Anadir metodo a una clase |
| `findclass` | 0xCC | TWO | Buscar clase por nombre; retorna `ClassInfo*` |
| `findmethod` | 0xCD | TWO | Buscar metodo por nombre; retorna `MethodInfo*` |
| `addadvice` | 0xCE | THREE | Registrar consejo AOP en un metodo |
| `findfield` | 0xCF | TWO | Buscar campo por nombre; retorna `FieldInfo*` |
| `callm` | 0xFD | TWO | Dispatch dinamico via `MethodInfo*` |
| `proceed` | 0xFE | ZERO | Invocar metodo target desde un advice AROUND |

---

## `defclass` (0xC9) — Definir una clase

### Encoding

```
[0x00][0xC9][byte2][0x00]
byte2 = (r_params << 4) | r_dst
```

- `r_dst`: registro donde se almacenara el `ClassInfo*` recien creado.
- `r_params`: registro que apunta a una estructura `DefClassParams` en VM memory.

### DefClassParams ABI (32 bytes en VM memory)

```
+0 [8 bytes] name_addr — VA del nombre de la clase (ASCII, no terminado en NUL)
+8 [4 bytes] name_len — longitud del nombre en bytes
+12 [4 bytes] flags — CLASS_FLAG_* (ver tabla abajo)
+16 [8 bytes] super_class — ClassInfo* de la clase base (0 si no hay herencia)
+24 [8 bytes] _reserved — reservado, debe ser 0
```

### Flags CLASS_FLAG_*

| Flag | Valor | Significado |
| :-------------------- | :----- | :---------------------------------------- |
| `CLASS_FLAG_ABSTRACT` | 0x01 | La clase tiene metodos abstractos |
| `CLASS_FLAG_FINAL` | 0x02 | La clase no puede heredarse |
| `CLASS_FLAG_INTERFACE`| 0x04 | Es una interfaz (sin campos, sin impl) |
| `CLASS_FLAG_GENERIC` | 0x08 | Template generico (no instanciar directo) |
| `CLASS_FLAG_CLOSURE` | 0x10 | Clase interna de closure (uso interno) |

### Comportamiento

1. Crea un `ClassInfo` nuevo y lo registra en el `ClassRegistry` del Loader.
2. Si `super_class != 0`, copia los fields/vtable del super (herencia).
3. Construye las tablas hash de lookup para fields y metodos (FNV-1a + linear probing).
4. Retorna el `ClassInfo*` en `r_dst`.

### Ejemplo

```
// Crear clase 'Perro' con super 'Animal':
mov r3, @Absolute("data.dc_Perro_params")
defclass r1, r3 // r1 = ClassInfo* de Perro
```

---

## `deffield` (0xCA) — Definir un campo

### Encoding

```
[0x00][0xCA][byte2][0x00]
byte2 = (r_params << 4) | r_class
```

- `r_class`: `ClassInfo*` de la clase a la que se anade el campo.
- `r_params`: puntero a `DefFieldParams` en VM memory.

### DefFieldParams ABI (32 bytes en VM memory)

```
+0 [8 bytes] name_addr — VA del nombre del campo
+8 [4 bytes] name_len — longitud en bytes
+12 [1 byte ] kind — FieldKind: 0=PRIMITIVE 1=REFERENCE 2=STATIC
+13 [1 byte ] access — FieldAccess: 0=PUBLIC 1=PRIVATE 2=PROTECTED
+14 [1 byte ] is_static — 1 si el campo es estatico, 0 si es de instancia
+15 [1 byte ] _pad — alineacion
+16 [4 bytes] size_bytes — tamaño del campo en bytes (4 u 8 tipicamente)
+20 [4 bytes] _pad2 — alineacion
+24 [8 bytes] type_class — ClassInfo* del tipo (0 para tipos primitivos)
```

### Comportamiento

- Anade el `FieldInfo` al array de fields del `ClassInfo`.
- Actualiza el offset del campo dentro del layout del objeto (acumulativo).
- Actualiza la tabla hash de lookup de fields.
- Retorna 1 en R0 si ok, 0 si fallo.
- Solo seguro durante `__module_init` (antes de la primera `NEWOBJ`).

---

## `defmethod` (0xCB) — Definir un metodo

### Encoding

```
[0x00][0xCB][byte2][0x00]
byte2 = (r_params << 4) | r_class
```

- `r_class`: `ClassInfo*` de la clase a la que se anade el metodo.
- `r_params`: puntero a `DefMethodParams` en VM memory.

### DefMethodParams ABI (40 bytes en VM memory)

```
+0 [8 bytes] name_addr — VA del nombre del metodo
+8 [4 bytes] name_len — longitud del nombre en bytes
+12 [4 bytes] descriptor_len — longitud del descriptor de firma en bytes
+16 [8 bytes] descriptor_addr — VA del descriptor de firma (ASCII)
+24 [8 bytes] code_vaddr — VA del punto de entrada del bytecode del metodo
+32 [8 bytes] flags — METHOD_FLAG_*
```

### Flags METHOD_FLAG_*

| Flag | Valor | Significado |
| :----------------------- | :---- | :------------------------------------- |
| `METHOD_FLAG_ABSTRACT` | 0x01 | Sin implementacion (throws si se llama)|
| `METHOD_FLAG_FINAL` | 0x02 | No puede ser sobreescrito |
| `METHOD_FLAG_STATIC` | 0x04 | Metodo de clase, no de instancia |
| `METHOD_FLAG_OVERRIDE` | 0x08 | Sobreescribe un slot de la vtable |
| `METHOD_FLAG_CONSTRUCTOR`| 0x10 | Es un constructor |
| `METHOD_FLAG_DESTRUCTOR` | 0x20 | Es un destructor (~Class()) |

### Comportamiento

- Si el metodo ya existe (mismo nombre) en la vtable heredada, reemplaza el slot.
- Si es nuevo, extiende la vtable.
- Retorna el indice en la vtable en R0, o `UINT32_MAX` si fallo.
- El `code_vaddr` es la VA del bytecode (label resuelto por el linker).

---

## `findclass` (0xCC) — Buscar una clase por nombre

### Encoding

```
[0x00][0xCC][byte2][0x00]
byte2 = (r_params << 4) | r_dst
```

### FindClassParams ABI (16 bytes en VM memory)

```
+0 [8 bytes] name_addr — VA del nombre a buscar
+8 [4 bytes] name_len — longitud del nombre en bytes
+12 [4 bytes] _pad — alineacion
```

### Comportamiento

- Busca en `ClassRegistry::class_map_` con O(1) amortizado.
- Retorna el `ClassInfo*` en `r_dst`, o 0 si no encontrado.
- No lanza excepcion si la clase no existe (usar `isnull` para verificar).

### Ejemplo

```
// Buscar clase 'Animal':
mov r3, @Absolute("data.fc_Animal_params")
findclass r1, r3 // r1 = ClassInfo* o 0 si no existe
isnull r2, r1 // r2 = 1 si no existe
```

---

## `findmethod` (0xCD) — Buscar un metodo por nombre

### Encoding

```
[0x00][0xCD][byte2][0x00]
byte2 = (r_params << 4) | r_dst
```

Mismo shape que `findclass`. Reutiliza `FindMethodParams` (24 bytes):

```
+0 [8 bytes] class_ptr — ClassInfo* donde buscar
+8 [8 bytes] name_addr — VA del nombre del metodo
+16 [4 bytes] name_len — longitud del nombre en bytes
+20 [4 bytes] _pad
```

### Comportamiento

- Lookup O(1) en `ClassInfo::method_lookup_table` (FNV-1a hash + linear probing).
- Retorna el `MethodInfo*` en `r_dst`, o 0 si no encontrado.

---

## `findfield` (0xCF) — Buscar un campo por nombre

### Encoding

```
[0x00][0xCF][byte2][0x00]
byte2 = (r_params << 4) | r_dst
```

Mismo shape que `findmethod`. Reutiliza `FindMethodParams` (mismo ABI de 24 bytes):

```
+0 [8 bytes] class_ptr — ClassInfo* donde buscar
+8 [8 bytes] name_addr — VA del nombre del campo
+16 [4 bytes] name_len — longitud del nombre en bytes
+20 [4 bytes] _pad
```

### Comportamiento

- Lookup O(1) en `ClassInfo::field_lookup_table`.
- Retorna el `FieldInfo*` en `r_dst`, o 0 si no encontrado.

---

## `addadvice` (0xCE) — Registrar un consejo AOP

### Encoding

```
[0x00][0xCE][byte2][byte3]
byte2 = (r_advice << 4) | r_target (convencion B: raw bytes)
byte3 = kind (0=BEFORE, 1=AFTER, 2=AROUND)
```

- `r_target`: `MethodInfo*` del metodo objetivo (el que va a ser interceptado).
- `r_advice`: `MethodInfo*` del metodo consejo (el que se ejecutara como advice).
- `byte3 = kind`: tipo de consejo.

### Tipos de consejo

| kind | Constante | Efecto |
| :--- | :-------- | :---------------------------------------------------------- |
| 0 | BEFORE | Se ejecuta ANTES del metodo target |
| 1 | AFTER | Se ejecuta DESPUES del metodo target (valor ya retornado) |
| 2 | AROUND | Reemplaza la llamada; usa `proceed` para invocar el target |

### Comportamiento

- Enlaza un `AdviceEntry { kind, advice_method, next }` al final de la cadena
 `MethodInfo::advice_chain`.
- Cuando `CALLVIRT`/`CALLM` ve `advice_chain != NULL`, recorre la cadena.
- Si `advice_chain == NULL` (por defecto), `CALLVIRT` hace dispatch directo con
 **overhead cero** (una comparacion + salto predicho).
- Retorna 1 en R0 si ok, 0 si fallo (firmas incompatibles, punteros invalidos).

### Restricciones

- Los advices deben tener la misma firma que el target (validado por `add_advice`).
- Solo el primer `AROUND` en la cadena tiene efecto actualmente (multi-AROUND nesting
 requiere stack de `proceed_targets`, pendiente de implementacion).

---

## `callm` (0xFD) — Dispatch dinamico via MethodInfo*

### Encoding

```
[0x00][0xFD][byte2][0x00]
byte2 = (r_method << 4) | r_obj
```

- `r_obj`: host_ptr al objeto receptor (el `this`).
- `r_method`: `MethodInfo*` del metodo a invocar.

### Comportamiento

1. Verifica que `r_obj != 0` y que `r_method != 0`.
2. Si `r_method->advice_chain != NULL`, recorre la cadena (mismo codigo que CALLVIRT).
3. En otro caso, hace dispatch directo a `r_method->code_vaddr`.
4. El objeto en `r_obj` queda en R1 (convencion `this`).
5. Los argumentos deben estar en R2..R12 antes de la llamada.
6. El resultado queda en R0 tras el retorno.

### Diferencia con CALLVIRT

`CALLVIRT` usa un slot de vtable (indice calculado en compile-time). `CALLM` usa un
`MethodInfo*` directo. Son equivalentes en rendimiento cuando `advice_chain == NULL`.
`CALLM` se usa en:

- Polimorfismo via tipo interfaz (donde no hay vtable_index conocido).
- Builtins de reflexion `invoke(method, this, args)`.
- Frontend Vesta en dispatch de interfaces.

---

## `proceed` (0xFE) — Invocar target desde advice AROUND

### Encoding

```
[0x00][0xFE][0x00][0x00] (ZERO arity, 4 bytes)
```

### Comportamiento

- Solo valido dentro de un advice `AROUND` (cuando el frame actual tiene `is_around=true`).
- Lee `frame.proceed_target` del frame actual (apunta al `MethodInfo*` del target original).
- Re-invoca el target con la calling convention actual:
 - R1 = this
 - R2..RN = argumentos actuales
 - R15 = argc
- Retorna el valor del target en R0.
- Falla con `FATAL_ILLEGAL_INSTRUCTION` si se llama fuera de un frame AROUND.

### Uso desde assembly

```
// Dentro de un advice @Around:
mov r1, r_this // (ya cargado por la VM al entrar)
proceed // invocar el target original
mov r_result, r0 // capturar resultado
```

### Builtin Vesta equivalente

```vx
@Around("Servicio.calcular")
public i64 alrededor(i64 x) {
    i64 resultado = proceed(); // <-- baja a: proceed; mov {dst}, r0
    return resultado * 2;
}
```

---

## Flujo completo de POO dinamica

El siguiente pseudocodigo muestra como el frontend Vesta compila una clase simple:

```
// Fuente Vesta:
class Animal {
    public string nombre;
    public i32 edad;
    Animal(string n, i32 e) { this.nombre = n; this.edad = e; }
    public string toString() { return "Animal(${nombre}, ${edad})"; }
}

// Generado en __module_init:
    mov r3, @Absolute("data.dc_Animal") // DefClassParams: nombre="Animal", flags=0, super=0
    defclass r1, r3 // r1 = ClassInfo*

    mov r3, @Absolute("data.df_nombre") // DefFieldParams: nombre="nombre", size=8, ref
    deffield r1, r3 // campo 'nombre'

    mov r3, @Absolute("data.df_edad") // DefFieldParams: nombre="edad", size=4, prim
    deffield r1, r3 // campo 'edad'

    mov r3, @Absolute("data.dm_ctor") // DefMethodParams: code_vaddr=Animal_ctor
    defmethod r1, r3 // metodo constructor

    mov r3, @Absolute("data.dm_toString") // DefMethodParams: code_vaddr=Animal_toString
    defmethod r1, r3 // metodo toString

// Generado en main:
    mov r3, @Absolute("data.fc_Animal")
    findclass r2, r3 // r2 = ClassInfo* de Animal (ya registrado)
    newobj r2 // R0 = GcHandle nuevo, r2 = host_ptr
    mov r1, r2 // this = el objeto
    mov r2, @StringRef("Rex") // arg1 = nombre
    mov r3, 5 // arg2 = edad
    callvirt r1, 0 // llamar ctor (vtable slot 0)
    callvirt r1, 1 // llamar toString (vtable slot 1)
```

---

## Rendimiento

| Operacion | Coste |
| :---------------------- | :----------------------------------------- |
| `defclass` | O(1) + hash table build; solo en init |
| `deffield`/`defmethod` | O(1) amortizado (puede realocar tablas) |
| `findclass` | O(1) en `unordered_map<string, ClassInfo*>`|
| `findmethod`/`findfield`| O(1) en tabla hash open-addressing |
| `callm` sin advices | Igual que `callvirt` con slot directo |
| `callm` con N advices | N CALLVIRT adicionales en cadena lineal |
| `proceed` | Un call al target original |
| `addadvice` | O(1) enlace al final de chain; solo init |

---

Implementacion: `src/runtime/exec_instruction_meta.cpp` (0xC9–0xCF),
`src/runtime/exec_instruction_oop.cpp` (0xFD callm, 0xFE proceed).

Ver tambien: [[OOP/CALLVIRT y CALLSUPER]], [[GC/Generacional (para objetos OOP)]],
[[Vesta/OOP]], [[Vesta/ReflexionAOP]]
