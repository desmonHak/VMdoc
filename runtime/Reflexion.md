# ¿Qué suele permitir hacer la reflexión en un lenguaje?
La reflexión permite que un programa inspeccione y manipule su propia estructura en tiempo de ejecución.

En la práctica, esto se traduce en varias capacidades clásicas:

## Inspección de clases (Class Introspection)
Permite preguntar:

1. ¿Cómo se llama esta clase?
2. ¿Cuál es su superclase?
3. ¿Qué interfaces implementa?
4. ¿Es una excepción?
5. ¿Qué métodos tiene?
6. ¿Qué campos tiene?

## Inspección de métodos (Method Introspection)
Permite:
1. Obtener el nombre del método
2. Obtener su bytecode
3. Obtener su tabla de excepciones
4. Saber cuántos handlers tiene
5. Saber qué tipos captura cada handler

## Inspección de excepciones y handlers
Puedes preguntar:

1. ¿Qué excepciones captura este método?
2. ¿Qué rango de instrucciones protege cada try?
3. ¿Dónde empieza cada catch?
4. ¿Qué tipo captura cada catch?
5. ¿Tiene finally? (type = ``NULL``)
6. Esto es reflexión sobre el flujo de control.

## Inspección del stack (Stack Introspection)
1. Permite obtener:
2. El método actual
3. El ``PC`` actual
4. El método llamador
5. La cadena completa de llamadas (stacktrace)
6. Los frames activos

## Manipulación de objetos en tiempo de ejecución
La reflexión avanzada permite:

1. Crear objetos dinámicamente
2. Invocar métodos dinámicamente
3. Acceder a campos dinámicamente
4. Modificar valores de campos
5. Crear instancias de clases cuyo nombre se conoce en tiempo de ejecución

Esto es tan simple como:
- tener un mapa string -> ``ClassInfo*``
- tener un constructor genérico ``vm_new_object(ClassInfo*)``
- tener un sistema de campos en ``ClassInfo``

## Comprobación dinámica de tipos
1. ``instanceof``
2. ``checkcast``

## Inspección del bytecode
```ClassInfo``` y ``MethodInfo`` contienen:

Eso significa que se puede:

- leer instrucciones
- analizar el flujo
- generar herramientas como desensambladores
- hacer análisis estático en tiempo de ejecución

Esto es reflexión a nivel de código.



-----


## 1 Convenciones básicas
- ``r0`` suele usarse como registro de resultado.
- ``r1``, ``r2``, … para parámetros.
- La pila se usa cuando no caben todos los parámetros en registros o para valores temporales.
- ``handle`` = referencia a un objeto de la VM (por ejemplo, ``ClassObject``, ``FieldObject``, etc.).

## 2 Instrucciones de reflexión sobre clases
### 2.1 Obtener clase por nombre
```c
GET_BY_NAME r_dest, r_name
```
Entrada:
``r_name`` -> ``handle`` a un string con el nombre de la clase
Salida:
``r_dest`` -> ``handle`` a ``ClassObject`` (o ``null`` si no existe)

### 2.2 Obtener nombre de una clase
```c
GET_NAME r_dest, r_class
```
Entrada:
``r_class`` -> ``handle`` a ``ClassObject``

Salida:
``r_dest`` -> ``handle`` a ``string`` con el nombre

### 2.3 Obtener superclase
```c
GET_SUPER r_dest, r_class
```

Entrada:
``r_class`` -> ``handle`` a ``ClassObject[]``

Salida:
``r_dest`` -> ``handle`` a ``ClassObject`` de la superclase (o ``null``)

2.4 Obtener número de interfaces
```c
GET_INTERFACE_COUNT r_dest, r_class
```
Entrada:
``r_class`` -> ``handle`` a ``ClassObject``

Salida:
``r_dest`` -> entero con el número de interfaces

2.5. Obtener interfaz por índice
```c
GET_INTERFACE_AT r_dest, r_class, r_index
```
Entrada:
``r_class`` -> ``handle`` a ``ClassObject``
``r_index`` -> índice (``0``..``n-1``)

Salida:
``r_dest`` -> ``handle`` a ``ClassObject`` de la interfaz

2.6. Obtener número de campos
```c
GET_FIELD_COUNT r_dest, r_class
```
Entrada:
``r_class`` -> ``handle`` a ``ClassObject``

Salida:
``r_dest`` -> entero con el número de campos

2.7. Obtener campo por índice
```c
GET_FIELD_AT r_dest, r_class, r_index
```
Entrada:
``r_class`` -> ``handle`` a ``ClassObject``
``r_index`` -> índice

Salida:
``r_dest`` -> ``handle`` a ``FieldObject``

2.8. Obtener número de métodos
```c
GET_METHOD_COUNT r_dest, r_class
```
Entrada:
``r_class`` -> ``handle`` a ``ClassObject``

Salida:
``r_dest`` -> entero

2.9. Obtener método por índice
```c
GET_METHOD_AT r_dest, r_class, r_index
```
Entrada:
``r_class`` -> ``handle`` a ``ClassObject``
``r_index`` -> índice

Salida:
``r_dest`` -> ``handle`` a ``MethodObject``

3. Instrucciones de reflexión sobre campos
3.1. Obtener nombre del campo
```c
GET_NAME r_dest, r_field
```
Entrada: ``r_field`` -> ``FieldObject``
Salida: ``r_dest`` -> ``string``

3.2. Obtener tipo del campo
```c
GET_TYPE r_dest, r_field
```
Entrada: ``r_field`` -> ``FieldObject``
Salida: ``r_dest`` -> ``ClassObject`` del tipo

3.3. Obtener modificador de acceso
```c
GET_ACCESS r_dest, r_field
```
Entrada: ``r_field`` -> ``FieldObject``
Salida: ``r_dest`` -> entero (``FIELD_PUBLIC``, etc.)

3.4. Saber si es estático
```c
IS_STATIC r_dest, r_field
```
Entrada: ``r_field`` -> ``FieldObject``
Salida: ``r_dest`` -> ``bool``

3.5. Leer campo de instancia
```c
GET r_dest, r_obj, r_field
```
Entrada:
``r_obj`` -> objeto de la VM
``r_field`` -> ``FieldObject``

Salida:
``r_dest`` -> valor del campo (tipo depende del campo; puedes usar convención de pila si es complejo)

3.6. Escribir campo de instancia
```c
SET r_obj, r_field, r_value
```
Entrada:
``r_obj`` -> ``objeto``
``r_field`` -> ``FieldObject``
``r_value`` -> valor (o en pila si es grande)

Salida: ninguna

3.7. Leer campo estático
```c
GET_STATIC r_dest, r_field
```
Entrada: ``r_field`` -> ``FieldObject``
Salida: ``r_dest`` -> valor

3.8. Escribir campo estático
```c
FIELD.SET_STATIC r_field, r_value
```
Entrada: r_field, r_value

Salida: ninguna

## 4. Instrucciones de reflexión sobre métodos
4.1. Obtener nombre del método
```c
METHOD.GET_NAME r_dest, r_method
```
Entrada: ``r_method`` -> ``MethodObject``
Salida: ``r_dest`` -> string

4.2. Invocar método de instancia
```c
METHOD.INVOKE r_result, r_method, r_obj, r_argc
```
Entrada:
``r_method`` -> ``MethodObject``
``r_obj`` -> objeto (o ``null`` si estático)
``r_argc`` -> número de argumentos
argumentos -> en pila o en registros (r2..rn)

Salida:
``r_result`` -> valor de retorno (o en pila)

Internamente:
la VM:
crea un FrameHeader
ajusta bp, sp, pc
salta al ``MethodInfo->code``

## 5. Instrucciones para creación dinámica de tipos

5.1. Crear clase vacía
```c
CLASS.NEW r_dest, r_name, r_super
```
Entrada:
``r_name`` -> ``string``
``r_super`` -> ``ClassObject`` o ``null``

Salida:
``r_dest`` -> ``ClassObject`` recién creado

5.2. Añadir campo
```c
CLASS.ADD_FIELD r_dest, r_class, r_name, r_type, r_access, r_kind, r_is_static
```

Entrada:
``r_class`` -> ``ClassObject``
``r_name`` -> ``string``
``r_type`` -> ``ClassObject`` del tipo (o código primitivo)
``r_access`` -> entero (``FIELD_PUBLIC``, etc.)
``r_kind`` -> entero (``FIELD_PRIMITIVE``, etc.)
``r_is_static`` -> ``bool``

Salida:
``r_dest`` -> ``FieldObject`` (opcional)

5.3. Añadir método
```c
CLASS.ADD_METHOD r_dest, r_class, r_name, r_code_ref
```
Entrada:

``r_class`` -> ``ClassObject``
``r_name`` -> ``string``
``r_code_ref`` -> referencia a bloque de bytecode (como tú lo representes)

Salida:
``r_dest`` -> ``MethodObject``

## 6. Instrucciones de tipos y excepciones
las formalizo:

6.1. INSTANCEOF
```c
TYPE.INSTANCEOF r_dest, r_obj, r_class
```
Entrada:
``r_obj`` -> objeto
``r_class`` -> ``ClassObject``

Salida:
``r_dest`` -> bool

Internamente usa ``is_instance_of``.

6.2. CHECKCAST
```c
TYPE.CHECKCAST r_obj, r_class
```
Entrada:
``r_obj`` -> ``objeto``
``r_class`` -> ``ClassObject``

Salida:
nada

Efecto:
si no es instancia -> lanza ``ClassCastException`` vía ``vm_throw``

6.3. ``THROW``
```c
EXC.THROW r_ex
```
Entrada:

``r_ex`` -> ``handle`` a ``ExceptionObject``

Salida:
no vuelve si la excepción no se captura

Efecto:
llama a ``vm_throw(vm, ex)``
