
|     Operación      | **Sin signo** | **Con signo** | No aplica |
| :----------------: | :-----------: | :-----------: | :-------: |
|      **Suma**      |    `ADDU`     |    `ADDS`     |           |
|     **Resta**      |    `SUBU`     |    `SUBS`     |           |
| **Multiplicación** |    `MULU`     |    `MULS`     |           |
|    **División**    |    `DIVU`     |    `DIVS`     |           |
|  **Comparación**   |    `CMPU`     |    `CMPS`     |           |
|    Incrementar     |               |               |    INC    |
|    Decrementar     |               |               |    DEC    |

# Nomenclaruta
- ``reg1``, ``reg2``: se usa 4bits para expresar cada registro.

# Modos de direccionamiento
Actualmente solo hay dos modos de direccionamiento, un modo básico de reg-reg
## Modo Basico

El modo básico agrupa el direccionamiento de registro a registro, de memoria a registro y de registro a memoria, sirve para operaciones básicas e intercambio.
## Modo SIB (Scalar + Indice * Base)
Este modo de acceso sirve principalmente para el acceso a array y estructuras de tipo indice

![[direccionamiento.png]]
# peculiaridades de las instrucciones aritméticas y lógicas
La mayoría de instrucciones de este tipo que operan con memoria, usan solo ``5bytes * 8 = 40bits`` para acceder a memoria, por tanto ``0x000000000000 - 0xFFFFFFFFFFFF`` es su rango de operación (Si el Registro de paginado (`RP`) esta configurado en 0).
Sin embargo, estas pueden usar un registro de ``64bits``(`RP`) para acceder a otras partes de la memoria si es que esto fuera realmente necesario.

# INC
Permite incrementar un registro en 1

| Instrucción | opcode1 | opcode2 |    flag    | byte (relleno o extensión o registro) | total bytes |
| :---------: | :-----: | :-----: | :--------: | :-----------------------------------: | :---------: |
|     INC     |   0x0   |   0x4   | 0b00000000 |           ``0b0000``  `reg`           |      4      |
# DEC
Permite decrementar un registro en 1

| Instrucción | opcode1 | opcode2 |    flag    | byte (relleno o extensión o registro) | total bytes |
| :---------: | :-----: | :-----: | :--------: | :-----------------------------------: | :---------: |
|     DEC     |   0x0   |   0x4   | 0b00000000 |            ``0b0001  `reg`            |      4      |

# ADD - Con signo (S) y sin signo (U)

| Instrucción                          | opcode1 | opcode2 |     1byte     |       1byte       | 1byte | 1byte | 1byte | 1byte | total bytes |
| :----------------------------------- | :-----: | :-----: | :-----------: | :---------------: | :---: | :---: | :---: | :---: | :---------: |
| ``ADDU reg1, reg2``                  |   0x0   |   0x5   |  0b00000000   | ``reg2`` ``reg1`` |       |       |       |       |      4      |
| ``ADDU reg1, [index*scalar + base]`` |   0x0   |   0x5   | 0b00001`reg1` |      ``SIB``      |       |       |       |       |      4      |
| ``ADDU [index*scalar + base], reg1`` |   0x0   |   0x5   | 0b00010`reg1` |      ``SIB``      |       |       |       |       |      4      |
| ``ADDU reg1, [mem]``                 |   0x0   |   0x5   | 0b10000`reg1` |       0xFF        | 0xFF  | 0xFF  | 0xFF  | 0xFF  |      8      |
| ``ADDU [mem], reg1``                 |   0x0   |   0x5   | 0b10001`reg1` |       0xFF        | 0xFF  | 0xFF  | 0xFF  | 0xFF  |      8      |
| ``ADDS reg1, reg2``                  |   0x0   |   0x5   |  0b01000000   | ``reg2`` ``reg1`` |       |       |       |       |      4      |
| ``ADDS reg1, [index*scalar + base]`` |   0x0   |   0x5   | 0b01001`reg1` |      ``SIB``      |       |       |       |       |      4      |
| ``ADDS [index*scalar + base], reg1`` |   0x0   |   0x5   | 0b01010`reg1` |      ``SIB``      |       |       |       |       |      4      |
| ``ADDS reg1, [mem]``                 |   0x0   |   0x5   | 0b11000`reg1` |       0xFF        | 0xFF  | 0xFF  | 0xFF  | 0xFF  |      8      |
| ``ADDS [mem], reg1``                 |   0x0   |   0x5   | 0b11001`reg1` |       0xFF        | 0xFF  | 0xFF  | 0xFF  | 0xFF  |      8      |



# SUB - Con signo (S) y sin signo (U)

| Instrucción                          | opcode1 | opcode2 |     1byte     |       1byte       | 1byte | 1byte | 1byte | 1byte | total bytes |
| :----------------------------------- | :-----: | :-----: | :-----------: | :---------------: | :---: | :---: | :---: | :---: | :---------: |
| ``SUBU reg1, reg2``                  |   0x0   |   0x6   |  0b00000000   | ``reg2`` ``reg1`` |       |       |       |       |      4      |
| ``SUBU reg1, [index*scalar + base]`` |   0x0   |   0x6   | 0b00001`reg1` |      ``SIB``      |       |       |       |       |      4      |
| ``SUBU [index*scalar + base], reg1`` |   0x0   |   0x6   | 0b00010`reg1` |      ``SIB``      |       |       |       |       |      4      |
| ``SUBU reg1, [mem]``                 |   0x0   |   0x6   | 0b10000`reg1` |       0xFF        | 0xFF  | 0xFF  | 0xFF  | 0xFF  |      8      |
| ``SUBU [mem], reg1``                 |   0x0   |   0x6   | 0b10001`reg1` |       0xFF        | 0xFF  | 0xFF  | 0xFF  | 0xFF  |      8      |
| ``SUBS reg1, reg2``                  |   0x0   |   0x6   |  0b01000000   | ``reg2`` ``reg1`` |       |       |       |       |      4      |
| ``SUBS reg1, [index*scalar + base]`` |   0x0   |   0x6   | 0b01001`reg1` |      ``SIB``      |       |       |       |       |      4      |
| ``SUBS [index*scalar + base], reg1`` |   0x0   |   0x6   | 0b01010`reg1` |      ``SIB``      |       |       |       |       |      4      |
| ``SUBS reg1, [mem]``                 |   0x0   |   0x6   | 0b11000`reg1` |       0xFF        | 0xFF  | 0xFF  | 0xFF  | 0xFF  |      8      |
| ``SUBS [mem], reg1``                 |   0x0   |   0x6   | 0b11001`reg1` |       0xFF        | 0xFF  | 0xFF  | 0xFF  | 0xFF  |      8      |


# MUL - Con signo (S) y sin signo (U)

| Instrucción                          | opcode1 | opcode2 |     1byte     |       1byte       | 1byte | 1byte | 1byte | 1byte | total bytes |
| :----------------------------------- | :-----: | :-----: | :-----------: | :---------------: | :---: | :---: | :---: | :---: | :---------: |
| ``MULU reg1, reg2``                  |   0x0   |   0x7   |  0b00000000   | ``reg2`` ``reg1`` |       |       |       |       |      4      |
| ``MULU reg1, [index*scalar + base]`` |   0x0   |   0x7   | 0b00001`reg1` |      ``SIB``      |       |       |       |       |      4      |
| ``MULU [index*scalar + base], reg1`` |   0x0   |   0x7   | 0b00010`reg1` |      ``SIB``      |       |       |       |       |      4      |
| ``MULU reg1, [mem]``                 |   0x0   |   0x7   | 0b10000`reg1` |       0xFF        | 0xFF  | 0xFF  | 0xFF  | 0xFF  |      8      |
| ``MULU [mem], reg1``                 |   0x0   |   0x7   | 0b10001`reg1` |       0xFF        | 0xFF  | 0xFF  | 0xFF  | 0xFF  |      8      |
| ``MULS reg1, reg2``                  |   0x0   |   0x7   |  0b01000000   | ``reg2`` ``reg1`` |       |       |       |       |      4      |
| ``MULS reg1, [index*scalar + base]`` |   0x0   |   0x7   | 0b01001`reg1` |      ``SIB``      |       |       |       |       |      4      |
| ``MULS [index*scalar + base], reg1`` |   0x0   |   0x7   | 0b01010`reg1` |      ``SIB``      |       |       |       |       |      4      |
| ``MULS reg1, [mem]``                 |   0x0   |   0x7   | 0b11000`reg1` |       0xFF        | 0xFF  | 0xFF  | 0xFF  | 0xFF  |      8      |
| ``MULS [mem], reg1``                 |   0x0   |   0x7   | 0b11001`reg1` |       0xFF        | 0xFF  | 0xFF  | 0xFF  | 0xFF  |      8      |


# DIV - Con signo (S) y sin signo (U)

| Instrucción                          | opcode1 | opcode2 |     1byte     |       1byte       | 1byte | 1byte | 1byte | 1byte | total bytes |
| :----------------------------------- | :-----: | :-----: | :-----------: | :---------------: | :---: | :---: | :---: | :---: | :---------: |
| ``DIVU reg1, reg2``                  |   0x0   |   0x7   |  0b00000000   | ``reg2`` ``reg1`` |       |       |       |       |      4      |
| ``DIVU reg1, [index*scalar + base]`` |   0x0   |   0x7   | 0b00001`reg1` |      ``SIB``      |       |       |       |       |      4      |
| ``DIVU [index*scalar + base], reg1`` |   0x0   |   0x7   | 0b00010`reg1` |      ``SIB``      |       |       |       |       |      4      |
| ``DIVU reg1, [mem]``                 |   0x0   |   0x7   | 0b10000`reg1` |       0xFF        | 0xFF  | 0xFF  | 0xFF  | 0xFF  |      8      |
| ``DIVU [mem], reg1``                 |   0x0   |   0x7   | 0b10001`reg1` |       0xFF        | 0xFF  | 0xFF  | 0xFF  | 0xFF  |      8      |
| ``DIVS reg1, reg2``                  |   0x0   |   0x7   |  0b01000000   | ``reg2`` ``reg1`` |       |       |       |       |      4      |
| ``DIVS reg1, [index*scalar + base]`` |   0x0   |   0x7   | 0b01001`reg1` |      ``SIB``      |       |       |       |       |      4      |
| ``DIVS [index*scalar + base], reg1`` |   0x0   |   0x7   | 0b01010`reg1` |      ``SIB``      |       |       |       |       |      4      |
| ``DIVS reg1, [mem]``                 |   0x0   |   0x7   | 0b11000`reg1` |       0xFF        | 0xFF  | 0xFF  | 0xFF  | 0xFF  |      8      |
| ``DIVS [mem], reg1``                 |   0x0   |   0x7   | 0b11001`reg1` |       0xFF        | 0xFF  | 0xFF  | 0xFF  | 0xFF  |      8      |


# CMP - Con signo (S) y sin signo (U)
| Flag              | **Condición**          | **Uso típico**       |
| ----------------- | ---------------------- | -------------------- |
| **ZF** (Zero)     | `op1 == op2`           | `JE/JZ`, `JNE/JNZ`   |
| **CF** (Carry)    | `op1 < op2` (unsigned) | `JB/JC`, `JAE/JNC`   |
| **SF** (Sign)     | `op1 < op2` (signed)   | `JL/JNGE`, `JGE/JNL` |
| **OF** (Overflow) | Overflow signed        | `JO`, `JNO`          |

```c
struct Flags {
    bool ZF;  // Zero Flag     -> JE/JNE
    bool CF;  // Carry Flag    -> JB/JAE (unsigned)
    bool SF;  // Sign Flag     -> JL/JGE (signed)  
    bool OF;  // Overflow Flag -> JO/JNO (signed)
};
```

## **1. Saltos por IGUALDAD (ZF)**

| Instrucción | **Condición** | **Uso**           | **Assembly**           |
| ----------- | ------------- | ----------------- | ---------------------- |
| `JE/JZ`     | `ZF=1`        | Igual/Cero        | `cmp r1,r2; je ok`     |
| `JNE/JNZ`   | `ZF=0`        | Diferente/No cero | `cmp r1,r2; jne error` |
## **2. Saltos UNSIGNED (CF)**

| Instrucción | **Condición**  | **Uso**                | **Assembly**            |
| ----------- | -------------- | ---------------------- | ----------------------- |
| `JB/JC`     | `CF=1`         | Menor (unsigned)       | `cmpu r1,r2; jb menor`  |
| `JAE/JNC`   | `CF=0`         | Mayor/Igual (unsigned) | `cmpu r1,r2; jae mayor` |
| `JBE/JNA`   | CF=1 \|\| ZF=1 | Menor/Igual (unsigned) | `cmpu r1,r2; jbe mayor` |
| `JA/JNBE`   | `CF=0 && ZF=0` | Mayor (unsigned)       | `cmpu r1,r2; ja loop`   |
## **3. Saltos SIGNED (SF, OF)**
| Instrucción | **Condición**      | **Uso**              | **Assembly**          |
| ----------- | ------------------ | -------------------- | --------------------- |
| `JL/JNGE`   | `SF ≠ OF`          | Menor (signed)       | `cmps r1,r2; jl neg`  |
| `JGE/JNL`   | `SF == OF`         | Mayor/Igual (signed) | `cmps r1,r2; jge pos` |
| `JLE/JNG`   | CF=1 \|\| SF≠OF    | Menor/Igual (signed) | `cmps r1,r2; jle pos` |
| `JG/JNLE`   | `ZF=0 && SF == OF` | Mayor (signed)       | `cmps r1,r2; jg max`  |
## **4. Saltos especiales (OF)**
| Instrucción | **Condición** | **Uso**         | **Assembly**           |
| ----------- | ------------- | --------------- | ---------------------- |
| `JO`        | `OF=1`        | Overflow signed | `cmps r1,r2; jo error` |
| `JNO`       | `OF=0`        | No overflow     | `cmps r1,r2; jno ok`   |
## **5. Salto incondicional**
| Instrucción | **Condición** | **Assembly** |
| ----------- | ------------- | ------------ |
| [[JMP]]     | Siempre       | `jmp loop`   |

| Instrucción                          | opcode1 | opcode2 |     1byte     |       1byte       | 1byte | 1byte | 1byte | 1byte | total bytes |
| :----------------------------------- | :-----: | :-----: | :-----------: | :---------------: | :---: | :---: | :---: | :---: | :---------: |
| ``CMPU reg1, reg2``                  |   0x0   |   0x7   |  0b00000000   | ``reg2`` ``reg1`` |       |       |       |       |      4      |
| ``CMPU reg1, [index*scalar + base]`` |   0x0   |   0x7   | 0b00001`reg1` |      ``SIB``      |       |       |       |       |      4      |
| ``CMPU [index*scalar + base], reg1`` |   0x0   |   0x7   | 0b00010`reg1` |      ``SIB``      |       |       |       |       |      4      |
| ``CMPU reg1, [mem]``                 |   0x0   |   0x7   | 0b10000`reg1` |       0xFF        | 0xFF  | 0xFF  | 0xFF  | 0xFF  |      8      |
| ``CMPU [mem], reg1``                 |   0x0   |   0x7   | 0b10001`reg1` |       0xFF        | 0xFF  | 0xFF  | 0xFF  | 0xFF  |      8      |
| ``CMPS reg1, reg2``                  |   0x0   |   0x7   |  0b01000000   | ``reg2`` ``reg1`` |       |       |       |       |      4      |
| ``CMPS reg1, [index*scalar + base]`` |   0x0   |   0x7   | 0b01001`reg1` |      ``SIB``      |       |       |       |       |      4      |
| ``CMPS [index*scalar + base], reg1`` |   0x0   |   0x7   | 0b01010`reg1` |      ``SIB``      |       |       |       |       |      4      |
| ``CMPS reg1, [mem]``                 |   0x0   |   0x7   | 0b11000`reg1` |       0xFF        | 0xFF  | 0xFF  | 0xFF  | 0xFF  |      8      |
| ``CMPS [mem], reg1``                 |   0x0   |   0x7   | 0b11001`reg1` |       0xFF        | 0xFF  | 0xFF  | 0xFF  | 0xFF  |      8      |

|  Flag  | **Condición**          | **`op1 < op2`** | **`op1 = op2`** | **`op1 > op2`** |
| :----: | :--------------------- | :-------------: | :-------------: | :-------------: |
| **ZF** | `(op1 - op2) == 0`     |       `0`       |     **`1`**     |       `0`       |
| **CF** | `op1 < op2` (unsigned) |     **`1`**     |       `0`       |       `0`       |
| **SF** | `NO usado`             |       `-`       |       `-`       |       `-`       |
| **OF** | `NO usado`             |       `-`       |       `-`       |       `-`       |
```c
void cmpu(uint32_t op1, uint32_t op2) {
    uint32_t result = op1 - op2;
    flags.ZF = (result == 0);
    flags.CF = (op1 < op2);  // Carry si underflow unsigned
    // SF y OF = NO modificados o definidos
}
```

|  Flag  | **Condición**          | **`op1 < op2`** | **`op1 = op2`** | **`op1 > op2`** |
| :----: | :--------------------- | :-------------: | :-------------: | :-------------: |
| **ZF** | `(op1 - op2) == 0`     |       `0`       |     **`1`**     |       `0`       |
| **CF** | `op1 < op2` (unsigned) |      `1*`       |       `0`       |       `0`       |
| **SF** | `MSB(result) == 1`     |     **`1`**     |       `0`       |       `0`       |
| **OF** | `Overflow signed`      |   **`varía`**   |       `0`       |   **`varía`**   |
```c
void cmps(uint32_t op1, uint32_t op2) {
    int32_t s1 = (int32_t)op1;
    int32_t s2 = (int32_t)op2;
    int32_t result = s1 - s2;
    
    flags.ZF = (result == 0);
    flags.SF = (result < 0);                    // Sign bit
    
    // Overflow signed
    flags.OF = ((s1 ^ s2) & 0x80000000) &&  // CHECKEA SOLO BIT SIGNO (MSB)
               ((s1 ^ result) & 0x80000000);  
               
    flags.CF = (op1 < op2);                     // Unsigned carry también
}
```