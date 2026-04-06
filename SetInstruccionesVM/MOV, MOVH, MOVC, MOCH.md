## MOV

Permite mover valores entre registros, necesita de [[LOADM]] y [[STOREM]] para cargar y dejar valores en la memoria. También permite mover valores inmediatos a registros.

## MOVH
Permite mover valores entre los registros de la VM y direcciones de host arbitrarias.
```c
// MOVH reg1, [index*scalar + base]
// MOVH [index*scalar + base], reg1

// permite leer 4 bytes de la direccion host resultante de la operacion
// r1 * 1 + r2
uint32_t r0w = *((uint32_t*)(((void*)r1) * 1) + r2)

MOVH r0w, [r1 * 1 + r2]

// ejemplo para leer memoria del host, por ejemplo la direccion 0x1f000
xor r1, r1
mov r2, 0x1f000
MOVH r0w, [r1 * 1 + r2] // r0w = *((uint32_t*)((0 * 1) + 0x1f000)
```
## MOVC
Permite mover valores si una codicion se da
```c
// MOVC dest, src, flag

// Copiar si el resultado anterior fue cero
CMP r1, r2
MOVC r3, r4, ZF  // si (r1 == r2) -> r3 = r4

// Selección condicional sin saltos
SUB r1, r2
MOVC r1, r3, ZF // si ((r1 - r2) == 0) -> r1 = r3

// Copiar si el resultado fue negativo
SUB r1, r2
MOVC r3, r4, SF // si r1 < r2 -> r3 = r4

// Implementar min(a, b)
CMP r1, r2
MOVC r1, r2, SF // si r1 < r2 -> r1 = r2 (max)

// Clamp negativo a cero
CMP r1, 0
MOVC r1, rZero, SF // si r1 < 0 → r1 = 0

// Copiar si hubo acarreo (útil para unsigned)
ADDU r1, r2
MOVC r3, r4, CF // si hubo carry → r3 = r4

// Detección de overflow en suma sin signo
ADDU rA, rB
MOVC rErr, rOne, CF // si CF=1 -> error

// Comparación unsigned
CMPU r1, r2
MOVC r3, r4, CF // si r1 < r2 (unsigned) -> r3 = r4

// Copiar si hubo overflow aritmético
ADDS r1, r2
MOVC rErr, rOne, OF // si overflow -> rErr = 1

// Saturación aritmética
ADDS r1, r2
MOVC r1, rMax, OF // si overflow -> r1 = MAX

// Detección de overflow en resta
SUBS rA, rB
MOVC rFlag, rOne, OF

// Copiar solo si estamos en modo distribuido
MOVC r1, r2, DM // si DM=1 -> r1 = r2

// Acceso a memoria condicional según modo
MOVC [rAddr], rVal, DM

// Selección booleana
CMP rCond, 1
MOVC rOut, rTrue, ZF
MOVC rOut, rFalse, NZ
```
## MOVCH
permite acceder a memoria del host bajo alguna clase de condicion.
**MOVCH** = _Conditional Move Host_
- `reg1` = destino
- `reg2` = fuente
- `flag` = (SF, ZF, CF, OF, DM)
Solo ejecuta la copia si el flag está activo.

```c
MOVCH reg1, [reg2], flag
MOVCH [reg1], reg2, flag
```


### Leer un valor host solo si el resultado anterior fue cero (ZF)
```c
CMP rA, rB
MOVCH r1, [r2], ZF // si rA == rB -> r1 = *(host_ptr = r2)
```
Útil para:
- cargar un valor alternativo
- seleccionar rutas sin saltos
- implementar “load if equal”

### Leer desde host solo si el resultado fue negativo (SF)
```c
SUB rX, rY
MOVCH r3, [r4], SF // si rX < rY -> r3 = *(host_ptr = r4)
```
Útil para:
- saturación
- clamps
- selección condicional

### Leer desde host solo si hubo carry (CF)
```c
ADDU r1, r2
MOVCH r3, [r4], CF // si hubo carry -> r3 = *(host_ptr = r4)
```
Útil para:
- aritmética sin signo
- detección de overflow unsigned

### Leer desde host solo si hubo overflow (OF)

### Leer desde host solo si estamos en modo distribuido (DM)
```c
MOVCH rThreadID, [rBase], DM

```
Útil para:
- sistemas paralelos
- modos especiales de ejecución
- lectura condicional de metadatos

----

## Codificación

Permite los mismos modos de codificación que las [[Aritmetica y logica|instrucciones arimetico logicas]] 

| Instrucción                          | opcode1 | opcode2 |           1byte           |       1byte       | 1byte | 1byte | 1byte |   4byte    | total bytes |
| :----------------------------------- | :-----: | :-----: | :-----------------------: | :---------------: | :---: | :---: | :---: | :--------: | :---------: |
| ``MOV reg1, reg2``                   |   0x0   |  0x14   |  0b`mode`0d0000<br>d = 0  | ``reg2`` ``reg1`` |       |       |       |            |      4      |
| TODO                                 |   0x0   |  0x14   | 0b`mode`0d`reg1`<br>d = 1 |                   |       |       |       |            |             |
|                                      |   0x0   |  0x14   |  0b`mode`1d0000<br>d = 0  |                   |       |       |       |            |      4      |
| TODO                                 |   0x0   |  0x14   | 0b`mode`1d`reg1`<br>d = 1 |                   |       |       |       |            |             |
| ``MOV reg1, inmmed``                 |   0x0   |  0x15   | 0b`mode`0d`reg1`<br>d = 0 |       0xFF        | 0xFF  | 0xFF  | 0xFF  | 0xFFFFFFFF |     10      |
| ``MOV inmmed, reg1``                 |   0x0   |  0x15   | 0b`mode`0d`reg1`<br>d = 1 |       0xFF        | 0xFF  | 0xFF  | 0xFF  | 0xFFFFFFFF |     10      |
| ``MOV [reg], inmmed``                |   0x0   |  0x15   | 0b`mode`1d`reg1`<br>d = 0 |       0xFF        | 0xFF  | 0xFF  | 0xFF  | 0xFFFFFFFF |     10      |
| `MOV reg, rip`                       |   0x0   |  0x15   | 0b`mode`1d`reg1`<br>d = 1 |                   |       |       |       |            |     10      |
| ``MOV reg1, [index*scalar + base]``  |   0x0   |  0x16   | 0b`mode`0d`reg1`<br>d = 0 |      ``SIB``      |       |       |       |            |      4      |
| ``MOV [index*scalar + base], reg1``  |   0x0   |  0x16   | 0b`mode`0d`reg1`<br>d = 1 |      ``SIB``      |       |       |       |            |      4      |
| ``MOVH reg1, [index*scalar + base]`` |   0x0   |  0x16   | 0b`mode`1d`reg1`<br>d = 0 |      ``SIB``      |       |       |       |            |      4      |
| ``MOVH [index*scalar + base], reg1`` |   0x0   |  0x16   | 0b`mode`1d`reg1`<br>d = 1 |      ``SIB``      |       |       |       |            |      4      |

>Posiblemente el introducir instrucciones de longitud variable dificulte implementar un modelo de "decodificación paralela estilo superscalar" en un futuro


| Instrucción                  | op1 | op2  | byte3 (mode/d/reg1) | byte4 (flag/reg2) | bytes |
| ---------------------------- | --- | ---- | ------------------- | ----------------- | ----- |
| ``MOVC reg1, [reg2], flag``  | 0x0 | 0x17 | 0b00 0 reg1         | 0b00 flag reg2    | 4     |
| ``MOVC [reg1], reg2, flag``  | 0x0 | 0x17 | 0b00 1 reg1         | 0b00 flag reg2    | 4     |
| ``MOVCH reg1, [reg2], flag`` | 0x0 | 0x17 | 0b10 0 reg1         | 0b00 flag reg2    | 4     |
| ``MOVCH [reg1], reg2, flag`` | 0x0 | 0x17 | 0b10 1 reg1         | 0b00 flag reg2    | 4     |
| ``MOVC reg1, reg2, flag``    | 0x0 | 0x18 | 0b00 0 reg1         | 0b00 flag reg2    | 4     |

permite mover a un registro dada una condicion
### **Byte 3 (mode/d/reg1)**


```
bit7-6 = mode
bit5   = d (0 = reg, 1 = [reg])
bit4-0 = reg1 (destino)
```

### **Byte 4**

```
bit7-5 = flag selector (3 bits -> 8 flags posibles)
bit4-0 = reg2 (fuente)
```

### **Total: 4 bytes**