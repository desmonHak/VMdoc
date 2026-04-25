# Macros parametricas

Las **macros parametricas** son macros que aceptan argumentos y generan codigo a partir
de ellos. Son mas potentes que las macros constantes porque pueden transformar codigo,
no solo sustituir valores.

**Analogia:** piensa en una plantilla de carta. La plantilla tiene huecos como
"Estimado [NOMBRE]". Rellenas los huecos con argumentos concretos y obtienes una carta
personalizada. La macro es la plantilla; los argumentos son los valores con los que
rellenas los huecos.

---

## Forma basica

```c
// Macro con un argumento
#define CUADRADO(x) ((x) * (x))

// Uso
uint64_t resultado = CUADRADO(5);  // se expande a: ((5) * (5)) = 25
uint64_t otro = CUADRADO(2 + 3);  // se expande a: ((2 + 3) * (2 + 3)) = 25
```

Nota: los parentesis alrededor de `x` en la definicion son importantes para evitar
errores de precedencia de operadores.

---

## Macro con codigo envuelto: `@wrapped_code`

Las macros parametricas en Vesta permiten recibir un **bloque de codigo** como
argumento usando la anotacion `@wrapped_code`. Esto se usa para construir estructuras
de control personalizadas.

```c
// Definicion de la macro forEach
// - int:x  -> variable que se externaliza (el iterador visible desde fuera)
// - n1, n2 -> argumentos de la macro (inicio y fin del rango)
#parametric int:x forEach(int n1, int n2) for (int x = n1; x < n2; x++) {
    @wrapped_code
}
```

`@wrapped_code` es el bloque de codigo que el usuario escribe entre llaves al llamar
a la macro. El preprocesador lo inserta en ese lugar durante la expansion.

---

## Uso de `forEach`

```c
// Llamada a la macro parametrica
// i es el iterador (variable externalizada int:x)
// El bloque { ... } es el @wrapped_code
forEach(1, 10) { i ->
    print(i)  // i toma los valores 1, 2, ..., 9
}
```

El preprocesador expande esto a:

```c
// Resultado tras la expansion
for (int i = 1; i < 10; i++) {
    print(i)
}
```

El usuario escribe `forEach(1, 10)` y el preprocesador genera el bucle `for` completo.

---

## Ventajas frente a funciones ordinarias

| Caracteristica         | Macro parametrica       | Funcion ordinaria              |
| :--------------------- | :---------------------- | :----------------------------- |
| Evaluacion             | En preprocesamiento     | En tiempo de ejecucion         |
| Puede envolver codigo  | Si (con @wrapped_code)  | No (necesita puntero a funcion)|
| Overhead en runtime    | Ninguno                 | Llamada + retorno              |
| Visible en el debug    | El codigo expandido     | El nombre de la funcion        |

---

## Casos de uso tipicos

```c
// Macro de asercion (para debug)
#define ASSERT(cond) if (!(cond)) { throw new AssertionError(#cond); }

ASSERT(x > 0);  // lanza excepcion si x <= 0

// Macro de intercambio
#define SWAP(T, a, b) { T tmp = a; a = b; b = tmp; }

int x = 5;
int y = 10;
SWAP(int, x, y);  // ahora x=10, y=5

// Macro de rango con paso
#parametric int:i forRange(int start, int end, int step) \
    for (int i = start; i < end; i += step) { @wrapped_code }

forRange(0, 100, 5) { i ->
    print(i)  // 0, 5, 10, ..., 95
}
```

---

Ver tambien:
- [[Macros constantes.md]] - macros de valor fijo
- [[Macros de definicion de sintaxis.md]] - macros que extienden la gramatica
- [[Bucles.md]] - como se usan forEach y forRange en el codigo de usuario
