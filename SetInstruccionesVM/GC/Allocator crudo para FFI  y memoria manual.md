# RawAllocator - Memoria cruda para FFI y gestión manual

Allocator manual por proceso sin GC automático. Retorna **punteros host reales** que se pueden pasar directamente a funciones nativas (C/C++, Win32 API, OpenSSL, etc.) a través de [[NativeCall (CallN)]].

> Si no conoces qué es la gestión de memoria ni la diferencia entre manual y automático, empieza por [[GC]] antes de leer esta página.

---

## ¿Qué es FFI y por qué necesita punteros reales?

**FFI** (*Foreign Function Interface*) es el mecanismo por el que VestaVM llama a funciones escritas en otro lenguaje - normalmente C. Ejemplos: llamar a `WriteFile` de Windows, usar OpenSSL para cifrado, invocar una biblioteca de gráficos, etc.

El problema es que esas funciones esperan recibir **la dirección real de un bloque de bytes en la RAM del proceso**. No entienden handles, no entienden objetos con cabeceras, no entienden la `HandleTable` del GC. Solo entienden "dame un puntero a N bytes contiguos".

```c
// Función C que espera un puntero a un buffer de 256 bytes:
int WriteFile(HANDLE hFile, const void *lpBuffer, DWORD nBytesToWrite, ...);
//                                   ^^^^^^^^^^^
//                                   necesita la dirección real
```

El `GcHeap` no puede satisfacer esto directamente: los handles son índices, no punteros, y el GC puede mover objetos. El `RawAllocator` sí puede: reserva un bloque contiguo en la RAM del host y devuelve su dirección real.

---

## ¿Qué significa "contiguo"?

Un bloque de N bytes es **contiguo** cuando `byte[0]`, `byte[1]`, ..., `byte[N-1]` están en posiciones consecutivas de memoria sin huecos ni saltos. La mayoría de las funciones C que reciben buffers asumen esto.

Si la memoria no fuera contigua, una función como `memcpy` que copia desde la dirección dada hasta `dirección + N` leería memoria que no pertenece al buffer - con resultados impredecibles (datos incorrectos, crash, vulnerabilidad de seguridad).

---

## Por qué no se usa la capa VirtualMemory / TLB

La capa [[TLB|VirtualMemory/TLB]] de VestaVM divide las reservas en páginas de 4096 bytes y las mapea en rangos **virtuales de la VM**, no en el espacio de direcciones del proceso host. Un bloque de N bytes que cruce un límite de página puede tener bytes físicos no contiguos desde el punto de vista del host.

```
Vista TLB (virtual VM):    [ 0x1000 .. 0x1FFF ][ 0x2000 .. 0x2FFF ]
                                        ↓                  ↓
Vista host (fisica):       [  pagina A en RAM  ][  pagina B en RAM  ]
                            (pueden NO ser adyacentes en el espacio de host)
```

Si una función C recibe `0x1000` y lee 8192 bytes, lee pagina A + pagina B. Si en el host esas páginas no son adyacentes, la lectura accede a memoria ajena.

`RawAllocator` llama directamente a `VirtualAlloc` (Windows) o `mmap` (POSIX) mediante `vm::allocate_memory()`, que garantiza un **rango contiguo único** en el espacio de direcciones real del proceso host - equivalente a `malloc(3)` pero sin fragmentador intermedio.

---

## Diferencia con el GcHeap

| Aspecto                  | GcHeap                                 | RawAllocator                          |
| :----------------------- | :------------------------------------- | :------------------------------------ |
| Qué devuelve al bytecode | Handle opaco (número entero)           | Puntero host real (dirección RAM)     |
| El objeto se puede mover | Sí (el handle oculta el movimiento)    | No (la dirección es permanente)       |
| Compatible con FFI       | No                                     | Sí                                    |
| Ciclo de vida            | Automático (GC recoge la basura)       | Manual (`FREE` obligatorio)           |
| Riesgo si se olvida      | Fuga leve (el proceso la recoge al salir) | Fuga de memoria hasta que el proceso termina |

---

## Ciclo de vida de un bloque

Un bloque de memoria en el `RawAllocator` pasa por tres estados:

```
ALLOC size
   |
   +--> bloque vivo  (la dirección es válida, los datos son accesibles)
   |          |
   |     REALLOC reg, new_size  (opcional: cambia el tamaño y posiblemente la dirección)
   |          |
   `-----> FREE reg  -->  bloque liberado (la dirección ya no es válida)
```

Si el proceso termina con bloques sin liberar, el destructor de `RawAllocator` llama `free_all()` automáticamente. No hay leaks a nivel de sistema operativo, pero sí recursos desperdiciados durante la ejecución.

---

## Instrucciones bytecode

| Instrucción            | Opcode1 | Opcode2 | Descripción                                               |
| :--------------------- | :-----: | :-----: | :-------------------------------------------------------- |
| `alloc reg_size`       |  `0x00` |  `0xB0` | Reserva `size` bytes contiguos; retorna ptr host en `R0`  |
| `free reg_ptr`         |  `0x00` |  `0xB1` | Libera el bloque cuyo ptr está en `reg`                   |
| `realloc reg_ptr, reg_size` | `0x00` | `0xB2` | Redimensiona el bloque; retorna nuevo ptr en `R0`    |

Todas son instrucciones de **4 bytes** con opcode extendido (`0x00`):

```
[0x00][opcode][ctrl_byte][reg_byte]
ctrl_byte: bits 7-6 = modo (tamaño del registro), bits 5-0 = 0
reg_byte:  bits 7-4 = reg2 (solo realloc), bits 3-0 = reg1
```

### Codificación - `alloc` y `free`

| opcode1 | opcode2 | ctrl_byte        | reg_byte          | total |
| :-----: | :-----: | :--------------- | :---------------- | :---: |
| `0x00`  | `0xB0`/`0xB1` | `mm 000000` | `0000 rrrr` |   4   |

- `mm` = tamaño del registro (`11` = qword recomendado)
- `rrrr` = registro fuente (contiene el tamaño en `alloc`, el puntero en `free`)

`alloc` retorna el puntero host real en `R0`. Si la asignación falla, `R0 = 0`. El bloque se inicializa a cero.  
`free` no hace nada si el puntero no fue asignado por este proceso - no hay crash por double-free.

### Codificación - `realloc`

| opcode1 | opcode2 | ctrl_byte    | reg_byte                  | total |
| :-----: | :-----: | :----------- | :------------------------ | :---: |
| `0x00`  | `0xB2`  | `mm 000000`  | `reg_size\|reg_ptr`       |   4   |

Ambos registros deben ser del mismo tamaño (qword recomendado). Retorna el nuevo puntero en `R0`.

| Caso                    | Resultado                              |
| :---------------------- | :------------------------------------- |
| `realloc r0, r1` donde r0=0  | Equivalente a `alloc`         |
| `realloc r0, r1` donde r1=0  | Equivalente a `free`; R0=0    |
| `new_size > old_size`   | Bytes extra inicializados a cero       |
| `new_size < old_size`   | Bytes sobrantes descartados            |

---

## Ejemplos en ensamblador Vesta (.vel)

### Patrón básico: buffer para una función nativa

```c
// Reservar 256 bytes para pasar a una función C
mov    r1, 256
alloc  r1           // R0 = ptr host (ej. 0x7F3A0000)
mov    r2, r0       // guardar el puntero en r2

// Cargar el cursor y escribir datos en el buffer
xchg   cur0, r2
mov    r5, 0x48656C6C6F000000   // "Hello\0\0\0" en little-endian
writecur cur0, r5

// Pasar el buffer a la función nativa
calln  SomeNativeFunc   // r2 debe estar en R0..R15 segun convencion

// Liberar al terminar
free   r2q           // ptr invalido a partir de aqui
```

### Patrón: crecer el buffer según necesidad

```c
mov    r1, 512
alloc  r1           // R0 = buffer inicial de 512 bytes
mov    r5, r0

// ... mas adelante, el buffer se queda pequeño ...

mov    r6, 2048
realloc r5, r6       // R0 = nuevo ptr de 2048 bytes (r5 ya no es valido)
mov    r5, r0        // actualizar r5 con el nuevo puntero

// ... seguir usando r5 ...

free   r5
```

### Patrón: tabla de strings para interop con C

```c
mov    r1, 1024
alloc  r1            // R0 = buffer de 1 KiB para strings
mov    r8, r0

// Cargar el cursor y rellenar strings
xchg   cur0, r8
// ... writecur para cada campo ...

; Pasar a funcion que espera un array de char*
calln  ParseArgs

free   r8
```

---

## ¿Cuándo usar RawAllocator en lugar de GcHeap?

Usa `RawAllocator` cuando:

- Necesitas pasar un buffer a una función C/Win32/POSIX via [[NativeCall (CallN)|CALLN]].
- La dirección del bloque debe ser estable (no puede cambiar mientras está en uso).
- El ciclo de vida es conocido y acotado: hay un punto claro donde el bloque ya no se necesita y puedes llamar a `FREE`.
- Trabajas con estructuras de datos C que contienen punteros internos (listas enlazadas C, structs anidados) - el GC no puede seguir esos punteros para determinar si están vivos.

Usa `GcHeap` cuando:

- Creas objetos del lenguaje (clases, instancias, closures) cuya vida es difícil de predecir.
- Prefieres no gestionar manualmente la memoria y que el runtime lo haga por ti.
- Los objetos pueden tener referencias entre sí que forman grafos complejos.

---

## Estadísticas disponibles

Accesibles en C++ via `RawAllocator::stats()` (tipo `RawStats`):

| Campo           | Descripción                                            |
| :-------------- | :----------------------------------------------------- |
| `alloc_count`   | Llamadas exitosas a `ALLOC`                            |
| `alloc_bytes`   | Bytes totales asignados                                |
| `free_count`    | Llamadas exitosas a `FREE`                             |
| `freed_bytes`   | Bytes totales liberados                                |
| `realloc_count` | Llamadas a `REALLOC` (incluye el caso `new_size == 0`) |
| `peak_bytes`    | Máximo de bytes vivos simultáneamente (high-water mark)|

> `realloc_count` se incrementa siempre, incluso cuando `new_size == 0` (free-por-realloc), porque la operación fue solicitada.

Un proceso sano debería tener `alloc_bytes - freed_bytes` cercano a cero al terminar (toda la memoria reservada fue liberada). Si `alloc_bytes >> freed_bytes` al final del proceso, hay una fuga.

---

## Seguridad de hilos

`RawAllocator` es **per-proceso** y no thread-safe. La sincronización la aporta el planificador (`Scheduler`): un `ProcessVM` solo corre en un hilo nativo a la vez.

Ver también: [[GC]], [[Generacional (para objetos OOP)|GcHeap]], [[NativeCall (CallN)]], [[VmInstance]]
