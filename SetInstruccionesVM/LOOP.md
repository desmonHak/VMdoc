# LOOP - Bucle con contador automatico

La instruccion **LOOP** realiza un salto hacia atras y decrementa (o incrementa) un contador
automaticamente. Es una forma compacta de escribir bucles sin tener que hacer la comparacion
y el salto a mano.

| Instruccion     | opcode0 | opcode1 | byte2 | byte3 | desplazamiento 32 bits | Tamano  |
| :-------------: | :-----: | :-----: | :---: | :---: | :--------------------: | :-----: |
| `loop etiqueta` | 0x00    | 0x31    | 0x00  | 0x00  | disp32 (relativo)      | 8 bytes |

---

## Para que sirve

Cuando quieres repetir un bloque de codigo un numero fijo de veces, necesitas un **bucle**.
Sin LOOP tendrias que escribir manualmente:

```c
// Sin LOOP: bucle manual con comparacion y salto
mov r09, 5          // contador = 5

inicio_bucle:
    // ... hacer algo 5 veces ...
    subu  r09, 1        // contador--
    cmpu  r09, 0        // comparar con 0
    jmp.jne inicio_bucle // si no es 0, repetir

// Con LOOP: el contador y la comparacion son automaticos
mov r09, 5          // contador = 5

inicio_bucle:
    // ... hacer algo 5 veces ...
    loop inicio_bucle   // R09--; si R09 != 0, saltar a inicio_bucle
```

LOOP ahorra dos instrucciones (`subu` + `cmpu`) por iteracion.

---

## El registro contador R09

LOOP siempre usa el registro **R09** como contador. Antes de empezar el bucle, debes cargar
en R09 el numero de iteraciones que quieres.

Analogia: R09 es como el marcador de una cuenta regresiva. Lo pones en 5, y cada vez que
el bucle da una vuelta, el marcador baja uno. Cuando llega a 0, el bucle para.

---

## Dos modos: decreciente e creciente (flag DF)

LOOP tiene dos modos controlados por el **flag DF** (Direction Flag, bandera de direccion),
que es parte de los registros de estado de la VM:

### Modo decreciente: DF = 0 (por defecto con `CLD`)

El contador R09 se decrementa en cada iteracion. El bucle para cuando R09 llega a 0.

```c
/*
 * Pseudocodigo de LOOP cuando DF = 0:
 */
if (DF == 0) {
    R09--;              // decrementar contador
    if (R09 != 0)
        goto etiqueta;  // si no llegamos a 0, repetir
    // si R09 == 0: no salta, el bucle termino
}
```

Este es el modo mas comun. Se activa con `cld` (Clear Direction flag):

```c
cld                     // DF = 0 (modo decreciente, es el default)
mov  r09, 5             // queremos 5 iteraciones

inicio:
    // codigo que se repite 5 veces
    loop inicio         // R09--; si R09 != 0 -> saltar a inicio
// aqui llegamos cuando R09 == 0 (despues de 5 iteraciones)
```

### Modo creciente: DF = 1 (con `STD`)

El contador R09 se incrementa. El bucle para cuando R09 alcanza el limite en R10.

```c
/*
 * Pseudocodigo de LOOP cuando DF = 1:
 */
if (DF == 1) {
    R09++;              // incrementar contador
    if (R09 < R10)      // R10 = limite superior
        goto etiqueta;
    // si R09 >= R10: no salta, el bucle termino
}
```

Se activa con `std` (Set Direction flag):

```c
mov  r10, 5             // limite = 5
xor  r09, r09           // R09 = 0 (punto de partida)
std                     // DF = 1 (modo creciente)

inicio:
    // codigo que se repite mientras R09 < 5
    loop inicio         // R09++; si R09 < R10 -> saltar a inicio
// aqui llegamos cuando R09 == 5

cld                     // buena practica: restaurar DF = 0 al terminar
```

> **Atencion:** `std` cambia el comportamiento de LOOP globalmente. Recuerda llamar
> `cld` cuando el bucle creciente termine, o los bucles siguientes se comportaran mal.

---

## Equivalencia con bucles en lenguajes de alto nivel

### Modo DF=0 equivale a un `while` decreciente:

```c
// Vesta (DF=0):
cld
mov r09, 5
inicio:
    // cuerpo
    loop inicio

// Equivalente en C:
int contador = 5;
while (contador > 0) {
    // cuerpo
    contador -= 1;
}
```

### Modo DF=1 equivale a un `for` con indice:

```c
// Vesta (DF=1):
mov r10, 5
xor r09, r09
std
inicio:
    // cuerpo (r09 es el indice: 0, 1, 2, 3, 4)
    loop inicio
cld

// Equivalente en C:
int limite = 5;
for (int i = 0; i < limite; i++) {
    // cuerpo (i es el indice)
}
```

---

## Codificacion binaria

```
+--------+--------+--------+--------+--------+--------+--------+--------+
| 0x00   | 0x31   | 0x00   | 0x00   |   desplazamiento de 32 bits       |
+--------+--------+--------+--------+--------+--------+--------+--------+
  byte0    byte1    byte2    byte3    bytes 4-7 (little-endian, relativo a RIP)
```

El desplazamiento es relativo a la direccion de la instruccion siguiente al LOOP.
Un desplazamiento negativo salta hacia atras (hacia el inicio del bucle, que es lo normal).

---

## Ejemplo completo: sumar los numeros del 1 al 5

```c
@Format("raw")
@SpaceAddress { @Name("mem"), @IniAddress(0x0), @EndAddress(0xFFFFFFFFFFFFFFFF) }
@Section { @Name("code"), @SpaceAddress("mem") @Align(0x1000) }

code:
    mov  rsp, 0x00FF0000
    mov  rbp, 0x00FF0000

    // Sumar 1+2+3+4+5 = 15
    // Usaremos R09 como contador (de 5 a 1) y R08 como acumulador
    mov  r08, 0         // acumulador = 0
    mov  r09, 5         // contador = 5 (empezamos en 5 y bajamos)
    cld                 // DF = 0 -> modo decreciente

suma_loop:
    addu r08, r09       // acumulador += contador  (5, luego 4, luego 3...)
    loop suma_loop      // R09--; si R09 != 0 -> repetir

    // Aqui: R08 = 5+4+3+2+1 = 15, R09 = 0
    hlt
```

---

## Ejemplo completo: inicializar un array con indices (modo creciente)

```c
code:
    mov  rsp, 0x00FF0000
    mov  rbp, 0x00FF0000

    // Crear un array de 5 elementos: [0, 1, 2, 3, 4]
    // en la pila (espacio de 40 bytes = 5 * 8 bytes)
    subsp rsp, 40

    mov  r10, 5         // limite = 5
    xor  r09, r09       // contador = 0
    std                 // DF = 1 -> modo creciente

    mov  r14, rsp       // r14 = direccion del inicio del array

init_loop:
    // escribir el indice actual (r09) en la posicion r14 + r09*8
    mov   r12, r09
    mulu  r12, 8        // offset = indice * 8 bytes
    addu  r12, r14      // r12 = &array[indice]

    mov  r13, r12
    xchg cur0, r13
    writecur cur0, r09  // array[indice] = indice

    loop init_loop      // R09++; si R09 < R10 -> repetir

    cld                 // restaurar DF = 0

    // Array en pila: [0, 1, 2, 3, 4]
    hlt
```

---

## Resumen

| DF | Accion en cada iteracion | Condicion de parada    | Instrucciones para activar |
| :-: | :---------------------- | :--------------------- | :------------------------- |
| 0  | `R09 -= 1`             | `R09 == 0`             | `cld` (o por defecto)      |
| 1  | `R09 += 1`             | `R09 >= R10`           | `std`                      |
