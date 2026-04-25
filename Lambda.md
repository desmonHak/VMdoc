# Lambdas y Closures en VestaVM

VestaVM soporta funciones anonimas (lambdas) y closures como ciudadanos
de primera clase del sistema de tipos.  Se implementan mediante dos modelos
complementarios segun el contexto de uso:

---

## Modelo A: Closure GC (`mkclosure` / `callclosure`)

- El closure se almacena como un `ClosureObject` en el **GC heap**.
- El GC rastrea las variables capturadas durante los ciclos de recoleccion.
- Requiere un `MethodInfo*` del loader (solo disponible con infraestructura OOP).
- Adecuado para lambdas de alto nivel en codigo Vesta con clases y excepciones.

```c
// El loader ha puesto el MethodInfo* del lambda en r9
mov   r8, 0
mkclosure r9, r8       // R0 = GcHandle del ClosureObject
callclosure r0         // invocar el lambda; ret_addr en pila; FrameHeader creado
```

---

## Modelo B: Closure raw (`mkrawclosure` / `callrawclosure`)

- El closure se almacena como un `RawClosureObject` fuera del GC.
- Ciclo de vida manual: el bytecode llama a `FREE` cuando ya no se necesita.
- Solo necesita la direccion de la funcion; no requiere `MethodInfo`.
- Adecuado para callbacks FFI y funciones nativas registradas dinamicamente.

```c
mov   r1, @Absolute("all.mi_funcion")
mov   r2, 0                        ; sin entorno capturado
mkrawclosure r1, r2                ; R0 = puntero raw al RawClosureObject

mov   r10, r0
mov   r1, 42                       ; argumento
mov   r15, 1                       ; argc
callrawclosure r10                 ; invoca mi_funcion; env en R14
```

---

## Tabla comparativa

| Criterio              | Modelo GC              | Modelo Raw              |
| :-------------------- | :--------------------- | :---------------------- |
| Instrucciones         | `mkclosure/callclosure`| `mkrawclosure/callrawclosure` |
| Gestiona el GC        | Si                     | No (manual con FREE)    |
| Requiere MethodInfo*  | Si (del loader)        | No                      |
| Soporte THROW/CATCH   | Si (FrameHeader)       | No                      |
| Uso tipico            | Lambdas OOP internos   | Callbacks FFI           |

---

## Documentacion detallada

- [[SetInstruccionesVM/CLOSURE.md]] - Referencia completa de las cuatro
  instrucciones con codificacion binaria, algoritmos y ejemplos.

## Ejemplos

- [[../../examples_codes_vm/ejemplo_closure_raw.vel]] - Ejemplo ejecutable
  del modelo raw: closure que suma dos enteros.
- [[../../examples_codes_vm/ejemplo_closure_gc.vel]] - Esqueleto conceptual
  del modelo GC con MethodInfo del loader.
