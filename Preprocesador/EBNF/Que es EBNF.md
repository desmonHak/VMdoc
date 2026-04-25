# Que es EBNF

**EBNF** (Extended Backus-Naur Form) es una notacion formal para describir la
**gramatica** de un lenguaje: las reglas que determinan que secuencias de simbolos
son validas en ese lenguaje.

**Analogia:** las reglas gramaticales del espanol dicen que "El gato come" es una
oracion valida, pero "Gato el come" no lo es. EBNF hace lo mismo para lenguajes de
programacion: describe que secuencias de tokens son programas validos.

La formula es: `EBNF = BNF + REGEX`

- **BNF** (Backus-Naur Form): reglas de produccion para describir gramaticas libres
  de contexto (igual que las gramaticas de los lenguajes naturales).
- **REGEX** (Expresiones regulares): para describir patrones de texto simples como
  "uno o mas digitos" o "letra seguida de letras y numeros".

---

## Para que sirve EBNF en VestaVM

El preprocesador de Vesta usa EBNF para dos propositos:

1. **Describir la gramatica de Vesta**: documentar formalmente que construcciones
   son validas en el lenguaje.

2. **Extender la gramatica**: con las macros de definicion de sintaxis, el usuario
   puede anadir nuevas construcciones al lenguaje usando notacion EBNF.

---

## Notacion EBNF basica

| Simbolo    | Significado                                   | Ejemplo                       |
| :--------- | :-------------------------------------------- | :---------------------------- |
| `::=`      | Se define como                                | `A ::= B C`                   |
| `\|`       | Alternativa (o bien ... o bien ...)           | `A ::= B \| C`                |
| `[ ]`      | Opcional (cero o una vez)                     | `A ::= B [C]`                 |
| `{ }`      | Repeticion (cero o mas veces)                 | `A ::= B {C}`                 |
| `( )`      | Agrupacion                                    | `A ::= (B \| C) D`            |
| `' '`      | Terminal literal (texto exacto)               | `A ::= 'if' B`                |
| `[a-z]`    | Rango de caracteres (notacion regex)          | `letra ::= [a-z]`             |
| `[0-9]+`   | Uno o mas digitos                             | `digito ::= [0-9]+`           |
| `[0-9]*`   | Cero o mas digitos                            |                               |
| `?`        | Vacio (epsilon, la cadena vacia)              | Caso base en recursion        |

---

## Ejemplos de reglas EBNF

```c
// Identificador: empieza por minuscula, seguido de letras/numeros
Identificador ::= [a-z][0-9a-zA-Z]*

// Entero: signo opcional + uno o mas digitos
Entero ::= [-+]?[0-9]+

// Numero flotante: signo + parte entera opcional + punto + decimales
Flotante ::= [-+]?([0-9]+)?\.[0-9]+
```

---

## Por que EBNF genera bytecode

EBNF sola solo describe **sintaxis** (que secuencias son validas). Para generar
bytecode es necesario anadir **gramaticas de atributos**: reglas semanticas que
calculan el codigo correspondiente a cada construccion sintatica.

Ver [[Reglas de sintaxis.md]] para la explicacion completa con ejemplos de como
EBNF + gramaticas de atributos generan bytecode VestaVM.

---

Ver tambien:
- [[Reglas de sintaxis.md]] - ejemplos concretos de reglas EBNF con generacion de bytecode
- [[../Macros de definicion de sintaxis.md]] - como anadir nuevas reglas a la gramatica de Vesta
