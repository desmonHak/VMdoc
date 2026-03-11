Toda dirección de la maquina virtual esta mapeada de memoria real que suele gestionar la VM.
Las direcciones de memoria de la VM se signa a través de paginas de 4096 bytes, por esto todo programador o usuario deberá asegurarse que la dirección que quiere usar para mapear memoria real, este alineado a 4096 y no se halla mapeada ya.
La VM mapea memoria de la siguiente manera:
```c
/**
 * Representa los permisos que tiene la memoria del host, y los permisos virtuales asociados,
 * validos para la VM.
 */
typedef struct permissions_mem_t {
    union {
        struct {
            uint8_t real_r:1;
            uint8_t real_w:1;
            uint8_t real_x:1;
            uint8_t real_padding:1; // relleno de un bit

            uint8_t vm_r:1;
            uint8_t vm_w:1;
            uint8_t vm_x:1;
            uint8_t vm_padding:1; // relleno de un bit
        };
        uint8_t raw;
    };
} permissions_mem_t;

/**
 * tipo de dato para punteros reales a mapear a la VM
 */
typedef void* real_ptr;

/**
 * punteros virtuales de la VM.
 */
typedef uint64_t vm_ptr;


/**
 * Estructura que asocia un puntero de la maquina virtual, con un puntero de la maquina real.
 * Los punteros de la maquina virtual no tienen por que coincidir con los punteros de la maquina real.
 *
 * si el "pointer_host" acaba en "pointer_host + size_mem", la direccion final representativa en la VM
 * sera "pointer_vm + size_mem".
 *
 * Los punteros virtuales que se establezcan manualmente, "pointer_vm" deberan ser multiplo de 4096.
 * Para esto se debe llamar a la instruccion `ptr_translation`, donde se le pasa a "r00"
 * una direccion del host que sea valida y accesible en el espacio de direcciones real de la VM,
 * a "r01" el tamaño de la memoria a ser accedida y en "r02" el puntero virtual multiplo de 4096 que
 * se quiere usar para acceder a la memoria.
 *
 * La VM permite reservar memoria, al usar "vmresb" la maquina virtual retorna una direccion virtual
 * en el registro "r00" que tiene asociada una direccion en la maquina real, el tamaño de esta direccion
 * virtual es de 4096 bytes.
 *
 */
typedef struct VMPointerTranslation {
    real_ptr         pointer_host;  /** puntero del host (puntero real en la maquina real). */
    vm_ptr             pointer_vm;  /** puntero representativo en la VM.*/
    size_t               size_mem;  /** tamaño de esta memoria, dependiente de "pointer_host". */
    permissions_mem_t permissions;  /** Permisos de esta memoria */
} VMPointerTranslation;
```
El puntero de la maquina host no necesita necesariamente que sea múltiplo de 4096, pero el puntero que se use en la maquina virtual para mapear esta dirección, si debe ser múltiplo de 4096, si la memoria a la que se quiere acceder es mayor a 4096 bytes, se deberá reservar de 4k en 4k hasta completar la reserva solicitada.

Los permisos que se asignen a la memoria, deberán respetarse, tantos los permisos virtual asignados por la VM, como los permisos reales asignados por el OS de turno.

----

Toda la memoria mapeada internamente se mapea como una Tabla de hash, donde la dirección virtual múltiplo de 4k hace de clave y donde y el valor obtenido equivale a el puntero mapeado:
```c
typedef struct VM_t {
    bytecode_table_t*   table;                  /** tabla de bytecode? */

    HashTable*          mem;                    /** tabla hash que representa la memoria
                                                 * la direcciones de memoria virtuales
                                                 * son la key que se usan para acceder a este
                                                 * hash map que contiene las propiedades de la
                                                 * memoria accedida
                                                 */

    ArrayList* /* ThreadVM_t* */ threads;       /** Array de hilos que la VM debe de ejecutar */

    ArrayList*/* module_t* */ table_modules;    /** tabla de modulos cargados `module_t` */

    ssize_t last_thread_index;                  /** Último hilo que fue RUNNING */
    state_vm    state;                          /** Estado de la maquina virtual */
} VM_t;

```