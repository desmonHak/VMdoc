# WEAKREF, DEREF_WEAK y FREE_WEAK - Referencias debiles al GC

Normalmente, cuando guardas un GcHandle en un registro o en el payload de otro objeto, ese
objeto **no puede ser recolectado** por el GC mientras la referencia exista. Eso se llama
una **referencia fuerte**: el GC ve que alguien lo necesita y lo mantiene vivo.

Pero a veces quieres observar un objeto sin impedir que muera. Por ejemplo:

- Un cache de objetos que ya no necesitas calcular de nuevo SI siguen en memoria, pero
  que puedes recalcular si el GC los elimino.
- Un sistema de listeners/observers donde el objeto observado puede desaparecer sin que
  el observer deba saber de antemano.
- Una tabla de objetos internados (strings, simbolos) donde quieres reutilizar el mismo
  objeto si todavia existe, pero crear uno nuevo si fue recolectado.

Para eso existen las **referencias debiles**: te permiten "ver" un objeto sin retenerlo.
Si el GC decide eliminarlo, la referencia debil se anula automaticamente.

| Instruccion  | opcode0 | opcode1 | Modo | Tamano  | Descripcion                                     |
| :----------: | :-----: | :-----: | :--: | :-----: | :---------------------------------------------- |
| `weakref`    |  0x00   |  0x30   | REG  | 4 bytes | Registrar GcHandle; R0 = indice opaco           |
| `deref_weak` |  0x00   |  0x32   | REG  | 4 bytes | Resolver: r_dst = handle si vivo, 0 si muerto   |
| `free_weak`  |  0x00   |  0x34   | REG  | 4 bytes | Liberar la entrada de la tabla                  |

Implementacion: `src/runtime/exec_instruction_weak.cpp`

---

## Como funciona internamente

El GcHeap mantiene un array de entradas `WeakEntry`:

```cpp
struct WeakEntry {
    uint32_t target;  // GcHandle del objeto observado
    bool     live;    // false = slot libre o objeto ya recolectado
};
```

Cada llamada a `weakref` asigna un slot en este array y devuelve su **indice** como un
`uint32_t` opaco. Durante el barrido del major GC, las entradas cuyo objeto fue recolectado
se marcan con `live = false` y `target = 0`.

El bytecode recibe ese indice opaco y lo usa con `deref_weak` para preguntar "sigue vivo?".

---

## WEAKREF - registrar una referencia debil

```c
weakref r1    // R0 = indice opaco de la nueva referencia debil
```

| Campo     | Descripcion                                                |
| :-------: | :--------------------------------------------------------- |
| `r1`      | Registro con el GcHandle del objeto a observar             |
| R0        | Indice opaco `uint32_t` para usar con deref_weak/free_weak |

El objeto referenciado **no** se marca como raiz GC. El GC puede recolectarlo en cualquier
ciclo siguiente si no hay referencias fuertes que lo mantengan vivo.

```c
// Crear un objeto y registrar una referencia debil a el
mov    r1, 64
newobj r1              // R0 = GcHandle del objeto
mov    r8, r0          // r8 = referencia FUERTE (mantiene el objeto vivo)

weakref r8             // R0 = indice opaco de la referencia debil
mov    r9, r0          // r9 = indice de la weak ref
```

---

## DEREF_WEAK - consultar si el objeto sigue vivo

```c
deref_weak r2, r3    // r2 = GcHandle si el objeto vive, 0 si fue recolectado
```

| Campo   | Descripcion                                                      |
| :-----: | :--------------------------------------------------------------- |
| `r3`    | Indice opaco de la weak ref (obtenido de `weakref`)              |
| `r2`    | Resultado: GcHandle si el objeto existe, 0 si fue recolectado    |

Logica interna:
- Si `weak_table[r3].live == true`: `r2 = weak_table[r3].target` (el GcHandle).
- Si `weak_table[r3].live == false`: `r2 = 0` (el objeto fue recolectado).

```c
// Consultar si el objeto observado sigue vivo
deref_weak r5, r9    // r5 = GcHandle si vivo, 0 si muerto

cmpu   r5, 0
jmp.jeq objeto_muerto

// El objeto sigue vivo: usarlo de forma segura
// r5 es un GcHandle valido
jmp.jmp usar_objeto

objeto_muerto:
    // El objeto fue recolectado; hacer algo alternativo
    // (recalcular, crear nuevo, etc.)
```

---

## FREE_WEAK - liberar la entrada de la tabla

```c
free_weak r9    // libera la entrada r9 de la tabla de weak refs
```

| Campo   | Descripcion                                         |
| :-----: | :-------------------------------------------------- |
| `r9`    | Indice opaco de la weak ref a liberar               |

Marca el slot como libre para ser reutilizado. Llamar a `free_weak` con un indice ya
liberado es un **no-op** (se ignora silenciosamente).

> **Importante:** la tabla de weak refs crece con cada llamada a `weakref` si los slots
> no se reciclan. Llama siempre a `free_weak` cuando ya no necesites la referencia debil.

---

## Patron tipico completo

```c
// Suponer que r1 = GcHandle de un objeto existente

// 1. Crear la referencia debil
weakref r1           // R0 = indice opaco
mov   r10, r0        // guardar el indice en r10

// ... el objeto puede ser recolectado por el GC en cualquier momento
// ... (si se suelta la referencia fuerte con DROP)

// 2. Consultar si el objeto sigue vivo
deref_weak r5, r10   // r5 = GcHandle si vivo, 0 si muerto

cmpu   r5, 0
jmp.jeq objeto_muerto

// El objeto esta vivo: usarlo
gcderef cur0, r5     // cur0 = payload del objeto
// ... leer o escribir datos ...
jmp.jmp fin

objeto_muerto:
    // El objeto fue recolectado; accion alternativa
    // (crear uno nuevo, usar un valor por defecto, etc.)

fin:
    // 3. Liberar la entrada cuando ya no se necesite
    free_weak r10
    hlt
```

---

## Cuando el GC anula las referencias debiles

El GC de VestaVM es generacional de dos fases:

1. **Minor GC** (colecta el heap joven): NO anula las weak refs. Los objetos promovidos
   al heap mayor siguen siendo validos. Las weak refs a esos objetos permanecen activas.

2. **Major GC** (colecta el heap completo): despues del barrido, itera sobre toda la tabla
   de weak refs. Para cada entrada activa comprueba si el GcHandle referenciado apunta a
   un objeto que sobrevivio el barrido. Si el objeto fue recolectado, pone `live = false`
   y `target = 0`.

```c
// Secuencia que puede anular una weak ref:
// 1. crear objeto, obtener weak ref
// 2. soltar la referencia fuerte con DROP
// 3. forzar un major GC con gcrun
// 4. deref_weak devolvera 0

newobj r1
mov    r8, r0          // referencia fuerte
weakref r8             // R0 = indice
mov    r9, r0          // guardar indice

drop   r8              // soltar la referencia fuerte
gcrun                  // forzar major GC -> el objeto puede ser recolectado

deref_weak r5, r9      // r5 = 0 si el GC lo recogio
```

> **Regla de seguridad:** nunca almacenes el GcHandle obtenido de `deref_weak` mas alla
> del uso inmediato. El patron seguro es:
> `deref_weak -> verificar != 0 -> usar inmediatamente -> no guardar el handle`

---

## Diferencia entre referencia fuerte y debil

| Criterio              | Referencia fuerte (registro/pila)   | Referencia debil (`weakref`)         |
| :-------------------- | :---------------------------------- | :----------------------------------- |
| Impide recoleccion    | Si                                  | No                                   |
| Valida siempre        | Si (mientras el proceso vive)       | No (puede anularse tras major GC)    |
| Requiere liberacion   | No (DROP opcional para el GC)       | Si (`free_weak` para el slot)        |
| Caso de uso           | Propietario del objeto              | Observador (cache, listener, etc.)   |

---

## Codificacion binaria (FIXED_4, 4 bytes)

```
+--------+--------+--------+--------+
| 0x00   | opcode | ctrl   | regs   |
+--------+--------+--------+--------+
  byte0    byte1    byte2    byte3

opcodes:
  0x30 = weakref
  0x32 = deref_weak
  0x34 = free_weak

byte3 para weakref y free_weak:
  bits 3-0 = r_handle / r_idx   (registro unico de entrada)

byte3 para deref_weak:
  bits 7-4 = r_dst   (nibble alto: registro donde escribir el GcHandle)
  bits 3-0 = r_idx   (nibble bajo: indice opaco de la weak ref)
```

Ejemplo: `deref_weak r5, r9` (r_dst=5, r_idx=9):
```
0x00  0x32  0x00  0x59
                   ^--- r_dst=5 en nibble alto, r_idx=9 en nibble bajo
```

---

Ver tambien: [[GC/Generacional (para objetos OOP)]], [[GC/GC]], [[GC/GCALLOC]]
