# Sistema de Paquetes de VestaVM

Modulo: `include/loader/package.h`, `src/loader/package.cpp`

## Vision general

El sistema de paquetes permite organizar codigo Vesta en modulos con dependencias
declaradas, versiones y puntos de entrada. El archivo de manifiesto es opcional:
un fichero `.vel` sin manifiesto puede compilarse y ejecutarse directamente.

---

## Manifiesto: package.vel

El archivo `package.vel` debe estar en la raiz del directorio del paquete.

### Formato

```
package "nombre.del.paquete" 1.2.3

entry "src/main.vel"

dep "org.vendor.lib"    >= 2.0.0
dep "org.another.pkg"  == 1.5.0
dep "com.util"         ^  3.0.0
```

### Campos

| Campo     | Descripcion                                              |
| :-------- | :------------------------------------------------------- |
| `package` | Nombre completo del paquete y su version (mayor.menor.parche) |
| `entry`   | Ruta relativa al archivo `.vel` de entrada (opcional)    |
| `dep`     | Dependencia con operador de version                      |

### Operadores de version

| Operador | Semantica                                                     |
| :------- | :------------------------------------------------------------ |
| `>=`     | Version mayor o igual (cualquier patch)                       |
| `==`     | Version exacta (los tres componentes deben coincidir)         |
| `^`      | Compatible semver: mismo major, minor >= indicado             |
| `~`      | Patch flexible: mismo major.minor, patch >= indicado          |
| `<`      | Version estrictamente menor                                   |
| `>`      | Version estrictamente mayor                                   |

---

## Busqueda de paquetes: VESTA_PKG_PATH

Cuando `PackageLoader::resolve()` busca una dependencia, recorre los directorios
listados en la variable de entorno `VESTA_PKG_PATH`:

- **Linux/macOS:** separados por `:`
- **Windows:** separados por `;`

En cada directorio busca `<nombre_paquete>/package.vel` donde el nombre del
paquete se convierte a ruta reemplazando `.` por `/`.

Ejemplo:
```
VESTA_PKG_PATH=/opt/vesta/pkgs:/home/user/vesta_libs
# Para "org.vendor.lib" busca:
#   /opt/vesta/pkgs/org/vendor/lib/package.vel
#   /home/user/vesta_libs/org/vendor/lib/package.vel
```

---

## Resolucion de dependencias

`PackageLoader::resolve(root_path)` implementa una resolucion DFS con:

1. **Deteccion de ciclos:** estado `PKG_IN_STACK` durante la visita activa.
2. **Verificacion de versiones:** `version_satisfies(dep_op, dep_ver, available_ver)`.
3. **Orden topologico:** postorder (dependencias antes que el dependiente).

### Estados de los nodos

| Estado           | Significado                                  |
| :--------------- | :------------------------------------------- |
| `PKG_UNVISITED`  | No visitado aun                              |
| `PKG_IN_STACK`   | En la pila de recursion actual (ciclo si se visita de nuevo) |
| `PKG_RESOLVED`   | Resuelto correctamente                       |
| `PKG_ERROR`      | Error durante la resolucion                  |

### Resultado

`PkgLoadResult` contiene:
- `order`: vector de `PkgNode*` en orden topologico (las dependencias primero).
- `error`: `PkgError` con el codigo y los nombres de los paquetes involucrados.

### Codigos de error

| Codigo                   | Causa                                                |
| :----------------------- | :--------------------------------------------------- |
| `PKG_ERR_NONE`           | Sin errores                                          |
| `PKG_ERR_NOT_FOUND`      | Paquete no encontrado en VESTA_PKG_PATH              |
| `PKG_ERR_VERSION_MISMATCH` | La version disponible no satisface el operador     |
| `PKG_ERR_CYCLE`          | Ciclo de dependencias detectado                      |
| `PKG_ERR_PARSE`          | Error de sintaxis en un package.vel                  |

---

## API de C++

```cpp
#include "loader/package.h"

loader::PackageLoader pkg_loader;

// Parsear el manifiesto raiz
loader::PkgManifest manifest;
if (!pkg_loader.parse_manifest("mi_proyecto/package.vel", manifest)) {
    // error de sintaxis
}

// Resolver dependencias (accede a VESTA_PKG_PATH)
loader::PkgLoadResult result = pkg_loader.resolve("mi_proyecto");
if (result.error.code != loader::PKG_ERR_NONE) {
    // manejar error
}

// Iterar en orden topologico
for (loader::PkgNode *node : result.order) {
    // node->manifest.name, node->manifest.version
    // node->manifest.entry  (ruta al .vel de entrada)
}

pkg_load_result_free(&result); // liberar memoria
```

---

## Carga dinamica vs. compilacion estatica

El sistema soporta dos modelos de uso:

| Modelo            | Descripcion                                              |
| :---------------- | :------------------------------------------------------- |
| **Dinamico**      | Los paquetes se buscan y cargan en tiempo de ejecucion usando VESTA_PKG_PATH |
| **Estatico**      | El compilador HLL resuelve dependencias en tiempo de compilacion, genera bytecode enlazado |

En el modelo estatico el HLL puede usar `PackageLoader` para obtener el orden
topologico y compilar todos los modulos en el orden correcto antes de invocar
al linker de VestaVM.

---

## Ejemplo de package.vel

```
package "com.misempresa.miapp" 1.0.0

entry "src/main.vel"

dep "org.vendor.serializer"  >= 2.1.0
dep "com.util.collections"   ^  1.0.0
```
