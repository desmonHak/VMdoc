# Macros anotaciones

Las **macros anotacion** son directivas del preprocesador que se escriben con el
prefijo `@` y que asocian metadatos o comportamiento especial a clases, metodos,
campos o al archivo completo.

A diferencia de las macros constantes o parametricas, las macros anotacion no
sustituyen texto: **configuran al compilador** para que genere codigo adicional
o aplique restricciones especiales.

**Analogia:** son como las etiquetas de clasificacion en los archivos de una oficina.
No cambian el contenido del documento; dicen como debe tratarse: "confidencial",
"urgente", "archivado".

---

## Anotaciones de clase

```c
// @Override: indica que el metodo sobreescribe al de la clase padre
@Override
public String toString() { return "..."; }

// @Final: la clase no puede ser extendida (heredada)
@Final
class Inmutable { }

// @Abstract: la clase no puede instanciarse directamente
@Abstract
class Figura {
    public float area();  // metodo sin cuerpo: las subclases lo implementan
}

// @NotExtendObject: la clase no hereda de Object
@NotExtendObject
class TipoPuro {
    uint64_t dato;
}
```

---

## Anotaciones del ensamblador

Estas son procesadas directamente por el ensamblador (no por el preprocesador
de alto nivel) y configuran el modulo, las exportaciones y los genericos:

```c
// Declarar el modulo del archivo
@Module(com.empresa.colecciones)

// Exportar simbolos publicos
@Export(Lista)
@Export(lista_nueva)
@Export(lista_agregar)

// Registrar parametros de tipo para monomorphization
@Generic(T)         // un parametro de tipo
@Generic(K, V)      // dos parametros (diccionario clave-valor)
```

Ver [[../SintaxisCore/Anotaciones modulo y genericos.md]] para la referencia completa.

---

## Anotaciones de configuracion del ejecutable

```c
// Formato del binario de salida
@Format("velb")

// Espacio de direcciones
@SpaceAddress {
    code_start = 0x1000;
    stack_size = 0x10000;
}

// Secciones del binario
@Section {
    .code  { name = ".text";   perms = READ | EXEC; }
    .data  { name = ".data";   perms = READ | WRITE; }
}

// Punto de entrada
@EntryPoint("modulo.main")
```

Ver [[Macros configuracion de ejecutable.md]] para la referencia completa.

---

## Anotaciones de importacion (FFI)

```c
// Importar una libreria nativa
@Lib("stdlib/native/io/vesta_io")

// Importar un simbolo especifico de una libreria
@Import("vio_println")
@Import("vio_print_int")
```

Estas anotaciones son procesadas por el loader al cargar el `.velb`: busca la
libreria en el filesystem y resuelve los simbolos importados.

---

## Tabla resumen de anotaciones

| Anotacion              | Nivel      | Efecto                                                      |
| :--------------------- | :--------- | :---------------------------------------------------------- |
| `@Override`            | Metodo     | Verifica sobreescritura; error si el padre no tiene el metodo |
| `@Final`               | Clase      | Impide herencia                                             |
| `@Abstract`            | Clase      | Impide instanciacion directa                                |
| `@NotExtendObject`     | Clase      | No hereda de Object                                         |
| `@Deprecated`          | Metodo     | Emite advertencia al usarse                                 |
| `@Synchronized`        | Metodo     | Genera monenter/monexit automaticamente                     |
| `@Module(nombre)`      | Archivo    | Declara el modulo activo                                    |
| `@Export(Simbolo)`     | Archivo    | Marca simbolo como publico                                  |
| `@Generic(T)`          | Archivo    | Registra parametros de tipo                                 |
| `@Format("velb")`      | Archivo    | Formato del binario de salida                               |
| `@SpaceAddress { }`    | Archivo    | Espacio de direcciones virtuales                            |
| `@Section { }`         | Archivo    | Definicion de secciones                                     |
| `@EntryPoint("sym")`   | Archivo    | Punto de entrada del programa                               |
| `@Lib("ruta")`         | Archivo    | Importar libreria nativa (FFI)                              |
| `@Import("funcion")`   | Archivo    | Importar funcion especifica                                 |

---

Ver tambien:
- [[../SintaxisCore/Decoradores.md]] - decoradores de alto nivel
- [[../SintaxisCore/Anotaciones modulo y genericos.md]] - @Module, @Export, @Generic en detalle
- [[Macros configuracion de ejecutable.md]] - @Format, @Section, @SpaceAddress
- [[Macros nativas.md]] - macros %native para funciones externas
