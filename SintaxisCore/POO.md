# Programacion Orientada a Objetos (POO) en Vesta

La **Programacion Orientada a Objetos** organiza el codigo en torno a **objetos** que
combinan datos (atributos) y comportamiento (metodos) en una misma entidad.

**Analogia:** un coche real tiene atributos (color, velocidad, nivel de combustible)
y comportamiento (acelerar, frenar, girar). En POO defines la "plantilla" del coche
(la clase `Coche`) y luego creas instancias concretas (objetos) a partir de esa plantilla.

En VestaVM, cada objeto en el heap tiene un **ObjectHeader de 24 bytes** que el runtime
usa para saber su tipo, gestionar el GC y controlar el acceso concurrente. Los campos
del objeto empiezan en el offset +24.

---

## Definicion de una clase

```c
class Animal {

    // Atributos (campos del objeto)
    String nombre;
    uint8_t edad;

    // Constructor principal
    public Animal(String nombre, uint8_t edad) {
        this.nombre = nombre;
        this.edad   = edad;
    }

    // Constructor por defecto: llama al principal con edad 0
    public Animal(String nombre) => this(nombre, 0);

    // Metodo publico
    public void saludar() {
        print("Hola, soy ");
        print(this.nombre);
    }

    // Destructor (llamado por el GC antes de liberar)
    public ~Animal() {
        // limpieza si es necesaria
    }

    // Sobreescritura de toString (heredado de Object)
    @Override
    toString() {
        return "Animal(" + nombre + ", " + edad + ")";
    }
}

// Crear instancias
Animal a1 = new Animal("Rex", 3);
Animal a2 = new Animal("Fluffy");  // edad = 0

a1.saludar();  // imprime "Hola, soy Rex"
```

---

## Getters y setters

Para controlar el acceso a los atributos, Vesta permite definir propiedades con
accesores `get` y `set`:

```c
class Persona {

    private uint8_t _edad;  // campo interno privado

    // Getter: acceso de lectura al campo
    public get edad => _edad;

    // Setter: acceso de escritura con validacion
    public set edad(uint8_t valor) {
        if (valor > 150) {
            throw new ArgumentException("Edad invalida");
        }
        _edad = valor;
    }

    public Persona(uint8_t edad) {
        this.edad = edad;  // usa el setter (con validacion)
    }
}

Persona p = new Persona(25);
uint8_t e = p.edad;   // usa el getter -> 25
p.edad = 30;          // usa el setter -> _edad = 30
```

---

## Modificadores de acceso

| Modificador | Visibilidad                                                              |
| :---------- | :----------------------------------------------------------------------- |
| `public`    | Accesible desde cualquier lugar                                          |
| `private`   | Solo accesible dentro de la propia clase                                 |
| `protect`   | Accesible desde la clase padre y sus subclases                           |
| `default`   | Accesible dentro de la misma unidad de compilacion (paquete)             |

Se pueden aplicar de forma individual o en bloque:

```c
class Animal {

    // Bloque public: todos los campos dentro son publicos
    public {
        String nombre;
        uint8_t edad;

        Animal(uint8_t edad) => this.edad = edad;
        get edad => edad;
    }

    // Bloque private
    private {
        set edad(uint8_t it) => edad = it;

        void calcular_peso() {
            // solo usable internamente
        }
    }

    // Tambien se puede indicar en linea
    public uint8_t dato_extra;
}
```

---

## Herencia

Una clase puede **heredar** de otra, obteniendo todos sus atributos y metodos.
Vesta soporta **herencia multiple** (heredar de varias clases a la vez):

```c
class Animal {
    String nombre;
    // ...
}

class Perro : Animal {
    // Hereda nombre de Animal
    uint8_t num_patas = 4;
    // ...
}

// Herencia multiple
class Draco : Perro, Nombre {
    String nombre;

    Draco(String nom) => {
        Perro::nombre  = nom;   // atributo nombre de la clase Perro
        Nombre::nombre = nom;   // atributo nombre de la clase Nombre
        this.nombre    = nom;   // atributo nombre de esta clase
    }
}
```

**Nota:** tanto `Nombre.campo` como `Nombre::campo` son sintaxis validas para
referenciar un campo de una clase especifica en caso de ambiguedad.

### `@NotExtendObject`

Por defecto toda clase hereda de `Object`. Para evitar esto:

```c
@NotExtendObject
class TipoPuro {
    // No tiene los metodos de Object (toString, hashCode, equals, ...)
    // No participa en el GC de la misma forma
}
```

---

## Polimorfismo y sobreescritura

Una subclase puede sobreescribir los metodos de la clase padre:

```c
class Animal {
    public String sonido() {
        return "...";
    }
}

class Perro : Animal {
    @Override
    public String sonido() {  // reemplaza Animal.sonido()
        return "Guau";
    }
}

class Gato : Animal {
    @Override
    public String sonido() {
        return "Miau";
    }
}

// Polimorfismo: la llamada al metodo correcto depende del tipo real del objeto
Animal a = new Perro();
a.sonido();  // "Guau" (no "...")
```

Esto funciona porque VestaVM usa una **tabla virtual (vtable)** para resolver el
metodo correcto en tiempo de ejecucion usando `CALLVIRT`.

---

## Interfaces y clases abstractas

```c
// Clase abstracta: no se puede instanciar directamente
@Abstract
class Figura {
    public float area();     // metodo abstracto: subclases deben implementarlo
    public float perimetro();
}

class Circulo : Figura {
    float radio;

    @Override
    public float area() {
        return 3.14159 * radio * radio;
    }

    @Override
    public float perimetro() {
        return 2.0 * 3.14159 * radio;
    }
}
```

---

## Representacion en memoria (ObjectHeader v2)

Cada objeto en el heap GC tiene un `ObjectHeader` de 24 bytes antes de sus propios campos:

```c
// Layout en memoria de un objeto Animal:
//
// offset  0 : class_ptr  (8 bytes) -- puntero a ClassInfo de Animal
// offset  8 : flags      (4 bytes) -- OBJ_FLAG_GC_OWNED, etc.
// offset 12 : hash_code  (4 bytes) -- codigo hash del objeto
// offset 16 : owner_pid  (4 bytes) -- PID propietario del monitor (0 = libre)
// offset 20 : lock_depth (2 bytes) -- profundidad de bloqueo reentrante
// offset 22 : _mon_pad   (2 bytes) -- relleno de alineacion
// offset 24 : nombre     (primer campo de Animal)
// offset 32 : edad       (segundo campo de Animal)
```

Los campos del usuario comienzan siempre en el offset **+24** (no +16 como en versiones
anteriores del ABI).

---

Ver tambien:
- [[Decoradores.md]] - @Override, @Abstract, @Final, @Synchronized
- [[Estructuras (struct).md]] - tipos de valor sin herencia OOP
- [[SetInstruccionesVM/GC/Generacional (para objetos OOP).md]] - ciclo de vida de objetos
- [[SetInstruccionesVM/MONITOR.md]] - sincronizacion con monenter/monexit
- [[SetInstruccionesVM/OOP/GENERICS.md]] - tipos genericos
