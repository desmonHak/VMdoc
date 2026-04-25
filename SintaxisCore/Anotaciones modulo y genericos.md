# Anotaciones de Modulo y Genericos

VestaVM introduce tres anotaciones del ensamblador para el sistema de modulos
y la monomorphization de tipos genericos:

| Anotacion       | Efecto                                                              |
| :-------------- | :------------------------------------------------------------------ |
| `@Module(pkg)`  | Declara el modulo del archivo fuente; activa el modulo en el contexto. |
| `@Export(Sym)`  | Marca el simbolo como publico y accesible desde otros modulos.       |
| `@Generic(T)`   | Registra parametros de tipo para monomorphization en el emitter.    |

Estas anotaciones son **directivas del ensamblador**: no generan bytecode
por si solas.  El ensamblador las procesa en la primera pasada y las escribe
en los metadatos del archivo `.velb` resultante, donde el linker y el loader
las usan para resolver visibilidad y especializar tipos.

Implementacion: `src/emmit/annotations.cpp`

---

## `@Module(nombre.calificado)`

```c
@Module(com.empresa.colecciones)
```

### Efecto

- Crea una entrada `ModuleEntry` en `Context::modules` con el nombre
  calificado como clave, si aun no existe.
- Establece `Context::current_module` al nombre dado.
- Todos los `@Export` y `@Generic` que sigan aplican a este modulo.

### Reglas

- Solo debe aparecer **una vez por archivo fuente**.  Si aparece mas de una
  vez, la segunda declaracion cambia el modulo activo (comportamiento valido
  para archivos de cabecera reutilizados, pero desaconsejado).
- El nombre calificado usa puntos como separadores: `paquete.subpaquete.modulo`.
- No puede estar vacio: `@Module()` lanza un error del ensamblador.

### Estructura interna (Context)

```cpp
struct ModuleEntry {
    std::string qualified_name;           // "com.empresa.colecciones"
    std::vector<std::string> exports;     // simbolos exportados
    std::vector<std::string> imports;     // simbolos importados (uso futuro)
};

// En Context:
std::unordered_map<std::string, ModuleEntry> modules;
std::string current_module;  // modulo activo durante la emision
```

### Ejemplo

```c
@Module(colecciones.lista)
@Module(com.vesta.stdlib.io)
```

---

## `@Export(Simbolo)`

```c
@Export(Lista)
@Export(lista_nueva)
@Export(lista_agregar)
```

### Efecto

- Requiere que `@Module` haya sido declarado antes en el mismo archivo.
- Anade el simbolo a `ModuleEntry::exports` del modulo activo.
- Los simbolos duplicados se ignoran (no se registran dos veces).

### Visibilidad

| Estado del simbolo | Accesible desde         |
| :----------------- | :---------------------- |
| Exportado          | Cualquier modulo        |
| No exportado       | Solo el mismo modulo    |

El loader verifica la visibilidad al resolver referencias cruzadas entre
modulos.  Si un modulo intenta acceder a un simbolo privado de otro modulo,
el loader lanza un error de acceso.

### Reglas

- El simbolo debe ser un identificador valido (sin espacios).
- Puede aparecer tantas veces como simbolos se quieran exportar.
- Si no hay un `@Module` activo, el ensamblador lanza un error.

### Ejemplo

```c
@Module(colecciones.lista)
@Export(Lista)          // clase publica
@Export(lista_nueva)    // funcion publica
                        // lista_interna no se exporta: es privada al modulo
```

---

## `@Generic(T)` y `@Generic(K,V)`

```c
@Generic(T)          // un parametro de tipo
@Generic(K,V)        // multiples parametros separados por comas
```

### Efecto

- Registra los parametros de tipo en `Context::generic_instances` con el
  nombre del parametro como clave y `nullptr` como valor (marcador).
- El emitter usa esta informacion durante la monomorphization para generar
  especializaciones concretas.

### Monomorphization en el emitter

La monomorphization se produce en tiempo de emision (no en tiempo de
ejecucion), de forma similar a los templates de C++:

1. El codigo cliente referencia `@Class("Lista<int>")`.
2. El emitter busca `"Lista<int>"` en `generic_instances`.
3. Si no existe, **clona** el `ClassInfo` de `"Lista"`, sustituye `T` por
   `int` en todos los campos (tamanos, offsets, nombres de metodos) y
   registra la especializacion como `"Lista<int>"`.
4. Las llamadas posteriores a `@Class("Lista<int>")` reutilizan la misma
   especializacion sin duplicarla.

El resultado es codigo especializado por tipo sin overhead en tiempo de
ejecucion: no hay boxing, no hay despacho dinamico adicional.

### Parametros multiples

```c
@Generic(K,V)    // define los parametros K y V
```

Los parametros separados por coma se recortan de espacios y se registran
individualmente.  `@Generic(K, V)` y `@Generic(K,V)` son equivalentes.

### Reglas

- No puede estar vacio: `@Generic()` lanza un error del ensamblador.
- Los nombres de parametros siguen las convenciones habituales: una o pocas
  letras mayusculas (`T`, `E`, `K`, `V`, `R`).
- Puede aparecer varias veces en el mismo archivo si hay multiples clases
  o funciones genericas, pero los parametros se acumulan en el mismo mapa.

### Ejemplo

```c
@Module(colecciones.mapa)
@Export(Mapa)
@Generic(K,V)    // Mapa<K,V>: clave de tipo K, valor de tipo V

@Export(conjunto)
@Generic(T)      // conjunto<T>: elementos de tipo T
```

---

## Interaccion entre las tres anotaciones

El orden tipico en un archivo fuente es:

```c
// 1. Declarar el modulo (obligatorio antes de @Export)
@Module(com.vesta.ejemplo)

// 2. Exportar los simbolos publicos
@Export(MiClase)
@Export(mi_funcion)

// 3. Declarar los parametros de tipo (si la clase es generica)
@Generic(T)

// 4. Resto de anotaciones de configuracion
@Format("velb")
@SpaceAddress { ... }
@Section { ... }

// 5. Codigo fuente
MiClase:
    // ...
```

---

## Alcance y limitaciones actuales

- **Modulos y visibilidad**: la verificacion de visibilidad en el loader
  esta pendiente de implementacion completa.  Actualmente `@Export` escribe
  los metadatos pero el loader no rechaza accesos a simbolos privados.
- **Imports entre modulos**: `@Import` registra funciones nativas (FFI).
  El mecanismo de importacion de simbolos de otros modulos Vesta (equivalente
  a `import` en Java o `use` en Rust) esta planificado para una version futura.
- **Monomorphization**: el emitter registra los parametros pero la
  especializacion automatica al referenciar `@Class("Lista<int>")` esta
  en desarrollo.  El mapa `generic_instances` actua como marcador hasta
  que la fase de clonacion este implementada.

---

## Ejemplo completo

Vease [[../../../examples_codes_vm/ejemplo_modulo.vel]] para un ejemplo
que usa `@Module`, `@Export` y `@Generic` en un modulo de lista generica.
