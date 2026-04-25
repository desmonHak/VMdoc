# LEAVE - Destruir el frame de pila

La instruccion **LEAVE** deshace el frame de pila creado por [[ENTER]], restaurando RSP y RBP
a sus valores anteriores a la llamada. Siempre va justo antes de [[RET]].

| Instruccion | opcode | Tamano  | Descripcion                  |
| :---------: | :----: | :-----: | :--------------------------- |
| `leave`     | 0x29   | 1 byte  | Destruir el frame de pila actual |

---

## Para que sirve

Cuando una funcion termina, debe "limpiar su escritorio" antes de irse. ENTER reservo espacio
en la pila; LEAVE lo libera y restaura el estado del frame anterior.

Analogia: si ENTER es "entrar a la oficina y preparar tu mesa de trabajo", LEAVE es "ordenar
tu mesa y dejarla como estaba antes de que llegaras".

---

## Comportamiento exacto

LEAVE hace exactamente lo mismo que estas dos instrucciones:

```c
mov rsp, rbp    // 1. Liberar TODO el espacio local (mover RSP de vuelta a donde esta RBP)
pop rbp         // 2. Restaurar el base pointer del frame anterior (sacar de la pila)
```

Esto es el proceso inverso de lo que hizo ENTER:
- ENTER hizo: `push rbp` + `mov rbp, rsp` + `subu rsp, N`
- LEAVE hace: `mov rsp, rbp` + `pop rbp`

---

## La secuencia tipica: ENTER -> ... -> LEAVE -> RET

Toda funcion que use ENTER debe terminar con LEAVE seguido de RET:

```c
mi_funcion:
    enter 16        // ENTER: guarda RBP, crea frame de 16 bytes
    // ...cuerpo de la funcion...
    leave           // LEAVE: libera frame, restaura RBP anterior
    ret             // RET: vuelve al llamante (usa el return address en la pila)
```

Si se olvida LEAVE y se llama directamente a RET, RET leria el saved_rbp como si fuera una
direccion de retorno, lo que provoca un salto incorrecto y probablemente un crash.

---

## Codificacion binaria

```
+--------+
| 0x29   |
+--------+
  byte0
```

LEAVE es una instruccion de 1 byte, igual que RET (0xC3) y HLT (0x01).

---

## Ejemplo completo: funcion anidada

```c
// Funcion interna: calcula el cuadrado de r1
cuadrado:
    enter 0         // frame sin variables locales (solo por convencion)
    mulu r1, r1     // r1 = r1 * r1
    mov  r0, r1     // retorno en r0
    leave           // liberar frame
    ret

// Funcion principal: suma de cuadrados
suma_cuadrados:
    enter 16        // reservar 16 bytes para dos variables locales

    // Calcular cuadrado del primer argumento (r1)
    push r1         // salvar r1 (lo necesitaremos despues)
    push r2         // salvar r2
    callvm cuadrado // cuadrado(r1); resultado en r0
    mov  [rbp - 8], r0  // guardar cuadrado(r1) en variable local 1
    pop  r2         // restaurar r2
    pop  r1         // restaurar r1

    // Calcular cuadrado del segundo argumento (r2)
    mov  r1, r2         // mover r2 a r1 (argumento de cuadrado)
    callvm cuadrado     // cuadrado(r2); resultado en r0
    mov  [rbp - 16], r0 // guardar cuadrado(r2) en variable local 2

    // Sumar los dos cuadrados
    mov  r1, [rbp - 8]  // cargar cuadrado(r1)
    addu r1, [rbp - 16] // r1 = cuadrado(r1) + cuadrado(r2)
    mov  r0, r1         // resultado en r0

    leave
    ret

code:
    mov rsp, 0x00FF0000
    mov rbp, 0x00FF0000

    mov r1, 3           // primer argumento: 3
    mov r2, 4           // segundo argumento: 4
    callvm suma_cuadrados // r0 = 3^2 + 4^2 = 9 + 16 = 25

    hlt
```

---

## Casos especiales

### Funcion sin ENTER (muy simple)

Si una funcion es tan simple que no necesita frame (no llama a otras funciones, no tiene
variables locales, y los registros que usa son caller-saved), puede omitir ENTER y LEAVE:

```c
funcion_minima:
    // Sin enter: usar directamente los registros
    addu r1, r2     // r1 = r1 + r2
    mov  r0, r1     // retorno
    ret             // volver (sin leave porque no hubo enter)
```

### Multiples puntos de salida

Si una funcion tiene varios puntos de retorno (early return), cada uno necesita LEAVE antes de RET:

```c
funcion_con_early_return:
    enter 8

    cmpu r1, 0
    jmp.je retorno_cero     // si r1 == 0, salir antes

    // Camino normal
    mov  [rbp - 8], r1
    // ... procesamiento ...
    mov  r0, 42
    leave                   // LEAVE en el camino normal
    ret

retorno_cero:
    mov  r0, 0
    leave                   // LEAVE en el early return
    ret
```
