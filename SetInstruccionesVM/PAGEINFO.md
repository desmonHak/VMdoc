Permite obtener información acerca de una pagina de la VM (sea la pagina de la instancia local/remota o de un manager remoto/local).

La pagina se debe colocar en un registro, si se quiere usar la instrucción para obtener información de una pagina especifica:

```c
mov r00, 0x1234 // 0x1234 => pagina 1, disp 123
PAGEINFO r00    // obtener informacion de la pagina 1
```
>La información devuelta aparece en los registros, sobrescribiendo valores antiguos.

Consulte [[MapMemory]] y [[REGISTROS#RP Registro de mapeado de memoria (``Register Page``)|Registro RP (Registro de Pagina)]] para obtener mas información.
