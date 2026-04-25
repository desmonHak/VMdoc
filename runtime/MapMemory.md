# Mapeo de memoria entre instancias

El **mapeo de memoria** permite que una VmInstance acceda a la memoria de otra instancia
(local o remota) como si fuera su propia memoria. Esto habilita la comunicacion de
datos entre instancias sin necesidad de copias: las dos instancias "ven" la misma
region de memoria fisica bajo sus propias direcciones virtuales.

**Analogia:** es como dos oficinas que comparten una carpeta de documentos en red.
Cada oficina tiene su propia "ruta" al documento (direccion virtual diferente), pero
ambas acceden al mismo papel fisico (memoria real).

Una VmManager puede permitir que una VmInstance o varias mapeen memoria de otras instancias locales y remotas.

>Las instrucciones de la VM estan hechas para usar normalmente minimo `48bits` de direccionamiento, ya que en la actualidad la mayoria de CPU's tienen limitaciones y no pueden acceder a mas a pesar de lo que la teoria dicta. Por lo tanto solo se puede acceder a `256 TiB` de memoria con `48bits`.

Dentro de los tipos de memorias mapeadas, estan las memorias **volatiles** que se liberan al finalizar la instancia, y las memorias **persistentes virtuales** que se mapean a archivos directamente.

Las direccion virtuales de tipo host son aquellas que solo existen dentro de la misma maquina y tienen la siguiente forma: ``0x1000:0x1234567890ABCDEF``

La primera direccion (`0x1000`) equivale a la direccion virtual dentro de la instancia. La segunda direccion representar la direccion de memoria real que esta mapeada como pagina de memoria.

Si el [[VmManager]] gestiona la direccion se considera que es una direccion de memoria publica ya que otras instancias pueden acceder a la misma memoria mapeada bajo la misma direccion virtual, en cambio, si la direccion es gestionada por la instancia ([[VmInstance]]), se considera una direccion privada que solo una instancia o unas pocas selectas pueden acceder.

----
# Tipos de direcciones

Las direcciones de memoria pueden ser del tipo remoto de las cuales se diferencian dos tipos:  
## Direcciones remotas de tipo [[VmInstance]]

```
VMI<1>{0x2000:192.168.1.10:80@0x2000}
```

-  ``VMI``: indica que es una instancia VM ([[VmInstance]]) remota.
- ``<1>``: indica el id de la instancia remota. 
- ``0x2000``: le sigue la direccion virtual para la instancia actual.
- `192.168.1.10:80`: seguido de la direccion IP y puerto de la maquina remota.
- `0x2000`: direccion virtual remota dentro de la instancia remota.

## Direcciones remotas de tipo [[VmManager]]
El segundo tipo de direcciones remotas es la siguiente:

```
VMM{0x1000:192.168.1.10:80@0x1000}
```

- ``VMM`` indica que se trata de una direccion del manager (publica) en la maquina remota. 
>
>El resto de parametros es igual que con VMI
>

----

Se puede dar el caso de que el manager y una instancia tengan direcciones mismas direcciones virtuales pero mapeen distintas memoria, en ese caso la instancia debe mapear la direccion del manager como otra direccion virtual internamente.

La mejor manera de saber si la memoria fue remota fue modificada para temas de sincronizacion es tener un contador de escrituras que incremente por cada nueva escritura realizada, y consultar el valor en lugar de buscar cambios directos en la memoria.
![[VmManaggerMapMemory.png]]