# LOADS - Carga con autoincremente (Load String)

`LOADS` es una instruccion que **lee un valor de la memoria VM** y luego **automaticamente
avanza el puntero** al siguiente elemento. Se usa tipicamente para procesar buffers, cadenas
de texto o arrays elemento por elemento, sin tener que escribir instrucciones manuales de
incremento despues de cada lectura.

Analogia: es como leer una cinta de casete. Cada vez que lees un trozo, la cabeza lectora
avanza automaticamente al siguiente trozo, lista para la siguiente lectura.

| Instruccion | Tamano de lectura | Incremento de R0 |
| :---------: | :---------------: | :--------------: |
| `loadsb`    | 1 byte (uint8)    | +1               |
| `loadsw`    | 2 bytes (uint16)  | +2               |
| `loadsd`    | 4 bytes (uint32)  | +4               |
| `loadsq`    | 8 bytes (uint64)  | +8               |

---

## Convencion de uso

`LOADS` usa dos registros fijos:

- **R0** = puntero al elemento actual (direccion VM donde leer). Se modifica automaticamente.
- **R1** = registro destino donde se escribe el valor leido.

```c
// Equivalente manual de loadsb:
mov    r1b, [r0]    // R1 = *(uint8_t*)R0  (leer 1 byte)
addu   r0, 1        // R0 += 1              (avanzar al siguiente)

// Con loadsb (1 instruccion hace lo mismo):
loadsb              // R1 = *(uint8_t*)R0, luego R0++
```

---

## Direction Flag (DF) - avanzar o retroceder

El sentido del avance lo controla el **Direction Flag (DF)** de la VM:

- `DF = 0` (por defecto): el puntero **incrementa** (avanza hacia delante). Es lo normal.
- `DF = 1`: el puntero **decrementa** (retrocede). Util para procesar buffers al reves.

Para cambiar el DF:
- `cld` - Clear Direction Flag: pone DF = 0 (avanzar)
- `std` - Set Direction Flag: pone DF = 1 (retroceder)

```c
// Procesar un buffer de 5 bytes de izquierda a derecha:
cld                             // DF = 0 (avanzar)
mov   r0, @Absolute("all.buf") // r0 = inicio del buffer

loadsb                          // R1 = buf[0], r0 avanza a buf[1]
loadsb                          // R1 = buf[1], r0 avanza a buf[2]
loadsb                          // R1 = buf[2], r0 avanza a buf[3]
loadsb                          // R1 = buf[3], r0 avanza a buf[4]
loadsb                          // R1 = buf[4], r0 avanza a buf[5]
```

---

## Ejemplo: leer una cadena de texto byte a byte

```c
// Buffer en la seccion de datos:
//   texto: .ascii "Hola\0"   (5 bytes: H, o, l, a, NUL)

code:
    mov   rsp, 0x00FF0000
    mov   rbp, 0x00FF0000
    cld                              // DF = 0 (izquierda a derecha)

    mov   r0, @Absolute("all.texto") // r0 = inicio de "Hola\0"

    loadsb    // R1 = 'H' (0x48), r0 -> 'o'
    loadsb    // R1 = 'o' (0x6F), r0 -> 'l'
    loadsb    // R1 = 'l' (0x6C), r0 -> 'a'
    loadsb    // R1 = 'a' (0x61), r0 -> '\0'
    loadsb    // R1 = '\0' (0x00), r0 -> despues del null

    hlt
```

---

## Ejemplo: copiar un array con bucle

```c
// Copiar un array de 8 bytes desde "src" a "dst"
//   src: .byte 1 2 3 4 5 6 7 8
//   dst: .byte 0 x8

code:
    mov   rsp, 0x00FF0000
    mov   rbp, 0x00FF0000
    cld

    mov   r0, @Absolute("all.src")   // r0 = inicio del array fuente
    mov   r5, @Absolute("all.dst")   // r5 = inicio del array destino
    mov   r9, 8                      // contador = 8 elementos

bucle:
    cmpu  r9, 0
    jmp.jeq fin               // si contador == 0, terminar

    loadsb                    // R1 = *r0, r0++
    mov    [r5], r1b          // *r5 = R1 (escribir en destino)
    addu   r5, 1              // r5++ (avanzar destino manualmente)
    subu   r9, 1              // contador--

    jmp.jmp bucle

fin:
    hlt
```

---

## Diferencia entre LOADS y MOV [mem]

| Criterio              | `mov r1b, [r0]` + `addu r0, 1` | `loadsb`                    |
| :-------------------- | :----------------------------- | :-------------------------- |
| Instrucciones         | 2                              | 1                           |
| Bytes de bytecode     | 4 + 10 = 14                    | 2                           |
| Direction flag        | No                             | Si (avanza o retrocede)     |
| Uso tipico            | Acceso aleatorio a arrays      | Procesamiento secuencial    |

`LOADS` es mas compacto para procesamiento secuencial. Para acceso aleatorio (saltar a
posiciones arbitrarias), usa `MOV` con calculo de direccion.

---

Ver tambien: [[MOV, MOVH, MOVC, MOCH]], [[LOOP]], [[cursor]]
