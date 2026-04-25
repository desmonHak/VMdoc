# Macros nativas

Las **macros nativas** son macros cuya implementacion esta escrita en una libreria
nativa externa (C, C++ o cualquier lenguaje que exporte funciones C) y se importan
al sistema de preprocesamiento de Vesta mediante la directiva `%native`.

**Analogia:** en lugar de escribir la implementacion de la macro en Vesta, le dices
al preprocesador "la implementacion de esta macro esta en esta libreria de funciones
ya compilada, usala directamente".

Esto permite reutilizar codigo nativo existente sin tener que reescribirlo en Vesta.

---

## Sintaxis de importacion

```c
// Importar la macro nativa exec_command del modulo os
%native("os") exec_command

// A partir de aqui exec_command esta disponible como una macro
exec_command("ls")        // ejecutar el comando ls
exec_command("pwd")       // ejecutar pwd
```

La directiva `%native("modulo")` le indica al preprocesador que busque la
implementacion en el modulo nativo especificado. El modulo puede ser:
- Una libreria dinamica (.dll en Windows, .so en Linux)
- Un modulo del runtime de VestaVM
- Una libreria estandar nativa

---

## Diferencia con CALLN / FFI

Las macros nativas y el sistema FFI (`CALLN`, `LOADLIB`, `GETPROC`) parecen similares
pero trabajan en niveles distintos:

| Aspecto             | Macro nativa (`%native`) | FFI (`CALLN`)                          |
| :------------------ | :----------------------- | :------------------------------------- |
| Nivel               | Preprocesamiento         | Tiempo de ejecucion                    |
| Cuando se resuelve  | Antes de compilar        | En la primera llamada                  |
| Puede generar codigo| Si (como cualquier macro)| No (solo invoca la funcion)            |
| Uso tipico          | Herramientas del build   | Funciones de librerias del sistema     |

---

## Casos de uso tipicos

```c
// Macro nativa para ejecutar comandos del sistema operativo
%native("os") exec_command

// Macro nativa para leer variables de entorno
%native("env") get_env_var

// Macro nativa para obtener la fecha actual en compilacion
%native("time") build_timestamp

// Uso combinado
exec_command("make clean")        // limpiar la build
String ts = build_timestamp();    // obtener timestamp de compilacion
```

---

## Implementacion en C

La implementacion de una macro nativa es una funcion C exportada con la convencion
estandar de plugins VestaVM:

```c
// Archivo: os_macros.c
// Compilar como: gcc -shared -fPIC -o libos_macros.so os_macros.c

#include <stdlib.h>

// Funcion exportada para exec_command
// El preprocesador la llama con los argumentos de la macro
void exec_command(const char *cmd) {
    system(cmd);
}
```

---

Ver tambien:
- [[Macros constantes.md]] - macros de valor fijo
- [[Macros parametricas.md]] - macros con argumentos
- [[../SetInstruccionesVM/NativeCall (CallN).md]] - FFI en tiempo de ejecucion
- [[../SetInstruccionesVM/NativePluginAPI.md]] - sistema de plugins nativos de VestaVM
