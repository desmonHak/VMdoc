# NEWOBJ y NEWOBJRAW - Creacion de objetos OOP

Estas dos instrucciones crean objetos en memoria siguiendo las reglas del sistema orientado
a objetos de VestaVM. La diferencia fundamental es quien gestiona la vida del objeto:
el **GC automatico** (`NEWOBJ`) o el **programador manualmente** (`NEWOBJRAW`).

> **Ver tambien:** [[GCALLOC]] para asignacion GC sin estructura OOP,
> [[Generacional (para objetos OOP)|GcHeap]] para el ciclo de vida del GC,
> [[CALLVIRT y CALLSUPER]] para llamar metodos en objetos.

---

## Conceptos previos

### Que es un objeto en VestaVM

Un objeto es un bloque de memoria con una **cabecera** (`ObjectHeader`) seguida de los
**campos** de datos de la clase. La cabecera le dice a la VM de que clase es el objeto,
si esta bajo el GC, y si tiene un monitor (mutex) activo.

```
Memoria del objeto:
+----------------------------+   ObjectHeader (24 bytes)
| class_ptr  (8 bytes)       |   <- apunta a la clase del objeto
| flags      (4 bytes)       |   <- bits de estado (GC, monitor...)
| hash_code  (4 bytes)       |   <- hash inicial del objeto
| owner_pid  (4 bytes)       |   <- PID del proceso que tiene el monitor (0=libre)
| lock_depth (2 bytes)       |   <- nivel de anidacion del monitor (reentrante)
| _mon_pad   (2 bytes)       |   <- padding de alineacion
+----------------------------+   Fin de cabecera
| campo_1    (N bytes)       |   <- primer campo del objeto
| campo_2    (N bytes)       |   <- segundo campo
| ...                        |
+----------------------------+
```

### Que es ClassInfo

`ClassInfo` es una estructura en memoria host que describe una clase: su nombre, cuantos
bytes ocupa cada instancia, sus campos, sus metodos, su herencia. El **loader** la crea al
cargar el programa; el bytecode recibe un puntero a ella para usarla.

### GcHandle vs puntero directo

- `NEWOBJ` devuelve un **GcHandle**: un numero entero opaco que identifica el objeto dentro
  del GC. No es una direccion de memoria real. Esto permite al GC mover el objeto sin que
  el bytecode lo note.
- `NEWOBJRAW` devuelve un **puntero host**: la direccion real del objeto en la RAM del host.
  El objeto no se puede mover; el programador es responsable de liberarlo.

---

## `NEWOBJ` - objeto bajo GC generacional

```c
newobj  reg_classinfo    // R0 = GcHandle del nuevo objeto
```

| Campo       | Valor                          |
| :---------: | :----------------------------- |
| Opcode1     | `0x00`                         |
| Opcode2     | `0xA0`                         |
| Tamano      | 4 bytes (FIXED_4)              |
| `reg_classinfo` | registro con host ptr a `ClassInfo` |
| Resultado   | `R0` = `GcHandle` (o `0xFFFFFFFF` si falla) |

### Que hace exactamente

1. Lee `classinfo->instance_size` para saber cuantos bytes necesita el objeto.
2. Llama al GC heap: `GcHeap::alloc(instance_size)`.
3. Si hay exito, escribe el `ObjectHeader` al inicio del bloque:
   - `class_ptr` = puntero a la ClassInfo recibida.
   - `flags` = `OBJ_FLAG_GC_OWNED` (bit 0 = 1) -- el GC es el dueno.
   - `hash_code` = los 32 bits bajos del GcHandle.
   - `owner_pid` = 0 (monitor libre).
   - `lock_depth` = 0.
4. Devuelve el `GcHandle` en `R0`.

Si `reg_classinfo` es cero o no hay memoria disponible: `R0 = GC_NULL_HANDLE (0xFFFFFFFF)`.

### Uso tipico

```c
// cls_ptr esta en r1 (host ptr a ClassInfo, cargado por el loader)
newobj  r1              // R0 = GcHandle al nuevo objeto

// Para acceder al payload del objeto (leer/escribir campos):
gcderef cur0, r0        // cur0 = puntero host al ObjectHeader

// Leer class_ptr del ObjectHeader (offset 0):
readcur r2, cur0        // r2 = ClassInfo* (los primeros 8 bytes del objeto)
```

### ObjectHeader escrito por NEWOBJ

```
Offset dentro del objeto:
  offset  0 [8B] class_ptr  = ClassInfo* (puntero a la clase)
  offset  8 [4B] flags      = 1 (OBJ_FLAG_GC_OWNED)
  offset 12 [4B] hash_code  = GcHandle & 0xFFFFFFFF
  offset 16 [4B] owner_pid  = 0 (monitor libre)
  offset 20 [2B] lock_depth = 0
  offset 22 [2B] _mon_pad   = 0
  -- Aqui empiezan los campos de usuario (offset 24) --
```

---

## `NEWOBJRAW` - objeto bajo allocator crudo (sin GC)

```c
newobjraw  reg_classinfo, reg_size    // R0 = host_ptr al ObjectHeader
```

| Campo           | Valor                                                       |
| :-------------: | :---------------------------------------------------------- |
| Opcode1         | `0x00`                                                      |
| Opcode2         | `0xD0`                                                      |
| Tamano          | 4 bytes (FIXED_4)                                           |
| `reg_classinfo` | registro con host ptr a `ClassInfo`                         |
| `reg_size`      | registro con el numero de bytes a reservar (0 = usar `instance_size`) |
| Resultado       | `R0` = host pointer al `ObjectHeader`                       |

### Que hace exactamente

1. Si `reg_classinfo` es cero: `R0 = 0`, no hace nada.
2. Si `reg_size != R00` (indice 0): usa ese valor como tamano total.
3. Si `reg_size == R00` (registro r0, convencion especial): usa `classinfo->instance_size`.
4. Reserva memoria con `RawAllocator::alloc(size)`.
5. Escribe el `ObjectHeader` (mismo layout que NEWOBJ):
   - `class_ptr` = `ClassInfo*`
   - `flags` = `OBJ_FLAG_RAW_OWNED` (bit 1 = 2) -- el programador es el dueno.
   - `hash_code` = los 32 bits bajos del puntero host.
   - `owner_pid` = 0, `lock_depth` = 0.
6. Devuelve el puntero host en `R0`.

El objeto resultante **no esta bajo GC**: debe liberarse manualmente con `FREE`.

### Uso tipico

```c
// Crear objeto con tamano explicito
mov       r1, my_class_ptr   // r1 = ClassInfo host ptr
mov       r2, 72             // tamano: 24B (ObjectHeader) + 48B de campos
newobjraw r1, r2             // R0 = host ptr al ObjectHeader

// Acceder a campos del objeto via cursor:
mov   r14, r0
xchg  cur0, r14              // cur0 = inicio del ObjectHeader

readcur r3, cur0             // r3 = class_ptr (offset 0, primeros 8 bytes)

// Acceder al primer campo del usuario (offset 24, despues del ObjectHeader):
mov   r14, r0
addu  r14, 24
xchg  cur1, r14              // cur1 = inicio de los campos de usuario

// Liberar cuando ya no se necesite:
free  r0
```

---

## ObjectHeader - layout completo (ABI v2, 24 bytes)

Todos los objetos OOP (tanto GC como raw) tienen los primeros 24 bytes en este formato:

```
+--------+--------+--------+--------+--------+--------+--------+--------+
| class_ptr                                                             |  +0  (8 bytes)
+--------+--------+--------+--------+--------+--------+--------+--------+
| flags           (4 bytes)         | hash_code       (4 bytes)        |  +8, +12
+--------+--------+--------+--------+--------+--------+--------+--------+
| owner_pid       (4 bytes)         | lock_depth (2B) | _mon_pad (2B)  |  +16, +20, +22
+--------+--------+--------+--------+--------+--------+--------+--------+
| -- inicio de campos de usuario --                                     |  +24
```

| Campo        | Offset | Tipo         | Descripcion                                        |
| :----------- | :----: | :----------- | :------------------------------------------------- |
| `class_ptr`  |  +0    | `ClassInfo*` | Puntero host a la clase del objeto                 |
| `flags`      |  +8    | `uint32_t`   | Bits de estado del objeto                          |
| `hash_code`  | +12    | `uint32_t`   | Hash inicial (handle o puntero truncado a 32 bits) |
| `owner_pid`  | +16    | `uint32_t`   | PID del proceso dueno del monitor (0 = libre)      |
| `lock_depth` | +20    | `uint16_t`   | Nivel de anidacion del mutex reentrante            |
| `_mon_pad`   | +22    | `uint16_t`   | Padding de alineacion (no usar)                    |

> **IMPORTANTE:** Los campos de usuario empiezan en el offset +24 (no +16 como en ABI v1).
> Si tienes codigo que usa offsets codificados, actualiza todos los valores mayores de 16.

### Flags del ObjectHeader (`OBJ_FLAG_*`)

| Constante             | Valor | Significado                                    |
| :-------------------- | :---: | :--------------------------------------------- |
| `OBJ_FLAG_GC_OWNED`   |  `1`  | Objeto gestionado por GC (`NEWOBJ`)             |
| `OBJ_FLAG_RAW_OWNED`  |  `2`  | Objeto asignado con raw allocator (`NEWOBJRAW`) |
| `OBJ_FLAG_CLOSURE`    |  `4`  | El objeto es un ClosureObject                   |
| `OBJ_FLAG_GC_MARKED`  |  `8`  | Marcado durante el ciclo GC actual              |

---

## Cuando usar NEWOBJ vs NEWOBJRAW

| Criterio               | `NEWOBJ`                          | `NEWOBJRAW`                         |
| :--------------------- | :-------------------------------- | :---------------------------------- |
| Ciclo de vida          | Automatico (GC)                   | Manual (`FREE`)                     |
| Resultado en R0        | `GcHandle` (indice opaco)         | Host pointer (direccion real)       |
| Acceso al payload      | `GCDEREF cur, handle` -> cursor   | Cursor directo                      |
| Flag en ObjectHeader   | `OBJ_FLAG_GC_OWNED` (1)           | `OBJ_FLAG_RAW_OWNED` (2)            |
| Compatible con FFI     | No (el handle no es una direccion)| Si (el puntero es real)             |
| Uso recomendado        | Objetos de usuario normales       | FFI, objetos de vida muy corta      |

---

## Codificacion binaria

### NEWOBJ (4 bytes)

```
+--------+--------+--------------------+--------------------+
| 0x00   | 0xA0   |  mode<<6 | 0x00   |  0000 | reg_cls    |
+--------+--------+--------------------+--------------------+
  byte0    byte1       byte2                 byte3
```

### NEWOBJRAW (4 bytes)

```
+--------+--------+--------------------+--------------------+
| 0x00   | 0xD0   |  mode<<6 | 0x00   |  reg_sz<<4|reg_cls |
+--------+--------+--------------------+--------------------+
  byte0    byte1       byte2                 byte3
```

- `reg_cls` (4 bits bajos de byte3) = indice del registro con `ClassInfo*`
- `reg_sz`  (4 bits altos de byte3) = indice del registro con el tamano; `0` = usar `instance_size`

---

## Ejemplo completo: crear y usar un objeto con NEWOBJ

```c
@Format("raw")
@SpaceAddress { @Name("mem"), @IniAddress(0x0), @EndAddress(0xFFFFFFFFFFFFFFFF) }
@Section { @Name("code"), @SpaceAddress("mem") @Align(0x1000) }

code:
    mov  rsp, 0x00FF0000
    mov  rbp, 0x00FF0000

    // Suponer que r1 contiene un ClassInfo* cargado por el loader
    // (en un programa real se obtiene con @Class("MiClase") o via reflexion)
    newobj  r1              // R0 = GcHandle del nuevo objeto

    // Verificar que no fallo
    mov   r10, 0xFFFFFFFF
    cmpu  r0, r10
    jmp.je error            // si R0 == GC_NULL_HANDLE, error

    // Acceder al primer campo de usuario (offset 24)
    gcderef cur0, r0        // cur0 = host ptr al ObjectHeader
    mov     r14, cur0       // (no podemos usar cur0 directamente en addu)
    // Nota: addu sobre cursor no existe; trabajamos con host ptr
    // Para acceder al offset +24, hacemos:
    //   1. guardar el handle
    //   2. calcular la direccion con addu sobre un registro general
    //   3. usar xchg para mover la direccion al cursor

    mov   r8, r0            // guardar handle en r8
    gcderef cur0, r8        // cur0 apunta al ObjectHeader (offset 0)

    // Para leer el campo de usuario en offset +24:
    // necesitamos un host ptr al campo; gcderef nos da el ptr al inicio del objeto
    // No podemos "sumar al cursor"; debemos usar un reg general:
    // (Esta es la limitacion de la interfaz de cursores)

    hlt

error:
    hlt
```

---

Ver tambien: [[GCALLOC]], [[Generacional (para objetos OOP)|GcHeap]], [[Allocator crudo para FFI y memoria manual|RawAllocator]], [[cursor]], [[GETCLASS]], [[CALLVIRT y CALLSUPER]], [[MONITOR]]
