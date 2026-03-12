La VM usa su propia convención de llamadas para conocer cuantos parámetros se han usado al hacer la llamada a una función interna (código que esta dentro de la VM) o una función externa (función real que no es parte de la VM y es necesario generar código para realizar la llamada).

Registros:
- `r15`: cantidad de parámetros de la función.
- `r00-r05`: los 6 primeros registros se usan para pasar los 6 primeros parámetros de la función En caso de necesitar mas, se colocaran en el stack.