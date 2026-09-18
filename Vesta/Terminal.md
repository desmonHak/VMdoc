# Control del terminal

Vesta trae el control del terminal como **builtins del lenguaje**, no como una
libreria: color de 24 bits, movimiento del cursor y borrado de pantalla. Bajan a
las secuencias VT100 por la misma primitiva que `print`, asi que funcionan
identico en interprete, JIT y binario nativo -- y sin enlazar nada.

```vesta
i32 main() {
	term_clear();
	term_move(2, 3);
	println("${fg_rgb(255, 128, 0)}naranja${RESET}");
	return 0;
}
```

---

## 1. Color de 24 bits

`fg_rgb(r, g, b)` y `bg_rgb(r, g, b)` se usan **dentro de una interpolacion**,
donde emiten la secuencia SGR de truecolor. Los componentes pueden ser literales
o valores de ejecucion.

```vesta
println("${fg_rgb(255, 128, 0)}naranja${RESET}");
println("${bg_rgb(0, 0, 255)}fondo azul${RESET}");

i32 r = 10;
i32 g = 200;
i32 b = 30;
println("${fg_rgb(r, g, b)}rgb de runtime${RESET}");
```

`RESET` es uno de los [identificadores ANSI magicos](Strings.md) (`RED`,
`GREEN`, `BOLD`, ...) que ya existen en la interpolacion. Los 16 colores
clasicos se escriben con ellos; `fg_rgb`/`bg_rgb` son para cuando hacen falta
los 24 bits.

Ejemplo: `examples_codes_vx/181_truecolor.vx`.

---

## 2. Cursor y pantalla

Estos van como **sentencias**, no dentro de una cadena:

| Builtin | Que emite | Para que |
| :------------------------ | :---------- | :------- |
| `term_clear()` | `ESC[2J ESC[H` | borra la pantalla y lleva el cursor al origen |
| `term_clear_line()` | `ESC[2K` | borra la linea actual |
| `term_move(fila, col)` | `ESC[f;cH` | coloca el cursor (1-based, como VT100) |
| `term_save_cursor()` | `ESC[s` | guarda la posicion |
| `term_restore_cursor()` | `ESC[u` | la recupera |
| `term_hide_cursor()` | `ESC[?25l` | lo oculta |
| `term_show_cursor()` | `ESC[?25h` | lo muestra |
| `term_reset()` | `ESC[0m` | vuelve a los atributos por defecto |

```vesta
i32 main() {
	term_hide_cursor();
	term_clear();

	for (i64 i = 0; i < 5; i = i + 1) {
		term_move((i32)i + 1, 1);
		term_clear_line();
		println("${fg_rgb(0, 200, 0)}linea ${i}${RESET}");
	}

	term_move(7, 1);
	term_show_cursor();
	term_reset();
	return 0;
}
```

> **El par que hay que cerrar siempre es `term_hide_cursor` /
> `term_show_cursor`.** Si el programa termina -- o revienta -- con el cursor
> oculto, la terminal se queda asi despues de salir. Con
> [excepciones](Excepciones.md) por medio, el sitio de `term_show_cursor()` es
> un `finally`.

---

## 3. Lo que NO hace

- **No lee el terminal**: no hay tamano de ventana, ni modo crudo, ni teclas.
  Solo escribe secuencias.
- **No detecta si la salida es un terminal.** Redirigida a un fichero, las
  secuencias van al fichero. Si eso importa, decidelo tu antes de llamarlas.
- **No hay base de datos de terminales** (terminfo): son escapes VT100 fijos,
  que es lo que entienden todos los emuladores actuales, incluida la consola de
  Windows 10 en adelante.

---

## Ver tambien

- [Strings](Strings.md) -- interpolacion, especificadores de formato
  (`${n:hex:>20}`) y los identificadores ANSI magicos.
- `examples_codes_vx/181_truecolor.vx` -- el color de 24 bits.
