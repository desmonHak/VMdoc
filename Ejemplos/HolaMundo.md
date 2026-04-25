# Hello World en VestaVM

Este tutorial muestra como imprimir texto en VestaVM sin hardcodear el string como bytes en los registros. El string se define con `db` en la seccion de datos (VM memory) y se copia al host con `vmcopy` para pasarlo a una funcion nativa.

---

## El problema: dos espacios de memoria

VestaVM tiene dos espacios de memoria completamente separados:

| Espacio        | Quien lo gestiona   | Acceso desde el bytecode  |
| :------------- | :------------------ | :------------------------ |
| **VM memory**  | TLB + ArenaManager  | `mov`, `adds`, `db`, `dq` |
| **Host memory**| OS / `alloc`        | cursores, `alloc`, `free` |

Las funciones nativas como `puts`, `WriteConsoleA` o `printf` reciben **punteros host** - no entienden las direcciones virtuales de la VM. Por eso, cualquier string que queramos pasar a una funcion nativa debe estar primero en host memory.

---

## Opcion A - string hardcodeado (patron antiguo)

```vel
// "Hello\0\0\0" en little-endian como qword: 0x0000006F6C6C6548
mov    r1, 8
alloc  r1
mov    r10, r0
xchg   cur0, r0
mov    r5, 0x0000006F6C6C6548   // "Hello\0\0\0"
writecur cur0, r5

mov    r15, 1
mov    r1, r10
calln  @Method("msvcrt.dll:puts")
free   r10
hlt
```

**Desventajas:**
- El string debe calcularse a mano en little-endian.
- Strings largos requieren multiples qwords, desplazamientos manuales y el patron `xchg`/`adds`/`xchg` para avanzar el cursor.
- Dificil de leer y mantener.

---

## Opcion B - `db` + `vmcopy` (patron recomendado)

```vel
@Format("raw")
@SpaceAddress {
    @Name("anonymous"),
    @IniAddress(0x0000000000000000),
    @EndAddress(0xFFFFFFFFFFFFFFFF)
}
@Section {
    @Name("all"),
    @SpaceAddress("anonymous")
    @Align(0x1000)
}

@Import {
    @Method {
        @Lib("msvcrt.dll")   // puts de la CRT de Windows
        @Name("puts")
    }
}

code:
    // 1. Obtener la direccion VM del string.
    //    @Absolute resuelve la etiqueta en tiempo de enlace (relocation).
    mov    r1, @Absolute("all.hello")     // r1 = VM address de "Hola, Mundo!"

    // 2. Reservar buffer host para la copia.
    mov    r2, 16
    alloc  r2                             // r0 = puntero host
    mov    r10, r0                        // guardar para free + calln
    xchg   cur0, r0                       // cur0 = host ptr  |  r0 = 0

    // 3. Copiar string de VM memory al buffer host en una instruccion.
    //    vmcopy avanza cur0 y r1 automaticamente en 13 bytes.
    mov    r2, 13                         // "Hola, Mundo!\0" = 13 bytes
    vmcopy cur0, r1, r2

    // 4. Llamar puts(host_ptr).
    //    puts imprime hasta '\0' y anade un newline automatico.
    mov    r15, 1
    mov    r1, r10
    calln  @Method("msvcrt.dll:puts")

    // 5. Liberar y terminar.
    free   r10
    hlt
end_code:

align 16
data:
    // 12 bytes de string + 4 bytes null padding = 16 bytes alineados
    hello db "Hola, Mundo!", 0x00, 0x00, 0x00, 0x00
end_data:
```

**Salida:**
```
Hola, Mundo!
```

---

## Como compilar y ejecutar

```bash
# Compilar .vel -> .velb
vesta --build hola_mundo.vel -o hola_mundo

# Ejecutar
vesta --run hola_mundo.velb
```

El archivo compilado esta en `examples_codes_vm/hola_mundo.vel`.

---

## Paso a paso explicado

### `@Absolute("all.hello")`

Produce una **relocation** de tipo `Absolute64` que el linker resuelve a la direccion virtual de la etiqueta `hello` dentro de la seccion `all`. Es el equivalente a `&hello` en C, pero para el espacio de direcciones de la VM.

### `alloc r2`

Reserva `r2` bytes de **host memory** (con `malloc` o equivalente del OS) y devuelve el puntero en `r0`. Este puntero es valido para pasarlo a funciones nativas.

### `xchg cur0, r0`

Intercambia el valor de `cur0` (que estaba en 0) con `r0` (que tiene el host ptr). Resultado: `cur0 = host ptr`, `r0 = 0`. El 0 resultante en `r0` es util como registro indice cero para instrucciones SIB si se necesita.

### `vmcopy cur0, r1, r2`

La instruccion clave. Copia `r2` bytes desde VM memory en la direccion `r1` al buffer host en `cur0`. Internamente:

1. Traduce la direccion virtual `r1` a traves del TLB.
2. Copia los bytes al destino host usando **la ruta SIMD mas rapida disponible** (AVX-512 -> AVX2 -> SSE2 -> memcpy), detectada una sola vez al inicio.
3. Avanza `cur0 += r2` y `r1 += r2` para facilitar copias secuenciales.

### `calln @Method("msvcrt.dll:puts")`

Llama a `puts` de la CRT de Windows. `r15 = 1` indica 1 argumento; `r1` contiene el puntero host al string. La funcion recorre la memoria hasta el `\0` e imprime la cadena seguida de un newline.

---

## Linux / macOS

Solo cambia la biblioteca:

```vel
@Import {
    @Method {
        @Lib("libc.so.6")   // Linux
        @Name("puts")
    }
}
// o en macOS:
//   @Lib("libSystem.dylib")
```

El resto del codigo es identico.

---

## Variante: copias secuenciales con `addcur`

Si necesitas construir el buffer en varias pasadas puedes combinar `vmcopy` con `addcur`:

```vel
mov    r1, @Absolute("all.parte1")
mov    r2, 6
vmcopy cur0, r1, r2        // copia "Hola, " (6 bytes); cur0 avanza 6

// cur0 ahora apunta al offset 6 del buffer
// r1 ahora apunta 6 bytes mas alla de parte1 (ya no util aqui)

mov    r1, @Absolute("all.parte2")
mov    r2, 7
vmcopy cur0, r1, r2        // copia "Mundo!\0" (7 bytes); cur0 avanza 7

// retroceder el cursor al inicio para pasar el puntero a puts
addcur cur0, 0xFFF1        // -13 en int16 (0x10000 - 13 = 0xFFF3... ajustar)
// alternativa: guardar r10 = host ptr y usar r10 directamente
```

> En la practica, para llamadas nativas es mas sencillo guardar el host ptr original en un registro (`mov r10, r0`) antes del `xchg` y usarlo directamente en `calln` sin retroceder el cursor.

---

## Ver tambien

- [[cursor|Instrucciones cursor]] - referencia completa de `readcur`, `writecur`, `gcderef`, `addcur`, `vmcopy`, `vcopyh`
- [[NativeCall (CallN)]] - convencion de llamada nativa, argumentos y valores de retorno
- [[Allocator crudo para FFI y memoria manual|ALLOC / FREE]] - gestion de host memory
- [[MOV, MOVH, MOVC, MOCH]] - instrucciones MOV y variantes de memoria
