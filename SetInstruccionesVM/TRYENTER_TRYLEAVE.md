# TRYENTER y TRYLEAVE - Frames de excepcion dinamicos

`tryenter` y `tryleave` permiten instalar y desinstalar bloques try/catch en tiempo de
ejecucion, sin necesidad de anotaciones estaticas en el bytecode del metodo.  Trabajan
conjuntamente con `do_throw` (el mecanismo de lanzamiento de excepciones de VestaVM)
para ofrecer manejo de errores estructurado.

| Instruccion | opcode0 | opcode1 | Modo | Tamano  | Descripcion                                        |
| :---------: | :-----: | :-----: | :--: | :-----: | :------------------------------------------------- |
| `tryenter`  |  0x00   |  0x44   | REG  | 4 bytes | Instalar ExceptionFrame en la pila del proceso     |
| `tryleave`  |  0x00   |  0x45   | NONE | 2 bytes | Desinstalar el ExceptionFrame del tope de la pila  |

Implementacion: `src/runtime/exec_instruction_exc.cpp`

---

## Para que sirve (nivel basico)

Imagina que estas caminando por un puente y llevas una cuerda de seguridad atada a la
barandilla (tryenter).  Si te caes (se lanza una excepcion), la cuerda te salva y te
lleva al punto de anclaje (el handler_pc).  Cuando cruzas el puente sin incidentes,
desatas la cuerda (tryleave) antes de continuar.

Sin `tryenter`/`tryleave`, el compilador tendria que emitir tablas de rangos estaticas
dentro del `MethodInfo` para describir que partes del codigo estan protegidas.  Con estas
instrucciones, los rangos se crean y destruyen en tiempo de ejecucion, lo que permite:

- Bloques try anidados de forma arbitraria
- Proteccion de bloques generados por el JIT sin reescribir el MethodInfo
- Combinacion con `swapctx` para implementar continuaciones o co-rutinas con manejo de errores

---

## Relacion con do_throw

Cuando se lanza una excepcion (instruccion `throw` o error interno de la VM), `do_throw`
busca el manejador en dos pasos:

1. **Primero** recorre la pila de ExceptionFrames dinamicos (`exc_frame_stack`).
   Si encuentra un frame cuyo campo `type` sea nullptr (catch-all) o sea la clase de la
   excepcion (o una superclase), salta al `handler_pc` almacenado en ese frame y
   desapila todos los frames hasta ese punto.

2. **Si no encuentra** nada en la pila dinamica, busca en la tabla estatica
   `MethodInfo.handlers` del metodo actual, como hacia la VM antes de este mecanismo.

Esto significa que `tryenter` tiene prioridad sobre las anotaciones estaticas.

---

## ExceptionFrame: la estructura en memoria host

```cpp
// Definido en include/runtime/proceso_runtime.h
struct ExceptionFrame {
    uint64_t    handler_pc;    // direccion VM absoluta del bloque catch
    ClassInfo  *type;          // tipo de excepcion capturado (nullptr = catch-all)
    ExceptionFrame *prev;      // enlace al frame anterior en la pila
};

// En ProcessVM:
ExceptionFrame *exc_frame_stack;   // tope de la pila; nullptr si la pila esta vacia
```

La pila de frames es una lista enlazada simple en memoria host (heap del sistema operativo,
no en el GC heap de la VM).  Cada `tryenter` hace `new ExceptionFrame` y lo encadena al
tope; cada `tryleave` hace `delete` del tope y restaura el enlace.

La pila vive en el proceso virtual (`ProcessVM`), por lo que cada proceso tiene su propio
conjunto de frames de excepcion, completamente aislado de otros procesos.

---

## Sintaxis y patron de uso

```asm
; Patron tipico emitido por un compilador de alto nivel:
;
;   try {
;       <bloque protegido>
;   } catch (SomeException e) {
;       <manejador>
;   }
;
; Se traduce a:

    mov   r1, @Absolute("all.catch_handler")  ; PC del bloque catch
    mov   r2, 0                                ; 0 = catch-all (nullptr ClassInfo*)
    tryenter r1, r2                            ; instalar frame

    ; --- bloque protegido ---
    ; cualquier excepcion dentro de aqui salta a catch_handler
    ; ...

    tryleave                                   ; salida normal: desinstalar frame
    jmp   @Absolute("all.after_try")           ; saltar el bloque catch

catch_handler:
    ; aqui se ejecuta si se lanzo una excepcion
    ; en r0 estara el objeto excepcion (segun convencion OOP de la VM)
    ; ...

after_try:
    ; codigo que sigue despues del try/catch
```

### Catch tipado

```asm
    mov   r1, @Absolute("all.io_handler")
    mov   r2, <ptr_a_ClassInfo_IOException>    ; capturar solo IOException
    tryenter r1, r2

    ; operaciones de I/O arriesgadas
    ; ...

    tryleave
    jmp   @Absolute("all.ok")

io_handler:
    ; solo se llega aqui si la excepcion es IOException o subclase
```

### Bloques try anidados

```asm
    ; bloque exterior: catch-all
    mov r1, @Absolute("all.outer_catch")
    mov r2, 0
    tryenter r1, r2

    ; bloque interior: captura IOError
    mov r1, @Absolute("all.inner_catch")
    mov r2, <ClassInfo_IOError>
    tryenter r1, r2

    ; codigo que puede lanzar distintas excepciones
    ; ...

    tryleave                ; desinstalar frame interior
    tryleave                ; desinstalar frame exterior

    ; los frames se revisan del mas reciente al mas antiguo
    ; (LIFO): inner_catch se revisa antes que outer_catch
```

---

## Encoding (nivel tecnico)

`tryenter` usa Convention B con `emit_str_two_reg`:

```
[0x00][0x44][byte2][0x00]
  byte2 = (r_handler << 4) | r_type
```

`tryleave` es FIXED_2 sin operandos:

```
[0x00][0x45]
```

El exec de `tryenter` extrae los registros con la convencion de byte2 crudo:

```cpp
const uint8_t r_handler = (instr.data_instruction.reg_data.reg1 >> 4) & 0xF; // nibble alto
const uint8_t r_type    =  instr.data_instruction.reg_data.reg1        & 0xF; // nibble bajo
```

Ambas instrucciones usan `decode_instr_raw_bytes` (Convention B).
`tryleave` usa `decode_fn = nullptr` en la tabla de decode porque no tiene operandos
y su tamano es FIXED_2 (no necesita leer bytes adicionales).

---

## Ciclo de vida del ExceptionFrame

```
tryenter                do_throw                   tryleave
    |                      |                           |
    v                      v                           v
new ExceptionFrame     recorre exc_frame_stack     delete top frame
enlaza al tope         si tipo coincide:            restaura prev al tope
                         salta a handler_pc
                         desapila frames intermedios
                         (delete cada uno)
```

Si el proceso termina (HLT o excepcion no capturada), el destructor de `ProcessVM`
debe limpiar los frames restantes para evitar fugas de memoria.  La implementacion
actual confia en que el codigo generado por el compilador siempre emita `tryleave`
en el camino de salida normal.

---

## Diferencia con MethodInfo.handlers

| Mecanismo               | Cuando se usa                                | Ventajas                                    |
| :---------------------: | :------------------------------------------- | :------------------------------------------ |
| `MethodInfo.handlers`   | Handlers conocidos en tiempo de compilacion  | Sin overhead en la ruta normal              |
| `tryenter`/`tryleave`   | Handlers dinamicos (JIT, closures, fibers)   | Flexibilidad total, sin anotaciones previas |

Para bytecode compilado estaticamente, `MethodInfo.handlers` es suficiente y mas eficiente.
`tryenter`/`tryleave` son utiles cuando el compilador JIT genera codigo a posteriori o
cuando se implementan abstracciones de control de flujo avanzadas (continuaciones,
corutinas con manejo de errores).
