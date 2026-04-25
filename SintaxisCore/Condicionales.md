# Condicionales en Vesta

Un **condicional** permite que el programa tome decisiones: ejecutar un bloque de codigo
solo si cierta condicion es verdadera, o elegir entre dos caminos distintos.

A nivel de bytecode VestaVM, cada condicional se traduce en una instruccion `CMP`
seguida de un salto condicional `JMPC` (o su forma corta `jmp.jXX`). El compilador
hace esa traduccion de forma transparente.

---

## if / else if / else

La forma mas basica de tomar una decision:

```c
uint8_t temperatura = 35;

if (temperatura > 30) {
    // se ejecuta si temperatura > 30
    print("Hace calor");
} else if (temperatura > 20) {
    // se ejecuta si temperatura esta entre 20 y 30
    print("Temperatura agradable");
} else {
    // se ejecuta si temperatura es <= 20
    print("Hace frio");
}
```

**Reglas:**
- Las llaves `{}` son obligatorias aunque el cuerpo sea de una sola linea.
- `else if` permite encadenar tantas condiciones como se necesite.
- `else` es opcional; si no aparece y ninguna condicion es verdadera, el programa
  simplemente continua.

---

## Expresion condicional (operador ternario)

Cuando el resultado es un valor, se puede usar la forma compacta:

```c
uint8_t valor = 5;

// condicion ? valor_si_verdadero : valor_si_falso
String msg = (valor > 0) ? "positivo" : "no positivo";
```

Equivalente a:

```c
String msg;
if (valor > 0) {
    msg = "positivo";
} else {
    msg = "no positivo";
}
```

---

## Condicional sin salto: MOVC

VestaVM tiene una instruccion especial `MOVC` (Move Conditional) que selecciona entre
dos valores SIN hacer un salto. Esto es mas eficiente en procesadores modernos porque
evita errores de prediccion de rama (branch misprediction).

En Vesta de alto nivel, el compilador puede generar `MOVC` automaticamente en casos
simples donde solo se elige entre dos valores:

```c
// El compilador puede optimizar esto a un MOVC:
int32_t x = (a > b) ? a : b;  // max(a, b) sin salto
```

---

## Operadores de comparacion

| Operador | Significado           | Ejemplo     |
| :------: | :-------------------- | :---------- |
| `==`     | Igual a               | `x == 5`    |
| `!=`     | Distinto de           | `x != 0`    |
| `<`      | Menor que             | `x < 10`    |
| `<=`     | Menor o igual que     | `x <= 10`   |
| `>`      | Mayor que             | `x > 0`     |
| `>=`     | Mayor o igual que     | `x >= 0`    |

---

## Operadores logicos

Se pueden combinar varias condiciones:

```c
bool tiene_edad = (edad >= 18);
bool tiene_carnet = (carnet == true);

// AND: ambas condiciones deben ser verdaderas
if (tiene_edad && tiene_carnet) {
    print("Puede conducir");
}

// OR: basta con que una sea verdadera
if (tiene_edad || tiene_carnet) {
    print("Tiene al menos uno de los requisitos");
}

// NOT: invierte la condicion
if (!tiene_carnet) {
    print("No tiene carnet");
}
```

| Operador | Nombre | Descripcion                                    |
| :------: | :----- | :--------------------------------------------- |
| `&&`     | AND    | Verdadero si AMBAS condiciones son verdaderas  |
| `\|\|`   | OR     | Verdadero si AL MENOS UNA es verdadera         |
| `!`      | NOT    | Invierte: verdadero -> falso, falso -> verdadero |

---

## Condicional nulo: isnull / unwrap

Para valores que pueden ser nulos (punteros, referencias a objetos), Vesta ofrece
dos instrucciones especializadas que se pueden invocar tambien desde el nivel de
bytecode:

```c
// Comprobar si un objeto es nulo sin excepcion
if (isnull(objeto)) {
    // objeto es null
}

// Obtener el valor asegurandonos de que no es nulo
// (lanza NullPointerException si es null)
MiClase ref = unwrap(puntero);
```

Estas se traducen directamente a las instrucciones `ISNULL` y `UNWRAP` del bytecode.
Ver [[NULLABLE.md]] para la referencia completa.

---

## Relacion con el bytecode VestaVM

```c
// Este if de alto nivel...
if (x > 5) {
    print("mayor");
}

// ...se compila a algo como:
// cmp  r1, 5         // comparar x con 5
// jmp.jle fin        // si x <= 5, saltar al fin
//   ; cuerpo del if (llamada a print)
// fin:
```

---

Ver tambien:
- [[Bucles.md]] - repeticion con while, for, loop
- [[SetInstruccionesVM/NULLABLE.md]] - isnull y unwrap en detalle
- [[SetInstruccionesVM/MOV, MOVH, MOVC, MOCH.md]] - instruccion MOVC sin salto
