# RET - Retornar de una funcion

La instruccion **RET** (return = retornar) devuelve el control al codigo que llamo a la
funcion actual. Lee la direccion de retorno de la pila y salta a ella.

| Instruccion | opcode | Tamano  | Descripcion                            |
| :---------: | :----: | :-----: | :------------------------------------- |
| `ret`       | 0xC3   | 1 byte  | Retornar al llamante usando la pila    |

---

## Para que sirve

Cuando una funcion termina su tarea, necesita "volver" al punto del programa donde fue
llamada. RET hace exactamente eso.

Analogia: si llamar a una funcion es como ir a hacer un encargo, RET es volver a tu posicion
original despues de completarlo.

---

## Comportamiento exacto

RET hace estas operaciones en orden:

```c
// Equivalente a:
ret_addr = [rsp]    // 1. Leer la direccion de retorno del tope de la pila
rsp = rsp + 8       // 2. Sacar ese valor de la pila (como un POP)
rip = ret_addr      // 3. Saltar a la direccion de retorno (continuar desde donde se llamo)
```

La direccion de retorno fue puesta en la pila por la instruccion CALLVM cuando se hizo la
llamada a la funcion.

---

## La secuencia de llamada completa

Para entender RET hay que ver el ciclo completo:

```c
// En el codigo llamante:
mov   r1, 5         // argumento 1 = 5
callvm mi_funcion   // 1. Push return_address, 2. Saltar a mi_funcion

// Cuando RET se ejecuta, la ejecucion continua AQUI:
mov   r2, r0        // usar el resultado devuelto en r0

//-----------------------------------------------------------

// En la funcion llamada:
mi_funcion:
    enter 0         // crear frame (guarda RBP, establece nuevo frame)
    // ... hacer trabajo ...
    mov  r0, 42     // poner resultado en r0 (convencion de retorno)
    leave           // restaurar RBP y liberar frame
    ret             // leer return_address de la pila y saltar alli
```

---

## El estado de la pila antes de RET

Para que RET funcione correctamente, RSP debe apuntar exactamente a la direccion de retorno
que CALLVM puso en la pila. Esto se garantiza si:
1. Cada PUSH tiene su correspondiente POP.
2. La funcion uso ENTER y LEAVE correctamente (LEAVE restaura RSP).
3. No se han hecho manipulaciones manuales incorrectas de RSP.

```c
// Estado de la pila justo antes de RET:
// [rsp]     = return_address  <- RET lee esto
// [rsp + 8] = saved_rbp       <- ya restaurado por LEAVE
// [rsp +16] = ...             <- frame del llamante (no tocamos esto)
```

---

## Codificacion binaria

```
+--------+
| 0xC3   |
+--------+
  byte0
```

RET es una instruccion de 1 byte.

---

## Convencion de valor de retorno

Por convencion, el valor de retorno de una funcion se pone en **r0** antes de ejecutar RET.
El llamante lo lee de r0 despues de que CALLVM retorna.

```c
// Funcion que devuelve un valor:
sumar:
    enter 0
    addu r1, r2     // r1 = r1 + r2
    mov  r0, r1     // poner resultado en r0
    leave
    ret             // retornar; r0 tiene el resultado

// Llamante:
    mov  r1, 3
    mov  r2, 7
    callvm sumar    // llama a sumar(3, 7)
    // r0 = 10 despues de retornar
```

---

## Ejemplo completo con llamadas anidadas

```c
// Funcion que calcula: max(a, b)
maximo:
    enter 0
    cmpu r1, r2     // comparar a y b (sin signo)
    jmp.jge es_mayor_o_igual
    mov  r0, r2     // b es mayor: devolver b
    leave
    ret
es_mayor_o_igual:
    mov  r0, r1     // a es mayor o igual: devolver a
    leave
    ret

// Funcion que calcula: max(a, b) + max(c, d)
suma_maximos:
    enter 8         // reservar 8 bytes para guardar el primer maximo

    // Calcular max(r1, r2)
    callvm maximo   // r0 = max(r1, r2)
    mov  [rbp - 8], r0  // guardar primer resultado

    // Calcular max(r3, r4) - mover argumentos
    mov  r1, r3
    mov  r2, r4
    callvm maximo   // r0 = max(r3, r4)

    // Sumar los dos maximos
    addu r0, [rbp - 8]  // r0 = max(r1,r2) + max(r3,r4)

    leave
    ret

code:
    mov rsp, 0x00FF0000
    mov rbp, 0x00FF0000

    mov r1, 5
    mov r2, 3
    mov r3, 8
    mov r4, 2
    callvm suma_maximos  // r0 = max(5,3) + max(8,2) = 5 + 8 = 13
    // r0 = 13
    hlt
```

---

## Errores comunes

| Error                              | Consecuencia                                      |
| :--------------------------------- | :------------------------------------------------ |
| Olvidar LEAVE antes de RET         | RET lee saved_rbp como return_address (crash)     |
| PUSH sin POP correspondiente       | RSP desalineado; RET lee dato incorrecto          |
| POP de mas antes de RET            | RSP apunta demasiado arriba; RET sale del frame   |
| Modificar RSP manualmente de forma incorrecta | RET no encuentra el return_address |
