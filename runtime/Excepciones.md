# Excepciones de software controladas por la VM

Una **excepcion** es un mecanismo para manejar situaciones de error de forma estructurada.
En lugar de comprobar manualmente si cada operacion fallo, el programa puede "lanzar"
una excepcion cuando algo va mal, y "capturarla" en un lugar superior de la pila de llamadas.

**Analogia:** es como el sistema de alarmas de un edificio. Si hay un incendio (error),
no tienes que ir puerta por puerta avisando: suena la alarma (se lanza la excepcion) y el
sistema de emergencias (el manejador de excepciones) toma el control.

En VestaVM, las excepciones se implementan mediante tres componentes:
- `ClassInfo`: la jerarquia de tipos de excepcion
- `HandlerException`: la tabla de rangos try/catch dentro de cada metodo
- `vm_throw`: el algoritmo de unwinding de la pila

---

## Estructuras de datos

```c
// registro global de clases
static ClassInfo* class_registry[];

/**
 * Modificadores de acceso
 */
typedef enum FieldAccess {
    FIELD_PUBLIC,
    FIELD_PRIVATE,
    FIELD_PROTECTED,
    FIELD_DEFAULT
} FieldAccess;

/**
 * Informacion de un campo
 */
typedef struct FieldInfo {
    stringx      name;          // nombre del campo
    FieldAccess  access;        // public/private/protected/default
    FieldKind    kind;          // tipo de dato
    ClassInfo*   type_class;    // si es FIELD_CLASS o FIELD_STRUCT
    uint32_t     size;          // tamano en bytes
    uint32_t     offset;        // offset dentro del objeto
    bool         is_static;     // si es un campo estatico
} FieldInfo;

/**
 * Tipos de campo
 * (pueden ser primitivos o clases definidas por el usuario)
 */
typedef enum FieldKind {
    FIELD_PRIMITIVE,
    FIELD_CLASS,
    FIELD_STRUCT,
    FIELD_TYPEDEF,
    FIELD_ENUM
} FieldKind;


/**
 * representacion de un string simple
 */
typedef struct stringx {
    uint8_t*    data;
    uint32_t    size;
} stringx;

/**
 * Informacion de las clases
 */
typedef struct ClassInfo {
    stringx              name;
    struct ClassInfo*   super;
    struct ClassInfo**  interfaces;
    size_t              interface_count;
    bool                is_exception;

    MethodInfo** methods;
    size_t method_count;

    FieldInfo** fields;
    size_t field_count;
} ClassInfo;

/**
 * Manejador de excepciones
 */
typedef struct HandlerException {
    ClassInfo*  type;        // null = catch-all
    uint32_t    start_pc;
    uint32_t    end_pc;
    uint32_t    handler_pc;
} HandlerException;

/**
 * Informacion del metodo
 */
typedef struct MethodInfo {
    stringx             name;
    HandlerException*   handlers;
    size_t              handler_count;
    uint8_t*            code;       // puntero al bytecode
    // aqui irian mas cosas: num locals, tamano de operand stack, etc.
} MethodInfo;

// Header de frame en la pila (en memoria de la VM)?
typedef struct FrameHeader {
    struct FrameHeader* prev;       // frame anterior (caller)
    MethodInfo*         method;     // metodo actual
    uint32_t            return_pc;  // PC al que volver si se hace return
    // despues de esto, en memoria, irian locals, operand stack, etc.
} FrameHeader;
```


```c
ClassInfo Throwable
    |
    +-- ClassInfo Exception
            |
            +-- ClassInfo IOException
                    |
                    +-- ClassInfo FileNotFoundException


ClassInfo Throwable = { STR("Throwable"), NULL, NULL, 0, true };
ClassInfo Exception = { STR("Exception"), &Throwable, NULL, 0, true };
ClassInfo IOException = { STR("IOException"), &Exception, NULL, 0, true };
ClassInfo FileNotFoundException = { STR("FileNotFoundException"), &IOException, NULL, 0, true };
```

```c
bool is_instance_of(ClassInfo* E, ClassInfo* C) {
    ClassInfo* current = E;

    while (current != NULL) {
        if (current == C)
            return true;

        for (size_t i = 0; i < current->interface_count; i++) {
            if (current->interfaces[i] == C)
                return true;
        }

        current = current->super;
    }

    return false;
}

void vm_checkcast(VM* vm, ExceptionObject* obj, ClassInfo* target_type) {
    if (obj == NULL) {
        // cast de null es valido -> no hace nada
        return;
    }

    if (!is_instance_of(obj->type, target_type)) {
        ExceptionObject* ex = malloc(sizeof(ExceptionObject));
        ex->type = &ClassCastExceptionClassInfo;
        vm_throw(vm, ex);
    }
}

//
// Crear objeto excepcion
//
ExceptionObject* vm_new_exception(ClassInfo* type /*, args... */) {
    ExceptionObject* ex = malloc(sizeof(ExceptionObject));
    ex->type = type;
    return ex;
}

//
// Busqueda de handler compatible en un metodo
//

HandlerException* find_handler_in_method(MethodInfo* method,
                                         uint32_t pc,
                                         ExceptionObject* ex)
{
    for (size_t i = 0; i < method->handler_count; i++) {
        HandlerException* h = &method->handlers[i];

        if (pc < h->start_pc || pc >= h->end_pc)
            continue;

        // finally / catch-all
        if (h->type == NULL)
            return h;

        // catch (Tipo e)
        if (is_instance_of(ex->type, h->type))
            return h;
    }

    return NULL;
}

//
// Manejo de excepcion no capturada
//

void vm_unhandled_exception(VM* vm, ExceptionObject* ex) {
    (void)vm;
    (void)ex;
    // Aqui imprimir stacktrace, tipo, etc
    // Por ahora, abortamos.
    abort();
}

//
// Unwind + salto al handler
//

void vm_throw(VM* vm, ExceptionObject* ex) {
    uint32_t pc = vm->pc;
    uint32_t bp = vm->bp;

    while (bp != 0) {
        FrameHeader* frame = (FrameHeader*)(vm->memory + bp);
        MethodInfo* method = frame->method;

        HandlerException* handler =
            find_handler_in_method(method, pc, ex);

        if (handler != NULL) {
            // 1 limpiar operand stack del frame actual
            //    Aqui podrias resetear vm->sp a un valor guardado en el frame.
            //    Para simplificar, asumimos que el handler sabe que esperar.

            // 2 empujar la excepcion en la pila de operandos
            vm_push_ptr(vm, ex);

            // 3 mover el PC al handler
            vm->pc = handler->handler_pc;

            // 4 seguimos ejecutando en este mismo frame
            vm->bp = bp;
            return;
        }

        // no hay handler en este frame -> desenrollar

        pc = frame->return_pc;      // PC del caller (si lo usas)
        bp = (uint32_t)(uintptr_t)frame->prev;
        vm->bp = bp;
        // vm->sp la ajustas segun tu convencion de llamada
    }

    // nadie la capturo -> uncaught exception
    vm_unhandled_exception(vm, ex);
}

// - El compilador genera HandlerException[] por metodo.
// - type = &Exception        -> catch (Exception e)  (captura subclases)
// - type = &Throwable        -> catch (Throwable e)  (captura todo)
// - type = NULL              -> finally / catch-all interno
```


## instanceof
Evalua si una clase E es instancia de C:
- Recorre la cadena de superclases hacia arriba
- Revisa interfaces
- Devuelve true si encuentra coincidencia

## checkcast
Valida que un objeto pueda convertirse a un tipo destino:
- null siempre es valido
- Si no es instancia -> lanza ClassCastException

## Busqueda de un handler compatible en un metodo
Reglas:

- El PC debe estar dentro del rango [start_pc, end_pc)
- Si type == NULL -> es un finally / catch-all
- Si type != NULL -> usar instanceof

# Algoritmo completo del unwind
Entrada:
- vm -> estado de la VM
- ex -> objeto excepcion

Salida:
- Salta al handler adecuado
- O aborta si nadie la captura

```c
1. Guardar el PC actual (vm->pc)
2. Guardar el BP actual (vm->bp)

3. Mientras BP != 0 (mientras haya frames en la pila):

    3.1 Obtener el frame actual:
        frame = memory[bp]

    3.2 Obtener el metodo del frame:
        method = frame->method

    3.3 Buscar un handler compatible:
        handler = find_handler_in_method(method, pc, ex)

    3.4 Si se encontro un handler:
        
        3.4.1 Limpiar la operand stack del frame actual
              (opcional segun tu diseno)

        3.4.2 Empujar la excepcion en la operand stack:
              push(ex)

        3.4.3 Mover el PC al handler:
              vm->pc = handler->handler_pc

        3.4.4 Mantener el mismo frame:
              vm->bp = bp

        3.4.5 Terminar el proceso (la ejecucion continua en el handler)

    3.5 Si NO se encontro handler:
        
        3.5.1 Desenrollar el frame:
              pc = frame->return_pc
              bp = frame->prev

        3.5.2 Actualizar vm->bp

4. Si se salio del bucle -> no hay handlers en ningun frame

5. Llamar a vm_unhandled_exception(vm, ex)



throw ex
|
v
buscar handler en metodo actual
|
v
?hay handler compatible?
    si -> saltar al handler
    no -> desenrollar frame
|
v
repetir en el caller
|
v
?se llego al tope de la pila?
    si -> excepcion no capturada -> abort


```



