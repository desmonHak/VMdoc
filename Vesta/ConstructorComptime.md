# Constructor comptime

Un tipo puede declarar un constructor que se ejecuta **en tiempo de
compilacion** y recibe la expresion **sin evaluar**:

```vx
import std.comptime.literal only IntLit, parse_int_lit;

/** @brief Entero de 128 bits guardado en dos palabras de 64. */
struct U128 {
    u64 lo;
    u64 hi;

    /**
     * @brief Construye el valor a partir de como se escribio.
     * @param e Los digitos, tal cual, sin interpretar.
     */
    public comptime U128(expr e) {
        IntLit v = parse_int_lit(e);
        this.lo = v.word(0);
        this.hi = v.word(1);
    }
}

i32 main() {
    // 2^128 - 1: los 128 bits a uno.  No cabe en ningun tipo nativo.
    U128 m = U128(340282366920938463463374607431768211455);
    println("lo=${m.lo} hi=${m.hi}");   // ambos a todo unos
    return 0;
}
```

El parseo ocurre al compilar: en el binario solo quedan las dos constantes.

## Cuando entra

Al escribir `T(...)` se prueban primero **todas las sobrecargas normales** del
constructor.  El constructor comptime entra **solo si ninguna encaja**.

Esto es deliberado: no compite con las sobrecargas ni las ensombrece.  Actua
cuando el lenguaje no sabe construir el valor por sus medios, y entonces el
tipo decide que hacer con lo que se escribio.

El caso que lo motiva son los valores mas anchos que la palabra.  Un numero de
128 bits no cabe en ningun tipo nativo, asi que ninguna sobrecarga puede
aceptarlo; antes eso era el final del camino (`funcion no declarada`).

## Por que `expr`

El parametro se declara `expr`, no `string` ni un tipo concreto.  Con `expr` el
compilador entrega el **texto** de lo que se escribio, sin interpretarlo.  Esto
importa: si intentara evaluarlo primero, un literal de 128 bits se truncaria
antes de llegar al constructor, que es justo lo que se quiere evitar.

Y como lo que llega es texto, sirve para cualquier expresion, no solo para
numeros: el tipo puede interpretarla como le convenga.

## Que hace el cuerpo

Dos formas, segun se marque o no como `@Macro`:

- **Sin `@Macro`**: rellena `this`, igual que un constructor normal, pero
  ejecutandose en tiempo de compilacion.  El resultado se materializa donde
  estaba la llamada, como si el valor se hubiera escrito a mano con la sintaxis
  de siempre.  En el binario solo quedan las constantes.
- **Con `@Macro`**: devuelve **codigo** Vesta, que se inyecta en el sitio de la
  llamada.  Util cuando construir el valor requiere generar una expresion en
  lugar de calcular unos campos.

## Coste

Ninguno en ejecucion.  Todo ocurre al compilar; el programa final contiene el
valor ya construido.

## Ejemplo

`examples_codes_vx/361_ctor_comptime.vx`.

