> **Atencion:** Esta pagina documenta la instruccion `GETPROC` para resolucion
> dinamica de simbolos (LOADLIB + GetProcAddress). Existe otra instruccion
> tambien llamada `getproc` (opcode 0x00 0xC6) que almacena el puntero al
> proceso actual y se usa para acceder a la memoria virtual de la VM.
> Vease [[GETPROC_GETVM_GETMGR]] para esa funcionalidad.

---

# GETPROC - Obtener la direccion de una funcion nativa

Una vez cargada una biblioteca nativa con `LOADLIB`, necesitas obtener la direccion de
una funcion especifica dentro de ella. `GETPROC` hace exactamente eso.

Equivalente a `GetProcAddress()` en Windows o `dlsym()` en Linux.

| Instruccion | opcode0 | opcode1 | Modo | Tamano  | Descripcion                                           |
| :---------: | :-----: | :-----: | :--: | :-----: | :---------------------------------------------------- |
| `getproc`   |  0x00   |  0x57   | REG  | 4 bytes | Obtiene dir de funcion; R14 = ptr (0 si no existe)    |

---

## Sintaxis

```c
getproc r_dest, r_name

// R0    debe contener la base address de la DLL (obtenida con loadlib)
// r_name: registro con la VM address de una cadena con el nombre de la funcion
// r_dest: registro donde se almacena la direccion de la funcion nativa
//         (se usa directamente con callnr)
```

---

## Uso tipico

```c
// Suponer que R0 = base address de kernel32.dll (obtenida con loadlib)
// y que all.nombre_func contiene "GetTickCount\0"

mov    r12, @Absolute("all.nombre_func")  // r12 = VM addr de "GetTickCount\0"
getproc r14, r12                          // R14 = direccion de GetTickCount
                                          // (internamente: GetProcAddress(R0, nombre))

// Verificar si la funcion fue encontrada
cmpu   r14, 0
jmp.jeq error_func     // si R14 == 0, la funcion no existe en la DLL

// Llamar la funcion
mov    r15, 0          // 0 argumentos
callnr                 // llama a GetTickCount()

// R0 = resultado
jmp.jmp fin

error_func:
    // la funcion no existe en la DLL
    hlt

fin:
    hlt
```

---

## Patron completo LOADLIB + GETPROC + CALLNR

```c
// En la seccion de datos:
//   lib_name:  .ascii "user32.dll\0"
//   func_name: .ascii "MessageBoxA\0"

code:
    mov   rsp, 0x00FF0000
    mov   rbp, 0x00FF0000

    // 1. Cargar la DLL
    mov   r14, @Absolute("all.lib_name")
    loadlib r0, r14           // R0 = base address de user32.dll

    cmpu  r0, 0
    jmp.jeq fallo

    // 2. Obtener la funcion
    mov   r12, @Absolute("all.func_name")
    getproc r14, r12          // R14 = ptr a MessageBoxA

    cmpu  r14, 0
    jmp.jeq fallo

    // 3. Preparar argumentos y llamar
    // MessageBoxA(HWND hwnd, LPCSTR lpText, LPCSTR lpCaption, UINT uType)
    // Necesitamos punteros HOST reales para los strings -> usar alloc
    mov   r15, 4              // 4 argumentos
    // ... preparar r1=hwnd, r2=texto, r3=titulo, r4=tipo ...
    callnr                    // llama a MessageBoxA

    // R0 = resultado (1=OK, 2=Cancel, etc.)
    hlt

fallo:
    hlt
```

---

## Diferencia con CALLN estatico

| Criterio          | `calln @Method("lib:func")`       | `loadlib` + `getproc` + `callnr`         |
| :---------------- | :-------------------------------- | :--------------------------------------- |
| Cuando se resuelve| En tiempo de carga (lazy)         | En tiempo de ejecucion (dinamico)        |
| Nombre conocido   | Si, en tiempo de compilacion      | No, puede ser una cadena calculada       |
| Flexibilidad      | Menor (nombre fijo)               | Mayor (nombre dinamico, plugins, etc.)   |
| Overhead          | Menor (indice en tabla)           | Mayor (llamada a GetProcAddress/dlsym)   |

Usa `calln @Method(...)` cuando el nombre de la funcion se conoce en tiempo de compilacion.
Usa `loadlib + getproc` cuando el nombre se calcula o viene de fuera (plugins, config).

---

Ver tambien: [[LOADLIB]], [[NativeCall (CallN)]], [[NativePluginAPI]]
