# Directivas de metaprogramacion del preprocesador

El preprocesador vpp incluye un conjunto de directivas de **metaprogramacion** que
van mas alla de la simple sustitucion de texto: permiten iterar sobre listas,
repetir bloques, definir variables del preprocesador, ejecutar comandos del sistema
y verificar condiciones en tiempo de preprocesamiento.

---

## Variables del preprocesador: `#set`

`#set` define o modifica una variable numerica o de cadena durante el
preprocesamiento. Los operadores soportados son:

```
= += -= *= /= %= &= |= ^= <<= >>= ++ --
```

```c
#set CONTADOR = 0
#set CONTADOR++           // CONTADOR = 1
#set CONTADOR += 10       // CONTADOR = 11
#set NOMBRE = "modulo_a"

// El valor se puede usar como macro normal
mov r1, CONTADOR          // -> mov r1, 11
```

`#set` es equivalente a `#define` pero no emite advertencia si la variable
ya estaba definida (semantica de asignacion, no de redefinicion).

---

## Bucle sobre lista: `#foreach` / `#endforeach`

`#foreach` itera sobre una lista de elementos y repite el bloque para cada uno:

```c
// Forma basica: lista inline
#foreach OP in (add, sub, mul, div)
// Instruccion OP generada por el preprocesador
emit_OP:
    nop1
#endforeach

// Resultado expandido:
// emit_add:
//     nop1
// emit_sub:
//     nop1
// emit_mul:
//     nop1
// emit_div:
//     nop1
```

Cada iteracion expande `OP` con el elemento correspondiente de la lista.
Se puede combinar con `##` para construir nombres:

```c
#define HANDLER(op)  handle_##op
#foreach OP in (click, key, resize)
    HANDLER(OP),
#endforeach
// -> handle_click, handle_key, handle_resize,
```

---

## Repeticion numerica: `#repeat` / `#endrepeat`

`#repeat EXPR` repite el bloque un numero fijo de veces. Dentro del bloque,
`__REPEAT_INDEX__` contiene el indice actual (0-based):

```c
// Emitir 4 instrucciones nop consecutivas
#repeat 4
    nop1    // iteracion __REPEAT_INDEX__
#endrepeat

// Generar una tabla de registros
#repeat 8
    .db r__REPEAT_INDEX__,
#endrepeat
// -> .db r0, .db r1, .db r2, ... .db r7,
```

---

## Arrays del preprocesador: `#array`

`#array` define una lista de elementos que puede recorrerse con `#foreach`:

```c
#array OPCODES (add, sub, mul, div, and, or, xor)
#array REGS    (r1, r2, r3, r4, r5, r6)

// Iterar sobre el array
#foreach OP in OPCODES
    case OP: ...
#endforeach
```

Los arrays se definen una vez y se pueden referenciar en multiples `#foreach`.

---

## Ejecutar comando externo: `#exec`

`#exec VARNAME comando` ejecuta un comando del sistema en tiempo de
preprocesamiento y guarda su salida como una macro:

```c
// Capturar la fecha de build con una herramienta externa
#exec BUILD_DATE  date +%Y%m%d
// BUILD_DATE queda definida con la salida del comando, p.ej. "20260426"

// Obtener el hash del commit actual
#exec GIT_HASH  git rev-parse --short HEAD

// Usar las macros resultantes
mov r1, BUILD_DATE
```

`#exec` es util para embeber informacion de version directamente en el bytecode.

---

## Asercion en preprocesamiento: `#assert`

`#assert EXPR` detiene la compilacion con un error si la expresion es falsa (0):

```c
#define API_VERSION 3

// Verificar que la version es compatible
#assert API_VERSION >= 2

// Con mensaje de error personalizado
#assert API_VERSION >= 2  "Se requiere API_VERSION >= 2"

// Verificar condicion de plataforma
#ifdef __WINDOWS__
    #assert 0  "Este modulo solo compila en Linux"
#endif
```

La expresion se evalua con el mismo motor que `#if`, por lo que puede usar
operadores aritmeticos, logicos, comparaciones y macros definidas previamente.

---

## Macros multilinea: `#macro` / `#endmacro`

Alternativa a `#define` para macros con cuerpo de varias lineas sin necesidad
de `\` de continuacion:

```c
#macro INIT_REGS(base, count)
    mov r1, base
    mov r2, count
    mov r3, 0
    adds r3, r1
#endmacro

// Uso
INIT_REGS(100, 4)
// Se expande en las 4 instrucciones anteriores
```

---

## Condicionales avanzados: `#if` con expresiones

Ademas de `#ifdef` / `#ifndef`, el preprocesador soporta `#if` con expresiones
aritmeticas completas, incluyendo la funcion `defined()`:

```c
#define VERSION 3

#if VERSION >= 2 && defined(FEATURE_X)
    // codigo para version 2+ con FEATURE_X
#elif VERSION == 1
    // codigo para version 1
#else
    // fallback
#endif

// Combinaciones de operadores soportados en #if:
// Aritmeticos:  + - * / % << >> ~ & | ^
// Comparacion:  == != < <= > >=
// Logicos:      && || !
// Especial:     defined(NAME)
```

---

## Directivas de informacion: `#warning` / `#error` / `#line`

```c
// Emitir un aviso (no detiene la compilacion)
#warning "Este modulo esta obsoleto, usar modulo_v2"

// Emitir un error y detener la compilacion
#error "No se puede compilar sin PLATFORM definida"

// Sobreescribir numero de linea y nombre de fichero en mensajes de error
#line 100 "archivo_generado.vel"
```

---

Ver tambien:
- [[Macros constantes.md]] - `#define` sin parametros y macros predefinidas
- [[Macros parametricas.md]] - macros funcion, `##`, `#`, `#macro`
- [[Macros nativas.md]] - `#import` de librerias de macros
