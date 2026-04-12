La VM usa su propia convención de llamadas para conocer cuantos parámetros se han usado al hacer la llamada a una función interna (código que esta dentro de la VM) o una función externa (función real que no es parte de la VM y es necesario generar código para realizar la llamada).

Registros:
- ``r00``: valor de retorno de la llamada si retorna
- `r15`: cantidad de parámetros de la función. máximo de ``r1`` a ``r12`` = 12 argumentos
- `r01-r12`: los registros se usan para pasar los 12 parámetros de la función

```c
mov r15, 1 // este metodo usa 1 args
mov r1, hola_mundo // ponemos el puntero de la direccion a imprimir
calln @Method("libc.dll:puts")
// el valor retornado esta en r0
hlt

hola_mundo db "Hola mundo", 0x0
```
vea [[NativeCall (CallN)]]
