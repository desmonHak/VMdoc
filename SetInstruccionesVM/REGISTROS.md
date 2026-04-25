# Registros de VestaVM

Un **registro** es una cajita numerada dentro del procesador virtual donde se guardan valores
temporales mientras el programa trabaja con ellos. Piensa en los registros como variables
ultrarapidas: acceder a un registro es instantaneo, mucho mas rapido que acceder a la memoria.

VestaVM tiene varios tipos de registros: generales, especiales, cursores y de banderas.

---

## Registros generales (r0 - r15)

Son los 16 registros de proposito general. Cada uno puede guardar un numero entero de hasta
64 bits (un valor entre 0 y 18,446,744,073,709,551,615). Se llaman r0, r1, r2, ... hasta r15.

Lo especial es que cada registro tiene acceso a partes mas pequenas de si mismo:

| Registro 64 bits | 32 bits bajos | 16 bits bajos | 8 bits bajos |
| :--------------: | :-----------: | :-----------: | :----------: |
| r0               | r0d           | r0w           | r0b          |
| r1               | r1d           | r1w           | r1b          |
| r2               | r2d           | r2w           | r2b          |
| r3               | r3d           | r3w           | r3b          |
| r4               | r4d           | r4w           | r4b          |
| r5               | r5d           | r5w           | r5b          |
| r6               | r6d           | r6w           | r6b          |
| r7               | r7d           | r7w           | r7b          |
| r8               | r8d           | r8w           | r8b          |
| r9               | r9d           | r9w           | r9b          |
| r10              | r10d          | r10w          | r10b         |
| r11              | r11d          | r11w          | r11b         |
| r12              | r12d          | r12w          | r12b         |
| r13              | r13d          | r13w          | r13b         |
| r14              | r14d          | r14w          | r14b         |
| r15              | r15d          | r15w          | r15b         |

- El sufijo **d** (dword) accede a los 32 bits menos significativos.
- El sufijo **w** (word) accede a los 16 bits menos significativos.
- El sufijo **b** (byte) accede a los 8 bits menos significativos.

Por ejemplo, si r0 contiene el valor `0x0000000012345678`:
- `r0`  (64 bits) = 0x0000000012345678
- `r0d` (32 bits) = 0x12345678
- `r0w` (16 bits) = 0x5678
- `r0b` ( 8 bits) = 0x78

Escribir en `r0d` rellena los 32 bits superiores con cero automaticamente.

### Codificacion de registros en instrucciones

Cada registro general tiene un codigo de 4 bits para su identificacion en la codificacion
binaria de las instrucciones:

```c
// Codigo de registro (4 bits):
// r0  = 0b0000 = 0
// r1  = 0b0001 = 1
// r2  = 0b0010 = 2
// ...
// r15 = 0b1111 = 15

// El modo de tamano se codifica con 2 bits adicionales:
// 0b00 = byte  (8 bits)
// 0b01 = word  (16 bits)
// 0b10 = dword (32 bits)
// 0b11 = qword (64 bits)

// Ejemplo: r1 en modo byte  = 0b00_0001 = r1b
// Ejemplo: r1 en modo word  = 0b01_0001 = r1w
// Ejemplo: r1 en modo dword = 0b10_0001 = r1d
// Ejemplo: r1 en modo qword = 0b11_0001 = r1  (64 bits)
```

---

## Registros especiales

Estos registros tienen un proposito fijo y no todos los usos son intercambiables con los
registros generales. Se codifican con 6 bits en lugar de 4.

| Registro  | Codificacion | Descripcion                                          |
| :-------: | :----------: | :--------------------------------------------------- |
| `cur0`    | 0b000000     | Cursor 0: puntero a memoria host (ver abajo)         |
| `cur1`    | 0b000001     | Cursor 1: puntero a memoria host                     |
| `cur2`    | 0b000010     | Cursor 2: puntero a memoria host                     |
| `cur3`    | 0b000011     | Cursor 3: puntero a memoria host                     |
| `rip`     | 0b001000     | Instruction Pointer: direccion de la instruccion actual |
| `rbp`     | 0b001001     | Base Pointer: base del frame de pila actual          |
| `rsp`     | 0b001010     | Stack Pointer: tope de la pila                       |
| `rflags`  | 0b001011     | Flags: resultado de la ultima comparacion u operacion |

### RIP (Instruction Pointer)

Contiene la direccion de la **proxima instruccion** a ejecutar. La VM lo actualiza
automaticamente despues de cada instruccion. Las instrucciones de salto (JMP, Jcc) modifican
RIP directamente para cambiar el flujo del programa.

No se debe modificar RIP directamente en codigo normal; se usan JMP/CALLVM/RET para eso.

### RSP (Stack Pointer)

Apunta al **tope de la pila**. La pila crece hacia abajo en VestaVM: cuando se empuja un
valor (PUSH), RSP se decrementa en el tamano del valor. Cuando se saca (POP), RSP se incrementa.

```c
// La pila crece hacia abajo:
// Direccion alta: [rbp + 8]  = return address (antes del frame)
//                [rbp + 0]  = saved rbp (guardado por ENTER)
//                [rbp - 8]  = primera variable local
//                [rbp - 16] = segunda variable local
// Direccion baja: [rsp]     = tope actual de la pila
```

### RBP (Base Pointer)

Apunta a la **base del frame actual**. ENTER lo establece y LEAVE lo restaura. Se usa para
acceder a variables locales y argumentos con desplazamientos fijos desde RBP.

---

## Registros cursor (cur0 - cur3)

Los cursores son registros especiales del tamano de un puntero nativo del sistema operativo
(8 bytes en 64 bits). Se usan **exclusivamente** para acceder a memoria del host (la memoria
real del proceso, no la memoria virtual de la VM).

La diferencia entre cursores y registros generales:
- Los **registros generales** (r0-r15) trabajan con la memoria virtual de la VM.
- Los **cursores** trabajan con la memoria fisica del proceso del sistema operativo.

Esto es importante para la interoperabilidad con codigo nativo (FFI).

```c
// Patron tipico: acceder a un puntero host guardado en r1
// SIN destruir r1:
mov   r14, r1           // r14 = copia de r1 (por si acaso)
xchg  cur0, r14         // cur0 = r1 (el puntero host), r14 = valor anterior del cursor
// ATENCION: r14 ahora contiene el valor viejo del cursor, no r1!
// Por eso guardamos r1 en r14 ANTES del xchg si lo necesitamos despues.

readcur r2, cur0        // r2 = *puntero (leer 8 bytes del host)
writecur cur0, r5       // *puntero = r5 (escribir en el host)
writecur cur0, r5d      // *puntero = r5d (escribir 4 bytes)
writecur cur0, r5w      // *puntero = r5w (escribir 2 bytes)
writecur cur0, r5b      // *puntero = r5b (escribir 1 byte)
```

Cuando un cursor tiene valor 0 (`NULL`) y se intenta acceder a el, el comportamiento es
indefinido (similar a un null pointer dereference en C).

---

## Registro de banderas (rflags)

Guarda el resultado de las operaciones aritmeticas y comparaciones. Cada bit es una bandera
independiente:

| Bandera | Bit | Nombre         | Se activa cuando...                                |
| :-----: | :-: | :------------- | :------------------------------------------------- |
| ZF      |  1  | Zero Flag      | El resultado fue exactamente cero                  |
| SF      |  0  | Sign Flag      | El resultado fue negativo (bit mas significativo = 1) |
| CF      |  2  | Carry Flag     | Hubo acarreo (overflow en aritmetica sin signo)    |
| OF      |  3  | Overflow Flag  | Hubo overflow (resultado no cabe como numero con signo) |
| DM      |  4  | Distributed Mode | La VM esta en modo distribuido (red)             |

```c
// Ejemplo de como las instrucciones actualizan los flags:
mov  r1, 5
mov  r2, 5
cmpu r1, r2     // 5 - 5 = 0: ZF=1, SF=0, CF=0, OF=0

mov  r1, 3
mov  r2, 7
cmpu r1, r2     // 3 - 7 con unsigned: CF=1 (borrow), ZF=0

mov  r1, 10
addu r1, 1      // r1 = 11: ZF=0, SF=0, CF=0, OF=0

// Los saltos condicionales (jmp.je, jmp.jne, etc.) leen estos flags.
```

---

## Registro RP (Register Page)

El registro RP gestiona el acceso a paginas de memoria. Esta compuesto por dos campos:

```c
// Estructura del registro RP (64 bits total):
// bits  0-11  = DP  (Disp Page): desplazamiento dentro de la pagina (12 bits)
// bits 12-63  = IDP (ID Page):   identificador de la pagina (52 bits)

// Equivale a: RP = (IDP << 12) | DP

// Ejemplo:
// RP = 0x0000000000000000 => IDP = 0x0000000000000, DP = 0x000 // pagina 0
// RP = 0x0000000000001000 => IDP = 0x0000000000001, DP = 0x000 // pagina 1
// RP = 0x0000000000001ABC => IDP = 0x0000000000001, DP = 0xABC // pagina 1, offset 0xABC
```

Cada pagina tiene 4096 bytes (4 KB). El DP da acceso a los 4096 bytes de la pagina actual.
Para acceder a otra pagina se cambia el IDP.

Ver tambien: [[PAGEINFO]] para obtener informacion de una pagina, [[RESBP]] para reservar
nuevas paginas.

---

## Convencion de uso de registros

Por convencion (no obligacion en modo raw, pero si recomendada para interoperabilidad):

| Registro(s) | Rol                    | Quien lo preserva     |
| :---------: | :--------------------- | :-------------------- |
| r0          | Valor de retorno       | -                     |
| r1 - r12    | Argumentos de entrada  | El llamante           |
| r13 - r15   | Callee-saved           | La funcion llamada    |
| rsp         | Puntero de pila        | Siempre valido        |
| rbp         | Base del frame         | La funcion llamada    |
| r14         | Scratch para cursores  | Por convencion        |

"Callee-saved" significa que si una funcion usa r13, r14 o r15, debe guardar sus valores
al inicio y restaurarlos al final antes de retornar.
