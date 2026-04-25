# WMINT - VM Interruptions (Interrupciones virtuales)

La instruccion `vmint` (o `wmint`) permite invocar una **interrupcion virtual**
dentro de VestaVM. Las interrupciones son mecanismos para manejar eventos especiales:
errores hardware, llamadas al sistema, breakpoints de depuracion, etc.

**Analogia:** las interrupciones son como un timbre de puerta. Cuando alguien llama
(ocurre un evento), el sistema para lo que esta haciendo, atiende al visitante (ejecuta
el manejador de la interrupcion) y luego vuelve a lo que estaba haciendo.

---

## Tabla de interrupciones (IVT)

La **IVT** (Interrupt Vector Table) es una tabla de 1024 punteros a funciones manejadoras.
En 64-bit ocupa dos paginas de memoria (1024 entradas x 8 bytes = 8192 bytes).

El opcode se codifica como `0xF000 | id_interrupcion`, por lo que las interrupciones
van de `0xF000` hasta `0xF400` (1024 entradas).

| Opcode    | Instruccion      | Uso predefinido                          |
| :-------: | :--------------- | :--------------------------------------- |
| `0xF000`  | `vmint 0x0`      | Division por cero                        |
| `0xF001`  | `vmint 0x1`      | Reservado                                |
| `0xF002`  | `vmint 0x2`      | Sin asignacion                           |
| `0xF003`  | `vmint 0x3`      | Breakpoint (para depuradores)            |
| `0xF004`  | `vmint 0x4`      | Overflow aritmetico                      |
| `0xF005`  | `vmint 0x5`      | Sin asignacion                           |
| `0xF006`  | `vmint 0x6`      | Opcode invalido (UD2 en x86)             |
| `0xF007`  | `vmint 0x7`      | Stack Fault (acceso fuera de la pila)    |
| `0xF008`  | `vmint 0x8`      | General Protection Fault (permiso)       |
| `0xF009`  | `vmint 0x9`      | Page Fault (memoria no mapeada)          |
| `0xF00A`  | `vmint 0xA`      | Code Fault (salto fuera de limites)      |
| `0xF00B`  | `vmint 0xB`      | Sin asignacion                           |
| `0xF00C`  | `vmint 0xC`      | Sin asignacion                           |
| `0xF00D`  | `vmint 0xD`      | Sin asignacion                           |
| `0xF00E`  | `vmint 0xE`      | Sin asignacion                           |
| `0xF00F`  | `vmint 0xF`      | Syscall virtual (despacha VMSyscall)     |
| `0xFXXX`  | `vmint 0xXXX`    | Interrupciones definidas por el usuario  |
| `0xF400`  | `vmint 0x400`    | Ultima interrupcion disponible           |

---

## Interrupciones predefinidas

Las siguientes interrupciones de la primera IVT son gestionadas automaticamente
por la VM (no necesitan manejador del usuario):

- **Stack Fault** (`0x7`): se dispara automaticamente cuando un proceso accede fuera
  de los limites de su pila definida.

- **General Protection Fault** (`0x8`): ocurre cuando un proceso viola los permisos
  de memoria de la VM o de la memoria real del host.

- **Page Fault** (`0x9`): se invoca cuando un proceso usa una instruccion para acceder
  o modificar memoria que no esta mapeada.

- **Code Fault** (`0xA`): ocurre cuando un proceso intenta saltar o ejecutar codigo
  fuera de sus limites definidos.

---

## Uso de vmint

```c
// Invocar el breakpoint (para depuracion)
vmint 0x3        // equivale a INT 3 en x86: pausa para el depurador

// Invocar una syscall virtual (vmint 0x0F)
mov r00, 1       // numero de syscall (sys_write en Linux 64-bit)
mov r01, 1       // primer argumento (stdout)
mov r02, msg     // segundo argumento (puntero al mensaje)
mov r03, 12      // tercer argumento (longitud)
vmint 0x0F       // despachar a la syscall virtual 0x1 mediante el handler 0x0F

// Invocar una interrupcion personalizada
vmint 0x100      // tu propia interrupcion en la entrada 0x100 de la IVT
```

---

## Multiples IVTs

La VM usa un registro global `IVT` que apunta a la tabla activa. Se pueden tener
multiples IVTs (una por modo de operacion, por ejemplo) y la VM puede cambiar entre
ellas. Para las IVTs adicionales, el significado de cada entrada depende de lo que
el programador haya definido.

---

Ver tambien:
- [[VMSyscall.md]] - referencia completa de las syscalls virtuales
- [[SetInstruccionesVM/ENC (Exec Native Code).md]] - ejecucion de codigo nativo directamente
