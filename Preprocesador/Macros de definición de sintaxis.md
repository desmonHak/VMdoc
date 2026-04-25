# Macros de definicion de sintaxis

Las **macros de definicion de sintaxis** permiten extender la gramatica del lenguaje
Vesta anadiendo nuevas construcciones sintaticas directamente en el codigo fuente.
Usan notacion EBNF para describir la nueva sintaxis.

**Analogia:** es como anadir palabras nuevas a un diccionario. En lugar de conformarte
con las palabras que ya existen (construcciones del lenguaje), puedes inventar nuevas
palabras con su definicion y reglas de uso.

---

## Notacion EBNF en Vesta

Vesta usa EBNF (Extended Backus-Naur Form) para describir reglas gramaticales.
Ver [[EBNF/Que es EBNF.md]] para una introduccion completa.

Una regla EBNF basica:

```c
// Regla: una expresion es un termino, o una expresion mas un termino, o ...
<expresion> ::= <termino>
              | <expresion> '+' <termino>
              | <expresion> '-' <termino>

<termino> ::= <factor>
            | <termino> '*' <factor>
            | <termino> '/' <factor>

<factor> ::= <numero>
           | '(' <expresion> ')'

<numero> ::= <digito>
           | <numero> <digito>

<digito> ::= '0' | '1' | '2' | '3' | '4' | '5' | '6' | '7' | '8' | '9'
```

---

## Reglas EBNF para un `if` tipico

```c
<if_stmt>   ::= 'if' '(' <condicion> ')' '{' <sentencia> '}' [ 'else' <sentencia> ]
<sentencia> ::= <if_stmt> | <asignacion> | ';'
```

Los corchetes `[ ]` indican opcionalidad: la clausula `else` puede aparecer o no.

---

## Gramaticas de atributos: de EBNF a bytecode

EBNF sola describe solo la estructura. Para generar bytecode VestaVM, cada regla
lleva una **accion semantica** entre llaves `{ }` que especifica que codigo generar:

```c
<programa> ::= <decls> <stmts> {
    programa.bytecode = decls.bytecode + stmts.bytecode;
}

<stmt> ::= <id> "=" <exp> ';' {
    stmt.bytecode = id.bytecode + exp.bytecode + "putstatic " + id.text;
}

<exp> ::= <exp> "+" <term> {
    exp.code = exp1.code + term.code + "iadd";
    exp.type = "int";
}

<term> ::= numero {
    term.code = "ldc " + numero.lexeme;
    term.type = "int";
}
```

Las acciones semanticas calculan **atributos**:
- **Sintetizados**: calculados de los hijos hacia arriba en el arbol (p.ej. `exp.code`).
- **Heredados**: pasan informacion del padre o hermanos hacia abajo (p.ej. el tipo esperado).

---

## Bytecode intermedio (DAG)

Para soportar multiples arquitecturas (x86, ARM, RISC-V), Vesta genera un
**bytecode intermedio de alto nivel** parecido a LLVM IR, que luego el backend
traduce a instrucciones nativas:

```c
// Bytecode intermedio generado para: x = 5 + 3
TEMP t1 = 5 + 3
x = t1
```

El compilador construye un **DAG** (Grafo Aciclico Dirigido) de dependencias para
determinar el orden correcto de emision de instrucciones:

```c
// Dependencias:
// CONST_5 -> ADD_t1 -> ASSIGN_x
// CONST_3 -> ADD_t1

// Orden BFS garantizado:
// CONST_5, CONST_3, ADD_t1, ASSIGN_x
```

Luego el backend emite codigo nativo para la arquitectura objetivo:

```c
// Para x86-64:
mov eax, 5    // CONST_5
mov ebx, 3    // CONST_3
add eax, ebx  // ADD: t1 = 5 + 3
mov [x], eax  // ASSIGN: x = t1
```

---

## Como anadir una nueva construccion sintatica

Para anadir, por ejemplo, una construccion `unless (cond) { }` (if negado):

```c
// 1. Definir la regla EBNF
#syntax unless_stmt ::= 'unless' '(' <condicion> ')' '{' <sentencia> '}' {
    // 2. Semantica: generar bytecode equivalente a if (!cond) { ... }
    unless_stmt.bytecode = negate(cond.bytecode) + sentencia.bytecode;
}

// 3. Uso en el codigo
uint8_t x = 5;
unless (x == 0) {
    print("x no es cero");  // solo se ejecuta si x != 0
}
```

El preprocesador expande `unless` a un `if` con la condicion negada.

---

## Relacion con el sistema de bytecode VestaVM

Las macros de definicion de sintaxis no cambian el bytecode VestaVM: solo cambian
la forma en que el compilador acepta y transforma el codigo fuente. El bytecode
generado es identico al que se generaria con las construcciones estandar del lenguaje.

---

Ver tambien:
- [[EBNF/Que es EBNF.md]] - introduccion a la notacion EBNF
- [[EBNF/Reglas de sintaxis.md]] - ejemplos detallados de reglas con atributos
- [[Macros parametricas.md]] - macros con argumentos para codigo de usuario
- [[../SintaxisCore/Bucles.md]] - construcciones de bucle estandar
