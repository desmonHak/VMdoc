La VM usa su propia convención de llamadas para conocer cuántos parámetros se han usado al hacer la llamada a una función interna (código que está dentro de la VM) o una función externa (función real que no es parte de la VM).

## Convencion para acceso a memoria VM desde funciones nativas

Cuando una funcion nativa necesita leer o escribir datos en la **memoria
virtual de la VM** (no en memoria del proceso host), se usa el patron
`(proc_ptr, vm_addr, len)`:

| registro | contenido                                       |
| :------: | :---------------------------------------------- |
| `r1`     | `proc_ptr` — resultado de la instruccion `getproc` |
| `r2`     | `vm_addr`  — direccion virtual dentro de la VM  |
| `r3`     | `len`      — numero de bytes a operar           |

```asm
getproc r1                              ; r1 = ProcessVM* del proceso actual
mov     r2, @Absolute("all.buffer")    ; r2 = direccion virtual del dato en la VM
mov     r3, 64                         ; r3 = longitud en bytes
mov     r15, 3
calln   @Method("mi_lib:mi_funcion")
```

En la funcion nativa (C) se usa `g_api->vm_read_bytes` / `vm_write_bytes`
para traducir la direccion VM a memoria host:

```c
uint64_t mi_funcion(uint64_t proc_ptr, uint64_t vm_addr, uint64_t len) {
    char buf[256];
    g_api->vm_read_bytes(proc_ptr, vm_addr, buf, len);
    /* ... procesar buf ... */
    return 0;
}
```

**No existe ningun argumento implicito** (sin `void *ctx`). El `proc_ptr`
se pasa explicitamente como primer argumento cuando se necesita.

Vease [[GETPROC_GETVM_GETMGR]] y [[NativePluginAPI]].

---

## Convención para llamadas nativas / externas a la VM

Registros:
- `r00`: valor de retorno de la llamada si retorna.
- `r15`: cantidad de parámetros de la función. Máximo de `r01` a `r12` = 12 argumentos.
- `r01-r12`: los registros se usan para pasar los parámetros.

```asm
mov r15, 1                          ; 1 argumento
mov r01, hola_mundo                 ; puntero al string
calln @Method("libc.dll:puts")
; el valor retornado está en r00
hlt

hola_mundo db "Hola mundo", 0x0
```
Véase [[NativeCall (CallN)]].
> Esta convención define un máximo de 12 parámetros para funciones nativas.

---

## Convención de llamadas para funciones internas

- `r00`: valor de retorno.
- `r15`: cantidad de parámetros pasados.
- `r01-r05`: los primeros 5 parámetros van en registros.
- Argumentos adicionales (6 en adelante): se empujan en el stack en **orden natural** (izquierda -> derecha, a5 primero, a6 después).

Ejemplo para `foo(a0, a1, a2, a3, a4, a5, a6)`:
- `r01 = a0`, `r02 = a1`, `r03 = a2`, `r04 = a3`, `r05 = a4`
- `a5` y `a6` se pasan por el stack.

### Secuencia de llamada completa

1. **Cargar registros con los primeros 5 argumentos:**
```asm
mov r01, a0
mov r02, a1
mov r03, a2
mov r04, a3
mov r05, a4
mov r15, 7      ; total de argumentos (registros + stack)
```

2. **Empujar argumentos extra en el stack (orden natural):**
```asm
push a5         ; se empuja primero (quedará más profundo en la pila)
push a6         ; se empuja segundo (quedará encima de a5)
```

Estado del stack tras los pushes (SP crece hacia abajo):
```
SP     -> a6     (top, último empujado)
SP+8   -> a5
SP+16  -> ...    (resto del stack del caller)
```

3. **Llamar a la función** - [[CALLVM]] empuja automáticamente la dirección de retorno y salta:
```asm
callvm foo
```

4. **Al entrar en `foo`, el callee ejecuta [[ENTER]]:**
```asm
enter 32        ; reserva 32 bytes para variables locales
```
equivale a:
```asm
push rbp        ; guarda el frame anterior
mov  rbp, rsp   ; establece el nuevo frame base
sub  rsp, 32    ; reserva espacio local
```

**Layout del stack frame tras `enter 32`:**
```
[rbp+24] = a6               <- argumento extra (segundo empujado)
[rbp+16] = a5               <- argumento extra (primero empujado)
[rbp+8]  = return_address   <- empujado por CALLVM
[rbp+0]  = saved rbp        <- empujado por ENTER (frame anterior)
[rbp-8]  = local0           ─┐
[rbp-16] = local1            │ variables locales
...                          │
[rbp-32] = local3           ─┘
```

### El callee NO limpia la pila

El **caller** limpia los argumentos extra del stack después del retorno (caller-cleans convention, igual que System V / ARM64 / RISC-V):
```asm
; tras retornar de foo:
pop r06    ; quitar a6 del stack
pop r07    ; quitar a5 del stack
; o bien: add rsp, 16
```

---

## Stack Trace

Esta convención permite construir un stack trace para depuración:

1. Leer el `rbp` actual.
2. Leer `[rbp+0]` -> **saved rbp** del frame anterior.
3. Leer `[rbp+8]` -> **dirección de retorno** de este frame.
4. Repetir con el rbp anterior hasta llegar a un frame nulo.

```c
frame0_rbp = vm->registers.rbp;
frame0_ret = mem[frame0_rbp + 8];   // dirección de retorno
frame1_rbp = mem[frame0_rbp + 0];   // rbp del frame anterior
frame1_ret = mem[frame1_rbp + 8];
frame2_rbp = mem[frame1_rbp + 0];
// ...
```

Esto da la cadena de llamadas:
```
fnA -> fnB -> fnC -> fnD
```

> Para que esto funcione, todas las funciones deben usar stack frame (`enter`/`leave`) y se debe mantener un mapa de símbolos `nombre -> dirección`.
```
at foo()  [0x1234]
at bar()  [0x5678]
at main() [0x9ABC]
```

---

## TCO - Tail Call Optimization

### ¿Cuándo se puede aplicar TCO?

Una llamada está en _posición de cola_ cuando es **lo último** que hace la función y su resultado se retorna directamente:

```asm
; cuerpo...
; preparar args para foo
callvm foo
ret          ; <- candidato a TCO
```

Si hay cualquier operación después de la llamada (sumar, comparar, etc.), **no hay TCO**.

### Idea clave: reutilizar el frame actual

Una llamada normal hace:
1. Empujar args extra en stack.
2. Empujar return address (`callvm`).
3. Saltar a la función destino.
4. En callee: `enter` crea nuevo frame.
5. Al final: `leave` + `ret` destruye frame y retorna.
6. Caller limpia args extra.

Con TCO se **elimina el nuevo frame**:
- No se empuja nueva return address (ya existe la del caller en `[rbp+8]`).
- No se crea nuevo frame (`enter`).
- Se reutilizan `rbp`/`rsp` del frame actual.
- Se salta directamente a la función destino.

Es decir, se convierte:
```asm
callvm foo
ret
```
en:
```asm
; preparar nuevos argumentos en r01-r05 y stack
jmp foo     ; salto directo sin crear nuevo frame
```

### Opción 1: instrucción explícita `tailcall`

```asm
tailcall fn_label, arg_count
```

Semántica:
1. Prepara `r01-r05` y args extra en stack.
2. `r15 = arg_count`.
3. No toca `[rbp+0]` (saved rbp) ni `[rbp+8]` (return address).
4. `RIP = fn_address` (sin empujar return address, sin `enter` nuevo).

La función destino se ejecuta como si hubiera sido llamada desde el caller original.

### Opción 2: detección de patrón `callvm + ret`

El ensamblador o compilador de alto nivel detecta el patrón y lo transforma:
```asm
; antes:
callvm fn
ret

; después (TCO aplicado):
tailcall fn
; (sin leave, sin ret)
```

### Interacción con `enter`/`leave`

Las funciones con TCO puro no necesitan `enter`/`leave`:
```asm
fact:               ; fact(n=r01, acc=r02)
    cmp r01, 0
    jmp.je base_case

    ; preparar fact(n-1, acc*n)
    sub r01, 1
    mul r02, r01
    mov r15, 2
    tailcall fact   ; <- sin enter, sin leave, sin ret

base_case:
    mov r00, r02
    ret
```

> Al no usarse `enter`, no es necesario usar `leave` antes del `tailcall`.
