# Instrucciones cursor - acceso a memoria real del host

Las instrucciones cursor permiten leer y escribir directamente en memoria real del proceso host (no en la memoria virtual de la VM). Son el puente entre el bytecode Vesta y:

- Bloques reservados con [[Allocator crudo para FFI  y memoria manual|ALLOC]] (punteros host reales).
- Payloads de objetos gestionados por el [[Generacional (para objetos OOP)|GcHeap]] (accedidos mediante `GCDEREF`).
- Cualquier dirección de host válida almacenada en un registro cursor.

> **¿Por qué hace falta esto?**
> La memoria virtual de la VM se gestiona a través de una capa TLB/VirtualMemory propia. Las instrucciones MOV, ADD, etc. operan sobre registros. Para acceder a un bloque de RAM del host (por ejemplo para rellenar un buffer antes de una llamada nativa), se necesita un mecanismo que desreferencie directamente la dirección host - eso son las instrucciones cursor.

---

## Registros cursor

Los cuatro registros cursor son registros especiales de 64 bits que contienen **direcciones del espacio de host**. Se codifican igual que el resto de registros especiales y pueden usarse con `XCHG` y `MOV` para cargar/guardar direcciones.

| Registro | Codificación |
| :------: | :----------: |
| `cur0`   |   `0b000000` |
| `cur1`   |   `0b000001` |
| `cur2`   |   `0b000010` |
| `cur3`   |   `0b000011` |

> Se recomienda que los registros cursor **solo contengan direcciones host válidas**. Cuando no se usen deben valer `0` (equivalente a `NULL`). Desreferenciar un cursor nulo o una dirección inválida produce comportamiento indefinido.

---

## Instrucciones

### `readcur` - leer desde memoria host

```c
readcur  dest_reg, curN
```

Lee N bytes de la dirección host almacenada en `curN` y los escribe en `dest_reg`. El tamaño viene determinado por el sufijo del registro destino (b=1, w=2, d=4, q=8 bytes).

| Opcode1 | Opcode2 | ctrl_byte              | reg_byte          | Total |
| :-----: | :-----: | :--------------------- | :---------------- | :---: |
| `0x00`  | `0xC0`  | `mm cc 00 00`          | `0000 rrrr`       |  4 B  |

- `mm` = tamaño: `00`=byte, `01`=word, `10`=dword, `11`=qword
- `cc` = índice del cursor (0-3)
- `rrrr` = registro general destino (R0-R15)

**Ejemplo:**
```c
alloc  r1q          // reservar memoria - R0 = ptr host
xchg   cur0, r0     // cur0 = ptr host
readcur r2q, cur0   // r2 = *(uint64_t*)cur0
readcur r3b, cur0   // r3 = *(uint8_t*)cur0  (solo el primer byte)
```

---

### `writecur` - escribir en memoria host

```c
writecur  curN, src_reg
```

Escribe N bytes del registro `src_reg` en la dirección host almacenada en `curN`. El tamaño viene determinado por el sufijo del registro fuente.

| Opcode1 | Opcode2 | ctrl_byte              | reg_byte          | Total |
| :-----: | :-----: | :--------------------- | :---------------- | :---: |
| `0x00`  | `0xC1`  | `mm cc 00 00`          | `0000 rrrr`       |  4 B  |

Misma codificación que `readcur`; el primer operando es el cursor (destino en memoria) y el segundo es el registro fuente.

**Ejemplo:**
```c
mov    r5, 0xFF00FF00   // valor a escribir
writecur cur0, r5       // *(uint64_t*)cur0 = r5
writecur cur1, r6d      // *(uint32_t*)cur1 = r6d (solo 32 bits)
```

---

### `gcderef` - desreferenciar handle del GC -> cursor

```c
gcderef  curN, handle_reg
```

Toma el `GcHandle` contenido en `handle_reg`, lo desreferencia mediante la `HandleTable` del GC y almacena el puntero raw al payload del objeto en `curN`. A partir de ese momento `curN` contiene una dirección host válida que puede usarse con `readcur` / `writecur`.

| Opcode1 | Opcode2 | ctrl_byte              | reg_byte          | Total |
| :-----: | :-----: | :--------------------- | :---------------- | :---: |
| `0x00`  | `0xC2`  | `00 cc 00 00`          | `0000 rrrr`       |  4 B  |

- `cc` = índice del cursor destino (0-3)
- `rrrr` = registro general que contiene el handle

> **⚠ Advertencia de invalidación**
> El puntero obtenido con `gcderef` **puede quedar inválido** tras cualquier ciclo de GC (`GCRUN`, o asignación que dispare un minor GC). Si el GC mueve el objeto la dirección en el cursor queda obsoleta. Úsalo solo en fragmentos cortos de código donde no haya asignaciones intermedias, o llama a `gcderef` de nuevo tras cada posible ciclo.

**Ejemplo:**
```c
mov    r1, 64        // 64 bytes de payload
newobj r1            // R0 = GcHandle
gcderef cur2, r0     // cur2 = puntero al payload del objeto
writecur cur2, r3    // escribe el primer campo (qword)
readcur  r4, cur2    // lee el primer campo de vuelta
```

---

## Avanzar el cursor manualmente

Las instrucciones cursor no incluyen auto-incremento. Para iterar sobre campos de una estructura, avanza el cursor con `XCHG` + `ADDU` + `XCHG`:

```c
; Leer dos campos qword consecutivos de un objeto GC
gcderef cur0, r0          // cur0 -> inicio del payload

readcur r1, cur0          // r1 = campo[0]

xchg    r10, cur0         // r10 = cur0
addu    r10, 8            // r10 += 8 (saltar 8 bytes)
xchg    cur0, r10         // cur0 = r10

readcur r2, cur0          // r2 = campo[1]
```

---

## Patrones de uso completos

### Leer/escribir un buffer de raw allocator

```c
// Reservar buffer de 256 bytes
mov    r1, 256
alloc  r1               // R0 = ptr host

// Guardar puntero en cursor
xchg   cur0, r0

// Escribir 8 bytes en el inicio
mov    r5, 0xDEADBEEFCAFEBABE
writecur cur0, r5

// Leer de vuelta
readcur r6, cur0        // r6 = 0xDEADBEEFCAFEBABE

// Liberar
xchg   r0, cur0
free   r0
```

### Inicializar un objeto GC campo a campo

```c
mov    r1, 24           // objeto de 3 campos qword (24 bytes)
newobj r1               // R0 = GcHandle (guardarlo en r8)
mov    r8, r0

gcderef cur1, r8        // cur1 -> payload

// campo 0 (offset 0)
mov    r2, 42
writecur cur1, r2

// campo 1 (offset 8)
xchg   r9, cur1
addu   r9, 8
xchg   cur1, r9
mov    r2, 100
writecur cur1, r2

// campo 2 (offset 16)
xchg   r9, cur1
addu   r9, 8
xchg   cur1, r9
mov    r2, 0
writecur cur1, r2
```

---

## Codificación detallada

Todas las instrucciones cursor son **extensiones de opcode** (opcode1 = `0x00`) de **4 bytes** fijos:

```
┌────────┬────────┬────────────────────┬────────────────────┐
│ 0x00   │ opcode │    ctrl_byte       │    reg_byte        │
│        │        │ mm cc 00 00        │ 0000 rrrr          │
└────────┴────────┴────────────────────┴────────────────────┘
          ↑         ↑↑  ↑↑               ↑↑↑↑
          │         ││  └─ cursor (0-3)  └─── registro gral (0-15)
          │         └─ modo/tamaño: 00=byte 01=word 10=dword 11=qword
          └ C0=readcur  C1=writecur  C2=gcderef
```

Para `gcderef` los bits de modo (`mm`) no se usan (siempre se lee el handle como qword).

---

Ver también: [[GC]], [[Generacional (para objetos OOP)|GcHeap]], [[Allocator crudo para FFI  y memoria manual|RawAllocator]], [[NativeCall (CallN)]], [[XCHG - Exchange]]
