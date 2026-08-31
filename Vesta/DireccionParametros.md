# `in` / `out` / `inout`: la direccion de un parametro

Una marca de direccion dice **quien lee y quien escribe** lo que se pasa.  No es
documentacion: el compilador la comprueba, y donde promete algo que se puede
aprovechar, lo aprovecha.

```vx
i64  suma(in i64* datos, i64 n);   // por ahi SOLO se lee
void parse(string s, out i64 r);   // r se escribe: el llamante cede un hueco
void bump(inout i64 contador);     // se lee y se escribe
```

Las tres palabras valen en los **siete** contextos donde se declaran
parametros -- funcion suelta, metodo, constructor, metodo de interfaz, funcion
`comptime`, lambda y `extern` --, y ademas en variables y en campos.

---

## Indice

- [1. Por que existe](#1-por-que-existe)
- [2. Sobre un VALOR: el parametro de salida](#2-sobre-un-valor-el-parametro-de-salida)
- [3. Sobre algo que APUNTA: permiso de lectura y escritura](#3-sobre-algo-que-apunta-permiso-de-lectura-y-escritura)
- [4. `const` es OTRO eje, y componen](#4-const-es-otro-eje-y-componen)
- [5. Es la misma regla que el borrow checker](#5-es-la-misma-regla-que-el-borrow-checker)
- [6. `out` se comprueba en TODOS los caminos](#6-out-se-comprueba-en-todos-los-caminos)
- [7. La direccion forma parte del TIPO de una funcion](#7-la-direccion-forma-parte-del-tipo-de-una-funcion)
- [8. En un `extern`, la marca DEFINE](#8-en-un-extern-la-marca-define)
- [9. Errores](#9-errores)

---

## 1. Por que existe

Sin la marca, una firma no distingue tres cosas que no se parecen en nada:

```vx
void f(i64* p);   // lee?  escribe?  las dos?  se queda el puntero?
```

Quien la llama tiene que leerse el cuerpo, y el compilador tiene que suponer lo
peor.  Con la marca, las dos preguntas tienen respuesta en la propia firma.

Ni `out` ni `inout` son palabras **reservadas**, y no pueden serlo: `out` es una
instruccion de x86 dentro de un bloque `asm`, y se usa como nombre de variable y
de parametro.  Se reconocen por la POSICION -- al principio de una declaracion y
seguidas de un tipo --, asi que esto sigue siendo valido:

```vx
void set_to(i32* out, i32 v) { *out = v; }   // `out` es solo un nombre
asm { out 0xE9, al }                          // y aqui es una instruccion
```

## 2. Sobre un VALOR: el parametro de salida

Sobre algo que no apunta a nada, `out` e `inout` son el **parametro de salida**
de C# o Ada: quien llama cede un hueco y el llamado lo escribe.

```vx
void dividir(i64 a, i64 b, out i64 cociente, out i64 resto) {
	cociente = a / b;
	resto    = a % b;
}

i64 main() {
	i64 c = 0;
	i64 r = 0;
	dividir(17, 5, c, r);   // sin `&`: lo pone el compilador
	return c + r;           // 3 + 2 = 5
}
```

Lo que viaja es la **direccion** del hueco -- escribir una copia no se veria
desde fuera --, asi que la firma convierte el parametro a `T*`.  El cuerpo lo
sigue viendo como una `T`, y el sitio de llamada toma la direccion solo.  Por
eso el argumento tiene que ser un **sitio donde escribir**: `dividir(17, 5, 3,
r)` no compila, porque el `3` no tiene donde recibir nada.

`in` sobre un valor es una copia de **solo lectura**: se cumple con el `const`
que el lenguaje ya tiene, no con un segundo mecanismo.

## 3. Sobre algo que APUNTA: permiso de lectura y escritura

Sobre un puntero o un array la marca dice que se puede hacer **por ahi**:

```vx
i64  suma(in i64* datos, i64 n);      // por ahi solo se lee
void llenar(out i64* destino, i64 n); // por ahi solo se escribe
void mezclar(inout i64* buf, i64 n);  // las dos cosas
```

`in T*` marca `const` lo **apuntado**, no el puntero: el puntero se puede
reasignar, lo de dentro no se puede tocar.

## 4. `const` es OTRO eje, y componen

La direccion y `const` responden a preguntas distintas, asi que se combinan y
cada nivel es independiente -- igual que en C, donde `const T*` y `T* const` no
son lo mismo:

```vx
in  i64 const* p;   // por ahi se lee, y lo apuntado es constante
out i64 const** q;  // se escribe el puntero de fuera; lo de dentro es constante
```

Prometer con la direccion algo que `const` prohibe es un error, no una
preferencia silenciosa: `out i64 const* p` dice "por aqui escribo" sobre algo
declarado constante, y se rechaza.

## 5. Es la misma regla que el borrow checker

`in` e `inout` **no son otro sistema**: son la misma pregunta que `borrow<T>` y
`borrow_mut<T>` con otra notacion, y por eso las cuatro reglas (R1-R4) valen
tal cual sobre los argumentos de una llamada.

```vx
void f(inout i64 a, inout i64 b);

i64 main() {
	i64 x = 1;
	f(x, x);   // ERROR: dos prestamos exclusivos del MISMO dueno
	return x;
}
```

No hizo falta ninguna regla de aliasing nueva: `inout` presta en exclusiva, `in`
presta compartido, y de ahi salen las combinaciones legales.

## 6. `out` se comprueba en TODOS los caminos

Un `out` que se escribe en una rama y en otra no, no cumple lo que promete.  La
comprobacion no es "se escribe alguna vez", es **en todos los caminos**:

```vx
void f(out i64 x, bool c) {
	if (c) {
		x = 1;
	}
}   // ERROR: por el camino en que `c` es falso, `x` se queda sin escribir
```

Lo contesta un hecho del ASA (`DefiniteStoreFacts`) calculado sobre el IR, que
es quien ya tiene el grafo de flujo: un segundo recorrido sobre el AST habria
sido producir dos veces lo mismo, y el segundo se queda atras.  El hecho es
perezoso y queda cacheado, y su respuesta tiene tres valores -- se escribe
siempre / falta por algun camino / no se sabe --, con el motivo cuando no se
sabe.

## 7. La direccion forma parte del TIPO de una funcion

Lo que el tipo de un puntero a funcion dice es **todo lo que quien llama por el
va a saber**: no hay firma que consultar.  Asi que la marca viaja en el tipo.

```vx
void escribe(out i64 r) { r = 7; }

cfn(out i64) -> void ok = &escribe;   // el tipo lleva la marca
i64 a = 0;
ok(a);                                 // el `&` lo pone el compilador
```

Dos tipos que difieran en las marcas son **incompatibles**, igual que dos que
difieran en la ABI:

```vx
cfn(i64*) -> void perdida = &escribe;   // ERROR
```

Sin eso, la marca se borraba justo al guardar la funcion en una variable, y con
ella se iban las dos cosas que da: el `&` automatico y las reglas de prestamo.

La salida es un **cast**, y vale en las **dos direcciones**:

```vx
// Quitar la marca: la renuncia queda escrita en el fuente.
cfn(i64*) -> void crudo = (cfn(i64*) -> void) &escribe;
crudo(&a);

// Ponerla: un cfn sin restricciones pasa a tenerlas.
void por_puntero(i64* p) { *p = 9; }
cfn(out i64) -> void marcado = (cfn(out i64) -> void) &por_puntero;
marcado(b);
```

Y lo que **no** cambia es como VIAJA el valor: un `out T` se convierte a `T*`,
que es lo que la otra forma ya es.  Los dos tipos llaman igual; lo unico
distinto es la promesa.  Esa es la diferencia con la ABI por registro, donde el
tipo si lleva por donde entra cada argumento -- ahi el cast cambiaria la
llamada, aqui solo cambia lo que se comprueba.

## 8. En un `extern`, la marca DEFINE

Dentro de Vesta la marca es un **contrato que se comprueba** contra el cuerpo.
En un `extern` no hay cuerpo que mirar, asi que ahi la marca **define**: es la
descripcion que el usuario da de una funcion que el compilador no puede
analizar, y de ella salen los efectos.

```vx
extern "kernel32.dll" {
	fn ReadFile(u64 h, out u8* buf, u32 n, out u32* leidos, u64 ov) -> i32;
}
```

Describir la llamada del todo -- en vez de suponer lo peor -- es lo que permite
optimizar alrededor de ella.

## 9. Errores

| Codigo | Que dice |
| :----- | :------- |
| VXT006 | La direccion promete escribir sobre algo declarado `const` |
| VXT009 | `out`/`inout` sobre una variable o un campo: no hay llamante que ceda un hueco |
| VXT010 | La marca en el tipo de RETORNO, donde no significa nada |
| VXT011 | El argumento de un `out` no es un sitio donde escribir |
| VXT012 | Un `out` que no se escribe |

---

## Ejemplos

| Fichero | Que cubre |
| :------ | :-------- |
| `521_direccion_parametros.vx` | Los siete contextos de declaracion |
| `522_efectos_externas.vx` | La marca en un `extern`, con `when:` |
| `523_direccion_campos.vx` | En campos de struct, clase y overlay |
| `524_parametros_de_salida.vx` | El parametro de salida, a fondo |
| `525_direccion_y_borrow.vx` | La misma regla que el borrow checker |
| `526_escritura_por_puntero_indirecto.vx` | Lo que escribe una funcion llamada por puntero |
| `527_direccion_en_el_tipo.vx` | La direccion dentro del tipo, y el cast en las dos direcciones |
| `528_direccion_tipo_err.vx` | Negativo: perder o inventar la marca es un error |
