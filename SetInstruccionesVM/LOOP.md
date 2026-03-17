# LOOP

Permite realizar bucles de una forma mas comodoa, aunque esta accion se puede realizar sin el uso de esta instrucción.

| instrucción | opcode0 | opcode1 | extend_operations | extend_operations | desplazamiento de ``32bits`` | tamaño  |
| :---------: | :-----: | :-----: | :-----: | :-----: | :----------------: | :-----: |
|   LOOP      |  0x00   |  0x31   |  0x00   |  0x00   | disp32  | 8 bytes |

La instruccion al ser llamada, permite realizar un salto a un desplazamiento de 32bits si el counter (`R09`) no llego a 0 si ``DF = 0``. En caso de llegar a 0, el codigo no realizara el salto y el bucle no continuara.

En caso de ser ``DF = 1``, el bucle incrementara, el usuario debera o no, limpiar el registro ``r09`` y fijar la cantidad de iteraciones en ``r10``.

```c
if (DF == 0) {        // bucle decreciente
    R09--;
    if (R09 != 0)
        goto etiqueta;
} else {              // DF == 1 -> bucle creciente
    R09++;
    if (R09 < R10)    // R10 = límite superior
        goto etiqueta;
}
```

Ejemplo:
```c
cld            // DF = 0 -> bucle decreciente
mov r09, 5

loop_start:

    // hacer algo

    loop loop_start



mov r10, 5      // ahora el limite es 5
xor r09, r09
std             // DF = 1 -> bucle creciente, el registro R09 deberia ponerse a 0,
                // se usara el R09 como counter por lo que se debe limpiar

loop_start:

    // hacer algo

    loop loop_start

```

Este ``loop`` permite hacer estructuras similares al siguiente ``while``:

```c
// DF = 0
int counter = 5;

while (counter > 0) {


    counter -= 1;
}

// DF = 1
int limit = 5;
int counter = 0;
while (counter < limit) {


    counter += 1;
}
```