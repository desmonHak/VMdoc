## Instrucciones de creación, control y finalización de hilo

- `thinit`: permite crear un nuevo hilo en la VM. Se debe configurar el hilo usando registros, para lo cual:
	- `r00`: se indicara el puntero inicial base y tope de pila para el hilo.
	- `r01`: limite de la pila, una vez el tope de la pila alcance este limite, el hilo no podrá acceder mas allá. 
	- En caso de que las instrucciones de pila u otras similares accedan mas allá, se generara en el hilo un `THREAD_SEGMENTATION_FAULT`, en caso de que el registro tope de pila llegue a ser inferior al puntero base de pila, se generara un `THREAD_STACK_UNDERFLOW`.
	- `r02`: puntero a la primera instrucción a ejecutar, se configurar el registro IP (___Instruction Pointer___).
	- `r03`: Puntero donde acaba el código ejecutable para este hilo, el hilo, se marcara como ``DEAD`` al llevar ``IP`` mas allá de esta dirección (``code_limit``), o hasta que se ejecute la instruccion `th_dead <reg>` (`thsetst  3 <reg>`), que cambiara el hilo a muerto.
Al ejecutar la instrucción, el hilo se marcara como ``READY``, y su ___TID___ se devolverá en el registro `r15`, se le podrá ceder el control usando `yield <reg>`
----
- `thgetst <reg>`: La instrucción `thgetst` (___Thread Get State___) Permite obtener el estado de un hilo en el registro `r00`. Para indicar el hilo del que obtener el estado, se debe ejecutar la instrucción con un registro que contenga el `tid` del hilo.
----
- `thsetst  <inmed4> <reg>`: 
	- `th_ready <reg>` alias `thsetst  0 <reg>` : Marca el hilo como ``READY`` (Listo para ejecutarse)
	- `th_running <reg>` alias `thsetst  1 <reg>` : No se permite, solo la VM controla el hilo que se marcara como ejecutando.
	- `th_blocked <reg>` alias `thsetst  2 <reg>`: Marca el hilo como `BLOCKED` (hilo bloqueado), puede desbloquear por un hilo externo que lo marque como `READY` o reactivarse por que se puso a dormir con `th_sleep`.
	- `th_dead <reg>` alias `thsetst  3 <reg>`: Marca el hilo como ``DEAD`` lo que indica que el hilo finalizo su ejecución o se mato.

### Instrucciones de control del hilo:
- `th_err`: Permite obtener el código de error del hilo:
```c
/**
 * Estados de error de los hilos
 */
typedef enum state_err_thread {
    THREAD_NO_ERROR = 0,            /** Sin error */
    THREAD_UNKNOWN_ERROR,           /** Error no clasificado */
    THREAD_SEGMENTATION_FAULT,      /** Un hilo intento acceder a memoria a la cual no tiene permisos */
    THREAD_ILLEGAL_INSTRUCTION,     /** Instrucción no reconocida o prohibida */
    THREAD_DIVISION_BY_ZERO,        /** División por cero */

    THREAD_STACK_OVERFLOW,          /** Stack del hilo se desbordó:
                                     * El tope de pila(sp) se encontro con el limite de pila del hilo.
                                     * Push más allá del límite de stack
                                     */

    THREAD_STACK_UNDERFLOW,         /**
                                     * Stack se leyó cuando estaba vacío( Pop de stack vacío ),
                                     *  SP se intento decrementar a un valor inferios a BP,
                                     *  el tope de pila siempre debe de ser superior o igual a
                                     *  el puntero de la base de pila
                                     */
    THREAD_INVALID_SYSCALL,         /** Llamada al sistema inválida o no soportada */
} state_err_thread;
```

- ``th_sleep``:
La instrucción duerme el hilo actual en ejecución. Se espera recibir el tiempo que dormir el hilo, en `r00`. Debe ser una hora en nanosegundo a la que despertar el hilo. Configura el campo interno `time_sleep`. Al dormir el hilo se marca como `BLOCKED`

- `yield`: Permite ceder el control para que el siguiente hilo sea ejecutado. Esta instrucción puede ejecutarse aunque el mecanismo de **"Quantum de ejecución"**/**"Time slice"**  este activo.
- `yield <reg>`: Permite ceder el control a un hilo especificado en un registro, por un ___TID___

- ``setts <inmed16>``
- `setts <reg>`: La instrucción `setts` (___Set Time Slice___) permite configurar el **"Quantum de ejecución"**/**"Time slice"**, debe ser un valor de 16 bits o un valor de hasta 64bits contenido en un registro. Para desactivar este mecanismo, en el hilo actual, llamar a la instrucción con valor 0 para configurar el hilo, que cambiara el campo interno `instructions_remaining`.

- `callthblock`: (``call thread block``) la función permite a los hilos emulados llamar a funciones externas bloqueantes, esto es necesario si se quiere que la función externa llamada no bloque todo el hilo principal de la VM. Esta instrucción especial creara un hilo real al momento de realizar la llamada externa, lo que permitirá seguir la ejecución del resto de la VM