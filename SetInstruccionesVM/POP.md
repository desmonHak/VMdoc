# POP - Sacar un valor de la pila

La instruccion **POP** saca el valor del tope de la pila y lo guarda en un registro.
Es la operacion complementaria a [[PUSH]].

| Instruccion    | opcode | byte2 (reg)     | Tamano  | Descripcion                  |
| :------------: | :----: | :-------------: | :-----: | :--------------------------- |
| `pop reg`      | 0x13   | 0b00 + reg(6b)  | 2 bytes | Sacar valor de pila a registro general |
| `pop reg_ext`  | 0x13   | 0b01 + reg_ext  | 2 bytes | Sacar valor de pila a registro especial |

Complementaria: ver [[PUSH]] para poner valores en la pila.

---

## Comportamiento de POP

Cuando se ejecuta `pop r5`:
1. `valor = [RSP]` (se lee el valor del tope de la pila)
2. `RSP = RSP + tamano` (la pila se encoge hacia arriba)
3. `r5 = valor` (el valor va al registro destino)

```c
// Ejemplo: RSP en 0xFEE8, hay 3 valores en la pila

// Estado de la pila antes de los pops:
// mem[0xFEE8] = valor3  <- tope (RSP apunta aqui)
// mem[0xFEF0] = valor2
// mem[0xFEF8] = valor1

pop r3          // r3 = valor3; RSP = 0xFEF0
pop r2          // r2 = valor2; RSP = 0xFEF8
pop r1          // r1 = valor1; RSP = 0xFF00
// La pila queda vacia (RSP volvio al valor inicial)
```

---

## Registros destino disponibles

### Registros generales (r0 - r15)

```c
pop r0          // sacar de la pila y poner en r0 (64 bits)
pop r0d         // sacar 32 bits de la pila y poner en r0d (los 32 superiores se ponen a 0)
pop r0w         // sacar 16 bits
pop r0b         // sacar 8 bits

pop r5
pop r14
```

### Registros especiales

```c
pop rbp         // restaurar el base pointer (muy comun en epilogo)
pop rsp         // cambiar el stack pointer (usar con cuidado)
pop rflags      // restaurar los flags de condicion
pop cur0        // restaurar el cursor 0
```

---

## Codificacion binaria

```
+--------+--------+
| 0x13   | reg    |
+--------+--------+
  byte0    byte1
```

**byte1 (reg) para registros generales:**
```
bits 7-6 = 0b00            (indica registro general)
bits 5-4 = modo de tamano  (0b11=64b, 0b10=32b, 0b01=16b, 0b00=8b)
bits 3-0 = indice registro (0=r0, 1=r1, ..., 15=r15)
```

**byte1 (reg) para registros especiales:**
```
bits 7-6 = 0b01            (indica registro especial)
bits 5-0 = codigo del registro especial
```

---

## Regla de oro: orden inverso

Los valores deben sacarse de la pila en el ORDEN INVERSO al que se pusieron. Piensa en los
platos: el ultimo que pusiste es el primero que sacas.

```c
// Correcto: LIFO (Last In First Out)
push r1         // 1o en entrar
push r2         // 2o en entrar
push r3         // 3o en entrar
pop  r3         // 1o en salir (ultimo en entrar)
pop  r2         // 2o en salir
pop  r1         // 3o en salir (primero en entrar)

// INCORRECTO: hacerlo en el mismo orden
push r1
push r2
pop  r1         // ERROR LOGICO: r1 recibe el valor de r2, no el suyo
pop  r2         // r2 recibe el valor original de r1
```

---

## Uso tipico: epilogo de funcion

El uso mas comun de POP es restaurar registros callee-saved al final de una funcion:

```c
mi_funcion:
    // Prologo (guardar registros que usaremos)
    push rbp
    mov  rbp, rsp
    push r13
    push r15

    // Cuerpo
    mov  r13, 42
    mov  r15, 99

    // Epilogo: restaurar en ORDEN INVERSO al push
    pop  r15        // restaurar r15
    pop  r13        // restaurar r13
    pop  rbp        // restaurar el frame anterior
    ret             // volver al llamante
```

Nota: la instruccion LEAVE hace automaticamente "mov rsp, rbp / pop rbp", lo que es
equivalente al "pop rbp" cuando no hay registros extra guardados.

---

## Diferencia entre POP rbp y LEAVE

| Situacion            | Instruccion | Que hace exactamente                              |
| :------------------- | :---------: | :------------------------------------------------ |
| Solo restaurar RBP   | `pop rbp`   | Saca de la pila solo el RBP guardado              |
| Liberar frame + RBP  | `leave`     | `mov rsp, rbp` (libera locals) + `pop rbp`        |

Si la funcion uso ENTER para crear el frame, debe usar LEAVE antes del POP de otros registros.
Si la funcion solo hizo PUSH rbp manual, puede usar POP rbp directamente.
