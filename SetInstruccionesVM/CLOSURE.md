# Instrucciones de Closures (Lambda / Funciones de Primera Clase)

Un **closure** (o lambda) es una funcion que "recuerda" el entorno en el que fue creada.
En lenguajes como Python, JavaScript o Kotlin, cuando defines una funcion anonima dentro
de otra funcion, esa funcion puede acceder a las variables locales de la funcion que la
contiene incluso despues de que la funcion exterior haya terminado. Eso es un closure.

VestaVM implementa dos modelos de closure para cubrir distintos casos de uso:

| Modelo  | Instrucciones                          | Descripcion                                           |
| :-----: | :------------------------------------: | :---------------------------------------------------- |
| **GC**  | `mkclosure` / `callclosure`            | Closure gestionado por el GC; requiere `MethodInfo*`. |
| **Raw** | `mkrawclosure` / `callrawclosure`      | Closure sin GC; ciclo de vida manual con `FREE`.      |

---

## Tabla de instrucciones

| Instruccion       | opcode0 | opcode1 | Modo | Tamano  | Descripcion                                        |
| :---------------: | :-----: | :-----: | :--: | :-----: | :------------------------------------------------- |
| `mkclosure`       |  0x00   |  0x20   | REG  | 4 bytes | Crear ClosureObject GC (`r_method`, `r_env` -> R0) |
| `callclosure`     |  0x00   |  0x21   | REG  | 4 bytes | Invocar ClosureObject GC (`r_closure`)             |
| `mkrawclosure`    |  0x00   |  0x22   | REG  | 4 bytes | Crear RawClosureObject sin GC (`r_fn`, `r_env` -> R0)|
| `callrawclosure`  |  0x00   |  0x23   | REG  | 4 bytes | Invocar RawClosureObject (`r_closure`)             |

Todas son instrucciones extendidas (prefijo `0x00`).

Implementacion: `src/runtime/exec_instruction_closure.cpp`

---

## Por que hay dos modelos

- **Modelo GC**: para lambdas dentro del sistema OOP de VestaVM. El runtime las gestiona
  automaticamente con el GC; soportan excepciones y se integran con la herencia.
- **Modelo Raw**: para callbacks FFI (funciones que se pasan a bibliotecas C nativas) o
  cuando se necesita el maximo rendimiento sin el overhead del GC. El programador gestiona
  la memoria manualmente.

---

## Modelo GC: `mkclosure` y `callclosure`

### Estructura ClosureObject en memoria host

```cpp
// Un ClosureObject en el GC heap:
struct alignas(8) ClosureObject {
    ObjectHeader header;   // cabecera GC (class_ptr con CLASS_FLAG_CLOSURE)
    MethodInfo  *method;   // lambda o funcion capturada (del loader)
    uint32_t    *captures; // array de GcHandle de variables capturadas
    size_t       cap_count;// numero de variables capturadas
};
```

El GC rastrea el `ClosureObject` durante el minor y major GC, incluyendo los `GcHandle`
del array de capturas como raices adicionales (para que las variables capturadas no
sean recolectadas mientras el closure siga vivo).

### `mkclosure r_method, r_env` - crear closure GC

```c
mkclosure r1, r2    // R0 = GcHandle del ClosureObject creado
```

| Parametro  | Descripcion                                                               |
| :--------: | :------------------------------------------------------------------------ |
| `r_method` | Registro con el host-pointer a un `MethodInfo` del loader (el lambda).    |
| `r_env`    | Registro con la direccion VM del bloque de entorno (variables capturadas). |
| Resultado  | `R0` = `GcHandle` del nuevo `ClosureObject`, o `GC_NULL_HANDLE` si falla. |

Algoritmo:
1. Lee `method = (MethodInfo*)regs[r_method]` y `env_addr = regs[r_env]`.
2. Aloca `sizeof(ClosureObject)` en el GC heap.
3. Inicializa la cabecera (`OBJ_FLAG_GC_OWNED`), el `method*` y el array de capturas.
4. Devuelve el `GcHandle` en `R0`.

```c
// Suponer que el loader pone el MethodInfo* del lambda en r9:
mov   r8, 0            // sin entorno capturado (puede ser 0 si no hay variables capturadas)
mkclosure r9, r8       // R0 = GcHandle del closure

mov   r11, r0          // guardar el handle
// Comprobar que no fallo:
mov   r3, 0xFFFFFFFF   // GC_NULL_HANDLE
cmpu  r11, r3
jmp.je error_gc        // saltar si fallo

callclosure r11        // invocar el lambda; la ejecucion continua tras el RET del lambda
```

> **Requisito:** `r_method` debe contener un `MethodInfo*` valido con `code_vaddr != 0`.
> Este puntero solo existe despues de que el loader haya cargado y enlazado el archivo.

### `callclosure r_closure` - invocar closure GC

```c
callclosure r11    // invoca el lambda del ClosureObject en r11
```

| Parametro    | Descripcion                                      |
| :----------: | :----------------------------------------------- |
| `r_closure`  | Registro con el `GcHandle` del `ClosureObject`.  |

Algoritmo:
1. Desreferencia el handle para obtener el `ClosureObject*`.
2. Lee `method->code_vaddr`.
3. Calcula `ret_addr = RIP + 4` (instruccion siguiente al `callclosure`).
4. Empuja un `FrameHeader` a `frame_stack` (necesario para `THROW`/`CATCH`).
5. Empuja `ret_addr` en la pila (RSP -= 8).
6. Salta a `method->code_vaddr`.

El lambda invocado debe terminar con `RET`.

---

## Modelo Raw: `mkrawclosure` y `callrawclosure`

### Estructura RawClosureObject en memoria host

```cpp
// Un RawClosureObject en el raw allocator:
struct alignas(8) RawClosureObject {
    uint64_t fn_addr;   // direccion de la funcion (bytecode o nativa)
    uint64_t env_addr;  // direccion VM del bloque de entorno
    size_t   env_size;  // tamano del bloque de entorno en bytes
};
```

El `RawClosureObject` se aloca con el `RawAllocator` (como `malloc`) y **no es rastreado
por el GC**. El bytecode es responsable de liberarlo con `FREE` cuando ya no se necesite.

### `mkrawclosure r_fn, r_env` - crear closure raw

```c
mkrawclosure r1, r2    // R0 = host-pointer al RawClosureObject
```

| Parametro | Descripcion                                                          |
| :-------: | :------------------------------------------------------------------- |
| `r_fn`    | Registro con la direccion de la funcion (label o funcion nativa).    |
| `r_env`   | Registro con la direccion VM del bloque de entorno (0 = sin env).    |
| Resultado | `R0` = puntero raw al `RawClosureObject`, o `0` si falla.            |

```c
// Crear un raw closure que apunta a la funcion "sumar":
mov   r1, @Absolute("code.sumar")   // r1 = direccion de la funcion
mov   r2, 0                         // r2 = sin entorno capturado
mkrawclosure r1, r2                 // R0 = ptr al RawClosureObject

mov   r10, r0           // guardar el puntero

// Preparar argumentos y llamar:
mov   r1, 10            // primer argumento
mov   r2, 32            // segundo argumento
mov   r15, 2            // argc = 2 (numero de argumentos)
callrawclosure r10      // llama sumar(10, 32); R0 = 42

// Liberar el closure cuando ya no se necesita:
free  r10               // equivalente a free() en C
```

### `callrawclosure r_closure` - invocar closure raw

```c
callrawclosure r10    // invoca fn_addr del RawClosureObject en r10
```

| Parametro    | Descripcion                                          |
| :----------: | :--------------------------------------------------- |
| `r_closure`  | Registro con el host-pointer al `RawClosureObject`.  |

Algoritmo:
1. Lee `fn_addr = rc->fn_addr`.
2. Copia `env_addr` al registro `R14` (convencion: R14 = entorno del closure raw).
3. Calcula `ret_addr = RIP + 4`.
4. Empuja `ret_addr` en la pila (RSP -= 8). No empuja FrameHeader.
5. Salta a `fn_addr`.

### Convencion de llamada de `callrawclosure`

| Registro | Rol                                                         |
| :------: | :---------------------------------------------------------- |
| R1-R12   | Argumentos de entrada (establecidos por el llamante)        |
| R0       | Valor de retorno (establecido por la funcion)               |
| R14      | Puntero al bloque de entorno (`env_addr` del RawClosure)    |
| R15      | Numero de argumentos (`argc`)                               |

La funcion llamada accede a las variables capturadas leyendo desde la direccion en R14.

---

## Diferencias entre los dos modelos

| Criterio              | GC (`mkclosure`)                    | Raw (`mkrawclosure`)                 |
| :-------------------- | :---------------------------------- | :----------------------------------- |
| Gestor de memoria     | GC generacional                     | RawAllocator (malloc)                |
| Ciclo de vida         | Automatico (GC)                     | Manual (requiere `FREE`)             |
| Requiere              | `MethodInfo*` del loader            | Cualquier direccion de funcion       |
| Soporte THROW/CATCH   | Si (empuja `FrameHeader`)           | No (no empuja `FrameHeader`)         |
| Variables capturadas  | Raices GC (no se recolectan)        | No rastreadas (pueden recolectarse)  |
| Caso de uso tipico    | Lambdas OOP internas del lenguaje   | Callbacks FFI, funciones nativas     |

---

## Codificacion binaria (FIXED_4, modo REG)

```
+--------+--------+----------+----------+
| 0x00   | opcode |  ctrl    |  regs    |
+--------+--------+----------+----------+
  byte0    byte1    byte2      byte3

byte3 para instrucciones de DOS operandos (mkclosure, mkrawclosure):
  bits 7-4 = reg2   (nibble alto: r_env)
  bits 3-0 = reg1   (nibble bajo: r_method o r_fn)

byte3 para instrucciones de UN operando (callclosure, callrawclosure):
  bits 7-4 = 0      (sin uso)
  bits 3-0 = reg1   (registro del closure)
```

---

## Relacion con el sistema OOP

Los closures GC se integran con el sistema OOP de VestaVM:

- El `ObjectHeader` del `ClosureObject` puede tener `class_ptr` apuntando a una
  `ClassInfo` con el flag `CLASS_FLAG_CLOSURE` activado.
- El GC escanea el array `captures` como raices adicionales durante el marcado,
  garantizando que las variables capturadas no sean recolectadas.
- `THROW` y `RETHROW` funcionan correctamente dentro de un lambda invocado con
  `CALLCLOSURE` porque se empuja un `FrameHeader` con la `ret_addr`.

Ver tambien: [[GC]], [[NEWOBJRAW y NEWOBJ]], [[NativeCall (CallN)]]
