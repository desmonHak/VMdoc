# Estilo y formato de Vesta

Este documento es el **estandar de formato del lenguaje**: como queda cada
construccion de Vesta despues de pasar `vesta fmt`.  No es una guia de
recomendaciones -- es la especificacion de lo que el formateador PRODUCE, y por
tanto la unica forma valida de escribir Vesta.

> **Este documento esta para REVISARLO Y CAMBIARLO.**  Cada regla lleva un
> numero (`R1`, `R2`, ...) para poder decir "cambia la R12" sin tener que
> describirla.  Mientras el formateador no este escrito, cambiar una regla aqui
> cuesta editar una linea.  Despues cuesta reformatear el mundo.

Los bloques de ejemplo llevan **tabuladores reales**.  Si los ves muy anchos, es
que tu visor los pinta a 8; el estandar los mide a 4 (ver `R3`).

---

## Indice

- [Estilo y formato de Vesta](#estilo-y-formato-de-vesta)
  - [Indice](#indice)
  - [1. Los cuatro principios](#1-los-cuatro-principios)
  - [2. Reglas base: espacio en blanco](#2-reglas-base-espacio-en-blanco)
  - [3. Reparto de lineas largas](#3-reparto-de-lineas-largas)
  - [3.b. Alineacion en columnas](#3b-alineacion-en-columnas)
    - [El coste de alinear, y como se paga](#el-coste-de-alinear-y-como-se-paga)
  - [4. Comentarios](#4-comentarios)
  - [5. Fichero, namespace e imports](#5-fichero-namespace-e-imports)
  - [6. Declaraciones de nivel superior](#6-declaraciones-de-nivel-superior)
  - [7. Tipos](#7-tipos)
  - [8. Funciones y parametros](#8-funciones-y-parametros)
  - [9. Clases](#9-clases)
  - [10. Structs, enums y concepts](#10-structs-enums-y-concepts)
  - [11. Genericos](#11-genericos)
  - [12. Sentencias](#12-sentencias)
  - [13. Expresiones y operadores](#13-expresiones-y-operadores)
  - [14. Cadenas e interpolacion](#14-cadenas-e-interpolacion)
  - [15. Anotaciones](#15-anotaciones)
  - [16. Ensamblador en linea](#16-ensamblador-en-linea)
  - [17. Ownership y concurrencia](#17-ownership-y-concurrencia)
  - [17bis. Numeros](#17bis-numeros)
    - [Quien los escribe](#quien-los-escribe)
  - [18. Lo que el formateador NO toca](#18-lo-que-el-formateador-no-toca)
  - [19. Decisiones abiertas](#19-decisiones-abiertas)
  - [Como se comprueba que esto es cierto](#como-se-comprueba-que-esto-es-cierto)
  - [20. Ejemplo completo](#20-ejemplo-completo)
    - [Lo que demuestra, regla por regla](#lo-que-demuestra-regla-por-regla)

---

## 1. Los cuatro principios

Todo lo que sigue se deduce de estos cuatro.  Si una regla concreta te chirria,
mira si el principio detras te parece bien: cambiar el principio cambia muchas
reglas a la vez.

**P1. Una sola forma.**  No hay opciones.  Dos personas que escriban lo mismo
obtienen el mismo texto, byte a byte.  Se renuncia a gustos personales a cambio
de que ningun diff sea de estilo.

**P2. El formateador nunca cambia el programa.**  Solo se mueve espacio en
blanco y saltos de linea.  Se comprueba, no se promete: la lista de tokens antes
y despues tiene que ser identica.

**P3. Indentar es semantico; alinear es cosmetico.**  Lo primero dice a que
profundidad estas y es un TABULADOR, para que cada cual lo vea como quiera.  Lo
segundo pone cosas en columna y son ESPACIOS, para que no se descuadre al
cambiar el ancho del tabulador.

**P4. El autor manda en la logica; el formateador en la forma.**  Donde partir
una expresion larga lo sabe quien la escribio -- ahi esta la articulacion del
pensamiento --, asi que no se toca.  Donde partir una lista de argumentos es
mecanico, y eso si lo hace el formateador.

---

## 2. Reglas base: espacio en blanco

**R1.  Se indenta con TABULADORES, uno por nivel de anidamiento.**
Nunca espacios para indentar, nunca un tabulador a medias.

**R2.  Se alinea con ESPACIOS, nunca con tabuladores.**
Un tabulador solo aparece al principio de la linea.  A partir del primer
caracter que no sea tabulador, ya no hay tabuladores en esa linea.

**R3.  El ancho maximo de linea es 80, contando cada tabulador como 4.**
Quien vea los tabuladores a 8 vera lineas mas largas: es el precio de que la
indentacion sea configurable, y es deliberado.

**R4.  La llave de apertura va al final de la linea, con un espacio delante.**

```vesta
i32 main() {
	return 0;
}
```

La regla vale en las DOS direcciones: una llave que venga en su propia linea
sube al final de la anterior.  Sin eso el mismo programa tendria dos formas, que
es justo lo que `P1` no permite.

**R5.  La llave de cierre va sola en su linea, a la altura de quien abrio.**
Excepcion: los cierres que continuan (`} else {`, `} catch (E e) {`,
`} while (cond);`) van pegados, en la misma linea.

El `while` es el unico ambiguo: el de un `do` continua su llave y va pegado,
pero uno corriente empieza una sentencia y no.  Se escriben igual, asi que la
unica forma de distinguirlos es recordar quien abrio el bloque que se cierra.

Y nada de esto se junta si el `}` lleva detras un comentario de linea: el
`else` acabaria DENTRO del comentario.

**R6.  Un cuerpo de UNA sentencia puede ir sin llaves, pero solo si cabe en la
MISMA linea.  Partido en dos lineas, las llaves son obligatorias.**

```vesta
// Sin llaves: cabe entero en una linea.
if (n < 2) return n;
for (i64 v : valores) total += v;
while (quedan()) avanzar();

// No cabe: con llaves.
if (usuario.tiene_permiso() && !usuario.bloqueado()) {
	conceder_acceso(usuario, recurso, nivel);
}
```

La forma partida en dos lineas NO existe en Vesta, y no es por gusto.  Este es
el fallo de TLS de Apple de 2014 (`goto fail`), que dejo la verificacion de
certificados rota en todos sus sistemas:

```c
if (err != 0)
	goto fail;
	goto fail;      // la indentacion miente: NO esta dentro del if
```

La indentacion dice una cosa y el compilador entiende otra, y no hay nada que lo
delate.  Con `R6` ese fallo **no se puede escribir**: para anadir una segunda
sentencia hay que romper la linea, y en cuanto se rompe el formateador pone las
llaves.

**R6b.  El formateador normaliza en las dos direcciones.**  Un cuerpo de una
sentencia que cabe se junta en una linea; uno que no cabe recibe llaves.  Asi
sigue habiendo una sola forma (`P1`).

**R7.  Una linea en blanco como maximo entre sentencias.**  Dos o mas seguidas
se reducen a una.  Al principio y al final de un bloque no se deja ninguna.

**R8.  Dos lineas en blanco entre declaraciones de nivel superior**
(funciones, clases, structs, enums).

**R9.  Sin espacio en blanco al final de la linea.**  Ninguna linea acaba en
espacio o tabulador.

**R10.  El fichero acaba en un unico salto de linea.**

**R11.  Fin de linea `LF`.**  Los `CRLF` se normalizan al formatear.

---

## 3. Reparto de lineas largas

**R12.  Una lista que no cabe en 80 se reparte con TODOS sus elementos uno por
linea; y una que SI cabe se junta, aunque viniera partida.  Nunca a medias.**

Las dos direcciones, y por eso hay una sola forma (`P1`): un `sumar(i32 a, i32
b)` partido en cuatro lineas se junta, porque cabe.

Esta es la regla que mas cambia el aspecto del codigo, y la de mas
consecuencias.  El estado intermedio -- algunos elementos en la primera linea y
el resto abajo -- es lo que produce diffs ilegibles: anades un argumento y se
reordenan cinco lineas.  Con "todo o uno por linea" no hay ese estado.

```vesta
// Cabe: se queda en una linea.
i32 sumar(i32 a, i32 b) {

// No cabe: TODOS uno por linea.
i32 procesar(
	HashMap cache,
	unique<Buffer> entrada,
	borrow_mut<Estado> estado
) {
```

**R13.  SIN coma final.**

Se penso al reves -- la coma final hace que anadir un elemento sea un diff de
una linea y no de dos, que es su unica ventaja --, pero el lenguaje no la
admite: `i32 sumar(i32 a, i32 b,)` no compila, y un formateador que produce
codigo que no compila no sirve para nada.

Habilitarla seria tocar el parser para tolerar un token opcional antes de cada
cierre.  Se descarto: por un diff de una linea no compensa cambiar la gramatica,
y sin ella el reparto se lee mejor.

**R14.  Se reparten:** parametros, argumentos de llamada, listas de
inicializacion, listas de herencia, argumentos de tipo generico y listas de
`import` selectivo.

**R15.  NO se reparten:** expresiones aritmeticas y logicas, y condiciones
compuestas.  Ahi el salto lo pones tu, y donde lo pongas se queda.

**R94.  Una CADENA DE LLAMADAS si se reparte, con un eslabon por linea y el
punto al PRINCIPIO.**  El receptor se queda en la primera linea.

Es la excepcion a `R15`, y esta puesta mirando hacia adelante: **cuando Vesta
tenga UFCS** (`f(x, y)` escrito como `x.f(y)`), la cadena deja de ser un caso
raro y pasa a ser la forma normal de encadenar trabajo sobre un valor.  Un
formateador que no sepa repartirlas se queda corto justo donde mas se usa.

```vesta
// Cabe: en una linea.
i64 total = xs.filter(es_valido).sum();

// No cabe: un eslabon por linea.
i64 total = xs
	.filter(es_valido)
	.map(a_precio)
	.descartar(cero)
	.sum();
```

El punto va al principio de la linea, no al final de la anterior: asi cada linea
empieza diciendo QUE hace, y se leen en columna.

**R95.  O toda la cadena en una linea, o todos sus eslabones repartidos**
(`R12`).  Nunca dos arriba y el resto abajo.

**R96.  Una cadena de un solo eslabon no se reparte**: si `x.f(a, b, c)` no
cabe, lo que se reparte son sus ARGUMENTOS (`R12`), no la cadena.

```vesta
// El formateador NO toca estos saltos: los pusiste tu.
bool valido = tiene_permiso(usuario)
	&& !esta_bloqueado(usuario)
	&& cuota_restante(usuario) > 0;
```

**R16.  Una linea que pasa de 80 y no se puede repartir se deja como esta.**
El formateador no rompe nada por la fuerza.  El linter avisara, cuando exista.

**R101.  Los parentesis de una ANOTACION si se reparten, aunque no lleven
comas.**  Es la unica excepcion a `R15`.

```vesta
Section Sections[num_sections] @offset(
	e_lfanew + 24 + opt_size
) stride(40);
```

Ahi dentro va una expresion, no una lista, pero es el unico sitio por donde esa
linea se acorta sin tocar la logica: el campo, su tipo y su `stride` tienen que
seguir juntos para poder leerse.  La condicion de un `if` NO entra en la
excepcion -- ahi el salto sigue poniendolo quien escribe.

**R102.  Se prueban TODAS las listas de la linea, no solo la primera.**  En
`Section S[n] @offset(...) stride(40)` la primera es el `[n]` del array, que no
tiene por donde cortarse; la buena viene despues.  Quedarse con la primera
dejaba esa linea larga para siempre.

---

## 3.b. Alineacion en columnas

Vesta alinea.  Es la diferencia mas visible respecto a Go y a Rust, y es
deliberada: un bloque de declaraciones se lee como una tabla, y la columna dice
de un vistazo que es cada cosa.

> Los numeros de regla son IDENTIFICADORES, no orden de lectura -- como los
> codigos de diagnostico.  Esta seccion se anadio despues, asi que sus reglas
> empiezan en `R83`; las que ya existian conservan su numero para que las
> referencias no se rompan.

**R83.  Un BLOQUE DE ALINEACION son las lineas consecutivas que estan al mismo
nivel y tienen la misma forma.**  El bloque se rompe con una linea en blanco, un
comentario suelto, un cambio de nivel o una linea de otra forma.  Eso es lo que
te da el control: si no quieres que dos cosas se alineen, las separas con una
linea en blanco.

**R84.  Una declaracion se descompone en CAMPOS, y cada campo ocupa una columna
tan ancha como el elemento MAS LARGO del bloque, mas un espacio.**  Es la misma
idea que un ensamblador -- etiqueta, mnemonico, operandos, comentario --,
aplicada al lenguaje.

| Forma | Campos, en orden |
| :--- | :--- |
| Variable, constante, campo | modificadores, tipo, nombre, `= valor` |
| Con tipo funcion | modificadores, params, `->`, retorno, nombre, `= v` |
| `using` | `using`, nombre, `= tipo` |
| `typedef` | `typedef`, tipo, nombre |
| Valor de enum | nombre, `= valor` |
| Campo de un `@overlay` | tipo, nombre, `@offset` literal |
| Campo de bits | tipo, nombre, `: bits` |
| Instruccion de `asm` | etiqueta, mnemonico, operandos, comentario |

```vesta
u8             contador = 0;
i32            total    = 0;
unique<Buffer> datos    = unique_box(buffer(4096));
```

> **Se probo alinear con paradas fijas cada 4** -- que no depende de las
> vecinas, y por tanto no ensucia el historial (ver abajo).  Se descarto porque
> **no alinea**: con tipos de longitudes variadas, el `=` acaba en cinco
> columnas distintas y se pierde justo aquello por lo que se alineaba.  Quitar
> el ruido destruyendo el objetivo no es arreglarlo.

**R85.  Los campos se separan por UN espacio como minimo**, tambien en la linea
mas larga del bloque.

Se probaron 1, 2 y 4.  Con dos se distinguen mejor las columnas cuando hay tres
o mas -- `private  Referencia  ref` se lee como tabla y no como frase --, pero
cuesta unas tres columnas por linea y con 80 de limite eso se nota.  Se
elige uno, que es lo que hacen gofmt y los ensambladores: el contenido ya separa
solo, porque un tipo, un nombre y un `=` no se confunden entre si.

Y conviene no pedirle a esta regla lo que no puede dar: **ensanchar la
separacion no empuja a modularizar, empuja a REPARTIR LISTAS**, que es otra cosa
-- partir una llamada en cuatro lineas no extrae ninguna funcion --.  La presion
hacia dividir el programa viene de `R3`, del limite de 80 y de que la
indentacion se lo coma: a tres niveles quedan 68 columnas, y ahi si duele hasta
que se extrae algo.  El resto lo dira el linter, con reglas de longitud de
funcion y de profundidad de anidamiento.

**R86.  Si alinear hiciera que alguna linea pasara de 80, el bloque NO se
alinea.**  Un nombre muy largo no puede empujar a sus vecinos fuera del limite.

**R87.  Solo comparten columnas las lineas con los MISMOS campos.**
`typedef u32 Edad;` y `using MyInt = i64;` no comparten estructura -- uno es
`tipo nombre` y el otro `nombre = tipo` --, asi que forman bloques distintos
aunque esten pegados.  Lo mismo una declaracion con `const` y una sin el.

**R88.  Cuando las formas se mezclan pero comparten un ANCLAJE, se alinea solo
por el.**  Los anclajes son `=`, `=>` y `//`.

```vesta
public  get age  => this.age;
private get edad => this.edad;

public get age  => this.age;
public Animal() => this(0);
```

**R92.  Dentro de un tipo se alinea por el `->` y por nada mas.**  El `->` es la
articulacion visible del tipo funcion y separa dos campos que se leen aparte.
Los parentesis, las comas y los corchetes NO son campos: alinear por ellos
obliga a alinear la estructura interna del tipo, y eso no termina en ninguna
parte.

```vesta
fn(u8, u8) -> u8   suma    = (a, b) => a + b;
cfn(i64)   -> void deleter = liberar;
```

**R93.  Un tipo sin `->` no comparte columnas con uno que lo tenga** (`R87`).

```vesta
u8[16]   buffer;
string[] args;

typedef u32 Edad;
typedef u64 Marca;

using MyInt  = i64;
using Handle = u32;
```

### El coste de alinear, y como se paga

Al hacer que la anchura de una columna dependa del elemento MAS LARGO, la forma
de una linea pasa a depender de sus vecinas: anades un campo largo y se
reescribe el bloque entero.  Medido sobre los 620 bloques del corpus, eso
ocurre en el **48% de las altas**, y **no lo arregla que todo el equipo use el
mismo formateador** -- el resultado es identico y determinista para todos,
pero se reescribe igual.

Lo que cuesta:

  - `git blame` deja de senalar al autor real de esas lineas.
  - Dos ramas que anaden un campo cada una chocan en diez lineas, no en una.
  - Revisar obliga a buscar cual de las once lineas importa.

**Esto no se puede arreglar sin dejar de alinear.**  Que la forma de una
linea dependa de sus vecinas ES lo que significa alinear; no es un defecto
que se pueda pulir.  Se probo la alternativa -- paradas fijas, que no
dependen de nadie -- y no alinea (ver el recuadro de `R84`).  Asi que la
pregunta no es como evitarlo, sino si compensa.  Dos datos del corpus dicen
que si:

**Donde cae el ruido.**  El 91% de los bloques alineables son variables LOCALES
de una funcion y solo el 9% son campos de una estructura.  Y esa proporcion es
buena noticia: el ruido de las locales cae dentro de una funcion que el commit
ya esta tocando, asi que no ensucia nada que no estuviera ya sucio.  El que si
duele -- realinear veinte campos de un struct que referencia medio proyecto --
es el 9%, y los campos de un struct se anaden pocas veces.

**Como se descuenta.**  Todo el ruido es, por construccion, espacio en blanco y
nada mas -- que es justo lo que descarta la opcion `-w`, que NO es de git ni de
ninguna plataforma concreta: viene del `diff` de POSIX y la tienen todos.

```
git    blame -w        git diff -w
hg     annotate -w     hg  diff -w
svn    blame -x -w     svn diff -x -w
fossil                 fossil diff -w
p4                     p4 diff -db
```

Lo honesto es decir que hay que acordarse de pedirla, y que en una interfaz web
es un clic que mucha gente no da.  Si algun dia pesa mas el historial que la
lectura, quitar `R84` devuelve todo a un espacio sin tocar ninguna otra regla.

---

## 4. Comentarios

**R17.  Un comentario nunca se mueve de sitio ni cambia de texto.**  Se
reindenta con el codigo al que acompaña, y nada mas.

**R18.  Un comentario en su propia linea se indenta como la sentencia que
sigue.**

**R19.  Un comentario de fin de linea va separado por un espacio del codigo.**

```vesta
i32 total = 0; // acumulador
```

**R20.  Los comentarios de fin de linea consecutivos se alinean entre si.**

Es lo que convierte una tabla de datos comentada en algo que se lee:

```vesta
u8 code[8] = {
	0x48, 0x89, 0xE5,       // mov rbp, rsp
	0x48, 0x83, 0xEC, 0x20, // sub rsp, 32
	0xC3                    // ret
};
```

El bloque se rompe con una linea sin comentario, igual que los demas (`R83`), y
si estirarlo sacara el comentario de las 80 columnas se deja donde estaba
(`R86`): un comentario empujado fuera de la pantalla no se lee mejor por estar
alineado.

**R103.  Una tabla de valores conserva SUS FILAS, y sus columnas se alinean por
la DERECHA.**

```vesta
i64 mat[9] = {
	 1, 2,   3,
	40, 5, 600,
	 7, 8,   9
};
```

Las filas las decidio quien escribio -- son la forma de la tabla, y ninguna
regla de ancho sabe mejor que el como se agrupan sus datos --, pero las columnas
son cosa del formateador.  Por la derecha porque asi se leen los numeros: con
las unidades en la misma columna, y ahi se ve de un vistazo si una cifra se sale
de rango.

**R104.  Una tabla con COMENTARIOS dentro no se realinea.**

Un array de datos comentado no es una tabla de numeros: es una EXPLICACION.  Los
opcodes de un trozo de codigo maquina, una tabla de saltos, una cabecera binaria
-- ahi los valores se agrupan a mano para que se entienda que hace cada tramo, y
mover las columnas rompe justo lo que el comentario estaba explicando.

```vesta
u8 c2[8] = {
	0x48, 0x89, 0xE5,
	/*extension*/ 0x48, 0x83, /*opcode*/ 0xEC, 0x20,
	0xC3
};
```

Vale igual para un comentario al final de la fila que para uno metido en medio.
Lo que si se alinea son los comentarios entre si (`R20`), que no toca los
valores.

```vesta
u8 rojo   = 0; // canal R
u8 verde  = 0; // canal G
u8 azul   = 0; // canal B
```

**R105.  Un comentario que no cabe en 80 columnas se deja como esta.**

Ni se parte, ni se acorta, ni se baja a otra linea.  Partir un comentario es
reescribir una frase que escribio otro, y el formateador no sabe donde termina
una idea; `R17` ya dice que un comentario no cambia de texto, y esto es esa
misma regla cuando la linea no cabe.

Dos consecuencias concretas:

  - Si un comentario de fin de linea YA se salia del limite, no se alinea con
    sus vecinos (`R20`): moverlo solo lo empujaria mas lejos.
  - Si alinearlo lo SACARA del limite, tampoco.  La comprobacion mira donde
    ACABA la linea, no donde empieza el comentario: mirando el inicio se colaba
    justo el caso que importa.

Avisar de un comentario demasiado largo es cosa del linter, que si puede decirlo
sin tocar nada.

**R21.  Un comentario de bloque cuyas lineas empiezan por `*` se reindenta
alineando los asteriscos bajo el `/*`.**  Es la forma normal de un comentario
largo, y alinearlos es lo que hace que se lea como un bloque y no como texto
suelto.

```vesta
/*
 * Lo que hace esta funcion, contado en varias lineas.
 *
 * El asterisco de cada linea queda bajo el del `/*`, y el `*/` cierra
 * a su altura.
 */
```

**R21b.  Un comentario de bloque que NO sigue ese patron se deja intacto.**
Solo se reindenta su primera linea.  Ahi dentro puede haber un diagrama, una
tabla o codigo comentado, y realinearlo lo destroza.

```vesta
/* tabla de estados:
     LIBRE  -> RESERVADA -> VENDIDA
                        \-> LIBRE
*/
```

**R97.  Los comentarios de linea `//` consecutivos y al mismo nivel forman un
bloque y se alinean entre si** (`R83`).  Si uno va al final de una linea de
codigo, el anclaje es el `//` (`R88`); si van solos, ya quedan alineados por la
indentacion.

---

## 5. Fichero, namespace e imports

**R22.  Orden del fichero:** comentario de cabecera, `namespace`, `import`,
declaraciones.

**R23.  Los `import` no se reordenan ni se agrupan.**  El orden lo pusiste tu y
puede tener un motivo.

**R24.  Un `import` selectivo que no cabe se reparte uno por linea** (`R12`).

```vesta
namespace app.render;

import std.io;
import std.collections.{
	ArrayList,
	HashMap,
	TreeMap,
};
extern import "stdlib/native/io/vesta_io";
```

**R25.  Un `namespace` de bloque indenta su contenido un nivel.**

```vesta
namespace ui.widgets {
	class Button {
		public void draw() { }
	}
}
```

---

## 6. Declaraciones de nivel superior

**R26.  Una declaracion por linea.**  Sin declaraciones multiples separadas por
coma.  Las consecutivas se alinean (`R83`-`R88`).

```vesta
      u8  contador = 0;
const f64 PI       = 3.14159;

typedef u32 Edad;
typedef u64 Marca;

using MyInt  = i64;
using Handle = u32;
```

---

## 7. Tipos

**R27.  El asterisco de puntero va PEGADO AL TIPO**, no al nombre: en Vesta el
tipo es una unidad y no hay declaraciones multiples que lo desmientan (`R26`).

```vesta
i64* p = &x;
i64** pp = &p;
void* opaco = null;
```

**R28.  Sin espacios dentro de `<>` de un generico.**

```vesta
HashMap<string, ArrayList<i64>> indice = hashmap(64);
```

**R29.  `>>` de genericos anidados se escribe pegado.**

```vesta
VirtualPtr<VirtualPtr<i64>> pp = ...;
```

**R30.  Los corchetes de array van pegados al tipo, sin espacios dentro.**  Los
nombres se alinean (`R84`).

```vesta
u8[16]   buffer;
string[] args;
```

**R31.  `?` y `!!` van pegados al tipo o al nombre, sin espacio.**

```vesta
Animal? quiza = null;
nonnull Animal seguro = a;
i32 f(Punto !!p) { }
```

**R32.  En un tipo funcion, `->` lleva un espacio a cada lado.**  Se alinean el
nombre y el `=`; y tambien el `->`, que es la articulacion del tipo (`R92`).

```vesta
fn(u8, u8) -> u8   suma    = (a, b) => a + b;
cfn(i64)   -> void deleter = liberar;
```

---

## 8. Funciones y parametros

**R33.  Sin espacio entre el nombre y el parentesis.**

**R34.  Sin espacios pegados por dentro de los parentesis.**

**R35.  Un espacio despues de cada coma, ninguno antes.**

```vesta
u64 fibonacci(u64 n) {
	if (n < 2) {
		return n;
	}
	return fibonacci(n - 1) + fibonacci(n - 2);
}
```

**R36.  Una funcion sin parametros se escribe `()`, sin espacio dentro.**

**R37.  El variadico lleva los puntos pegados al tipo.**

```vesta
i64 sumar_todos(i64... valores) {
	i64 total = 0;
	for (i64 v : valores) {
		total += v;
	}
	return total;
}
```

**R38.  `register("rXX")` va delante del tipo, separado por un espacio.**

```vesta
i64 leer_contador(register("rax") i64 slot) { }
```

**R39.  Un cuerpo de expresion `=>` cabe en una linea o se reparte por su
lista, nunca se convierte en bloque.**  Los `=>` consecutivos se alinean, y con
ellos los modificadores (`R84`).

```vesta
public  get age  => this.age;
private get edad => this.edad;

public get age  => this.age;
public Animal() => this(0);
```

---

## 9. Clases

**R40.  La lista de herencia va tras `:` con un espacio a cada lado, comas
separadas por espacio.**  Si no cabe, se reparte (`R12`).

**R41.  Orden dentro de la clase: campos, constructores, metodos, propiedades,
destructor.**  El formateador **no reordena**: es una convencion que verificara
el linter.

**R42.  Los modificadores van en este orden: acceso, `static`, `final`.**

**R43.  Un miembro por linea, separados por una linea en blanco como maximo.**

```vesta
class Animal : Nombre, IFoo, IBar {
	public u8 age = 0;
	private static u64 counter = 0;

	public Animal() => this(0);

	public Animal(u8 age) {
		this.age = age;
		Animal.counter += 1;
	}

	public u8 method1() {
		return this.age + 1;
	}

	public get age => this.age;

	public set age(u8 v) {
		this.age = v;
	}

	@Override
	public string toString() {
		return "Animal ${age}";
	}

	public ~Animal() { }
}
```

**R44.  Un cuerpo vacio se escribe `{ }`, con un espacio dentro.**  Distingue de
un vistazo "no hace nada" de "se me olvido".

**R45.  `impl` se formatea como una clase.**

```vesta
impl Punto {
	f64 modulo() => sqrt(this.x * this.x + this.y * this.y);
}

impl Comparable for Punto {
	i32 compare(Punto otro) {
		return 0;
	}
}
```

---

## 10. Structs, enums y concepts

**R46.  Los campos de un struct se alinean** (`R83`-`R86`).

```vesta
struct Cabecera {
	u32    magic;
	u16    version;
	u16    flags;
	u64    offset;
	string nombre;
}
```

**R47.  Los valores de un enum se alinean por su `=`** (`R84`).

**R99.  Los campos de BITS se alinean por sus dos puntos.**

Lo que se compara de un vistazo en un `struct` de bits es cuantos ocupa cada
campo, y si suman lo que tienen que sumar; con las anchuras desalineadas eso hay
que ir sumandolo a mano.

```vesta
struct Flags {
	u32 a    : 3;
	u32 b    : 5;
	u32 c    : 8;
	u32 rest : 16;
}
```

Estos dos puntos son el SEPTIMO uso del simbolo, y el unico que no hizo falta
distinguir: lleva espacios a los dos lados igual que el ternario, asi que la
regla general ya acertaba.  La forma es inconfundible de todas formas -- tipo,
nombre, dos puntos, un entero y punto y coma --, que es lo que permite
alinearlos sin cruzarlos con `case Foo:` ni con una herencia.

**R100.  Un campo con BLOQUE se formatea como un metodo, y no se alinea con los
demas.**

El desplazamiento de un campo no tiene por que ser un numero: puede calcularse.
Ese campo deja de tener la forma de una declaracion -- su linea acaba en `{`, no
en `;` -- y por tanto no comparte columnas con los de offset literal (`R87`).
Su cuerpo se indenta como cualquier bloque, y lo que venga tras el cierre
-- `stride(N)` -- se queda pegado a la llave, como un `} else {`.

Alinearlo con los simples seria peor: la columna la fijaria el mas largo de dos
cosas que no se leen juntas, y los offsets literales -- que si se comparan entre
si -- perderian su rejilla.

```vesta
@overlay struct ImportDesc {
	u32 oft_rva  @0x00;
	u32 name_rva @0x0C;
	u32 ft_rva   @0x10;

	Thunk thunks[] @offset {
		u32 rva = this.oft_rva;
		if (rva == 0) rva = this.ft_rva;
		return parent<PeImage>().translate(rva);
	} stride(8);

	u8 dll_name @offset {
		return parent<PeImage>().translate(this.name_rva);
	};
}
```

**R98.  Los campos de un `@overlay struct` se alinean por su DESPLAZAMIENTO.**

Una vista no lleva `=`: lleva el sitio que ocupa cada campo en la memoria ajena
que describe.  Y alinear ahi importa mas que en ningun otro sitio, porque lo que
se esta escribiendo es la disposicion de un formato binario y los offsets en
columna son justo lo que se compara contra la especificacion.

```vesta
@overlay struct DosHeader {
	u16 e_magic  @0x00;
	i32 e_lfanew @0x3C;
}
```

```vesta
enum Color {
	Rojo  = 1,
	Verde = 2,
	Azul  = 4,
}
```

**R48.  Un enum con carga (ADT) va uno por linea.**

```vesta
enum Maybe<T> {
	Some(T),
	None
}
```

**R49.  Un `concept` de predicado cabe en una linea; el de bloque se formatea
como una funcion.**

```vesta
concept Numeric<T> = is_int<T>() || is_float<T>();

concept Sumable<T> {
	comptime {
		return has_method<T>("add");
	}
}
```

---

## 11. Genericos

**R50.  Los parametros de tipo van pegados al nombre, sin espacios dentro.**

**R51.  Una clausula `where` va en su propia linea, indentada un nivel, con un
requisito por linea si no cabe.**

```vesta
T mayor<T>(T a, T b) where T: Comparable {
	return a > b ? a : b;
}

R reducir<T, R>(ArrayList xs, R inicial)
	where T: Numeric,
	      R: Numeric + Default
{
	return inicial;
}
```

> La ultima forma es la unica del documento donde la llave de apertura va sola
> en su linea: tras un `where` repartido, pegarla al ultimo requisito la
> escondia.  Ver `D3` en decisiones abiertas.

---

## 12. Sentencias

**R52.  Un espacio entre la palabra clave y su parentesis** (`if (`, `while (`,
`for (`, `switch (`, `catch (`).  Esto la distingue de una llamada (`R33`).

**R53.  `else`, `catch` y `finally` van pegados a la llave de cierre.**

```vesta
if (cond) {
	uno();
} else if (otra) {
	dos();
} else {
	tres();
}
```

**R89.  Una cadena `if` / `else if` / `else` cuyas ramas caben todas en una
linea se alinea por el `(` y por el cuerpo** (`R6`, `R84`).  O se alinea la
cadena entera, o ninguna de sus ramas: media cadena alineada se lee peor que
ninguna.

```vesta
if      (cond) uno();
else if (otra) dos();
else           tres();
```

**R54.  En un `for` clasico, un espacio despues de cada punto y coma.**

```vesta
for (i32 i = 0; i < n; i += 1) {
	procesar(i);
}
```

**R55.  En un foreach, un espacio a cada lado de `:` y de `in`.**

```vesta
for (i64 x : valores) { }
for (x in valores) { }
```

**R56.  `do ... while` cierra en la misma linea, con punto y coma.**

```vesta
do {
	avanzar();
} while (quedan());
```

**R57.  Una etiqueta de `goto` va sin indentar, a la altura de la llave.**

```vesta
i32 buscar() {
	for (i32 i = 0; i < n; i += 1) {
		if (encontrado(i)) {
			goto salir;
		}
	}
salir:
	return -1;
}
```

**R58.  `try` / `catch` / `finally` como `R53`; `synchronized` y `monitor` como
un bloque normal.**

```vesta
try {
	arriesgado();
} catch (FatalError e) {
	println("fallo: ${e}");
} finally {
	limpiar();
}

synchronized (obj) {
	contador += 1;
}

monitor (obj) {
	wait;
	notify;
}
```

**R59.  En `match`, cada `case` va a un nivel; su cuerpo a dos.**

**R60.  Un `case` con cuerpo de una expresion usa `=>` y cabe en una linea.**

```vesta
match (valor) {
	case Some(x) if x > 0 => positivo(x);
	case Some(x) => cero_o_menos(x);
	case None => {
		registrar("sin valor");
		return 0;
	}
}
```

---

## 13. Expresiones y operadores

**R61.  Un espacio a cada lado de todo operador binario y de asignacion.**

**R62.  Sin espacio tras un operador unario** (`!`, `-`, `&`, `*`, `~`).

**R63.  Sin espacio antes del punto y coma, ni antes de la coma.**

**R64.  Sin espacio alrededor de `.` ni de `[ ]` de indice.**

**R65.  El ternario lleva espacio a cada lado de `?` y `:`.**

**R66.  Un cast va pegado a lo que convierte.**

```vesta
i64 total = (a + b) * c - d / 2;
bool ok = !fallo && (n > 0 || forzar);
i32 corto = (i32) largo;
f64 x = puntos[i].coord;
i32 signo = n < 0 ? -1 : 1;
i64* q = &puntos[0];
```

**R67.  El formateador NO anade ni quita parentesis.**  Los que pusiste para
que se lea mejor se quedan.

**R68.  Una lambda de expresion no lleva llaves; una de bloque se formatea como
una funcion.**

```vesta
fn(u8, u8) -> u8 suma = (a, b) => a + b;

xs.foreach((i64 x) => {
	println("${x}");
});
```

---

## 14. Cadenas e interpolacion

**R69.  El contenido de una cadena NO se toca.**  Ni los espacios, ni los
escapes, ni lo que haya dentro de `${...}`.

**R70.  El contenido de una cadena de triple comilla no se reindenta.**  Los
espacios de dentro son parte del valor.

```vesta
string plantilla = """
    linea uno
    linea dos
""";
```

**R71.  Una cadena larga no se parte.**  Partir una cadena cambia el programa.

---

## 15. Anotaciones

**R72.  Una anotacion va en su propia linea, encima de lo que anota.**  Aunque
viniera pegada: `@overlay struct PeImage` se parte en dos lineas.

```vesta
@overlay
struct PeImage {
```

Solo la que ABRE la linea.  El `@offset(...)` de un campo va en medio de su
declaracion y forma parte de ella (`R76`): romperlo ahi partiria el campo por la
mitad.

**R73.  Varias anotaciones van una por linea, en el orden que las pusiste.**

**R74.  Una anotacion sin argumentos se escribe sin parentesis.**

**R75.  Sin espacios dentro de los parentesis de una anotacion.**

```vesta
@Target("os:windows")
@Naked
void trampolin() { }

@Async
@complexity(O(n), n = arg0.size)
i32 procesar(ArrayList xs) {
	return 0;
}
```

**R76.  Una anotacion de parametro va en linea, antes del tipo** (no rompe la
lista).

---

## 16. Ensamblador en linea

**R77.  Dentro de un bloque `asm`, la INDENTACION es tuya y las COLUMNAS son
del formateador.**

Es el mismo reparto que `P3` hace en todo el lenguaje, aplicado aqui: indentar
dice algo (donde empieza un bucle, que va dentro de que) y eso lo decides tu;
alinear en columna es cosmetico y lo hace la maquina.  Asi la indentacion sigue
sirviendo para marcar estructura -- que es lo que se perderia si el formateador
no tocara nada, y tambien lo que se perderia si lo reindentara todo a la fuerza.

**R78.  El bloque entero se reindenta como una sentencia, manteniendo la
indentacion RELATIVA de dentro.**

**R79.  Las etiquetas locales conservan su nivel.**  No se mueven al margen ni
se indentan con las instrucciones: donde las pusiste, se quedan.

**R90.  Los anclajes del asm son los OPERANDOS y el COMENTARIO.**  Se alinean
por bloques, con las mismas reglas que el resto del lenguaje (`R83`-`R86`): el
bloque se rompe con una linea en blanco, una etiqueta o un cambio de nivel.

```vesta
i32 comparar(u8* a, u8* b, i64 n) {
	asm volatile {
		xor   r8, r8
	.loop:
		cmp   r8, r9
		jge   .fin
		movzx r10, byte [rdi + r8]   // ca
		movzx r11, byte [rdx + r8]   // cb
		cmp   r10, r11
		jne   .distinto
		inc   r8
		jmp   .loop
	.fin:
		xor   rax, rax
	.distinto:
		mov   rax, 1
	}
	return 0;
}
```

El mnemonico mas largo del bloque (`movzx`) fija la columna de los operandos, y
el comentario mas a la derecha fija la del comentario.  Una etiqueta rompe el
bloque, asi que cada tramo se alinea por su cuenta.

**R91.  Lo que hay DENTRO de una instruccion no se toca**: ni el orden de los
operandos, ni los espacios de dentro de `[rdi + r8]`, ni los sufijos.  Ahi ya no
es formato, es el programa.

**R80.  Una funcion `@Asm` entera se trata igual.**

---

## 17. Ownership y concurrencia

**R81.  Los tipos de ownership no llevan nada especial:** se formatean como
cualquier generico (`R28`).

**R82.  Un `spawn` con bloque se formatea como un bloque normal.**

```vesta
unique<Buffer> propio = unique_box(buffer(4096));
shared<Cache> compartido = shared_box(cache(64));
borrow<Cache> prestado = lend(compartido);

Future<i32> f = fetch("...");
i32 r = await f;

spawn {
	trabajar();
}

spawn on(2) {
	trabajar_ahi();
}
```

---

## 17bis. Numeros

Un numero largo no se lee: se **cuenta**.  `10000000` obliga a contar ceros para
saber si son diez millones o cien.  Las cuatro reglas de esta seccion hacen lo
mismo en cada base -- agrupar por lo que ahi significa algo -- y ninguna cambia
el valor del literal.

**R106.  Los digitos hexadecimales van en MAYUSCULA; el prefijo, en minuscula.**

La `x` minuscula se distingue del digito que la sigue, y los digitos en
mayuscula se distinguen de un identificador.  Es lo que ya usa el 90% del
codigo escrito, y lo habitual en tablas de opcodes y volcados de memoria.

```vx
0xFF   0x0F   0xDEADBEEF        // asi
0xff   0Xff   0xdeadBEEF        // no
```

**R107.  Un literal con base explicita se rellena con ceros hasta una anchura
que nombre un tipo.**

| Base | Se agrupa de | Anchuras |
| :--- | :--- | :--- |
| `0x` | 2 digitos (un byte) | 2, 4, 8, 16 -- que son `u8`, `u16`, `u32`, `u64` |
| `0b` | 4 bits (un digito hex) | 4, 8, 12, 16, 20, ... |
| `0o` | 3 digitos (un byte, redondeando) | 3, 6, 9, ... |

```vx
0xF        -> 0x0F              // un byte
0xFFF      -> 0x0FFF            // dos
0xABCDEF   -> 0x00ABCDEF        // 6 digitos no son ningun tipo: sube a 8
0b101      -> 0b0101
0o7        -> 0o007
```

El decimal no se rellena: un cero por delante no dice nada, y en C hasta
significa otra cosa.  Un literal mas ancho que `u64` se deja como esta.

Excepcion: si YA lleva `_` entre los digitos, no se toca.  Ahi el `_` marca los
campos de un formato (`0xDEAD_BEEF`, `0b1010_0001`) y reagrupar seria discutir
con quien lo escribio.

**R108.  Un literal LLEVA su tipo, y el sufijo va separado con `_`.**

Un numero suelto no dice de que tipo es, y para saberlo hay que ir a buscar la
declaracion.  El sufijo lo dice en el sitio, que es lo mismo que hacen `L` y
`F` en C con los nombres de Vesta.  Vale en las cuatro bases.

```vx
42_i8      3.14_f64      0xFF_u32      0b1010_u8      0o755_u16      1e9_i64
42i8       3.14f64       0xFFu32                                     // no
```

Pegado al numero se confunde con el, y en hexadecimal hasta con un digito: por
eso el `_`.

El formateador lo PONE donde puede saber el tipo sin compilar, que es la
declaracion -- `TIPO nombre = <literal>;` --, contando el signo como parte del
valor (`i8 x = -128_i8;` cabe, `127` es el tope por arriba).  Donde no se puede
saber -- un argumento, una expresion -- no se inventa ninguno.

Tres sitios donde NO se pone, y los tres cambiarian el programa:

- un valor que **no cabe** en el tipo declarado (`u8 x = 300;`).  Es un error
  del autor y lo dice el compilador; el formateador no lo empeora dejando
  ademas el fichero sin compilar por otro sitio.
- un entero declarado de tipo flotante (`f64 x = 1;`): el sufijo tiene que ser
  de la misma familia que el literal.
- dentro de un bloque `asm`, donde los numeros son operandos de la maquina y no
  valores de Vesta (`mov rax, 42`).  Es la misma linea que traza `R77`.

**R109.  Un decimal de cinco digitos o mas agrupa sus millares con `_`.**

Cuatro digitos se leen de un vistazo (`1000`, un ano como `2026`) y un `_` solo
anadiria ruido.  Cinco ya no: `10000` puede ser diez mil o cien mil.

```vx
1_000_000      65_536      10_000      1_000_000.25
1000           2026        255                       // se quedan
```

Solo la parte ENTERA: detras del punto no hay millares que contar, y agrupar
ahi deja restos como `0.123_456_7`, que se lee peor que el numero sin tocar.

**R114.  Una expresion partida dejando el operador al final alinea esos
operadores en columna.**

Es como se escribe una mascara de bits larga, y los operadores en columna son
lo que deja ver que la lista esta completa: falta uno y salta a la vista.  Sin
alinearlos cada `|` queda pegado a su operando y la lista se lee como un
parrafo.

```vx
public const u32 TOKEN_ALL_ACCESS = (
    STANDARD_RIGHTS_REQUIRED |
    TOKEN_ASSIGN_PRIMARY     |
    TOKEN_DUPLICATE          |
    TOKEN_QUERY
);
```

Vale para cualquier operador binario al final de linea (`|`, `&`, `^`, `+`,
`-`, `*`, `/`, `%`, `||`, `&&`, `<<`, `>>`).

**R112.  Un contrato repartido es una tabla de dos columnas.**

Una anotacion que no cabe se parte como cualquier lista -- uno por elemento
(`R12`) -- pero ademas sus VALORES se ponen en columna, porque lo que hay
dentro son pares `etiqueta: valor` y se leen comparandolos entre si:

```vx
@nothrow
@nopanic
@alloc(partial: 0, total: 0)
@stack(0, when: arch:x86_64)
@stack(partial: 0, total: 32, when: arch:arm64)
@complexity(
	partial_pre:  O(1),
	partial_post: O(1),
	total_pre:    O(n),
	total_post:   O(n)
)
```

La etiqueta mas larga fija la columna, igual que en un bloque de declaraciones
(`R83`-`R88`).  Las que caben en una linea se quedan en una linea.

Cada anotacion va en su propia linea (`R72`), aunque se escribieran pegadas.

Dentro de los parentesis, el `:` de una etiqueta va pegado a su nombre --
`partial: 0`, como en un mapa -- y los siguientes del mismo argumento van
pegados por los dos lados, porque ahi `clave:valor` es una sola cosa:
`when: arch:arm64`.

Una anotacion SIN comas no es una lista sino una expresion, y se sigue
partiendo por su apertura (`R101`).

### Quien los escribe

Formatear y CALCULAR un contrato son dos cosas distintas, y por eso son dos
opciones distintas:

| | Que hace | Cuesta |
| :--- | :--- | :--- |
| `vesta fmt` | coloca los contratos que ya estan escritos | solo el lexer |
| `--analyze --analyze-write` | los CALCULA y los escribe | compila el fichero |

Saber el coste, el marco de pila o si una funcion puede lanzar exige compilarla
y recorrer su IR.  Formatear, en cambio, tiene que ser instantaneo -- se hace
en cada guardado del editor --, asi que la libreria del formateador no depende
de nada mas que del lexer y jamas va a compilar nada.

Lo que si comparten es la FORMA: `--analyze-write` pasa cada contrato que
genera por el formateador, de modo que lo que escribe ya cumple `R112` y
`vesta fmt` lo deja igual.  La forma se decide en un solo sitio.

Del fichero anotado solo se tocan las anotaciones: quien pidio anotar no pidio
reformatear su codigo.  Si el fichero esta escrito con espacios, el contrato
sale con espacios; si con tabuladores, con tabuladores.

**R111.  El signo de un numero tiene su propia columna.**

Un `-` (o un `+`) corre su numero una columna respecto a los de al lado, y las
cifras dejan de poder compararse de un vistazo, que es justo para lo que se
ponen en columna:

```vx
i8 minimo = -128_i8;        i8 minimo = -128_i8;
i8 maximo = 127_i8;         i8 maximo =  127_i8;
```

Vale para cualquier valor, no solo para un literal: una funcion que devuelve un
numero se puede negar igual, y entonces corre su linea lo mismo que un `-128`.

```vx
i64 a = -suma(1, 2);
i64 b =  suma(3, 4);
```

Si en el bloque nadie lleva signo, no hay ninguna columna que reservar.

**R110.  Dentro de una llamada que captura el TEXTO de su argumento no se toca
nada.**

Una funcion con un parametro `expr` no recibe el valor del argumento sino su
texto tal como se escribio.  Ahi los espacios son contenido:

```vx
comptime string emit(expr code) { ... }

string s = emit( print("hola"); );   // s vale ` print("hola"); `
```

Quitar ese espacio no reordena el programa: lo reescribe.  Y `P2` no puede
verlo, porque los tokens son identicos y la diferencia solo aparece al ejecutar
el comptime.  Se descubrio asi -- un compilador de Brainfuck escrito en
comptime dejo de compilar al formatearlo, y el unico cambio era un espacio.

El formateador reconoce esas llamadas por tres vias, ninguna de las cuales
cablea el nombre de una funcion concreta:

1. Las declaradas en el propio fichero, buscando `expr` en sus parametros.
2. Las importadas, que se las pasa quien tenga los imports resueltos -- el
   compilador o el LSP -- en `raw_capture_names`.
3. Un `;` entre los parentesis de una llamada: una expresion Vesta nunca lleva
   uno, asi que lo de dentro no es una expresion sino texto.

---

## 18. Lo que el formateador NO toca

Resumen de las excepciones, que es lo que conviene tener claro de un vistazo:

| Que | Por que |
| :--- | :--- |
| La indentacion dentro de `asm` (`R77`) | Ahi marca estructura, y es tuya |
| Dentro de una instruccion (`R91`) | El orden de los operandos es el programa |
| El contenido de las cadenas (`R69`) | Cambiarlo cambia el valor |
| El texto de los comentarios (`R17`) | No es suyo |
| Donde partiste una expresion (`R15`) | Ahi esta tu articulacion logica |
| Los `import` (`R23`) | El orden puede tener un motivo |
| El orden de los miembros (`R41`) | Es cosa del linter, no de mover codigo |
| Los parentesis que pusiste (`R67`) | Los pusiste para leerlo |
| El argumento de una captura `expr` (`R110`) | Ahi los espacios son el dato |
| Un `_` ya puesto en un hex o binario (`R107`) | Marca los campos de un formato |

---

## 19. Decisiones abiertas

Lo que aun no esta cerrado.  Cada una cambia reglas concretas.

**D0.  la comprobacion de `P2` se hace por INTENCION.**

`P2` se comprobaba comparando la lista de TOKENS -- una aproximacion
conservadora a "el programa no cambia", que no necesita el parser.  El precio
era que prohibia cualquier regla que anadiera, quitara o moviera un token,
aunque el programa siguiera siendo el mismo: eso dejaba fuera `R6`, `R22`,
`R29`, `R42` y `R74`.

Las cinco equivalencias se comprobaron EJECUTANDO las dos formas y comparando
el resultado; las cinco daban lo mismo.  Lo que fallaba no eran las reglas,
era la forma de comprobarlas.

Ahora el formateador **declara** cada transformacion que aplica, y la
comprobacion exige que la diferencia entre el antes y el despues sea
exactamente la declarada.  Una diferencia que nadie declaro sigue siendo un
fallo, aunque caiga en una regla conocida: la red se queda igual de fina, y
solo se abren las puertas que se han medido una a una.

Cuatro estan implementadas (`R6`, `R29`, `R42`, `R74`).  **`R22` se queda
fuera**, y no por dificultad: en los 811 ficheros del proyecto no hay NI UNO
con un `import` fuera de sitio.  Mover bloques enteros es ademas la unica que
necesitaria otro mecanismo de verificacion -- las otras cuatro sustituyen
tokens vecinos --, y el linter puede exigir el orden sin tocar el fichero.
Complejidad para arreglar cero casos.

**D1.  CERRADA: se alinea.**  Ver la seccion 3.b y su apartado sobre el coste.
Lo que queda vivo de aquella duda es solo el umbral: hoy el unico freno es el
limite de 80 (`R86`).  Si aparece un bloque donde un nombre larguisimo desplaza
a diez vecinos sin llegar a pasarse de 80, habra que anadir un segundo freno.

**D2.  `R41` (orden de los miembros): dejarlo al linter, o que el formateador
reordene.**  Reordenar es comodo, pero mueve codigo -- y `P2` dice que solo se
mueve espacio en blanco.  Hoy la propuesta es no reordenar.

**D3.  `R51` (llave tras un `where` repartido).**  Es la unica excepcion a `R4`
en todo el documento.  La alternativa es pegarla al ultimo requisito, que la
esconde.  Una excepcion es fea; una llave escondida tambien.

**D4.  Ancho 80 (`R3`).**  Con tabuladores medidos a 4.  Si el codigo con
genericos y ownership resulta apretado, 100 es la otra opcion razonable.

**D5.  `R8` (dos lineas entre declaraciones de nivel superior).**  Una tambien
es defendible; dos separa mejor cuando las funciones son cortas.

**D6.  Que hacer con una linea que ya cabe pero esta repartida.**  Hoy `R12` la
JUNTA (si cabe entera, va entera).  Eso reformatea codigo que quiza partiste a
proposito.  La alternativa es no juntar nunca, solo repartir -- pero entonces se
pierde la forma unica de `P1`.

---

## Como se comprueba que esto es cierto

Dos propiedades, verificables sobre los 502 ficheros de `examples_codes_vx/` y
`stdlib/vx/`:

```
fmt(fmt(x)) == fmt(x)          idempotente: formatear dos veces da lo mismo
tokens(fmt(x)) == tokens(x)    P2: el programa no cambia, solo su forma
```

La segunda es la que impide que el formateador rompa codigo en silencio, y es
barata: la lista de tokens ya la produce el lexer.

---

## 20. Ejemplo completo

Todo lo anterior junto, en un modulo que existe solo para tocar cada regla.  No
esta escrito a mano: lo genera un script que APLICA las reglas, porque un
ejemplo mal alineado en un documento sobre alineacion no valdria nada.  Si
cambias una regla, este ejemplo cambia con ella.

Lleva tabuladores reales y ninguna linea pasa de 80 columnas midiendolos a 4.

```vesta
/*
 * catalogo.vx -- ejemplo completo del estilo de Vesta.
 *
 * Toca a proposito todas las reglas: alineacion por campos, reparto de
 * listas, cadenas al estilo UFCS, cuerpos de una linea, asm con sus
 * columnas, ownership y comentarios.
 */

namespace app.catalogo;

import std.io;
import std.collections.{
	ArrayList,
	HashMap
};
extern import "stdlib/native/io/vesta_io";


// R87: typedef y using tienen campos distintos -> son dos bloques.
typedef u32 Referencia;
typedef u64 Marca;

using Precio = i64;
using Cesta  = ArrayList;

// R87 otra vez: con const y sin el tampoco comparten columnas.
const f64 IVA        = 0.21;
const i64 MAX_PIEZAS = 4096;

u32 siguiente_ref = 1;


/// Estado de una pieza en el almacen.
enum Estado {
	Libre     = 0,
	Reservada = 1,
	Vendida   = 2
}


struct Medidas {
	f64 ancho;  // en milimetros
	f64 alto;   // idem
	u32 gramos; // peso en seco

	f64 area() => this.ancho * this.alto;
}


class Pieza : Articulo, Serializable {
	private Referencia ref    = 0;
	private Precio     precio = 0;
	private Estado     estado = Estado.Libre;

	private unique<Buffer> ficha = unique_box(buffer(256));

	public Pieza() => this(0, 0);

	public Pieza(Referencia ref, Precio precio) {
		this.ref    = ref;
		this.precio = precio;
	}

	// R88: formas distintas, pero comparten el ancla =>.
	public get ref          => this.ref;
	public get precio       => this.precio;
	public Precio con_iva() => this.precio + IVA;

	public set precio(Precio v) {
		if (v < 0) throw new FatalError("precio negativo");
		this.precio = v;
	}

	@Override
	public string toString() => "Pieza ${ref} a ${precio}";

	public ~Pieza() { }
}


/// R12: no cabe -> todos los parametros uno por linea, sin coma (R13).
i64 valorar(
	HashMap catalogo,
	borrow<Cesta> cesta,
	borrow_mut<Estado> estado
) {
	// R94: la cadena no cabe -> un eslabon por linea, punto delante.
	i64 bruto = cesta
		.filter(esta_disponible)
		.map(a_precio)
		.sum();

	// R6: un cuerpo de una sentencia que cabe va sin llaves, en linea.
	if (bruto <= 0) return 0;

	i64 total = 0;
	for (i64 p : cesta) total += p;

	// R89: la cadena if/else entera cabe -> se alinea por el (.
	if      (total > 10000) aplicar_descuento(total);
	else if (total > 1000)  aplicar_puntos(total);
	else                    registrar(total);

	return total;
}


/// R51: la clausula where va en su propia linea.
T mayor<T>(T a, T b) where T: Comparable {
	return a > b ? a : b;
}


/// R77: dentro del asm la indentacion es tuya; las columnas, del
/// formateador (R90).  Una etiqueta rompe el bloque de alineacion.
i32 comparar(u8* a, u8* b, i64 n) {
	asm volatile {
		xor r8, r8
	.loop:
		cmp   r8, r9
		jge   .fin
		movzx r10, byte [rdi + r8] // ca
		movzx r11, byte [rdx + r8] // cb
		cmp   r10, r11
		jne   .distinto
		inc   r8
		jmp   .loop
	.fin:
		xor rax, rax
	.distinto:
		mov rax, 1
	}
	return 0;
}


i32 main(string[] args) {
	unique<HashMap> catalogo = unique_box(hashmap(64));
	Cesta           cesta    = arraylist(16);

	match (clasificar(args)) {
		case Some(x) if x > 0 => println("${x} piezas");
		case Some(x) => println("nada que hacer");
		case None => {
			println("catalogo vacio");
			return 1;
		}
	}

	return 0;
}
```

### Lo que demuestra, regla por regla

| Donde mirar | Reglas |
| :--- | :--- |
| Cabecera `/* * */` con los asteriscos en columna | `R21` |
| `import` selectivo repartido uno por linea | `R14`, `R24` |
| `typedef` y `using` como bloques separados | `R87` |
| `const f64 IVA` sin arrastrar a `u32 siguiente_ref` | `R87` |
| Valores del `enum` alineados por el `=` | `R47`, `R84` |
| Campos del `struct` y sus comentarios en columna | `R46`, `R20`, `R88` |
| Campos de la clase alineados en tres columnas | `R84` |
| `get` y `con_iva()` alineados solo por el `=>` | `R88` |
| Parametros de `valorar` uno por linea, sin coma | `R12`, `R13` |
| Cadena `.filter().map().sum()` con el punto delante | `R94`, `R95` |
| `if (bruto <= 0) return 0;` y el `for` sin llaves | `R6` |
| Cadena `if`/`else if`/`else` alineada por el `(` | `R89` |
| `where T: Comparable` en la firma | `R51` |
| `asm`: etiquetas a su nivel, columnas alineadas | `R77`, `R79`, `R90` |
| `match` con guarda y con cuerpo de bloque | `R59`, `R60` |
