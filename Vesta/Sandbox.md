# Sandbox basado en capabilities

Vesta tiene un modelo de seguridad **capability-based** completamente
configurable que permite restringir qué operaciones puede ejecutar
cada módulo. Cero overhead en el modo por defecto; protege contra
plugins no confiables sin afectar al rendimiento del código confiable.

Documentos relacionados:

- [[Modulos]] — sistema estático de módulos.
- [[CargaDinamica]] — `loadmodule` (transitividad de caps al plugin).
- [[FFI]] — caps `FFI_CALL`/`FFI_OPEN` sobre llamadas nativas.

## Resumen rápido

```bash
# Modo por defecto (sin sandbox): ALL caps concedidas, cero overhead.
vm --run programa.velb

# Sandbox total: nada funciona excepto computación pura e IPC local.
vm --run programa.velb --vx-caps "none"

# Sandbox fino: solo lo necesario.
vm --run programa.velb --vx-caps "fs:read=/tmp,net=localhost:8080,ffi:call=kernel32.dll"
```

## Modelo de capabilities

El sandbox tiene **10 capabilities** ortogonales y **4 whitelists**
granulares. Cada módulo cargado lleva su conjunto de caps y
whitelists. El runtime las consulta antes de cada operación peligrosa:

| Cap | Cubre | Refinamiento |
|:---|:---|:---|
| `fs:read` | `vio_fopen/fread`, lectura del filesystem | Whitelist de paths |
| `fs:write` | `vio_fwrite/fclose`, escritura del filesystem | Whitelist de paths |
| `net` | Sockets, conexiones TCP/UDP | Whitelist de host:port |
| `ffi:call` | `CALLN` a DLL/SO ya cargada | Whitelist de DLLs |
| `ffi:open` | `ffi_open`, `dlopen`, `LoadLibrary` | Whitelist de DLLs (compartida) |
| `spawn` | `spawn`, `spawn here`, `spawn on(N)` | — |
| `distrib` | `rspawn`, `msgsend` cross-node, `memsync` | — |
| `classreg` | `addadvice` (hook AOP cross-módulo) | — |
| `mem:host` | LOAD/STORE con host_ptr (`movh`, `fload`, `fstore`) | Whitelist de rangos de memoria |
| `loadmod` | `loadmodule`, `unloadmodule` | Whitelist de paths (compartida con fs) |

### Configuraciones por defecto

| Modo | Bits | Whitelists | Comportamiento |
|:---|:---|:---|:---|
| Sin `--vx-caps` | `ALL` (0xFFFFFFFF) | vacías | **Sin sandbox**: cero overhead, todo permitido. |
| `--vx-caps ""` | `ALL` | vacías | Idéntico a sin flag. |
| `--vx-caps "all"` | `ALL` | vacías | Idéntico, explícito. |
| `--vx-caps "none"` | `NONE` (0) | vacías | Sandbox total: bytecode puro e IPC mailbox. |

## Sintaxis del string de caps

```text
spec ::= 'all' | 'none' | cap_list
cap_list ::= cap_item (',' cap_item)*
cap_item ::= cap_name ('=' arg_list)?
arg_list ::= arg (';' arg)*
arg ::= path | host_port | dll_name | mem_range
mem_range ::= HEX '-' HEX
```

### Caps simples

```bash
--vx-caps "fs:read,net"
# Concede solo FS_READ y NET; el resto queda denegado.
```

### Caps con whitelists

```bash
--vx-caps "fs:read=/tmp,net=localhost:8080,ffi:call=kernel32.dll;user32.dll"
# Concede:
# FS_READ con whitelist de paths = ["/tmp"]
# NET con whitelist de hosts = ["localhost:8080"]
# FFI_CALL con whitelist de DLLs = ["kernel32.dll", "user32.dll"]
# (los múltiples valores se separan con ';' dentro del =)
```

### Rangos de memoria

```bash
--vx-caps "mem:host=0x10000000-0x20000000"
# Cap MEM_HOST denegada (no concedida como bool), pero el rango
# [0x10000000, 0x20000000) queda whitelisted. Solo los accesos a
# host_ptrs en ese rango son permitidos.
```

### Combinación

```bash
--vx-caps "fs:read=/safe/data,fs:write=/tmp,net,ffi:call=kernel32.dll,loadmod=/var/plugins"
```

`fs:read` solo de `/safe/data`; `fs:write` solo a `/tmp`; `net` sin
restricción de host; `ffi:call` solo a `kernel32.dll`; `loadmod` solo
desde `/var/plugins`; el resto denegado.

## Whitelists granulares

### Whitelist de paths (`fs:read` / `fs:write` / `loadmod`)

Compartida entre `fs:read`, `fs:write` y `loadmod`. Match por PREFIJO
con normalización de separadores:

```text
--vx-caps "fs:read=/tmp"
# Accesos permitidos:
# /tmp/foo.txt -> OK (coincide con el prefijo /tmp)
# /tmp/subdir/x.txt -> OK
# /var/log/sys.log -> denegado
# /tmpfoo.txt -> OK (coincide con el prefijo /tmp sin trailing slash)
```

**Recomendación**: usar trailing slash explícito para evitar coincidencias
inesperadas:

```text
--vx-caps "fs:read=/tmp/"
# Ahora /tmpfoo.txt NO coincide (no empieza con /tmp/).
```

Los separadores `\` y `/` se normalizan a `/` para una comparación
uniforme entre Windows y POSIX.

### Whitelist de hosts (`net`)

Match exacto contra `host` o `host:port`:

```text
--vx-caps "net=localhost,api.example.com:8080"
# Permitidos:
# localhost -> OK
# localhost:1234 -> OK (host coincide, puerto libre)
# api.example.com:8080 -> OK (match exacto)
# api.example.com -> OK (host coincide, puerto libre)
# api.example.com:9000 -> denegado (host coincide, puerto no whitelisted)
```

Cuando la entry tiene `:port`, se exige match exacto también del puerto.

### Whitelist de DLLs (`ffi:call` / `ffi:open`)

Match exacto contra el nombre de DLL/SO (case-insensitive en Windows):

```text
--vx-caps "ffi:call=kernel32.dll;user32.dll"
# CALLN a kernel32.dll:XXX o user32.dll:YYY -> OK.
# CALLN a custom.dll:ZZZ -> denegado.
```

### Rangos de memoria (`mem:host`)

Lista de rangos `[start, end)`. Cada acceso host_ptr se valida contra
todos los rangos; si ninguno contiene la dirección, el acceso se
deniega.

```text
--vx-caps "mem:host=0x10000000-0x20000000,0x80000000-0x90000000"
# Solo los accesos en esos dos rangos pasan.
```

Importante: si la cap `mem:host` está concedida como bool (sin `=`),
los rangos se ignoran (cap = todo permitido). Si está denegada pero
los rangos no están vacíos, solo esos rangos son permitidos.

## Comportamiento ante denegación

Cuando un módulo intenta una operación sin la cap correspondiente, el
runtime lanza un `FatalError` capturable mediante `try/catch`:

```java
i32 main() {
    try {
        println("intentando llamar FFI sin la cap");
    } catch (FatalError e) {
        if (e.kind == FATAL_ILLEGAL_INSTRUCTION) {
            return -1;
        }
        throw; // otro tipo de error fatal
    }
    return 0;
}
```

Mensajes típicos:

```text
CALLN: el modulo actual no tiene la capability 'ffi:call' (sandbox)
```

```text
dlopen: DLL 'untrusted.dll' no esta en la whitelist (sandbox)
```

```text
loadmodule: path '/etc/passwd' fuera de la whitelist (sandbox)
```

## Transitividad: los plugins heredan caps

Cuando un módulo carga otro mediante `loadmodule(path)`, el módulo
cargado HEREDA las caps del caller con AND-mask:

```bash
# El main tiene fs:read=/tmp + loadmod (sin classreg).
vm --run main.velb --vx-caps "fs:read=/tmp,loadmod"
```

```java
// main.vx
i32 main() {
    loadmodule("plugin.velb");
    // El plugin recién cargado tiene caps = ALL AND (fs:read=/tmp + loadmod)
    // = fs:read=/tmp + loadmod
    // El plugin NO puede hacer addadvice, spawn, distrib ni nada más
    // que lo que tiene el main.
    return 0;
}
```

**Garantía de no-escalation**: un plugin restringido nunca puede cargar
otro plugin con MÁS caps que las suyas. AND-mask es matemáticamente
monotónico decreciente.

## Aplicación automática al módulo principal

```bash
vm --run main.velb --vx-caps "..."
# main.velb recibe esas caps inmediatamente tras load_executable.
# Cualquier dependencia cargada vía loadmodule las hereda con AND-mask.
```

Output cuando hay caps explícitas:

```text
[sandbox] modulo principal con caps: fs:read=/tmp,net=localhost:8080,ffi:call=kernel32.dll
```

## Cero overhead en el modo por defecto

Si NO se pasa `--vx-caps`, el módulo principal tiene
`caps.bits = 0xFFFFFFFF` (ALL) y `whitelists` vacías. El predicado
`Caps::unrestricted()` devuelve `true`.

Cada cap check del runtime se reduce a:

```cpp
if (!loader_public.check_cap_at_pc(rip, REQUIRED_MASK)) {
    throw_fatalf(vm, FATAL_ILLEGAL_INSTRUCTION, "...");
    return;
}
```

Que expande a:

```cpp
// fast path: el executable encontrado tiene caps == ALL.
if ((exe->caps.bits & REQUIRED_MASK) == REQUIRED_MASK) {
    // pass
}
```

Un AND bitwise + un compare + un branch predicted-not-taken. El CPU
lo resuelve always-not-taken en aproximadamente 1 ns. La suite de
tests completa pasa en modo por defecto sin regresión medible
respecto al modo sin sandbox.

## Operaciones con cap check

| Operación | Cap requerida | Whitelist consultada |
|:---|:---|:---|
| `CALLN` (FFI a plugin/DLL) | `ffi:call` | — (futuro: dll_whitelist) |
| `dlopen` / `LoadLibrary` | `ffi:open` | `dll_whitelist` |
| `callni` (FFI dinámico) | `ffi:call` | — |
| `addadvice` (AOP hook) | `classreg` | — |
| `rspawn` | `distrib` | — |
| `msgsend` remoto (PID con bit63=1) | `distrib` | — |
| `spawn`, `spawn here`, `spawn on` | `spawn` | — |
| `loadmodule` | `loadmod` | `path_whitelist` |
| `unloadmodule` | `loadmod` | — |

Las caps `fs:read`, `fs:write`, `net` y `mem:host` están documentadas
en la API pero no están aún wireadas en operaciones específicas; ese
trabajo queda pendiente por ser más invasivo (cambios al runtime de
FFI nativo). Quienes necesiten estas caps pueden combinar:

- `fs:read` + whitelist restrictiva de DLLs (sin `kernel32.dll` el
 plugin no puede invocar `CreateFileA`).
- `net` + whitelist de DLLs (sin acceso a `Ws2_32.dll`, sin sockets).
- `mem:host` requiere validación runtime de cada LOAD/STORE host_ptr.
 Como mitigación intermedia se puede aislar el plugin en un
 ProcessVM con `gc_heap` separado, donde no comparte vm_mem ni heap
 con el host (trabajo de capas adicionales del sandbox).

## Modelo de defensa por capas

| Capa | Cubre | Estado |
|:---|:---|:---|
| Capability-based in-process (este documento) | Bloqueo de builtins peligrosos a nivel VM. | **Implementado**. |
| ProcessVM aislado | `gc_heap` y `vm_mem` separado por plugin. | Diseño preparado; pendiente de implementación. |
| Aislamiento OS-level | `fork` + `seccomp`/`AppContainer` + chroot. | Diseño preparado; pendiente de implementación. |

La combinación es **defense-in-depth**: cada capa cubre un modelo de
amenaza distinto.

| Amenaza | Mitigación |
|:---|:---|
| Plugin invoca un builtin peligroso de la VM | Capability-based (cap denegada). |
| Plugin hace deref de host_ptr del padre | ProcessVM aislado (`gc_heap` separado). |
| Plugin con `ffi:call` invoca una DLL maliciosa que hace syscalls arbitrarias | Aislamiento OS-level (kernel-level). |
| Plugin crashea con access violation / segfault | Recuperación VEH + setjmp/longjmp (ya implementada: captura en `try/catch`). |
| Plugin consume CPU/memoria sin límite (DoS) | Aislamiento OS-level (`setrlimit` / Job Object). |

## Ejemplos de uso

### Scripting de usuarios cooperativos

```bash
# El usuario carga scripts en su IDE. Confiamos en el código pero
# queremos auditoría de caps y lifecycle controlado.
vm --run ide.velb --vx-caps "fs:read=/home/usuario/scripts,fs:write=/tmp,loadmod=/home/usuario/scripts"
```

### Plugin marketplace (más paranoico)

```bash
# Plugin descargado de un marketplace. No confiamos en el código.
# Solo permitimos computación e IPC con el host.
vm --run app.velb --vx-caps "fs:read=/safe/configs,loadmod=/var/lib/plugins"
# El plugin no puede:
# - Llamar a FFI (ffi:call denegada)
# - Abrir sockets (net denegada)
# - Crear procesos (spawn denegada)
# - Hookear AOP (classreg denegada)
# - Cargar otros plugins fuera de /var/lib/plugins (loadmod con path-whitelist)
```

### CTF u online REPL adversarial

Aquí el capability-based in-process por sí solo no es suficiente.
Requiere aislamiento OS-level. Como mitigación intermedia, cada
submission puede correr en un proceso VM separado spawneado mediante
`rspawn` (la VM ya soporta multi-proceso) con sus propios límites de
recursos a nivel OS.

## Verificación

Tests integradores en `tests/bugs/m_sandbox_test/`:

```bash
bash tests/vx/test_vx_e2e.sh | grep "sandbox"
```

## API C++ interna

Para extensiones del runtime que necesiten consultar caps:

```cpp
#include "loader/sandbox.h"
#include "loader/loader.h"

// En el handler de un opcode nuevo:
void exec_instr_mi_op(ProcessVM *vm, const DecodedInstr &instr) {
    // Chequeo simple por cap bool.
    if (!vm->scheduler.vm_reference.loader_public
    .check_cap_at_pc(vm->registers.rip.qword(), loader::Caps::FFI_CALL)) {
        throw_fatalf(vm, FATAL_ILLEGAL_INSTRUCTION,
        "mi_op: el modulo actual no tiene 'ffi:call' (sandbox)");
        return;
    }
    // ... operación ...
}

// Chequeo con consulta de whitelist:
const loader::Executable *exe = vm->scheduler.vm_reference.loader_public
.find_module_at_pc(vm->registers.rip.qword());
if (exe != nullptr) {
    if (!exe->caps.has(loader::Caps::FFI_OPEN)) {
        throw_fatal(...);
        return;
    }
    if (!exe->caps.dll_allowed("mi_lib.dll")) {
        throw_fatal(...);
        return;
    }
}
```

API pública de `loader::Caps`:

```cpp
struct Caps {
    uint32_t bits; // bitmask de caps
    CapWhitelists wl; // dll_whitelist, path_whitelist, host_whitelist, mem_ranges

    bool unrestricted() const noexcept; // bits==ALL && wl.empty()
    bool has(uint32_t required) const noexcept;
    bool dll_allowed(string name) const noexcept;
    bool path_allowed(string path) const noexcept;
    bool host_allowed(string hostport) const noexcept;
    bool mem_addr_allowed(uint64_t ptr) const noexcept;
    void intersect(uint32_t mask) noexcept; // AND-mask (no-escalation)
};
```

Parsing y serialización:

```cpp
Caps caps = loader::parse_caps("fs:read=/tmp,net=localhost:8080");
string s = loader::caps_to_string(caps); // formato canónico
const char *name = loader::cap_name(Caps::FFI_CALL); // "ffi:call"
```
