# Reflexion en VestaVM

La **reflexion** permite que un programa inspeccione y manipule su propia estructura
en tiempo de ejecucion: obtener el nombre de una clase, listar sus metodos, invocar
un metodo cuyo nombre se conoce solo en tiempo de ejecucion, etc.

**Analogia:** es como tener un espejo dentro del programa. El programa puede "mirarse"
a si mismo y ver como esta construido, en lugar de solo ejecutarse sin introspection.

En VestaVM, la reflexion es posible porque cada clase cargada tiene un `ClassInfo*`
con toda la informacion de sus campos y metodos, y ese puntero es accesible desde
el bytecode via la instruccion `TYPESWITCH` o directamente desde el heap GC.

---

## Que suele permitir hacer la reflexion en un lenguaje?

La reflexion permite que un programa inspeccione y manipule su propia estructura en tiempo de ejecucion.

En la practica, esto se traduce en varias capacidades clasicas:

## Inspeccion de clases (Class Introspection)
Permite preguntar:

1. ?Como se llama esta clase?
2. ?Cual es su superclase?
3. ?Que interfaces implementa?
4. ?Es una excepcion?
5. ?Que metodos tiene?
6. ?Que campos tiene?

## Inspeccion de metodos (Method Introspection)
Permite:
1. Obtener el nombre del metodo
2. Obtener su bytecode
3. Obtener su tabla de excepciones
4. Saber cuantos handlers tiene
5. Saber que tipos captura cada handler

## Inspeccion de excepciones y handlers
Puedes preguntar:

1. ?Que excepciones captura este metodo?
2. ?Que rango de instrucciones protege cada try?
3. ?Donde empieza cada catch?
4. ?Que tipo captura cada catch?
5. ?Tiene finally? (type = ``NULL``)
6. Esto es reflexion sobre el flujo de control.

## Inspeccion del stack (Stack Introspection)
1. Permite obtener:
2. El metodo actual
3. El ``PC`` actual
4. El metodo llamador
5. La cadena completa de llamadas (stacktrace)
6. Los frames activos

## Manipulacion de objetos en tiempo de ejecucion
La reflexion avanzada permite:

1. Crear objetos dinamicamente
2. Invocar metodos dinamicamente
3. Acceder a campos dinamicamente
4. Modificar valores de campos
5. Crear instancias de clases cuyo nombre se conoce en tiempo de ejecucion

Esto es tan simple como:
- tener un mapa string -> ``ClassInfo*``
- tener un constructor generico ``vm_new_object(ClassInfo*)``
- tener un sistema de campos en ``ClassInfo``

## Comprobacion dinamica de tipos
1. ``instanceof``
2. ``checkcast``

## Inspeccion del bytecode
```ClassInfo``` y ``MethodInfo`` contienen:

Eso significa que se puede:

- leer instrucciones
- analizar el flujo
- generar herramientas como desensambladores
- hacer analisis estatico en tiempo de ejecucion

Esto es reflexion a nivel de codigo.



-----


## 1 Convenciones basicas
- ``r0`` suele usarse como registro de resultado.
- ``r1``, ``r2``, ... para parametros.
- La pila se usa cuando no caben todos los parametros en registros o para valores temporales.
- ``handle`` = referencia a un objeto de la VM (por ejemplo, ``ClassObject``, ``FieldObject``, etc.).

## 2 Instrucciones de reflexion sobre clases
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

2.4 Obtener numero de interfaces
```c
GET_INTERFACE_COUNT r_dest, r_class
```
Entrada:
``r_class`` -> ``handle`` a ``ClassObject``

Salida:
``r_dest`` -> entero con el numero de interfaces

2.5. Obtener interfaz por indice
```c
GET_INTERFACE_AT r_dest, r_class, r_index
```
Entrada:
``r_class`` -> ``handle`` a ``ClassObject``
``r_index`` -> indice (``0``..``n-1``)

Salida:
``r_dest`` -> ``handle`` a ``ClassObject`` de la interfaz

2.6. Obtener numero de campos
```c
GET_FIELD_COUNT r_dest, r_class
```
Entrada:
``r_class`` -> ``handle`` a ``ClassObject``

Salida:
``r_dest`` -> entero con el numero de campos

2.7. Obtener campo por indice
```c
GET_FIELD_AT r_dest, r_class, r_index
```
Entrada:
``r_class`` -> ``handle`` a ``ClassObject``
``r_index`` -> indice

Salida:
``r_dest`` -> ``handle`` a ``FieldObject``

2.8. Obtener numero de metodos
```c
GET_METHOD_COUNT r_dest, r_class
```
Entrada:
``r_class`` -> ``handle`` a ``ClassObject``

Salida:
``r_dest`` -> entero

2.9. Obtener metodo por indice
```c
GET_METHOD_AT r_dest, r_class, r_index
```
Entrada:
``r_class`` -> ``handle`` a ``ClassObject``
``r_index`` -> indice

Salida:
``r_dest`` -> ``handle`` a ``MethodObject``

3. Instrucciones de reflexion sobre campos
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

3.4. Saber si es estatico
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
``r_dest`` -> valor del campo (tipo depende del campo; puedes usar convencion de pila si es complejo)

3.6. Escribir campo de instancia
```c
SET r_obj, r_field, r_value
```
Entrada:
``r_obj`` -> ``objeto``
``r_field`` -> ``FieldObject``
``r_value`` -> valor (o en pila si es grande)

Salida: ninguna

3.7. Leer campo estatico
```c
GET_STATIC r_dest, r_field
```
Entrada: ``r_field`` -> ``FieldObject``
Salida: ``r_dest`` -> valor

3.8. Escribir campo estatico
```c
FIELD.SET_STATIC r_field, r_value
```
Entrada: r_field, r_value

Salida: ninguna

## 4. Instrucciones de reflexion sobre metodos
4.1. Obtener nombre del metodo
```c
METHOD.GET_NAME r_dest, r_method
```
Entrada: ``r_method`` -> ``MethodObject``
Salida: ``r_dest`` -> string

4.2. Invocar metodo de instancia
```c
METHOD.INVOKE r_result, r_method, r_obj, r_argc
```
Entrada:
``r_method`` -> ``MethodObject``
``r_obj`` -> objeto (o ``null`` si estatico)
``r_argc`` -> numero de argumentos
argumentos -> en pila o en registros (r2..rn)

Salida:
``r_result`` -> valor de retorno (o en pila)

Internamente:
la VM:
crea un FrameHeader
ajusta bp, sp, pc
salta al ``MethodInfo->code``

## 5. Instrucciones para creacion dinamica de tipos

5.1. Crear clase vacia
```c
CLASS.NEW r_dest, r_name, r_super
```
Entrada:
``r_name`` -> ``string``
``r_super`` -> ``ClassObject`` o ``null``

Salida:
``r_dest`` -> ``ClassObject`` recien creado

5.2. Anadir campo
```c
CLASS.ADD_FIELD r_dest, r_class, r_name, r_type, r_access, r_kind, r_is_static
```

Entrada:
``r_class`` -> ``ClassObject``
``r_name`` -> ``string``
``r_type`` -> ``ClassObject`` del tipo (o codigo primitivo)
``r_access`` -> entero (``FIELD_PUBLIC``, etc.)
``r_kind`` -> entero (``FIELD_PRIMITIVE``, etc.)
``r_is_static`` -> ``bool``

Salida:
``r_dest`` -> ``FieldObject`` (opcional)

5.3. Anadir metodo
```c
CLASS.ADD_METHOD r_dest, r_class, r_name, r_code_ref
```
Entrada:

``r_class`` -> ``ClassObject``
``r_name`` -> ``string``
``r_code_ref`` -> referencia a bloque de bytecode (como tu lo representes)

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
si no es instancia -> lanza ``ClassCastException`` via ``vm_throw``

6.3. ``THROW``
```c
EXC.THROW r_ex
```
Entrada:

``r_ex`` -> ``handle`` a ``ExceptionObject``

Salida:
no vuelve si la excepcion no se captura

Efecto:
llama a ``vm_throw(vm, ex)``
