```c
Identificador ::= [a-z][0-9a-zA-z]*
```
el identificado debe empezar por una letra en minusculas y luego puede tener una cantidad indeterminada de numeros, letras mayusculas o minusculas.

```c
// forma 1 de representar
Entero ::= [-+]?[0-9]+

// forma 2 de representar
Entero ::= [-:\+]?[0-9]+
```
minimo tiene un digito o puede tener mas, puede empezar por un signo

```c
Flotante ::= [-+]?([0-9]+)?\.[0-9]+
```