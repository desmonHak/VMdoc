# Llamada uniforme y argumentos con nombre

En Vesta, **`x.f(a)` y `f(x, a)` son la misma llamada escrita de dos formas**.
No hay que declarar nada para que la primera exista: cualquier funcion libre
cuyo primer parametro admita el receptor se puede llamar por el punto.

```vesta
i64 doble(i64 x) => x * 2;

i64 a = doble(6);   // 12
i64 b = 6.doble();  // 12 -- la misma llamada
```

Lo que aporta no es poder, es **orden de lectura**. Estas dos calculan lo mismo:

```vesta
i64 r = div(mul(add(2, 4), 10), 5);  // se lee de dentro afuera, y hacia atras
i64 r = 2.add(4).mul(10).div(5);     // se lee en el orden en que ocurre
```

Y con ello desaparece la disyuntiva de siempre: una operacion no tiene que
elegir entre ser metodo (para poder encadenarla) o funcion libre (para no atar
el tipo). Se declara libre y se llama de las dos formas.

> **Es una REESCRITURA, no una resolucion aparte.** El comprobador convierte
> `x.f(a)` en `f(x, a)` y a partir de ahi todo es el camino de siempre: la
> eleccion entre sobrecargas, la comprobacion de argumentos, los prestamos y el
> bajado son literalmente el mismo codigo. Por eso las dos formas no pueden
> divergir, y por eso el JIT y el binario nativo no se enteran de que existe.

---

## 1. Que se puede llamar por el punto

Cualquier receptor: primitivos, punteros, arrays, cadenas, structs y clases.

```vesta
struct Punto { i64 x; i64 y; }

i64 area(Punto p) => p.x * p.y;
i64 largo(string s) => (i64)s.length();

Punto q;
i64 a = q.area();       // area(q)
i64 b = "hola".largo(); // largo("hola")
```

Un literal de cadena es un `ptr` a datos estaticos y solo se **promueve** a
`string` donde hace falta -- por eso `largo("hola")` compila --, asi que
`"hola".largo()` encuentra lo mismo.

### 1.1. Si el tipo YA tiene un metodo con ese nombre

Entonces hay dos candidatos para la misma llamada, y **no se elige en
silencio**: es un error (`VX2068`) que cita los dos.

No es una precaucion teorica. Cual ganase dependeria de los `import` de quien
llama, asi que anadir un import en un fichero cambiaria a que cuerpo va una
llamada que ya estaba escrita y que nadie toco.

### 1.2. Si el receptor no encaja pero la candidata existe

El caso tipico es un **tipo fuerte**: un literal es `i64`, y un tipo fuerte
tiene identidad propia a proposito.

```vesta
typedef u32 Edad new;
u64 anyos(Edad e) => (u64)e + 1;

41.anyos()  // no la encuentra: 41 es i64, no Edad
```

Negarlo a secas dejaria buscando una funcion que esta ahi al lado, asi que el
error dice cual es y con que cast se llega (`VX2071`).

---

## 2. Donde cae el receptor: el hueco `_`

Por defecto el receptor va **delante**: `x.f(a)` es `f(x, a)`. Con `_` va donde
se diga:

```vesta
i64 restar(i64 a, i64 b) => a - b;

i64 h = 10.restar(40, _);  // restar(40, 10) = 30
```

Es lo que permite llamar por el punto a una firma cuyo primer parametro **no es
el sujeto** -- `memcpy(dst, src, n)`, `enmarca(marco, texto)` -- sin retorcer la
firma para acomodar la forma de llamar.

Es una **reordenacion** y ahi acaba: a partir de ese punto todo es posicional
como siempre, asi que la eleccion de sobrecarga es exactamente la misma.

Dos huecos son dos receptores y solo hay uno (`VX2072`); y un `_` en una llamada
libre no tiene receptor que colocar (`VX2073`).

---

## 3. Argumentos con nombre

Un argumento se puede dar por el **nombre de su ranura**, con la misma grafia
que un init designado:

```vesta
i64 mide(i64 ancho, i64 alto) => ancho * alto;

i64 a = mide(3, 2);                  // posicional
i64 b = mide(.alto = 2, .ancho = 3); // por nombre, en cualquier orden
i64 c = mide(.ancho = 3, 2);         // mezclados
```

La regla de colocacion es UNA: **lo nombrado va a su ranura, y lo posicional a
las que queden libres**. De ahi sale sin caso especial que el receptor de una
llamada por el punto caiga en la ranura que queda:

```vesta
i64 d = 3.mide(.alto = 2);  // el receptor cae en `ancho`
```

Errores: una ranura que no existe (`VX2074`), dos argumentos que caen en la
misma (`VX2075`), una que se queda sin argumento (`VX2076`).

> **El nombre de un parametro es parte del contrato.** Renombrarlo rompe a quien
> llama, igual que cambiarle el tipo. Por eso viaja con la firma al `.vxi` y
> sigue valiendo al cruzar el modulo.

---

## 4. Sobrecarga

Dos declaraciones pueden compartir nombre si algo las separa. Lo que puede
separarlas son tres cosas, en este orden:

| las separa | ejemplo |
| :--------- | :------ |
| el numero de argumentos | `mas(a, b)` / `mas(a, b, c)` |
| los tipos | `hace(i64)` / `hace(f64)` |
| **el nombre de las ranuras** | `pesa(kilos, gramos)` / `pesa(libras, onzas)` |

Vale en los seis caminos de llamada -- funcion libre, constructor de struct,
metodo de struct, metodo de clase, `static` y `super` -- y tambien por el punto,
porque el punto reescribe a la forma libre y la eleccion es la misma.

**Gana la exacta antes que la compatible.** Con `hace(i64)` y `hace(f64)`, un
`hace(1_i64)` va al primero aunque el segundo tambien lo aceptaria por
conversion. El tipo del retorno sale de la elegida, y el orden en que se
declaren no cambia nada.

### 4.1. Sobrecargar por el nombre de las ranuras

Dos funciones con el **mismo nombre, el mismo numero de argumentos y los mismos
tipos** son dos funciones distintas si sus ranuras se llaman distinto:

```vesta
i64 pesa(i64 kilos, i64 gramos) => kilos * 1000 + gramos;
i64 pesa(i64 libras, i64 onzas) => libras * 454 + onzas;

i64 a = pesa(.kilos = 2, .gramos = 5);  // 2005
i64 b = pesa(.onzas = 5, .libras = 2);  //  913
```

Declararlas vale siempre. Lo que falla es **la llamada que no las separa**: una
posicional (`pesa(2, 5)`) no dice cual, y se dice cual es la ranura que las
distingue (`VX2077`). Basta con nombrar una.

Vale igual por el punto -- `2.pesa(.onzas = 5)` --, en funciones libres, en
metodos de struct y de clase.

Con `@Override`, en cambio, el nombre de la ranura **identifica**: un metodo que
toma lo mismo pero llama distinto a sus ranuras NO sustituye al de la base,
declara otro. Marcarlo `@Override` es un error que dice como se llaman las
ranuras del que si esta en la jerarquia (`VX2078`).

---

## 5. Cruzando el modulo

Todo lo anterior vale igual sobre una funcion importada: el punto, las ranuras
por nombre, el hueco y la sobrecarga por nombre de ranura.

```vesta
import geo.metrico  only doble, pesa;
import geo.imperial only doble as doble_i, pesa;

i64 a = 6.doble();                   // geo.metrico
i64 b = 6.doble_i();                 // geo.imperial, por su nombre local
i64 c = 2.pesa(.onzas = 5);          // las dos conviven: las separa la ranura
```

Dos homonimas de la misma firma **y la misma ranura** no se pueden importar sin
renombrar: no hay nada que las separe, y elegir por el orden de los `import`
seria decidir por un criterio que no esta escrito en ningun sitio. Se dice, y se
apunta al `import` que lo causa.

### 5.1. El punto resuelve lo que esta EN AMBITO

Esto es lo que mas despista, asi que conviene decirlo entero: **si `f(x)` no
resuelve, `x.f()` tampoco**, porque son la misma llamada. Y lo que pone un
nombre en ambito es el `import`, no que la funcion exista.

```vesta
import geo.metrico;      // el namespace, no sus nombres

6.doble()                // no: `doble` no esta en ambito (`doble(6)` tampoco)
geo.metrico.doble(6)     // si: la llamada cualificada
```

Lo mismo entre dos namespaces del **mismo fichero**: lo que uno declara no esta
en ambito desde otro, se llamen sus ranuras como se llamen. El error dice de
que namespace es y como llamarla (`VX2079` si esta importado por su nombre,
`VX2080` si es del mismo fichero).

### 5.2. Y una GENERICA importada, igual

Una plantilla viaja como fuente y se instancia en quien la usa, asi que al
inyectarla se la renombra a su etiqueta (`std__func__apply`). Eso es interno: se
escribe con su nombre, por el punto o libre, exactamente igual que cualquier
otra importada.

```vesta
import std.func only apply;

u64 n = apply(s, &largo);            // libre
u64 m = s.apply(&largo);             // por el punto: la misma llamada
```

---

## 6. Calificar la llamada por el punto: `$`

Cuando no se quiere traer el nombre al ambito, `$` dice de que namespace es la
funcion, en la llamada misma:

```vesta
i64 a = 6.doble$geo.metrico();            // el namespace entero
i64 b = 6.doble$m();                      // o su alias, tras `import ... as m`
i64 c = 2.pesa$geo.imperial(.onzas = 5);  // compone con las ranuras
string d = "eh".enmarca$geo.formas("<<", _); // y con el hueco
```

**La calificacion va DETRAS del nombre, no delante.** Asi lo que sigue al punto
es siempre la funcion, y eso importa: un campo puede llamarse igual que un
namespace.

```vesta
struct Medida { i64 geo; }

m.geo                    // el campo
m.doble$geo.metrico()    // la funcion del namespace: no hay confusion posible
```

Con la calificacion delante (`m.geo.metrico.doble()`) el mismo texto
significaria una cosa u otra segun los campos del tipo del receptor, que se
declaran en otro fichero: anadir un campo cambiaria en silencio a que cuerpo va
una llamada ya escrita.

`$` se atiende **antes** de buscar un metodo en el receptor -- quien lo escribe
ya dijo cual quiere -- y solo existe tras un receptor: en cabeza ya esta
`geo.metrico.doble(6)`, y dos grafias para la misma llamada serian dos formas
sin criterio para elegir.

Lo que sigue al `$` tiene que ser un namespace que el fichero declare o importe
(`VX2081`).

---

## 7. Elegir la funcion: `std.func.apply`

Lo que sigue al punto tiene que ser un **nombre**, asi que una eleccion no cabe
ahi: `s.(alto ? grita : susurra)()` no existe, y no deberia.

El hueco no esta en el punto sino en el argumento. Una funcion ya es un valor
(`&grita` es un `cfn`), asi que la eleccion se escribe donde se escriben todas
-- en una expresion -- y lo unico que falta es quien la reciba:

```vesta
import std.func only apply, tap;

s.apply(alto ? &grita : &susurra);   // la eleccion, sin sintaxis nueva

u64 n = s.apply(&largo);             // y devuelve lo que devuelva la elegida
5.tap(&nota).apply(&mas_uno);        // `tap` deja pasar el valor: efecto y sigue
```

`apply` liga su retorno con el de la funcion que reciba, **`void` incluido**, asi
que no hace falta una segunda por no devolver nada.

No cuesta nada frente a escribir la llamada a mano: son genericas, cada uso se
monomorfiza a una funcion cuyo cuerpo es la llamada y el inliner se la come; y
cuando la elegida se conoce al compilar, la devirtualizacion convierte ademas la
indirecta en directa.

---

## 8. Formato

El estandar ([EstiloYFormato.md](EstiloYFormato.md)) fija tres cosas:

- `R89`: los argumentos nombrados de una llamada repartida se alinean por su `=`.
- `R94`/`R95`: una cadena de llamadas va entera en una linea o con **todos** sus
  eslabones repartidos, uno por linea.
- `R96b`: la calificacion `$` va pegada por los dos lados.

---

## 9. Donde mirarlo funcionando

| Ejemplo | Que cubre |
| :------ | :-------- |
| `549_ufcs_llamada_uniforme.vx` | receptor struct, primitivo, puntero y clase; sobrecarga entre las candidatas; y `apply`/`tap` de otro modulo con la elegida por un ternario |
| `551_ufcs_encadenado.vx` | encadenar sobre literales, cruzar de familia, el hueco `_` |
| `552_argumentos_nombrados.vx` | ranuras por nombre, mezcladas, y el nombre eligiendo entre hermanas |
| `556_ufcs_xmodulo.vx` | cruzando el modulo: `as`, sobrecarga por ranura, `$` |
| `558_ufcs_ns_fichero.vx` | namespaces del mismo fichero, la cualificada y `$` |
| `550`, `557`, `559` | los negativos: cada error con su codigo |
