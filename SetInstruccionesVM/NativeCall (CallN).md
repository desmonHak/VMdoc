Permite hacer una llamada a una función nativa.
Lea [[ConvenciónDeLlamadas]] para saber como pasar los argumentos.

```c
// primera forma de CALLN (0x00 0x55)

mov r15, 0 // este metodo no usa args
calln @Method("kernel32.dll:GetTickCount")

// el valor retornado esta en r0


// segunda forma de CALLN (0x55)
// la segunda forma necesita poner la direccion del metodo
// a llamar en el registro 14, es la forma mas corta de llamar a una instruccion.
mov r15, 0 // este metodo no usa args
mov r14, @Method("kernel32.dll:GetTickCount")
callnr


// LOADLIB + GETPROC -> reflexión nativa

// resolucion de un simibolo en run time en la VM
LOADLIB r0, r14 // cargar el base addres de la dll en r0, r14 debe ser una direccion que apunte a la cadena con el nombre de la dll
GETPROC r14, r12 // GETPROC espera que R0 contenga el base address de la dll que usar para obtener la funcion. en este caso r12 debe contener una direccion que apunte a una cadena del metodo a cargar. r14 en este caso es el registro que guardara el address obtenido.
mov r15, 0 // llamamos a la funcion obtenida con 0 args.
callnr
```

- `r00` -> retorno
- `r01-r12` -> argumentos
- `r15` -> argc
- `r14` -> índice dinámico de función nativa, address al momento de ejecutar la instruccion.
- `r13` -> libre para VM

| instrucción | opcode0 | opcode1 |     direccion     |  tamaño  |
| :---------: | :-----: | :-----: | :---------------: | :------: |
|    CALLN    |  0x00   |  0x55   | address (64 bits) | 10 bytes |
|   CALLNR    |  0x55   |         |                   |  1 byte  |
>La primera call es la que el usuario usa convencionalmente en la forma de:
>`calln @Method("kernel32.dll:GetTickCount")`.
>la segunda call es la que el usuario usa si no se especifica un @Method a llamar, en cuyo caso, la instrucción supone que la dirección del método nativo esta en el registro r14. 
## Proceso de emisión / ensamblado
Las instrucciones CALLN contienen como dirección un valor de 0 al momento de ser ensambladas, al encontrar una instrucción de este tipo el ensamblador crea una relocalización para que el linker cree una tabla de relocalizaciones para el loader. 

Ejemplo ensamblado:
```c
00 55 00 00 00 00 00 00 00 00
```

El ensamblador genera:
```c
RelocEntry {
    offset = posición del índice en el bytecode,
    type   = RELOC_CALLN_IMPORT,
    symbol = "kernel32.dll:GetTickCount"
}
```

## El linker asigna índices a los imports:
```c
Index | Library        | Function
-----------------------------------------
0     | kernel32.dll   | GetTickCount
1     | kernel32.dll   | GetCurrentProcessId
2     | user32.dll     | MessageBoxA
...
```
>El `linker` al momento de crear la sección de funciones importadas, no incluye el índex, solo almacena un offset al nombre de la función y otros offset al nombre de la librería.

Luego **parchea** todas las CALLN:
```c
CALLN 0   -> para GetTickCount
CALLN 1   -> para GetCurrentProcessId
```
El ejecutable final contiene:
- sección `.imports`

## LOADER -> carga la tabla de imports en la VM
```c
vm->imports = vector<ImportEntry>{
    { "kernel32.dll", "GetTickCount",        nullptr },
    { "kernel32.dll", "GetCurrentProcessId", nullptr },
    { "user32.dll",   "MessageBoxA",         nullptr },
};
```

Cada entrada tiene:
```cpp
struct ImportEntry {
    std::string lib;
    std::string func;
    void*       ptr;   // <- siempre empieza en nullptr
};
```

Cuando la VM ejecuta:
```c
calln 0
```

Hace:
```c
void VM::exec_CALLN(uint64_t index) {
    auto& imp = imports[index];

    // Lazy resolve
    if (imp.ptr == nullptr) {
        void* lib  = load_library(imp.lib);
        void* func = get_proc(lib, imp.func);

        if (!func) {
            throw VMException("Import resolution failed");
        }

        imp.ptr = func; // <- cache permanente
    }

    // Preparar argumentos
    int argc = regs[15];
    uint64_t args[12];
    for (int i = 0; i < argc && i < 12; i++)
        args[i] = regs[i + 1]; // r01-r12

    // Llamada real
    uint64_t ret = call_native(imp.ptr, args, argc);

    // Guardar retorno
    regs[0] = ret;
}
```
### Primera llamada
En la primera llamada existe un **overhead**:
> **Trabajo extra, coste adicional o carga innecesaria que ocurre para poder hacer algo, pero que no forma parte del trabajo útil.**
- `imp.ptr == nullptr`
- se carga la DLL
- se resuelve la función
- se guarda en `imp.ptr`
- se llama

### Siguientes llamadas
Después de la primera llama, la dirección ya a sido resuelta por lo que ya no hay **overhead**
- `imp.ptr != nullptr`
- se llama directamente
- cero overhead
