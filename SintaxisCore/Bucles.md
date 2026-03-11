## While
```c
uint8_t i = 0;
while (i < 10) {
	i++;
}
```

## Loop 
```c
loop {
	/* bucle infinito hasta q ocurra una break */
}
```
## Do-While

```c
uint8_t i = 0;
do {
	i++;
} while(i < 10);

while (i < 10) do {
	i++;
} 
```

## For
```java

// primera forma valida 1
for (uint8_t i = 0; i < 10; i++) {
}

// segunda forma valida, for-in
for (uint8_t i: range(0, 10))
/**
 - range es una macro que genera un array de valores de 0 a 10
 */
 
 // forEach es una macro de tipo parametrica
 forEach(1, 10) { i->
	 print(i) // aqui es el iterador
 }
```

## Switch-case

```java

String text = "Hola mundo";
switch (text) {

	case "texto1": 
		/* TODO */ 
		break;
		
	// si se usa la forma de llaves, no sera necesario indicar un break,
	// si se pusiera algun break, el caso finalizaria en el lugar realizado
	case "texto2": {
		/* TODO */
	}
	
	// si ocurre un caso donde se de texto3 o 4 se ejecutara el mismo code
	case "texto3":
	case "texto4": {
		/* TODO */
	}
	
	// otra forma de expresar varias condicones en un caso
	case [
		"texto5",
		"texto6",
		"texto7"
	]: {
	}
	
	// tambien de puede poner default, y tambien se puede usar llaves en lugar
	// de sintaxis flecha
	_ => /* TODO */

}
```