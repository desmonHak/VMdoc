La VM usa su propia convención de llamadas para conocer cuantos parámetros se han usado al hacer la llamada a una función interna (código que esta dentro de la VM) o una función externa (función real que no es parte de la VM y es necesario generar código para realizar la llamada).

## Convención para llamadas nativas / externas a la VM:

Registros:
- ``r00``: valor de retorno de la llamada si retorna
- `r15`: cantidad de parámetros de la función. máximo de ``r1`` a ``r12`` = 12 argumentos
- `r01-r12`: los registros se usan para pasar los 12 parámetros de la función

```c
mov r15, 1 // este metodo usa 1 args
mov r1, hola_mundo // ponemos el puntero de la direccion a imprimir
calln @Method("libc.dll:puts")
// el valor retornado esta en r0
hlt

hola_mundo db "Hola mundo", 0x0
```
vea [[NativeCall (CallN)]]
>Esta convención define un máximo de 12 parámetros para funciones nativas.

## Convención de llamadas para funciones internas.
- ``r00``: valor de retorno de la llamada si retorna
- `r15`: cantidad de parámetros de la función. 
- ``r01-r05``: los primeros 5 parámetros se posicionan en los 5 registros, en caso de usa mas, se deberá pasar a través del stack. Los argumentos extra se empujan en orden natural (izquierda -> derecha)

Ejemplo:
```c
foo(a0, a1, a2, a3, a4, a5, a6)
```
VM haría:
- `r01 = a0`
- `r02 = a1`
- `r03 = a2`
- `r04 = a3`
- `r05 = a4`
- ``stack[0]`` = a5
- ``stack[1]`` = a6

Cuando haces CALL:
1. Empujas los argumentos extra
2. Empujas la dirección de retorno
3. Saltas a la función

Ejemplo:
```c
push a5
push a6
push return_address
jmp fn
```
**``stack[0]`` = arg6, ``stack[1]`` = arg7, ``stack[2]`` = arg8…**

```c
SP -> arg5
SP+8 -> arg6
SP+16 -> arg7
...
```

### El callee NO limpia la pila
El caller limpia la pila después del retorno.
Esto es lo que hacen:
- System V
- ARM64
- RISC‑V

Supongamos:
```c
foo(a0, a1, a2, a3, a4, a5, a6)
```

1. Cargar registros
```c
r01 = a0
r02 = a1
r03 = a2
r04 = a3
r05 = a4
```

2. Empujar argumentos extras:
```c
push a5
push a6
```

3. Empujar return address (lo hace [[CALLVM]] automáticamente)
```c
push rip_next
```

4. Saltar a la función (lo hace [[CALLVM]] automáticamente)
```c
rip = fn_address
```

5. Callee ejecuta [[ENTER]]
```c
enter 32
```
esto equivale a:
```c
push rbp
mov rbp, rsp
sub rsp, 32
```

Ahora el stack frame queda así:
```c
[rbp+16] = a6
[rbp+8]  = a5
[rbp+0]  = return_address
[rbp-8]  = local0
[rbp-16] = local1
...
```

6. Al finalizar la función, se ejecuta antes de retorna una instrucción [[LEAVE]]
```c
leave
```
esto equivale a:
```c
mov rsp, rbp
pop rbp
```

7. Una vez se retorno al punto donde se hizo la llamada, el caller debe limpiar la pila, usando sub/add para modificar los registros de pila, o usando la instrucción ``pop``:
```c
pop a6
pop a5
```

## Stack Trace
Esta convención de llamada nos permite luego poder hacer sistemas de depuración y hacer stack trace:
1. Leer el `rbp` actual.
2. Leer `[rbp+0]` → dirección de retorno.
3. Leer `[rbp+8]` → frame anterior.
4. Repetir hasta llegar a un frame nulo.

Ejemplo:
```c
frame0_rbp = vm->registers.rbp
frame0_ret = mem[frame0_rbp + 0]
frame1_rbp = mem[frame0_rbp + 8]
frame1_ret = mem[frame1_rbp + 0]
frame2_rbp = mem[frame1_rbp + 8]
...
```

Esto da:
```c
fnA -> fnB -> fnC -> fnD
```

>Para que esto se pueda realizar, todas las funciones deben tener stack frame, y se debe guardar un mapa de símbolos del tipo nombre -> direccion:
```c
at foo() [0x1234]
at bar() [0x5678]
at main() [0x9ABC]
```

## TCO - para llamadas recursivas.
**qué es “tail position”, y cómo se implementa TCO**.

### 1. ¿Cuándo se puede hacer TCO?
Una llamada está en _posición de cola_ cuando:
- es **lo último** que hace la función,
- después de la llamada **no hay más código** que usar el resultado salvo retornarlo.

En pseudo‑VM:
```c
// cuerpo...
prep_args_for foo
call foo
ret          // <- esto es TCO candidate
```
Si hay cualquier cosa después de la llamada (sumar, comparar, etc.), **no hay TCO**.

### 2. Idea clave: "no llames, salta reutilizando el frame"
Ahora mismo, una llamada normal hace:
1. Empujar args extra en stack
2. Empujar return address
3. `rip = fn_address`
4. En callee: `enter` crea frame
5. Al final: `leave` destruye frame
6. `ret` vuelve al caller
7. Caller limpia args extra

Con TCO queremos:
- **no crear un nuevo frame**,
- **no empujar nueva return address**,
- **reutilizar el frame actual**,
- **saltar directamente a la función destino**.

Es decir: convertir:
```c
CALLVM foo
RET
```
en algo como:
```c
// reescribir argumentos en el frame actual
// ajustar stack si hay args extra
// saltar a foo como si esta función “se convirtiera” en foo
JMP foo
```
Para hacer TCO:
1. **No haces** `CALLVM`, hace una `TAILCALLVM` lógico.
2. **No empujas return address** (ya tiene una en `[rbp+0]`).
3. **No haces** `enter` **nuevo** (reutiliza el mismo `rbp`/`rsp` como frame de la nueva función).
4. Solo:
    - pone los nuevos argumentos en `r01–r05` y en el stack (encima del frame actual o reusando zona de args),
    - actualiza `r15`,
    - salta a la dirección de la función destino.

En otras palabras: **la función actual "se transforma" en la siguiente**.

### Formas posibles
Existe dos formas de ejecutar este tipo de optimizacion
#### Opción 1: instrucción explícita `tailcall`

una instrucción de VM:
```c
tailcall fn_label, arg_count
```
Semántica:
1. Prepara `r01–r05` y args extra en stack como siempre.
2. `r15 = arg_count`.
3. No toca `[rbp]` ni `[rbp+8]`.
4. `rip = fn_address` (sin empujar return address, sin `enter` nuevo).

La función destino se ejecuta como si hubiera sido llamada desde el caller original.

#### Opción 2: optimización de patrón `call + ret`
En el assembler / compilador de alto nivel detecta:
```c
// preparar args
callvm fn
ret
```
y lo transforma en:
```c
// preparar args
tailcall fn
// (no hay leave, no hay ret)
```
### Interacción con enter / leave
las funciones TCO-heavy no usan enter/leave:
```c
// cuerpo
prep_args
tailcallvm fn
```
VM hace:
```c
r01..r05 = nuevos args
stack args = nuevos args extra
r15 = count
rip = fn_address
```
Al no usarse un [[ENTER]] en este tipo de funciones, no es necesario usar [[LEAVE]]

```c
fact(n, acc):
    if n == 0:
        return acc
    return fact(n-1, acc*n)
```
En bytecode, TCO:
```c
// fact(n, acc)
// r01 = n
// r02 = acc

fact:
	cmp r01, 0
	je base_case

	// preparar args para fact(n-1, acc*n)
	sub r01, 1
	mul r02, r01

	tailcall fact   // <-- sin enter, sin leave, sin ret
	                // la función se recicla a sí misma
	base_case:
		mov r00, r02
		ret
```