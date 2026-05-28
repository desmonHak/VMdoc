# Carga dinámica de módulos

Vex soporta cargar `.velb` adicionales en runtime mediante
`loadmodule(path)` y descargarlos con `unloadmodule(path)`. Es útil
para:

- **Plugins**: extender un programa sin recompilarlo.
- **Hot-reload**: recargar `.velb` modificado sin reiniciar la VM.
- **REPL**: cargar código de usuario sobre la marcha.
- **Módulos opcionales**: cargar features pesados solo cuando se invoquen.

Documentos relacionados:

- [[Modulos]] — sistema estático de módulos (compile-time `import`)
- [[Sandbox]] — capabilities sobre módulos cargados dinámicamente
- [[ReflexionAOP]] — acceder a clases del plugin vía `forName`, `newInstance`, `invoke`

## Builtins

### `loadmodule(path: string_lit) -> i64 init_pc`

Carga el `.velb` desde el filesystem. Devuelve `init_pc` (distinto de
cero si la carga fue correcta, 0 si falló). Tras la carga, las clases
del plugin quedan registradas en el `ClassRegistry` global y son
accesibles por reflexión.

```java
i32 main() {
    i64 result = loadmodule("plugins/extra.velb");
    if (result == 0) {
        println("Error: no se pudo cargar el plugin");
        return -1;
    }
    // El plugin ya está cargado. Sus clases públicas son accesibles.
    i64 cls = forName("PluginExtension");
    if (cls == 0) {
        println("El plugin no expone PluginExtension");
        return -1;
    }
    i64 obj = newInstance(cls);
    i64 method = getMethod(cls, "execute");
    i64 r = invoke(method, obj);
    return (i32) r;
}
```

Restricciones:

- `path` debe ser un **string literal** en tiempo de compilación. No se admite variable.
- Path absoluto o relativo al cwd del proceso VM.
- Máximo 8192 bytes de path.

Compilación del plugin (preventiva contra solapamiento de VA):

```bash
# Compilar el plugin con base address distinta de 0 para evitar solape
# con el caller cuando se cargue.
vm --vex plugin.vex -o plugin --vex-base 0x10000000
```

Tras `loadmodule`, el loader aplica **rebase transparente** si la VA
del plugin solapa con executables ya cargados. El usuario no necesita
coordinar VAs manualmente; el flag `--vex-base` es solo una sugerencia
y el loader la respeta si no hay conflicto, reasignando
automáticamente en caso contrario.

### `unloadmodule(path: string_lit) -> i32 status`

Descarga un módulo previamente cargado. Devuelve `1` si se descargó,
`0` si el path no estaba cargado.

```java
i32 main() {
    loadmodule("plugins/v1.velb");
    // ... uso del plugin ...
    i32 ok = unloadmodule("plugins/v1.velb");
    // Tras unload: el bytecode y datos del plugin son liberados.
    // Las clases registradas en ClassRegistry SOBREVIVEN (intencional).
    return ok;
}
```

**Comportamiento del ClassRegistry**: las clases registradas por
`__module_init` del plugin no se desregistran al hacer unload. Esto es
intencional: los objetos vivos del runtime cuyo `class_ptr` apunta a
esas clases no quedan colgados. Para invalidar las clases de forma
explícita hay que terminar el proceso entero.

### Patrón hot-reload

```java
string PLUGIN = "plugins/hot.velb";

i32 reload_plugin() {
    unloadmodule(PLUGIN);
    i64 r = loadmodule(PLUGIN);
    return (r == 0) ? -1 : 0;
}

i32 main() {
    loadmodule(PLUGIN);
    while (true) {
        run_plugin_once();
        if (file_modified_since_last_check(PLUGIN)) {
            reload_plugin();
        }
        sleep(100);
    }
}
```

Útil para editores IDE, REPL y sistemas con plugins en desarrollo
activo.

## Cómo interactúa con el sistema estático

Hay **dos sistemas distintos** que pueden coexistir:

| Sistema | Mecanismo | Cuándo usar |
|:---|:---|:---|
| Estático ([[Modulos]]) | `import "lib";` en source | Dependencias conocidas en tiempo de compilación. Type-safe. Inline al `.velb` final. |
| Dinámico (este documento) | `loadmodule("lib.velb")` en runtime | Plugins, hot-reload, REPL. Acceso por reflexión. |

Pueden combinarse:

```java
// El proyecto principal tiene un import estático de utils.
import "utils" only Helper;

i32 main() {
    Helper h = new Helper(); // símbolo estático
    h.do_something();

    // Y carga un plugin dinámicamente.
    loadmodule("plugins/extra.velb");
    i64 cls = forName("ExtraClass"); // símbolo dinámico por reflexión
    i64 obj = newInstance(cls);
    return 0;
}
```

## Reflexión sobre módulos cargados

Tras `loadmodule`, las clases del plugin están registradas en el
ClassRegistry global y se acceden mediante los builtins de reflexión:

```java
loadmodule("plugin.velb");

// Lookup por nombre
i64 cls = forName("PluginClass");

// Inspección
i64 field = getField(cls, "value");
i64 method = getMethod(cls, "increment");

// Instanciación
i64 obj = newInstance(cls);

// Invocación
i64 result = invoke(method, obj);

// Drop manual (opcional; el GC los libera cuando ningún root los referencia)
drop(obj);
```

Ver [[ReflexionAOP]] para la lista completa de builtins de reflexión.

## Cómo interactúa con el sandbox

Los módulos cargados dinámicamente HEREDAN las capabilities del
caller con AND-mask (sin posibilidad de escalation):

```bash
vm --run main.velb --vex-caps "fs:read=/tmp,loadmod"
# El main.velb tiene fs:read=/tmp + loadmod.
# Si carga "plugin.velb" mediante loadmodule, el plugin hereda esas
# mismas caps (AND con las suyas propias = idénticas).
```

Si el caller no tiene la cap `loadmod`, `loadmodule` falla con
`FatalError(SECURITY_VIOLATION)` capturable mediante `try/catch`:

```java
try {
    loadmodule("plugin.velb");
} catch (FatalError e) {
    println("Sandbox bloqueo: ${e.message}");
    return -1;
}
```

Adicionalmente, si el caller tiene `loadmod` con whitelist de paths,
los paths fuera del whitelist son rechazados:

```bash
vm --run main.velb --vex-caps "loadmod=/var/plugins"
# Solo puede cargar plugins de /var/plugins/*.velb
```

Ver [[Sandbox]] para la lista completa de capabilities.

## Convenciones del plugin

### `main` del plugin

El `main` del plugin se ejecuta automáticamente al cargar (como
sub-llamada síncrona desde el caller). Convención: devolver `1` para
indicar "module loaded ok", `0` para "init failed".

```java
// plugin.vex
class PluginGreeter {
    public i64 valor() { return 12345; }
}

i32 main() {
    // Inicialización del plugin (registrar callbacks, etc.).
    return 1;
}
```

El `__module_init` (registrador de clases) corre ANTES de `main`. Así
las clases ya están disponibles cuando el caller hace `forName` tras
el `loadmodule`.

### IPC con el caller

El plugin se puede comunicar con el caller mediante mailbox
(msgsend/msgrecv) si están en el mismo proceso, o por callbacks vía
reflexión:

```java
// caller.vex
loadmodule("plugin.velb");
i64 cls = forName("Plugin");
i64 obj = newInstance(cls);
i64 setup = getMethod(cls, "setup");
invoke(setup, obj, host_pid); // pasamos el PID del caller al plugin
```

```java
// plugin.vex
class Plugin {
    public i64 host_pid;
    public void setup(i64 host) {
        this.host_pid = host;
    }
    public void notify(string msg) {
        msgsend(this.host_pid, msg);
    }
}
```

## Detalles internos

### Opcodes bytecode

| Opcode | Mnemónico | Uso |
|:---|:---|:---|
| `0x00 0x59` | `loadmod r_addr, r_len` | Carga el `.velb` desde el path en `vm_mem[r_addr, r_len]`. Invoca automáticamente el `main` del plugin. Devuelve `init_pc` en R0 (0 si falla). |
| `0x00 0x6D` | `unloadmod r_addr, r_len` | Descarga el módulo por path. R0 = 1 si OK, 0 si no se encontró. |

### API C++

```cpp
// Carga dinámica con rebase transparente.
uint64_t Loader::load_module_dynamic(VM &vm, vector<uint8_t> bytes);

// Variante que también registra source_path para unload posterior.
uint64_t Loader::load_module_dynamic_with_path(VM &vm, vector<uint8_t> bytes,
string source_path);

// Descarga por path.
bool Loader::unload_module_dynamic(VM &vm, string path);
```

### Rebase transparente

Cuando dos `.velb` declaran código en la misma VA, el loader detecta
el solape y re-asigna el segundo a `next_dyn_base` (por defecto
`0x80000000+`). Reescribe los símbolos `@Absolute("code.X")` mediante
la tabla de relocations del `.velb` para apuntar al nuevo offset.

Esto permite cargar plugins compilados sin coordinar VAs manualmente.

## Limitaciones y consideraciones

### Plugins de código confiable

Por defecto, un plugin hereda CAPS = ALL del caller. Si cargas un
plugin no confiable, restringir siempre vía `--vex-caps`:

```bash
# NO recomendado con plugins no confiables:
vm --run main.velb # el plugin hereda ALL caps

# Recomendado:
vm --run main.velb --vex-caps "loadmod=/var/safe_plugins/,fs:read=/tmp"
# El main solo puede cargar plugins de /var/safe_plugins/, y solo leer
# de /tmp. Cualquier plugin cargado hereda esas mismas restricciones.
```

### Memoria liberada al unload

Al hacer `unloadmodule`, el bytecode y las secciones de datos del
plugin son liberados de `vm_mem` (el rango de VA queda disponible para
futuros loads). Pero:

- Los objetos vivos creados por el plugin SOBREVIVEN en el `gc_heap` del proceso.
- Sus `class_ptr` siguen apuntando a `ClassInfo*` del ClassRegistry global.
- La VM no garantiza cleanup de recursos del plugin (file handles, sockets, threads).

Es responsabilidad del plugin liberar sus recursos en respuesta a una
señal del caller antes del `unloadmodule`. Patrón:

```java
// caller envía "shutdown" al plugin
i64 method = getMethod(cls, "shutdown");
invoke(method, plugin_obj);
unloadmodule("plugin.velb");
```

### `main` reentrante

Si se llama `loadmodule` con el mismo path dos veces sin un
`unloadmodule` intermedio, el módulo se carga dos veces (dos rangos
de VA distintos). El `__module_init` se ejecuta también dos veces, y
el ClassRegistry tendrá dos entradas con el mismo nombre (la segunda
sobrescribe la primera).

Convención recomendada: hacer `unloadmodule` antes de re-`loadmodule`
para hot-reload.

## Verificación

Tests integradores en `tests/bugs/m_dyn_test/`:

- `plugin.vex` — módulo trivial con clase `DynModule`.
- `main.vex` — ciclo completo: load + use + unload + reload + use.

```bash
bash tests/vex/test_vex_e2e.sh | grep "M.dyn"
```
