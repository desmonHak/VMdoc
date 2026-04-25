# SUBSP y ADDSP - Aritmetica de puntero de pila con inmediato

`SUBSP` y `ADDSP` ajustan el puntero de pila (`RSP`) o el puntero base de frame (`RBP`)
sumando o restando un valor inmediato de 64 bits. Se introdujeron porque el codificador
de registros generales solo acepta `r0-r15`, haciendo imposible escribir `subu rsp, N`
o `addu rbp, N` de forma directa.

| Instruccion    | opcode0 | opcode1 | Modo       | Tamano             |
| :------------: | :-----: | :-----: | :--------: | :----------------: |
| `subsp`        |  0x00   |  0x2E   | MIXED_SIZE | 11 bytes (RSP/RBP) |
| `addsp`        |  0x00   |  0x2F   | MIXED_SIZE | 11 bytes (RSP/RBP) |

Implementacion: `src/runtime/exec_instruction_spimm.cpp`

---

## Para que sirve

La pila (stack) crece hacia abajo en VestaVM. Para reservar espacio para variables locales
en una funcion, se resta a RSP. Para liberarlo al terminar, se suma.

Sin SUBSP/ADDSP tendrias que usar `ENTER`/`LEAVE`, que combinan varias operaciones de pila.
SUBSP/ADDSP te dan control individual sobre RSP y RBP cuando necesitas mas precision.

```c
// Sin estas instrucciones (no funciona, rsp no es un registro general r0-r15):
subu rsp, 64    // ERROR: rsp no es un r0..r15

// Con SUBSP (correcto):
subsp rsp, 64   // RSP -= 64 (reservar 64 bytes en la pila)
```

---

## Instrucciones

```c
subsp  rsp, N    // RSP -= N  (reservar N bytes en la pila, bajando RSP)
addsp  rsp, N    // RSP += N  (liberar N bytes de la pila, subiendo RSP)

subsp  rbp, N    // RBP -= N  (ajustar el base pointer)
addsp  rbp, N    // RBP += N  (ajustar el base pointer)
```

---

## Semantica exacta

| Instruccion     | Efecto                         |
| :-------------- | :----------------------------- |
| `subsp rsp, N`  | `RSP = RSP - N` (64 bits)      |
| `addsp rsp, N`  | `RSP = RSP + N` (64 bits)      |
| `subsp rbp, N`  | `RBP = RBP - N` (64 bits)      |
| `addsp rbp, N`  | `RBP = RBP + N` (64 bits)      |

Las operaciones son aritmetica de 64 bits sin signo, igual que `subu`/`addu`.

---

## Patron de uso tipico: prologo y epilogo de funcion

El uso mas comun es reservar espacio para variables locales al entrar en una funcion
y liberarlo antes de retornar:

```c
mi_funcion:
    // Prologo: guardar rbp del llamante y crear frame propio
    push rbp            // guardar el frame del llamante
    mov  rbp, rsp       // RBP = cima actual (inicio del nuevo frame)

    // Reservar 64 bytes para variables locales
    subsp rsp, 64       // RSP -= 64

    // Ahora:
    // [rbp + 0] = saved_rbp del llamante (empujado por push rbp)
    // [rbp + 8] = return address (empujado por callvm antes de saltar aqui)
    // [rbp - 8]  hasta [rbp - 64] = espacio para variables locales (64 bytes)

    // ... cuerpo de la funcion ...
    // Las variables locales se acceden como [rsp + offset] o [rbp - offset]

    // Epilogo: liberar variables locales y restaurar rbp
    addsp rsp, 64       // RSP += 64  (liberar el espacio local)
    pop   rbp           // restaurar RBP del llamante
    ret
end_mi_funcion:
```

---

## Diferencia con ENTER y LEAVE

`ENTER N` equivale exactamente a estas tres instrucciones en secuencia:
```c
push rbp
mov  rbp, rsp
subsp rsp, N
```

Y `LEAVE` equivale a:
```c
addsp rsp, 0         // (o simplemente: mov rsp, rbp)
pop   rbp
```

| Criterio           | `ENTER N`                          | `push rbp + mov rbp,rsp + subsp rsp,N` |
| :----------------- | :--------------------------------- | :------------------------------------- |
| Guarda RBP         | Si (automatico)                    | Si (manual, pero equivalente)          |
| Reserva espacio    | Si (sub rsp, N)                    | Si                                     |
| Tamano instruccion | 10 bytes                           | 1 + 4 + 11 = 16 bytes                  |
| Flexibilidad       | Solo RSP, solo prologo completo    | RSP o RBP de forma independiente       |

Usa `ENTER` cuando quieras el prologo completo estandar. Usa `SUBSP`/`ADDSP` cuando
necesites ajustar RSP o RBP por separado en situaciones especiales.

---

## Codificacion binaria (MIXED_SIZE, siempre 11 bytes para RSP/RBP)

```
+--------+--------+--------+--------+--------+--------+--------+--------+--------+--------+--------+
| 0x00   | opcode | ctrl   | imm[0] | imm[1] | imm[2] | imm[3] | imm[4] | imm[5] | imm[6] | imm[7] |
+--------+--------+--------+--------+--------+--------+--------+--------+--------+--------+--------+
  byte0    byte1    byte2    byte3                                                           byte10
```

**byte2 (ctrl):**

```
bits 7-6 = modo   (siempre 0b11 = 3, porque RSP y RBP son registros de 64 bits)
bits 1-0 = sp_bp  (0 = RSP,  1 = RBP)
```

El inmediato ocupa siempre 8 bytes (little-endian). El tamano total es siempre 11 bytes:
2 bytes de opcode + 1 byte ctrl + 8 bytes inmediato.

Ejemplo: `subsp rsp, 64` (opcode 0x2E, RSP, valor 64 = 0x40):
```
0x00 0x2E 0xC0 0x40 0x00 0x00 0x00 0x00 0x00 0x00 0x00
               ctrl  imm64 (64 en little-endian)
```

---

## Consideraciones

- El valor `N` debe ser multiplo de 8 para mantener RSP alineado a 8 bytes. Las llamadas a
  funciones nativas esperan la pila alineada.
- `SUBSP`/`ADDSP` no comprueban si RSP cruza los limites de la pila VM. Es responsabilidad
  del programador mantener RSP en un rango valido.
- El valor inmediato es de 64 bits sin signo, permitiendo ajustes de pila muy grandes.

---

## Ejemplo completo

```c
@Format("raw")
@SpaceAddress { @Name("mem"), @IniAddress(0x0), @EndAddress(0xFFFFFFFFFFFFFFFF) }
@Section { @Name("code"), @SpaceAddress("mem") @Align(0x1000) }

code:
    mov  rsp, 0x00FF0000
    mov  rbp, 0x00FF0000

    mov  r1, 10
    mov  r2, 20
    callvm suma_manual

    // R0 = 30 (resultado)
    hlt

// Funcion que suma r1 + r2 y devuelve en r0
// Usa subsp/addsp manualmente en lugar de ENTER/LEAVE
suma_manual:
    push rbp            // guardar frame del llamante
    mov  rbp, rsp       // nuevo frame
    subsp rsp, 16       // reservar 2 variables locales de 8 bytes

    // Guardar r1 y r2 en variables locales (rbp-8 y rbp-16)
    mov  [rbp - 8], r1
    mov  [rbp - 16], r2

    // Calcular r1 + r2
    mov  r0, [rbp - 8]
    addu r0, [rbp - 16]

    // Epilogo manual
    addsp rsp, 16       // liberar las 2 variables locales
    pop   rbp           // restaurar rbp del llamante
    ret
```
