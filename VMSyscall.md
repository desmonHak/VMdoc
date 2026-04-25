# VMSyscall - Syscalls virtuales de la VM

Una **syscall** (llamada al sistema) es la forma en que un programa pide al sistema
operativo que haga algo que el programa no puede hacer por si solo: escribir en la
pantalla, leer un archivo, abrir una conexion de red, etc.

VestaVM define su propio mecanismo de syscalls virtuales que se traduce a llamadas
reales del sistema operativo subyacente (Linux, Windows). Esto permite que el mismo
codigo VestaVM funcione en multiples sistemas operativos sin cambios.

Cuando se invoca `wmint 0x0F`, la VM llama a la syscall virtual numero indicada en
`r00`. La tabla de syscalls virtuales tiene **1024 entradas** y ocupa dos paginas de
memoria (4096 bytes x 2 = 8192 bytes con punteros de 8 bytes).

---

## Como invocar una syscall virtual

```c
// Invocar la syscall virtual numero 0 (syscall real del SO)
mov r00, 0      // numero de la syscall virtual (0 = syscall real del SO)
wmint 0x0F      // invocar la interrupcion virtual 0x0F que despacha la syscall
```

La primera syscall virtual (`vmsyscall(0)`) siempre apunta a una subrutina nativa
que invoca syscalls reales del sistema operativo. Las demas entradas pueden ser
definidas por el programador.

---

## vmsyscall(0) en Linux 32-bit

En Linux 32-bit, las syscalls usan la convencion `int 0x80`:

| Registro VM | Registro x86 | Significado              |
| :---------: | :----------: | :----------------------- |
| r00         | EAX          | Numero de syscall        |
| r01         | EBX          | Primer argumento         |
| r02         | ECX          | Segundo argumento        |
| r03         | EDX          | Tercer argumento         |
| r04         | ESI          | Cuarto argumento         |
| r05         | EDI          | Quinto argumento         |
| r06         | EBP          | Sexto argumento          |

**Ejemplo: sys_write en Linux 32-bit**

```c
// Codigo x86 nativo equivalente:
// mov eax, 4       // syscall numero 4 = sys_write
// mov ebx, 1       // fd = 1 (stdout)
// mov ecx, mensaje // puntero al buffer
// mov edx, 5       // longitud
// int 0x80

// Equivalente en VestaVM:
mov r00, 4        // syscall numero 4 = sys_write
mov r01, 1        // fd = 1 (stdout)
mov r02, mensaje  // puntero al buffer
mov r03, 5        // longitud

mov r00, 0        // usar la syscall virtual 0 (syscall real del SO)
wmint 0x0F        // invocar la interrupcion
```

---

## vmsyscall(0) en Linux 64-bit

En Linux 64-bit, las syscalls usan la instruccion `syscall`:

| Registro VM | Registro x64 | Significado              |
| :---------: | :----------: | :----------------------- |
| r00         | RAX          | Numero de syscall        |
| r01         | RDI          | Primer argumento         |
| r02         | RSI          | Segundo argumento        |
| r03         | RDX          | Tercer argumento         |
| r04         | R10          | Cuarto argumento         |
| r05         | R8           | Quinto argumento         |
| r06         | R9           | Sexto argumento          |

**Ejemplo: sys_write en Linux 64-bit**

```c
// Codigo x64 nativo equivalente:
// mov rax, 1        // syscall numero 1 = sys_write
// mov rdi, 1        // fd = 1 (stdout)
// mov rsi, mensaje  // buffer
// mov rdx, 5        // longitud
// syscall

// Equivalente en VestaVM:
mov r00, 1        // syscall numero 1 = sys_write (en x64 es 1, no 4)
mov r01, 1        // fd = 1 (stdout)
mov r02, mensaje  // puntero al buffer
mov r03, 5        // longitud

mov r00, 0        // usar la syscall virtual 0
wmint 0x0F        // invocar la interrupcion
```

---

## vmsyscall(0) en Windows 64-bit

En Windows 64-bit, las syscalls NT usan `syscall` con convencion diferente:

| Registro VM | Registro x64 | Significado              |
| :---------: | :----------: | :----------------------- |
| r00         | RAX          | Numero de syscall NT     |
| r01         | RCX          | Primer argumento         |
| r02         | RDX          | Segundo argumento        |
| r03         | R8           | Tercer argumento         |
| r04         | R9           | Cuarto argumento         |

En Windows se recomienda usar llamadas indirectas a funciones de `ntdll.dll` en lugar
de syscalls directas, ya que los numeros de syscall cambian entre versiones de Windows.

```c
// Windows 64-bit: preparar syscall NT via ntdll
// mov r10, rcx     // Windows requiere r10 = rcx para syscall
// mov eax, ID      // numero de syscall NT
// syscall
```

---

## Definir syscalls adicionales

Las 1024 entradas de la IVT (Interrupt Vector Table) se pueden usar para definir
funciones personalizadas accesibles desde bytecode:

```c
// Todos los hilos comparten la misma tabla de syscalls
// Para registrar una nueva syscall virtual:
// 1. Obtener el puntero base de la tabla
// 2. Escribir el puntero a tu funcion en la entrada deseada
// 3. Invocar con wmint 0x0F + numero de entrada

// La tabla accede a la entrada N con:
// IVT + sizeof(void*) * N
```

---

Ver tambien:
- [[wmint (VM Interuptions).md]] - instruccion wmint y tabla IVT
- [[SetInstruccionesVM/NativeCall (CallN).md]] - forma recomendada de llamar funciones del SO
- [[SetInstruccionesVM/ENC (Exec Native Code).md]] - ejecucion directa de codigo nativo
