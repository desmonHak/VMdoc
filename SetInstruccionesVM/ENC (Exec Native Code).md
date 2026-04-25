# ENC - Exec Native Code (Ejecutar codigo nativo)

`ENC` permite ejecutar un bloque de codigo nativo (shellcode x86/x64 o de otra
arquitectura) directamente desde una instancia de VestaVM. Es la instruccion de mas bajo
nivel del sistema: salta a codigo maquina real sin pasar por el interprete de bytecode.

> **Advertencia:** `ENC` requiere el permiso `permissions.ENC` activado en la instancia.
> Sin ese permiso, la instruccion genera un error de permiso. Ver [[VMINFO]].

> **Peligro:** el codigo nativo se ejecuta con los mismos privilegios del proceso del SO
> que hospeda la VM. Un error en el shellcode puede producir un crash de la VM entera o
> abrir una vulnerabilidad de seguridad.

| Instruccion | opcode0 | opcode1 | Tamano  | Descripcion                                         |
| :---------: | :-----: | :-----: | :-----: | :-------------------------------------------------- |
| `enc`       |  0x00   |  0x0C   | 4 bytes | Ejecutar codigo nativo en la dir host apuntada por R0 |

---

## Como funciona

```c
// Paso 1: escribir el shellcode en memoria host (usando alloc + writecur)
mov   r1, 32              // tamano del shellcode en bytes
alloc r1                  // R0 = puntero host al buffer
mov   r8, r0              // guardar el puntero

mov   r14, r0
xchg  cur0, r14           // cur0 = puntero al buffer

// Escribir instrucciones nativas x86-64 (ejemplo: nop + ret):
mov   r5, 0x90C3          // 0x90 = NOP, 0xC3 = RET en x86-64
writecur cur0, r5w        // escribir 2 bytes

// Paso 2: ejecutar el shellcode
mov   r0, r8              // R0 = puntero host al shellcode
enc                       // saltar al codigo nativo en R0
                          // el control vuelve aqui cuando el nativo hace RET

// Paso 3: liberar el buffer
free  r8
```

---

## Convencion de llamada con el codigo nativo

Cuando `ENC` salta al codigo nativo, los registros de proposito general de la VM (R0-R15)
son los registros del proceso host en ese momento. El codigo nativo puede leer/modificar
esos valores. Cuando el codigo nativo ejecuta `RET` (o su equivalente en la arquitectura),
el control vuelve a la instruccion siguiente a `ENC`.

---

## Caso de uso: integracion con Keystone

`ENC` se integra con el ensamblador Keystone que tiene la VM, permitiendo generar y
ejecutar codigo nativo de forma programatica:

```c
// Usar el REPL o CLI de VestaVM para generar codigo nativo con Keystone,
// guardarlo en un buffer y ejecutarlo con ENC.
// Este patron permite JIT (Just-In-Time compilation) dentro de la VM.
```

---

## Notas de seguridad

- `ENC` es una instruccion de **uso avanzado**. Solo se necesita en casos muy especificos:
  JIT compilation, acceso a instrucciones de hardware no cubiertas por la VM, interop
  con shellcode existente.
- Para llamar funciones nativas de bibliotecas ya cargadas, usa `CALLN`/`CALLNR`/`LOADLIB`+
  `GETPROC`, que son mucho mas seguros y con mejor soporte de depuracion.
- El permiso `ENC` deberia estar desactivado en produccion salvo que sea estrictamente
  necesario.

---

Ver tambien: [[NativeCall (CallN)]], [[LOADLIB]], [[VMINFO]]
