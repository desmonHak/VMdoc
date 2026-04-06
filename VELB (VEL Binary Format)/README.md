VELB es el formato binario del lenguaje para la maquina virtual VEL.
En este formato se definen las secciones de código, datos y metadatos necesarios para que tu código sea cargado y ejecutado por la VM.

Técnicamente hablando, la VM podría cargar únicamente las instrucciones que quieres ejecutar, y esto estaria bien, pero normalmente nos interesara saber meta información de nuestro código, para poder realizar cosas como la reflexión, o el manejo de excepciones.

----

El formato empieza con un [[Header_VELB]] el cual es la cabecera del ejecutable y contiene toda la información básica para que el loader pueda cargar o interpretar el mismo.