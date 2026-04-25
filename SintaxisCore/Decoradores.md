# Decoradores en Vesta

Un **decorador** es una anotacion que se coloca sobre una clase, metodo o campo para
modificar su comportamiento o agregar metadatos sin cambiar el codigo interno.

**Analogia:** piensa en una etiqueta adhesiva que pones en un producto. El producto
no cambia, pero la etiqueta dice algo sobre el (p.ej. "fragil", "precio reducido").
En Vesta, los decoradores son esas etiquetas que el compilador o el runtime leen para
aplicar comportamiento especial.

Los decoradores se escriben con el prefijo `@` seguido del nombre y, opcionalmente,
parametros entre parentesis.

---

## Decoradores de clase

### `@Override`

Indica que un metodo sobreescribe el de la clase padre. El compilador verifica que
el metodo realmente exista en la clase padre (error si no existe).

```c
class Animal {
    public String sonido() {
        return "...";
    }
}

class Perro : Animal {

    @Override
    public String sonido() {  // sobreescribe Animal.sonido()
        return "Guau";
    }
}
```

### `@NotExtendObject`

Por defecto todas las clases heredan de `Object`. Este decorador lo desactiva,
creando una clase que no hereda de nada (util para tipos de datos puros o FFI):

```c
@NotExtendObject
class PuntoRaw {
    float x;
    float y;
    // No tiene toString(), hashCode(), equals() de Object
}
```

### `@Final`

Impide que la clase sea extendida (heredada). El compilador lanzara un error si
alguien intenta crear una subclase:

```c
@Final
class Singleton {
    // Nadie puede heredar de esta clase
}
```

### `@Abstract`

Marca la clase como abstracta: no se pueden crear instancias directamente, solo
subclases concretas. Es el equivalente a `abstract class` en Java:

```c
@Abstract
class Figura {
    public float area();  // metodo abstracto (sin cuerpo)
}

class Circulo : Figura {
    float radio;

    @Override
    public float area() {
        return 3.14159 * radio * radio;
    }
}
```

---

## Decoradores de metodos

### `@Deprecated`

Marca un metodo como obsoleto. El compilador emite una advertencia cuando se usa:

```c
class MiAPI {

    @Deprecated
    public void metodoViejo() {
        // usar metodoNuevo() en su lugar
    }

    public void metodoNuevo() { }
}
```

### `@Synchronized`

El runtime adquiere el monitor del objeto (`monenter`) antes de entrar al metodo
y lo libera (`monexit`) al salir. Equivalente a `synchronized` en Java:

```c
class Contador {
    uint64_t valor = 0;

    @Synchronized
    public void incrementar() {
        valor++;  // seguro en entornos multi-hilo
    }
}
```

---

## Decoradores del ensamblador (preprocesador)

Estos decoradores son procesados por el preprocesador del ensamblador Vesta antes
de la compilacion. No generan bytecode directamente sino que configuran el emitter:

| Decorador            | Efecto                                                          |
| :------------------- | :-------------------------------------------------------------- |
| `@Module(nombre)`    | Declara el modulo del archivo                                   |
| `@Export(Simbolo)`   | Exporta el simbolo como publico                                 |
| `@Generic(T)`        | Registra parametros de tipo para monomorphization               |
| `@Format("velb")`    | Indica el formato de salida del archivo compilado               |
| `@SpaceAddress { }`  | Define el espacio de direcciones del ejecutable                 |
| `@Section { }`       | Define secciones de codigo y datos                              |
| `@Lib("ruta")`       | Importa una libreria nativa (FFI)                               |
| `@Import("funcion")` | Importa una funcion especifica de una libreria                  |

Para mas detalle sobre los decoradores del ensamblador ver
[[Anotaciones modulo y genericos.md]] y [[../Preprocesador/Macros anotaciones.md]].

---

## Decoradores personalizados

Vesta permite definir decoradores propios como clases especiales:

```c
// Definir un decorador personalizado
@Interface
class MiDecorador {
    String descripcion;

    // Constructor del decorador
    MiDecorador(String desc) {
        this.descripcion = desc;
    }

    // Metodo que se aplica a la clase o metodo decorado
    void apply(Object target) {
        // logica del decorador
    }
}

// Usar el decorador
@MiDecorador("Esta clase hace algo especial")
class MiClase {
    // ...
}
```

---

## Orden de evaluacion

Cuando una clase o metodo tiene varios decoradores, se aplican de abajo a arriba
(el decorador mas cercano al elemento se aplica primero):

```c
@Decorador1         // se aplica tercero
@Decorador2         // se aplica segundo
@Decorador3         // se aplica primero (el mas cercano)
class MiClase { }
```

---

Ver tambien:
- [[POO.md]] - clases, herencia y modificadores de acceso
- [[Anotaciones modulo y genericos.md]] - anotaciones del ensamblador
- [[../Preprocesador/Macros anotaciones.md]] - macros anotacion del preprocesador
- [[SetInstruccionesVM/MONITOR.md]] - instrucciones monenter/monexit para @Synchronized
