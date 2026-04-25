# LOADLIB - Cargar una biblioteca nativa en tiempo de ejecucion

`LOADLIB` carga una biblioteca dinamica (DLL en Windows, .so en Linux) en el proceso y
devuelve su **direccion base** en R0. Esa direccion es necesaria para luego obtener la
direccion de funciones individuales con `GETPROC`.

Esto es equivalente a `LoadLibrary()` en Windows o `dlopen()` en Linux.

| Instruccion | opcode0 | opcode1 | Modo | Tamano  | Descripcion                                          |
| :---------: | :-----: | :-----: | :--: | :-----: | :--------------------------------------------------- |
| `loadlib`   |  0x00   |  0x56   | REG  | 4 bytes | Carga DLL/so; R0 = base address (0 si falla)         |

---

## Sintaxis

```c
loadlib r_dest, r_name

// r_name: registro con la direccion VM de una cadena que contiene el nombre de la lib
// r_dest: registro donde se almacenara la base address de la DLL cargada
//         (tambien se guarda en R0)
```

---

## Uso tipico

```c
// Paso 1: colocar en memoria VM el nombre de la biblioteca
// (en la seccion de datos del .vel)
//   nombre_dll: .ascii "kernel32.dll\0"

// Paso 2: cargar la direccion del nombre en un registro
mov   r14, @Absolute("all.nombre_dll")   // r14 = VM addr de "kernel32.dll\0"

// Paso 3: cargar la DLL
loadlib r0, r14   // R0 = base address de kernel32.dll (0 si no se encuentra)

// Paso 4: verificar si la carga tuvo exito
cmpu  r0, 0
jmp.jeq error_dll    // si R0 == 0, la DLL no se encontro

// La DLL esta cargada; r0 tiene la base address
// Guardar para usarla con GETPROC
mov   r8, r0

// Continuar con GETPROC para obtener funciones...
jmp.jmp continuar

error_dll:
    // la DLL no pudo cargarse (no existe, sin permisos, etc.)
    hlt

continuar:
    hlt
```

---

## Patron completo LOADLIB + GETPROC + CALLNR

```c
// Cargar GetTickCount de kernel32.dll de forma dinamica

// Preparar strings en la seccion de datos:
//   nombre_dll:  .ascii "kernel32.dll\0"
//   nombre_func: .ascii "GetTickCount\0"

// 1. Cargar la DLL
mov    r14, @Absolute("all.nombre_dll")
loadlib r0, r14              // R0 = base address de kernel32.dll
mov    r8, r0                // guardar base address

// 2. Obtener la funcion
mov    r12, @Absolute("all.nombre_func")
getproc r14, r12             // R14 = puntero a GetTickCount (usa R0 como base)

// 3. Llamar la funcion
mov    r15, 0                // 0 argumentos
callnr                       // llama a lo que hay en R14

// R0 = resultado de GetTickCount (milisegundos desde arranque del sistema)
```

---

## Notas

- El nombre de la biblioteca es una cadena terminada en `\0` almacenada en la **VM
  memory**. La VM la lee y convierte a un puntero host antes de llamar a `LoadLibrary`.
- Si la biblioteca ya estaba cargada, el sistema operativo devuelve la misma base address
  (sin recargarla).
- `R0 = 0` indica fallo (DLL no encontrada, sin permisos, formato incorrecto, etc.).
- Las DLLs cargadas con `loadlib` no se descargan automaticamente cuando el proceso VM
  termina; el sistema operativo las libera al cerrar el proceso.

---

Ver tambien: [[GETPROC]], [[NativeCall (CallN)]], [[NativePluginAPI]]
