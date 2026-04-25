# TLB - Translation Lookaside Buffer (Cache de traduccion de direcciones)

El **TLB** es una cache que acelera la traduccion de **direcciones virtuales de la VM**
a **direcciones fisicas en la memoria del proceso host**.

**Analogia:** cuando buscas una palabra en un diccionario, la primera vez tardas porque
tienes que hojear las paginas. Pero si usas un marcapaginas o tomas nota de la pagina
donde estaba, la segunda busqueda es instantanea. El TLB es ese marcapaginas: guarda
las traducciones de direcciones que ya hizo antes.

---

## Por que VestaVM necesita traduccion de direcciones

Los programas VestaVM usan **direcciones virtuales** (numeros arbitrarios como `0x1000`).
Estas no son direcciones reales en la RAM del ordenador: son numeros que la VM traduce
a posiciones de memoria reales cuando el programa accede a ellas.

Esta traduccion se hace a traves de una **tabla de paginas de 3 niveles** (similar
al sistema de paginacion de los CPU x64 modernos):

```
Direccion virtual de 64 bits:
[24 bits PT2][16 bits PT1][12 bits PT][12 bits offset]
```

---

## Estructura de la tabla de paginas

| Nivel    | Bits | Mascara    | Entradas maximas | Tamano de tabla | Desplazamiento |
| :------- | :--: | :--------- | :--------------: | :-------------- | :------------: |
| PT2      |  24  | `0xFFFFFF` | 16.777.216 (16M) | 128 MB          | bit 40         |
| PT1      |  16  | `0xFFFF`   | 65.536 (64K)     | 512 KB          | bit 24 (*)     |
| PT       |  12  | `0xFFF`    | 4.096 (4K)       | 32 KB           | bit 12         |
| Offset   |  12  | `0xFFF`    | -                | 4 KB (pagina)   | bit 0          |

(*) En la implementacion real, el desplazamiento de PT1 es bit 32 para el rango de 64 bits.

---

## Diagrama de la direccion virtual

```
Bit 63                                                   Bit 0
+------------------+------------------+----------+----------+
|   PT2 (24 bits)  |   PT1 (16 bits)  |  PT (12) | Off (12) |
+------------------+------------------+----------+----------+
   0xFFFFFF           0xFFFF             0xFFF      0xFFF
```

---

## Como funciona el TLB en VestaVM

1. El programa accede a la direccion virtual `0xABCDEF1234`.
2. La VM consulta el TLB: "he traducido esta direccion antes?".
3. Si esta en el TLB (**TLB hit**): usa la direccion host almacenada. O(1).
4. Si NO esta en el TLB (**TLB miss**): recorre la tabla de paginas de 3 niveles,
   obtiene la direccion host, y la guarda en el TLB para futuras consultas.

El TLB hace que el acceso a memoria sea rapido en el caso comun (acceso repetido
a las mismas paginas), sin necesidad de recorrer los 3 niveles cada vez.

---

## Invalidacion del TLB

Cuando se mapea o desmapea una pagina (via `RESBP` o `FREEP`), la entrada
correspondiente del TLB se invalida para evitar traducir a una direccion ya
no valida.

---

## Relacion con ArenaManager

El TLB no gestiona la memoria directamente: eso lo hace el `ArenaManager`.
El TLB es solo la cache de traduccion de direcciones. El `ArenaManager` mantiene
las arenas de memoria reales y actualiza la tabla de paginas cuando se reserva
o libera una pagina.

```
Programa: accede a 0x1000
     |
     v
TLB: esta en cache? -> Si -> usar dir host directamente (rapido)
                    -> No -> recorrer PT2 -> PT1 -> PT -> obtener dir host
                             -> guardar en TLB -> usar dir host
```

---

Ver tambien:
- [[VmManager.md]] - gestion de instancias y memoria compartida
- [[../SetInstruccionesVM/RESBP.md]] - reservar paginas de memoria
- [[../SetInstruccionesVM/FREEP.md]] - liberar paginas de memoria
- [[../SetInstruccionesVM/PAGEINFO.md]] - consultar informacion de una pagina
- [[../Loader/EspacioDeDireccionesDeLaMemoria.md]] - mapa del espacio de direcciones
