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

## 6. Concurrencia y asincronia en compilacion nativa

El modelo de concurrencia de Vex es **cooperativo** (asincronia con tareas verdes,
igual que en el interprete/JIT), **no** hilos 1:1 del sistema operativo.  `spawn`
encola una tarea; el bloqueo (`await`) bombea la cola hasta que el resultado esta
listo.  Todo el scheduler es codigo Vex que el compilador incluye en el binario
(`stdlib/vex/vex_async.vex`), por lo que el ejecutable nativo es autonomo (no
depende de la VM ni de ninguna DLL).

### Lo que YA funciona en nativo (validado)

- **`spawn { ... }`** — encola una tarea (sin/con capturas y argumentos).
- **`Future<T>` + `await`** — creacion del future, `fulfill`, y `await` que bombea
  la cola hasta resolverlo.
- **`@Async`** — funciones asincronas con argumentos (sin limite de aridad, hasta
  los 12 del lenguaje), devuelven un future implicito.
- **Mailbox** — `msgsend(pid, valor)` / `msgrecv()` (paso de mensajes por valor).
- **`pid()`** — identificador de la tarea en ejecucion.
- **`synchronized (obj) { ... }`** — seccion critica (monitor con `lock cmpxchg`),
  con limpieza en el `return`.

Estas piezas estan **completas** y producen el mismo resultado que el interprete.
Casos como `spawn` + `future` + `await`, `@Async` con args, y `synchronized` se
compilan a `.exe`/`.elf` autonomos.

### Lo que NO funciona todavia (limitaciones conocidas)

- **Tareas que se bloquean MUTUAMENTE** (p.ej. un *ping-pong*: la tarea A espera
  un mensaje de B y B espera uno de A).  El modelo actual corre cada tarea
  *hasta el final* (run-to-completion); no sabe **suspender** una tarea a mitad
  para reanudarla luego.  Estos programas **compilan**, pero dan un resultado
  **incorrecto** (la tarea bloqueada lee 0 en vez de esperar).
- **`wait` / `notify` / `notifyAll`** dentro de un `monitor` — necesitan el mismo
  mecanismo de suspension que el punto anterior.

### Lo que queda PENDIENTE (trabajo futuro)

- **Fibras (context-switch cooperativo)** — es la pieza que falta para suspender y
  reanudar tareas: guardar/restaurar `{rsp, rip, registros callee-saved}` y una
  pila por fibra.  Desbloquea el *ping-pong* y `wait`/`notify`.  Se implementara
  en Vex (con inline-asm `@Naked` + reserva de pila via `extern`), sin la VM.
- **Pool de hilos reales (paralelismo)** — N workers del SO ejecutando las tareas
  verdes en paralelo (modelo M:N).  Encima de las fibras.
- **`rspawn` (procesos remotos / distribuido)** — requiere red (TCP); queda
  **fuera del alcance** de un binario standalone.

> En resumen: la **asincronia de una sola via** (lanzar tareas, esperar
> resultados, mensajes, secciones criticas) funciona en nativo hoy.  La
> **coordinacion con bloqueo mutuo** (ping-pong, wait/notify) y el **paralelismo
> real** estan pendientes de las fibras y el pool de hilos.

---

## 7. Paridad ELF / Linux

El mismo codigo Vex compila a PE (Windows) y a ELF (Linux) con `--format pe`
o `--format elf`.  El estado de paridad es:

### Lo que es identico en ambos formatos

- **Codigo Vex generado** (aritmetica, structs, clases, strings, colecciones,
  smart pointers, `strconv`, control de flujo, FFI dinamico): **libre de libc**.
  Los strings usan SSO (datos inline) o el slab propio `vex_mem` (`__vex_malloc`),
  nunca `malloc` de libc.  `panic`/`print` salen por syscalls.  Validado en ELF:
  `strconv` y `ArrayList` devuelven el resultado correcto sin enlazar libc en la
  parte generada.
- **El stub `_start`**: en ELF termina el proceso con
  `syscall exit_group(231)` (NO `exit(60)`), pasando `main()`->`exit-code`.
  Se usa `exit_group` porque `exit(60)` solo termina el *hilo* llamante y en un
  proceso single-thread el codigo de salida no se propaga de forma fiable
  (p.ej. WSL2 lo reporta como 0).  En PE el `_start` sale por
  `kernel32!ExitProcess`.

### Lo que es especifico de cada plataforma

- **Las librerias en C** del proyecto (`vesta_collections`, `vesta_math`,
  `libvesta_gc`) usan libc (`malloc`/`free`/`memcpy`/`qsort`/`sqrt`...).  Esto es
  **legitimo**: son librerias C y pueden depender de libc.  Pero por ello su `.a`
  es **especifica del formato del objeto**: una `.a` COFF (construida en Windows)
  solo sirve para binarios PE; para ELF hace falta la `.a` en formato ELF
  (construida en Linux).  Es la misma realidad que cualquier libreria C: el
  binario objeto es por-plataforma.
- En consecuencia: **construir en Windows produce binarios PE completos**
  (incluidas colecciones/math/gc estaticas); **construir en Linux produce
  binarios ELF completos**.  Cross-compilar las librerias C de un formato a otro
  requiere un toolchain cruzado (no es una limitacion del codegen, que ya emite
  ELF correcto).

### Nota para validar binarios ELF desde Windows

Al ejecutar un binario ELF a traves del puente msys -> WSL, el `$?` del shell no
es fiable (incluso un `gcc -static` de control devuelve 0).  El codigo de salida
**real** se obtiene con `gdb -q -batch -ex run <bin>` (lo reporta como
`exited with code 0NN`, en **octal**: `052` = 42).

---

## Vease tambien

- [[FFI]] - llamar a librerias del sistema y de terceros (`extern`).
- [[SmartPointers]] - liberacion determinista (RAII) sin recolector.
- [[Modulos]] - organizar el codigo en modulos antes de compilarlo.
