# Macros constantes

Las **macros constantes** son identificadores que representan un valor fijo calculado
en tiempo de preprocesamiento. El preprocesador sustituye el nombre de la macro por
su valor antes de que el compilador vea el codigo.

**Analogia:** son como notas adhesivas. Escribes `MAX_USUARIOS` en tu codigo y el
preprocesador lo sustituye por `1000` antes de compilar. El compilador nunca llega
a ver `MAX_USUARIOS`: solo ve `1000`.

---

## Macros predefinidas del sistema

El preprocesador de Vesta incluye un conjunto de macros predefinidas con informacion
del momento de compilacion:

```c
__LINE__          // numero de linea actual en el archivo fuente
__TIME__          // hora local de compilacion: "HH:MM:SS"
__DATE__          // fecha local de compilacion: "MMM DD YYYY"
__DATE_NUM__      // fecha como numero: YYYYMMDD (ej. 20260425)
__TIME_NUM__      // hora como numero: HHMMSS (ej. 143022)
__UTC_DATE__      // fecha UTC de compilacion
__UTC_TIME__      // hora UTC de compilacion
__UTC_DATE_NUM__  // fecha UTC como numero
__UTC_TIME_NUM__  // hora UTC como numero
__POSIX_TIME__    // segundos POSIX (Unix timestamp) de compilacion
```

Estas macros son utiles para insertar informacion de version o depuracion
directamente en el binario compilado.

---

## Macros definidas por el usuario

Se definen con la directiva `#define`:

```c
// Constante numerica
#define MAX_USUARIOS 1000
#define PI 3.14159

// Constante de cadena
#define VERSION "1.0.0"

// Uso en codigo
uint64_t capacidad = MAX_USUARIOS;  // el preprocesador lo convierte en 1000
float area = PI * radio * radio;
```

---

## Macros calculadas en preprocesamiento

Algunas macros pueden calcularse a partir de otras en tiempo de preprocesamiento:

```c
#define ANCHO  800
#define ALTO   600
#define PIXELS (ANCHO * ALTO)  // calculado en preprocesamiento: 480000

// El compilador solo ve literales, no operaciones
uint64_t pantalla = PIXELS;  // equivalente a: uint64_t pantalla = 480000;
```

---

## Condicionamiento con macros

Las macros constantes permiten compilar codigo de forma condicional:

```c
#define DEBUG 1

#if DEBUG
    // Este codigo solo se incluye cuando DEBUG es 1
    print("Modo debug activado");
#endif
```

---

Ver tambien:
- [[Macros parametricas.md]] - macros que aceptan argumentos
- [[Macros nativas.md]] - macros que llaman a funciones nativas
- [[Macros configuracion de ejecutable.md]] - macros para configurar el ejecutable
