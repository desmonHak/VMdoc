# Instrumentacion: `@Hook`

`@Hook(<punto>)` marca una funcion como **proveedora** del gancho que se llama
al entrar o al salir de las demas. Es el equivalente de
`-finstrument-functions` de GCC/Clang, pero como notacion del lenguaje: lo
mismo sirve para perfilar que para trazar, contar cobertura o auditar.

```vesta
@Hook(enter)
void al_entrar(string fn_name) {
	println("-> ${fn_name}");
}

@Hook(exit)
void al_salir(string fn_name) {
	println("<- ${fn_name}");
}
```

Tres propiedades que lo definen:

- **Su sola presencia lo activa.** Sin ningun `@Hook` no se teje nada y el
  binario sale exactamente igual, igual que con `@AllocatorOverride` o
  `@PanicHandler`. No hay flag que recordar.
- **Se resuelve ENTERO al compilar**, asi que baja a una llamada normal y
  funciona en los tres modos -- incluido el binario nativo en `--target bare`,
  donde el [`@Aspect` dinamico](ReflexionAOP.md) no llega porque registra el
  advice en ejecucion.
- **El gancho declara la firma que QUIERE**, y se le pasan esos campos y solo
  esos.

---

## 1. Los puntos

| Punto | Cuando |
| :------- | :---------------------------------------- |
| `enter` | al entrar en la funcion |
| `exit` | al salir de la funcion |
| `unwind` | al capturar una excepcion, tras el salto |

`unwind` existe por una razon que se ve en cuanto se mide: **una excepcion no
sale por el epilogo**. Salta al `catch`, y las funciones que atraviesa no
llaman a su gancho de `exit` -- con tres funciones anidadas salen 3 entradas y
1 sola salida --. Sin `unwind`, un gancho que lleve una pila no puede
rebobinarla. El compilador avisa (`VXW932`) si el modulo lanza excepciones y
hay un `@Hook(exit)` sin su `unwind`.

---

## 2. Los campos

El gancho pide en su firma los campos que necesita, por NOMBRE:

| Campo | Tipo | Disponible en | Que es |
| :---------- | :------- | :-------------------- | :----- |
| `fn_id` | `u32` | enter, exit, unwind | identifica la funcion instrumentada |
| `fn_name` | `string` | enter, exit, unwind | su nombre, ya aplanado |
| `call_site` | `u64` | enter, exit | la direccion a la que volvera: QUIEN la llamo |
| `ret_value` | `u64` | exit | lo que devuelve |

Pedir un campo que ese punto no ofrece -- `ret_value` en `enter`, por ejemplo
-- es un error de compilacion que **enumera los que si hay** (`VXE933`).

Dos notas que ahorran una medicion mentirosa:

- **`fn_id` es un numero, no una direccion**, para que quepa en un inmediato y
  funcione tambien sin tabla de simbolos. `fn_name` es lo que se imprime.
- **`call_site` es la direccion del espacio donde se EJECUTA** -- de la maquina
  virtual al interpretar, del proceso al compilar --, asi que sirve para
  distinguir sitios de llamada, no para imprimirla esperando el mismo numero en
  los tres modos.

---

## 3. Que se instrumenta, y que no

Por defecto, todo. Se acota de dos maneras:

### 3.1. Un selector en el `@Hook`

```vesta
@Hook(enter, "std.*")
void solo_la_stdlib(string fn_name) { ... }
```

El selector es una cadena con un glob contra el nombre completo. Si no casa con
ninguna funcion, el compilador **avisa** (`VXW931`): sin eso el programa
compila, corre y no instrumenta nada, o sea que medirias nada creyendo que
mediste.

### 3.2. `@NoInstrument` en la funcion

```vesta
@NoInstrument
i32 sin_medir(i32 x) => x + 1;
```

El compilador ya excluye al **propio gancho**; esto es para las funciones de
apoyo que el gancho llame. Es lo que permite instrumentar todo lo demas sin
que el gancho se mida a si mismo y entre en recursion.

---

## 4. Lo que el compilador SABE decir

Ninguno de estos mensajes se limita a rechazar: enumeran lo disponible, y esa
lista sale de la tabla de `include/vx/hook_points.h`, no del texto. Al anadir
un punto o un campo, el diagnostico se actualiza solo.

| Codigo | Que dice |
| :------- | :------- |
| `VXE930` | falta el punto: `@Hook(<punto>)`, con la lista |
| `VXE931` | punto desconocido, con la lista |
| `VXE932` | el selector tiene que ser una cadena |
| `VXE933` | ese campo no esta en ese punto, con los que si |
| `VXW930` | el campo aun no se rellena: el gancho lo recibira como 0 |
| `VXW931` | el selector no casa con nada: correra SIN instrumentar |
| `VXW932` | hay excepciones y un `exit` sin `unwind`: la cuenta no cuadrara |
| `VXW934` | un `asm` que escribe `rsp` invalida el `call_site` |

El ultimo merece una nota: **se puede decir porque el bloque `asm` no es
opaco**. La tabla de efectos ya declara que `push`/`pop` escriben `rsp`, asi
que el compilador sabe cuando la direccion de retorno deja de estar donde el
codigo la va a buscar. Otros compiladores tratan el asm en linea como una
barrera y devolverian el valor igual, mal.

---

## 5. `@Hook` frente a `@Aspect`

Los dos tejen codigo alrededor de una funcion, y no compiten:

| | `@Hook` | [`@Aspect`](ReflexionAOP.md) |
| :--------------- | :------------------------ | :--------------------------- |
| cuando se resuelve | al compilar | en ejecucion (registra advice) |
| a que se aplica | funciones, por glob | metodos, por pointcut |
| que puede hacer | observar | observar y SUSTITUIR (`@Around`) |
| en `--target bare` | si | no |

`@Hook` es para medir; `@Aspect` para cambiar el comportamiento.

---

## Ver tambien

- `examples_codes_vx/543_hook_instrumentacion.vx` -- los tres puntos, con la
  traza que demuestra que `unwind` hace falta.
- `examples_codes_vx/544_hook_nombres.vx` -- un trazador que imprime el nombre.
- [Reflexion y AOP](ReflexionAOP.md) -- el tejido dinamico.
- [Excepciones](Excepciones.md) -- por que una excepcion no pasa por `exit`.
