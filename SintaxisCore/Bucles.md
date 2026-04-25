# Bucles en Vesta

Un **bucle** es una forma de repetir un bloque de codigo varias veces sin tener que
escribirlo manualmente una y otra vez. Por ejemplo, si quieres imprimir los numeros
del 1 al 100, no tienes que escribir 100 instrucciones: usas un bucle.

Vesta ofrece varias formas de bucle para distintas situaciones. Todas ellas se compilan
a instrucciones de bytecode VestaVM: saltos condicionales (`JMP`, `JMPC`), comparaciones
(`CMP`) y tablas de despacho (`JUMPTABLE`). Esto significa que el codigo que escribes
en Vesta de alto nivel es simplemente una forma mas comoda de expresar lo que la VM
ejecuta a nivel de bytecode.

---

## While

El bucle `while` repite un bloque **mientras** una condicion sea verdadera.
Primero comprueba la condicion; si es falsa desde el principio, el cuerpo no se ejecuta.

```c
// Ejemplo: contar de 0 a 9
uint8_t i = 0;
while (i < 10) {
    i++;  // incrementar i en cada vuelta
}
// Al salir, i vale 10
```

**Analogia:** imagina que estas llenando cubos de agua. Compruebas si el deposito
esta lleno (condicion). Si no lo esta, llevas un cubo mas (cuerpo del bucle). Si ya
esta lleno, paras.

---

## Do-While

El bucle `do-while` ejecuta el cuerpo **al menos una vez**, y luego comprueba la
condicion para decidir si repite.

```c
// Forma 1: do { ... } while (cond)
uint8_t i = 0;
do {
    i++;  // se ejecuta siempre al menos una vez
} while (i < 10);

// Forma 2: while (cond) do { ... }
// (sintaxis alternativa valida en Vesta, mismo significado)
while (i < 10) do {
    i++;
}
```

**Diferencia clave con `while`:** si la condicion ya es falsa al principio,
`while` no ejecuta el cuerpo; `do-while` lo ejecuta igualmente una vez.

---

## Loop (bucle infinito)

`loop` crea un bucle que repite para siempre. Se usa cuando no sabes de antemano
cuantas veces quieres iterar, o cuando el bucle debe correr hasta que algo externo
(un `break` o una excepcion) lo detenga.

```c
loop {
    // este bloque se ejecuta infinitamente
    // hasta que se encuentre un break o se lance una excepcion
    if (condicion_de_parada) {
        break;  // salir del bucle
    }
}
```

**Caso tipico:** servidores, bucles de eventos, procesadores de mensajes.

---

## For

El bucle `for` es la forma mas compacta de escribir un bucle con:
- una variable de control (inicializacion)
- una condicion de parada
- una actualizacion al final de cada vuelta

### Forma clasica (C-style)

```c
// for (init; condicion; actualizacion)
for (uint8_t i = 0; i < 10; i++) {
    // i toma los valores 0, 1, 2, ..., 9
}
```

### Forma for-in (iteracion sobre un rango)

```c
// Iterar sobre el rango [0, 10)
// range es una macro que genera los valores de 0 a 9
for (uint8_t i : range(0, 10)) {
    // i toma los valores 0, 1, 2, ..., 9
}
```

`range(inicio, fin)` es una macro del preprocesador que genera un rango de enteros.
No crea un array en memoria - el compilador lo transforma en un bucle ordinario.

### Forma forEach (macro parametrica)

```c
// forEach(inicio, fin) { variable ->
//     cuerpo
// }
forEach(1, 10) { i ->
    print(i)  // i es el iterador; imprime 1, 2, ..., 10
}
```

`forEach` es una macro parametrica del preprocesador. Recibe el rango y el cuerpo
del bucle como argumentos. Es equivalente al `for-in` pero con sintaxis de flecha
para el iterador.

---

## Switch-Case

`switch` evalua una expresion y salta al caso que coincide con el valor.
A nivel de bytecode, el compilador puede usar `JUMPTABLE` (O(1)) o `TYPESWITCH`
(O(n) por tipo) segun el tipo de la expresion.

```c
// Comparar texto con varios casos posibles
String text = "Hola mundo";

switch (text) {

    // Caso simple: requiere break para no "caer" al siguiente caso
    case "texto1":
        /* accion para texto1 */
        break;

    // Caso con llaves: el break es implicito al cerrar las llaves
    case "texto2": {
        /* accion para texto2 */
        // no hace falta break; las llaves delimitan el bloque
    }

    // Multiples etiquetas para el mismo cuerpo
    case "texto3":
    case "texto4": {
        /* se ejecuta si el valor es "texto3" O "texto4" */
    }

    // Lista de valores en un solo caso (sintaxis alternativa)
    case [
        "texto5",
        "texto6",
        "texto7"
    ]: {
        // se ejecuta si el valor es cualquiera de los tres
    }

    // Caso por defecto (equivalente a "else"): sintaxis flecha o llaves
    _ => /* accion por defecto si ningun caso coincidio */

}
```

### Notas sobre switch

| Sintaxis         | Break implicito | Descripcion                                 |
| :--------------- | :-------------: | :------------------------------------------ |
| `case X: stmt`   | No              | Necesita `break` manual                     |
| `case X: { }`   | Si              | Las llaves delimitan el bloque; break implicito |
| `case [X,Y]: {}` | Si              | Lista de valores equivalentes               |
| `_ =>`           | Si              | Caso por defecto (default)                  |

---

## Tabla comparativa de bucles

| Bucle    | Comprueba condicion | Garantiza al menos una ejecucion | Usa cuando...                              |
| :------- | :-----------------: | :------------------------------: | :----------------------------------------- |
| `while`  | Al inicio           | No                               | No sabes cuantas vueltas dara              |
| `do-while` | Al final          | Si                               | El cuerpo debe ejecutarse al menos una vez |
| `loop`   | Nunca (infinito)    | Si                               | Necesitas un bucle eterno con break        |
| `for`    | Al inicio           | No                               | Sabes exactamente el rango de iteracion    |
| `forEach`| Al inicio (macro)   | No                               | Iteracion funcional sobre un rango         |

---

## Control de flujo dentro de bucles

```c
// break: sale del bucle inmediatamente
while (true) {
    if (condicion) break;  // salir del bucle
}

// continue: salta al inicio de la siguiente vuelta
for (uint8_t i = 0; i < 10; i++) {
    if (i == 5) continue;  // salta i=5, continua con i=6
    print(i);
}
```

---

## Relacion con el bytecode VestaVM

Los bucles de alto nivel se compilan a instrucciones de bajo nivel de la VM:

```c
// Este bucle de alto nivel...
for (uint8_t i = 0; i < 10; i++) { }

// ...se convierte en bytecode similar a esto:
// mov r1, 0          // i = 0
// inicio_bucle:
//   cmp r1, 10       // comparar i con 10
//   jmp.jge fin      // si i >= 10, salir
//   add r1, 1        // i++
//   jmp inicio_bucle // volver al inicio
// fin_bucle:
```

Para `switch` con valores enteros contiguos, el compilador puede usar `JUMPTABLE`
(salto en O(1) via tabla de punteros) en lugar de comparar caso por caso.

---

Ver tambien:
- [[Condicionales.md]] - if/else y expresiones condicionales
- [[SetInstruccionesVM/JUMPTABLE_TYPESWITCH.md]] - instrucciones de despacho O(1)/O(n)
- [[Preprocesador/Macros parametricas.md]] - como funciona `forEach` como macro
