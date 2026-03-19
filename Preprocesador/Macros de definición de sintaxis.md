Estas notaciones se basan en [EBNF](https://es.wikipedia.org/wiki/Notaci%C3%B3n_de_Backus-Naur)

un ejemplo:
```dart
<expresión> ::= <término> | <expresión> '+' <término> | <expresión> '-' <término>
<término>   ::= <factor>  | <término> '*' <factor> | <término> '/' <factor>
<factor>    ::= <número>  | '(' <expresión> ')'
<número>    ::= <dígito>  | <número> <dígito>
<dígito>    ::= '0' | '1' | '2' | '3' | '4' | '5' | '6' | '7' | '8' | '9'
```

Para un `if` típico en programación:
```dart
<if_stmt>   ::= 'if' '(' <condición> ')' '{' <sentencia> '}' [ 'else' <sentencia> ]
<sentencia> ::= <if_stmt> | <asignación> | ';'
```
Los corchetes `[]` indican opcionalidad (extensión común del BNF básico). [doc](https://www.fing.edu.uy/inco/cursos/teoleng/obligatorio/presentacion.pdf)

----
El [EBNF](https://es.wikipedia.org/wiki/Notaci%C3%B3n_de_Backus-Naur) define solo la sintaxis, no el ``bytecode`` directamente, pero se extiende con **gramáticas de atributos** para asociar reglas semánticas que generan código intermedio como ``bytecode`` durante el análisis.  
Estas reglas calculan atributos (sintetizados o heredados) que representan fragmentos de ``bytecode``, evaluados en el árbol sintáctico

## Gramáticas de Atributos
Agrega atributos a símbolos no terminales y reglas semánticas en llaves `{ }`:
- **Sintetizados**: Calculados de hijos (ascenden el árbol), ideales para bytecode de subexpresiones.
- **Heredados**: De padre/hermanos (descienden), para contexto como tipos.[](https://www.studysmarter.es/resumenes/ciencias-de-la-computacion/teoria-de-la-computacion/forma-de-backus-naur/)​  
    Ejemplo en tabla:

|Regla Gramatical|Regla Semántica|
|---|---|
|`<exp> ::= <exp> '+' <term>`|`{ exp.code = exp1.code; exp.code += LOAD + term.code + ADD; }` [scribd](https://es.scribd.com/document/510592327/Traductor-dirigido-por-la-sintaxis-Recuperado-automaticamente)​|
|`<exp> ::= <term>`|`{ exp.code = term.code; }`|

----
 ## Ejemplo
 ```
 <programa> ::= <decls> <stmts> { programa.bytecode = decls.bytecode + stmts.bytecode }

<decls> ::= <decl> <decls> { decls.bytecode = decl.bytecode + decls.bytecode }
          | ε { decls.bytecode = "" }

<decl> ::= "int" <id> ';' { decl.bytecode = "field public static " + id.text + " I" }

<stmts> ::= <stmt> <stmts> { stmts.bytecode = stmt.bytecode + stmts.bytecode }
          | ε { stmts.bytecode = "" }

<stmt> ::= <id> "=" <exp> ';' { stmt.bytecode = id.bytecode + exp.bytecode + "putstatic " + id.text + "/I" }

<id> ::= letra { id.text = letra.lexeme; id.bytecode = "getstatic " + id.text + "/I" }

<exp> ::= <exp> "+" <term> { exp.code = exp1.code || term.code || "iadd"; exp.type = "int" }
        | <term> { exp.code = term.code; exp.type = term.type }

<term> ::= número { term.code = "ldc " + número.lexeme; term.type = "int" }
         | <id> { term.code = id.code; term.type = "int" }

 ```
## Explicación Línea por Línea

**`<programa> ::= <decls> <stmts> { programa.bytecode = decls.bytecode + stmts.bytecode }`**
- El programa completo concatena bytecode de declaraciones + statements.
- Se ejecuta DESPUÉS de procesar completamente `<decls>` y `<stmts>`.


**`<decls> ::= <decl> <decls> { decls.bytecode = decl.bytecode + decls.bytecode }`**
- Recursiva: procesa primera `<decl>`, luego el resto `<decls>`.
- **Orden**: decl1 -> decls_resto -> concatenar al final.
- Para `int x; int y;` genera: `"field public static x I"` + `"field public static y I"`.

**`<decls> ::= ε { decls.bytecode = "" } | ...`**
- Caso vacío (sin más declaraciones): bytecode vacío.

**`<decl> ::= "int" <id> ';' { decl.bytecode = "field public static " + id.text + " I" }`**

- Crea campo estático JVM con nombre de `<id>.text`.
- Para `int x;` -> `id.text = "x"` -> `"field public static x I"`.


**`<stmts> ::= <stmt> <stmts> { stmts.bytecode = stmt.bytecode + stmts.bytecode }`**
- Igual que `<decls>`: stmt1 + stmt2 + ...

**`<stmt> ::= <id> "=" <exp> ';' { stmt.bytecode = id.bytecode + exp.bytecode + "putstatic " + id.text + "/I" }`**
- **getstatic** (carga variable) + **exp.code** (calcula expresión) + **putstatic** (guarda resultado).
- Para `x = 5 + 3;` genera: `getstatic x/I + ldc 5 + ldc 3 + iadd + putstatic x/I`.

**`<id> ::= letra { id.text = letra.lexeme; id.bytecode = "getstatic " + id.text + "/I" }`**

- Guarda nombre (`x`) y genera instrucción `getstatic x/I` para cargar su valor.

**`<exp> ::= <exp> "+" <term> { exp.code = exp1.code || term.code || "iadd"; exp.type = "int" }`**
- `exp1.code || term.code || "iadd"` significa concatenar códigos + instrucción suma JVM.
- `||` es notación para concatenación de strings de bytecode.

**`<exp> ::= <term> { exp.code = term.code; exp.type = term.type }`**
- Expresión simple = solo el término.

**`<term> ::= número { term.code = "ldc " + número.lexeme; term.type = "int" }`**
- `ldc 5` carga constante 5 en pila JVM.

**`<term> ::= <id> { term.code = id.code; term.type = "int" }`**
- Reusa código de `<id>` (getstatic).

```
x = 5 + 3;
|
v
<stmt>
├── <id> ("x") -> getstatic x/I
├── <exp>
│   └── <exp> + <term>
│       ├── <term>(5) -> ldc 5
│       ├── <term>(3) -> ldc 3
│       └── iadd
└── putstatic x/I

// codigo resultante
field public static x I
field public static y I
getstatic x/I
ldc 5
ldc 3
iadd
putstatic x/I
getstatic x/I
putstatic y/I

// codigo del programa original
int x;
int y;
x = 5 + 3;
y = x;
```


----
# Despues de entender que es EBNF
Para un bytecode portable que compile a múltiples arquitecturas (x86, ARM, RISC-V), necesitas un **bytecode de alto nivel** con un **BFS (grafo de dependencias)** para ordenar instrucciones y optimizaciones. 


### Gramática con Atributos + EBNF
```dart
<programa> ::= <decls> <stmts> { 
  programa.bytecode = decls.bytecode + stmts.bytecode;
  programa.dag = buildDAG(programa.bytecode);
  programa.target = emitMultiArch(programa.dag, "x86"); 
}


<stmt> ::= <id> "=" <exp> ';' { 
  stmt.temp_count = newTemp();
  stmt.code = exp.code + "\nTEMP t" + stmt.temp_count + " = " + exp.result;
  stmt.code += "\n" + id.name + " = t" + stmt.temp_count;
  stmt.nodes = exp.nodes;  // Nodos del DAG
}

<exp> ::= <exp1> "+" <term> {
  exp.result = "t" + newTemp();
  exp.code = exp1.code + term.code + "\n" + exp.result + " = " + exp1.result + " + " + term.result;
  exp.nodes = mergeDAG(exp1.nodes, term.nodes, ADD_OP);
}

<term> ::= número { 
  term.result = número.lexeme;
  term.code = "";
  term.nodes = [ConstNode(número.lexeme)];
}
```

### Grafo de Dependencias (BFS)
```dart
grafo = {
  "t1": ["CONST_5", "CONST_3", "ADD"],
  "x": ["t1"]
}

// BFS para orden topológico
queue = ["CONST_5", "CONST_3"]  // Sin dependencias
while queue:
  nodo = queue.pop(0)
  for dependiente in nodo.usados_por:
    dependiente.pendientes -= 1
    if dependiente.pendientes == 0:
      queue.append(dependiente)
```

### Generador Multi-Arquitectura
```dart
function emitMultiArch(dag, target) {
  switch(target) {
    case "x86":
      return `
        mov eax, 5    ; CONST_5
        mov ebx, 3    ; CONST_3  
        add eax, ebx  ; t1
        mov [x], eax  ; x = t1
      `;
    case "arm":
      return `
        MOV R0, #5    ; CONST_5
        MOV R1, #3    ; CONST_3
        ADD R0, R0, R1; t1
        STR R0, [x]   ; x = t1
      `;
    case "risc-v":
      return `
        li a0, 5      ; CONST_5
        li a1, 3      ; CONST_3
        add a0, a0, a1; t1
        sw a0, 0(x)   ; x = t1
      `;
  }
}
```
1. Gramática -> Bytecode: TEMP t1 = 5 + 3 \n x = t1
2. BFS -> Orden: CONST_5, CONST_3, ADD(t1), ASSIGN(x)  
3. Backend -> Código máquina nativo (x86/ARM/RISCV)

Este bytecode **NO es para VM** (como JVM), sino **intermedio de alto nivel** (como LLVM IR) que genera código máquina nativo:
```dart
TEMP t1 = 5 + 3    <- Bytecode portable
x = t1             <- (independiente de x86/ARM)
```
**Ventaja**: Un solo frontend genera código para x86, ARM, RISC-V desde el mismo bytecode.

```dart
Programa: x = 5 + 3
|
v Árbol Sintáctico
    +
   / \
  5   3
|
v DAG (Directed Acyclic Graph)
CONST_5 ─┐
         ADD ── t1 ── ASSIGN ── x
CONST_3 ─┘
```
**Problema**: `CONST_5` y `CONST_3` deben emitirse **ANTES** de `ADD`, `ADD` antes de `ASSIGN`.

#### **Paso 1: Construir Grafo**
```dart
nodos = {
  "CONST_5": {depende_de: [], usados_por: ["ADD_t1"], pendientes: 0},
  "CONST_3": {depende_de: [], usados_por: ["ADD_t1"], pendientes: 0},
  "ADD_t1":  {depende_de: ["CONST_5", "CONST_3"], usados_por: ["ASSIGN_x"], pendientes: 2},
  "ASSIGN_x":{depende_de: ["ADD_t1"], usados_por: [], pendientes: 1}
}
```

#### **Paso 2: BFS con Cola**
```dart
Cola inicial: ["CONST_5", "CONST_3"]  <- pendientes = 0 (sin dependencias)

Iteración 1:
- Procesa CONST_5
- ADD_t1.pendientes = 2-1 = 1
- Cola: ["CONST_3"]

Iteración 2:
- Procesa CONST_3  
- ADD_t1.pendientes = 1-1 = 0 -> Añade ADD_t1 a cola
- Cola: ["ADD_t1"]

Iteración 3:
- Procesa ADD_t1
- ASSIGN_x.pendientes = 1-1 = 0 -> Añade ASSIGN_x
- Cola: ["ASSIGN_x"]

Iteración 4:
- Procesa ASSIGN_x FIN

```

**Orden BFS final**: `CONST_5, CONST_3, ADD_t1, ASSIGN_x`

#### 4. Gramática con Atributos (Línea por Línea)
```dart
<stmt> ::= <id> "=" <exp> ';' { 
  stmt.temp_count = newTemp();  // t1, t2, t3...
  stmt.code = exp.code + "\nTEMP t" + stmt.temp_count + " = " + exp.result;
  stmt.code += "\n" + id.name + " = t" + stmt.temp_count;
  stmt.nodes = exp.nodes;  // [CONST_5, CONST_3, ADD_t1]
}
```

**Para `x = 5 + 3`** genera:
```dart
TEMP t1 = 5 + 3
x = t1
```

```dart
<exp> ::= <exp1> "+" <term> {
  exp.result = "t" + newTemp();  // "t1"
  exp.code = exp1.code + term.code + "\n" + exp.result + " = " + exp1.result + " + " + term.result;
  exp.nodes = mergeDAG(exp1.nodes, term.nodes, ADD_OP);  // Nodos del DAG
}
```

#### 5. Backend Multi-Arquitectura (Real)
```dart
// Orden BFS: CONST_5, CONST_3, ADD_t1, ASSIGN_x
emit("x86") -> 
mov eax, 5      ; ← CONST_5
mov ebx, 3      ; ← CONST_3
add eax, ebx    ; ← ADD_t1  
mov [x], eax    ; ← ASSIGN_x

emit("ARM") -> 
MOV R0, #5      ; ← CONST_5  
MOV R1, #3      ; ← CONST_3
ADD R0, R0, R1  ; ← ADD_t1
STR R0, [x]     ; ← ASSIGN_x
```

#### 6. ¿Por qué NO DFS aquí?
```dart
DFS iría: CONST_5 -> ADD_t1 -> ASSIGN_x -> (¡stuck! falta CONST_3)
BFS garantiza: todas las dependencias resueltas antes
```