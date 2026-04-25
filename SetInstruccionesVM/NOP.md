# NOP - No hacer nada

La instruccion **NOP** (no operation = no operacion) es la instruccion mas simple posible:
no hace absolutamente nada. El procesador la lee, la decodifica, y pasa a la siguiente
instruccion sin modificar ningun registro, flag ni memoria.

| Instruccion | opcode  | Tamano   | Descripcion    |
| :---------: | :-----: | :------: | :------------- |
| `nop`       | 0x33    | 1 byte   | No hacer nada (variante corta) |
| `nop2`      | 0x00 0x33 | 2 bytes | No hacer nada (variante larga) |

---

## Para que sirve una instruccion que no hace nada

Aunque parece inutil, NOP tiene varios usos practicos importantes:

### 1. Relleno de alineacion (padding)

El codigo a veces necesita estar alineado a multiples de 4, 8 o 16 bytes para que el
procesador lo ejecute mas eficientemente. Si el codigo natural deja un hueco, se rellenan
esos bytes con NOPs.

```c
// Supongamos que una etiqueta necesita estar en una direccion multiplo de 8:
// Tenemos 3 bytes de instrucciones y necesitamos llegar al siguiente multiplo de 8.
nop             // relleno: ocupa 1 byte extra
nop             // relleno: 2 bytes
nop             // relleno: 3 bytes
// Ahora la siguiente etiqueta esta alineada a 8 bytes:
funcion_alineada:
    mov r0, 42
    hlt
```

### 2. Reservar espacio para parchear codigo

A veces se reservan NOPs en el codigo para poder reemplazarlos mas tarde con instrucciones
reales sin tener que reubicar todo el codigo.

```c
// Espacio para una instruccion futura (2 bytes):
nop2    // sera reemplazado mas tarde si se necesita
nop2    // o quedara como NOP si no se necesita
```

### 3. Debugging y medicion de tiempos

Durante el desarrollo, un NOP puede usarse como punto de ruptura temporal o para insertar
retardos minimos medibles.

### 4. Sincronizacion en codigo auto-modificable

En arquitecturas donde el codigo puede modificarse a si mismo, una serie de NOPs crea una
"zona de aterrizaje" donde se puede escribir codigo sin afectar instrucciones vecinas.

---

## Codificacion binaria

### NOP de 1 byte

```
+--------+
| 0x33   |
+--------+
  byte0
```

### NOP de 2 bytes

```
+--------+--------+
| 0x00   | 0x33   |
+--------+--------+
  byte0    byte1
```

La variante de 2 bytes usa el prefijo extendido `0x00` seguido del opcode `0x33`.

---

## Garantias

- **No modifica ningun registro**: r0-r15, rip (avanza normalmente), rsp, rbp quedan igual.
- **No modifica rflags**: las banderas (ZF, SF, CF, OF, DM) no cambian.
- **No accede a memoria**: no lee ni escribe ninguna posicion de memoria.
- **Solo avanza RIP**: el puntero de instruccion avanza el tamano de la instruccion NOP (1 o 2 bytes).

---

## Ejemplo de uso en alineacion de seccion

```c
@Section { @Name("code"), @SpaceAddress("mem") @Align(0x1000) }

code:
    // codigo que ocupa, digamos, 7 bytes
    mov r0, 42          // 10 bytes (mov reg, imm64)
    hlt                 // 1 byte

    // Para que la siguiente funcion empiece en multiplo de 8:
    nop                 // relleno de 1 o 2 bytes segun necesidad
    nop2

funcion_alineada:       // ahora esta en multiplo de 8
    mov r0, 99
    hlt
end_code:
```
