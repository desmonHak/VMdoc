La VM pose dos tipos de multihilo. Si la VM ejecuta el bytecode en crudo, entonces se uso un modelo de [[MultithreadGreen|Green thread]] donde los únicos hilos reales que existen, es a la hora de hacer call's externas a la VM y en cada instancia de VM.
Se puede usar varias estancias de VM para emular multiples hilos

En cambio si se usa el modo de ejecución JIT, se procede a la compilación en run time y al uso de hilos en tiempo de ejecución.