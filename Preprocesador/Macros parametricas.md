Estas macros permite recibir argumentos y devolverlos bajo un nombre en clave.

Ejemplo:
```dart
// la funcion espera dos valores enteros y devolver un valor entero bajo el nombre
// de i, este tipo de macro solo valida si el tipo de dato es el que debe, y añade // el code indicado
#parametric int:x forEach(int n1, int n2) for (int x = n1; i < n2; i++) { 
	@wrapped_code 
}
```
 - `int:x` variable a externalizar de la macro.

La macro envuelve el código, `@wrapped_code` es el código contenido al hacer la llamada de la macro:
```c
 // forEach es una macro de tipo parametrica
 forEach(1, 10) { i-> // aqui i es la variable de la macro externalizada
	 print(i) // aqui i es el iterador del bucle
 }
```

resultado final del pre-procesado:
```c
for (int i = 0; i < 10; i++) { 
	 print(i) // aqui i es el iterador del bucle
}
```