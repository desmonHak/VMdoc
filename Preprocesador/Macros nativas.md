Estas macros corresponden a funciones externas al código compilado que puede definirse en librerías nativas y tienen alguna 
```c
// cargar la macro nativa exec_command del modulo os
%native("os") exec_command

// ejecutar el comando ls
exec_command("ls")
```