Crea un stack frame, se puede salir de el usando [[LEAVE]]
```c
push rbp
mov rbp, rsp
sub rsp, frame_size
```