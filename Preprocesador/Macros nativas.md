# Importacion de librerias de macros: `#import`

El preprocesador vpp distingue entre dos formas de incluir ficheros:

| Directiva             | Semantica                                                     |
| :-------------------- | :------------------------------------------------------------ |
| `#include "path"`     | Incluye un fichero relativo al fichero actual (siempre)       |
| `#include <path>`     | Incluye un fichero desde las `include_paths` configuradas     |
| `#import <modulo>`    | Incluye una libreria de macros vpp (auto-once, desde `import_paths`) |

La diferencia clave de `#import` frente a `#include`:
- **Auto-once**: el mismo modulo nunca se incluye dos veces aunque se llame `#import`
  desde distintos ficheros (equivale a un `#pragma once` implicito).
- **Rutas de busqueda propias**: usa `import_paths`, no `include_paths`.
- **Extension implicita**: busca el fichero con extension `.vph`, `.vel` y sin extension.

---

## Uso basico de `#import`

```c
// Importar una libreria de macros vpp desde el directorio de macros del proyecto
#import <math_macros>        // busca math_macros.vph, math_macros.vel, math_macros

// El modulo puede definir macros que quedan disponibles en el fichero actual
CLAMP(valor, 0, 100)         // macro definida en math_macros
```

---

## Directorio `include_lib`

Al construir `vm`, el compilador copia el directorio `preprocessor/include_lib/`
junto al ejecutable. Las macros de stdlib vpp se encuentran ahi. Por ejemplo:

```c
// Importar macros de utilidad para ensamblador Vesta
#import <vel/registers>    // define RX(n) -> rN, REG_SCRATCH -> r14, etc.
#import <vel/abi>          // define CALL_SETUP(n), RET_REG, etc.
```

---

## `#pragma once`

Cuando se usa `#include` en lugar de `#import`, se puede añadir `#pragma once`
al inicio del fichero incluido para evitar inclusiones multiples:

```c
// archivo: mi_cabecera.vph
#pragma once

#define MI_CONSTANTE 42
#define MI_MACRO(x)  ((x) + MI_CONSTANTE)
```

Si `mi_cabecera.vph` se incluye desde varios ficheros, el preprocesador solo
procesa su contenido la primera vez.

---

## Diferencia con FFI / CALLN

`#import` es una directiva del **preprocesador** (tiempo de compilacion) que
importa definiciones de macros. No tiene ninguna relacion con el sistema FFI
de VestaVM (`@Lib`, `calln`) que importa funciones nativas en **tiempo de ejecucion**.

| Aspecto             | `#import` (preprocesador)         | `@Lib` + `calln` (runtime)            |
| :------------------ | :-------------------------------- | :------------------------------------ |
| Fase                | Preprocesamiento                  | Carga y ejecucion                     |
| Importa             | Definiciones de macros (texto)    | Funciones nativas compiladas (.dll/.so)|
| Resultado           | Texto expandido                   | Llamada a funcion de libreria         |
| Ejemplo             | `#import <vel/registers>`         | `@Lib("stdlib/native/io/vesta_io")`   |

---

## Incluir ficheros generados: `#exec` + `#include`

Un patron avanzado combina `#exec` para generar un fichero de macros en tiempo
de compilacion y luego `#include` para incluirlo:

```c
// Generar un fichero con informacion de version a partir de git
#exec _GIT  git describe --tags --abbrev=8
#define BUILD_VERSION _GIT

// O generar e incluir un fichero completo
// (requiere que el script escriba el fichero en disco)
```

---

Ver tambien:
- [[Macros constantes.md]] - `#define`, macros predefinidas de tiempo y plataforma
- [[Macros parametricas.md]] - macros funcion y operadores `##` / `#`
- [[Macros builder.md]] - `#set`, `#foreach`, `#repeat`, `#array`, `#exec`, `#assert`
- [[../SetInstruccionesVM/NativeCall (CallN).md]] - FFI en tiempo de ejecucion
