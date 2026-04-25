# JUMPTABLE y TYPESWITCH - Despacho por valor y por tipo

Imagina que tienes que elegir entre 10 opciones segun el valor de una variable. Sin estas
instrucciones, tendrias que escribir 10 comparaciones y 10 saltos condicionales, uno tras
otro. Eso es lento y verboso.

`JUMPTABLE` y `TYPESWITCH` resuelven esto de forma directa:

- **JUMPTABLE**: dado un numero entero (0, 1, 2, ...), salta directamente al caso
  correspondiente en O(1), sin importar cuantos casos haya. Como un array de saltos.

- **TYPESWITCH**: dado un objeto GC, encuentra el caso que corresponde a su tipo exacto.
  Util para implementar `match`/`when` de lenguajes funcionales o patrones de visitante.

Analogia para JUMPTABLE: tienes un libro de recetas con indice. En lugar de leer pagina
por pagina hasta encontrar "ensalada", abres directamente la pagina 7 donde dice ensalada.

| Instruccion   | opcode0 | opcode1 | Modo | Tamano  | Descripcion                       |
| :-----------: | :-----: | :-----: | :--: | :-----: | :-------------------------------- |
| `jumptable`   |  0x00   |  0x27   | REG  | 4 bytes | Salto O(1) indexado por valor     |
| `typeswitch`  |  0x00   |  0x28   | REG  | 4 bytes | Salto O(n) indexado por tipo      |

Implementacion: `src/runtime/exec_instruction_pattern.cpp`

---

## JUMPTABLE - switch O(1) sobre un entero

### Sintaxis

```c
jumptable r_val, r_table, count
```

| Operando  | Descripcion                                                        |
| :-------: | :----------------------------------------------------------------- |
| `r_val`   | Registro con el valor entero (indice). Debe ser 0 .. count-1.     |
| `r_table` | Registro con la direccion VM del array de `uint64` de destinos.   |
| `count`   | Numero de entradas en la tabla (1-255, codificado en 1 byte).     |

### Semantica exacta

```c
if (r_val < count) {
    RIP = vm_memory[r_table + r_val * 8];  // salto O(1)
}
// si r_val >= count: no hace nada (cae a la instruccion siguiente)
```

Si `r_val` esta dentro del rango (0 a count-1), la VM lee la direccion de la tabla en la
posicion `r_val` y salta ahi. Si esta fuera del rango, la instruccion actua como NOP y la
ejecucion continua con la siguiente instruccion (caso por defecto).

### Formato de la tabla en memoria VM

La tabla es un array de `count` valores de 8 bytes (uint64) consecutivos:

```
tabla[0] = direccion del caso 0   (8 bytes, little-endian)
tabla[1] = direccion del caso 1
...
tabla[N-1] = direccion del caso N-1
```

### Ejemplo: switch sobre un codigo de color

```c
// r1 = codigo de color (0=rojo, 1=verde, 2=azul)
// r2 = direccion de la tabla de saltos
mov   r2, @Absolute("all.tabla_colores")
jumptable r1, r2, 3    // si r1 < 3, saltar al caso; si r1 >= 3, caer a default

// caso por defecto (r1 >= 3 o r1 invalido):
mov r0, 0x000000       // negro
jmp.jmp fin

caso_rojo:
    mov r0, 0xFF0000
    jmp.jmp fin

caso_verde:
    mov r0, 0x00FF00
    jmp.jmp fin

caso_azul:
    mov r0, 0x0000FF
    jmp.jmp fin

fin:
    hlt

// Tabla de saltos (en la seccion de datos):
// 3 entradas de 8 bytes cada una
tabla_colores:
    .qword @Absolute("code.caso_rojo")
    .qword @Absolute("code.caso_verde")
    .qword @Absolute("code.caso_azul")
```

### Ejemplo: switch sobre un opcode de una mini-VM

```c
// r3 = opcode (0=ADD, 1=SUB, 2=MUL, 3=DIV)
mov   r4, @Absolute("all.tabla_ops")
jumptable r3, r4, 4    // despachar en O(1)
jmp.jmp op_invalido    // fallthrough: opcode desconocido

op_add:
    addu r0, r1, r2
    jmp.jmp op_fin

op_sub:
    subu r0, r1, r2
    jmp.jmp op_fin

op_mul:
    mulu r0, r1, r2
    jmp.jmp op_fin

op_div:
    divu r0, r1, r2
    jmp.jmp op_fin

op_invalido:
    mov r0, 0
op_fin:
    ret

tabla_ops:
    .qword @Absolute("code.op_add")
    .qword @Absolute("code.op_sub")
    .qword @Absolute("code.op_mul")
    .qword @Absolute("code.op_div")
```

---

## TYPESWITCH - dispatch por tipo de objeto

### Sintaxis

```c
typeswitch r_obj, r_table, count
```

| Operando  | Descripcion                                                               |
| :-------: | :------------------------------------------------------------------------ |
| `r_obj`   | Registro con el GcHandle del objeto cuyo tipo se inspecciona.             |
| `r_table` | Registro con la direccion VM del array de pares `(ClassInfo*, addr)`.     |
| `count`   | Numero de pares en la tabla (1-255).                                      |

### Semantica exacta

```c
ClassInfo *class_ptr = gc_heap.deref(r_obj)->class_ptr;  // tipo del objeto

for (int i = 0; i < count; i++) {
    if (class_ptr == table[i].test_class) {   // igualdad de puntero
        RIP = table[i].target_addr;            // salto O(i)
        break;
    }
}
// si ninguna clase coincide: no hace nada (cae a la instruccion siguiente)
```

La comparacion es por **igualdad exacta de puntero** (identidad de tipo). Esto significa
que compara si el objeto es exactamente de ese tipo, no si hereda de el. Para comprobar
herencia, usar `instanceof` seguido de saltos condicionales.

### Formato de la tabla en memoria VM

La tabla es un array de `count` pares de 16 bytes cada uno:

```c
// Par de 16 bytes para cada caso:
struct {
    uint64_t class_info_ptr;    // ClassInfo* del tipo a comparar (8 bytes)
    uint64_t target_addr;       // direccion a la que saltar si coincide (8 bytes)
} tabla[count];
```

### Ejemplo: dispatch por tipo de objeto

```c
// r1 = GcHandle del objeto a despachar
// r2 = tabla de tipos (los ClassInfo* los rellena el loader)
mov   r2, @Absolute("all.tabla_tipos")
typeswitch r1, r2, 2    // buscar el tipo en la tabla

// tipo no encontrado en la tabla: caso por defecto
jmp.jmp caso_desconocido

caso_entero:
    // el objeto es exactamente un Integer
    // ... procesar como entero ...
    jmp.jmp fin_dispatch

caso_cadena:
    // el objeto es exactamente un String
    // ... procesar como cadena ...
    jmp.jmp fin_dispatch

caso_desconocido:
    // tipo no reconocido
    mov r0, 0

fin_dispatch:
    hlt

// Tabla de tipos: pares (ClassInfo*, addr_caso)
// ClassInfo* en offset 0 de cada par (el loader las rellena en tiempo de carga)
tabla_tipos:
    .qword @Class("Integer")                    // ClassInfo* de Integer
    .qword @Absolute("code.caso_entero")
    .qword @Class("String")                     // ClassInfo* de String
    .qword @Absolute("code.caso_cadena")
```

---

## Diferencia entre JUMPTABLE y TYPESWITCH

| Criterio             | `jumptable`                      | `typeswitch`                         |
| :------------------- | :------------------------------- | :----------------------------------- |
| Complejidad          | O(1): acceso directo por indice  | O(n): busqueda lineal por tipo       |
| Entrada              | Valor entero (0 .. count-1)      | GcHandle de cualquier objeto GC      |
| Mecanismo            | Tabla de punteros indexada       | Comparacion de `ClassInfo*`          |
| Caso de uso tipico   | `switch` sobre enum/entero       | `match`/`when` por tipo en OOP       |
| Cuando no coincide   | NOP (cae a la siguiente)         | NOP (cae a la siguiente)             |
| Fuerza herencia      | No aplica                        | No (solo igualdad exacta de tipo)    |

---

## Comparacion con alternativa manual (cadena de CMP)

Sin `jumptable`, un switch sobre 4 casos requiere 4 comparaciones y 4 saltos:

```c
// Sin jumptable: O(n) comparaciones
cmpu  r1, 0
jmp.jeq caso_0
cmpu  r1, 1
jmp.jeq caso_1
cmpu  r1, 2
jmp.jeq caso_2
cmpu  r1, 3
jmp.jeq caso_3
jmp.jmp default_caso

// Con jumptable: O(1) siempre
mov   r2, @Absolute("all.tabla")
jumptable r1, r2, 4
jmp.jmp default_caso
```

Con 255 casos, `jumptable` sigue siendo O(1). La cadena de `cmp+jmp` seria O(n=255).

---

## Consideraciones importantes

- El inmediato `count` se codifica en 1 byte: maximo **255 entradas** por tabla. Para
  tablas mas grandes, encadenar varias llamadas con rangos distintos.
- La tabla debe estar **alineada a 8 bytes** para garantizar lecturas `uint64` correctas.
- `TYPESWITCH` solo compara identidad exacta de `ClassInfo*`. No sigue la jerarquia de
  herencia. Para polimorfismo con herencia, usa `instanceof` seguido de saltos.
- Si `r_obj` contiene `GC_NULL_HANDLE` o un handle invalido en `TYPESWITCH`, el
  comportamiento es indefinido (posible segfault al desreferenciar el ObjectHeader).
  Verifica con `isnull` antes si el handle puede ser nulo.

---

## Codificacion binaria (FIXED_4, 4 bytes)

```
+--------+--------+--------+--------+
| 0x00   | opcode | byte2  | byte3  |
+--------+--------+--------+--------+
  byte0    byte1    byte2    byte3

opcode: 0x27 para jumptable, 0x28 para typeswitch

byte2:
  bits 7-4 = r_val / r_obj   (registro con el valor o handle a despachar)
  bits 3-0 = r_table          (registro con la direccion VM de la tabla)

byte3:
  bits 7-0 = count            (numero de entradas, 1-255)
```

Ejemplo: `jumptable r1, r2, 3` (r_val=1, r_table=2, count=3):
```
0x00  0x27  0x12  0x03
              ^---  nibble alto = r_val=1, nibble bajo = r_table=2
                    count = 3
```

---

Ver tambien: [[JMP]], [[OOP/Reflexion OOP]], [[OOP/CALLVIRT y CALLSUPER]]
