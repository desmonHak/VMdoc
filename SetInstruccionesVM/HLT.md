# HLT - Detener el proceso

La instruccion **HLT** (halt = detener) finaliza la ejecucion del proceso actual de forma
limpia. Es el equivalente a llegar al final de un programa: el proceso termina, libera sus
recursos y el scheduler registra su finalizacion.

| Instruccion | opcode | Tamano  | Descripcion               |
| :---------: | :----: | :-----: | :------------------------ |
| `hlt`       | 0x01   | 1 byte  | Detener el proceso actual |

Implementacion: `src/runtime/exec_instruction.cpp`

---

## Para que sirve

Cuando un programa termina de hacer su trabajo, necesita una forma de decirle a la VM
"ya termine, puedes limpiar mis recursos". HLT es esa senal.

Analogia: es como apagar un electrodomestico cuando terminas de usarlo. El dispositivo no
explota ni se cae, simplemente se detiene limpiamente.

Sin HLT, el proceso intentaria seguir leyendo instrucciones mas alla del codigo valido,
lo que provocaria comportamiento indefinido o un error de acceso de memoria.

---

## Comportamiento exacto

Cuando el ejecutor alcanza HLT:

1. El proceso entra en estado `HALTED`.
2. El scheduler decrementa su contador de procesos vivos (`alive_count`).
3. Cuando `alive_count` llega a cero en todos los schedulers, la VM se detiene.
4. El valor de R0 en el momento del HLT es el **codigo de salida** del proceso.

```c
// Programa minimo valido en VestaVM:
mov  r0, 0      // codigo de salida = 0 (exito)
hlt             // terminar el proceso
```

```c
// Programa que termina con un codigo de error:
mov  r0, 1      // codigo de salida = 1 (error generico)
hlt
```

---

## Diferencias importantes

| Situacion      | Instruccion  | Que ocurre                                        |
| :------------- | :----------: | :------------------------------------------------ |
| Terminar       | `hlt`        | El proceso muere, alive_count baja                |
| Ceder turno    | `yield`      | El proceso sigue vivo pero cede el CPU al scheduler |
| Esperar future | `await r1`   | El proceso se bloquea hasta que un future se resuelva |
| Bloquear mutex | `monenter r1`| El proceso espera hasta poder adquirir el monitor  |

---

## Codificacion binaria

```
+--------+
| 0x01   |
+--------+
  byte0
```

HLT es una de las instrucciones mas cortas: solo ocupa 1 byte en el bytecode.

---

## Ejemplo completo

```c
// Secciones del programa
@Format("raw")
@SpaceAddress { @Name("mem"), @IniAddress(0x0), @EndAddress(0xFFFFFFFFFFFFFFFF) }
@Section { @Name("code"), @SpaceAddress("mem") @Align(0x1000) }

code:
    // Inicializar la pila
    mov rsp, 0x00FF0000
    mov rbp, 0x00FF0000

    // Hacer el trabajo del programa
    mov r1, 10          // valor de entrada
    addu r1, 32         // calcular algo
    mov r0, r1          // poner resultado en r0 (convencion de retorno)

    hlt                 // terminar; la VM recoge r0 como codigo de salida
end_code:
```

---

## HLT y el ciclo de vida del scheduler

La VM no termina con el primer HLT. Termina cuando **todos** los procesos han llegado a HLT.
Esto permite programas con multiples procesos concurrentes donde cada uno termina
independientemente.

```c
// Proceso principal:
mov  r1, @Absolute("code.funcion_hijo")
spawn r1            // crear proceso hijo; r0 = PID del hijo
// ... hacer trabajo del padre ...
hlt                 // el padre termina; el hijo sigue si aun no hlt

// Proceso hijo (arranaca en la funcion apuntada por r1):
funcion_hijo:
    // ... hacer trabajo del hijo ...
    hlt             // el hijo termina
// Solo cuando AMBOS procesos haltan, la VM se detiene completamente
```
