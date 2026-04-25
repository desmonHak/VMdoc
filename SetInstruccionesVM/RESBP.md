# RESBP - Reservar o mapear una pagina de memoria

`RESBP` (Reserve/Map Block Page) permite a la VM pedir al runtime que reserve una nueva
pagina de memoria virtual (4096 bytes) o que mapee una direccion real del host a una
direccion virtual de la VM.

En sistemas operativos modernos, la memoria se gestiona en **paginas** de 4096 bytes.
`RESBP` es la instruccion de bajo nivel para solicitar esas paginas, equivalente a
`VirtualAlloc` en Windows o `mmap` en Linux, pero dentro del sistema de memoria virtual
de VestaVM.

| Instruccion | opcode0 | opcode1 | Tamano  | Descripcion                                     |
| :---------: | :-----: | :-----: | :-----: | :---------------------------------------------- |
| `resbp`     |  0x00   |  0x20   | 2 bytes | Reservar o mapear una pagina segun flags en R1  |

---

## Registros de entrada

| Registro | Rol                                                                   |
| :------: | :-------------------------------------------------------------------- |
| R0       | Direccion virtual deseada (la pagina comenzara aqui, alineada 4096B) |
| R1       | Flags que indican el tipo de operacion (ver tabla)                   |
| R2       | Direccion real del host a mapear (solo si bit 6 de R1 = 1)           |

---

## Tabla de flags (R1)

| Bit | Nombre                | Descripcion                                                  |
| :-: | :-------------------- | :----------------------------------------------------------- |
| 0   | reserva local instancia | Reservar pagina en la instancia actual (privada)           |
| 1   | reserva local manager   | Reservar pagina en el manager (compartida entre instancias)|
| 2   | reserva remota instancia| Reservar pagina en una instancia remota                    |
| 3   | reserva remota manager  | Reservar pagina en un manager remoto                       |
| 4   | solicitar pagina nueva  | 1 = pedir memoria nueva del SO                             |
| 5   | mapear dir virtual      | 1 = mapear una dir virtual existente del manager           |
| 6   | mapear dir host         | 1 = mapear una dir real del host (R2 = dir real)           |
| 7   | reservado               | Debe ser 0                                                 |

---

## Ejemplos

### Reservar memoria privada a nivel de instancia

```c
mov   r0, 0x1000          // la pagina empieza en 0x1000 (cubre 0x1000-0x1FFF)
mov   r1, 0b00010000      // bit 4 = solicitar pagina nueva, bit 0 = instancia local
resbp                     // reservar la pagina

// Ahora se puede usar el rango 0x1000-0x1FFF
mov   r5, 42
mov   [r0], r5            // escribir en la pagina recien reservada
```

### Mapear una direccion host a la VM

```c
mov   r0, 0x1000                    // dir virtual de destino en la VM
mov   r1, 0b01000000                // bit 6 = mapear dir host
mov   r2, 0xFFFFFFFFFFFF1234        // dir real del host a mapear
resbp                               // mapear

// Ahora acceder a 0x1000 en la VM lee/escribe en 0xFFFFFFFFFFFF1234 del host
```

### Reservar memoria compartida en el manager

```c
// Paso 1: reservar en el manager (visible para todas las instancias locales)
mov   r0, 0x2000          // dir virtual en el manager
mov   r1, 0b00010010      // bit 4 = nueva pagina, bit 1 = manager local
resbp

// Paso 2: mapear la dir del manager en esta instancia
mov   r1, 0b00100001      // bit 5 = mapear dir virtual del manager, bit 0 = instancia
mov   r2, 0x2000          // dir virtual del manager a mapear
resbp

// Ahora esta instancia puede acceder a la memoria compartida del manager via 0x2000
```

---

## Notas

- `RESBP` requiere el permiso correspondiente en `permissions` (ver [[VMINFO]]).
- Las paginas de la VM tienen 4096 bytes (0x1000). La direccion en R0 se alinea
  automaticamente a multiplos de 4096.
- Para liberar una pagina reservada, usar [[FREEP]].
- Para consultar el estado de una pagina, usar [[PAGEINFO]].
- El mapeo de direcciones host (`bit 6`) da acceso directo a RAM del proceso host.
  Usar con extrema precaucion: un acceso invalido tumba la VM.

---

Ver tambien: [[FREEP]], [[PAGEINFO]], [[VMINFO]]
