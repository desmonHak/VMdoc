Una [[VmManager]] puede permitir que una [[VmInstance]] o varias mapeen memoria de otras instancias([[VmInstance|VmInstances]]) locales y remotas.

>Las instrucciones de la VM están hechas para usar normalmente mínimo `48bits` de direccionamiento, ya que en la actualidad la mayoría de CPU's tienen limitaciones y no pueden acceder a mas a pesar de lo que la teoría dicta. Por lo tanto solo se puede acceder a `256 TiB` de memoria con `48bits`.

Dentro de los tipos de memorias mapeadas, están las memorias **volátiles** que se liberan al finalizar la instancia, y las memorias **persistentes virtuales** que se mapean a archivos directamente.

Las dirección virtuales de tipo host son aquellas que solo existen dentro de la misma maquina y tienen la siguiente forma: ``0x1000:0x1234567890ABCDEF``

La primera dirección (`0x1000`) equivale a la dirección virtual dentro de la instancia. La segunda dirección representar la dirección de memoria real que esta mapeada como pagina de memoria.

Si el [[VmManager]] gestiona la dirección se considera que es una dirección de memoria publica ya que otras instancias pueden acceder a la misma memoria mapeada bajo la misma dirección virtual, en cambio, si la dirección es gestionada por la instancia ([[VmInstance]]), se considera una dirección privada que solo una instancia o unas pocas selectas pueden acceder.

----
# Tipos de direcciones

Las direcciones de memoria pueden ser del tipo remoto de las cuales se diferencian dos tipos:  
## Direcciones remotas de tipo [[VmInstance]]

```
VMI<1>{0x2000:192.168.1.10:80@0x2000}
```

-  ``VMI``: indica que es una instancia VM ([[VmInstance]]) remota.
- ``<1>``: indica el id de la instancia remota. 
- ``0x2000``: le sigue la dirección virtual para la instancia actual.
- `192.168.1.10:80`: seguido de la dirección IP y puerto de la maquina remota.
- `0x2000`: dirección virtual remota dentro de la instancia remota.

## Direcciones remotas de tipo [[VmManager]]
El segundo tipo de direcciones remotas es la siguiente:

```
VMM{0x1000:192.168.1.10:80@0x1000}
```

- ``VMM`` indica que se trata de una dirección del manager (publica) en la maquina remota. 
>
>El resto de parámetros es igual que con VMI
>

----

Se puede dar el caso de que el manager y una instancia tengan direcciones mismas direcciones virtuales pero mapeen distintas memoria, en ese caso la instancia debe mapear la dirección del manager como otra dirección virtual internamente.

La mejor manera de saber si la memoria fue remota fue modificada para temas de sincronización es tener un contador de escrituras que incremente por cada nueva escritura realizada, y consultar el valor en lugar de buscar cambios directos en la memoria.
![[VmManaggerMapMemory.png]]