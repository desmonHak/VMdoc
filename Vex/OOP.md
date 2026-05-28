# Programacion Orientada a Objetos en Vex

Vex implementa POO dinamica con reflexion en runtime. Las clases no se persisten como
metadata estatica en el `.velb`; el bytecode las registra en el `ClassRegistry` del
`Loader` durante la ejecucion de `__module_init` (que corre antes de `main`).

Esto unifica clases estaticas, dinamicas y las cargadas con `loadmodule` en una sola ruta.

---

## Definicion de clases

```java
class Animal {
    // Campos de instancia
    public string nombre;
    private i32 edad;

    // Campos estaticos
    private static u64 contador = 0;

    // Constructor por defecto con delegacion
    public Animal() => this("Sin nombre", 0);

    // Constructor principal
    public Animal(string nombre, i32 edad) {
        this.nombre = nombre;
        this.edad = edad;
        Animal.contador += 1;
    }

    // Metodo de instancia
    public i32 getEdad() {
        return this.edad;
    }

    // Metodo estatico
    public static u64 getCuenta() {
        return Animal.contador;
    }

    // Destructor (RAII: se llama al exit del scope del propietario)
    public ~Animal() {
        println("Animal ${nombre} destruido");
    }

    // toString para depuracion
    @Override
    public string toString() {
        return "Animal(${nombre}, ${edad})";
    }
}
```

### Instanciacion

```java
Animal a = new Animal("Rex", 5);
Animal b = new Animal("Fido", 3);

println("a.getEdad() = ${a.getEdad()}"); // 5
println("total animales = ${Animal.getCuenta()}");// 2
```

---

## Modificadores de acceso

| Modificador | Acceso desde |
| :----------- | :----------------------------------------------- |
| `public` | Cualquier lugar |
| `private` | Solo la clase que lo define |
| `protected` | La clase y sus subclases |
| (ninguno) | Equivale a `public` en el estado actual |

---

## Keyword `final`

```java
// Clase no heritable:
final class Punto {
    public f64 x;
    public f64 y;
}

class Base {
    // Metodo no sobrescribible:
    public final void metodoFinal() {
        println("no se puede sobrescribir");
    }

    public void metodoNormal() { ... }
}
```

---

## Herencia simple

```java
class Perro : Animal {
    private string raza;

    public Perro(string nombre, i32 edad, string raza) {
        super(nombre, edad); // llamar al constructor del super
        this.raza = raza;
    }

    @Override // OBLIGATORIO si sobrescribe un metodo
    public string toString() {
        return "Perro(${nombre}, ${edad}, ${raza}) - super: ${super.toString()}";
    }

    // Metodo propio de Perro
    public void ladrar() {
        println("${nombre}: GUAU!");
    }
}
```

**Nota:** `@Override` es obligatorio al sobrescribir. Su ausencia es un error de
compilacion. Esto evita errores silenciosos por diferencia de firma.

### Ejemplo de polimorfismo

```java
Animal[] zoo = {new Animal("Gato", 2), new Perro("Rex", 5, "Labrador")};

for (Animal a : zoo) {
    println(a.toString()); // dispatch virtual correcto
}
```

---

## Interfaces

```java
interface Saludable {
    void saludar();
    string obtenerSaludo();
}

interface Identificable {
    i64 getId();
}

class Persona : Animal, Saludable, Identificable {
    private i64 id;

    public Persona(string nombre, i32 edad, i64 id) {
        super(nombre, edad);
        this.id = id;
    }

    @Override
    public void saludar() {
        println("Hola, soy ${this.nombre}");
    }

    @Override
    public string obtenerSaludo() {
        return "Hola desde ${this.nombre}";
    }

    @Override
    public i64 getId() {
        return this.id;
    }
}
```

Caracteristicas de las interfaces:
- Solo declaran firmas (no implementaciones).
- Los metodos abstractos terminan en `;` (sin cuerpo).
- Una clase puede implementar multiples interfaces.
- La clase base (herencia simple) debe ser la primera despues de `:`.

### Polimorfismo via interfaz

```java
Saludable[] saludadores = {new Persona("Ana", 30, 1), new Persona("Luis", 25, 2)};

for (Saludable s : saludadores) {
    s.saludar(); // dispatch correcto aunque el tipo estatico sea Saludable
}
```

---

## Propiedades (get/set)

```java
class Caja {
    private i32 _valor = 0;

    // Getter con expression body
    public get valor => this._valor;

    // Setter con validation
    public set valor(i32 v) {
        if (v < 0) panic("valor negativo no permitido");
        this._valor = v;
    }

    // Property de solo lectura (no se define setter)
    public get doble => this._valor * 2;
}

Caja c = new Caja();
c.valor = 10; // llama al setter
println("${c.valor}"); // llama al getter: 10
println("${c.doble}"); // 20
// c.doble = 5; // ERROR: propiedad de solo lectura
```

---

## Campos y metodos estaticos

```java
class Contador {
    public static i64 total = 0; // campo estatico compartido por todas las instancias

    public Contador() {
        Contador.total += 1;
    }

    public static void resetear() {
        Contador.total = 0;
    }

    public static i64 obtenerTotal() {
        return Contador.total;
    }
}

new Contador(); new Contador(); new Contador();
println("Total = ${Contador.obtenerTotal()}"); // Total = 3
Contador.resetear();
println("Total = ${Contador.obtenerTotal()}"); // Total = 0
```

Los campos estaticos se almacenan en `ClassInfo::static_data` y se acceden con los
opcodes `getstatic` (0x60) y `setstatic` (0x61).

---

## Expression-bodied members

La sintaxis `=>` introduce un cuerpo de expresion. Solo funciona con `{ return expr; }`:

```java
class Matematica {
    public i32 cuadrado(i32 x) => x * x; // equivale a { return x * x; }
    public i32 cubo(i32 x) => x * x * x;
    public f64 hipotenusa(f64 a, f64 b) => sqrt(a*a + b*b);

    // Constructor delegado:
    public Matematica() => this(0);
    public Matematica(i32 config) { ... }
}
```

La forma `=> { ... }` (bloque) es rechazada por el parser.

---

## Anotacion @Inline

Expande el cuerpo del metodo en el call site sin emitir CALLVIRT:

```java
class Vector2 {
    public f64 x, y;

    @Inline
    public f64 magnitud() => sqrt(x*x + y*y);

    @Inline
    public Vector2 normalizado() => new Vector2(x / magnitud(), y / magnitud());
}

Vector2 v = new Vector2(3.0, 4.0);
f64 m = v.magnitud(); // expandido inline: sqrt(3*3 + 4*4) = 5.0
```

Solo se aplica a metodos con cuerpo `{ return expr; }`. Si el receptor es una interfaz, se
emite CALLVIRT como fallback.

---

## Destructor y RAII

El destructor `~Class()` se llama automaticamente al salir del scope donde fue declarada
la variable (RAII - Resource Acquisition Is Initialization):

```java
class Recurso {
    private string nombre;
    private u8* buffer;

    public Recurso(string nombre, i32 size) {
        this.nombre = nombre;
        this.buffer = malloc(size);
        println("Recurso ${nombre} creado");
    }

    public ~Recurso() {
        free(buffer); // libera memoria host
        println("Recurso ${nombre} liberado");
    }
}

void usarRecursos() {
    Recurso r1 = new Recurso("A", 1024); // r1 creado
    Recurso r2 = new Recurso("B", 2048); // r2 creado
    // ... usar r1 y r2 ...
} // r2 destruido, luego r1 destruido (orden LIFO)

usarRecursos();
// Salida:
// Recurso A creado
// Recurso B creado
// Recurso B liberado
// Recurso A liberado
```

**Regla de escape:** las instancias con destructor no pueden asignarse a campos de objetos
ni a arrays. Solo pueden vivir en variables locales (RAII). Para objetos de larga vida,
usar clases sin destructor + `dispose()` manual, o usar el GC.

---

## Synchronized y monitors

```java
class Contador {
    private i64 valor = 0;

    public void incrementar() {
        synchronized (this) { // adquiere el monitor de este objeto
            this.valor += 1;
        } // libera el monitor al salir (try/finally implicito)
    }

    public i64 obtener() {
        synchronized (this) {
            return this.valor;
        }
    }
}

// Builtins de monitor (dentro de synchronized):
synchronized (obj) {
    wait(obj); // libera el monitor y suspende el proceso
    notify(obj); // despierta un proceso en espera
    notifyAll(obj); // despierta todos los procesos en espera
}
```

`synchronized` emite `monenter` al entrar y `monexit` al salir. El monitor es reentrante
(el mismo proceso puede adquirirlo multiples veces).

---

## Anotacion @Sync

Equivale a envolver el cuerpo completo del metodo con `synchronized(this)`:

```java
class CuentaBancaria {
    private f64 saldo = 0.0;

    @Sync
    public void depositar(f64 monto) {
        this.saldo += monto;
    }

    @Sync
    public f64 obtenerSaldo() {
        return this.saldo;
    }
}
```

---

## ObjectHeader ABI (24 bytes)

Cada instancia de clase en el GC heap empieza con un `ObjectHeader`:

```c
Offset 0 - 7 : ClassInfo* (8 bytes, puntero a la clase)
Offset 8 - 11 : flags (4 bytes, OBJ_FLAG_*)
Offset 12 - 15 : hash_code (4 bytes, hash del objeto)
Offset 16 - 19 : owner_pid (4 bytes, PID del propietario del monitor)
Offset 20 - 21 : lock_depth (2 bytes, profundidad reentrante del monitor)
Offset 22 - 23 : _mon_pad (2 bytes, alineacion)
Offset 24+ : campos del usuario
```

El campo `class_ptr` permite que el runtime localice la vtable, los campos y los metodos
en O(1) desde cualquier objeto.

---

## Carga dinamica de clases

```java
// Cargar un modulo .velb y sus clases en runtime:
i64 resultado = loadmodule("plugins/extra.velb");

// Ahora las clases del modulo estan en el ClassRegistry:
Class cls = Class.forName("PluginServicio");
Object svc = cls.newInstance(); // ver ReflexionAOP.md
```

Las clases cargadas dinamicamente se registran en el mismo `ClassRegistry` global que las
clases del modulo principal. Sus `__module_init` se ejecutan al cargar.

---

Ver tambien: [[TiposDatos]], [[Async]], [[ReflexionAOP]], [[SetInstruccionesVM/OOP/CALLVIRT y CALLSUPER]]
