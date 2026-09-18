# Parametros: una sola gramatica

**Lo que va entre los parentesis de una declaracion no depende de lo que se
declara.** Una funcion suelta, un metodo, un constructor, un metodo de
interfaz, una funcion `comptime` y una lambda leen sus parametros con la MISMA
gramatica: las doce formas valen en los siete sitios.

Es una promesa del lenguaje, no una coincidencia de la implementacion: no hay
ninguna forma de parametro que valga en una funcion y no en un metodo, ni al
reves. La matriz completa -- las doce formas por los siete contextos -- se
**ejecuta** en `examples_codes_vx/482_matriz_parametros.vx`, porque que compile
no basta: un parametro puede declararse bien y llegar mal.

---

## 1. Las doce formas

```vesta
i64 f_plano(i64 x) => x;                       // por valor
i64 f_puntero(i64* p) => *p;                   // puntero crudo
i64 f_nonulo(i64 !!x) => x;                    // no puede ser nulo
i64 f_cstyle(i64 (*cb)(i64), i64 v) => cb(v);  // puntero a funcion estilo C
i64 f_cfn(cfn(i64) -> i64 cb, i64 v) => cb(v); // puntero a funcion crudo
i64 f_fn(fn(i64) -> i64 cb, i64 v) => cb(v);   // lambda (puede capturar)
i64 f_array(i64[] xs) => xs[0] + xs[1];        // array nativo
i64 f_cadena(string s) => s.length();          // cadena gestionada
i64 f_struct(Punto pt) => pt.x + pt.y;         // struct POR VALOR
i64 f_clase(Caja k) => k.v;                    // referencia de clase
i64 f_registro(register("rax") i64 x) => x;    // ligado a un registro fisico
i64 f_compartido(shared i64 x) => x;           // en el heap compartido
```

Y el **variadico**, que recoge los que sobren:

```vesta
// El variadico llega como un puntero al primero; `vacount()` dice cuantos son.
i64 suma(i64* xs, i64 n) {
	i64 a = 0;
	for (i64 i = 0; i < n; i = i + 1) { a = a + xs[i]; }
	return a;
}

i64 total(i64... xs) => suma(xs, vacount());
```

Vale con parametros fijos delante -- `i64 desde(i64 base, i64... xs)`, donde los
de delante son obligatorios y de ahi en adelante cualquier cantidad -- y en los
siete contextos, no solo en una funcion suelta.

Lo que un variadico implica, y vale igual en los siete contextos: la aridad
declarada es un **minimo**, no un numero exacto; los argumentos de mas se
comprueban contra el tipo del **elemento**; y el array que los recoge lo
construye el sitio de llamada, no quien escribe la funcion.

Se prueba en `examples_codes_vx/481_variadicos_todos.vx`.

Ademas, cada parametro puede llevar su **direccion** -- `in`, `out`, `inout` --,
que es un eje aparte y tiene [su propia pagina](DireccionParametros.md).

---

## 2. Los siete contextos

| Donde | Ejemplo |
| :------------------ | :------ |
| funcion libre | `i64 f(i64 x)` |
| metodo de struct | `struct S { i64 m(i64 x) => x; }` |
| metodo de clase | `class C { public i64 m(i64 x) => x; }` |
| constructor | `class C { public C(i64 x) { ... } }` |
| metodo de interfaz | `interface I { i64 m(i64 x); }` |
| funcion `comptime` | `comptime i64 f(i64 x) => x;` |
| lambda | `(i64 x) => x` |

Todas las casillas de la matriz son SI. Lo unico que cambia son **dos limites**,
y ninguno es de la gramatica.

---

## 3. Limite 1: cuantos caben, y solo en la maquina virtual

En la maquina virtual los argumentos viajan en **doce registros** -- el modelo
mas rapido para un interprete, y por eso se queda; el JIT usa el mismo banco --.

| Declaracion | Tope |
| :--------------------- | :--- |
| funcion libre | 12 |
| metodo o constructor | **11** (el primero lo ocupa `this`) |
| variadico | se lleva 2 (la direccion del array y cuantos son) |
| **binario nativo** | **sin limite**: la convencion del procesador derrama en la pila lo que no cabe |

Pasarse en la maquina virtual es un **error de compilacion**, y el mensaje
apunta al modo nativo, que no tiene el limite.

`examples_codes_vx/483_params_limite_por_modo.vx` usa `@Target` para cubrir las
dos ramas.

---

## 4. Limite 2: que se le puede pasar a una `comptime`

Las doce formas se **declaran** igual en una funcion `comptime`. Lo que cambia
es con que se la puede **llamar**: solo con lo que sea constante al compilar.
Una direccion de funcion o un struct construidos en ejecucion no lo son, por
definicion.

---

## 5. Y lo que un argumento admite es lo mismo en las siete formas de llamar

Dos conversiones que ocurren en el sitio de llamada, y ocurren igual se llame
como se llame (posicional, por nombre, por el punto, con hueco...):

- una **constante que cabe en un newtype** se re-tipa sola;
- el **NOMBRE de una funcion** donde se espera una funcion se convierte en un
  valor-funcion.

Se ejecuta en `examples_codes_vx/489_argumentos_uniformes.vx`.

---

## Ver tambien

- [Direccion de parametros](DireccionParametros.md) -- `in`/`out`/`inout`.
- [Llamada uniforme](LlamadaUniforme.md) -- argumentos por nombre, el hueco `_`
  y la sobrecarga.
- [Closures](Closures.md) -- `fn(...)` frente a `cfn(...)`.
- [Tipos de datos](TiposDatos.md) -- que es cada una de las formas.
