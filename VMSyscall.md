Cuando se llama a [[wmint (VM Interuptions)|wmint]] con el ID `0x0F` se procede a llamar a una syscall virtual definida en la tabla de llamadas de la VM. La tabla de llamadas es un puntero mapeado por la VM donde  tiene una cantidad de ``1024`` entradas (``(4096 * 2) / 8``), es decir, ocupa dos paginas de memoria donde cada entrada es un puntero de 8bytes, mientras que en 32 bytes la tabla tiene una cantidad de 1024 entrada compuesta de `(4096 / 4)`, por lo tanto las interrupciones van desde ``0x0`` hasta `0x400` (1024).

```js
mov r00, 0  // indicamos que la syscall virtual a ejecutar sera la 0x0
wmint 0x0F  // invocar la interrupcion virtual 0x0F que crea una syscall virtual
```

La primera syscall virtual (`vmsyscall(0)`)  siempre apuntara a una subrutina que permite invocar syscall reales del sistema donde se ejecute la VM. 

## `vmsycall(0)` en los distintos sistemas:
Las syscall virtual apuntara a subrutinas de codigo real que toman distinta forma segun el OS o metal bare y la arquitectura.
### En linux 32bits:
En linux se define lo siguiente:
- ``EAX``:	Número de syscall
- ``EBX``:	Primer argumento (arg1)
- ``ECX``:	Segundo argumento (arg2)
- ``EDX``:	Tercer argumento (arg3)
- ``ESI``:	Cuarto argumento (arg4)
- ``EDI``:	Quinto argumento (arg5)
- ``EBP``:	Sexto argumento (arg6)

Por lo tanto, para invocar a `sys_write` que tiene el syscall ID `0x4` en linux, se indicara el ID en la VM a través del registro virtual `r01` que será sinónimo de `EAX`, mientras que `r02` será `EBX` y así sucesivamente, a la subrutina a la que se apunta con `vmsycall(0)` en este caso tiene la siguiente forma:

```js
vmsyscall_0:
	int 0x80
```
donde se salta a esta subrutina  poniendo los valores indicados en los registros.

El código:
```js
mov eax, 4        // syscall número 4 = sys_write
mov ebx, 1        // fd = 1 (stdout)
mov ecx, mensaje  // puntero al buffer
mov edx, 5        // longitud
int 0x80          // invoca la syscall
```

Equivaldría a:
```js
mov r00, 4        // syscall número 4 = sys_write
mov r01, 1        // fd = 1 (stdout)
mov r02, mensaje  // puntero al buffer
mov r03, 5        // longitud

// equivalente a int 0x80, invoca la syscall real
mov r00, 0  // indicamos que la syscall virtual a ejecutar sera la 0x0
wmint 0x0F  // invocar la interrupcion virtual 0x0F que crea una syscall virtual
```

### En linux 64bits:
En linux se define lo siguiente:
- ``RAX``:	Número de syscall
- ``RDI``:	Primer argumento (arg1)
- ``RSI``:	Segundo argumento (arg2)
- ``RDX``:	Tercer argumento (arg3)
- ``R10``:	Cuarto argumento (arg4)
- ``R8``:    	Quinto argumento (arg5)
- ``R9``:	        Sexto argumento (arg6)

Por lo tanto, para invocar a `sys_write` que tiene el syscall ID `0x4` en linux, se indicara el ID en la VM a través del registro virtual `r01` que será sinónimo de `RAX`, mientras que `r02` será `RDI` y así sucesivamente, a la subrutina a la que se apunta con `vmsycall(0)` en este caso tiene la siguiente forma:

```js
vmsyscall_0:
	syscall
```
donde se salta a esta subrutina  poniendo los valores indicados en los registros.

El código:
```js
mov     rax, 1          // syscall número 1 = sys_write
mov     rdi, 1          // fd = 1 (stdout)
mov     rsi, mensaje    // buffer
mov     rdx, 5          // longitud
syscall                 // realiza la syscall
```

Equivaldría a:
```js
mov r00, 4        // syscall número 4 = sys_write
mov r01, 1        // fd = 1 (stdout)
mov r02, mensaje  // puntero al buffer
mov r03, 5        // longitud

// equivalente a int 0x80, invoca la syscall real
mov r00, 0  // indicamos que la syscall virtual a ejecutar sera la 0x0
wmint 0x0F  // invocar la interrupcion virtual 0x0F que crea una syscall virtual
```

### En Windows 32bits (aun no definido):
En Windows 32-bit moderno, las funciones de `ntdll.dll` usan instrucciones como:
```js
mov eax, syscall_number
mov edx, fs:[0xC0]       // dirección del KERNEL32.sys service dispatcher
call edx                 // (antiguamente era int 0x2e)
```

|Registro|Uso|
|---|---|
|`EAX`|Número de syscall|
|`EDX`|Dirección del `KiFastCallEntry` (dispatcher)|
|`ESP`|Puntero a los argumentos|
|`EAX`|Retorno (valor o NTSTATUS)|

En versiones anteriores (Windows XP y anteriores), se usaba:
```js
int 0x2E
```
### En Windows 64bits:

|Registro|Uso|
|---|---|
|`RAX`|Número de syscall|
|`RCX`|Primer argumento|
|`RDX`|Segundo argumento|
|`R8`|Tercer argumento|
|`R9`|Cuarto argumento|
|`[RSP+8]`|Quinto argumento|
|`[RSP+16]`|Sexto argumento|
```js
mov     r10, rcx      // Windows requiere que r10 contenga la dirección de retorno para syscall
mov     eax, 0x3D     // syscall number for NtWriteFile (puede variar)
syscall               // transición al kernel
```

Lo mejor en este caso será hacer una syscall indirecta a alguna función NT de la ntdll. 

## Definir otras syscalls
todos los hilos comparten la tabla de llamadas, esta tabla es apuntada a través de un registro global de la VM donde se accede usando un index que se multiplica por el tamaño de palabra para acceder a la entrada que se quiera.