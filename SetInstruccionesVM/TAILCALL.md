# TAILCALL - Llamada en posicion de cola (Tail Call Optimization)

Cuando una funcion llama a otra funcion, la VM necesita recordar donde volver cuando la
segunda funcion termine. Para eso guarda la **direccion de retorno** en la pila. Si esa
segunda funcion llama a una tercera, apila otra direccion. Asi sucesivamente.

El problema aparece con la **recursion profunda**: una funcion que se llama a si misma
miles de veces apila miles de direcciones de retorno hasta que la pila se desborda
(stack overflow) y el proceso muere.

`TAILCALL` resuelve esto cuando la llamada recursiva es la **ultima operacion** de la
funcion (posicion de cola). En ese caso, en lugar de apilar una nueva direccion de
retorno, reutiliza la ya existente del llamante original. La pila no crece.

Analogia: imagina que tu jefe te da un sobre para entregar en otra oficina, y cuando
llega el mensajero A, en vez de mandar al mensajero B de vuelta a ti, el mensajero A
le pasa el sobre directamente al mensajero B diciendole "cuando termines, vuelve a donde
yo iba a volver". No hay intermediarios acumulados.

| Instruccion | opcode0 | opcode1 | Modo     | Tamano  | Descripcion                                    |
| :---------: | :-----: | :-----: | :------: | :-----: | :--------------------------------------------- |
| `tailcall`  |  0x00   |  0x24   | REG      | 4 bytes | Llamada en cola: reutiliza frame, no apila ret |

Implementacion: `src/runtime/exec_instruction_closure.cpp`

---

## Para que sirve: el problema de la recursion

Sin TCO, cada llamada recursiva consume espacio de pila:

```c
// factorial(n) sin TCO: necesita O(n) espacio de pila
// factorial(1000) apila 1000 direcciones de retorno
factorial:
    cmpu  r1, 1
    jmp.jle base         // si n <= 1, retornar 1
    subu  r1, 1          // n - 1
    // ... necesita apilar ret_addr y llamar recursivamente ...
    // pero callvm o push+jmp apila una nueva direccion cada vez
    // con n=1000: la pila tiene 1000 niveles -> stack overflow posible

base:
    mov r0, 1
    ret
```

Con TCO usando un **acumulador** (parametro extra que acumula el resultado):

```c
// factorial_acc(r1=n, r2=acc): O(1) pila, sin importar el valor de n
factorial_acc:
    cmpu  r1, 1
    jmp.jbe base         // si n <= 1, retornar acc

    mulu  r2, r1, r2     // acc = n * acc
    subu  r1, 1          // n = n - 1

    mov   r3, @Absolute("code.factorial_acc")
    tailcall r3          // reutiliza el frame actual; pila no crece

base:
    mov r0, r2           // resultado = acc
    ret
```

Con `tailcall`, `factorial_acc(1000000, 1)` funciona igual que `factorial_acc(3, 1)`:
la pila siempre tiene exactamente un nivel.

---

## Semantica de TAILCALL

```c
tailcall r_fn    // salta a fn_addr sin apilar una nueva direccion de retorno
```

Pasos que ejecuta la instruccion:

1. **Si existe un FrameHeader OOP activo** (empujado por CALLVIRT o CALLCLOSURE):
   - Restaura `RSP = frame->frame_base` (libera el espacio del frame actual).
   - Desapila el FrameHeader de `frame_stack` (`frame_stack = frame->prev`).
   - Libera la memoria del FrameHeader.

2. **No apila ninguna nueva direccion de retorno.** La ret_addr que el llamante original
   dejo en la pila es la que usara la funcion destino cuando ejecute `RET`.

3. **Salta directamente** a `fn_addr = regs[r_fn]`.

---

## Diferencia con una llamada convencional

| Criterio               | Llamada convencional           | TAILCALL                        |
| :--------------------- | :----------------------------- | :------------------------------ |
| Apila ret_addr         | Si (RSP -= 8)                  | No (reutiliza la existente)     |
| Apila FrameHeader OOP  | Si (callvirt / callclosure)    | No (desapila el actual)         |
| Crecimiento de pila    | +1 frame por nivel             | Constante (O(1))                |
| Recursion N niveles    | O(N) pila, puede desbordar     | O(1) pila, nunca desborda       |
| Cuado usarlo           | Cuando necesitas el resultado  | Solo cuando es la ultima accion |

---

## Patron de uso: factorial tail-recursive

```c
// Invocar factorial_acc(10, 1) desde el codigo principal
code:
    mov   rsp, 0x00FF0000
    mov   rbp, 0x00FF0000

    mov   r1, 10           // n = 10
    mov   r2, 1            // acc = 1 (acumulador inicial)

    // Empujar la direccion de retorno manualmente (equivale a CALL):
    mov   r9, 0
    subu  rsp, 8
    mov   r4, @Absolute("code.despues")
    mov   [rsp + r9*1], r4  // *RSP = direccion a donde volver

    jmp.jmp factorial_acc   // saltar a la funcion (ret_addr ya esta en pila)

despues:
    // r0 = 3628800 (10!)
    hlt

// Implementacion tail-recursive
factorial_acc:
    cmpu  r1, 1
    jmp.jbe base_case      // si n <= 1, retornar acc

    mulu  r2, r1, r2       // acc = n * acc
    subu  r1, 1            // n = n - 1

    mov   r3, @Absolute("code.factorial_acc")
    tailcall r3             // llamada en cola: no crece la pila

base_case:
    mov   r0, r2            // resultado = acc acumulado
    ret                     // usa la ret_addr que el llamante original dejo
```

---

## Recursion mutua entre funciones distintas

`TAILCALL` no esta limitado a la recursion directa. Dos o mas funciones pueden llamarse
mutuamente en posicion de cola sin crecer la pila, implementando maquinas de estados:

```c
// par(r1=n) -> R0: 1 si n es par, 0 si es impar
par:
    xor   r0, r0             // r0 = 0 (preparar resultado impar por defecto)
    cmpu  r1, r0
    jmp.jeq par_base         // si n == 0, es par

    subu  r1, 1              // n - 1
    mov   r3, @Absolute("code.impar")
    tailcall r3              // impar(n-1) en cola

par_base:
    mov   r0, 1              // es par
    ret

// impar(r1=n) -> R0: 1 si n es impar, 0 si es par
impar:
    xor   r0, r0
    cmpu  r1, r0
    jmp.jeq impar_base       // si n == 0, no es impar

    subu  r1, 1
    mov   r3, @Absolute("code.par")
    tailcall r3              // par(n-1) en cola

impar_base:
    mov   r0, 0              // no es impar
    ret
```

Con `TAILCALL`, `par(1000000)` funciona sin stackoverflow aunque haya un millon de
llamadas mutuas.

---

## Maquina de estados con TAILCALL

Un caso de uso comun en compiladores es implementar automatas finitos. Cada estado es una
funcion; la transicion al siguiente estado es un `tailcall`:

```c
// Maquina de estados: A -> B -> C -> FIN
estado_A:
    // ... procesar datos en estado A ...
    mov   r3, @Absolute("code.estado_B")
    tailcall r3    // transicion a B sin crecer la pila

estado_B:
    // ... procesar datos en estado B ...
    mov   r3, @Absolute("code.estado_C")
    tailcall r3

estado_C:
    // ... estado final ...
    mov   r0, 1    // exito
    ret
```

---

## Interaccion con el sistema OOP y excepciones

Cuando una funcion es invocada con `CALLVIRT` o `CALLCLOSURE`, se apila un `FrameHeader`
en `frame_stack` (necesario para que `THROW` pueda buscar el handler correcto). Si esa
funcion usa `TAILCALL` internamente:

- `TAILCALL` desapila el FrameHeader actual antes de saltar.
- El FrameHeader del llamante (si existe) permanece en la cadena.
- La cadena de excepciones sigue siendo correcta para THROW/RETHROW.

Si la funcion actual NO tiene FrameHeader (invocada con el patron push+jmp manual), la
instruccion no intenta desapilar nada y simplemente salta.

---

## Consideraciones importantes

- La funcion destino **debe retornar con `RET`** para usar la ret_addr correcta.
- Todos los argumentos de la funcion destino deben estar en los registros **antes** del
  `tailcall`. Una vez ejecutado, ya no hay oportunidad de ajustar nada.
- `TAILCALL` no apila ninguna ret_addr nueva. Si la funcion destino hace una llamada
  convencional hacia otra funcion, esa llamada si crecera la pila normalmente.
- La funcion debe estar en "posicion de cola": la llamada recursiva es lo **ultimo** que
  hace antes de retornar. Si despues de la llamada hay que multiplicar, sumar, etc., no
  es posicion de cola y no se puede usar `TAILCALL`.

---

## Codificacion binaria (FIXED_4, modo REG)

```
+--------+--------+----------+----------+
| 0x00   | 0x24   |  ctrl    |  regs    |
+--------+--------+----------+----------+
  byte0    byte1    byte2      byte3

ctrl (byte2): 0x00

regs (byte3):
  bits 7-4 = 0000   (nibble alto no usado)
  bits 3-0 = r_fn   (registro con la direccion de la funcion destino)
```

Ejemplo: `tailcall r3` (r_fn = 3):
```
0x00  0x24  0x00  0x03
```

---

Ver tambien: [[CALLVM]], [[ENTER]], [[RET]], [[CLOSURE]]
