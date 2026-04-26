# Macros parametricas (macros funcion)

Las **macros funcion** son macros que aceptan argumentos. A diferencia de las macros
constantes (que solo sustituyen un nombre por un valor), las macros funcion pueden
transformar codigo en funcion de los argumentos que reciben.

El preprocesador de VestaVM (`vpp`) implementa macros funcion al estilo C/C++.

---

## Definicion basica

```c
// Macro con un argumento
#define CUADRADO(x)  ((x) * (x))

// Macro con dos argumentos
#define MAX(a, b)    ((a) > (b) ? (a) : (b))
#define MIN(a, b)    ((a) < (b) ? (a) : (b))

// Uso
uint64_t r1 = CUADRADO(5);     // ((5) * (5))   = 25
uint64_t r2 = CUADRADO(2 + 3); // ((2+3)*(2+3)) = 25  (parentesis importantes)
uint64_t r3 = MAX(r1, r2);     // r1 > r2 ? r1 : r2
```

Los parentesis alrededor de cada parametro en el cuerpo previenen errores de
precedencia cuando se pasa una expresion como argumento.

---

## Macros variadic

El preprocesador soporta macros con numero variable de argumentos usando `...` y
el nombre especial `__VA_ARGS__`:

```c
#define LOG(fmt, ...) log_write(fmt, __VA_ARGS__)
#define PRINT(...)    vio_println(__VA_ARGS__)

// Uso
LOG("valor=%d error=%s", codigo, mensaje);
PRINT("hola mundo");
```

---

## Operador `#` (stringify)

El operador `#` delante de un parametro convierte el argumento en una cadena
de texto literal:

```c
#define STRINGIFY(x) #x

STRINGIFY(hola)      // -> "hola"
STRINGIFY(42)        // -> "42"
STRINGIFY(a + b)     // -> "a + b"

// Util para mensajes de error con el nombre del simbolo
#define ASSERT(cond) \
    if (!(cond)) { panic("ASSERT fallo: " #cond); }

ASSERT(x > 0);  // -> if (!(x > 0)) { panic("ASSERT fallo: x > 0"); }
```

---

## Operador `##` (pegado de tokens)

El operador `##` une dos tokens adyacentes en uno solo, sin espacios:

```c
#define CONCAT(a, b)   a##b
#define REG(n)         r##n

CONCAT(my, func)   // -> myfunc
REG(3)             // -> r3
REG(14)            // -> r14

// Generar nombres de funciones a partir de un prefijo
#define HANDLER(name)  void handle_##name(Event *e)

HANDLER(click)    // -> void handle_click(Event *e)
HANDLER(keydown)  // -> void handle_keydown(Event *e)
```

---

## Macros multilinea con `#macro` / `#endmacro`

Para macros con cuerpo de varias lineas, el preprocesador soporta la directiva
`#macro` / `#endmacro`, que es mas legible que el uso de `\` de continuacion de linea:

```c
#macro SWAP(T, a, b)
    T __tmp = a;
    a = b;
    b = __tmp;
#endmacro

// Uso
int x = 5, y = 10;
SWAP(int, x, y);    // ahora x=10, y=5
```

El cuerpo entre `#macro` y `#endmacro` puede contener cualquier numero de lineas.
Los parametros se sustituyen igual que en las macros `#define`.

---

## Macros recursivas e higiene

El preprocesador de vpp detecta ciclos de expansion y los detiene para evitar
recursion infinita. Una macro no se expande dentro de su propio cuerpo.

```c
#define X(a)  X(a+1)   // NO se expande recursivamente: vpp rompe el ciclo
```

---

## Eliminacion con `#undef`

```c
#define LIMITE 100

// ... usar LIMITE ...

#undef LIMITE          // eliminar la definicion

// A partir de aqui LIMITE no esta definido
#ifndef LIMITE
    // este bloque se compila
#endif
```

---

## Macros predefinidas de plataforma

El preprocesador define automaticamente macros segun el sistema operativo y
la arquitectura detectados al compilar `vm`:

| Macro             | Se define cuando...               |
| :---------------- | :-------------------------------- |
| `__WINDOWS__`     | Compilando en Windows             |
| `__LINUX__`       | Compilando en Linux               |
| `__MACOS__`       | Compilando en macOS               |
| `__X86_64__`      | Arquitectura x86-64               |
| `__ARM64__`       | Arquitectura AArch64 / ARM64      |
| `__VESTA__`       | Siempre definida (es vpp)         |

Estas macros son utiles para codigo condicional multiplataforma:

```c
#ifdef __WINDOWS__
    #define PATH_SEP  "\\"
#else
    #define PATH_SEP  "/"
#endif
```

---

Ver tambien:
- [[Macros constantes.md]] - macros `#define` sin parametros
- [[Macros builder.md]] - `#set`, `#foreach`, `#repeat`, `#array`, `#exec`, `#assert`
- [[Macros nativas.md]] - `#import` de librerias de macros vpp
