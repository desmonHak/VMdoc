
Permite indicar que se quiere entrar en el modo distribuido contra un Host.


| opcode1 | opcode2 |    IPv4    |  port  |
| :-----: | :-----: | :--------: | :----: |
|   0x0   |   0x0   | 0x12345678 | 0x0000 |
```
00 00  | C0 A8 00 01| 00  50
opcode | 192.168.0.1|   80

8bytes
```

| opcode1 | opcode2 |                  IPv6                   |  port  |
| :-----: | :-----: | :-------------------------------------: | :----: |
|   0x0   |   0x1   | 2001:0db8:85a3:0000:0000:8a2e:0370:7334 | 0x0000 |
```
00 01 | 20 01 0D B8 85 A3 00 00 00 00 8A 2E 03 70 73 34 | 00 50
opcode|    2001:0db8:85a3:0000:0000:8a2e:0370:7334      |   80

20bytes
```

Es necesario salirse de este modo para seguir ejecutando código en la maquina Host usando la instrucción [[EDM (Exit distributed mode)]].

Al salirse del modo, las instrucciones descritas desde el momento en el que se entro en este modo anterior, serán enviadas al host descrito, con metadatos como el ID de la instancia actual de la VM, la dirección IP y el puerto de la instancia y otro conjunto de datos.