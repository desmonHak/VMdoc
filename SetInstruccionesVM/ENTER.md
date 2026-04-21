Crea un stack frame para la función actual. Se usa al inicio de cada función que necesite variables locales o que siga la [[ConvenciónDeLlamadas]]. Se deshace con [[LEAVE]].

| instrucción      | opcode1 | byte2 (reservado) | frame_size (64 bits) | tamaño   |
| :--------------: | :-----: | :---------------: | :------------------: | :------: |
| `enter frame_sz` |  0x28   |       0x00        |  tamaño en bytes     | 10 bytes |

**Comportamiento equivalente:**
```asm
push rbp          ; guarda el base pointer del frame anterior
mov  rbp, rsp     ; establece el base pointer del nuevo frame
sub  rsp, frame_sz ; reserva espacio para variables locales
```

**Layout del stack tras `enter N`** (asumiendo que [[CALLVM]] ya empujó la dirección de retorno):

```
[rbp + 8]  = return_address   <- empujado por CALLVM
[rbp + 0]  = saved rbp        <- empujado por ENTER (frame anterior)
[rbp - 8]  = local_0          <- primer local (dentro del frame)
[rbp - 16] = local_1
...
[rbp - N]  = último local
```

> Para funciones sin variables locales se puede usar `enter 0`, que sólo guarda `rbp` y establece el frame.

**Ejemplo:**
```asm
my_fn:
    enter 32        ; reserva 32 bytes de espacio local
    ; acceder a locals: [rbp-8], [rbp-16], [rbp-24], [rbp-32]
    ; ...
    leave
    ret
```
