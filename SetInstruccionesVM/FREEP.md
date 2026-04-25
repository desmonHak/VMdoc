# FREEP - Liberar una pagina de memoria reservada

`FREEP` libera una pagina de memoria previamente reservada con `RESBP`. Despues de
llamar a `FREEP`, las direcciones virtuales de esa pagina dejan de ser accesibles y
cualquier intento de leer o escribir en ellas produce un error de acceso a memoria.

| Instruccion | opcode0 | opcode1 | Tamano  | Descripcion                              |
| :---------: | :-----: | :-----: | :-----: | :--------------------------------------- |
| `freep`     |  0x00   |  0x0B   | 2 bytes | Liberar la pagina cuya dir esta en R0    |

---

## Uso

```c
// Suponer que la pagina en 0x1000 fue reservada anteriormente con RESBP
mov   r0, 0x1000    // R0 = direccion virtual de la pagina a liberar
freep               // la pagina 0x1000-0x1FFF queda liberada
                    // ya no se puede leer ni escribir en ese rango
```

---

## Notas

- `FREEP` solo puede liberar paginas reservadas por la instancia actual. No puede
  liberar paginas del manager ni de otras instancias.
- Si la direccion en R0 no corresponde a una pagina reservada, el comportamiento
  es un no-op o genera un error de permiso segun la configuracion.
- Tras `FREEP`, cualquier acceso a las direcciones liberadas produce
  `THREAD_SEGMENTATION_FAULT` (violacion de segmento).

---

Ver tambien: [[RESBP]], [[PAGEINFO]], [[VMINFO]]
