Permite acceder a direcciones reales del Host. La instrucción cursor tiene en general dos formas de operar, puede avanzar hacia delante, o puede retroceder posiciones de memoria para obtener valores. Las instrucciones cursor usan registros cursor.

----

## Registros cursores disponibles actualmente:

>Todo registro cursor es del tamaño de un puntero en la plataforma objetivo, excepto algún caso en especifico. Los registros cursores pueden contener cualquier tipo de valor ya que se considera un registro mas, pero se recomienda siempre que estos registros solo se usen con motivo de acceder a memoria del Host, usando direcciones conocidas. Por tanto se recomienda tener también direcciones validas en estos registros, o en caso de no usarse, tener el campo en 0 indicando que es "`NULL`" la dirección a acceder.

| registro(``reg_cur``) | codificacion |
| :-------------------: | :----------: |
|       ``cur0``        |    ``00``    |
|       ``cur1``        |    ``01``    |
|       ``cur2``        |    ``10``    |
|       ``cur3``        |    ``11``    |

----
## Modos de operación del cursor (`byte flag2`)

### tamaño del desplazamiento

| cursor_mode |                       codificación                        |
| :---------: | :-------------------------------------------------------: |
|      0      |      desplazamiento de cursor por tamaño de registro      |
|      1      | desplazamiento de cursor por aritmética de desplazamiento |
>Si `cursor_mode == 0`, no es necesario indicar el tamaño del avance/retroceso del cursor, pues se calculara automáticamente en base al tamaño del registro general usado. Si el tamaño del registro general es de 1 bytes, al registro cursor de le desplaza en un byte, si el registro general es de 8 bytes, el registro cursor se desplaza 8 bytes. 
>
>`cursor_mode == 1`, en caso de activar el bit se deberá indicar en un registro general(`reg_disp`) la cantidad de desplazamiento a aplicar. Se puede aplicar un desplazamiento de 0, en ese caso, el cursor nunca avanzara ni retrocederá.

### Tipos de desplazamiento del cursor.

| direction | codificación                                                                                                                     |
| :-------: | :------------------------------------------------------------------------------------------------------------------------------- |
|     0     | Desplazamiento hacia delante. Este desplazamiento suma el tamaño indicado a la dirección almacenada en el registro cursor.       |
|     1     | Desplazamiento hacia la izquierda. Este desplazamiento resta el tamaño indicado a la dirección almacenada en el registro cursor. |


----

| opcode1 |         byte flag1         |            byte flag2            |   byte flag3    |
| :-----: | :------------------------: | :------------------------------: | :-------------: |
|   0x3   | ``modo`` `reg` ``reg_cur`` | `direction` `cursor_mode`00 0000 | `reg_disp` 0000 |
>El campo byte flag1 se compone por un campo modo de 2 bits, donde se indica si el registro general se E/S de datos es de 1, 2, 4 u 8 bytes. El campo `reg` indica el [[REGISTROS#Registros generales|registro general]] que se quiere usar como E/S de los datos, este campo permite usar registros de `r0` a `r15`. El campo `reg_cur` indica que registro cursor usar, el cual puede ir de `cur0` a `cur3`.

>En caso de que `cursor_mode == 1` el `byte flag3` se tendrá en cuenta para indicar el desplazamiento a realizar, indicando el registro general a usar, este campo hace uso de 4bits para describir el registro a usar.
>En caso de que `cursor_mode == 0` el campo `reg_disp` no se tendrá en cuenta, se espera que sea 0000


**Auto‑decrementar 4 bytes (leer 32 bits) y retroceder:**
```c
// leemos el valor actual, almacenamos en r2 y retrocedemos el cursor
cursor.advanced.r cur1, r2d   // r2d = [cur1]; cur1 -= 4
```

**Auto‑incrementar 8 bytes (leer 64 bits) y avanzar:**
```c
cursor.advanced.r cur0, r0   // r0 = [cur0]; cur0 += 8
```

**desplazar cursor en base a un registro(resta):**
```c
cursor.advanced.r.addi cur0, r0, r1   // r0 = [cur0]; cur0 += r1
```

**Escritura con auto‑increment 64 bits**
escribe 8 bytes desde `r0` en la dirección contenida en `cur0`, luego suma 8 a `cur0`.
```c
// *(uint64_t*)cur0 = r0; cur0 += 8
cursor.advanced.w cur0, r0
```

**Escritura con auto‑decrement 32 bits**
escribe 4 bytes (registro de 32 bits `r2d`) en `cur1`, luego resta 4 a `cur1`.
```c
// *(uint32_t*)cur1 = r2d; cur1 -= 4
cursor.advanced.w cur1, r2d
```

**Escritura usando desplazamiento desde registro (cursor_mode = 1)**
```c
// *(uint64_t*)cur0 = r0; cur0 += r3
// r3 contiene desplazamiento en bytes (puede ser positivo o negativo)
cursor.advanced.w.addi cur0, r0, r3
```