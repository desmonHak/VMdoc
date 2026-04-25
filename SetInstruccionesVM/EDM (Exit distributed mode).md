# EDM - Exit Distributed Mode (Salir del modo distribuido)

El **modo distribuido** de VestaVM permite que un bloque de instrucciones se ejecute
en una maquina remota en lugar de en la maquina local. Se entra en ese modo con `EDMW`
y se sale con `EDM`.

Cuando se ejecuta `EDM`, todas las instrucciones acumuladas desde que se entro en el
modo distribuido (con `EDMW`) se **envian por red** al host remoto para su ejecucion.

| Instruccion | opcode0 | opcode1 | Tamano  | Descripcion                                           |
| :---------: | :-----: | :-----: | :-----: | :---------------------------------------------------- |
| `edm`       |  0x00   |  0x02   | 2 bytes | Salir del modo distribuido y enviar bloque al remoto  |

---

## Flujo de uso

```c
// 1. Entrar en modo distribuido apuntando a un host remoto
edmw 192.168.0.1, 8080     // conectar a 192.168.0.1:8080 y entrar en modo distribuido

// 2. Las instrucciones siguientes se acumulan (no se ejecutan localmente)
mov   r1, 42
addu  r1, 10
// ... mas instrucciones a ejecutar en el remoto ...

// 3. Salir del modo distribuido: enviar el bloque al remoto
edm

// El bloque (mov r1, 42; addu r1, 10; ...) se ejecuta en 192.168.0.1:8080
// La ejecucion local continua aqui despues del envio
```

---

## Que se envia al remoto

Al ejecutar `EDM`, la VM empaqueta y envia:

- Las instrucciones bytecode acumuladas desde el `EDMW` anterior.
- Metadatos: ID de la instancia local, IP y puerto de la instancia local, version de
  la VM, y otros datos de contexto.

---

## Flag DM

Mientras la VM esta en modo distribuido, el flag `DM` del registro de flags esta a 1.
Puedes consultarlo para saber si estas en modo distribuido:

```c
// Comprobar si estamos en modo distribuido
movc  r1, r2, DM    // si DM=1 (en modo distribuido), r1 = r2
```

---

## Notas

- `EDM` sin un `EDMW` previo es un no-op (no hay bloque que enviar).
- El permiso `permissions.EDM_EDMW` debe estar activo en la instancia (ver [[VMINFO]]).
- La ejecucion en el remoto es asincrona: la maquina local no espera el resultado
  salvo que se use algun mecanismo de sincronizacion adicional.

---

Ver tambien: [[EDMW (Enter Distributed Mode With...)]], [[VMINFO]], [[VMINFOMANAGER]]
