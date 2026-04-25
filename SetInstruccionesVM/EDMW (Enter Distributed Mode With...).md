# EDMW - Enter Distributed Mode With (Entrar en modo distribuido)

VestaVM tiene la capacidad de ejecutar bloques de codigo en maquinas remotas conectadas
por red. `EDMW` activa el **modo distribuido**: a partir de este punto, las instrucciones
ya no se ejecutan localmente sino que se acumulan para ser enviadas al host remoto cuando
se ejecute `EDM`.

Analogia: es como dictar una lista de tareas a un asistente remoto. Le das la direccion
del asistente (IP:puerto), le dictas las instrucciones, y cuando terminas de dictar
(`EDM`), el asistente ejecuta todo lo que le pediste.

| Instruccion | opcode0 | opcode1 | Tamano   | Descripcion                                               |
| :---------: | :-----: | :-----: | :------: | :-------------------------------------------------------- |
| `edmw` IPv4 |  0x00   |  0x00   | 8 bytes  | Entrar en modo distribuido con host IPv4:puerto           |
| `edmw` IPv6 |  0x00   |  0x01   | 20 bytes | Entrar en modo distribuido con host IPv6:puerto           |

---

## Codificacion IPv4 (8 bytes)

```
+------+------+----+----+----+----+------+------+
| 0x00 | 0x00 | ip0| ip1| ip2| ip3| p_hi | p_lo |
+------+------+----+----+----+----+------+------+
  byte0  byte1  bytes 2-5: IPv4   bytes 6-7: puerto

Ejemplo: 192.168.0.1:80
00 00  C0 A8 00 01  00 50
```

## Codificacion IPv6 (20 bytes)

```
+------+------+----------16 bytes IPv6-----------+------+------+
| 0x00 | 0x01 |  b0  b1  b2 ... b15              | p_hi | p_lo |
+------+------+-----------------------------------+------+------+
  byte0  byte1  bytes 2-17: IPv6 raw    bytes 18-19: puerto

Ejemplo: 2001:0db8:85a3::8a2e:0370:7334, puerto 80
00 01  20 01 0D B8 85 A3 00 00 00 00 8A 2E 03 70 73 34  00 50
```

---

## Uso tipico

```c
// Ejecutar un calculo en el host remoto 192.168.1.10, puerto 9000

// Paso 1: entrar en modo distribuido (IPv4)
// En .vel, la sintaxis de alto nivel es:
edmw 192.168.1.10, 9000    // activar modo distribuido

// Paso 2: las instrucciones siguientes se acumulan para el remoto
mov   r1, 100
mov   r2, 200
addu  r0, r1, r2           // calcular 100 + 200 en el remoto

// Paso 3: enviar el bloque y salir del modo distribuido
edm                        // el bloque se ejecuta en 192.168.1.10:9000
```

---

## Flag DM

Al entrar en modo distribuido, el flag `DM` del registro de flags se pone a 1. Al salir
con `EDM`, el flag `DM` vuelve a 0. Puedes consultar este flag para detectar el modo:

```c
// Leer el flag DM
movc  r5, r6, DM    // si DM=1 (modo distribuido activo), r5 = r6
```

---

## Notas

- El permiso `permissions.EDM_EDMW` debe estar activo en la instancia (ver [[VMINFO]]).
- `EDMW` establece la conexion de red al remoto. Si el remoto no esta disponible o el
  puerto esta cerrado, la VM genera un error de conexion.
- Se puede saber si se esta en modo distribuido consultando el flag `DM` del registro
  de flags (ver [[REGISTROS]]).
- `EDM` debe ejecutarse para cerrar el bloque. Si el proceso termina sin llamar a `EDM`,
  el bloque acumulado se descarta.

---

Ver tambien: [[EDM (Exit distributed mode)]], [[VMINFO]], [[REGISTROS]]
