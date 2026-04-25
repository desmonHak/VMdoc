# VELB - VEL Binary Format

**VELB** es el formato binario compilado para VestaVM. Es el equivalente al `.class`
de Java o al `.exe` de Windows: el archivo que la VM puede cargar y ejecutar directamente.

**Analogia:** si tu programa `.vel` es el plano de un edificio (codigo fuente legible),
el `.velb` es el edificio ya construido (codigo binario listo para ejecutar). El
compilador transforma el plano en el edificio.

VELB es el formato binario del lenguaje para la maquina virtual VEL.
En este formato se definen las secciones de codigo, datos y metadatos necesarios para que tu codigo sea cargado y ejecutado por la VM.

Tecnicamente hablando, la VM podria cargar unicamente las instrucciones que quieres ejecutar, y esto estaria bien, pero normalmente nos interesara saber meta informacion de nuestro codigo, para poder realizar cosas como la reflexion, o el manejo de excepciones.

----

El formato empieza con un [[Header_VELB]] el cual es la cabecera del ejecutable y contiene toda la informacion basica para que el loader pueda cargar o interpretar el mismo.