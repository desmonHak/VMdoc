# CALLVIRT y CALLSUPER - Despacho dinamico por vtable

Estas instrucciones implementan el **polimorfismo**: la capacidad de llamar al metodo
correcto segun el tipo real del objeto en tiempo de ejecucion, sin saber de antemano
exactamente que clase es.

> **Ver tambien:** [[THROW y RETHROW]], [[NEWOBJRAW y NEWOBJ]], [[Reflexion OOP|GETCLASS]]

---

## Para que sirve el despacho virtual

Imagina que tienes una clase `Animal` con un metodo `hablar()`, y dos subclases: `Perro`
(que implementa `hablar()` como "Guau") y `Gato` (que lo implementa como "Miau").

Si tienes una variable que puede contener cualquier `Animal`, ?como sabe el programa si
llamar al `hablar()` del Perro o del Gato? Con **CALLVIRT**: en lugar de codificar la
direccion del metodo directamente, CALLVIRT mira la **vtable** del objeto en tiempo de
ejecucion y salta al metodo correcto.

```c
// Sin polimorfismo (estatico, tienes que saber el tipo exacto):
callvm metodo_perro_hablar    // solo funciona con Perro
callvm metodo_gato_hablar     // solo funciona con Gato

// Con polimorfismo (dinamico, funciona con cualquier Animal):
callvirt r1, 0                // llama a vtable[0] del objeto en r1
                              // si r1 es un Perro -> llama metodo_perro_hablar
                              // si r1 es un Gato  -> llama metodo_gato_hablar
```

---

## Que es una vtable

La **vtable** (virtual dispatch table, tabla de despacho virtual) es un array de punteros
a metodos que forma parte de cada `ClassInfo`. Cada posicion del array corresponde a un
metodo virtual de la clase.

```
ClassAnimal.vtable:    [ hablar ]  [ moverse ]  [ comer ]
                          idx=0      idx=1        idx=2

ClassPerro.vtable:     [ hablar_perro ]  [ moverse_perro ]  [ comer ]
                               idx=0           idx=1          idx=2 (heredado)

ClassGato.vtable:      [ hablar_gato ]  [ moverse_gato  ]  [ comer ]
                               idx=0          idx=1          idx=2 (heredado)
```

`CALLVIRT objeto, 0` siempre llama al metodo en la posicion 0 de la vtable del tipo real
del objeto, sin importar que tipo declarado tenga la variable.

---

## `CALLVIRT` - llamada virtual sobre un objeto

```c
callvirt  reg_obj, vtable_idx    // despacha vtable[vtable_idx] del objeto
```

| Campo        | Valor                                                      |
| :----------: | :--------------------------------------------------------- |
| Opcode1      | `0x00`                                                     |
| Opcode2      | `0xD1`                                                     |
| Tamano       | 4 bytes (FIXED_4)                                          |
| `reg_obj`    | registro con host ptr al `ObjectHeader` del objeto receptor|
| `vtable_idx` | indice inmediato de 8 bits en la vtable (0-255)            |

### Algoritmo paso a paso

1. Lee `obj_ptr = regs[reg_obj]`.
2. Si `obj_ptr == 0`: error de segmentacion (`EVT_ERROR`). No puedes llamar metodos en null.
3. Accede al `ObjectHeader` en `obj_ptr` y lee `class_ptr` (offset +0).
4. Verifica que `vtable_idx < class_ptr->vtable_size`.
5. Obtiene `method = class_ptr->vtable[vtable_idx]`.
6. Verifica que `method != nullptr` y `method->code_vaddr != 0`.
7. Empuja un `FrameHeader` a `vm->frame_stack`:
   - `prev = frame_stack actual`
   - `method = MethodInfo*` (necesario para que THROW encuentre los handlers)
   - `return_pc = RIP + 4` (instruccion siguiente al CALLVIRT)
   - `frame_base = RSP`
8. Empuja `return_pc` a la **pila de datos** (RSP -= 8; `*RSP = return_pc`), igual que `CALLVM`.
9. Salta a `method->code_vaddr`.

### Retorno

El metodo invocado debe terminar con `RET`. Cuando `RET` se ejecuta:
- Extrae el `FrameHeader` del tope de `frame_stack`.
- Lee el return address de la pila (RSP += 8).
- Salta a ese return address (la instruccion despues del CALLVIRT).

### Ejemplo

```c
// Suponer r1 = host ptr al ObjectHeader de algun objeto Animal
// La clase tiene vtable_size >= 1 y vtable[0] apunta a un metodo valido

mov   rsp, 0x8000       // inicializar pila
mov   rbp, 0x8000

callvirt r1, 0          // llama al metodo en vtable[0] del objeto en r1
                        // (cual metodo exactamente depende del tipo real del objeto)

// La ejecucion continua aqui cuando el metodo ejecute RET
mov r10, r0             // usar el valor de retorno (en R0 por convencion)
```

---

## `CALLSUPER` - llamada a metodo de la clase padre

```c
callsuper  reg_classinfo, vtable_idx    // despacha vtable[idx] de la clase dada
```

| Campo           | Valor                                                       |
| :-------------: | :---------------------------------------------------------- |
| Opcode1         | `0x00`                                                      |
| Opcode2         | `0xD2`                                                      |
| Tamano          | 4 bytes (FIXED_4)                                           |
| `reg_classinfo` | registro con host ptr al `ClassInfo` de la clase padre      |
| `vtable_idx`    | indice en esa vtable (0-255)                                |

`CALLSUPER` es casi identico a `CALLVIRT` con una diferencia: en lugar de leer la clase
del objeto en tiempo de ejecucion, usa directamente el `ClassInfo*` que se le pasa.

Esto implementa `super.metodo()`: llamas exactamente al metodo de la clase padre, no al
sobreescrito por la clase del objeto.

```c
// r2 = ClassInfo* de la clase padre, con vtable[0] apuntando a un metodo valido
callsuper r2, 0         // llama ClassParent.vtable[0] directamente
```

---

## Codificacion binaria

Ambas instrucciones usan el formato `oop_reg_imm8`:

```
+--------+--------+------------------+----------------------+
| 0x00   | opcode |  reg1 & 0x0F     |  vtable_idx (imm8)   |
+--------+--------+------------------+----------------------+
  byte 0   byte 1       byte 2               byte 3
```

- `opcode` = `0xD1` para CALLVIRT, `0xD2` para CALLSUPER
- `reg1`   = indice del registro receptor (objeto o ClassInfo)
- `imm8`   = indice de vtable (0-255)

---

## Estado de la pila al entrar en el metodo

Cuando el metodo comienza a ejecutarse, la pila tiene este aspecto:

```
RSP  ->  [ return_pc    8 bytes ]   <- empujado por CALLVIRT
         [ ...otros frames...  ]
RBP     (sin modificar por CALLVIRT)
```

El metodo puede usar `ENTER N` / `LEAVE` / `RET` exactamente igual que con `CALLVM`:

```c
mi_metodo_virtual:
    enter 16        // push RBP; RBP=RSP; RSP-=16 (espacio para variables locales)
    // ...cuerpo del metodo...
    // R0 = valor de retorno (por convencion)
    leave           // RSP=RBP; pop RBP (liberar frame local)
    ret             // pop return_pc; saltar a el (volver al CALLVIRT)
```

---

## FrameHeader y relacion con THROW

El `FrameHeader` empujado por `CALLVIRT`/`CALLSUPER` es lo que permite a `THROW` encontrar
el handler de excepcion correcto. Cada `MethodInfo` puede tener una tabla de handlers con
rangos de PC y destinos de salto.

```
frame_stack (lista enlazada, mas reciente arriba):
  +---------------------------+
  | FrameHeader (top)         |
  |   method = MethodInfo*    | <- tiene handler_count y handlers[]
  |   return_pc = callvirt+4  |
  |   frame_base = RSP antes  |
  |   prev -> (frame anterior) |
  +---------------------------+
```

Ver [[THROW y RETHROW]] para el detalle del algoritmo de busqueda de handlers.

---

## Herencia y vtable: como funciona en la practica

La vtable de una clase derivada **copia** la vtable de la clase base e **invalida**
(sobreescribe) las entradas que el hijo reimplementa:

```
ClassBase.vtable:     [ metodo_A ]  [ metodo_B ]  [ metodo_C ]
                          idx=0        idx=1          idx=2

ClassDerived.vtable:  [ metodo_A ]  [ metodo_B2 ]  [ metodo_C ]
                          idx=0        idx=1           idx=2
                                        ^
                                   override de B: apunta al codigo del hijo
```

`CALLVIRT objeto, 1` llama:
- `metodo_B` si el objeto es de tipo `ClassBase`.
- `metodo_B2` si el objeto es de tipo `ClassDerived`.

El bytecode es identico en ambos casos; solo cambia la vtable del tipo real.

---

## Diferencia entre CALLVIRT y CALLVM

| Instruccion   | Busca el metodo en...         | Crea FrameHeader | Uso tipico              |
| :-----------: | :---------------------------- | :--------------: | :---------------------- |
| `callvm addr` | Direccion absoluta            | No               | Funciones libres        |
| `callvmr reg` | Registro                      | No               | Funciones calculadas    |
| `callvirt r, i`| Vtable del tipo real del objeto | Si             | Metodos OOP virtuales   |
| `callsuper r, i`| Vtable de una clase concreta | Si              | super.metodo() explicito|

---

Ver tambien: [[THROW y RETHROW]], [[NEWOBJRAW y NEWOBJ]], [[Reflexion OOP]], [[RET]], [[ENTER]], [[LEAVE]]
