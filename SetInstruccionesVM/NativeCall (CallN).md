# NativeCall (CALLN y CALLNR) - Llamadas a funciones nativas

Hasta ahora hemos visto instrucciones que operan dentro de la VM. Pero muchas veces
necesitamos acceder a funcionalidad del sistema operativo o de bibliotecas externas:
imprimir en pantalla, leer archivos, usar OpenSSL para cifrado, llamar a la Win32 API, etc.

Para eso existe `CALLN` (Call Native): una instruccion que llama directamente a una funcion
escrita en C o C++ que esta fuera de la VM.

Analogia: la VM es una oficina cerrada con sus propios trabajadores. A veces necesitas
llamar a un proveedor externo. `CALLN` es el telefono que conecta la oficina con el mundo
exterior.

| Instruccion | opcode0 | opcode1 |  Tamano  | Descripcion                                       |
| :---------: | :-----: | :-----: | :------: | :------------------------------------------------ |
| `calln`     |  0x00   |  0x55   | 10 bytes | Llamada a funcion nativa por indice (con @Method) |
| `callnr`    |  0x55   |   ---   |  1 byte  | Llamada a funcion nativa cuya dir esta en R14     |

---

## Convencion de llamada

Antes de ejecutar `calln` o `callnr`, los argumentos deben estar en los registros
correctos segun esta convencion:

| Registro | Rol                                              |
| :------: | :----------------------------------------------- |
| R0       | Valor de retorno (la VM lo escribe aqui)         |
| R1-R12   | Argumentos de entrada (el llamante los pone)     |
| R13      | Libre para uso interno de la VM                  |
| R14      | Direccion de la funcion (solo para `callnr`)     |
| R15      | Numero de argumentos (`argc`)                    |

Si la funcion recibe 3 argumentos, se ponen en R1, R2, R3 y R15 = 3.
Si no recibe argumentos, R15 = 0.

> **Sin contexto implicito:** las funciones nativas exportadas para `calln` NO reciben
> ningun `void *ctx` implicito como primer parametro. La convencion es: R1-R12 son los
> argumentos y nada mas. Esto es diferente de algunas convenciones de callback.

---

## CALLN - llamada por nombre de libreria y funcion

```c
mov   r15, 0                                    // 0 argumentos
calln @Method("kernel32.dll:GetTickCount")      // R0 = valor de retorno
```

```c
// Llamar con argumentos: mostrar un mensaje en Windows
mov   r1, 0                        // hwnd = NULL
mov   r2, @Absolute("all.msg")     // puntero VM al texto (ver nota abajo)
mov   r3, @Absolute("all.titulo")  // puntero VM al titulo
mov   r4, 0                        // MB_OK
mov   r15, 4                       // 4 argumentos
calln @Method("user32.dll:MessageBoxA")
// R0 = resultado (IDOK = 1, etc.)
```

> **Nota importante:** las funciones de Windows esperan punteros host reales, no
> direcciones VM. Para pasar strings o buffers a funciones nativas, primero copia los
> datos a un bloque `alloc` y pasa ese puntero. Ver [[cursor]] y [[GC/Allocator crudo]].

---

## CALLNR - llamada a direccion en R14 (forma corta)

```c
mov   r14, @Method("kernel32.dll:GetTickCount")   // poner la dir en R14
mov   r15, 0                                       // 0 argumentos
callnr                                             // 1 byte, llama a lo que hay en R14
// R0 = valor de retorno
```

`CALLNR` es identico a `CALLN` pero la direccion de la funcion la lee de R14 en lugar
de estar embebida en el bytecode. Util cuando la direccion de la funcion es dinamica
(calculada en tiempo de ejecucion).

---

## Pipeline completo: de .vel a ejecucion

### Paso 1: Ensamblado

Cuando el ensamblador encuentra `calln @Method("kernel32.dll:GetTickCount")`, genera:

1. Los bytes de la instruccion con la direccion en cero (placeholder):
   ```
   00 55 00 00 00 00 00 00 00 00
   ```

2. Una **entrada de relocalizacion**:
   ```cpp
   RelocEntry {
       offset = posicion del indice en el bytecode,
       type   = RELOC_CALLN_IMPORT,
       symbol = "kernel32.dll:GetTickCount"
   }
   ```

### Paso 2: Enlazado (Linker)

El linker asigna un indice numerico a cada funcion importada y crea una tabla de imports:

```
Indice | Libreria       | Funcion
------------------------------------------
  0    | kernel32.dll   | GetTickCount
  1    | kernel32.dll   | GetCurrentProcessId
  2    | user32.dll     | MessageBoxA
```

Luego **parchea** todas las instrucciones `CALLN` con su indice correspondiente:

```
calln 0   // para GetTickCount
calln 1   // para GetCurrentProcessId
calln 2   // para MessageBoxA
```

El ejecutable final (.velb) contiene una seccion `.imports` con esta informacion.

### Paso 3: Carga (Loader)

El loader lee la seccion `.imports` y construye la tabla de imports en memoria:

```cpp
vm->imports = vector<ImportEntry>{
    { "kernel32.dll", "GetTickCount",        nullptr },
    { "kernel32.dll", "GetCurrentProcessId", nullptr },
    { "user32.dll",   "MessageBoxA",         nullptr },
};
```

Cada entrada tiene:
```cpp
struct ImportEntry {
    std::string lib;    // nombre de la biblioteca
    std::string func;   // nombre de la funcion
    void*       ptr;    // nullptr hasta la primera llamada (lazy resolve)
};
```

### Paso 4: Ejecucion (Lazy resolve)

La primera vez que la VM ejecuta `calln 0`:

```cpp
void VM::exec_CALLN(uint64_t index) {
    auto& imp = imports[index];

    if (imp.ptr == nullptr) {
        // Primera llamada: resolver la funcion
        void* lib  = load_library(imp.lib);    // LoadLibrary / dlopen
        void* func = get_proc(lib, imp.func);  // GetProcAddress / dlsym

        if (!func) {
            throw VMException("Import resolution failed: " + imp.func);
        }

        imp.ptr = func;  // cachear permanentemente para futuras llamadas
    }

    // Preparar argumentos desde los registros R1-R12
    int argc = regs[15];
    uint64_t args[12];
    for (int i = 0; i < argc && i < 12; i++)
        args[i] = regs[i + 1];

    // Ejecutar la funcion nativa real
    uint64_t ret = call_native(imp.ptr, args, argc);

    // Guardar el valor de retorno en R0
    regs[0] = ret;
}
```

**Primera llamada:** overhead de `LoadLibrary` + `GetProcAddress`. El puntero se cachea.
**Siguientes llamadas:** cero overhead; se llama directamente con `imp.ptr`.

---

## Resolucion dinamica con LOADLIB y GETPROC

Para resolver un simbolo en tiempo de ejecucion (sin conocer el nombre en tiempo de
compilacion), usa las instrucciones `loadlib` + `getproc`:

```c
// Cargar una DLL cuyo nombre esta en VM memory en all.nombre_dll
mov   r14, @Absolute("all.nombre_dll")  // r14 = VM addr de la cadena con el nombre
loadlib r0, r14   // R0 = base address de la DLL cargada

// Obtener la direccion de una funcion dentro de esa DLL
mov   r12, @Absolute("all.nombre_func") // r12 = VM addr del nombre de la funcion
getproc r14, r12  // R14 = direccion de la funcion (usa el base en R0)

// Llamar a la funcion obtenida dinamicamente
mov   r15, 0      // 0 argumentos
callnr            // llama a lo que hay en R14
// R0 = valor de retorno
```

Este patron es el equivalente VM de `LoadLibrary` + `GetProcAddress` en Windows, o
`dlopen` + `dlsym` en Linux. Permite cargar plugins o decidir que funcion llamar en
tiempo de ejecucion.

---

## Uso con la stdlib nativa de VestaVM

VestaVM incluye una stdlib nativa en `stdlib/native/`. Para usarla:

```c
// Imprimir un entero (vesta_io.vio_print_int)
getproc r1          // R1 = ProcessVM* del proceso actual
mov     r2, 42
mov     r15, 2
calln   @Method("stdlib/native/io/vesta_io:vio_print_int")

// Calcular la raiz cuadrada (vesta_math.vio_sqrt)
// La convencion de vesta_math usa bits IEEE 754 para doubles:
// sqrt(4.0) -> pasar 0x4010000000000000 (bits de 4.0 en float64)
mov   r1, 0x4010000000000000   // bits de 4.0
mov   r15, 1
calln @Method("stdlib/native/math/vesta_math:vio_sqrt")
// R0 = bits de 2.0 = 0x4000000000000000
```

---

## Codificacion binaria

### CALLN (10 bytes)

```
+--------+--------+--------+--------+--------+--------+--------+--------+--------+--------+
| 0x00   | 0x55   |  idx0  |  idx1  |  idx2  |  idx3  |  idx4  |  idx5  |  idx6  |  idx7  |
+--------+--------+--------+--------+--------+--------+--------+--------+--------+--------+
  byte0    byte1   bytes 2-9: indice de 64 bits en little-endian (= indice en tabla imports)
```

El indice de 64 bits referencia la entrada de la tabla de imports. En el ensamblado inicial
vale 0; el linker lo parchea con el indice correcto.

### CALLNR (1 byte)

```
+--------+
| 0x55   |
+--------+
```

Un solo byte. La direccion de la funcion se lee de R14 en tiempo de ejecucion.

---

## Resumen de instrucciones relacionadas

| Instruccion  | opcode | Descripcion                                                    |
| :----------: | :----: | :------------------------------------------------------------- |
| `calln`      | 00 55  | Llamar funcion por indice (embebido en bytecode por el linker) |
| `callnr`     | 55     | Llamar funcion cuya dir esta en R14 (1 byte)                   |
| `loadlib`    | 00 56  | Cargar DLL/so en memoria; R0 = base address                    |
| `getproc`    | 00 57  | Obtener dir de funcion de una DLL cargada; R14 = dir           |

---

Ver tambien: [[GC/Allocator crudo para FFI y memoria manual]], [[cursor]], [[NativePluginAPI]]
