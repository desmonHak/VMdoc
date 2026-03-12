La instrucción permite reservar una nueva pagina de memoria. O mapear una dirección real del host a dirección virtual.

- `r00`: dirección virtual.
- `r01`: flags para la instrucción.

| bit 7 |               bit 6                |                    bit 5                    |                   bit 4                    |         bit 3          |          bit 2           |         bit 1         |          bit 0          |
| :---: | :--------------------------------: | :-----------------------------------------: | :----------------------------------------: | :--------------------: | :----------------------: | :-------------------: | :---------------------: |
|       | mapear una dirección del host? (0) | Se quiere mapear una dirección virtual? (1) | solicita pagina nueva de memoria? (1) <br> | reserva remota manager | reserva remota instancia | reserva local manager | reserva local instancia |


| opcode1 | opcode2 |
| :-----: | :-----: |
|   0x0   |   0xA   |

## Reservar nueva memoria a nivel de instancia en local (privado)

```c
mov r00, 0x1000 // solicitamos reserbar una pagina de memoria y 
				// asignarle la direccion virtual 0x1000, la 
				// pagina abarca desde 0x1000 a 0x1FFF.
mov r01, 0b00010000 // reserva memoria local instancia
RESBP // intentamos hacer la reserba
```
Ahora la instancia puede usar las direcciones ``0x1000 - 0x1FFF`` para leer y guardar información.
## Mapear nueva memoria a nivel de instancia en local (privado)

```c
mov r00, 0x1000 // solicitamos reserbar una pagina de memoria y 
				// asignarle la direccion virtual 0x1000, la 
				// pagina abarca desde 0x1000 a 0x1FFF.
mov r01, 0b01000000 // indicamos que sera a nivel de instancia y es un mapeo
mov r02, 0xFFFFFFFFFFFF1234 // direcion real a mapear
RESBP // intentamos hacer mapear
```
Ahora la instancia puede acceder a memoria de otros programas u direcciones no reservadas por la VM a través del rango `0x1000-0x1FFF`

## Reservar nueva memoria a nivel de manager en local (publico para las instancias locales)
```c
mov r00, 0x1000 // solicitamos reserbar una pagina de memoria y 
				// asignarle la direccion virtual 0x1000, la 
				// pagina abarca desde 0x1000 a 0x1FFF.
mov r01, 0b00010010 // reserva memoria manager
RESBP // intentamos hacer la reserba

mov r01, 0b00100001 // mapeo a nivel de instancia, una 
					// direccion del manager (direccion virtual de manager) 

// en r00 se indica la direccion virtual del manager a mapear, como ya se configuro previamente no hace falta volver a hacerlo.
mov r02, 0x1000 // direccion virtual para la instancia, no tiene por que ser la misma que la del manager
RESBP
```
Ahora todas las instancias puede usar las direcciones ``0x1000 - 0x1FFF`` del manager para leer y guardar información. 
> Deben reservar una dirección en la instancia también para. poder acceder a memoria del mapeada por el manager .
## Mapear nueva memoria a nivel de manager en local (publico para las instancias locales)

```c
mov r00, 0x1000 // solicitamos reserbar una pagina de memoria y 
				// asignarle la direccion virtual 0x1000, la 
				// pagina abarca desde 0x1000 a 0x1FFF.
mov r01, 0b01000010 // indicamos que sera a nivel de instancia y es un mapeo
mov r02, 0xFFFFFFFFFFFF1234 // direcion real a mapear
RESBP // intentamos hacer mapear

mov r01, 0b00100001 // mapeo a nivel de instancia, una 
					// direccion del manager (direccion virtual de manager) 

// en r00 se indica la direccion virtual del manager a mapear, como ya se configuro previamente no hace falta volver a hacerlo.
mov r02, 0x1000 // direccion virtual para la instancia, no tiene por que ser la misma que la del manager
RESBP
```
Ahora todas las instancias puede usar las direcciones ``0x1000 - 0x1FFF`` del manager para leer y guardar información. 
> Deben reservar una dirección en la instancia también para. poder acceder a memoria del mapeada por el manager .