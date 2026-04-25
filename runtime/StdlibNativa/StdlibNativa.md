# Libreria estandar nativa de VestaVM

Los modulos nativos de la stdlib estan en `stdlib/native/` y se compilan
como librerias dinamicas (`.dll` / `.so`).  Se importan en bytecode .vel
con la ruta relativa sin extension:

```c
@Import {
    @Method {
        @Lib("stdlib/native/io/vesta_io")
        @Name("vio_println")
    }
}
```

El loader resuelve la ruta relativa al directorio del ejecutable `vesta`.

---

## Modulos disponibles

### vesta_io - Entrada/Salida

Ruta: `stdlib/native/io/vesta_io`  
Fuente: `stdlib/native/io/vesta_io.c`

Todas las funciones de cadena o buffer leen/escriben en la **memoria virtual
de la VM** usando el patron `(proc_ptr, vm_addr, len)`.
El `proc_ptr` se obtiene con la instruccion `getproc`.

| funcion             | argumentos (r1..rN)                                    | retorno          |
| :------------------ | :----------------------------------------------------- | :--------------- |
| `vio_print`         | proc_ptr, vm_addr, len                                 | 0                |
| `vio_println`       | proc_ptr, vm_addr, len                                 | 0                |
| `vio_print_int`     | valor (int64_t como uint64_t)                          | 0                |
| `vio_print_uint`    | valor (uint64_t)                                       | 0                |
| `vio_print_hex`     | valor (uint64_t)                                       | 0                |
| `vio_print_float`   | bits IEEE 754 del double                               | 0                |
| `vio_fopen`         | proc_ptr, path_vm_addr, path_len, mode_vm_addr, mode_len | handle (uint64_t) |
| `vio_fclose`        | handle                                                 | 0 ok / -1 error  |
| `vio_fread`         | proc_ptr, vm_addr, size, handle                        | bytes leidos     |
| `vio_fwrite`        | proc_ptr, vm_addr, size, handle                        | bytes escritos   |
| `vio_fflush`        | handle (0 = stdout)                                    | 0 ok / -1 error  |

**Macro de depuracion:** `VESTA_IO_DEBUG` (default 0). Compilar con
`-DVESTA_IO_DEBUG=1` para que `vesta_init` emita `[vesta_io] cargado`.

#### Ejemplo: imprimir un string de la VM

```c
@Import {
    @Method { @Lib("stdlib/native/io/vesta_io") @Name("vio_println") }
}

code:
    getproc r1
    mov     r2, @Absolute("all.msg")
    mov     r3, 12
    mov     r15, 3
    calln   @Method("stdlib/native/io/vesta_io:vio_println")
    hlt
end_code:

data:
    msg db "Hola, Mundo!", 0x00
end_data:
```

---

### vesta_math - Matematicas

Ruta: `stdlib/native/math/vesta_math`  
Fuente: `stdlib/native/math/vesta_math.c`

Todas las funciones operan sobre valores numericos (`uint64_t` enteros o
bits IEEE 754 para doubles). No hay argumentos puntero a memoria VM.

| funcion        | argumentos         | retorno                  | descripcion                     |
| :------------- | :----------------- | :----------------------- | :------------------------------ |
| `vmath_sqrt`   | bits (f64)         | bits f64                 | raiz cuadrada                   |
| `vmath_pow`    | base_bits, exp_bits | bits f64                | base^exp                        |
| `vmath_abs`    | n (int64 como u64) | uint64_t                 | valor absoluto entero           |
| `vmath_fabs`   | bits (f64)         | bits f64                 | valor absoluto double           |
| `vmath_floor`  | bits (f64)         | bits f64                 | redondeo hacia -inf             |
| `vmath_ceil`   | bits (f64)         | bits f64                 | redondeo hacia +inf             |
| `vmath_round`  | bits (f64)         | bits f64                 | redondeo al mas cercano         |
| `vmath_min`    | a, b (uint64)      | uint64_t                 | minimo entero sin signo         |
| `vmath_max`    | a, b (uint64)      | uint64_t                 | maximo entero sin signo         |
| `vmath_clamp`  | x, lo, hi (uint64) | uint64_t                 | acotacion entera sin signo      |
| `vmath_fmin`   | a_bits, b_bits     | bits f64                 | minimo double                   |
| `vmath_fmax`   | a_bits, b_bits     | bits f64                 | maximo double                   |
| `vmath_log`    | bits (f64)         | bits f64                 | logaritmo natural               |
| `vmath_log2`   | bits (f64)         | bits f64                 | logaritmo base 2                |
| `vmath_log10`  | bits (f64)         | bits f64                 | logaritmo base 10               |
| `vmath_sin`    | bits (f64)         | bits f64                 | seno (radianes)                 |
| `vmath_cos`    | bits (f64)         | bits f64                 | coseno (radianes)               |
| `vmath_tan`    | bits (f64)         | bits f64                 | tangente (radianes)             |

**Bits IEEE 754 frecuentes:**

| valor double | bits (uint64_t hex) |
| :----------: | :-----------------: |
| 0.0          | 0x0000000000000000  |
| 1.0          | 0x3FF0000000000000  |
| 1.5          | 0x3FF8000000000000  |
| 2.0          | 0x4000000000000000  |
| 4.0          | 0x4010000000000000  |
| -5.0         | 0xC014000000000000  |

**Macro de depuracion:** `VESTA_MATH_DEBUG` (default 0). Compilar con
`-DVESTA_MATH_DEBUG=1` para que `vesta_init` emita `[vesta_math] cargado`.

#### Ejemplo: calcular raiz cuadrada de 4.0

```c
@Import {
    @Method { @Lib("stdlib/native/math/vesta_math") @Name("vmath_sqrt") }
}

code:
    mov     r1, 0x4010000000000000   // bits IEEE 754 de 4.0
    mov     r15, 1
    calln   @Method("stdlib/native/math/vesta_math:vmath_sqrt")
    // r0 = 0x4000000000000000 (bits de 2.0)
    hlt
end_code:
```

---

## Compilacion de los modulos

Los modulos se compilan automaticamente con CMake cuando la opcion
`VESTA_BUILD_STDLIB` esta activada (ON por defecto):

```bash
cmake -DVESTA_BUILD_STDLIB=ON ..
cmake --build .
```

Las DLLs/.so generadas deben estar en el mismo directorio que el ejecutable
`vesta` o en el PATH del sistema para que el loader las encuentre.

---

Vease [[NativePluginAPI]] para como escribir nuevos plugins nativos.
Vease [[GETPROC_GETVM_GETMGR]] para obtener el contexto del proceso desde bytecode.
