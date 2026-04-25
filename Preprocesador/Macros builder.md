# Macros builder

Las **macros builder** son un tipo especial de macro del preprocesador de Vesta que
permiten construir estructuras de datos complejas o configuraciones de forma declarativa
y legible, que luego se transforman en codigo o datos antes de la compilacion.

**Analogia:** piensa en un asistente de configuracion grafico. En lugar de escribir
largas estructuras de codigo a mano, describes lo que quieres de forma simple y el
asistente genera el codigo final por ti.

---

## Forma general

Una macro builder se define usando `#builder` y acepta un bloque de configuracion
entre llaves:

```c
// Definicion de una macro builder para crear configuraciones de red
#builder NetworkConfig {
    host:     String = "localhost";  // campo con valor por defecto
    port:     uint16_t = 8080;
    timeout:  uint32_t = 5000;
    tls:      bool = false;
}

// Uso: crear una configuracion sobreescribiendo solo lo necesario
NetworkConfig config = NetworkConfig {
    host    = "192.168.1.10";  // sobreescribir host
    port    = 9000;            // sobreescribir port
    // timeout y tls usan los valores por defecto
};
```

El preprocesador expande esto a la inicializacion del struct o clase correspondiente.

---

## Builder para estructuras de datos

Los builders son especialmente utiles para crear objetos complejos paso a paso,
similar al patron de diseno "Builder" del mundo OOP:

```c
// Builder para construir una peticion HTTP
#builder HttpRequest {
    method:  String = "GET";
    url:     String;          // campo obligatorio (sin valor por defecto)
    headers: HashMap<String, String> = new HashMap();
    body:    String = "";
    timeout: uint32_t = 30000;
}

// Uso
HttpRequest req = HttpRequest {
    method = "POST";
    url    = "http://api.ejemplo.com/datos";
    body   = "{\"clave\": \"valor\"}";
    // headers y timeout usan los valores por defecto
};
```

---

## Builder para tablas de dispatch

Un uso avanzado de los builders es construir tablas de despacho (`JUMPTABLE`)
en tiempo de compilacion:

```c
// Builder para una tabla de opcodes
#builder OpcodeTable {
    entries: [(uint8_t opcode, FunctionPtr handler)] = [];
}

OpcodeTable mi_tabla = OpcodeTable {
    entries = [
        (0x01, handler_add),
        (0x02, handler_sub),
        (0x03, handler_mul),
        (0x04, handler_div),
    ];
};
```

El preprocesador genera el array de punteros de funcion en la seccion `.rodata`.

---

## Diferencias con structs ordinarios

| Caracteristica           | Struct ordinario         | Macro builder                         |
| :----------------------- | :----------------------- | :------------------------------------ |
| Valores por defecto      | No (inicializacion manual) | Si (especificados en la definicion)  |
| Campos opcionales        | No (todos obligatorios)  | Si (los que tienen valor por defecto) |
| Verificacion de campos   | En tiempo de compilacion | En preprocesamiento                   |
| Expresividad             | Baja (solo datos)        | Alta (puede incluir logica de build)  |

---

Ver tambien:
- [[Macros parametricas.md]] - macros con argumentos
- [[Macros anotaciones.md]] - anotaciones de configuracion
- [[../SintaxisCore/Estructuras (struct).md]] - structs de bajo nivel
- [[../SintaxisCore/POO.md]] - clases OOP completas
