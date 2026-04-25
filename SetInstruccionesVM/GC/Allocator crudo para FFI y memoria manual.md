# RawAllocator - Memoria cruda para FFI y gestion manual

Allocator manual por proceso sin GC automatico. Retorna **punteros host reales** que se
pueden pasar directamente a funciones nativas (C/C++, Win32 API, OpenSSL, etc.) a traves
de las instrucciones de llamada nativa (`calln`).

> Si no conoces que es la gestion de memoria ni la diferencia entre manual y automatico,
> empieza por [[GC]] antes de leer esta pagina.

---

## Que es FFI y por que necesita punteros reales

**FFI** (Foreign Function Interface) es el mecanismo por el que VestaVM llama a funciones
escritas en otro lenguaje, normalmente C. Ejemplos: llamar a `WriteFile` de Windows, usar
OpenSSL para cifrado, invocar una biblioteca de graficos, leer de un socket, etc.

El problema es que esas funciones esperan recibir **la direccion real de un bloque de bytes
en la RAM del proceso**. No entienden handles, no entienden objetos con cabeceras, no
entienden la HandleTable del GC. Solo entienden "dame un puntero a N bytes contiguos".

```c
// Funcion C que espera un puntero a un buffer de 256 bytes:
int WriteFile(HANDLE hFile, const void *lpBuffer, DWORD nBytesToWrite, ...);
//                                   ^^^^^^^^^^^
//                                   necesita la direccion real en la RAM del proceso
```

El `GcHeap` no puede satisfacer esto directamente: los handles son indices, no punteros,
y el GC puede mover objetos. El `RawAllocator` si puede: reserva un bloque contiguo en
la RAM del host y devuelve su direccion real.

---

## Que significa "contiguo"

Un bloque de N bytes es **contiguo** cuando `byte[0]`, `byte[1]`, ..., `byte[N-1]` estan
en posiciones consecutivas de memoria sin huecos ni saltos. La mayoria de las funciones C
que reciben buffers asumen esto.

Si la memoria no fuera contigua, una funcion como `memcpy` que copia desde la direccion
dada hasta `direccion + N` leeria memoria que no pertenece al buffer, con resultados
impredecibles (datos incorrectos, crash, vulnerabilidad de seguridad).

---

## Por que no se usa la capa VirtualMemory / TLB

La capa VirtualMemory/TLB de VestaVM divide las reservas en paginas de 4096 bytes y las
mapea en rangos **virtuales de la VM**, no en el espacio de direcciones del proceso host.
Un bloque que cruce un limite de pagina puede tener bytes fisicos no contiguos desde el
punto de vista del host.

```c
// Vista TLB (virtual VM):    [ 0x1000 .. 0x1FFF ][ 0x2000 .. 0x2FFF ]
//                                     |                    |
// Vista host (fisica):       [  pagina A en RAM  ][  pagina B en RAM  ]
//                             (pueden NO ser adyacentes en RAM del host)
//
// Si una funcion C recibe 0x1000 y lee 8192 bytes, accede a pagina A + pagina B.
// Si en el host esas paginas no son adyacentes -> accede a memoria ajena -> crash.
```

`RawAllocator` llama directamente a `VirtualAlloc` (Windows) o `mmap` (POSIX), que
garantiza un **rango contiguo unico** en el espacio de direcciones real del proceso host,
equivalente a `malloc(3)` pero sin fragmentador intermedio.

---

## Diferencia con el GcHeap

| Aspecto                   | GcHeap                                 | RawAllocator                          |
| :------------------------ | :------------------------------------- | :------------------------------------ |
| Que devuelve al bytecode  | Handle opaco (numero entero)           | Puntero host real (direccion RAM)     |
| El objeto se puede mover  | Si (el handle oculta el movimiento)    | No (la direccion es permanente)       |
| Compatible con FFI        | No                                     | Si                                    |
| Ciclo de vida             | Automatico (GC recoge la basura)       | Manual (`free` obligatorio)           |
| Riesgo si se olvida       | Fuga leve (el proceso la recoge al salir) | Fuga hasta que el proceso termina  |

---

## Ciclo de vida de un bloque

```c
// alloc size
//    |
//    +--> bloque vivo  (la direccion es valida, los datos son accesibles)
//    |          |
//    |     realloc reg, new_size  (opcional: cambia el tamano, posiblemente la dir)
//    |          |
//    `-----> free reg  -->  bloque liberado (la direccion ya no es valida)
```

Si el proceso termina con bloques sin liberar, el destructor de `RawAllocator` llama
`free_all()` automaticamente. No hay leaks a nivel de sistema operativo, pero si recursos
desperdiciados durante la ejecucion.

---

## Instrucciones bytecode

| Instruccion                  | Opcode0 | Opcode1 | Descripcion                                              |
| :--------------------------- | :-----: | :-----: | :------------------------------------------------------- |
| `alloc reg_size`             |  0x00   |  0xB0   | Reserva `size` bytes contiguos; retorna ptr host en R0   |
| `free reg_ptr`               |  0x00   |  0xB1   | Libera el bloque cuyo ptr esta en `reg`                  |
| `realloc reg_ptr, reg_size`  |  0x00   |  0xB2   | Redimensiona el bloque; retorna nuevo ptr en R0          |

Todas son instrucciones de **4 bytes** con opcode extendido (`0x00`):

```
[0x00][opcode][ctrl_byte][reg_byte]
ctrl_byte: bits 7-6 = modo (tamano del registro), bits 5-0 = 0
reg_byte:  bits 7-4 = reg2 (solo realloc), bits 3-0 = reg1
```

### ALLOC

```c
alloc r1    // R0 = puntero host al bloque de r1 bytes (0 si falla)
```

- El bloque se inicializa a cero.
- Si la asignacion falla: `R0 = 0`.
- El puntero devuelto en R0 es la **direccion real en la RAM del host**.

### FREE

```c
free r2    // libera el bloque apuntado por r2; r2 ya no es valido despues
```

- No hace nada si el puntero no fue asignado por este proceso.
- No hay crash por double-free.

### REALLOC

```c
realloc r5, r6    // R0 = nuevo ptr del bloque redimensionado a r6 bytes
```

| Caso                        | Resultado                              |
| :-------------------------- | :------------------------------------- |
| `realloc` con ptr=0         | Equivalente a `alloc`                  |
| `realloc` con new_size=0    | Equivalente a `free`; R0=0             |
| `new_size > old_size`       | Bytes extra inicializados a cero       |
| `new_size < old_size`       | Bytes sobrantes descartados            |

> **Advertencia:** tras `realloc`, el puntero original en `r5` puede quedar invalido
> porque el bloque puede moverse a una nueva direccion. Siempre actualiza tu variable
> con el nuevo valor de R0.

---

## Ejemplos en bytecode Vesta

### Patron basico: buffer para una funcion nativa

```c
// Reservar 256 bytes para pasar a una funcion C
mov    r1, 256
alloc  r1           // R0 = ptr host (ej. 0x7F3A0000)
mov    r2, r0       // guardar el puntero en r2

// Comprobar si la asignacion tuvo exito
cmpu   r2, 0
jmp.jeq alloc_fallo

// Cargar el cursor y escribir datos en el buffer
mov    r14, r2
xchg   cur0, r14    // cur0 = ptr al buffer
mov    r5, 0x48656C6C6F000000   // "Hello\0\0\0" en little-endian
writecur cur0, r5

// Pasar el buffer a la funcion nativa (convencion: r1 = ptr, r2 = tamano)
mov    r1, r2
mov    r2, 256
mov    r15, 2
calln  @Method("stdlib/native/io/vesta_io:vio_write")

// Liberar al terminar
free   r2           // ptr invalido a partir de aqui
jmp.jmp fin

alloc_fallo:
    // sin memoria disponible
    hlt

fin:
    hlt
```

### Patron: crecer el buffer segun necesidad

```c
mov    r1, 512
alloc  r1           // R0 = buffer inicial de 512 bytes
mov    r5, r0

// ... mas adelante, el buffer se queda pequeno ...

mov    r6, 2048
realloc r5, r6      // R0 = nuevo ptr de 2048 bytes
mov    r5, r0       // IMPORTANTE: actualizar r5 con el nuevo puntero
                    // (r5 antiguo puede no ser valido despues de realloc)

// ... seguir usando r5 ...

free   r5
```

### Patron: tabla de strings para interop con C

```c
// Reservar un buffer de 1 KiB para strings de resultados
mov    r1, 1024
alloc  r1            // R0 = buffer de 1 KiB
mov    r8, r0

// Comprobar si fallo
cmpu   r8, 0
jmp.jeq sin_memoria

// Cargar el cursor y rellenar strings
mov    r14, r8
xchg   cur0, r14     // cur0 = inicio del buffer
// ... writecur para cada string ...

// Pasar a funcion que espera un array de char* con el formato de C
mov    r1, r8
mov    r15, 1
calln  @Method("mylib:parse_args")

// Liberar cuando ya no se necesite
free   r8
jmp.jmp fin

sin_memoria:
    hlt

fin:
    hlt
```

---

## Cuando usar RawAllocator en lugar de GcHeap

Usa `RawAllocator` cuando:

- Necesitas pasar un buffer a una funcion C/Win32/POSIX via `calln`.
- La direccion del bloque debe ser estable (no puede cambiar mientras esta en uso).
- El ciclo de vida es conocido y acotado: hay un punto claro donde ya no se necesita.
- Trabajas con estructuras de datos C que contienen punteros internos (listas enlazadas
  C, structs anidados). El GC no puede seguir esos punteros internos.

Usa `GcHeap` cuando:

- Creas objetos del lenguaje (clases, instancias, closures) cuya vida es dificil de
  predecir.
- Prefieres no gestionar manualmente la memoria.
- Los objetos forman grafos complejos de referencias entre si.

---

## Estadisticas disponibles

Accesibles en C++ via `RawAllocator::stats()` (tipo `RawStats`):

| Campo           | Descripcion                                             |
| :-------------- | :------------------------------------------------------ |
| `alloc_count`   | Llamadas exitosas a `alloc`                             |
| `alloc_bytes`   | Bytes totales asignados                                 |
| `free_count`    | Llamadas exitosas a `free`                              |
| `freed_bytes`   | Bytes totales liberados                                 |
| `realloc_count` | Llamadas a `realloc` (incluye el caso `new_size == 0`) |
| `peak_bytes`    | Maximo de bytes vivos simultaneamente (high-water mark) |

Un proceso sano deberia tener `alloc_bytes - freed_bytes` cercano a cero al terminar
(toda la memoria reservada fue liberada). Si `alloc_bytes >> freed_bytes` al final, hay
una fuga de memoria.

---

## Seguridad de hilos

`RawAllocator` es **per-proceso** y no thread-safe. La sincronizacion la aporta el
planificador (`Scheduler`): un `ProcessVM` solo corre en un hilo nativo a la vez.

---

Ver tambien: [[GC]], [[Generacional (para objetos OOP)]], [[NativeCall (CallN)]], [[cursor]]
