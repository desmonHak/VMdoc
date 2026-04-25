# PUSH - Poner un valor en la pila

La instruccion **PUSH** guarda el valor de un registro en la pila (stack). La pila es una
estructura de datos tipo LIFO (Last In First Out): el ultimo valor que se pone es el primero
que se saca.

| Instruccion   | opcode | byte2 (reg)      | Tamano  | Descripcion                 |
| :-----------: | :----: | :--------------: | :-----: | :-------------------------- |
| `push reg`    | 0x12   | 0b00 + reg(6b)   | 2 bytes | Poner registro general en pila |
| `push reg_ext`| 0x12   | 0b01 + reg_ext   | 2 bytes | Poner registro especial en pila |

Complementaria: ver [[POP]] para sacar valores de la pila.

---

## Que es la pila (stack)

Imagina una pila de platos: solo puedes poner un plato encima del monton (push) o sacar el
plato de arriba (pop). No puedes acceder a los platos del medio sin desapilar los de arriba.

En VestaVM la pila:
- Crece hacia **abajo** en memoria (RSP disminuye al poner, aumenta al sacar).
- RSP siempre apunta al **tope de la pila** (el dato mas reciente).
- Se usa para guardar temporalmente registros, argumentos de funciones y direcciones de retorno.

---

## Comportamiento de PUSH

Cuando se ejecuta `push r5`:
1. `RSP = RSP - tamano` (la pila crece hacia abajo)
2. `[RSP] = valor del registro` (se escribe el valor en la nueva posicion)

```c
// Ejemplo visual: RSP comienza en 0xFF00

push r1         // RSP = 0xFF00 - 8 = 0xFEF8; mem[0xFEF8] = r1
push r2         // RSP = 0xFEF8 - 8 = 0xFEF0; mem[0xFEF0] = r2
push r3         // RSP = 0xFEF0 - 8 = 0xFEE8; mem[0xFEE8] = r3

// Para recuperar los valores (en orden inverso):
pop  r3         // r3 = mem[0xFEE8]; RSP = 0xFEF0
pop  r2         // r2 = mem[0xFEF0]; RSP = 0xFEF8
pop  r1         // r1 = mem[0xFEF8]; RSP = 0xFF00
```

---

## Registros que se pueden empujar

### Registros generales (r0 - r15)

```c
push r0         // empujar el valor de r0 (64 bits)
push r0d        // empujar los 32 bits bajos de r0
push r0w        // empujar los 16 bits bajos de r0
push r0b        // empujar los 8 bits bajos de r0

push r5         // empujar r5
push r14        // empujar r14
```

### Registros especiales

```c
push rbp        // empujar el base pointer (muy comun en prologo de funcion)
push rsp        // empujar el stack pointer actual
push rip        // empujar el instruction pointer
push rflags     // empujar los flags de condicion
push cur0       // empujar el cursor 0 (puntero a memoria host)
```

---

## Codificacion binaria

```
+--------+--------+
| 0x12   | reg    |
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
bits 5-0 = codigo registro especial (ver tabla en REGISTROS.md)
```

---

## Uso tipico: prologo de funcion

El uso mas comun de PUSH es guardar registros que la funcion necesita preservar (callee-saved):

```c
mi_funcion:
    // Prologo: guardar registros que vamos a usar
    push rbp            // guardar el base pointer anterior
    mov  rbp, rsp       // establecer el nuevo frame
    push r13            // r13 es callee-saved: guardarlo si lo usamos
    push r15            // idem para r15

    // Cuerpo de la funcion
    mov  r13, 100       // usar r13 libremente
    mov  r15, r13
    addu r15, 1

    // Epilogo: restaurar registros en ORDEN INVERSO
    pop  r15
    pop  r13
    pop  rbp
    ret
```

Nota: la instruccion ENTER hace automaticamente "push rbp / mov rbp, rsp / sub rsp, N",
asi que en la practica se usa ENTER en lugar de hacer PUSH manualmente para el prologo.

---

## Usos comunes de PUSH

1. **Guardar registros callee-saved** al inicio de una funcion.
2. **Pasar argumentos** a funciones que los esperan en la pila (convencion de pila).
3. **Guardar temporalmente** un valor cuando todos los registros estan ocupados.
4. **Salvar el estado** antes de una operacion que modifica varios registros.
