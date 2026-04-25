# Macros de configuracion de ejecutable

Las **macros de configuracion de ejecutable** son directivas del preprocesador que
permiten controlar como se estructura el archivo binario final (`.velb`): el tamano
de las secciones, los permisos de memoria, los nombres de secciones, el punto de
entrada y otros metadatos del ejecutable.

**Analogia:** son como los formularios que rellenas al construir un edificio. Antes
de construir nada (compilar codigo), describes cuantas plantas tendra, que planos
seguira y que permisos tiene. La obra (compilacion) usa esa configuracion.

---

## Directiva `@SpaceAddress`

Define el espacio de direcciones virtuales del ejecutable: donde empieza el codigo,
donde empieza la pila, el tamano del heap, etc.

```c
// Configurar el espacio de direcciones del ejecutable
@SpaceAddress {
    code_start = 0x1000;      // el codigo comienza en la direccion virtual 0x1000
    stack_size = 0x10000;     // tamano de la pila: 64 KB
    heap_size  = 0x100000;    // tamano inicial del heap: 1 MB
}
```

---

## Directiva `@Section`

Define secciones del binario con nombres, permisos y tamanos:

```c
// Definir secciones del archivo .velb
@Section {
    .code {
        name    = ".text";    // nombre de la seccion de codigo
        perms   = READ | EXEC; // permisos: lectura y ejecucion
    }

    .data {
        name    = ".data";    // seccion de datos inicializados
        perms   = READ | WRITE;
    }

    .rodata {
        name    = ".rodata";  // seccion de solo lectura (constantes)
        perms   = READ;
    }
}
```

---

## Directiva `@Format`

Indica el formato del archivo de salida:

```c
@Format("velb")     // formato VestaLangBinary (por defecto)
@Format("vela")     // formato VEL Archive (libreria estatica)
```

---

## Directiva `@EntryPoint`

Especifica el simbolo que es el punto de entrada del programa (equivalente a `main`):

```c
@EntryPoint("mi_programa.inicio")

// El runtime comenzara la ejecucion en la etiqueta "inicio" del modulo "mi_programa"
inicio:
    // primer codigo que se ejecuta
```

---

## Directiva `@Version`

Embebe informacion de version en los metadatos del ejecutable:

```c
@Version {
    major = 1;
    minor = 0;
    patch = 0;
    label = "release";
}
```

---

## Ejemplo completo de configuracion

```c
// Cabecera tipica de un archivo .vel con configuracion completa
@Format("velb")
@Version { major = 1; minor = 0; patch = 0; }
@Module(com.empresa.miapp)

@SpaceAddress {
    code_start = 0x1000;
    stack_size = 0x20000;    // 128 KB de pila
    heap_size  = 0x400000;   // 4 MB de heap inicial
}

@Section {
    .code  { name = ".text";   perms = READ | EXEC; }
    .data  { name = ".data";   perms = READ | WRITE; }
    .const { name = ".rodata"; perms = READ; }
}

@EntryPoint("com.empresa.miapp.main")

@Export(main)
main:
    // codigo del programa
```

---

## Tabla de macros de configuracion

| Macro             | Proposito                                                    |
| :---------------- | :----------------------------------------------------------- |
| `@Format`         | Formato del binario de salida (velb, vela)                   |
| `@SpaceAddress`   | Espacio de direcciones virtuales (code_start, stack, heap)   |
| `@Section`        | Definicion de secciones con nombre y permisos                |
| `@EntryPoint`     | Simbolo de entrada del programa                              |
| `@Version`        | Metadatos de version embebidos en el binario                 |
| `@Module`         | Nombre calificado del modulo (ver Anotaciones modulo)        |

---

Ver tambien:
- [[Macros constantes.md]] - macros de valor fijo
- [[../SintaxisCore/Anotaciones modulo y genericos.md]] - @Module, @Export, @Generic
- [[../VELB (VEL Binary Format)/Header_VELB.md]] - estructura del header del binario .velb
- [[../VELB (VEL Binary Format)/TablaDeSecciones.md]] - tabla de secciones del .velb
