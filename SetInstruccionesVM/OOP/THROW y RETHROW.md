# THROW y RETHROW - Lanzamiento de excepciones

`THROW` inicia el mecanismo de desenrollado de pila buscando un handler compatible. `RETHROW` relanza la excepción activa almacenada en `vm->current_exception` sin crear una nueva.

> **Ver también:** [[CALLVIRT y CALLSUPER]] (crea los `FrameHeader` que THROW recorre), [[NEWOBJRAW y NEWOBJ]] (crea los objetos excepción).

---

## `THROW` - lanzar una excepción

```asm
throw  reg_exc    ; reg_exc = host ptr al ObjectHeader de la excepción
```

| Campo     | Valor              |
| :-------: | :----------------- |
| Opcode1   | `0x00`             |
| Opcode2   | `0xD3`             |
| Tamaño    | 4 bytes (FIXED_4)  |
| `reg_exc` | registro con host pointer al `ObjectHeader` del objeto excepción |

Si `reg_exc == 0`: segmentation fault (`EVT_ERROR`) inmediato.

### Algoritmo de búsqueda de handler

```
1. Guarda exception_ptr en vm->current_exception
2. Lee exc_class = ObjectHeader.class_ptr
3. Recorre la cadena vm->frame_stack (más reciente -> más antiguo):
   Para cada FrameHeader:
     a. Calcula pc_offset = RIP - method->code_vaddr
     b. Para cada HandlerException h en method->handlers[]:
        - Si pc_offset ∉ [h.start_pc, h.end_pc): skip
        - Si h.type == nullptr (catch-all) OR is_instance_of(exc_class, h.type):
          * Elimina los FrameHeader más recientes que este
          * R0 = exception_ptr
          * RIP = method->code_vaddr + h.handler_pc
          * did_jump = true -> RETURN (handler encontrado)
4. Si no hay handler: mata el proceso (EVT_ERROR)
```

### HandlerException - estructura de la tabla de handlers

```cpp
struct HandlerException {
    ClassInfo *type;       // nullptr = catch-all (finally / catch(Exception))
    uint32_t   start_pc;   // inicio del bloque try (offset desde code_vaddr)
    uint32_t   end_pc;     // fin exclusivo del bloque try
    uint32_t   handler_pc; // destino de salto (offset desde code_vaddr)
};  // 24 bytes (type=8B + tres dwords=12B + 4B padding de alineacion)
```

Layout en memoria:

```
+0   type        (8B)  ClassInfo* (null = catch-all)
+8   start_pc    (4B)  uint32_t - offset inicio del bloque try
+12  end_pc      (4B)  uint32_t - offset fin exclusivo del bloque try
+16  handler_pc  (4B)  uint32_t - offset del bloque catch/finally
+20  --          (4B)  padding de alineacion
```

Total: 24 bytes por entrada.

El campo `type` se compara con `is_instance_of(exc_class, h.type)`, que recorre recursivamente `supers[]` e `interfaces[]`: un handler de tipo `Exception` captura tanto `RuntimeException` como `IOException`.

### Estado al entrar en el handler

- `R0` = host pointer al objeto excepción (el mismo que se pasó a `throw`)
- `RIP` = `code_vaddr + handler_pc`
- Los `FrameHeader` más nuevos que el frame del handler han sido eliminados
- La pila (RSP) **no se modifica** por `THROW`; el handler debe usar `RET` para volver al caller original

### Ejemplo completo

```asm
; --- setup (normalmente hecho por el loader) ---
; method_info tiene:
;   code_vaddr    = @Absolute("code.try_section")
;   handler_count = 1
;   handlers[0]:  type=null, start_pc=0, end_pc=4, handler_pc=4

try_section:
    throw r2            ; +0 (4 bytes FIXED_4): lanza el objeto en r2

catch_handler:          ; +4: aquí llega el control si throw ocurre en [0,4)
    ; R0 = objeto excepción
    mov  r10, 0xAA      ; procesamiento del error
    ret                 ; vuelve al caller del CALLVIRT
```

---

## `RETHROW` - relanzar la excepción activa

```asm
rethrow    ; sin operandos
```

| Campo   | Valor              |
| :-----: | :----------------- |
| Opcode1 | `0x00`             |
| Opcode2 | `0xD4`             |
| Tamaño  | 2 bytes (FIXED_2)  |

Equivalente a `throw vm->current_exception`. Útil para:
- Bloques `finally` que quieren limpiar recursos y relanzar.
- Capturar, loggear y volver a lanzar sin perder el objeto original.

Si `vm->current_exception == 0` (no hay excepción activa): mismo comportamiento que `throw 0` -> `EVT_ERROR`.

### Ejemplo

```asm
; En un handler catch-all que solo hace logging:
catch_all:
    ; R0 = excepción activa
    ; ... log o limpieza ...
    rethrow             ; relanza hacia el frame anterior
```

---

## Desenrollado de la pila de frames

`THROW` elimina `FrameHeader`s al subir por la jerarquía de llamadas. Visualmente:

```
Antes de THROW (lanzada en method_C):
  frame_stack:  [C] -> [B] -> [A] -> null

  method_A tiene un handler que cubre el rango donde fue llamado method_B.

Después de THROW (handler encontrado en frame A):
  frame_stack:  [A] -> null   (B y C eliminados)
  RIP = code_vaddr_A + handler_pc_A
  R0  = exception_ptr
```

---

## Patrones de uso avanzados

### Jerarquía de excepciones

```asm
; Crear handler para Exception y subclases
; HandlerException.type = ExceptionClassInfo*
; is_instance_of comprueba:
;   exc_class == ExceptionClassInfo   -> captura
;   exc_class.supers[0] == ExceptionClassInfo -> captura (subclase)
;   ninguna relación -> no captura
```

### Bloques finally (catch-all)

```asm
; HandlerException.type = null -> captura cualquier excepción
; Después de ejecutar el código de limpieza, hacer RETHROW si no se quiere
; absorber la excepción:
finally_block:
    ; limpieza...
    rethrow
```

### Múltiples handlers en un método

Un `MethodInfo` puede tener `handler_count > 1`. Los handlers se evalúan en orden; el primero que coincida gana:

```
handlers[0]: type=FileNotFoundException, [0, 20), handler=20
handlers[1]: type=IOException,           [0, 20), handler=40
handlers[2]: type=null (finally),        [0, 60), handler=60
```

---

## Codificación binaria

### THROW

```
┌────────┬────────┬────────────────────┬────────────────────┐
│ 0x00   │ 0xD3   │  mode<<6 | 0x00    │  0000 | reg_exc    │
└────────┴────────┴────────────────────┴────────────────────┘
```

### RETHROW

```
┌────────┬────────┐
│ 0x00   │ 0xD4   │
└────────┴────────┘
```

---

Ver también: [[CALLVIRT y CALLSUPER]], [[NEWOBJRAW y NEWOBJ]], [[Reflexion OOP]], [[RET]]
