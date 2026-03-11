```java
class Animal: Nombre {
	uint8_t age = 0;

	public Animal(uint8_t age) {
		this.age = age
	}
    

	// Constructor por defecto que llama al constructor principal con edad 0
	public Animal() => this(0); 

	public void method1() {
	}
	
	private set age(uint8_t age) => this.age = age;
	private get age     		 => age;

	public ~Animal() {
	}

	@Override
	toString() {
		return "Animal ${age}"
	}
}
```

los métodos sobrescritos como `toString` no necesitan indicar los valores a devolver aunque, ya que al ser un método sobrescrito se conoce los tipos necesarios y los nombres de los parámetros, será entonces necesario solo indicar el nombre de los parámetros en el método por cuestión de legibilidad.

----
## Modificadores de acceso.
se podrá indicar modificadores de acceso a los métodos y atributos de una clase. Los modificadores disponibles son los siguientes:


| Nombre del modificador | funcionalidad                                                                         |
| ---------------------- | ------------------------------------------------------------------------------------- |
| `public`               | acceso publico al método o atributo en todas las unidades de compilación              |
| `private`              | acceso privado al método o privado, no visible mas que por la propia clase            |
| `protect`              | los métodos y atributos solo podrán ser accedidos pos la clase padre y la subclase    |
| `default`              | Permiso por defecto, el valor es visible por todos en una misma unidad de compilacion |

Se puede definir los modificadores de acceso de forma individual (en linea) o de forma ``multi-linea`` (en bloque):
```java

class Animal {

	public {
		uint8_t dato1;
		uint8_t dato2;
	}

	public uint8_t dato3;


	public {
	
		uint8_t age;
	
		Animal(uint8_t age) => this.age = age;
		
		get age => age;
	}
	
	private {
		set age(it) => age = it;
		
		void run() {
			/* TODO */
		}
	}
}
```
También se permite el bloque al estilo C++

## Herencia múltiple

Por defecto toda clase hereda de la clase padre ``Object`` que implementa un conjunto de métodos y atributos útiles.

```java

class Animal { 
	String nombre;
	/* TODO */
}

@NotExtendObject
class Nombre { 
	String nombre;
	/* TODO */ 
}

class Perros: Animal { /* TODO */ }

class Draco: Perros, Nombre { 
	String nombre;
	/* TODO */ 
	Draco(String nombre) => {
		Perros::nombre = nombre; // cambiamos al atributo nombre de la clase Perros
		Nombre.nombre = nombre; // cambiamos el atributo nombre de la clase nombre
		this.nombre = nombre; // cambia el nombre de la clase actual, tambien se 
			// puede usar this::nombre o Draco::nombre, this siempre hara 
			// referencia a la clase actual.
	}
}
```

Con la notación `@NotExtendObject` se podrá evitar que tu clase herede de `Object` si no deseas esto.
Tanto la sintaxis `Nombre.nombre` como `Nombre::nombre` so igual de validas.