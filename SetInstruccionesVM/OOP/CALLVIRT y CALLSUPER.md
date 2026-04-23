# CALLVIRT y CALLSUPER - Despacho dinámico por vtable

Estas instrucciones implementan el **despacho virtual** (polimorfismo en tiempo de ejecución). Ambas buscan un método en la vtable de una clase y saltan a él empujando un `FrameHeader` a la cadena de llamadas OOP (necesario para `THROW`/`RETHROW`) y el return address a la pila (para que `RET` funcione).

> **Ver también:** [[THROW y RETHROW]], [[NEWOBJ y NEWOBJRAW]], [[Reflexion OOP|GETCLASS]]

---

## `CALLVIRT` - llamada virtual sobre un objeto

```asm
callvirt  reg_obj, vtable_idx    ; despacha vtable[vtable_idx] del objeto
```

| Campo      | Valor              |
| :--------: | :----------------- |
| Opcode1    | `0x00`             |
| Opcode2    | `0xD1`             |
| Tamaño     | 4 bytes (FIXED_4)  |
| `reg_obj`  | registro con host ptr al `ObjectHeader` del receptor |
| `vtable_idx` | índice de 8 bits en la vtable de la clase (0–255)  |

### Algoritmo

1. Lee `obj_ptr = regs[reg_obj]`.
2. Si `obj_ptr == 0`: error de segmentación (`EVT_ERROR`).
3. Accede al `ObjectHeader` en `obj_ptr` y lee `class_ptr`.
4. Verifica que `vtable_idx < class_ptr->vtable_size`.
5. Obtiene `method = class_ptr->vtable[vtable_idx]`.
6. Verifica que `method != nullptr` y `method->code_vaddr != 0`.
7. Empuja un `FrameHeader` a `vm->frame_stack`:
   - `prev       = frame_stack actual`
   - `method     = MethodInfo*`
   - `return_pc  = RIP + 4` (instrucción siguiente)
   - `frame_base = RSP`
8. Empuja `return_pc` a la **pila** (RSP -= 8; `*RSP = return_pc`), igual que `CALLVM`.
9. Salta a `method->code_vaddr`.

### Retorno

El método invocado debe terminar con `RET`. La instrucción `RET`:
- Extrae el `FrameHeader` del tope de `frame_stack`.
- Lee el return address de la pila (RSP += 8).
- Salta a ese return address.

### Ejemplo

```asm
; Suponer r1 = host ptr al ObjectHeader (ObjectHeader.class_ptr apunta a una
; ClassInfo con vtable_size >= 1 y vtable[0]->code_vaddr válido)

mov   rbp, 0x8000       ; inicializar pila
mov   rsp, 0x8000

callvirt r1, 0          ; llama vtable[0] del objeto en r1
                        ; (dispatch polimórfico)

; La ejecución continúa aquí tras el RET del método
addu  r10, 1
```

---

## `CALLSUPER` - llamada a método de clase padre explícita

```asm
callsuper  reg_classinfo, vtable_idx    ; despacha vtable[idx] de la clase dada
```

| Campo          | Valor              |
| :------------: | :----------------- |
| Opcode1        | `0x00`             |
| Opcode2        | `0xD2`             |
| Tamaño         | 4 bytes (FIXED_4)  |
| `reg_classinfo`| registro con host ptr al `ClassInfo` de la clase padre |
| `vtable_idx`   | índice en esa vtable (0–255)                           |

`CALLSUPER` es idéntico a `CALLVIRT` excepto que el receptor es una `ClassInfo*` directa, no un objeto. Esto permite implementar `super.method()` o llamadas no polimórficas a una clase concreta.

### Ejemplo

```asm
; r2 = ClassParent*, con vtable[0] apuntando a un método válido
callsuper r2, 0         ; llama ClassParent.vtable[0] directamente
```

---

## Codificación binaria

Ambas instrucciones usan el formato `oop_reg_imm8`:

```
┌────────┬────────┬──────────────────┬──────────────────────┐
│ 0x00   │ opcode │  reg1 & 0x0F     │  vtable_idx (imm8)   │
└────────┴────────┴──────────────────┴──────────────────────┘
  byte 0   byte 1       byte 2               byte 3
```

- `opcode` = `0xD1` para CALLVIRT, `0xD2` para CALLSUPER
- `reg1`   = índice del registro receptor (objeto o ClassInfo)
- `imm8`   = índice de vtable (0–255)

---

## Invariantes de la pila al entrar en el método

Cuando el método comienza a ejecutarse:

```
RSP  ->  [ return_pc    8 bytes ]
         [ ...otros frames...  ]
RBP     (sin modificar por CALLVIRT)
```

El método puede usar `ENTER N` / `LEAVE` / `RET` normalmente:

```asm
my_virtual_method:
    enter 16        ; push RBP; RBP=RSP; RSP-=16 (espacio para locales)
    ; ...cuerpo...
    leave           ; RSP=RBP; pop RBP
    ret             ; RET extrae FrameHeader + pop return_pc
```

---

## FrameHeader y relación con THROW

El `FrameHeader` empujado por `CALLVIRT`/`CALLSUPER` es lo que permite a `THROW` encontrar el handler correcto. Cada `MethodInfo` puede declarar una tabla de `HandlerException` con rangos de PC y destinos de salto.

```
frame_stack:
  ┌─────────────────────────────┐
  │ FrameHeader (top)           │
  │   method = MethodInfo*      │<- handler_count, handlers[]
  │   return_pc = callvirt+4    │
  │   frame_base = RSP inicial  │
  │   prev ->  (frame anterior)  │
  └─────────────────────────────┘
```

Ver [[THROW y RETHROW]] para el detalle del algoritmo de búsqueda de handlers.

---

## Vtable y herencia

La vtable de una clase derivada **copia** la vtable de la clase base e **invalida** las entradas que sobreescribe. Esta convención debe ser mantenida por el loader o el código ensamblador:

```
ClassBase.vtable:   [ método_A ]  [ método_B ]  [ método_C ]
ClassDerived.vtable:[ método_A ]  [ método_B2]  [ método_C ]
                                        ↑
                                   override slot 1
```

`CALLVIRT objeto, 1` llama `método_B` si el objeto es de tipo `ClassBase`, o `método_B2` si es de tipo `ClassDerived`: el polimorfismo clásico.

---

Ver también: [[THROW y RETHROW]], [[NEWOBJRAW y NEWOBJ]], [[Reflexion OOP]], [[RET]], [[ENTER]], [[LEAVE]]
