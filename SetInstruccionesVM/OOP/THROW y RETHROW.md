# THROW y RETHROW - Lanzamiento de excepciones

`THROW` inicia el mecanismo de manejo de errores: busca el handler de excepcion mas cercano
en la cadena de llamadas y transfiere el control a el. `RETHROW` relanza la excepcion activa
sin crear una nueva, util para bloques `finally`.

> **Ver tambien:** [[CALLVIRT y CALLSUPER]] (crea los `FrameHeader` que THROW recorre),
> [[NEWOBJRAW y NEWOBJ]] (crea los objetos excepcion).

---

## Para que sirven las excepciones

Las excepciones son un mecanismo para manejar errores de forma estructurada. Sin excepciones,
cada funcion deberia comprobar el valor de retorno de cada llamada para detectar errores, lo
que llena el codigo de `if`s y hace muy dificil seguir el flujo normal.

Con excepciones:
1. Cuando algo falla, se **lanza** una excepcion (`THROW`).
2. La VM busca automaticamente el handler mas cercano en la cadena de llamadas.
3. El handler recibe el objeto excepcion y puede procesarlo o relanzarlo.

Analogia: es como un contrato de seguro. Cuando algo malo pasa (accidente = excepcion),
no tienes que gestionar el desastre tu mismo; el seguro (handler) se activa automaticamente.

---

## `THROW` - lanzar una excepcion

```c
throw  reg_exc    // reg_exc = host ptr al ObjectHeader del objeto excepcion
```

| Campo     | Valor                                                            |
| :-------: | :--------------------------------------------------------------- |
| Opcode1   | `0x00`                                                           |
| Opcode2   | `0xD3`                                                           |
| Tamano    | 4 bytes (FIXED_4)                                                |
| `reg_exc` | registro con host pointer al `ObjectHeader` del objeto excepcion |

Si `reg_exc == 0` (null): error de segmentacion (`EVT_ERROR`) inmediato.

### Algoritmo de busqueda de handler

Cuando `THROW` se ejecuta, la VM sube por la cadena de `FrameHeader` buscando quien puede
manejar esta excepcion:

```
1. Guarda exception_ptr en vm->current_exception
2. Lee exc_class = ObjectHeader.class_ptr (que clase es la excepcion)
3. Recorre la cadena vm->frame_stack (mas reciente -> mas antiguo):
   Para cada FrameHeader:
     a. Calcula pc_offset = RIP - method->code_vaddr
        (en que punto del metodo estabamos cuando se lanzo la excepcion)
     b. Para cada HandlerException h en method->handlers[]:
        - Si pc_offset NO esta en [h.start_pc, h.end_pc): este handler no aplica
        - Si h.type == nullptr (catch-all) O is_instance_of(exc_class, h.type):
          * Elimina los FrameHeader mas recientes que este
          * R0 = exception_ptr
          * RIP = method->code_vaddr + h.handler_pc (salta al bloque catch)
          * Retorna (handler encontrado, excepcion manejada)
4. Si no hay ningun handler: mata el proceso con EVT_ERROR (excepcion sin manejar)
```

### Estructura HandlerException (tabla de handlers)

Cada `MethodInfo` puede declarar una tabla de `HandlerException`:

```cpp
struct HandlerException {
    ClassInfo *type;       // null = catch-all (catch de cualquier excepcion / finally)
    uint32_t   start_pc;   // inicio del bloque try (offset desde code_vaddr)
    uint32_t   end_pc;     // fin exclusivo del bloque try
    uint32_t   handler_pc; // donde saltar al capturar (offset desde code_vaddr)
};  // 24 bytes total
```

Layout en memoria:

```
+0   type        (8B)  ClassInfo* (null = catch-all)
+8   start_pc    (4B)  uint32_t - inicio del bloque try
+12  end_pc      (4B)  uint32_t - fin exclusivo del bloque try
+16  handler_pc  (4B)  uint32_t - destino del bloque catch
+20  --          (4B)  padding de alineacion
```

El campo `type` se compara con herencia transitiva: un handler de tipo `Exception` captura
tanto `RuntimeException` como `IOException` si ambas heredan de `Exception`.

### Estado al entrar en el handler

Cuando `THROW` encuentra un handler y transfiere el control:

- `R0` = host pointer al objeto excepcion (el mismo que se paso a `throw`).
- `RIP` = `code_vaddr + handler_pc` (inicio del bloque catch).
- Los `FrameHeader` mas nuevos que el frame del handler han sido eliminados.
- La pila (RSP) **no se modifica** por `THROW`; el handler puede usar `RET` para volver.

### Ejemplo completo

```c
// Suponer que method_info tiene:
//   code_vaddr    = direccion de try_section
//   handler_count = 1
//   handlers[0]:  type=null (catch-all), start_pc=0, end_pc=4, handler_pc=4

try_section:
    throw r2            // offset +0 (4 bytes FIXED_4): lanza el objeto en r2
                        // THROW busca handler: [0,4) cubre offset 0 -> handler_pc=4

catch_handler:          // offset +4: aqui llega el control si throw ocurre en [0,4)
    // R0 = objeto excepcion (el mismo que estaba en r2 antes del throw)
    mov  r10, 0xAA      // procesamiento del error
    ret                 // vuelve al caller del metodo que contenia este try/catch
```

---

## `RETHROW` - relanzar la excepcion activa

```c
rethrow    // sin operandos; relanza vm->current_exception
```

| Campo   | Valor             |
| :-----: | :---------------- |
| Opcode1 | `0x00`            |
| Opcode2 | `0xD4`            |
| Tamano  | 2 bytes (FIXED_2) |

Equivalente a `throw vm->current_exception`. Util para:
- **Bloques finally**: ejecutar limpieza y luego relanzar.
- **Logging**: capturar, registrar el error y volver a lanzar sin perder el objeto original.

Si `vm->current_exception == 0` (no hay excepcion activa): mismo comportamiento que
`throw 0` -> `EVT_ERROR`.

```c
// Handler que solo hace logging y luego relanza:
catch_all:
    // R0 = excepcion activa (guardada automaticamente por THROW)
    // ... aqui podrias llamar a una funcion de log ...
    rethrow             // relanza hacia el frame anterior
```

---

## Desenrollado de la pila de frames

`THROW` elimina `FrameHeader`s al subir por la jerarquia de llamadas hasta encontrar
un handler. Visualmente:

```
Antes de THROW (excepcion lanzada en method_C):
  frame_stack (tope a la izquierda):  [C] -> [B] -> [A] -> null

Suponer que method_A tiene un handler que cubre el rango donde llamo a method_B.

Despues de THROW (handler encontrado en frame A):
  frame_stack:  [A] -> null   (FrameHeaders de B y C eliminados)
  RIP = code_vaddr_A + handler_pc_A
  R0  = exception_ptr
```

---

## Patrones de uso avanzados

### Jerarquia de excepciones

```c
// Si la jerarquia es: IOException -> Exception -> Throwable
// y el handler tiene type=Exception:
// is_instance_of comprueba:
//   exc_class == ExceptionClassInfo   -> true  (mismo tipo)
//   exc_class.supers[0] == Exception  -> true  (si RuntimeException hereda de Exception)
// Por tanto captura IOException, RuntimeException y cualquier subclase de Exception
```

### Bloques finally (catch-all)

```c
// HandlerException.type = null -> captura cualquier excepcion
// Patron: hacer limpieza y relanzar

finally_block:
    // limpieza de recursos...
    rethrow             // la excepcion sigue propagandose hacia arriba
```

### Multiples handlers en un metodo

Un `MethodInfo` puede tener `handler_count > 1`. Los handlers se evaluan en orden; el
primero que coincida con el tipo y el rango de PC gana:

```
// Tabla de handlers (en orden de evaluacion):
handlers[0]: type=FileNotFound, [0, 20), handler_pc=20    // catch especifico
handlers[1]: type=IOException,  [0, 20), handler_pc=40    // catch mas general
handlers[2]: type=null (finally),[0, 60), handler_pc=60   // siempre se ejecuta
```

Si la excepcion es `FileNotFoundException`:
- Handler[0] coincide (FileNotFound hereda de IOException? si, y es el tipo exacto).
- Se salta al handler_pc=20.
- El handler de IOException (handler[1]) no se ejecuta.
- El finally (handler[2]) si se puede ejecutar si el handler[0] lo relanza.

---

## Codificacion binaria

### THROW (4 bytes)

```
+--------+--------+--------------------+--------------------+
| 0x00   | 0xD3   |  mode<<6 | 0x00   |  0000 | reg_exc    |
+--------+--------+--------------------+--------------------+
  byte 0   byte 1       byte 2               byte 3
```

### RETHROW (2 bytes)

```
+--------+--------+
| 0x00   | 0xD4   |
+--------+--------+
  byte 0   byte 1
```

---

Ver tambien: [[CALLVIRT y CALLSUPER]], [[NEWOBJRAW y NEWOBJ]], [[Reflexion OOP]], [[RET]]
