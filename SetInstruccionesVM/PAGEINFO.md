# PAGEINFO - Consultar informacion de una pagina de memoria

La VM organiza su memoria en paginas de 4096 bytes. `PAGEINFO` permite consultar los
metadatos de una pagina concreta: si esta reservada, sus permisos (lectura/escritura/
ejecucion), a que instancia pertenece, etc.

| Instruccion | opcode0 | opcode1 | Modo | Tamano  | Descripcion                                       |
| :---------: | :-----: | :-----: | :--: | :-----: | :------------------------------------------------ |
| `pageinfo`  |  0x00   |  0x0A   | REG  | 2 bytes | Consultar info de la pagina apuntada por R0       |

La informacion devuelta se escribe en los registros R1-R4 (sobrescribiendo sus valores
anteriores). Los campos exactos dependen de la configuracion de la instancia.

---

## Uso

```c
mov    r0, 0x1234    // 0x1234 = pagina 1, offset 0x234
                     // La pagina es la parte alta de la direccion / 0x1000
pageinfo r0          // Consultar info de la pagina que contiene la dir 0x1234
                     // R1-R4 reciben los datos de la pagina
```

---

## Direccion de pagina

Dado que cada pagina ocupa 4096 bytes (0x1000), la pagina a la que pertenece una
direccion se calcula como `dir & ~0xFFF` (los 12 bits bajos son el offset dentro de
la pagina). `PAGEINFO` acepta cualquier direccion en el rango de la pagina.

---

## Notas

- La informacion disponible puede incluir: flags de permiso (R/W/X), tipo de pagina
  (instancia/manager/mapeada), instancia propietaria, y estado (reservada/libre).
- Ver [[RESBP]] para reservar paginas y [[FREEP]] para liberarlas.
- Para memoria compartida entre instancias, ver [[VMINFOMANAGER]].

---

Ver tambien: [[RESBP]], [[FREEP]], [[VMINFO]]
