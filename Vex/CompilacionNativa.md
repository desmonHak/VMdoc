# Compilacion nativa en Vex

Vex puede compilar a binarios nativos del sistema operativo: ejecutables
standalone, objetos, librerias estaticas y librerias compartidas, **sin
necesidad de la maquina virtual ni de un compilador de C externo** (gcc/clang/
ld no hacen falta).  El enlazador y el archivador son propios del lenguaje.

Esta guia cubre:

1. [Compilar a un ejecutable nativo](#1-compilar-a-un-ejecutable-nativo)
2. [El recolector de basura (GC) en compilacion nativa](#2-el-recolector-de-basura-gc-en-compilacion-nativa)
3. [Crear librerias: objeto, estatica (`.a`) y compartida (`.so`/`.dll`)](#3-crear-librerias)
4. [Usar librerias desde el lenguaje](#4-usar-librerias-desde-el-lenguaje)
5. [Enlazado estatico: binarios sin dependencias externas](#5-enlazado-estatico-binarios-sin-dependencias-externas)

El formato de salida se elige con `--format`:

- `--format pe` produce binarios de Windows (`.exe`, `.obj`, `.dll`).
- `--format elf` produce binarios de Linux (ejecutable, `.o`, `.so`).

Si no se indica, se usa el formato del sistema donde se compila (PE en Windows,
ELF en Linux).

---

## 1. Compilar a un ejecutable nativo

Un programa con `main` se compila a un ejecutable que el sistema operativo
arranca directamente:

```bash
vm --vex programa.vex -m aot -o programa
```

El valor que devuelve `main` es el codigo de salida del proceso:

```vex
i64 main() {
    return 42;     // el proceso termina con codigo 42
}
```

```bash
vm --vex programa.vex -m aot -o programa.exe   # Windows (PE)
./programa.exe                                  # codigo de salida = 42
```

El ejecutable es autonomo: no necesita la VM ni librerias del lenguaje.  Solo
depende de las librerias del sistema operativo (que siempre estan presentes).

---

## 2. El recolector de basura (GC) en compilacion nativa

### Como gestiona la memoria un binario nativo

Por defecto, las clases en compilacion nativa siguen un modelo de **propiedad
con liberacion determinista** (RAII): un objeto se libera automaticamente al
salir del ambito donde se creo, sin recolector y sin coste de fondo.  Esto es
suficiente para la mayoria del codigo y es lo mas predecible.

Cuando el grafo de objetos tiene **ciclos** o vidas que no encajan en un ambito
(estructuras compartidas, grafos, cachés), conviene una gestion automatica de
memoria.  Para eso Vex ofrece el tipo `gc<T>`.

### Como funciona el GC nativo

El recolector que usa un binario nativo es el **mismo motor** que emplea el
interprete del lenguaje, empaquetado dentro del propio binario.  Es un
recolector *mark-and-sweep* generacional:

- **Asignacion**: `new` sobre un tipo `gc<T>` reserva el objeto en un monticulo
  gestionado.  No hay que liberarlo a mano.
- **Raices precisas**: el compilador genera, para cada punto donde el GC puede
  actuar, un mapa de que registros y posiciones de la pila contienen
  referencias vivas.  El recolector lee esos mapas y recorre la pila nativa de
  forma exacta, sin falsos positivos.
- **Recoleccion**: cuando un objeto deja de ser alcanzable desde las raices, el
  recolector lo libera.  Los ciclos se recogen correctamente (a diferencia del
  conteo de referencias).

### Como usar el GC

Se declara la variable con el tipo `gc<Clase>` y se construye con `new`:

```vex
class Nodo {
    i64 valor;
    Nodo(i64 v) { this.valor = v; }
}

i64 main() {
    gc<Nodo> n = new Nodo(42);
    return n.valor;          // 42; el objeto lo recoge el GC, no hay que liberarlo
}
```

No hay ninguna llamada de liberacion: el recolector se encarga.  Puedes crear
millones de objetos en un bucle sin fugas; la memoria se mantiene acotada.

### Que hay que hacer para activarlo

Nada especial mas alla de usar `gc<T>`.  Al compilar a un ejecutable, si el
programa usa `gc<T>`, **la libreria del recolector se incluye y se enlaza de
forma automatica**:

```bash
vm --vex programa.vex -m aot -o programa.exe
```

El binario resultante lleva el recolector dentro (no es una dependencia
externa): arranca y gestiona la memoria por si mismo.

> La libreria del recolector (`libvesta_gc.a`) se busca junto al ejecutable del
> lenguaje.  Si la distribuyes, colocala al lado del binario `vm`.

---

## 3. Crear librerias

### 3.1. Objeto relocatable (`.o` / `.obj`)

Un objeto contiene el codigo compilado pero todavia no es ejecutable; sirve para
enlazarlo despues:

```bash
# Linux (ELF .o)
vm --vex modulo.vex -m aot --emit obj --format elf -o modulo.o

# Windows (COFF .obj)
vm --vex modulo.vex -m aot --emit obj --format pe -o modulo.obj
```

El simbolo `main` (si existe) y las funciones quedan disponibles para el
enlazado.

### 3.2. Libreria estatica (`.a`)

Una libreria estatica agrupa varios objetos en un solo archivo.  Se crea con el
archivador integrado:

```bash
vm --ar libmate.a suma.o resta.o multiplica.o
```

El `.a` lleva un indice de simbolos, de modo que al enlazar **solo se incluyen
los objetos que se usan** (los demas se descartan).  El archivo es estandar:
tambien lo entienden `ar`, `nm` y los enlazadores del sistema.

Tambien puedes crear cada objeto desde Vex y archivarlos juntos:

```bash
vm --vex suma.vex      -m aot --emit obj --format pe -o suma.obj
vm --vex resta.vex     -m aot --emit obj --format pe -o resta.obj
vm --ar libmate.a suma.obj resta.obj
```

### 3.3. Libreria compartida (`.so` / `.dll`)

Una libreria compartida se carga en tiempo de ejecucion y exporta sus funciones:

```bash
# Linux (.so)
vm --vex libmate.vex -m aot --emit shared --format elf -o libmate.so

# Windows (.dll)
vm --vex libmate.vex -m aot --emit shared --format pe -o mate.dll
```

Todas las funciones del modulo quedan exportadas y se pueden resolver por nombre
(equivalente a `dlopen`/`dlsym` en Linux o `LoadLibrary`/`GetProcAddress` en
Windows).

---

## 4. Usar librerias desde el lenguaje

### 4.1. Declarar las funciones externas

Para llamar a una funcion que vive en otra libreria, se declara con `extern`,
indicando de que libreria proviene:

```vex
extern "mate" { fn suma(i64 a, i64 b) -> i64; }   // de una .a / .o
i64 main() {
    return suma(40, 2);    // 42
}
```

Para una libreria compartida se usa el nombre del archivo:

```vex
extern "mate.dll" { fn suma(i64 a, i64 b) -> i64; }   // de una .dll
```

### 4.2. Enlazar

El enlazador integrado junta el objeto del programa con las librerias.  Acepta
objetos (`.o`/`.obj`), librerias estaticas (`.a`) y librerias compartidas
(`.so`/`.dll`) mezcladas:

```bash
# 1. compilar el programa a objeto
vm --vex programa.vex -m aot --emit obj --format pe -o programa.obj

# 2a. enlazar con una libreria ESTATICA (se incluye en el binario)
vm --link programa.obj libmate.a -o programa.exe --format pe

# 2b. o enlazar con una libreria COMPARTIDA (dependencia de runtime)
vm --link programa.obj mate.dll  -o programa.exe --format pe
```

En ambos casos el resultado es un ejecutable.  La diferencia esta en como queda
la dependencia, que es el tema de la siguiente seccion.

> Tambien puedes pasar una `.dll` o `.so` propia (o de terceros) como entrada al
> enlazador: sus funciones exportadas se usan para resolver los `extern`.

---

## 5. Enlazado estatico: binarios sin dependencias externas

Hay dos maneras de que un programa use una libreria, con consecuencias
distintas sobre las dependencias del binario final:

| Tipo | Que ocurre al enlazar | Dependencia en runtime |
| :--- | :-------------------- | :--------------------- |
| Estatica (`.a` / `.o`) | El codigo usado se **copia dentro** del binario | Ninguna: el binario es autonomo |
| Compartida (`.so` / `.dll`) | Solo se anota una **referencia** a la libreria | La libreria debe acompanar al binario |

### Incluir los simbolos de forma estatica

Para que el binario **no dependa de librerias externas**, enlaza con la version
**estatica** (`.a` o los `.o` directamente).  El enlazador copia dentro del
ejecutable unicamente los simbolos que el programa usa:

```bash
# El programa usa suma() de libmate.a -> el codigo de suma() queda DENTRO de
# programa.exe.  No hace falta distribuir libmate.a junto al ejecutable.
vm --link programa.obj libmate.a -o programa.exe --format pe
./programa.exe        # corre solo, sin libmate.a al lado
```

Solo se incluyen los miembros del `.a` que aportan simbolos realmente
referenciados; lo que no se usa no engorda el binario.

Esto aplica igual a la libreria del recolector: cuando un programa usa `gc<T>`,
su libreria se enlaza de forma estatica y queda **dentro** del ejecutable.  El
binario gestiona su propia memoria sin depender de nada externo.

### Funciones del sistema

Las funciones del sistema operativo (reserva de memoria, E/S de bajo nivel,
etc.) se resuelven contra las librerias del propio sistema, que **siempre estan
presentes** en cualquier instalacion (no son dependencias que tengas que
distribuir).  El enlazador detecta automaticamente que libreria del sistema
exporta cada funcion leyendo su tabla de simbolos, sin listas fijas.

### Resumen practico

- ¿Quieres un binario que corra solo, sin acompanarlo de nada?  Enlaza con `.a`
  / `.o` (estatico).
- ¿Quieres compartir una libreria entre varios programas y actualizarla por
  separado?  Usa `.so` / `.dll` (compartida) y distribuyela junto a los
  programas que la usan.
- El recolector de basura, si lo usas, siempre queda incluido de forma estatica:
  el ejecutable es autonomo.

---

## Vease tambien

- [[FFI]] - llamar a librerias del sistema y de terceros (`extern`).
- [[SmartPointers]] - liberacion determinista (RAII) sin recolector.
- [[Modulos]] - organizar el codigo en modulos antes de compilarlo.
