# Sincronizacion en Vex

Primitivas de sincronización para concurrencia entre procesos: monitores
reentrantes (`synchronized`), wait/notify (similar a Java), y patrón de
exception-safe automatico.

---

## Indice

- [Sincronizacion en Vex](#sincronizacion-en-vex)
 - [Indice](#indice)
 - [1. Modelo de procesos en VestaVM](#1-modelo-de-procesos-en-vestavm)
 - [2. `synchronized (obj) { ... }`](#2-synchronized-obj---)
 - [3. Monitores reentrantes](#3-monitores-reentrantes)
 - [4. `wait`, `notify`, `notifyAll`](#4-wait-notify-notifyall)
 - [5. Cleanup automatico (try/finally implicito)](#5-cleanup-automatico-tryfinally-implicito)
 - [6. Layout runtime: ObjectHeader monitor fields](#6-layout-runtime-objectheader-monitor-fields)
 - [7. Patron productor-consumidor](#7-patron-productor-consumidor)
 - [8. Limitaciones](#8-limitaciones)

---

## 1. Modelo de procesos en VestaVM

VestaVM es una VM **multi-proceso** (similar a Erlang): cada `spawn { ... }` crea
un ProcessVM independiente con su propio stack y registros. Los procesos se
ejecutan en N schedulers que mappean a OS threads (`--schedulers N`).

Para coordinar entre procesos:
- **Memory shared** vía `synchronized` (monitor del objeto GC).
- **Message passing** vía `msgsend`/`msgrecv` (mailboxes).
- **Futures** para resultados async.

Ver [[Async]] para spawn/await/IPC. Este doc cubre `synchronized` y monitores.

---

## 2. `synchronized (obj) { ... }`

Adquiere el monitor de un objeto GC durante el body. Garantiza exclusión mutua
entre procesos que entren al mismo monitor:

```vex
class Counter {
    public i32 value = 0;
}

i32 main() {
    Counter c = new Counter();
 
    // Spawn 4 procesos que incrementan c.value 1M veces cada uno
    for (i32 i = 0; i < 4; i = i + 1) {
        spawn {
            for (i32 j = 0; j < 1000000; j = j + 1) {
                synchronized (c) {
                    c.value = c.value + 1;
                }
            }
        };
    }
    // (esperar a los procesos...)
    return c.value; // = 4000000 garantizado (sin race)
}
```

**Lowering**:
- Convierte `obj` (host_ptr) -> GcHandle vía nuevo opcode `gchandle r_dst, r_src`
 (0x56, O(1) lookup en `ptr_to_handle_` del GcHeap).
- Emite `monenter r_handle` al entry, `monexit r_handle` al exit (en todas
 las salidas: normal, return, throw).

---

## 3. Monitores reentrantes

Un proceso puede adquirir el monitor del MISMO objeto múltiples veces (recursivo):

```vex
class Counter {
    public i32 value = 0;
 
    public void increment() {
        synchronized (this) {
            this.value = this.value + 1;
        }
    }
 
    public void increment_twice() {
        synchronized (this) {
            this.increment(); // reentra al mismo monitor (lock_depth = 2)
            this.increment(); // lock_depth = 3, luego 2
        } // lock_depth = 1, luego 0 (libera)
    }
}
```

**Lock depth counter** en `ObjectHeader` (campo `lock_depth: u16`). `monenter`
incrementa; `monexit` decrementa. Solo al llegar a 0 se libera el monitor (y
se despierta el siguiente proceso esperando).

---

## 4. `wait`, `notify`, `notifyAll`

Para coordinación productor-consumidor sin busy-loop:

```vex
class Queue {
    public i32[100] data;
    public i32 head = 0;
    public i32 tail = 0;
 
    public void produce(i32 v) {
        synchronized (this) {
            this.data[this.tail] = v;
            this.tail = (this.tail + 1) % 100;
            notify(this); // despierta UN consumer esperando
        }
    }
 
    public i32 consume() {
        synchronized (this) {
            while (this.head == this.tail) {
                wait(this); // suelta monitor + suspende; reentry al despertar
            }
            i32 v = this.data[this.head];
            this.head = (this.head + 1) % 100;
            return v;
        }
    }
}
```

| Builtin | Bytecode | Comportamiento |
| :----------------- | :-------------- | :---------------------------------------------- |
| `wait(obj)` | `monwait` | Suelta monitor + suspende. Re-adquiere al wake. |
| `notify(obj)` | `monnoti` | Despierta UN proceso de la cola de espera. |
| `notifyAll(obj)` | `monnota` | Despierta TODOS los procesos esperando. |

**IMPORTANTE**: las 3 deben llamarse DENTRO de `synchronized (obj) { ... }` o
el comportamiento es indefinido. Validación estática pendiente (hoy es runtime
check).

**Patron clásico** (always loop on wait):

```vex
synchronized (obj) {
    while (!condition()) {
        wait(obj); // protege contra spurious wakeups
    }
    // ...usar...
}
```

---

## 5. Cleanup automatico (try/finally implicito)

`synchronized` garantiza que `monexit` se ejecuta en TODAS las salidas, incluso
exceptions y returns:

```vex
i32 risky() {
    Counter c = ...;
    synchronized (c) {
        if (some_cond) {
            return c.value; // monexit se ejecuta antes del ret
        }
        if (other_cond) {
            throw new MyException(); // monexit se ejecuta antes del throw propagado
        }
        c.value++;
    } // monexit normal aqui
    return -1;
}
```

**Implementación**:

1. **`cleanup_stack_`** (vector<CleanupAction>) en `Lowering`. `synchronized`
 push `tryleave + monexit` antes del body, pop después.
2. **`lower_return`** emite `emit_cleanups_all()` (LIFO) antes del RET.
3. **Exception safety**: `tryenter catch-all` con handler que hace `monexit +
 rethrow`. El `lower_try` refactorizado usa SSA values + `{src0}/{src1}`
 substitution en RAW_ASM para no clobberar SSA values vivos del caller.

Resultado: `monexit` está GARANTIZADO independientemente de cómo se sale del
synchronized block.

---

## 6. Layout runtime: ObjectHeader monitor fields

```cpp
struct alignas(8) ObjectHeader { // 24 bytes
    ClassInfo *class_ptr; // +0
    uint32_t flags; // +8
    uint32_t hash_code; // +12
    uint32_t owner_pid; // +16 local_pid del owner del monitor (0 = libre)
    uint16_t lock_depth; // +20 reentrant lock count
    uint16_t _mon_pad; // +22 alignment padding
};
```

`monenter` hace CAS sobre `owner_pid`:
- Si `owner_pid == 0` (libre) -> CAS a current_pid, `lock_depth = 1`.
- Si `owner_pid == current_pid` (reentry) -> `lock_depth++`.
- Si `owner_pid != current_pid` (held by otro) -> push current_proc a la
 wait queue del monitor, suspend (state = BLOCKED).

`monexit`:
- `lock_depth--`.
- Si `lock_depth == 0` -> `owner_pid = 0`, despertar el siguiente en la wait
 queue (FIFO).

---

## 7. Patron productor-consumidor

```vex
class Channel {
    public i32 buf = 0;
    public bool full = false;
 
    public void put(i32 v) {
        synchronized (this) {
            while (this.full) {
                wait(this); // espera espacio libre
            }
            this.buf = v;
            this.full = true;
            notify(this); // despierta consumidor
        }
    }
 
    public i32 take() {
        synchronized (this) {
            while (!this.full) {
                wait(this); // espera dato disponible
            }
            i32 v = this.buf;
            this.full = false;
            notify(this); // despierta productor
            return v;
        }
    }
}

i32 main() {
    Channel ch = new Channel();
 
    spawn {
        for (i32 i = 0; i < 100; i = i + 1) {
            ch.put(i);
        }
    };
 
    i32 sum = 0;
    for (i32 i = 0; i < 100; i = i + 1) {
        sum = sum + ch.take();
    }
    return sum; // = 4950 (sum 0..99)
}
```

---

## 8. Limitaciones

1. **Validación estática wait/notify**: el compilador no verifica que `wait`/
 `notify`/`notifyAll` se llamen dentro de `synchronized (obj)`. Solo runtime
 check (falla con FATAL si se llaman fuera del monitor).

2. **No hay `tryLock(obj, timeout)`** equivalente. `synchronized` siempre
 bloquea hasta adquirir. Workaround: spawn child + `await fut con timeout`.

3. **No hay `condition variables` separadas**: cada monitor tiene UNA cola
 de espera única. Para múltiples conditions, usar varios objetos lock o
 pasar mensajes via mailbox.

4. **Sincronización cross-node** (distrib): `synchronized` es local al nodo
 (mismo VM). Para cross-node, usar `rspawn` + futures + `msgsend` (sin
 shared memory).

5. **Reentrancia profunda no validada**: `lock_depth` es u16 (max 65535
 reentradas). Recursión infinita en `synchronized (this)` desborda
 silenciosamente.

---

Ver también: [[Async]] (spawn / msgsend / msgrecv para comunicación entre
procesos sin shared memory), [[OOP]] (objetos GC son owner natural del monitor),
[[Excepciones]] (try/catch interactúa correctamente con synchronized cleanup).
