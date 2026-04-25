# ISNULL y UNWRAP - Manejo seguro de valores nulos (Nullable)

En programacion, un valor **nulo** (null) representa la ausencia de un valor: un puntero
que no apunta a nada, un objeto que no existe, una referencia invalida. Trabajar con valores
nulos sin cuidado es una de las causas mas comunes de errores en software (el famoso
"NullPointerException" en Java o "segmentation fault" en C).

VestaVM proporciona dos instrucciones nativas para manejar nulos de forma clara y segura:

| Instruccion | opcode0 | opcode1 | Modo | Tamano  | Descripcion                                          |
| :---------: | :-----: | :-----: | :--: | :-----: | :--------------------------------------------------- |
| `isnull`    |  0x00   |  0x25   | REG  | 4 bytes | `r_dst = (r_src == 0) ? 1 : 0` sin excepcion         |
| `unwrap`    |  0x00   |  0x26   | REG  | 4 bytes | `r_dst = r_src` si no es nulo; NullPointerException si nulo |

Todas son instrucciones extendidas (prefijo `0x00`).

Implementacion: `src/runtime/exec_instruction_closure.cpp`

---

## Que es un valor nulo en VestaVM

En VestaVM, el valor nulo se representa como el entero `0`. Un registro que contiene `0`
donde se esperaba un puntero, un GcHandle, o una referencia a objeto se considera **nulo**.

Formas en que puede aparecer un nulo:

```c
// GcHandle nulo: gcalloc devuelve 0xFFFFFFFF si no hay memoria
mov    r1, 64
gcalloc r1             // R0 = GcHandle (0xFFFFFFFF si falla)

// Puntero raw nulo: alloc devuelve 0 si no hay memoria
mov    r1, 1024
alloc  r1              // R0 = puntero host (0 si falla)

// Un campo de objeto puede estar sin inicializar (= 0)
// Un argumento de funcion puede ser null intencionalmente
```

La convencion en VestaVM es: **0 = ausencia de valor** (equivale a `null` en Java/Kotlin,
`None` en Python, `nil` en Lua, o `nullptr` en C++).

---

## ISNULL - comprobar sin riesgo

```c
isnull r1, r0    // r1 = 1 si r0 es nulo, r1 = 0 si r0 no es nulo
```

`ISNULL` es la forma **segura** de comprobar si un valor es nulo. Nunca lanza excepciones
y nunca modifica los flags de comparacion de la VM. El resultado (0 o 1) se almacena en
un registro normal, lo que permite guardarlo, compararlo o usar su valor mas adelante.

### Semantica exacta

```
r_dst = (r_src == 0) ? 1 : 0
```

Si `r_src` vale exactamente `0`: `r_dst` recibe `1` (es nulo).
Si `r_src` tiene cualquier otro valor: `r_dst` recibe `0` (no es nulo).

### Por que no simplemente usar CMP?

La diferencia clave es que `isnull` **no toca los flags** (ZF, SF, CF, OF, DM):

| Criterio              | `cmpu r0, 0` / `cmp r0, r3` | `isnull r1, r0`          |
| :-------------------- | :--------------------------- | :----------------------- |
| Modifica flags        | Si (ZF, SF, CF, OF)          | No                       |
| Resultado             | En los flags                 | En un registro (0 o 1)   |
| Uso en expresiones    | Requiere salto inmediato     | Valor almacenable        |
| Se puede guardar      | No (los flags se sobreescriben) | Si                    |

Esto es util cuando necesitas comprobar varios valores y guardar los resultados antes de
decidir que hacer:

```c
// Comprobar dos punteros y guardar los resultados
isnull r10, r0    // r10 = 1 si r0 es nulo
isnull r11, r1    // r11 = 1 si r1 es nulo

// Ahora r10 y r11 tienen la informacion; los flags no se han usado
// Se puede hacer logica sobre ellos:
addu r12, r10
addu r12, r11     // r12 = cuantos de los dos son nulos (0, 1, o 2)
```

### Uso tipico: verificar resultado de una asignacion de memoria

```c
// Reservar memoria y comprobar si fallo
mov    r0, 1024
alloc  r0              // R0 = puntero (0 si no hay memoria)

isnull r1, r0          // r1 = 1 si alloc fallo
cmpu   r1, 1
jmp.je alloc_fallo     // si es nulo (r1 == 1), ir al manejador de error

// Aqui: r0 es un puntero valido
// continuar con el puntero...
jmp.jmp continuar

alloc_fallo:
    // manejar la falta de memoria
    hlt

continuar:
    // usar r0 normalmente
```

### Uso alternativo: elegir entre dos valores

```c
// Seleccionar un valor por defecto si el original es nulo
isnull r2, r0         // r2 = 1 si r0 es nulo
cmpu   r2, 0
movc   r0, r1, ZF     // si r0 no era nulo (r2==0, ZF=1): r0 = r0 (no cambia)
                      // si r0 era nulo (r2==1, ZF=0): r0 permanece 0

// Alternativa mas clara:
isnull r2, r0         // r2 = 1 si r0 nulo
cmpu   r2, 1
movc   r0, r3, ZF     // si r0 era nulo (r2==1, ZF=1): r0 = r3 (valor por defecto)
```

---

## UNWRAP - obtener el valor o morir

```c
unwrap r1, r0    // r1 = r0 si r0 != 0; lanza NullPointerException si r0 == 0
```

`UNWRAP` es la forma **agresiva** de trabajar con valores potencialmente nulos. Funciona
como el operador `!!` de Kotlin o el metodo `.unwrap()` de Rust: si el valor existe, lo
extrae; si no existe, el proceso termina inmediatamente con una excepcion.

### Semantica exacta

```
si r_src != 0:
    r_dst = r_src
    // ejecucion continua normalmente

si r_src == 0:
    // lanzar NullPointerException:
    // - aloca un ObjectHeader minimo en RawAllocator
    // - vm->err_thread = THREAD_NULL_POINTER
    // - proceso -> estado DEAD
    // - ejecucion NO continua
```

### Por que existe UNWRAP si ya esta ISNULL

`UNWRAP` es para cuando **es un error que el valor sea nulo**. En esos casos, forzar la
terminacion con una excepcion clara es mejor que continuar con datos corruptos.

Analogia: en una fabrica, si la cinta transportadora trae una pieza y resulta que la pieza
no existe (es nula), es mejor parar toda la maquina que seguir adelante y producir un
producto defectuoso. `UNWRAP` para la maquina inmediatamente.

```c
// Uso simple: exigir que r0 sea valido
unwrap r1, r0
// Si llegamos aqui: r1 tiene el valor de r0 y r0 era no nulo
// Si r0 era 0: el proceso ya termino con NullPointerException
```

### Patron recomendado: comprobar antes de desenvolver

Cuando el nulo es un caso posible pero no un error fatal, combina `isnull` + manejo manual
+ `unwrap` (solo en el camino donde ya se sabe que es no nulo):

```c
// Patron defensivo
isnull r3, r0
cmpu   r3, 1
jmp.je valor_nulo     // si es nulo, manejar caso especial

// Aqui sabemos que r0 != 0; unwrap es seguro
unwrap r1, r0
// r1 tiene el valor garantizado no nulo

jmp.jmp continuar

valor_nulo:
    // manejar el caso nulo sin terminar el proceso
    mov r1, 0           // valor por defecto u otra logica
    jmp.jmp continuar

continuar:
    // r1 tiene el valor (desenvuelto o por defecto)
    hlt
```

---

## NullPointerException interna

Cuando `UNWRAP` encuentra un valor nulo, construye una excepcion y termina el proceso:

```cpp
// Lo que hace UNWRAP internamente cuando r_src == 0:
ObjectHeader hdr;
hdr.class_ptr = nullptr;           // excepcion anonima (sin ClassInfo)
hdr.flags     = OBJ_FLAG_RAW_OWNED;
hdr.hash_code = 0;
hdr.owner_pid = 0;
hdr.lock_depth = 0;

vm->current_exception = &hdr;
vm->err_thread        = THREAD_NULL_POINTER;
vm->scheduler.on_event(EVT_ERROR); // proceso pasa a estado DEAD
```

El campo `err_thread` permite distinguir esta causa de terminacion de otras:

| `err_thread`                | Valor | Causa                                       |
| :-------------------------- | :---: | :------------------------------------------ |
| `THREAD_NO_ERROR`           |  0    | Terminacion normal (HLT)                    |
| `THREAD_SEGMENTATION_FAULT` |  2    | Acceso a memoria sin permisos               |
| `THREAD_NULL_POINTER`       |  8    | UNWRAP encontro un valor nulo               |

---

## Comparacion con alternativas manuales

### Con CMP + salto (equivalente manual a ISNULL):

```c
// Sin isnull (usando CMP y salto):
xor    r1, r1        // r1 = 0 (referencia para comparar)
cmpu   r0, r1        // establece ZF si r0 == 0
jmp.je es_nulo       // salta si r0 == 0

// No es nulo; continuar
jmp.jmp no_nulo

es_nulo:
    // manejar nulo
no_nulo:
    // continuar

// Con isnull (mas limpio, no toca flags):
isnull r2, r0        // r2 = 1 si r0 == 0
cmpu   r2, 1
jmp.je es_nulo
```

### Con CMP (equivalente manual a UNWRAP):

```c
// Sin unwrap:
xor    r1, r1
cmpu   r0, r1
jmp.je lanzar_null_exception
mov    r1, r0
jmp.jmp continuar

lanzar_null_exception:
    // muchas instrucciones para construir la excepcion...

// Con unwrap (una sola instruccion):
unwrap r1, r0
continuar:
```

---

## Codificacion binaria (FIXED_4, modo REG)

```
+--------+--------+----------+----------+
| 0x00   | opcode |  ctrl    |  regs    |
+--------+--------+----------+----------+
  byte0    byte1    byte2      byte3

opcode: 0x25 para isnull, 0x26 para unwrap

ctrl (byte2): 0x00 (sin modificadores)

regs (byte3):
  bits 7-4 = reg_dst   (nibble alto: registro destino)
  bits 3-0 = reg_src   (nibble bajo: registro fuente)
```

Ejemplo: `isnull r1, r0` (r_dst = 1, r_src = 0):
```
0x00  0x25  0x00  0x10
               ^--- r_dst=1 en nibble alto, r_src=0 en nibble bajo
```

---

## Ejemplo completo: busqueda en tabla con null check

```c
// Suponer que tenemos una tabla de punteros en VM memory
// Leer la entrada r5-esima y verificar si es valida

// Calcular direccion de la entrada: tabla_base + r5 * 8
mov    r6, @Absolute("all.tabla_ptrs")   // r6 = base de la tabla
mov    r7, r5
mov    r0, [r7*8 + r6]                   // r0 = tabla[r5]

// Verificar si la entrada existe
isnull r8, r0
cmpu   r8, 1
jmp.je entrada_nula                      // si es nula, manejar

// La entrada existe; usarla de forma segura
unwrap r9, r0                            // r9 = puntero valido garantizado
// ... operar con r9 ...
jmp.jmp fin

entrada_nula:
    // la posicion r5 no tiene un objeto valido
    hlt

fin:
    hlt
```

---

Ver tambien: [[THROW y RETHROW]], [[GC/GCALLOC]], [[Aritmetica y logica]]
