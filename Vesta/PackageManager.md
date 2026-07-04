# Manejador de paquetes (`vm pkg`)

`vm pkg` es el manejador de paquetes y dependencias de VestaVM.
Disenado con **anti-malware by design**: blindado contra los vectores
de ataque historicos de `npm`, `pip` y `cargo` desde el primer dia.

Este manual cubre:

1. Como **crear un paquete** propio.
2. Como **distribuirlo** (mediante un repositorio git como GitHub,
   GitLab, Gitea o Codeberg -- modelo federado, sin registry central).
3. Como **consumir** paquetes de otros.
4. Como **firmar y verificar** integridad y autoria.

## Por que un manejador propio

| Problema en npm/pip/cargo | Solucion en VestaVM |
| :--- | :--- |
| Scripts `postinstall` ejecutan codigo arbitrario al instalar | Paquetes son solo codigo Vesta + datos. Cero ejecucion en install. |
| `^1.2` floating sin lock fuerte | `vex.lock` obligatorio con `sha256` por dep. Sin sha256, falla cerrado. |
| Typosquatting (`raqquests` vs `requests`) | Trust pins por glob: `[trust] "@vesta/*" = "kpub1:..."`. |
| Cualquier paquete puede leer `/etc/passwd` | Capabilities declaradas por paquete + grant explicito. Default deny-all. |
| Maintainer takeover silente | Author fingerprint check: cambio entre versiones produce error. |
| Transitive dep explosion anonima | `vm pkg add` muestra TODOS los autores transitivos antes de instalar. |
| Build scripts no sandboxed | Comptime macros corren en `ComptimeRuntime` con sandbox cap-restricted. |

## Conceptos clave

### Manifest: `vex.toml` o `vex.json`

Cada proyecto y cada paquete tiene un manifest. Formato dual:

- `vex.toml`: canonico humano (recomendado).
- `vex.json`: equivalente machine-friendly para tooling.

El parser detecta el formato por extension. Si ambos coexisten en el
mismo proyecto, error explicito. Conversion entre ambos con
`vm pkg convert toml` o `vm pkg convert json`.

### Lockfile: `vex.lock`

Auto-generado por `vm pkg`. **Debe commitearse a git**. Contiene los
hashes `sha256` EXACTOS de cada dep resuelta y el commit/rev al
momento de la resolucion. Cualquier intento de instalar contenido
que no matchee estos hashes falla.

### Ubicaciones

```text
proyecto/
  vex.toml                          # manifest
  vex.lock                          # lockfile (commit-able)
  .gitignore                        # auto-generado por vm pkg init
  vex_modules/                      # deps locales (estilo node_modules)
  src/                              # tu codigo

$VEX_HOME/                          # global per-usuario
  cache/<sha256>/                   # cache de tarballs descargados
  packages/<name>/<version>/        # deps instaladas con install = "global"
  keys/                             # claves publicas y privadas
  trust.toml                        # pins de autores confiables
```

**Resolucion de `$VEX_HOME`** (en orden):

1. Variable de entorno `VEX_HOME` (override explicito).
2. Per-usuario: `%APPDATA%\Vesta\` en Windows, `~/.vesta/` en POSIX.
3. System-wide cuando la VM se instala como administrador.

### `vex_modules/` es read-only

`vex_modules/` recibe el contenido descargado y verificado. Cualquier
modificacion local rompe el sha256 del lockfile y `vm pkg verify`
falla. **No hay forma de meter codigo "extra" post-install** --
diferencia critica con `node_modules` donde es comun parchear in-place.

Para hacer fork local de una dep: declarar en `vex.toml` con
`path = "../mi-fork"` (sin sha256/firma, asume confianza dev).

## Crear un paquete propio

### Inicializar el paquete

```bash
mkdir mi-paquete && cd mi-paquete
vm pkg init                         # interactivo: nombre, version, autor
```

Te pregunta:

- **Nombre del paquete**: usar formato `@autor/nombre` para qualified
  (recomendado para evitar colisiones), o nombre plano si es interno.
- **Version**: semver `MAJOR.MINOR.PATCH`.
- **License**: SPDX identifier (MIT, Apache-2.0, GPL-3.0, etc.).
- **Autor**: tu nombre o handle.

Resultado: `vex.toml` + `.gitignore` apropiado.

Para inicializar en JSON en vez de TOML:

```bash
vm pkg init --json
```

### Estructura recomendada

```text
mi-paquete/
  vex.toml                # manifest
  vex.lock                # lockfile (solo si tu paquete consume otras deps)
  README.md               # documentacion
  LICENSE                 # texto de la license
  src/
    lib.vx               # entry point publico del paquete
    internal/
      helpers.vx         # codigo no-publico
  tests/
    test_lib.vx
  examples/
    demo.vx
```

El **entry point convencional** es `src/lib.vx`. Cuando alguien hace
`import "@autor/mi-paquete";`, ese fichero (y los que reexporte) es lo
visible.

### Declarar la API publica

```vex
// src/lib.vx
import "src/internal/helpers";       // privado por defecto

// Reexportar simbolos publicos:
public import "src/internal/helpers" only Counter, Result;

// O definir simbolos publicos directamente aqui:
public fn version() -> string { return "0.1.0"; }
```

Sin `public`, los simbolos son privados al paquete y no son visibles
para quien lo consuma. Ver `Modulos.md` para detalles del sistema de
modulos y reexport.

### Declarar capabilities

Si tu paquete necesita acceder a recursos, declaralo en el manifest:

```toml
[capabilities]
declared = ["fs:read=/tmp", "net=api.foo.com:443"]
```

El consumer DEBE dar grant explicito a cada cap para que tu paquete
pueda usarlas. Si NO declaras una cap pero la usas en runtime, el
sandbox aborta.

Para libs low-level (FFI, kernel calls, drivers):

```toml
[capabilities]
unsafe = true
```

`unsafe = true` bypasa el sandbox completo. El consumer tiene que
poner `trust_unsafe = true` explicito en su `[dependencies]` para
aceptar la lib.

### Verificar antes de distribuir

```bash
vm pkg verify                       # check estructura, hashes locales
vm --vex src/lib.vx -o /tmp/test   # confirma que compila standalone
```

## Firmar el paquete

Las firmas Ed25519 garantizan que el contenido que tus usuarios
descargan es exactamente el que tu publicaste, y que viene de ti.

### Generar tu par de claves

```bash
vm pkg keygen --out ~/.vesta/keys/private/mi_clave.pem
```

Output ejemplo:

```text
ok: generando par Ed25519: OK
[OK] clave privada guardada en ~/.vesta/keys/private/mi_clave.pem (chmod 600 en POSIX)
  fingerprint: kpub1:788c97df7d6dada3875ae024770f7860da349b988fc1fde4a65f0196e0053fc6
[INFO] comparte esta fingerprint con quienes vayan a verificar tus paquetes
```

**Guarda la clave privada de forma segura.** Si se compromete,
cualquiera puede publicar paquetes con tu nombre. Si la pierdes,
no puedes publicar updates firmados con tu identidad.

La **fingerprint publica** (`kpub1:...`) es lo que distribuyes a tus
usuarios para que pineen tu autoria.

### Anyadir tu propio pin (para tests)

```bash
vm pkg trust add kpub1:788c97df... --name mi_alias
vm pkg trust list                    # ver pins activos
vm pkg trust revoke kpub1:788c...    # marcar como revoked
```

## Distribuir el paquete

VestaVM **NO tiene registry central** (por diseno: anti-takeover y
federado). Se distribuye a traves de **cualquier repositorio git
publico**: GitHub, GitLab, Gitea, Codeberg, o tu propio servidor.

### Subir el paquete al repositorio git

Pasos tipicos (tu eliges el host):

```bash
# Inicializa el repo local
cd mi-paquete
git init
git add .
git commit -m "v0.1.0 inicial"

# Crea el remote en el host de tu eleccion (manualmente) y haz push:
git remote add origin <URL de tu repo>
git push -u origin main

# Crear un tag inmutable por version (RECOMENDADO):
git tag v0.1.0
git push --tags
```

**Recomendaciones para distribucion segura**:

- Usa **tags inmutables** (`v0.1.0`) en lugar de branches. Una vez
  publicado un tag, no muevas el commit al que apunta.
- **Firma los tags con git** (`git tag -s v0.1.0`) o, mejor aun,
  publica la fingerprint Vesta (`kpub1:...`) en tu README para que
  los consumidores la pineen.
- Si haces release nuevo con cambios incompatibles, **sube major**
  (`v1.0.0 -> v2.0.0`). El package manager respeta semver.

### Compartir tu fingerprint con los usuarios

Pon en el README de tu repo:

```markdown
## Verificar autoria

Esta libreria esta firmada con la clave Vesta:

    kpub1:788c97df7d6dada3875ae024770f7860da349b988fc1fde4a65f0196e0053fc6

Para pinear como autor confiable:

    vm pkg trust add kpub1:788c97df... --name mi_alias
```

Asi los consumidores pueden registrar tu fingerprint la primera vez
que consumen tu lib, y cualquier release futuro firmado por otra clave
sera rechazado automaticamente.

## Consumir paquetes

### Anyadir una dependencia

```bash
# Desde un repo git (tag inmutable, recomendado):
vm pkg add @autor/mi-paquete --git https://github.com/autor/mi-paquete --tag v0.1.0

# Desde un commit exacto (mas estricto, no se mueve nunca):
vm pkg add @autor/mi-paquete --git https://github.com/autor/mi-paquete --rev abc123de...

# Estilo Go (auto-resuelve el host como git URL):
vm pkg add github.com/autor/mi-paquete@v0.1.0

# Path local (desarrollo):
vm pkg add @me/utils --path ../utils-lib

# Dev-dependency (solo para tests, no se distribuye):
vm pkg add --dev @vesta/test
```

Cada `vm pkg add` muestra un **audit transitivo** antes de aplicar:

```text
Agregando @autor/mi-paquete v0.1.0...
  deps transitivas (3):
    @autor/mi-paquete v0.1.0
      autor: kpub1:788c... (no pinneado, primera vez)
      unsafe: false
      capabilities: fs:read=/tmp
    @vesta/io 2.0.1
      autor: kpub1:abc... (trusted via @vesta/*)
      unsafe: false
      capabilities: (ninguna)
    @vesta/strings 1.0.5
      autor: kpub1:abc... (trusted via @vesta/*)
      unsafe: false
      capabilities: (ninguna)

Continuar? [y/N]
```

Si algun paquete es `unsafe = true`, warning explicito:

```text
ATENCION: @vendor/driver declara UNSAFE.
  - Bypasara todo el sandbox.
  - Fingerprint autor: kpub1:... (no pinneado)
Anade --trust-unsafe para confirmar.
```

### Instalar segun el lockfile

```bash
vm pkg install                      # descarga TODO segun vex.lock
                                    # verifica sha256 y firmas
                                    # falla si algun hash no matchea
```

Esto descarga cada dep al `vex_modules/` del proyecto, validando que
el contenido recibido (incluido el commit exacto si es git) produce
el sha256 declarado en `vex.lock`. Si alguna no matchea, aborta sin
copiar nada -- detecta tampering y cambios silenciosos.

### Usar el paquete en tu codigo

```vex
// src/main.vx
import "@autor/mi-paquete";

fn main() -> i32 {
    let v = autor::mi_paquete::version();
    println("{v}");
    return 0;
}
```

Ver `Modulos.md` para sintaxis completa de `import`, `only`, alias.

### Reproducir un build de otra persona

```bash
git clone <repo del proyecto>
cd <repo>
vm pkg install              # descarga TODO segun vex.lock
                            # mismo bit-perfect que el del autor
```

El `vex.lock` commit-eado en su repo garantiza que descargas exactamente
lo mismo que ellos tenian.

## Verificacion e integridad

### Verificar lo instalado

```bash
vm pkg verify
```

Output ejemplo:

```text
Verificando integridad de paquetes
==================================
[OK] @vesta/io verificado
[OK] @vesta/buffer verificado
[OK] @autor/mi-paquete verificado
  ok: 3
  fail: 0
```

Recalcula el `sha256` del contenido de cada dep en `vex_modules/` y lo
compara con el declarado en `vex.lock`. Detecta tampering local,
cache corrupto o discrepancias post-merge en git.

### Auditoria de seguridad

```bash
vm pkg audit
```

Detecta:

- Paquetes con `unsafe = true` en la cadena transitiva.
- Paquetes sin firma o de autores no pinneados.
- **Cambios de autor** entre versiones (signal de posible takeover).
- **Cambios de `sha256`** con la misma version (signal de tampering).
- **Capabilities expandidas** vs la version anterior.

Output con severidades:

```text
Auditoria de dependencias
==========================
[CRITICAL] @some/lib  AUTHOR_CHANGED
  el autor cambio entre 1.2.0 (kpub1:abc...) y 1.2.1 (kpub1:xyz...)
  hint: posible takeover; verifica manualmente con el publisher original

[WARNING] @vendor/driver  UNSAFE
  el paquete se declara unsafe (acceso bajo nivel)
  hint: revisa el codigo manualmente antes de install

Resumen: 1 critical  1 warnings
```

Exit code `2` cuando hay findings critical; `0` cuando todo limpio.

## Esquemas

### Manifest TOML

```toml
[package]
name        = "@autor/mi-paquete"   # @org/name (qualified) o nombre plano
version     = "0.3.1"
authors     = ["David"]
license     = "MIT"
edition     = 1                     # version del schema del manifest

[capabilities]
declared = ["fs:read=/tmp", "net=api.foo.com:443"]
# unsafe = true                     # solo para libs low-level

[dependencies]
# git con tag (recomendado: inmutable si el autor no lo mueve):
"@vesta/buffer" = { git = "https://github.com/vesta/buffer", tag = "v1.2.3", sha256 = "..." }

# git con rev exacto (mas estricto):
"@vesta/http" = { git = "https://github.com/vesta/http", rev = "abc123de...", sha256 = "..." }

# git con branch (sigue HEAD; menos seguro, util en dev):
"@me/utils" = { git = "https://github.com/me/utils", branch = "develop", sha256 = "..." }

# path local (sin sha256/firma, asume confianza dev):
"@me/local" = { path = "../local-lib" }

# install global vs local (default = local en vex_modules/):
"@vesta/stdlib" = { version = "1.0", sha256 = "...", install = "global" }

# unsafe escape-hatch consumer-side:
"@vendor/driver" = { sha256 = "...", trust_unsafe = true, grant = ["ffi:call"] }

[trust]
"@vesta/*" = "kpub1:a1b2c3..."      # cualquier lib bajo @vesta/ firmada por esta key
"@me/*"    = "kpub1:d4e5f6..."
"exact/foo" = "kpub1:..."            # match exacto

[scripts]
build = "vm --vex src/main.vx -o build/app"
test  = "vm --run tests/run_all.velb"
fmt   = "vm fmt src/"

[dev-dependencies]
"@vesta/test" = { git = "https://github.com/vesta/test", tag = "v0.1.0", sha256 = "..." }
```

### Manifest JSON equivalente

```json
{
  "package": {
    "name": "@autor/mi-paquete",
    "version": "0.3.1",
    "authors": ["David"],
    "license": "MIT",
    "edition": 1
  },
  "capabilities": {
    "declared": ["fs:read=/tmp", "net=api.foo.com:443"]
  },
  "dependencies": {
    "@vesta/buffer": {
      "git": "https://github.com/vesta/buffer",
      "tag": "v1.2.3",
      "sha256": "..."
    }
  },
  "trust": {
    "@vesta/*": "kpub1:a1b2c3..."
  }
}
```

### Lockfile `vex.lock`

Auto-generado. No editar a mano:

```toml
[meta]
version       = "1"
generated_at  = "2026-05-26T12:34:56Z"
resolver      = "mvr-1"

[[package]]
name           = "@vesta/buffer"
version        = "1.2.3"
source         = "git+https://github.com/vesta/buffer.git"
resolved_rev   = "abc123de..."      # commit hash exacto resuelto
sha256         = "deadbeef..."      # hash del contenido descargado
author_fp      = "kpub1:..."
signature_alg  = "ed25519"
unsafe         = false
declared_caps  = ["fs:read=/tmp"]
deps           = ["@vesta/io"]
install_at     = "project"
```

## Algoritmo de resolucion

**Minimum Version Resolution (MVR)**, estilo Go modules. Deterministico,
sin SAT solver.

Reglas:

1. Cada dep declara una version semver minima (`^1.2` significa >= 1.2.0,
   < 2.0.0).
2. Si varias deps piden la misma lib con versiones distintas, se elige
   la MAYOR de las minimas (la unica que satisface a todas).
3. Si dos versiones de la misma lib se requieren en diferentes mayores
   (uno pide 1.x, otro pide 2.x con breaking changes), ambas conviven
   en el binario final con namespacing automatico.
4. Conflictos de tipos cross-version los reporta el type checker en
   compile time, no el package manager.

Especificadores semver soportados en `[dependencies]`:

- `"1.2.3"` -- exact.
- `"^1.2"` -- >= 1.2.0, < 2.0.0 (caret, default semver).
- `"~1.2.3"` -- >= 1.2.3, < 1.3.0 (tilde).
- `">=1.0"` -- mayor o igual.
- `">1.0"` -- mayor estricto.
- `"*"` o `"latest"` -- ultima disponible.

## Patrones tipicos

### Proyecto nuevo

```bash
mkdir mi-app && cd mi-app
vm pkg init
vm pkg add github.com/autor/lib@v1.0.0
vm pkg install
vm --vex src/main.vx -o build/app
```

### Lib local en desarrollo

```toml
# en mi-app/vex.toml
[dependencies]
"@me/utils" = { path = "../utils-lib" }
```

```bash
vm pkg install
```

Crea symlink/copia desde `../utils-lib`; cualquier edicion se refleja
inmediatamente.

### Auditar antes de aceptar un paquete dudoso

```bash
vm pkg add @sospechoso/foo --git https://github.com/sospechoso/foo --tag v1.0.0
# Lee el audit antes de confirmar con 'y'.
# Si algo huele mal, responde 'N'.
```

### Actualizar con cuidado

```bash
vm pkg update @vesta/buffer         # solo esa dep
vm pkg verify                       # check integridad post-update
vm pkg audit                        # signals de takeover/tampering
```

### Reproducir un build

```bash
git clone <repo>
cd <repo>
vm pkg install                      # exactamente lo que el autor tenia
```

## Anti-malware: detalles del modelo

### Lockfile con sha256 obligatorio

Toda dep no-local DEBE tener `sha256` en el lockfile. Si falta,
`vm pkg install` falla con error claro. Cualquier mismatch entre
contenido descargado y `sha256` declarado aborta el install
ANTES de copiar nada a `vex_modules/`.

### Capabilities + grant explicito

Cada paquete declara `[capabilities]` en su manifest. El consumer
SOLO da las caps que aparecen en `grant` de su `[dependencies]`.

**Regla critica**: `grant ⊆ declared`. El consumer puede dar MENOS
de lo que el paquete declara, pero NUNCA mas. Si el paquete intenta
usar una cap fuera de su grant, error runtime.

Cross-package, las caps se AND-en transitivamente: si el root no
tiene `fs:read`, ninguna dep transitiva la obtiene aunque el grant
directo la incluya.

### Firmas Ed25519

Cada paquete (excepto deps con `path = "..."`) debe estar firmado con
Ed25519. El fingerprint = SHA-256 del pubkey en 64 hex chars,
prefijado por `kpub1:`.

`vm pkg verify` rechaza paquetes con:

- Firma corrupta o ausente.
- Fingerprint que no matchea con `[trust]`.
- Cambio de autor entre versiones.

### No postinstall scripts en deps

A diferencia de npm, el manager NO ejecuta scripts del paquete
descargado. Si un paquete necesita "build steps":

- Codigo runtime regular: el consumer lo compila como cualquier
  modulo Vesta.
- Codigo comptime: usar `@Macro` que corre en `ComptimeRuntime` con
  sandbox cap-restricted. Ver `Metaprogramacion.md`.

Esto cierra el vector de ataque mas comun de npm.

### Sandbox transitivo

Las caps se AND-en cross-package automaticamente. Si un paquete
malicioso intenta cargar otro con mas caps que las suyas via
`loadmodule`, el loader rechaza. Ver `Sandbox.md`.

### Audit obligatorio en cada cambio

`vm pkg add` / `update` / `remove` muestra:

- Autores transitivos y sus trust status.
- Paquetes unsafe.
- Capabilities cambiadas.
- Cambios de autor.

El usuario confirma con `y` antes de aplicar.

### Cache global por hash

El cache global (`$VEX_HOME/cache/<sha256>/`) es content-addressed:
la clave es el hash. Cualquier corrupcion del cache se detecta al
re-verificar el hash en el siguiente `vm pkg install`.

## Referencia rapida de comandos

| Comando | Descripcion |
| :--- | :--- |
| `vm pkg init [--json]` | Crea `vex.toml` o `vex.json` + `.gitignore` |
| `vm pkg add <spec> [--dev]` | Agrega dep + actualiza lock + audit |
| `vm pkg remove <name>` | Quita dep + actualiza lock |
| `vm pkg install` | Descarga + verifica todo segun lock |
| `vm pkg update [name]` | Re-resuelve deps segun reglas semver |
| `vm pkg list [--tree]` | Lista deps directas o arbol completo |
| `vm pkg verify` | Check `sha256` + firmas vs `vex.lock` |
| `vm pkg audit` | Auditoria de seguridad transitiva |
| `vm pkg convert <toml\|json>` | Convierte formato del manifest |
| `vm pkg keygen [--out PATH]` | Genera par Ed25519 + fingerprint |
| `vm pkg trust list` | Lista pins de autores confiables |
| `vm pkg trust add <fp> [--name N]` | Pinea un autor |
| `vm pkg trust revoke <fp>` | Marca pin como revoked |
| `vm pkg run <script>` | Ejecuta entry de `[scripts]` |

## Ver tambien

- [Modulos.md](Modulos.md) -- sistema de modulos `import`.
- [Sandbox.md](Sandbox.md) -- modelo capability-based.
- [CargaDinamica.md](CargaDinamica.md) -- `loadmodule` y hot reload.
- [Metaprogramacion.md](Metaprogramacion.md) -- `@Macro` y `ComptimeRuntime`.
