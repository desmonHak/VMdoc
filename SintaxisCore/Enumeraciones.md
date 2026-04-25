# Enumeraciones en Vesta

Una **enumeracion** (enum) es un tipo de dato que solo puede tomar uno de un conjunto
fijo de valores con nombre. En lugar de usar numeros magicos como `0`, `1`, `2` en
tu codigo, usas nombres legibles como `ROJO`, `VERDE`, `AZUL`.

**Analogia:** piensa en los dias de la semana. Son exactamente 7 valores posibles,
tienen nombres concretos, y es mucho mas claro escribir `DIA.LUNES` que escribir `1`.

A nivel de bytecode VestaVM, los valores enum son enteros de 64 bits. El compilador
sustituye cada nombre por su valor numerico en tiempo de compilacion.

---

## Enum basico

```c
// Declaracion de un enum
enum Color {
    ROJO,    // vale 0 (el primero siempre empieza en 0 por defecto)
    VERDE,   // vale 1
    AZUL     // vale 2
}

// Uso
Color c = Color.ROJO;

if (c == Color.VERDE) {
    print("Es verde");
}
```

---

## Enum con valores explicitos

Se pueden asignar valores especificos a cada variante:

```c
enum DiaSemana {
    LUNES    = 1,  // empieza en 1
    MARTES   = 2,
    MIERCOLES = 3,
    JUEVES   = 4,
    VIERNES  = 5,
    SABADO   = 6,
    DOMINGO  = 7
}

enum Permisos {
    NINGUNO  = 0,
    LEER     = 1,
    ESCRIBIR = 2,
    EJECUTAR = 4   // potencia de 2 para poder combinar con OR
}
```

---

## Enum con datos (variante algebraica)

Vesta permite enums mas ricos donde cada variante puede llevar datos asociados,
similar a los tipos suma de Rust o Haskell:

```c
enum Resultado {
    OK(uint64_t valor),     // exito con un valor
    ERROR(String mensaje)   // fallo con un mensaje
}

// Uso
Resultado r = Resultado.OK(42);

switch (r) {
    case Resultado.OK(v): {
        print("Exito: ");
        print(v);
    }
    case Resultado.ERROR(msg): {
        print("Error: ");
        print(msg);
    }
}
```

---

## Uso en switch-case

Los enums son el caso de uso natural para `switch`. El compilador puede optimizarlos
con la instruccion `JUMPTABLE` (O(1)) cuando los valores son contiguos:

```c
enum Estado { IDLE, CORRIENDO, PAUSADO, DETENIDO }

Estado estado = Estado.CORRIENDO;

switch (estado) {
    case Estado.IDLE: {
        print("En reposo");
    }
    case Estado.CORRIENDO: {
        print("Ejecutando");
    }
    case Estado.PAUSADO: {
        print("En pausa");
    }
    case Estado.DETENIDO: {
        print("Detenido");
    }
}
```

---

## Combinacion de flags con OR

Cuando los valores del enum son potencias de dos, se pueden combinar como flags:

```c
enum Permiso {
    NINGUNO  = 0,
    LEER     = 1,   // 0b001
    ESCRIBIR = 2,   // 0b010
    EJECUTAR = 4    // 0b100
}

// Combinar permisos
uint64_t mis_permisos = Permiso.LEER | Permiso.ESCRIBIR;  // 0b011 = 3

// Comprobar si tiene un permiso especifico
if (mis_permisos & Permiso.LEER) {
    print("Tiene permiso de lectura");
}
```

---

## Tabla resumen

| Caracteristica          | Descripcion                                              |
| :---------------------- | :------------------------------------------------------- |
| Tipo base               | `uint64_t` (entero de 64 bits)                           |
| Valor por defecto       | Empieza en 0, incrementa de 1 en 1                       |
| Valores explicitos      | Se pueden asignar valores enteros arbitrarios            |
| Variantes con datos     | Cada variante puede llevar campos adicionales            |
| Uso tipico con switch   | El compilador puede usar JUMPTABLE para O(1)             |

---

Ver tambien:
- [[Bucles.md]] - uso de enum en bucles for-in
- [[SetInstruccionesVM/JUMPTABLE_TYPESWITCH.md]] - despacho O(1) para enums
- [[POO.md]] - clases y tipos mas complejos
