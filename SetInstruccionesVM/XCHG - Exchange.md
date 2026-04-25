# XCHG - Intercambio de registros

La instruccion **XCHG** (exchange = intercambiar) intercambia los valores de dos registros
de forma atomica: el valor del registro 1 va al registro 2 y viceversa, sin perder ningun dato.

| Instruccion       | opcode | byte2 | byte3 | byte4 | Tamano  |
| :---------------: | :----: | :---: | :---: | :---: | :-----: |
| `xchg reg1, reg2` | 0x14   | flags | REG1  | REG2  | 4 bytes |

---

## Para que sirve

A veces necesitas intercambiar los contenidos de dos registros. Sin XCHG tendrias que usar
un registro temporal:

```c
// Sin XCHG: necesitas un registro temporal extra (r3)
mov  r3, r1     // temporal = r1
mov  r1, r2     // r1 = r2
mov  r2, r3     // r2 = temporal (valor original de r1)

// Con XCHG: atomico, sin temporal
xchg r1, r2     // r1 <-> r2 en una sola instruccion
```

---

## Tipos de intercambio

### Entre dos registros generales

```c
// Estado antes: r1=10, r2=20
xchg r1, r2
// Estado despues: r1=20, r2=10

// Estado antes: r5=0xABCD, r7=0x1234
xchg r5, r7
// Estado despues: r5=0x1234, r7=0xABCD
```

### Entre registro general y cursor

Este es el uso mas importante de XCHG en VestaVM. Los cursores (cur0-cur3) son registros
especiales que apuntan a memoria host. Para trabajar con un puntero host guardado en un
registro general, hay que "moverlo" al cursor primero.

**ATENCION**: xchg cur0, r14 DESTRUYE el valor de r14. El registro r14 pasa a contener el
valor anterior del cursor, NO el valor que tenia antes del xchg.

```c
// Patron INCORRECTO (r1 se pierde):
xchg  cur0, r1      // cur0 = valor de r1; PERO r1 = valor anterior de cur0 (no lo que quieres)
// r1 ya no contiene el puntero host!

// Patron CORRECTO (usar r14 como scratch):
mov   r14, r1       // copiar el puntero host a r14 (r14 es scratch por convencion)
xchg  cur0, r14     // cur0 = r1 (el puntero host); r14 = valor anterior de cur0 (descartado)
// Ahora cur0 apunta a la memoria host, y r1 no se toco
readcur r2, cur0    // r2 = valor en la memoria host apuntada por cur0
```

### Entre dos cursores

```c
xchg cur0, cur1     // intercambiar los dos cursores
xchg cur1, cur0     // idem en sentido contrario
```

---

## El patron seguro para acceso a memoria host

Cuando tienes un puntero a una estructura en memoria host y quieres leer o escribir
campos de esa estructura:

```c
// Suponer: r8 = puntero al inicio de una estructura host
// Quiero leer un campo en el offset +16 de esa estructura

// Paso 1: calcular la direccion del campo (sin destruir r8)
mov   r14, r8       // r14 = copia de r8
addu  r14, 16       // r14 = r8 + 16 (direccion del campo)

// Paso 2: mover la direccion al cursor
xchg  cur0, r14     // cur0 = r8+16; r14 = valor anterior del cursor (ignorar)

// Paso 3: leer el campo
readcur r5, cur0    // r5 = valor de *(r8 + 16) en memoria host (64 bits)
readcur r5d, cur0   // alternativa: leer solo 32 bits
```

Para **escribir** en un campo de la estructura:

```c
// Quiero escribir el valor de r9 en el campo en offset +24 de la estructura en r8

mov   r14, r8       // copia del puntero base
addu  r14, 24       // r14 = r8 + 24
xchg  cur0, r14     // cur0 = r8+24
writecur cur0, r9   // *(r8 + 24) = r9 (64 bits)
writecur cur0, r9d  // alternativa: escribir 32 bits
writecur cur0, r9w  // alternativa: escribir 16 bits
writecur cur0, r9b  // alternativa: escribir 8 bits
```

---

## Codificacion binaria

```
+--------+--------+--------+--------+
| 0x14   | flags  | REG1   | REG2   |
+--------+--------+--------+--------+
  byte0    byte1    byte2    byte3
```

**byte2 (REG1) y byte3 (REG2):**
```
bit  7   = tipo (0=registro general, 1=registro especial/extension)
bits 6-0 = codigo del registro
```

---

## Ejemplo completo: construir una estructura en memoria host

```c
// Crear un struct en memoria host y rellenar sus campos
// Suponer: alloc devuelve un puntero host en r0

mov   r1, 24        // tamano del struct: 24 bytes
alloc r1            // r0 = puntero al struct en memoria host
mov   r8, r0        // guardar el puntero en r8

// Escribir campo 1 en offset 0 (un valor de 64 bits)
mov   r14, r8       // r14 = puntero base
xchg  cur0, r14     // cur0 = r8
mov   r5, 42        // valor a escribir
writecur cur0, r5   // struct[0] = 42

// Escribir campo 2 en offset 8 (un valor de 32 bits)
mov   r14, r8
addu  r14, 8        // r14 = r8 + 8
xchg  cur0, r14     // cur0 = r8+8
mov   r5, 100
writecur cur0, r5d  // struct[8] = 100 (32 bits)

// Escribir campo 3 en offset 12 (otro valor de 32 bits)
mov   r14, r8
addu  r14, 12       // r14 = r8 + 12
xchg  cur0, r14
mov   r5, 200
writecur cur0, r5d  // struct[12] = 200 (32 bits)

// Leer campo 1 de vuelta
mov   r14, r8
xchg  cur0, r14
readcur r9, cur0    // r9 = struct[0] = 42
```

---

## Resumen de casos de uso

| Caso de uso                         | Instruccion                          |
| :---------------------------------- | :----------------------------------- |
| Intercambiar dos registros generales | `xchg r1, r2`                       |
| Mover puntero host a cursor          | `mov r14, r_ptr / xchg cur0, r14`   |
| Intercambiar dos cursores            | `xchg cur0, cur1`                   |
| Implementar swap sin temporal       | `xchg r1, r2` (en lugar de 3 MOVs)  |
