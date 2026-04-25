# CALLVM / CALLVMR - Llamar a una funcion

La instruccion **CALLVM** llama a una funcion: empuja la direccion de retorno en la pila y
salta al codigo de la funcion destino. Cuando la funcion ejecuta [[RET]], la ejecucion
vuelve a la instruccion siguiente despues del CALLVM.

| Instruccion      | opcode | byte2 | destino          | Tamano   | Descripcion               |
| :--------------: | :----: | :---: | :--------------: | :------: | :------------------------ |
| `callvm addr`    | 0x10   | 0x00  | addr de 64 bits  | 10 bytes | Llamar a direccion absoluta |
| `callvmr reg`    | 0x16   | reg   | -                | 2 bytes  | Llamar a direccion en registro |

Ver tambien: [[RET]], [[ENTER]], [[ConvencionDeLlamadas]]

---

## Para que sirve

Cuando un programa necesita ejecutar una tarea que puede reutilizarse, se separa en una
funcion. CALLVM es la instruccion que "llama" a esa funcion.

Analogia: CALLVM es como hacer una llamada telefonica. Dejas anotado tu numero de telefono
(la direccion de retorno en la pila) para que la persona (la funcion) sepa a donde devolverte
la llamada cuando termine.

---

## Comportamiento de CALLVM

Cuando se ejecuta `callvm mi_funcion`:
1. Calcula `ret_addr = RIP + tamano_instruccion` (la siguiente instruccion despues del callvm).
2. `RSP -= 8` (hace espacio en la pila).
3. `[RSP] = ret_addr` (guarda la direccion de retorno).
4. `RIP = addr` (salta a la funcion).

```c
// En el llamante:
callvm mi_funcion   // 1. Calcula ret_addr (donde continuar despues)
                    // 2. Empuja ret_addr en la pila (RSP -= 8)
                    // 3. Salta a mi_funcion

// Cuando mi_funcion ejecute RET:
// 1. Lee ret_addr = [RSP]
// 2. RSP += 8
// 3. RIP = ret_addr (vuelve aqui)
mov r2, r0          // usar el resultado de mi_funcion
```

---

## Variantes

### CALLVM con direccion absoluta (callvm addr)

La direccion de la funcion se codifica directamente en la instruccion:

```c
callvm 0x1000       // llama a la funcion en la direccion 0x1000

// Con etiqueta (el ensamblador la resuelve automaticamente):
callvm mi_funcion   // el ensamblador sustituye mi_funcion por su direccion
```

### CALLVMR con direccion en registro (callvmr reg)

La direccion esta en un registro, util para llamadas indirectas o a funciones calculadas
en tiempo de ejecucion:

```c
mov   r1, @Absolute("code.mi_funcion")  // obtener la direccion de la funcion
callvmr r1                               // llamar a la direccion en r1

// Tambien util para tablas de funciones:
mov   r2, tabla_de_funciones
addu  r2, r3           // r2 = tabla_de_funciones + indice * 8
mov   r4, [r2]         // r4 = direccion de la funcion
callvmr r4             // llamar a la funcion indirecta
```

---

## Convencion de llamada

Para que las funciones sean interoperables, se sigue esta convencion:

| Registro(s) | Rol antes de CALLVM    | Rol despues (al retornar) |
| :---------: | :--------------------- | :------------------------ |
| r0          | (libre)                | Valor de retorno de la funcion |
| r1          | Argumento 1            | (puede haber cambiado)    |
| r2          | Argumento 2            | (puede haber cambiado)    |
| ...         | ...                    | ...                       |
| r12         | Argumento 12           | (puede haber cambiado)    |
| r13         | (callee-saved)         | Preservado por la funcion |
| r14         | (callee-saved)         | Preservado por la funcion |
| r15         | (callee-saved / argc)  | Preservado por la funcion |

```c
// Ejemplo: llamar a una funcion con 3 argumentos
mov   r1, 10        // argumento 1
mov   r2, 20        // argumento 2
mov   r3, 30        // argumento 3
callvm calcular     // llamar a calcular(10, 20, 30)
                    // r0 = resultado de la funcion
mov   r5, r0        // guardar el resultado
```

---

## Codificacion binaria

### CALLVM con direccion absoluta (10 bytes)

```
+--------+--------+----------------------------------+
| 0x10   | 0x00   | direccion de 64 bits (8 bytes)   |
+--------+--------+----------------------------------+
  byte0    byte1    bytes 2-9
```

### CALLVMR con registro (2 bytes)

```
+--------+--------+
| 0x16   | reg    |
+--------+--------+
  byte0    byte1
```

El campo `reg` usa el mismo formato que [[PUSH]]/[[POP]]:
- Bit 7: 0=registro general, 1=registro especial
- Bits 5-4: modo de tamano (solo si reg general)
- Bits 3-0: codigo del registro

---

## Ejemplo completo: funcion con argumentos y retorno

```c
@Format("raw")
@SpaceAddress { @Name("mem"), @IniAddress(0x0), @EndAddress(0xFFFFFFFFFFFFFFFF) }
@Section { @Name("code"), @SpaceAddress("mem") @Align(0x1000) }

code:
    mov rsp, 0x00FF0000
    mov rbp, 0x00FF0000

    // Llamar a potencia(2, 8) = 256
    mov   r1, 2         // base
    mov   r2, 8         // exponente
    callvm potencia     // r0 = 2^8 = 256
    // r0 = 256
    hlt

// Funcion: calcula base^exp (solo exponentes positivos)
// r1 = base, r2 = exp
// r0 = resultado
potencia:
    enter 0
    mov   r0, 1         // resultado inicial = 1
    cmpu  r2, 0
    jmp.je fin_potencia // si exp == 0, retornar 1

bucle_potencia:
    mulu  r0, r1        // resultado *= base
    subu  r2, 1         // exp--
    cmpu  r2, 0
    jmp.jne bucle_potencia  // si exp != 0, continuar

fin_potencia:
    leave
    ret

end_code:
```

---

## Diferencia entre CALLVM y JMP

| Instruccion | Guarda ret_addr | Puede volver | Uso tipico          |
| :---------: | :-------------: | :----------: | :------------------ |
| `callvm`    | Si (en pila)    | Si (con RET) | Llamar a funciones  |
| `jmp`       | No              | No           | Saltar a una etiqueta |

CALLVM = llamar con posibilidad de volver. JMP = saltar sin guardar donde estabas.
