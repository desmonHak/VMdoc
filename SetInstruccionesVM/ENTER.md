# ENTER - Crear un frame de pila

La instruccion **ENTER** crea un "frame de pila" (stack frame) al inicio de cada funcion.
Un frame es el espacio reservado en la pila para las variables locales de esa funcion y para
guardar el estado del frame anterior.

| Instruccion       | opcode | byte2 | frame_size (64 bits) | Tamano   |
| :---------------: | :----: | :---: | :------------------: | :------: |
| `enter frame_sz`  | 0x28   | 0x00  | N bytes              | 10 bytes |

Complementaria: ver [[LEAVE]] para deshacer el frame, [[RET]] para retornar.

---

## Para que sirve un frame de pila

Cuando una funcion se llama, necesita:
1. **Recordar de donde vino**: guardar la direccion de retorno (la hace CALLVM automaticamente).
2. **Guardar el estado del frame anterior**: guardar el valor de RBP del llamante.
3. **Reservar espacio para variables locales**: decrementar RSP en N bytes.

Sin ENTER, si dos funciones usan los mismos registros, se pisarian mutuamente los valores.
El frame de pila crea una "caja" privada para cada llamada a funcion.

Analogia: es como asignar un escritorio nuevo a cada empleado cuando entra a trabajar.
Cuando termina, el escritorio se limpia y el siguiente empleado lo usa.

---

## Comportamiento exacto de ENTER N

ENTER N hace exactamente lo mismo que estas tres instrucciones:

```c
push rbp            // 1. Guardar el base pointer del frame anterior en la pila
mov  rbp, rsp       // 2. El nuevo frame empieza donde esta ahora la pila
subu rsp, N         // 3. Bajar RSP N bytes para reservar espacio a las variables locales
```

Tras `enter N`, el layout del stack queda asi (de mayor a menor direccion):

```
Direccion alta:
  [rbp + 8]  = return_address    <- guardado por CALLVM antes de saltar
  [rbp + 0]  = saved_rbp         <- guardado por ENTER (frame del llamante)
  [rbp - 8]  = variable_local_1  <- primer espacio local
  [rbp - 16] = variable_local_2  <- segundo espacio local
  ...
  [rbp - N]  = variable_local_N/8  <- ultimo espacio local
  [rsp]      = tope actual de la pila
Direccion baja (la pila crece hacia aqui)
```

---

## Acceder a variables locales

Las variables locales se acceden con desplazamientos negativos desde RBP:

```c
mi_funcion:
    enter 24        // reservar 24 bytes: 3 variables de 8 bytes cada una

    // Variable 1: en [rbp - 8]
    mov   r1, 42
    mov   [rbp - 8], r1     // guardar en variable local 1

    // Variable 2: en [rbp - 16]
    mov   r2, 100
    mov   [rbp - 16], r2

    // Variable 3: en [rbp - 24]
    mov   r3, r1
    addu  r3, r2
    mov   [rbp - 24], r3

    // Leer las variables locales
    mov   r4, [rbp - 8]     // r4 = 42
    mov   r5, [rbp - 16]    // r5 = 100
    mov   r6, [rbp - 24]    // r6 = 142

    leave
    ret
```

---

## enter 0: frame sin variables locales

Cuando la funcion no necesita variables locales pero si quiere seguir la convencion de llamada
(por ejemplo, para poder llamar a otras funciones o para que el debugging funcione bien):

```c
funcion_simple:
    enter 0         // solo guarda RBP y establece el frame (sin espacio para locales)
    // Usar solo registros, no variables en la pila
    mov r0, r1
    addu r0, r2
    leave
    ret
```

---

## Codificacion binaria

```
+--------+--------+------------------------+
| 0x28   | 0x00   | frame_size (8 bytes)   |
+--------+--------+------------------------+
  byte0    byte1    bytes 2-9
```

- `byte0` = opcode 0x28
- `byte1` = 0x00 (reservado)
- `bytes 2-9` = frame_size como entero de 64 bits en little-endian

---

## Ejemplo completo: funcion con parametros y locales

```c
// Funcion: calcula a*b + c
// Argumentos: r1=a, r2=b, r3=c
// Retorno: r0 = resultado
calcular:
    enter 8             // reservar 8 bytes para una variable local

    // Guardar el producto intermedio en la variable local
    mulu r1, r2         // r1 = a * b (MULS si son con signo)
    mov  [rbp - 8], r1  // guardar producto en variable local

    // Sumar c
    addu r1, r3         // r1 = a*b + c
    mov  r0, r1         // poner resultado en r0 (convencion de retorno)

    leave               // liberar el frame
    ret                 // volver al llamante

// En el codigo principal:
code:
    mov rsp, 0x00FF0000
    mov rbp, 0x00FF0000

    mov r1, 5           // a = 5
    mov r2, 3           // b = 3
    mov r3, 10          // c = 10
    callvm calcular     // llamar a la funcion

    // r0 = 5*3+10 = 25
    hlt
```

---

## ENTER vs PUSH rbp manual

| Forma          | Codigo                                    | Cuando usar              |
| :------------- | :---------------------------------------- | :----------------------- |
| ENTER N        | `enter N`                                 | La mayoria de las veces  |
| Manual         | `push rbp / mov rbp, rsp / subu rsp, N`   | Cuando necesitas control exacto |
| Sin frame      | (nada)                                    | Funciones muy simples que no llaman a otras |

ENTER es simplemente una forma abreviada de las tres instrucciones manuales.
