# Plan Maestro de Reingeniería — Ecosistema Mire

**Versión:** 3.0 — plan definitivo por librería
**Fecha:** 2026-11-13
**Ámbito:** `ilogical/ kioto/ mire/ libs/ owl/ avenys/ wasm/ sqlite/` + paquetes satélite
**Estado:** propuesta para revisión en la reunión de mire-lang

> Este documento sustituye al plan v2. Todo lo marcado como **[verificado]** se comprobó leyendo el árbol real; lo marcado como **[reproducir]** es una inferencia estática que debe confirmarse con una ejecución antes de convertirse en tarea cerrada.

---

## Índice

- [0. Cómo leer este documento](#0-cómo-leer-este-documento)
- [1. Decisiones cerradas](#1-decisiones-cerradas)
- [2. Principios rectores](#2-principios-rectores)
- [3. Convención `mire::case`](#3-convención-mirecase)
- [4. Arquitectura objetivo](#4-arquitectura-objetivo)
- [5. Avenys — el compilador](#5-avenys--el-compilador)
- [6. Owl — gestor de proyectos](#6-owl--gestor-de-proyectos)
- [7. Mire — identidad de la librería](#7-mire--identidad-de-la-librería)
- [8. Ilogical — dueño de los codecs](#8-ilogical--dueño-de-los-codecs)
- [9. Kioto — librería estándar](#9-kioto--librería-estándar)
- [10. Paquetes satélite](#10-paquetes-satélite)
- [11. Sistema de tests](#11-sistema-de-tests)
- [12. Reset de repositorios](#12-reset-de-repositorios)
- [13. Diagnósticos, errores, warnings y ayuda](#13-diagnósticos-errores-warnings-y-ayuda)
- [14. Propuestas de sintaxis](#14-propuestas-de-sintaxis)
- [15. Fases de ejecución](#15-fases-de-ejecución)
- [16. Métricas de aceptación](#16-métricas-de-aceptación)
- [17. Riesgos](#17-riesgos)
- [18. Preguntas abiertas](#18-preguntas-abiertas)
- [Anexo A — Catálogo de códigos](#anexo-a--catálogo-de-códigos)
- [Anexo B — Checklist por release](#anexo-b--checklist-por-release)

---

## 0. Cómo leer este documento

Cada capítulo de librería responde a las mismas seis preguntas:

1. **Qué existe hoy** — inventario verificado, con rutas y LOC.
2. **Qué es stub o fósil** — y por qué, con evidencia.
3. **Qué se elimina** — no se implementa: desaparece del repo y del export.
4. **Qué se implementa** — con firma objetivo y criterio de aceptación.
5. **Qué tests nacen con ello** — un archivo por símbolo, con caso de éxito, límite y anomalía.
6. **Qué se migra** — compatibilidad, alias, ciclos de deprecación.

Convenciones de este documento:

| Marca | Significado |
|-------|-------------|
| **[verificado]** | Leído en el árbol el 2026-11-13. |
| **[reproducir]** | Inferencia estática; exige un comando de confirmación. |
| **P1…P8** | Principios rectores (§2). |
| **F0…F9** | Fases de ejecución (§15). |
| `→` | Renombre o migración. |
| `✗` | Se elimina. |

---

## 1. Decisiones cerradas

Estas decisiones vienen de la revisión del plan v2 y **no se vuelven a discutir** salvo que aparezca evidencia nueva.

| # | Decisión | Consecuencia |
|---|----------|--------------|
| D1 | **Runtime `base` es el default permanente y conservador.** No se retira cuando `experimental` se estabilice. | Ambos tiers usan PAL escrita en Mire y la misma API; `base` evita asm y optimizaciones agresivas, mientras `experimental` puede usarlas. `experimental` sigue siendo opt-in y se identifica en `status()` y `--version`. |
| D2 | **`mire::case` aplica solo a las librerías del ecosistema** (`kioto`, `mire`, `ilogical`, `owl`, `sqlite`, `wasm`, `sdl`). | El parser no se toca. `_` sigue siendo válido dentro de un identificador. Lo que cambia es el **namespace público**: cada separador lógico es `::`. |
| D3 | **`--link` / `-L`** para librerías del ecosistema y **`--hash` / `-H`** para el hash esperado. | Hay colisión con el `-L` actual (linker search dir); se resuelve en §5.2. |
| D4 | **`symbols/`** (inglés) como nombre del directorio de tests por símbolo. | `kioto/tests/async/symbols/mod.mr`. |
| D5 | **`ilogical` es el dueño total de JSON y TOML.** | `kioto` pierde `json` y `toml`; solo puede re-exportar si hace falta compatibilidad. |
| D6 | **`--bundle static` por defecto** para binarios de usuario; prefiere `.a` y cae atrás a `.so` con aviso. | `--bundle static-strict` prohíbe el fallback y es obligatorio para releases autocontenidos; `--bundle dynamic` pide `.so` explícitamente. |
| D7 | **Se borran los `test_*.mire` de `sqlite/code/`.** | Lo que valga la pena se reescribe en `sqlite/tests/`. |
| D8 | **Se mantienen `load`, `load!` y `use!` obligatorio.** | No se unifican. La resolución por símbolo no cambia la sintaxis de carga. |

### 1.1 Corrección obligatoria sobre D3: colisión de `-L`

**[verificado]** `avenys/src/cli/options.rs:220` ya usa `-L | --link` para *directorios de búsqueda del linker nativo*. La decisión D3 le da `-L` a las librerías del ecosistema. Propuesta de reparto definitivo:

| Flag | Significado | Cambio |
|------|-------------|--------|
| `-L, --link name=path` | Librería del ecosistema (repetible) | **Nuevo uso de `-L`** |
| `-H, --hash sha256:<hex>` | Hash esperado de la librería anterior | Nuevo |
| `-l, --link-lib <name>` | Librería nativa a enlazar | Sin cambio |
| `--link-dir <path>` | Directorio de búsqueda del linker nativo | **Reemplaza al viejo `-L`** |
| `--config <file>` | — | **Eliminado** |

Ventana de compatibilidad (un release):

- La regla se aplica igual a `-L` y `--link`, que continúan siendo alias durante toda la transición.
- Si el argumento contiene `=`, se interpreta como librería del ecosistema.
- Si no contiene `=` y es un directorio existente, se emite `warning[W0061 legacy_link_flag]` con fix-it a `--link-dir` y se trata como directorio nativo.
- Si no contiene `=` ni identifica un directorio existente, se emite `E0703 invalid_value` con las dos interpretaciones posibles.
- En 5.0, el caso sin `=` es error `E0703 invalid_value` para ambos alias.

Esto permite que la sintaxis pedida (`mire build --link kioto=... -H sha256:...`) funcione desde el primer día sin romper scripts viejos de golpe.

---

## 2. Principios rectores

| # | Principio | Regla operativa | Verificación |
|---|-----------|-----------------|--------------|
| P1 | **Cero stubs** | Si algo no se va a implementar, **no se escribe el archivo**. Se anota en este plan y, cuando se implemente, nace con su test. | `find . -name '*.m*' -size -120c` no debe devolver módulos exportados. |
| P2 | **Cero fósiles** | Código muerto, duplicado o que usa strings como transporte de datos estructurados se elimina o se delega a un parser real. | Revisión por PR + grep de `substr`/`index` sobre TOML/JSON/argv. |
| P3 | **`mire::case`** | El namespace público usa `::` en cada separador lógico. | Auditoría de `[exports]` y de firmas `pub fn`. |
| P4 | **Todo símbolo público se declara en tests** | M1 mide presencia mediante `@[covers(...)]`; M2 mide los casos de contrato (éxito, límite y anomalía) o exenciones vigentes. | M1=100 % y M2≥90 % al cierre (§11.7); nunca se usa cobertura de líneas. |
| P5 | **Tests limpios** | Ningún test escribe en la raíz del repo. Cada suite usa `fixtures/tmp/` y limpia. | `git status --porcelain` vacío tras `owl test`. |
| P6 | **Runtime Mire, ABI externo solo en transición** | El compilador no incorpora objetos C. Durante F4 puede enlazar `libmire_rt.a` externo; el estado final implementa el mismo API PAL en Mire. | Estado y gate de F4/F9 en §4 y §15. |
| P7 | **Repos limpios** | Solo fuente, docs, tests y scripts. Artefactos fuera del árbol o en `bin/` ignorado. | Matriz §12. |
| P8 | **Diagnósticos primero** | Un error sin código, sin span y sin ayuda es un bug de primera clase, no un detalle cosmético. | §13 y métricas §16. |

### 2.1 Definiciones

- **Stub:** archivo o símbolo que existe, se exporta, y no hace lo que su nombre promete. También cuenta el que solo tiene un comentario.
- **Fósil:** código que ya no se ejecuta, que duplica otro, o que resuelve un formato estructurado con manipulación de strings.
- **Símbolo público:** todo `pub fn`, `pub struct`, `pub enum` alcanzable desde el `[exports]` del paquete.
- **Test unitario por símbolo:** un archivo que importa el paquete como consumidor externo y ejerce un símbolo con casos medibles.

---

## 3. Convención `mire::case`

### 3.1 La regla, en una frase

> **El namespace es `::`; el identificador puede seguir usando `_`.**

Eso significa:

- `kioto::math::int::abs` — correcto.
- `i64::input`, `str::from::i64`, `vec::push::str` — correcto.
- `kioto_math_int_abs` — **incorrecto** como símbolo público.
- `my_helper_fn` — correcto: es un identificador, no un namespace.
- `_externs` — correcto: es un archivo interno, no se exporta.

### 3.2 Por qué el parser no se toca

El parser ya acepta `::` en rutas y `_` en identificadores. `mire::case` es una **convención de biblioteca**, no una regla gramatical. Forzarla en el parser rompería código de usuario y no aporta nada: el problema que se quiere resolver es que las librerías del ecosistema expongan una superficie predecible.

### 3.3 Estado real: kioto ya está cerca

**[verificado]** `kioto/modules/strings/mod.mr` ya usa el patrón correcto:

```mire
pub fn from: () {
    pub fn i64: (value :i64) :str { return rt_i64_to_string(value) }
    pub fn bool: (value :bool) :str { ... }
    pub fn f64: (value :f64) :str { ... }
}

pub fn to: () {
    pub fn i64: (value :&str) :i64 { return rt_string_to_i64(*value) }
}

pub fn starts: () { pub fn with: (s :&str, prefix :&str) :bool { ... } }
pub fn ends:   () { pub fn with: (s :&str, suffix :&str) :bool { ... } }
```

Ese es exactamente el estilo objetivo. El trabajo es **extenderlo**, no inventarlo.

### 3.4 Renombres concretos por librería

| Librería | Hoy | `mire::case` | Nota |
|----------|-----|--------------|------|
| kioto::strings | `char_at` | `char::at` | |
| kioto::strings | `from_byte` | `from::byte` | |
| kioto::strings | `index` | `index::of` | Devuelve `i64`; ver §9.2 |
| kioto::strings | `to::i64` | `to::i64` (firma nueva) | `i64` → `result[i64 str]` |
| kioto::net | `PAL_SOCKET_TCP()` | `socket::kind::tcp()` | Constantes como funciones |
| kioto::net | `PAL_SOCKET_UDP()` | `socket::kind::udp()` | |
| kioto::cli | `parse` | `args::parse` | Y devolver struct tipada, no `map[str str]` |
| kioto::env | `get` / `cwd` | `var::get` / `dir::cwd` | |
| kioto::math::int | `abs` | `int::abs` | Ya está en namespace |
| owl::util | `read_line_trim` | `read::line::trim` | Interno |
| owl::util | `toml_get` | `toml::get` | **Se elimina igualmente** (§6.4) |
| owl::util | `json_get_str` | `json::get::str` | **Se elimina** (§6.4) |

### 3.5 El ejemplo canónico completo

```mire
load mire::std

set AlphaI64 = i64::input("Write any integer number: ")

pub fn i64: () {
    pub fn input: (msg :&str) :i64 {
        while true {
            use std::print(msg)
            set line = std::read_line()
            match str::to::i64(line) {
                Result.Ok(v)  { return v }
                Result.Err(e) { use std::eprintln(e.message) }
            }
        }
        return 0
    }
}
```

Requisitos para que esto compile:

1. `mire::std` debe exponer `print`, `read_line`, `eprintln`.
2. `str::to::i64` debe devolver `result[i64 Error]`.
3. `Error` debe tener `.message`.
4. `i64::input` debe ser alcanzable como `i64::input` (namespace + símbolo).

Los cuatro son parte del plan (§7 y §13.16).

---

## 4. Arquitectura objetivo

```
┌──────────────────────────────────────────────────────────────────────┐
│                              USUARIO                                 │
│   mire build --release --name "kioto-fork" -L kioto=... -H sha256:…  │
└───────────────────────────────┬──────────────────────────────────────┘
                                │
                    ┌───────────▼───────────┐
                    │        OWL            │
                    │  ┌─────────────────┐  │
                    │  │ compiler/       │  │  resuelve, produce CompileRequest
                    │  │ self/           │  │  config, lock, cache, registry
                    │  └─────────────────┘  │
                    └───────────┬───────────┘
                                │ protobuf (frame length-prefixed)
                    ┌───────────▼───────────┐
                    │       AVENYS          │
                    │  lexer → parser →     │
                    │  typeck → MSS → MIR → │
                    │  LLVM → link          │
                    └───────────┬───────────┘
                                │
              ┌─────────────────┼─────────────────┐
              │                 │                 │
     ┌────────▼───────┐ ┌───────▼───────┐ ┌───────▼────────┐
     │ runtime base   │ │ PAL Mire      │ │ deps nativas   │
     │ lib externa C  │ │ experimental │ │ .a/.so/.o      │
     └────────────────┘ └───────────────┘ └────────────────┘
```

**El diagrama muestra el estado de transición F0–F3.** En ese estado, `base` enlaza `libmire_rt.a` como artefacto externo: Avenys no incorpora sus objetos C. En el estado final no existe la caja C y ambos tiers se implementan en Mire:

| Estado | Runtime `base` | Runtime `experimental` | ¿Cumple P6? |
|--------|----------------|------------------------|-------------|
| Hoy | C enlazado por Avenys | No existe | No |
| Transición (F4) | `libmire_rt.a` externo, por C ABI | PAL Mire opt-in | Sí: el compilador no embebe objetos C |
| Final (F9+) | PAL Mire conservadora, sin asm ni optimizaciones agresivas | PAL Mire bootstrapeada, puede usar asm/optimizaciones habilitadas | Sí |

`base` permanece como tier conservador incluso después de estabilizar `experimental`; permite aislar regresiones de optimización o plataformas no compatibles. El valor predeterminado continúa siendo `base`.

Reglas de la arquitectura:

1. **Owl no compila.** Resuelve, verifica y traduce a `CompileRequest`.
2. **Avenys no lee `owl.toml`.** No conoce TOML. Recibe todo por protocolo.
3. **El runtime base siempre puede compilar un `main` vacío** sin red y sin stdlib.
4. **La stdlib es un paquete** (`mire`) que enriquece, no un requisito.
5. **Los codecs viven en `ilogical`.** Ni Owl ni Kioto parsean JSON/TOML a mano.

---

## 5. Avenys — el compilador

**Versión actual:** 4.4.44 (`avenys/Cargo.toml`)
**Tamaño:** ~53 000 líneas Rust + objetos C de PAL
**Rol objetivo:** compilar un `CompileRequest`; nada más.

### 5.1 Inventario verificado

| Ruta | Estado | Acción |
|------|--------|--------|
| `src/lexer/` | Funcional | Mantener; añadir spans de byte (§13.3) |
| `src/parser/` | Funcional | Mantener; mejorar recuperación de errores (§13.14) |
| `src/types/` | Funcional | Mantener |
| `src/compiler/typeck*` | Funcional; códigos E01xx | Mantener; enriquecer diagnósticos |
| `src/compiler/mir/` | Funcional | Mantener |
| `src/avens/config.rs` | **Parsea TOML** | Reducir a fixtures de test del compilador |
| `src/avens/manifest.rs` | **Parsea `owl.toml`** | Eliminar del camino de build; solo tests |
| `src/avens/build_pipeline.rs` | 1012 líneas; `default_output_dir` con 3 ramas | Simplificar (§5.4) |
| `src/avens/build_support.rs` | 1587 líneas; compila C con tcc/clang, enlaza | Revisar; separar `link/` y `c_build/` |
| `src/pal/core/pal_core.c` | Runtime C | Migrar a Mire (§5.5) |
| `src/pal/core/pal_dispatch.c` | Tabla de dispatch | Migrar a Mire |
| `src/pal/core/pal_sandbox.c` | Sandbox C | Migrar a Mire o shim mínimo |
| `src/pal/core/pal_optional.c` | Stubs | **Eliminar** |
| `src/pal/core/pal_abi.h` | Contrato ABI | **Mantener** (único header C permitido) |
| `src/pal/linux/pal_linux.c` | 49 KB de syscalls | Migrar a `mire::pal::linux` |
| `src/pal/wasi/` | Vacío | No escribir hasta que WASI sea target real |
| `src/incremental/` | 1200+ líneas, cache funcional pero compleja | Refactor + esquema nuevo (§5.4) |
| `src/cli/options.rs` | `--config`, env vars, 800+ líneas de parse manual | Reescribir sobre tabla declarativa (§5.2) |
| `src/cli/test.rs` | 1000+ líneas; harness de tests | Extraer a `test_harness/` |
| `src/protocol/` | `CompileRequest` v1 + goldens | Extender a v2 (§5.3) |
| `src/.cobject_cache/*.o` | **Objetos C versionados dentro del repo** | **Borrar**; mover a `bin/.cache/cobjects` o `~/.cache/mire` |
| `bin/debug/test/**` | Artefactos `tests_*.mire` en el repo | **Borrar** + `.gitignore` |
| `target/` | Build de Cargo | `.gitignore` |
| `avenys/docs/errors/README.md` | 498 líneas, **desincronizado** | Regenerar desde el registro (§13.19) |

**[verificado]** `avenys/src/.cobject_cache/` contiene decenas de `.o` versionados. Es un fósil doble: artefacto dentro del árbol y caché sin sal de versión del compilador.

### 5.2 CLI definitiva

```text
mire <command> [options]

Commands:
  build     Compila a un artefacto
  run       Compila y ejecuta
  test      Compila y ejecuta tests
  check     Solo análisis (sin codegen)
  debug     Emite IR/ASM/tokens/AST
  explain   Explica un código de diagnóstico
  doctor    Diagnóstico del toolchain y del entorno
  cache     verify | gc | doctor | clear
  fix       Aplica correcciones automáticas (dry-run por defecto)
  completions  bash | zsh | fish
  man       Genera páginas de manual
  help      Ayuda por tema
```

#### 5.2.1 Flags de build/run/check

| Flag | Corto | Argumento | Semántica |
|------|-------|-----------|-----------|
| `--release` | `-r` | — | Perfil release (`-O3` implícito si no se pasa `-O`) |
| `--debug` | `-d` | — | Perfil debug (default) |
| `--opt-level` | `-O` | `0..3,s,z` | Nivel de optimización |
| `--name` | — | `<string>` | Nombre del artefacto; Owl lo sanea |
| `--link` | `-L` | `name=path` | Librería del ecosistema; repetible |
| `--hash` | `-H` | `sha256:<hex>` | Hash esperado de la última `--link` |
| `--link-lib` | `-l` | `<name>` | Librería nativa |
| `--link-dir` | — | `<path>` | Directorio del linker nativo (viejo `-L`) |
| `--target` | — | `<triple>` | Ver §5.2.3 |
| `--runtime` | — | `base\|experimental` | Default `base` |
| `--bundle` | — | `static\|static-strict\|dynamic` | Default `static`; fallback dinámico con aviso |
| `--artifact` | — | `bin\|static\|shared` | Tipo de salida |
| `--output-dir` | — | `<path>` | Directorio de salida explícito |
| `--cache-dir` | — | `<path>` | Directorio de cache explícito |
| `--jobs` | `-j` | `<n>` | Paralelismo (`0` = CPUs lógicas) |
| `--verbose` | `-v` | — | Trazas de carga/enlace |
| `--quiet` | `-q` | — | Solo errores |
| `--color` | — | `auto\|always\|never` | Política de color |
| `--error-format` | — | `human\|short\|json\|json-rendered-ansi\|sarif` | §13.6 |
| `--error-limit` | — | `<n>` | Máximo de diagnósticos mostrados |
| `--explain` | — | `<code>` | Explicación larga al final (§13.8) |
| `--locked` | — | — | Falla si un hash no coincide |
| `--offline` | — | — | Prohíbe red |
| `--keep-temps` | — | — | No borra temporales |
| `--deny` | `-D` | `<lint>` | Warn → error |
| `--warn` | `-W` | `<lint>` | Activa lint |
| `--allow` | `-A` | `<lint>` | Silencia lint |
| `--forbid` | `-F` | `<lint>` | Deny no anulable |
| `--cap-lints` | — | `allow\|warn` | Tope para dependencias |
| `--message-format` | — | `human\|json` | Compat con `--error-format` |
| `--proto` | — | — | Lee `CompileRequest`, escribe `CompileResponse` |
| `--help` | `-h` | — | Ayuda corta |
| `--version` | `-V` | — | Versión + target + runtime |

#### 5.2.2 Ejemplos canónicos

```bash
# Proyecto gestionado por Owl (Owl genera exactamente esto)
mire build --release --name "kioto-fork" \
  -L kioto="$HOME/.owl/libs/kioto@2.5.0"    -H sha256:ab12… \
  -L mire="$HOME/.owl/libs/mire@2.1.0-beta" -H sha256:cd34… \
  -L ilogical="$HOME/.owl/libs/ilogical@0.1.4" -H sha256:ef56… \
  -O3 -v

# Standalone
mire run mod.mire
# → mod::bin/debug/mod

# Tests con perfil optimizado
mire test --release -O3

# Cross-compilación
mire build --target aarch64-unknown-linux-gnu --runtime base

# Depuración de la cache
mire cache verify --verbose
mire cache doctor
```

#### 5.2.3 Targets

| Alias aceptado | Triple canónico |
|----------------|-----------------|
| `amd64-unknown-gnulinux`, `x86_64-unknown-gnulinux` | `x86_64-unknown-linux-gnu` |
| `aarch64-unknown-gnulinux` | `aarch64-unknown-linux-gnu` |
| `riscv64-unknown-gnulinux` | `riscv64-unknown-linux-gnu` |

Regla: se aceptan los alias de la organización y se **normalizan** antes de tocar LLVM. Un alias desconocido produce `E0703 invalid_value` con lista de valores válidos y sugerencia por distancia de edición.

#### 5.2.4 Ayuda declarativa (anti-drift)

Hoy `print_help` es una pared de `println!` en `src/cli/mod.rs:44-106`, y cada subcomando repite su propia ayuda (`test.rs:35-56`). Eso garantiza que la ayuda y el parser se desincronicen.

Propuesta: una tabla única de flags, y que **parser y ayuda se generen de ella**.

```rust
// src/cli/spec.rs
pub struct FlagSpec {
    pub long: &'static str,
    pub short: Option<char>,
    pub arg: Option<ArgKind>,
    pub help: &'static str,
    pub group: Group,
    pub since: &'static str,
    pub conflicts: &'static [&'static str],
    pub aliases: &'static [&'static str],
}

pub const BUILD_FLAGS: &[FlagSpec] = &[
    FlagSpec { long: "release", short: Some('r'), arg: None,
               help: "Compila con optimizaciones (perfil release)",
               group: Group::Profile, .. },
    FlagSpec { long: "link", short: Some('L'), arg: Some(ArgKind::NameValue),
               help: "Librería del ecosistema a enlazar: name=path (repetible)",
               group: Group::Dependencies, .. },
    FlagSpec { long: "hash", short: Some('H'), arg: Some(ArgKind::Sha256),
               help: "Hash esperado de la librería anterior",
               group: Group::Dependencies, .. },
    // …
];
```

Consecuencias:

- `mire build --help` se renderiza desde `BUILD_FLAGS`.
- `mire help build` muestra lo mismo + ejemplos + códigos de salida.
- `mire completions` se genera del mismo spec.
- Un test de CI compara `mire build --help` con el golden y falla si cambió sin actualizar docs.

### 5.3 Protocolo Owl ↔ Avenys, v2

**Estado actual [verificado]:**

- `avenys/src/protocol/messages.rs` define `CompileRequest`/`CompileResponse` v1 con goldens (`src/protocol/golden/v1_*.hex`).
- `owl/src/protocol/messages.mire` + `wire.mire` implementan el mismo codec en Mire (938 líneas) y **nadie los usa**.
- Owl invoca `rt_proc_capture_argv2(mire, args, …)` (`owl/src/util/mod.mire:646`) y pasa un TOML temporal con `--config`.

**Objetivo:** el protobuf es la única interfaz de build.

```protobuf
message SourceEntry {
  string id = 1;
  string path = 2;
  string sha256 = 3;
  uint64 size = 4;
  uint32 flags = 5;
  repeated string symbols = 6;   // NUEVO: símbolos que aporta
  string package = 7;            // NUEVO: kioto | mire | ilogical | user
  string version = 8;            // NUEVO
}

message Library {
  string name = 1;
  string path = 2;
  string version = 3;
  string sha256 = 4;
  bool   embedded = 5;           // NUEVO: --bundle static lo embebe
}

message DiagnosticPolicy {
  string error_format = 1;       // human|short|json|sarif
  string color = 2;              // auto|always|never
  uint32 error_limit = 3;
  bool   explain = 4;
  repeated LintLevel lints = 5;
  string cap_lints = 6;
}

message CompileRequest {
  uint32 protocol_version = 1;   // 2
  string request_id = 2;
  string entry = 3;
  LoadPolicy load_policy = 4;
  repeated SourceEntry manifest = 5;
  repeated CArtifact c_artifacts = 6;
  SecurityPolicy security = 7;
  repeated string flags = 8;
  string artifact = 9;
  string target = 10;
  string cache_dir = 11;
  string output_dir = 12;
  string name = 13;              // NUEVO
  repeated Library libraries = 14; // NUEVO
  RuntimeTier runtime = 15;      // NUEVO
  repeated string c_sources = 16;
  repeated string c_includes = 17;
  repeated string cflags = 18;
  repeated string c_libs = 19;
  repeated string defines = 20;
  repeated MacroBinding macros = 21;
  CachePolicy cache = 22;        // NUEVO
  repeated string link_dirs = 23;
  DiagnosticPolicy diagnostics = 24; // NUEVO
  string bundle = 25;            // NUEVO: static|static-strict|dynamic
  bool   locked = 26;            // NUEVO
}

message CompileResponse {
  CompileStatus status = 1;
  string output_path = 2;
  string output_sha256 = 3;      // NUEVO
  string cache_key = 4;          // NUEVO
  repeated Diagnostic diagnostics = 5; // NUEVO: estructurados
  string rendered_diagnostics = 6;     // NUEVO: ya formateados
  uint64 elapsed_ms = 7;         // NUEVO
  string stats_json = 8;         // NUEVO: archivos, símbolos, cache hits
  BundleResult bundle = 9;       // NUEVO: modo resuelto y deps dinámicas
}

message BundleResult {
  string resolved = 1;           // static|static-partial|dynamic
  repeated string dynamic_dependencies = 2;
}
```

Compatibilidad: v1 sigue decodificando porque los campos nuevos son ignorables. Los goldens v1 se conservan y se añaden `v2_compile_req.hex` / `v2_compile_resp.hex`.

**Regla dura:** Avenys **no** abre `owl.toml`, **no** abre `mire-config.toml`, **no** lee `MIRE_CONFIG`, `MIRE_SECURITY`, `MIRE_MACROS`, `MIRE_CACHE_DIR` ni `MIRE_OUTPUT_DIR`.

### 5.4 Cache: esquema, claves y fin del “cache exorcism”

#### 5.4.1 Modo standalone

`mire run mod.mire` produce:

```text
mod::bin/
├── debug/
│   ├── mod
│   └── mod.manifest.json
├── release/
│   ├── mod
│   └── mod.manifest.json
├── test/
│   └── …
├── struc/
│   └── <target>/<layout-hash>/   # structs, enums, vtables, layouts
└── .cache/
    ├── version.json
    ├── index/
    │   ├── files/                # hash(archivo) → meta
    │   ├── analyses/             # fingerprint → análisis parcial
    │   ├── builds/               # build key → artefacto
    │   ├── mir/                  # hash(fn) → MIR
    │   └── symbols/              # (pkg,mod,fn) → archivo
    ├── blobs/                    # contenido direccionado
    ├── wal/                      # write-ahead log
    ├── quarantine/               # caches incompatibles/corruptas
    └── tmp/                      # publicación atómica
```

Reglas:

- El nombre `mod` sale del archivo sin extensión.
- `::` es legal en nombres de directorio en Linux, macOS y Windows; aun así, `--output-dir` permite evitarlo.
- `mod::bin/` se añade a `.gitignore` global del usuario y a la plantilla de cada repo.
- Si el archivo se llama `main.mire`, el directorio es `main::bin/`; si el usuario quiere otro nombre, `--name`.

#### 5.4.2 Modo proyecto

```text
<proyecto>/bin/
├── debug/kioto-fork
├── debug/kioto-fork.manifest.json
├── release/kioto-fork
├── release/kioto-fork.manifest.json
├── test/
├── struc/
└── .cache/            # idéntico al standalone
```

- El binario se llama **siempre** igual en todos los perfiles.
- Saneado: `"kioto fork"` → `kioto-fork`; colapsar `-`, rechazar `..`, `/`, `\`, NUL.
- Un proyecto puede sobrescribir el nombre con `[build] name = "…"`, pero Owl sigue siendo quien lo sanea.

#### 5.4.3 `manifest.json`

```json
{
  "schema": 3,
  "name": "kioto-fork",
  "profile": "release",
  "target": "x86_64-unknown-linux-gnu",
  "runtime": "base",
  "bundle_requested": "static",
  "bundle_resolved": "static-partial",
  "dynamic_dependencies": ["sodium"],
  "build_key": "sha256:…",
  "output_sha256": "sha256:…",
  "compiler": { "version": "4.5.0", "commit": "…" },
  "inputs": [{ "path": "src/main.mire", "sha256": "…" }],
  "dependencies": [{ "name": "kioto", "version": "3.1.0", "sha256": "…" }],
  "created_ms": 1763000000000
}
```

`mire run` compara el manifest antes de `exec`. Si el `build_key` recalculado no coincide, recompila. **Nunca** ejecuta un binario cuyo manifest no corresponda.

#### 5.4.4 Build key completa

```text
build_key = sha256(
    compiler.version
  + compiler.commit
  + profile
  + opt_level
  + target
  + runtime
  + bundle_requested + bundle_resolved
  + sorted(native_dependency_identity_and_linkage)
  + output_name
  + sorted(flags)
  + sorted(defines)
  + Σ sha256(fuente)      // contenido, no mtime
  + Σ sha256(dependencia) // contenido de cada lib enlazada
  + sorted(symbol_set)    // símbolos realmente usados
  + security_policy_hash
)
```

Hoy **[reproducir]** la clave no incluye el grafo completo de dependencias ni el conjunto de símbolos, y el artefacto se escribe in-place. Eso explica el “cache exorcism”: se ejecuta el binario viejo porque la clave nueva colisiona con la vieja.

#### 5.4.5 Publicación atómica

```text
1. compilar a .cache/tmp/<uuid>/
2. fsync(archivo)
3. fsync(directorio)
4. rename(.cache/tmp/<uuid>/out, bin/<profile>/<name>)   // mismo filesystem
5. write(bin/<profile>/<name>.manifest.json.tmp)
6. fsync + rename
```

Nunca se escribe el binario final in-place. Nunca se ejecuta un binario sin manifest válido.

#### 5.4.6 Cuarentena automática

| Situación | Acción |
|-----------|--------|
| `version.json` con `schema` distinto | Mover `.cache` a `.cache/quarantine/<ts>/` y reconstruir. `warning[W0062 cache_schema_migrated]`. |
| Blob con hash que no corresponde | Cuarentena del blob; recompilar solo lo afectado. |
| WAL truncado o línea corrupta | Descartar la línea, continuar, registrar en `quarantine/wal-<ts>.log`. |
| `rename` cruza filesystems | Error `E0503 cache_write_failed` con sugerencia `--cache-dir` en el mismo FS. |
| Lock de cache tomado por otro proceso | Esperar con timeout; luego error claro con PID. |

**Regla de oro:** el usuario nunca debe ejecutar `rm -rf .cache`. Si hace falta, el compilador lo hace solo y lo dice.

#### 5.4.7 Refactor de `src/incremental/`

| Hoy | Después |
|-----|---------|
| `cache.rs` (1200+ líneas) | `wal.rs`, `blob_store.rs`, `index.rs`, `publish.rs`, `quarantine.rs` |
| `cache_types.rs` | `schema.rs` con `enum CacheSchema` y migradores |
| `NEW_FORMAT_VERSION` hardcodeado | `CACHE_SCHEMA` leído de `version.json` |
| `BuildCacheEntry` sin símbolos | `symbols: Vec<SymbolId>` |
| Test harness bajo clave normal (`cache.rs:42-44`) | Namespace `test/` separado |

Tests obligatorios de cache:

1. Dos procesos escribiendo a la vez → sin corrupción.
2. WAL truncado a la mitad → recuperación.
3. `version.json` viejo → cuarentena + rebuild.
4. Blob corrupto → solo se recompila lo afectado.
5. LRU: `blobs/` ≤ límite, `test/` ≤ 256 entradas.
6. `mire run` dos veces sin cambios → segunda no recompila y ejecuta el binario correcto.
7. Cambio de un byte en una dependencia → rebuild y binario nuevo.

### 5.5 Migración de PAL C → Mire

**Contrato ABI que sobrevive** (único C permitido en el estado final):

```c
/* avenys/src/pal/core/pal_abi.h */
void    mire_panic(const char* msg, int64_t len);
void*   mire_alloc(int64_t size, int64_t align);
void    mire_release(void* region, int64_t size);
int64_t mire_syscall6(int64_t nr, int64_t a1, int64_t a2,
                      int64_t a3, int64_t a4, int64_t a5, int64_t a6);
```

Orden de migración:

| Paso | Origen | Destino | Test |
|------|--------|---------|------|
| 1 | `pal_core.c` | `mire/src/pal/core.mire` | alloc/free/overflow/panic |
| 2 | `pal_linux.c` | `mire/src/pal/linux.mire` | open/read/write/stat/mmap/fork/exec/wait/socket |
| 3 | `pal_sandbox.c` | `mire/src/pal/sandbox.mire` | landlock/seccomp deny + allow |
| 4 | `pal_dispatch.c` | `mire/src/pal/dispatch.mire` | resolución de externs por nombre |
| 5 | `pal_optional.c` | **no se reescribe** | — |

Mientras no exista `asm("syscall")` o `syscall6` intrínseco, el shim `mire_syscall6` es la única función C nueva y se elimina en F4.

**Criterio de aceptación:** los 30+ kernels de `avenys/bench/kernels/` no regresan más de **2 %** por paso, comparados contra `avenys/bench/BASELINE.md`.

### 5.6 Bugs y fósiles de Avenys

| Ubicación | Problema | Acción |
|-----------|----------|--------|
| `cli/options.rs:414-416` | `mode` se fuerza a `Release` si `opt_level != O0`; `--debug -O2` queda incoherente | Precedencia explícita: el último flag gana, con `warning` si hay conflicto |
| `cli/options.rs:563-601` | Estado global por env vars | **Eliminar**; todo por `CompileRequest` |
| `cli/options.rs:220` | `-L` con dos significados | Reparto §1.1 |
| `default_output_dir` (`build_pipeline.rs:898`) | 3 ramas (`MIRE_OUTPUT_DIR`, project root, source parent) | `--output-dir` explícito; si falta, regla única §5.4 |
| `avens/config.rs` | `MireManifest` con alias `project`/`package`/`owl` | Solo fixtures de test |
| `pal/core/pal_optional.c` | Stubs | **Eliminar** |
| `src/.cobject_cache/` | `.o` versionados | **Borrar** |
| `bin/debug/test/**` | Artefactos | **Borrar** |
| `docs/errors/README.md` | Drift con el código | Regenerar (§13.19) |
| `ErrorKind::Runtime` | Todo es `E0015`, sin código específico | Taxonomía §13.4 |
| `Span` (line, column) | No hay byte offsets; rompe tabs/UTF-8 | Migrar a `ByteSpan` (§13.3) |

### 5.7 Tests de Avenys

- **Golden de diagnósticos:** `avenys/tests/diagnostics/cases/<code>/<caso>.mire` + `.stderr`.
- **JSON:** cada caso compara el esquema JSON, no solo el texto.
- **Cache:** `avenys/tests/cache/` con los 7 casos de §5.4.7.
- **Protocolo:** round-trip v1 + v2 y compatibilidad de campos desconocidos.
- **CLI:** `avenys/tests/cli/` con `--help`, flags desconocidos, conflictos.
- **Bench:** `avenys/bench/run.sh` como gate de rendimiento.

---

## 6. Owl — gestor de proyectos

**Versión actual:** 1.6.12 (`owl/owl.toml`)
**Tamaño:** ~10 291 líneas Mire en `owl/src/`
**Rol objetivo:** resolver el grafo de dependencias y producir un `CompileRequest`. No compila.

### 6.1 Separación arquitectónica

```text
owl/src/
├── compiler/          # NUEVO: todo lo que Avenys necesita
│   ├── request.mire   # CompileRequest v2
│   ├── flags.mire     # traducción de opciones a flags
│   ├── symbols.mire   # resolución por símbolo (§2.4 del plan v2)
│   └── invoke.mire    # spawn de Avenys / --proto
├── self/              # NUEVO: autosustento
│   ├── cfg.mire       # lectura/escritura de owl.toml
│   ├── resolve.mire   # grafo, versiones, semver
│   ├── lock.mire      # owl.lock
│   ├── cache.mire     # ~/.owl/cache
│   ├── libs.mire      # ~/.owl/libs
│   ├── registry.mire  # índice remoto
│   └── doctor.mire    # diagnóstico
├── commands/          # build, run, test, check, new, install, …
├── registry/          # cliente de registro
├── crypto/            # → delegar a kioto::crypto
└── util/              # solo utilidades sin parsing de formatos
```

Reglas:

1. `compiler/` **no** lee `owl.toml` para construir flags; recibe un `ResolvedProject`.
2. `self/` **no** conoce a Avenys; produce `ResolvedProject`.
3. `commands/` orquesta: `self/` → `compiler/` → proceso.
4. `util/` no puede contener `toml_get`, `json_get` ni parseo de argv.

Migración de `[exports]`:

```toml
[exports]
compiler = "code/compiler"
self     = "code/self"
build    = "code/commands/build"
run      = "code/commands/run"
# …
```

### 6.2 Bug crítico: bloqueo en proyecto fresco sin `owl.lock`

**Evidencia [verificado]:**

| Archivo | Línea | Problema |
|---------|-------|----------|
| `owl/src/lockfile/mod.mire` | 204-457 | `generate()` tiene un bloque duplicado casi literal (~280-434 y ~340-434) |
| `owl/src/lockfile/mod.mire` | 54-141 | `in_sync()` compara por conteo de líneas, no por entradas |
| `owl/src/lockfile/mod.mire` | 686-700 | `all()` descarga si `pkg_path == ""` o el path no existe, sin comprobar versión+hash local |
| `owl/src/build/mod.mire` | 157-174 | `ensure_project_dependencies` llama a `install::run` siempre que `in_sync()` sea falso |

**Comportamiento observado:** un proyecto nuevo con dependencias ya instaladas en `~/.owl/libs` intenta instalar/descargar antes de compilar. Sin red, bloquea.

**Diseño correcto (offline-first):**

```text
ensure_project_dependencies(project):
    if owl.lock exists and in_sync(project, lock):
        return Ok(())                      # cero red

    resolved = resolve_local(project)      # path | ~/.owl/libs/<name>@<ver> | ~/.owl/cache
    if resolved.is_complete():
        lock = generate(resolved)          # solo escribe el lock
        return Ok(())                      # cero red

    if offline:
        return Err(E0601 package_not_found) # con lista de faltantes y cómo instalarlas

    install_missing(resolved.missing)      # una sola vez, con confirmación
    lock = generate(resolve_local(project))
    return Ok(())
```

`in_sync` se reescribe sobre un parser TOML real y compara por entrada:

```text
for each dep in owl.toml:
    lock_entry = lock.deps.find(name = dep.name)
    if lock_entry is none: return false
    if lock_entry.version != resolved.version: return false
    if lock_entry.sha256  != installed.sha256:  return false
    if lock_entry.path    != resolved.path:     return false
return true
```

`generate` se reduce a **un solo recorrido** del grafo resuelto. El duplicado se elimina.

### 6.3 Autosustento

| Pieza | Estado | Objetivo |
|-------|--------|----------|
| `~/.owl/cfg/` | Existe parcialmente | Configuración global de Owl (registry, mirrors, offline, default target) |
| `~/.owl/libs/<name>@<ver>/` | Existe | Librerías instaladas, con `meta.toml` (nombre, versión, sha256, source) |
| `~/.owl/cache/resolution/<sha256(owl.toml)>` | Existe | Resolución canónica cacheada |
| `~/.owl/cache/registry/` | Existe | Índice remoto con TTL |
| `~/.owl/cache/tarballs/` | Nuevo | Tarballs verificados por hash |
| `owl doctor` | Nuevo | Reporta libs nativas faltantes (caso `libsodium`), hashes, target, runtime |

**Auto-resolución sin `owl.toml` (modo script):**

1. Buscar `~/.owl/libs/*/meta.toml`.
2. Construir un índice `name → versiones → path`.
3. Si el script hace `load kioto`, resolver a la versión más alta compatible con el `mire` instalado.
4. Si hay ambigüedad, `E0602 version_conflict` con las opciones y un flag `--pick name@version`.

**Nunca** se consulta la red para una dependencia que ya está instalada y cuyo hash coincide.

### 6.4 Fósiles de Owl

| Ubicación | Problema | Acción |
|-----------|----------|--------|
| `code/util/mod.mire:710-891` | `write_mire_config` escribe `[build]` **dos veces** (líneas 732 y 747) | **Eliminar**; lo reemplaza el protobuf |
| `code/util/mod.mire:26-214` | `toml_get`, `toml_get_array`, `toml_set`, `toml_extract_val` con `index`/`substr` | **Eliminar**; usar `ilogical::format::toml` |
| `code/util/mod.mire:386-418` | `json_get_str`, `json_get_int_str` manuales | **Eliminar**; usar `ilogical::format::json` |
| `code/build/mod.mire:19-152` | `append_c_and_security_flags` construye argv a mano y duplica `extern-libs`/`extern_libs` | **Eliminar**; va en `CompileRequest` |
| `code/build/mod.mire:180-229` | `test_config` escribe TOML temporal | **Eliminar**; `mire test --proto` |
| `code/build/mod.mire:404-414` | `extra_flags` se parte por espacios | `vec[str]` desde argv; nunca re-parsear |
| `code/lockfile/mod.mire:54-141` | `in_sync` por conteo de líneas | Parser real |
| `code/lockfile/mod.mire:204-457` | Bloque duplicado | Un recorrido |
| `code/_externs/` | externs manuales `lib "c"` | Migrar a `mire::pal` |
| `code/crypto/` | Duplica `kioto::crypto` | Delegar al paquete |
| `kioto = { path = "." }` en `kioto/owl.toml` | Auto-dependencia | **Eliminar**; puede causar bucles de resolución |

### 6.5 Comandos de Owl

| Comando | Hoy | Objetivo |
|---------|-----|----------|
| `owl build` | argv + TOML temporal | Resuelve → `CompileRequest` → Avenys |
| `owl run` | build + spawn | Build + `exec` verificado por manifest |
| `owl test` | TOML temporal | `mire test --proto` |
| `owl check` | checkup | `mire check --proto` |
| `owl new` | scaffold | Igual, sin escribir cache en el proyecto |
| `owl install` | Descarga | Idempotente, hash-first, sin red si ya está |
| `owl add` / `owl remove` | — | Modifica `owl.toml` + lock de forma atómica |
| `owl update` | — | Re-resuelve respetando semver |
| `owl tree` | — | Grafo de dependencias legible |
| `owl why <pkg>` | — | Por qué está esa dependencia |
| `owl doctor` | — | **Nuevo** |
| `owl oracle <symbol>` | — | **Nuevo**: `kioto::strings::trim` → paquete, versión, hash, archivo |
| `owl cache verify\|gc\|clear` | — | **Nuevo** |
| `owl fix` | — | Aplica fix-its de diagnósticos en el proyecto |

### 6.6 Resolución por símbolo

Sin cambiar `load`/`load!`/`use!` (D8), Owl puede enviar a Avenys **solo los archivos que contienen los símbolos llamados**:

1. Analiza los `.mire` del proyecto: `load pkg::mod` y llamadas `pkg::mod::fn(...)`.
2. Consulta el índice de cada paquete (`owl.toml` `[exports]` + cabecera de archivo).
3. Construye el conjunto transitivo de archivos.
4. Envía `SourceEntry { path, symbols }` solo para esos archivos.
5. Avenys compila lo recibido; la cache indexa por `(archivo, hash, symbol_set)`.

**Efecto:** un programa que usa `strings::trim` compila 1-3 archivos de kioto, no 47.

**Fallback:** `owl.toml [load] eager = ["kioto::net"]` fuerza un módulo completo. Si el enlace falla por símbolo faltante, el error `E0401 undefined_symbol` cita paquete, módulo y archivo candidato.

### 6.7 Tests de Owl

```text
owl/tests/
├── lock/
│   ├── fresh_project_no_lock.mire
│   ├── deps_already_installed.mire
│   ├── lock_out_of_sync.mire
│   ├── lock_corrupt.mire
│   └── offline_missing_dep.mire
├── resolve/
│   ├── semver_conflict.mire
│   ├── circular_dependency.mire
│   └── path_dependency.mire
├── compiler/
│   ├── request_v2_roundtrip.mire
│   └── flags_translation.mire
└── cli/
    ├── unknown_flag.mire
    └── help_golden.mire
```

Criterio: `owl build` en proyecto fresco con deps instaladas termina en **< 2 s** y con **0 accesos de red**.

---

## 7. Mire — identidad de la librería

**Versión:** 2.1.0-beta
**Tamaño:** ~87 archivos
**Rol objetivo:** runtime mínimo embebido + stdlib distribuible.

### 7.1 Decisión de identidad: híbrido

| Artefacto | Contenido | Distribución |
|-----------|-----------|--------------|
| **Runtime bootstrap** | arena/alloc, panic, ABI, entry point, PAL base | **Embebido en Avenys**; no es paquete owl |
| **Stdlib `mire`** | `alloc/`, `std/`, `aux/` | **Paquete owl** versionado |

Regla: el compilador **siempre** puede compilar un `main` vacío sin red. La stdlib enriquece; no es requisito.

Por qué no mover todo al compilador: la stdlib cambia más rápido que el compilador y debe poder actualizarse sin recompilar Avenys. Por qué no dejar todo como paquete: el compilador necesita `panic`, `alloc` y el entry point para producir un binario válido.

### 7.2 `mire/code/` — eliminación de objetos C

**[verificado]** `mire/code/` contiene exactamente:

```text
mire/code/c/json_parser.c
mire/code/c/toml_parser.c
mire/code/mod.mire          # main trivial de prueba
```

y `mire/owl.toml`:

```toml
[c]
sources = ["code/c/json_parser.c", "code/c/toml_parser.c"]
libs = ["m"]
```

Consumidores:

- `kioto/modules/json/mod.mire` → `extern fn c_json_*`
- `kioto/modules/toml/mod.mire` → `extern fn c_toml_*`

Plan:

1. Implementar JSON y TOML en Mire puro en `ilogical::format` (§8).
2. `kioto::json` y `kioto::toml` **desaparecen** (D5); si se necesita compatibilidad, re-exportan `ilogical::format::{json,toml}` con deprecación.
3. Borrar `mire/code/` completo y la sección `[c]` de `mire/owl.toml`.
4. Tests unitarios por símbolo del parser (§8.7).

Esto resuelve a la vez: objeto C eliminado, parser único, `ilogical` deja de ser un stub.

### 7.3 `mire/src/aux/pal.mire` — migrar a PAL bootstrapeada

**[verificado]** El archivo declara >100 `extern ... lib "pal"` (`rt_io_*`, `pal_alloc`, `pal_time_*`, `rt_strings_*`, etc.). No es un “bug silencioso”: es una dependencia estructural con el runtime C. Falla en cuanto se compila con `--runtime experimental` sin ese runtime, y ata la stdlib al compilador.

Plan:

1. `mire/src/aux/pal.mire` deja de declarar `extern` y pasa a re-exportar `mire::pal::*`.
2. `mire::pal` se implementa en Mire sobre `mire_syscall6` (§5.5).
3. `mire/src/std/{in,out,err}` usan `mire::pal::io`, no `rt_io_*`.
4. `mire/src/alloc/*` usan `mire::pal::mem`, no `rt_vecs_*`/`rt_strings_*`.
5. Durante F4, `base` enlaza el artefacto C externo `libmire_rt.a` por ABI y `experimental` usa PAL Mire; al final, ambos tiers usan la PAL Mire con optimización conservadora/agresiva respectivamente.

**No** se parchea con alias: se migra.

### 7.4 Estado de `alloc/` y `std/`

| Ruta | Uso actual | Acción |
|------|------------|--------|
| `src/alloc/vec/mod.mire` | `rt_vecs_*` | Migrar a `pal::mem` + `struc/` |
| `src/alloc/map/mod.mire` | `rt_map*` | Ídem |
| `src/alloc/str/mod.mire` | `rt_strings_*` | Ídem |
| `src/alloc/bytes/mod.mire` | `rt_*` | Ídem |
| `src/alloc/arr`, `maybe`, `result` | Stubs parciales | Completar con tests |
| `src/std/out`, `in`, `err` | `mire::aux::pal` | Migrar a `mire::pal::io` |
| `src/aux/macros/*` | `assert`, `dbg`, `panic`, `checks` | Mantener; añadir `must_use`, `deprecated` |
| `src/lib.mire` | `status()` con string gigante | Reconstruir cuando el optimizador no reordene `+` (bug documentado) |

**[verificado]** `mire/src/alloc/bin/.cache/` y `mire/bin/.cache/` están dentro del repo. Se borran (§12).

### 7.5 Tests de Mire

```text
mire/tests/
├── mod.mr
├── _support/{harness.mire,fixture.mire}
├── alloc/
│   ├── vec/{mod.mr,symbols/{push,get,pop,len,clear}.mire,logs/}
│   ├── map/{mod.mr,symbols/…}
│   ├── str/{mod.mr,symbols/…}
│   ├── bytes/…
│   ├── arr/…
│   ├── maybe/…
│   └── result/…
├── std/
│   ├── out/{mod.mr,symbols/…}
│   ├── in/{mod.mr,symbols/…}
│   └── err/{mod.mr,symbols/…}
└── aux/
    ├── macros/{mod.mr,symbols/…}
    └── panic/…
```

Recuperar `json_test.mire` (referenciado en artefactos generados y ausente) o eliminarlo del índice si ya no aplica.

---

## 8. Ilogical — dueño de los codecs

**Versión:** 0.1.4
**Tamaño:** ~133 archivos, de los cuales 20 son directorios vacíos con solo `owl.toml`
**Rol objetivo:** único dueño de JSON, TOML, binario, encoding, URL y HTTP.

### 8.1 Inventario verificado

```text
ilogical/src/
├── lib.mire
├── serve.mr
├── base/mod.mire
├── schema/mod.mire
├── encoding/mod.mire          + 3 subdirs vacíos (base64, hex, percent)
├── format/mod.mire            + 3 subdirs vacíos (bin, json, toml)
├── url/mod.mire               + 3 subdirs vacíos (parse, resolve, encode)
└── web/mod.mire               + 13 subdirs vacíos
```

**[verificado]** `format/mod.mire` delega a símbolos que el runtime no exporta:

```mire
pub fn json_decode: (s :str) :i64 { return rt_json_decode(s) }
pub fn toml_decode: (s :str) :i64 { return rt_toml_decode(s) }
pub fn json_encode_pretty: (v :i64, indent :str) :str { return rt_json_encode_pretty(v, indent) }
```

Mismo patrón que el bug de `mire/src/aux/pal.mire`: `extern` a símbolos inexistentes.

### 8.2 Qué se elimina

- Los 20 directorios con solo `owl.toml`.
- Las entradas `[exports]` asociadas.
- Los `rt_json_*` / `rt_toml_*` / `rt_bin_*` que no existen.
- `src/serve.mr` como binario separado: se mueve a `examples/` o se elimina.

**Regla P1:** no se escribe `json/mod.mire` hasta que haya parser real. El directorio aparece cuando nace el código y su test.

### 8.3 Modelo de datos: `base::Value`

```mire
pub enum Value {
    Null
    Bool(bool)
    I64(i64)
    F64(f64)
    Str(str)
    Arr(vec[Value])
    Map(map[str Value])
}
```

Requisitos:

- `Value` debe ser inspeccionable sin punteros opacos.
- Igualdad estructural (`Value == Value`).
- `Value::get::path("a.b[0].c")` para acceso tipado.
- `Value::as::i64()` devuelve `result[i64 Error]`, nunca `0`.

### 8.4 JSON en Mire puro

API objetivo (`ilogical::format::json`):

```mire
pub fn parse: (s :&str) :result[Value Error]
pub fn parse_file: (path :&str) :result[Value Error]
pub fn encode: (v :&Value) :result[str Error]
pub fn encode_pretty: (v :&Value, indent :i64) :result[str Error]
pub fn encode_into: (v :&Value, out :&mut str) :result[i64 Error] // bytes escritos
pub fn encode_lossy: (v :&Value) :str // NaN/Inf → null
```

Requisitos del parser:

| Caso | Comportamiento |
|------|----------------|
| Números fuera de rango i64 | `result.Err(Error { code: "json/out_of_range" })` |
| `\uXXXX` inválido | Error con offset exacto |
| Surrogates huérfanos | Error |
| Profundidad > `max_depth` (default 256) | Error `json/depth_exceeded` |
| Coma final | Error `json/trailing_comma` |
| Comentarios | Error `json/comment_not_allowed` |
| BOM | Aceptar y saltar |
| UTF-8 inválido | Error con byte offset |
| NUL embebido en string | Aceptar (`\u0000`) |
| Documento vacío | Error `json/empty_input` |
| Muy grande (> `max_bytes`, default 64 MiB) | Error `json/too_large` |

El parser **no** asigna por token; usa un arena/slice y construye `Value` al final. Debe ser binario-safe.

Errores de encode JSON: `json/nan_not_representable`, `json/infinity_not_representable`, `json/depth_exceeded`, `json/too_large`, `json/invalid_utf8` y `json/buffer_too_small`. `encode_into` devuelve bytes escritos en `Ok`; `Err` diferencia buffer insuficiente de valor no representable. TOML admite NaN/Inf; sus errores de encode incluyen `toml/depth_exceeded` y `toml/too_large`. Tests deben alcanzar cada código declarado y comprobarlo por separado.

### 8.5 TOML en Mire puro

API objetivo (`ilogical::format::toml`):

```mire
pub fn parse: (s :&str) :result[Value Error]
pub fn parse_file: (path :&str) :result[Value Error]
pub fn encode: (v :&Value) :result[str Error]
pub fn encode_into: (v :&Value, out :&mut str) :result[i64 Error] // bytes escritos
```

Cobertura mínima (TOML 1.0):

- Tablas `[a.b]`, arrays de tablas `[[a]]`.
- Claves desnudas, con comillas, con puntos.
- Strings básicos, literales, multilínea.
- Enteros (dec/hex/oct/bin), floats (incl. `inf`, `nan`), booleanos.
- Fechas/horas → `Value::Str` normalizado (no se inventa un tipo DateTime hasta que exista).
- Arrays heterogéneos → error si TOML lo prohíbe.
- Duplicados → error `toml/duplicate_key` con ambas ubicaciones.
- Profundidad máxima configurable.

**Este parser reemplaza:**
- `mire/code/c/toml_parser.c`
- `owl/src/util/mod.mire` `toml_get*`/`toml_set`
- `kioto/modules/toml/mod.mire`

### 8.6 Encoding, URL y Web

| Módulo | Estado | Plan |
|--------|--------|------|
| `encoding::hex` | Vacío | Implementar `encode`/`decode` con tests (mover desde kioto) |
| `encoding::base64` | Vacío | Implementar estándar + URL-safe + sin padding |
| `encoding::percent` | Vacío | Implementar para URL |
| `url::parse` | Vacío | Parser RFC 3986 |
| `url::encode` | Vacío | Query encoding |
| `url::resolve` | Vacío | Resolución relativa |
| `web::methods` | Vacío | Enum `Method` |
| `web::headers` | Vacío | Mapa case-insensitive |
| `web::request`/`response` | Vacío | Structs + serialización HTTP/1.1 |
| `web::client` | Vacío | TCP plano; TLS en fase posterior |
| `web::tls`, `certificates`, `verification` | Vacío | No escribir hasta tener backend TLS |
| `web::cookies` | Vacío | Parser `Set-Cookie` |
| `web::transport`, `connection` | Vacío | Interno del cliente |

### 8.7 Tests de Ilogical

```text
ilogical/tests/
├── format/
│   ├── json/{mod.mr,symbols/{parse,encode,encode_pretty}.mire,logs/}
│   └── toml/{mod.mr,symbols/{parse,encode}.mire,logs/}
├── encoding/
│   ├── hex/…
│   ├── base64/…
│   └── percent/…
├── url/{mod.mr,symbols/{parse,encode,resolve}.mire}
└── schema/…
```

Criterio: JSON y TOML deben pasar el corpus de casos de §8.4 y §8.5, y un benchmark contra el parser C actual no debe ser más de 3× más lento (mientras se optimiza; el objetivo final es paridad).

---

## 9. Kioto — librería estándar

**Versión:** 3.1.0
**Tamaño:** 15 módulos, ~425 archivos, ~5 000 líneas Mire
**Rol objetivo:** stdlib de alto nivel sobre `mire::pal` e `ilogical`.

### 9.0 Hallazgo crítico: la allowlist de seguridad está rota

**[verificado]** `kioto/owl.toml` declara:

```toml
[security]
mode = "strict"
externs = ["pal_*", "rt_list_*", "rt_lists_*", "rt_vecs_*", "rt_dict*", "rt_map*",
           "rt_string*", "rt_i64_to_string", "rt_f64_to_string", "rt_bool_to_string",
           "rt_managed_*"]
```

Y el enforcement existe en `avenys/src/compiler/typeck.rs:281-296` y `:653-657`:

```rust
if config.mode != SecurityMode::Strict { return true; }
if config.externs.is_empty() { return false; }
config.externs.iter().any(|p| /* glob */)
```

Cruzando los 151 `extern fn` de `kioto/modules/**` contra los patrones:

- **85 de 151 quedan fuera del allowlist.**
- Ejemplos: `rt_crypto_sha256_hex`, `rt_pal_root_open`, `rt_proc_shell`, `rt_ws_decode_frame`, `rt_str_from_byte`, `c_json_parse`, `c_toml_parse`, `rt_get_args`.
- Los patrones `rt_list_*`, `rt_lists_*`, `rt_dict*`, `rt_map*` **no corresponden a ningún extern actual**: son residuos de Kioto 2.x.

Hay dos hipótesis que se deben distinguir; que un build termine no demuestra por sí solo un agujero:

- **H1 — allowlist incompleta:** el enforcement sí corre, pero la allowlist de Kioto no cubre 85 externs. El build de Kioto debe fallar con E0022.
- **H2 — política de dependencias omitida:** el enforcement funciona en la raíz, pero no valida una dependencia strict. Una raíz open que carga una dependencia strict con un extern no permitido debe fallar citando esa dependencia.

El contrato esperado es que `strict` se aplique también a dependencias; una allowlist vacía deniega todo, pero debe producir un error claro al cargar la configuración, no una cascada de E0022.

**Comando de confirmación:**

```bash
cd kioto && owl build 2>&1 | head -40
```

**Corrección:**

1. Regenerar la allowlist desde el código, no a mano:
   ```bash
   owl lint externs --write   # extrae extern fn y produce el array
   ```
2. Mientras `--runtime base` exista, la allowlist debe incluir todos los `rt_*` reales.
3. Tras la migración a PAL (§5.5), debe quedar **solo** `pal_*` y `mire_*`.
4. `c_json_*`/`c_toml_*` desaparecen con §7.2/§8.
5. Añadir `kioto/tests/security/allowlist.mire` (cada extern cubierto), `avenys/tests/security/strict_root.mire` (extern no permitido → E0022), `avenys/tests/security/strict_dependency.mire` (la dependencia strict también se valida y se cita) y `owl/tests/security/self_build.mire` (Kioto compila tras corregir la lista).
6. Rechazar allowlist strict vacía con diagnóstico de configuración y conservar W0054 `stale_security_allowlist` para patrones sin coincidencia.

### 9.1 Inventario general

| Módulo | Archivos | LOC | Estado | Stubs / faltantes | Decisión |
|--------|----------|-----|--------|-------------------|----------|
| `strings` | `mod.mr`, `_externs.mr` | ~110 | Real, FFI | `to::i64` sin `result` | Migrar a PAL; tests |
| `time` | `mod.mr` | 20 | Mínimo | `sleep`, `mono`, `Duration`, `fmt` | Ampliar |
| `async` | `mod.mr`, `_externs.mr` | 91 | **Stub funcional** | `sleep`, `timeout`, `interval`, `instant`, `await`, `run`, `runtime`, `spawn` | Rehacer |
| `proc` | `mod.mr`, `_externs.mr` | ~200 | Real | Externs a PAL | Conservar |
| `fs` | `mod.mr`, `_externs.mr` | 925 | **Real** | Tests ensucian raíz | Conservar |
| `net` | `mod.mr`, `ws.mr`, `_externs.mr` | 361 | Parcial | DNS, timeouts, TLS | Rehacer API |
| `mem` | `mod.mr` | 13 | Mínimo | `map`, `stats` | Ampliar |
| `cpu` | `mod.mr` | 6 | Mínimo | `info`, `features` | Ampliar |
| `env` | `mod.mr` | ~25 | Mínimo | `set`, `unset`, iter | Ampliar |
| `cli` | `mod.mr`, `_externs.mr` | ~35 | Parcial | Flags tipadas, `--x=y`, `--` | Rehacer |
| `log` | `mod.mr` | 20 | Mínimo | Niveles, filtro, stderr | Rehacer |
| `math` | 11 subdirs | 2857 | Parcial | 4 stubs de 2 líneas | Recortar + completar |
| `crypto` | hash/encode/checksum/random/sign | ~450 | Parcial | libsodium, allowlist | Migrar |
| `json` | `mod.mire` | 129 | **C FFI** | — | **Eliminar** (D5) |
| `toml` | `mod.mire` | 81 | **C FFI** | — | **Eliminar** (D5) |

### 9.2 `kioto::strings`

**Estado [verificado]:** `from::i64`, `from::bool`, `from::f64`, `to::i64`, `starts::with`, `ends::with`, `replace::all`, `pad::left`, `is::empty` ya siguen `mire::case`. Todo es FFI a `rt_strings_*`.

Problemas:

| Símbolo | Firma actual | Problema |
|---------|--------------|----------|
| `to::i64` | `(value :&str) :i64` | Devuelve `0` ante error (fallback silencioso) **[reproducir comportamiento exacto de `rt_string_to_i64`]** |
| `char_at` | `(s, index) :i64` | Nombre fuera de convención |
| `from_byte` | `(byte) :str` | Idem |
| `index` | `(s, sub) :i64` | Debería ser `index::of`; `-1` como “no encontrado” es ambiguo |
| `substr` | `(s, start, len) :str` | Sin validación de rango documentada |
| `split` | `(s, sep) :vec[str]` | Sin caso `sep == ""` documentado |

API objetivo:

```mire
pub fn to: () {
    pub fn i64: (value :&str) :result[i64 ParseError]
    pub fn i64::or: (value :&str, fallback :i64) :i64   // explícito, no silencioso
    pub fn f64: (value :&str) :result[f64 ParseError]
}

pub fn index: () {
    pub fn of: (s :&str, sub :&str) :maybe[i64]   // none = no encontrado
}

pub fn char: () { pub fn at: (s :&str, index :i64) :maybe[i64] }
pub fn from: () { pub fn byte: (b :i64) :result[str Error]; /* + i64, bool, f64 */ }
```

Tests (`kioto/tests/strings/symbols/`):

| Símbolo | Éxito | Límite | Anomalía |
|---------|-------|--------|----------|
| `to::i64` | `"42"` → `Ok(42)` | `"0"`, `"-9223372036854775808"` | `"abc"` → `Err(parse)`; `""`; `" 42"` |
| `to::i64::or` | `"x", 7` → `7` | `""` → fallback | no aborta |
| `index::of` | `"abc","b"` → `Some(1)` | `""` → `None` | `sub` más largo que `s` |
| `char::at` | `"ñ",0` → codepoint | último byte | índice negativo/fuera de rango |
| `substr` | rango válido | `start == len` | `start > len`, `len` negativo |
| `split` | `"a,b"` | `""` | `sep == ""` |

### 9.3 `kioto::time`

**Estado [verificado]:** 20 líneas. Solo `now::ms`, `now::ns`, `elapsed`, `mark`. Todo FFI `pal_time_*`.

Faltantes: `sleep`, reloj monotónico, `Duration`, `Instant`, formateo ISO-8601, parsing.

API objetivo:

```mire
pub struct Duration { ms :i64 }

pub fn now: () {
    pub fn ms: () :i64
    pub fn ns: () :i64
    pub fn mono: () :i64          // monotónico, no salta con NTP
}

pub fn sleep: (d :Duration) :result[() Error]
pub fn mark: () :Instant
pub fn elapsed: (since :Instant) :Duration
pub fn fmt: () {
    pub fn iso8601: (epoch_ms :i64) :str
    pub fn rfc3339: (epoch_ms :i64) :str
}
pub fn parse: (s :&str) :result[i64 Error]
```

Reglas:

- `sleep` con `ms < 0` → `Err(time/invalid_duration)`, **no** `EINVAL` crudo.
- `elapsed` nunca negativo (si el reloj retrocede, devuelve 0 y emite `W0055 clock_regression` en debug).
- `now::mono` no se usa para fechas; solo para medir.

Tests: `sleep(0)` no bloquea; `sleep(10)` mide ≥ 10 ms y < 200 ms; `sleep(-1)` → error específico; `elapsed` ≥ 0; `fmt::iso8601(0)` → `1970-01-01T00:00:00Z`.

### 9.4 `kioto::async` — el caso paradigmático

**Estado [verificado]:** 80 líneas en `mod.mr` + 11 en `_externs.mr`.

Lo que existe:

```mire
pub struct Channel { handle :i64 }
pub struct Task    { value :str }

pub fn task: () {
    pub fn ready: (value :str) :Task
    pub fn value: (t :&Task, fallback :&str) :str
}
pub fn channel: () {
    pub fn create, send, recv, close
}
```

Lo que **no** existe y el nombre del módulo promete: `spawn`, `await`, `run`, `runtime`, `sleep`, `timeout`, `interval`, `instant`.

Problemas de diseño:

1. `Task` es un valor ya terminado, no una promesa. El comentario lo admite (“holds the result, not a promise of one”), pero entonces `async` no es async.
2. `Task.value :str` impide transportar `i64`, `bool` o structs.
3. `Channel` solo mueve bytes; no hay `send[T]`/`recv[T]`.
4. Todo va a `pal_channel_*` en C; no hay scheduler ni runtime en Mire.
5. `spawn`/`wait` viven en `proc`, no en `async`.

Decisión de alcance: **`async` no implementa un scheduler completo en F1**. Implementa un modelo cooperativo mínimo sobre PAL, con API honesta:

```mire
pub struct Task[T] { id :i64 }

pub fn task: () {
    pub fn ready[T]: (v :T) :Task[T]
    pub fn is_ready: (t :&Task[T]) :bool
}

pub fn spawn[T]: (f :fn() :T) :Task[T]
pub fn await[T]: (t :Task[T]) :result[T Error]
pub fn sleep: (d :time::Duration) :result[() Error]
pub fn timeout[T]: (d :time::Duration, f :fn() :T) :result[T TimeoutError]
pub fn interval: (d :time::Duration) :Interval
pub fn instant: () :time::Instant
pub fn run: (f :fn()) :result[() Error]      // arranca el runtime cooperativo
pub fn runtime: () :RuntimeInfo
```

Si los genéricos con `fn()` no están listos, la primera entrega usa `Task` no genérico con `value :bytes` y conversión explícita, y se documenta como limitación temporal. **No se escribe `spawn` que no arranque nada.**

Contrato por símbolo (tests en `kioto/tests/async/symbols/`):

| Símbolo | Éxito | Límite | Anomalía |
|---------|-------|--------|----------|
| `task::ready` | valor 42 | string vacío | NUL embebido |
| `task::is_ready` | true tras ready | recién creado | task ya consumida → false o error |
| `spawn` | ejecuta y devuelve | función vacía | función que aborta → `Err` |
| `await` | devuelve valor | await dos veces | task cancelada → `Err(task/cancelled)` |
| `sleep` | 10 ms | 0 ms | negativo → `Err(time/invalid_duration)` |
| `timeout` | completa antes | 0 ms | vence → `Err(timeout)` |
| `interval` | 3 ticks | close inmediato | doble close no aborta |
| `instant` | diff ≥ 0 | mismo instante | reloj retrocede → 0 |
| `run` | termina | sin tareas | doble `run` → `Err(runtime/already_running)` |
| `channel::send` | 4 bytes | 0 bytes | buffer lleno → `Err` o bloqueo documentado |
| `channel::recv` | recibe | buffer justo | close → EOF |
| `channel::close` | cierra | doble close | handle inválido |

### 9.5 `kioto::proc`

**Estado [verificado]:** ~187 líneas, bien diseñado. `run::create`, `run::spawn`, `run::output`, `run::output_cwd` pasan argv a `execvp`; `shell` es un símbolo separado, denegado por defecto.

Acciones:

- Migrar `rt_proc_*` a `mire::pal::proc`.
- `Process { handle :i64 }` → `Process` con cierre determinista; documentar que copiar el struct copia el handle.
- `run::output` debe devolver `result[Output Error]`, no `str` con `-1` mágico.
- `shell` debe requerir `--allow-shell` explícito y registrar `W0056 shell_invocation`.

Tests: `/bin/true` → 0; `/bin/false` → 1; binario inexistente → `Err(proc/not_found)`; argumento con espacio → un solo argv; `shell("echo $HOME")` → expande; `shell` sin permiso → error de seguridad.

### 9.6 `kioto::fs`

**Estado [verificado]:** 925 líneas, el módulo más maduro. Sandbox con raíz, symlinks, locks.

Problemas:

- 45 externs `rt_pal_*`/`rt_sbx_*` que no están en la allowlist (§9.0).
- Tests crean directorios en la raíz del repo (§11.4).
- Sin `fs::copy`, `fs::rename` atómico documentado, `fs::metadata` tipado.

Acciones:

- Migrar externs a `mire::pal::fs`.
- Añadir `Metadata { size, mode, is_dir, is_symlink, mtime }`.
- `open` debe devolver `result[File Error]`; `read`/`write` con `result[i64 Error]`.
- Documentar y testear: path traversal, symlink escape, TOCTOU, NUL en path, path vacío, path > PATH_MAX.

Tests: mover `tests/fs/sandbox` y `tests/fs/write` a `tests/fs/symbols/…` con `fixtures/tmp/`; el sandbox debe crear y borrar su propio árbol.

### 9.7 `kioto::net` y `net::ws`

**Estado [verificado]:** 103 + 258 líneas. `Socket`, `Listener`, PAL constants como funciones.

Problemas:

| Problema | Evidencia | Acción |
|----------|-----------|--------|
| Constantes como funciones | `pub fn PAL_SOCKET_TCP: () :i32` | `socket::kind::tcp()` |
| Sin DNS | `connect` recibe IP | Añadir `dns::resolve` |
| Sin timeouts | `recv` bloquea | `socket::set_timeout` |
| Sin TLS | — | Fase posterior (`ilogical::web::tls`) |
| `ws` con `rt_ws_*` fuera de allowlist | `ws.mr:252-253` | Migrar a PAL o implementar framing en Mire |
| Sin `shutdown` | — | Añadir |

API objetivo:

```mire
pub fn dns: () {
    pub fn resolve: (host :&str) :result[vec[addr] Error]
}
pub fn socket: () {
    pub fn kind: () { pub fn tcp: () :Kind; pub fn udp: () :Kind }
    pub fn connect: (addr :&addr, timeout :time::Duration) :result[Socket Error]
    pub fn set_timeout: (s :&Socket, d :time::Duration) :result[() Error]
    pub fn shutdown: (s :&Socket) :result[() Error]
}
pub fn listener: () {
    pub fn bind: (addr :&addr) :result[Listener Error]
    pub fn accept: (l :&Listener) :result[Socket Error]
}
```

Tests: connect a servidor local efímero; connect a puerto cerrado → `Err(net/refused)`; DNS `localhost` → 127.0.0.1; timeout de 10 ms; accept tras close → error claro; ws handshake + frame de 0 bytes.

### 9.8 `kioto::mem` y `kioto::cpu`

**Estado [verificado]:** `mem` 13 líneas (3 externs), `cpu` 6 líneas.

`mem` objetivo:

```mire
pub fn total: () :i64
pub fn available: () :i64
pub fn process: () :i64
pub fn stats: () :MemStats          // total, free, buffers, cached, swap
pub fn map: (fd :i64, len :i64) :result[ptr Error]
pub fn unmap: (p :ptr, len :i64) :result[() Error]
```

`cpu` objetivo:

```mire
pub fn count: () :i64
pub fn info: () :CpuInfo            // vendor, model, mhz
pub fn features: () :vec[str]       // sse4_2, avx2, avx512f, …
```

Tests: `total > 0`; `available <= total`; `process > 0`; `cpu::count >= 1`; `features` no vacío en x86_64.

### 9.9 `kioto::env`

**Estado [verificado]:** ~25 líneas. `args`, `cwd`, `get`. Todo FFI.

Problemas:

- `args` devuelve `vec[str]` desde `rt_get_args(argc, argv)`; depende de que el runtime haya capturado argv.
- `get` devuelve `&str`; un nombre inexistente no se distingue de uno vacío.
- Sin `set`/`unset`, sin iterador de entorno.

API objetivo:

```mire
pub fn var: () {
    pub fn get: (name :&str) :maybe[str]
    pub fn set: (name :&str, value :&str) :result[() Error]
    pub fn unset: (name :&str) :result[() Error]
    pub fn all: () :map[str str]
}
pub fn dir: () {
    pub fn cwd: () :result[str Error]
    pub fn home: () :maybe[str]
    pub fn tmp: () :str
}
pub fn args: () :vec[str]
pub fn args::os: () :vec[str]          // sin pérdida si no es UTF-8
```

Tests: `var::get("NO_EXISTE_123")` → `None`; `set` + `get` → `Some`; `set("", …)` → error; `args` con 0 args; `dir::cwd` tras `chdir`; `dir::home` sin `HOME` → `None`.

### 9.10 `kioto::cli`

**Estado [verificado]:** ~35 líneas. `parse(raw :&vec[str]) :map[str str]`.

Bugs concretos del parser actual:

1. Solo entiende `--flag value`, **no** `--flag=value`.
2. No entiende flags cortas (`-v`, `-j4`).
3. No entiende `--` como terminador.
4. Un flag repetido sobrescribe el anterior (no hay `vec`).
5. Un flag sin valor se guarda como `"1"` sin distinguir de `"1"` literal.
6. No hay tipos: todo es `str`.
7. No hay ayuda, ni validación, ni errores.

API objetivo:

```mire
pub struct Spec {
    name :str
    short :maybe[char]
    kind :ArgKind          // Flag | Value | Count | Multi
    required :bool
    default :maybe[str]
    help :str
}

pub struct Matches { /* … */ }

pub fn parser: (specs :&vec[Spec]) :Parser
pub fn Parser::parse: (args :&vec[str]) :result[Matches CliError]
pub fn Parser::help: () :str
pub fn Parser::usage: () :str
```

Reglas:

- `--name=value` y `--name value`.
- `-j4` y `-j 4`.
- `--` termina el parseo; lo demás va a `positional`.
- Repetidos → `Multi`.
- Desconocido → `Err(CliError { code: "cli/unknown_flag", suggestion })`.
- Falta valor → `Err(cli/missing_value)`.
- Conflicto → `Err(cli/conflict)`.

Tests: cada una de las 7 reglas, más `--help` golden, `-h`, `--version`, flag desconocido con sugerencia, `--` con args que parecen flags.

### 9.11 `kioto::log`

**Estado [verificado]:** 20 líneas. `info`, `warn`, `error`; concatenan con `rt_string_concat` y escriben con `dasu`. **Todo a stdout**, sin nivel, sin timestamp, sin filtro.

Problemas:

- `error` a stdout rompe la separación estándar y el piping.
- `[ERROR]` como prefijo es el único “protocolo”.
- Sin `debug`/`trace`.
- Sin filtro por nivel (`RUST_LOG`-like).
- `rt_string_concat` es un extern fuera de allowlist.

API objetivo:

```mire
pub enum Level { Trace, Debug, Info, Warn, Error }

pub fn set_level: (l :Level)
pub fn set_format: (f :Format)        // Text | Json
pub fn info:  (msg :&str)
pub fn warn:  (msg :&str)
pub fn error: (msg :&str)
pub fn debug: (msg :&str)
pub fn trace: (msg :&str)
pub fn with_fields: (msg :&str, fields :&map[str str])
```

Reglas:

- `Trace`/`Debug`/`Info` → stdout.
- `Warn`/`Error` → **stderr**.
- Formato por defecto: `2026-11-13T12:00:00Z INFO  msg`.
- `Json`: `{"ts":…,"level":"info","msg":"…","fields":{…}}`.
- Nivel por `KIOTO_LOG` o `set_level`.
- Sin asignación en el path caliente (buffer reutilizado).

Tests: nivel filtra; `error` va a stderr; JSON parseable; mensaje con `\n` no rompe una línea JSON; nivel inválido en env → warning y default `info`.

### 9.12 `kioto::math`

**[verificado]** 2857 líneas en 11 subdirectorios. Los stubs:

| Archivo | LOC | Contenido |
|---------|-----|-----------|
| `math/complex/mod.mr` | 2 | solo comentario |
| `math/decimal/mod.mr` | 2 | solo comentario |
| `math/hyperbolic/mod.mr` | 2 | solo comentario |
| `math/special/mod.mr` | 2 | solo comentario |

**Decisión P1:** se **borran** los 4 directorios, se quitan de `math/mod.mr` y de `[exports]`. Se anotan aquí como pendientes:

- `complex`: requiere un tipo `Complex { re :f64, im :f64 }` y operadores.
- `decimal`: requiere precisión arbitraria o `i128`; decidir representación antes de escribir.
- `hyperbolic`: `sinh`, `cosh`, `tanh`, `asinh`, `acosh`, `atanh`; dependen de `exp`/`log`.
- `special`: `erf`, `gamma`, `beta`; alto costo, baja prioridad.

Cuando se implementen, nacen con `math/<mod>/mod.mr` + `tests/math/<mod>/symbols/…`.

Módulos que sí se quedan y se completan:

| Módulo | Estado | Acciones |
|--------|--------|----------|
| `int` | 329 LOC | `abs`, `min`, `max`, `clamp`, `gcd`, `lcm`, `pow` con overflow checks |
| `float` | 306 LOC | `abs`, `floor`, `ceil`, `round`, `trunc`, `is_nan`, `is_inf`, `approx_eq` |
| `trig` | 230 LOC | `sin`, `cos`, `tan`, `asin`, `acos`, `atan`, `atan2` |
| `power` | 809 LOC | Revisar duplicación con `int::pow`; unificar |
| `random` | 161 LOC | Separar PRNG de `crypto::random`; documentar que no es criptográfico |
| `seq` | 99 LOC | `range`, `map`, `filter`, `fold`; añadir `enumerate` |
| `stats` | 269 LOC | `mean`, `median`, `variance`, `stddev`; usar `f64` |
| `sum` | 140 LOC | `sum`, `product`, `minlist`, `maxlist` |
| `consts` | 45 LOC | `PI`, `E`, `TAU`, `INF`, `NAN` |

Fósiles de math:

- `rt_math_sqrt` y `rt_f64_to_i64` en `stats/mod.mr:32-33` **no están en la allowlist** y duplican funciones que deberían ser Mire puro.
- `power` y `int::pow` implementan lo mismo con estrategias distintas.
- `math/mod.mr` documenta que las reducciones devuelven `f64` porque `vec` no soporta `f64`; eso es un límite de `mire::vec`, no de math.

Tests: un archivo por símbolo; casos de overflow (`i64::MAX + 1`), NaN, `-0.0`, precisión (`0.1 + 0.2`), y `random` con semilla fija para reproducibilidad.

### 9.13 `kioto::crypto`

**Estado [verificado]:**

| Submódulo | Archivos | Dependencia |
|-----------|----------|-------------|
| `hash/sha256` | 9 LOC | `rt_crypto_sha256_hex` (libsodium) |
| `hash/sha512` | 9 LOC | `rt_crypto_sha512_hex` |
| `hash/_externs` | 8 LOC | 5 externs |
| `encode/hex` | 48 LOC | Mire |
| `encode/base64` | 85 LOC | Mire + `rt_crypto_base64_file` |
| `checksum` | 91 LOC | `rt_crypto_sha512_file_hex` |
| `random/secure` | 68 LOC | `rt_*` |
| `sign/ed25519` | 110 LOC | 3 externs libsodium |

Problemas:

1. **Dependencia de libsodium** en el runtime: es la causa de que los binarios precompilados de `owl`/`avenys` fallaran (“faltaba libsodium”). Un binario del gestor no debería necesitar crypto.
2. Todos los `rt_crypto_*` están fuera de la allowlist.
3. `sha256.mr`/`sha512.mr` son envoltorios de 9 líneas; la implementación no está en Mire.
4. `random/secure` no documenta si es CSPRNG ni de dónde sale la entropía.

Plan:

- **Corto plazo:** sacar crypto del runtime base. `kioto::crypto` declara su dependencia nativa y `--bundle static` la embebe (§10.1). Los binarios de `owl`/`avenys` dejan de enlazar libsodium.
- **Mediano plazo:** SHA-256 y SHA-512 en Mire puro (~200 líneas cada uno) con tests vectoriales NIST. HMAC encima.
- **Largo plazo:** `ed25519` puede quedarse como dependencia nativa opcional o migrarse a un paquete `crypto-native`; no se reimplementa a mano sin una revisión criptográfica.
- `base64`/`hex` deben moverse a `ilogical::encoding` y `kioto::crypto` re-exportar.

Tests: vectores NIST para SHA-256/512 (`""`, `"abc"`, 1 MiB); hex round-trip; base64 con padding, sin padding, URL-safe; checksum de archivo vacío y de 1 byte; ed25519 con clave de prueba; `random::secure` no devuelve dos veces lo mismo.

### 9.14 `kioto::json` y `kioto::toml` — eliminación

**Decisión D5.** Ambos módulos son wrappers de `c_json_*` / `c_toml_*` (objetos C de `mire/code/`).

Plan:

1. `ilogical::format::{json,toml}` implementa los codecs (§8).
2. `kioto::json` y `kioto::toml` **se eliminan** del repo y de `[exports]`.
3. Si un consumidor los usa, Owl emite `E0025 unresolved_import` con fix-it:
   ```
   help: `kioto::json` fue eliminado en Kioto 4.0; usa `ilogical::format::json`
   ```
4. Durante un release de convivencia, pueden existir como archivos de 3 líneas que re-exportan; **no** como stubs vacíos.

### 9.15 Estructura de tests de Kioto

```text
kioto/tests/
├── mod.mr
├── _support/
│   ├── harness.mire        # assert_eq, expect_err, measure
│   ├── fixture.mire        # tmp dir aislado, cleanup
│   └── log.mire            # anomalies.log / metrics.json
├── async/
│   ├── mod.mr
│   ├── symbols/
│   │   ├── mod.mr
│   │   ├── task.mire
│   │   ├── channel.mire
│   │   ├── spawn.mire
│   │   ├── await.mire
│   │   ├── sleep.mire
│   │   ├── timeout.mire
│   │   ├── interval.mire
│   │   ├── instant.mire
│   │   ├── run.mire
│   │   └── runtime.mire
│   ├── logs/.gitkeep
│   └── fixtures/tmp/.gitkeep
├── time/symbols/{now,mono,sleep,elapsed,mark,fmt,parse}.mire
├── strings/symbols/{to,from,index,char,substr,split,join,trim}.mire
├── fs/symbols/{open,read,write,copy,rename,metadata,sandbox}.mire
├── proc/symbols/{run,spawn,output,shell,exit}.mire
├── net/symbols/{dns,socket,listener,connect,accept,timeout,ws}.mire
├── mem/symbols/{total,available,process,stats,map}.mire
├── cpu/symbols/{count,info,features}.mire
├── env/symbols/{var,dir,args}.mire
├── cli/symbols/{parse,help,usage}.mire
├── log/symbols/{level,info,warn,error,json}.mire
├── math/symbols/{int,float,trig,power,random,seq,stats,sum,consts}.mire
├── crypto/symbols/{sha256,sha512,hex,base64,checksum,ed25519,random}.mire
└── security/allowlist.mire
```

Cada `symbols/<x>.mire` contiene:

```mire
load kioto
load kioto::tests::_support::harness

// Constantes medibles: el resultado esperado está en el nombre.
cons CASE_SLEEP_10_MS_MIN = 10
cons CASE_SLEEP_10_MS_MAX = 200

pub fn main: () {
    // Éxito
    set t0 = time::mark()
    use time::sleep(time::Duration { ms: 10 })
    set dt = time::elapsed(t0)
    harness::assert::between("sleep/10ms", dt.ms, 10, 200)

    // Límite
    harness::assert::ok("sleep/0ms", time::sleep(time::Duration { ms: 0 }))

    // Anomalía: debe dar un error específico, no abortar.
    match time::sleep(time::Duration { ms: -1 }) {
        Result.Ok(_)  { harness::fail("sleep/negative", "expected error, got ok") }
        Result.Err(e) { harness::assert::eq("sleep/negative/code", e.code, "time/invalid_duration") }
    }
}
```

El harness escribe en `logs/anomalies.log` una línea por caso anómalo:

```text
ts=2026-11-13T12:00:00Z suite=async/sleep case=negative_duration input=-1 expected=err(time/invalid_duration) got=err(time/invalid_duration) ok=true
```

Y en `logs/metrics.json`:

```json
{"suite":"async/sleep","cases":3,"passed":3,"failed":0,"duration_ms":14}
```

`logs/` se versiona solo con `.gitkeep`; los archivos generados van a `.gitignore`.

---

## 10. Paquetes satélite

### 10.1 Binarios autosuficientes: `--bundle`

**Motivación real:** al instalar los binarios precompilados de `owl` y `avenys`, ambos funcionaban pero **faltaba `libsodium`**, y ninguno arrancó. Es un fallo de distribución, no del usuario.

Diagnóstico:

- `kioto::crypto` enlaza `rt_crypto_*`, que vive en el runtime C, que a su vez enlaza `libsodium.so`.
- `owl` y `avenys` cargan `kioto` en su propio build, así que heredan la dependencia.
- Un gestor de paquetes no debería necesitar crypto nativa para arrancar.

Semántica de `--bundle`:

1. `static` (default) busca `lib<name>.a` en `~/.owl/libs/<pkg>/native/lib`, `--link-dir` y directorios del sistema; si existe, enlaza estático.
2. Si no hay `.a` pero sí `.so`, emite W0069 `static_bundle_fallback`, enlaza dinámico y registra `bundle_resolved: static-partial` con cada dependencia dinámica.
3. Si no existe ninguno, falla con E0404 `native_dependency_missing` e instrucciones de instalación.
4. `static-strict` nunca hace fallback y falla con E0404; es obligatorio en releases de Owl/Avenys y en el gate `scratch`. `dynamic` solicita enlace dinámico explícito.

| Capa | Qué se embebe | Cómo |
|------|---------------|------|
| `--bundle static` | `.a` disponible | Preferencia estática, con fallback W0069 y manifest parcial |
| `--bundle static-strict` | Todas las deps requieren `.a` | Sin fallback; E0404 si falta el archivo |
| `--bundle dynamic` | Nada; deja `.so` | Enlace dinámico explícito |
| `owl` / `avenys` | **Nada de crypto** | Crypto se mueve a `kioto::crypto` como dep opcional |
| `mire doctor` | Detecta `.so` faltantes | Equivalente a `ldd` sin shell |

Ruta concreta para libsodium:

1. Sacar `rt_crypto_*` del runtime base.
2. `kioto::crypto` declara `[native] libs = ["sodium"]` y `--bundle static` la embebe desde `libsodium.a`.
3. Los binarios de `owl`/`avenys` dejan de enlazarla.
4. `mire doctor` diferencia `.a` ausente con `.so` disponible (W0069 y opciones) de dependencia ausente (E0404 con instrucciones por distro).
5. A medio plazo, SHA-256/512 en Mire puro eliminan la dep para el caso más común.

Esto resuelve dependencias nativas de programas Mire; **no** resuelve el enlace de Avenys con LLVM 22, que pertenece a la distribución del toolchain Rust.

**Regla de aceptación:** en `scratch`, ejecutar `owl --version`, `mire --version`, `owl build` y `mire build` de un hello-world que haga `load kioto::crypto`; releases usan `--bundle static-strict` y ambos binarios deben arrancar.

Mensaje de doctor cuando solo existe la biblioteca dinámica:

```text
warning[W0069]: no static archive for `sodium`; linking dynamically
  = note: `--bundle static` prefers `.a`; only `libsodium.so` was found
  = help: install the static archive (Debian: `sudo apt install libsodium-dev`)
  = help: or request dynamic linking explicitly: `--bundle dynamic`
  = help: use `--bundle static-strict` to fail instead of falling back in CI
```

Si no existe ni `.a` ni `.so`, emitir E0404 con instrucciones equivalentes para Arch, Debian y Fedora.

### 10.2 `libs/`

**Estado [verificado]:** 11 archivos.

```text
libs/
├── docs/{CREATING-LIBRARIES.md,KEY-ROTATION.md}
├── packages/kioto-2.3.2.tar.zst
├── packages/sdl-1.0.1.tar.zst
├── reg/{registry.toml,registry.json,index.toml,index.toml.sig}
├── README.md
└── LICENSE
```

Problemas:

- Tarballs binarios versionados en el repo (peso, caducidad).
- `kioto-2.3.2` y `sdl-1.0.1` no son las versiones actuales.
- `index.toml.sig` sin documentar el proceso de firma en CI.

Plan:

1. Sacar `packages/*.tar.zst` del repo; publicarlos como release assets.
2. `libs/reg/` mantiene solo el índice y su firma; un script `scripts/regenerate-index.sh` los produce.
3. `docs/CREATING-LIBRARIES.md` se amplía con: layout, `owl.toml` mínimo, `[exports]`, tests, versionado, firma.
4. `docs/KEY-ROTATION.md` se enlaza desde `owl doctor`.
5. CI: publicar índice firmado por release.

### 10.3 `wasm/`

**Estado [verificado]:** 11 archivos.

```text
wasm/
├── core/{host,fetch,storage,canvas,memory}/mod.mire
├── code/mod.mire
├── bin/mod
├── mire-config-wasm.toml
├── owl.toml
└── owl.lock
```

Problemas:

- `mire-config-wasm.toml` viola el nuevo modelo (Avenys no lee config).
- `bin/mod` es un artefacto.
- `core/*` depende de `lib "pal"` y `lib "c"`; no hay PAL WASI (`avenys/src/pal/wasi/` está vacío).
- `wasm/tests/` no existe.

Plan:

1. Borrar `bin/` y `mire-config-wasm.toml`.
2. Marcar el paquete como `experimental` en `status()` y en `owl.toml`.
3. No exportar módulos que no tengan implementación real.
4. `core/memory` y `core/host` son los únicos con posibilidad de implementarse sobre `--target wasm32-*`.
5. `core/fetch`, `storage`, `canvas` quedan en el plan, sin archivo, hasta que exista host API.

### 10.4 `sqlite/`

**Estado [verificado]:** 61 archivos.

```text
sqlite/code/
├── mod.mire              # bindings FFI reales
├── test_c_func.mire
├── test_main.mire
├── test_minimal.mire
├── test_no_ffi.mire
├── test_open.mire
├── test_open_only.mire
├── test_pointer.mire
├── test_output.mire
├── test_simple_c.mire
└── test_simple.mire
```

**Decisión D7:** se borran los 11 `test_*.mire`. Lo que valga la pena se reescribe en `sqlite/tests/`.

Plan:

1. `sqlite/code/mod.mire` se mantiene; se audita que cada `extern` esté en `[security].externs`.
2. `sqlite/tests/` con estructura por símbolo:
   ```text
   sqlite/tests/
   ├── mod.mr
   └── symbols/{open,exec,prepare,step,column,bind,close}.mire
   ```
3. `sqlite/tests/log/` se mueve a `logs/` y se ignora.
4. `owl.toml` `[c] libs = ["sqlite3"]` se mantiene; `--bundle static` embebe `libsqlite3.a` si está.
5. `extern-libs = ["sqlite3","c"]` se valida contra la allowlist.

### 10.5 `sdl/`

**Estado [verificado]:** 304 archivos, tests existentes (`smoke`, `headless`, `sdl2_headless`, `sdl3_*`, `namespace_regression`, `structs_smoke`).

Plan:

- Mantener.
- Migrar tests a `tests/symbols/…` con fixtures headless.
- Confirmar que `[security]` cubre todos los `extern "SDL_*"`.
- `--bundle static` debe poder embeber SDL3 si hay `.a`.

### 10.6 Paquetes de aplicación

| Paquete | Archivos | Estado | Decisión |
|---------|----------|--------|----------|
| `blu` | 28 | App de ejemplo | Mantener como `examples/blu` o repo aparte |
| `token` | 203 | App | Mantener; borrar `bin/` |
| `stress` | 1095 | Bench/stress | Mantener; borrar `backups/` |
| `websocket-chat` | 146 | Demo | Mantener; borrar `chat.db` y datos generados |
| `MireData` | 355 | Datos de ejemplo | Mantener como fixture de tests, no como paquete |
| `mire-lang` | 0 | Vacío | Borrar |
| `mireLSP` | 0 | Vacío | Borrar o inicializar submodule |
| `mire-lang.github.io` | 12 | Web | Mantener |
| `grammar` | 7 | Gramática TextMate | Mantener |
| `linguist` | 4275 | Tooling externo | Mantener como herramienta, no como librería |
| `docker-tests` | 6457 | Copias vendorizadas | Revisar; probablemente borrar y usar CI |
| `registry` | 10 | Servidor de registro | Mantener |
| `testproj` | 0 | Vacío | Borrar |
| `docs/avenys` | 0 | Vacío | Borrar |

---

## 11. Sistema de tests

### 11.1 Contrato por símbolo

Todo símbolo público tiene **tres** casos como mínimo:

1. **Éxito:** entrada conocida → salida exacta.
2. **Límite:** vacío, cero, máximo, mínimo, NaN, UTF-8 inválido, buffer justo.
3. **Anomalía:** entrada que el símbolo no debería recibir → error **específico**, sin abortar, sin desbordar, sin escribir fuera de `fixtures/tmp/`.

Un test que solo hace `return true` no es un test. Un test que compara contra sí mismo tampoco.

### 11.2 Aislamiento

`mire test` y `owl test` deben:

1. Crear `tests/<suite>/fixtures/tmp/<pid>/`.
2. Hacer `chdir` ahí antes de la suite.
3. Ejecutar la suite.
4. Borrar el directorio al terminar, incluso si falla.
5. Restaurar el `cwd` anterior.

Ningún test puede escribir en la raíz del repo. Si lo hace, el runner falla con `E0705 test_polluted_workspace`.

### 11.3 Runner

```bash
mire test [paths...] [options]

  --release, -r       Perfil release
  --opt-level, -O     Nivel
  --jobs, -j          Paralelismo
  --filter <substr>   Solo tests que matcheen
  --list              Lista sin ejecutar
  --no-run            Solo compila
  --verbose, -v
  --format human|json|junit
  --log               Escribe logs/metrics.json
  --snapshot          Actualiza goldens
  --fail-fast
  --test-threads <n>
```

Salida humana:

```text
running 42 tests
test async/sleep/positive ......... ok (12 ms)
test async/sleep/zero ............. ok (0 ms)
test async/sleep/negative ......... ok (1 ms)
test async/timeout/expires ........ FAILED (105 ms)

---- async/timeout/expires ----
expected: Err(timeout)
   got:  Ok(0)
   at:   kioto/tests/async/symbols/timeout.mire:34

failures:
    async/timeout/expires

test result: FAILED. 41 passed; 1 failed; 0 ignored
```

### 11.4 Política de logs

| Archivo | Contenido | Versionado |
|---------|-----------|------------|
| `logs/anomalies.log` | Una línea por caso anómalo | No (`.gitignore`) |
| `logs/metrics.json` | Suite, casos, duración | No |
| `logs/golden/*.txt` | Salidas esperadas | Sí |

Formato de anomalía (estable, parseable):

```text
ts=<ISO8601> suite=<suite> case=<case> input=<repr> expected=<repr> got=<repr> ok=<bool>
```

### 11.5 Tests de rendimiento

- `avenys/bench/kernels/` se conserva.
- `avenys/bench/BASELINE.md` congela el baseline por kernel.
- CI corre `bench/run.sh` y falla si un kernel regresiona > 2 %.
- `mire test --release -O3` es el perfil de los tests de rendimiento.

### 11.6 Tests de diagnósticos

Ver §13.18. Resumen:

- `avenys/tests/diagnostics/cases/<code>/<caso>.mire` + `.stderr` + `.json`.
- Comparación de texto **y** de esquema JSON.
- Cada `DiagnosticCode` debe tener al menos un caso.

### 11.7 Cobertura por símbolo

Se publican dos métricas distintas. **M1 — presencia** es el porcentaje de símbolos públicos exportados que aparecen en al menos un test con `@[covers(pkg::mod::sym)]`. El denominador son todos los `pub fn` alcanzables desde `[exports]`, incluidos los anidados (`strings::from::i64`); se excluyen `main`, nombres con prefijo `_` y símbolos bajo `@[no_main]`. El numerador se obtiene exclusivamente del atributo explícito, nunca del análisis de llamadas.

**M2 — contrato** es el porcentaje de esos símbolos para los que hay casos etiquetados de éxito, límite y anomalía, o una exención aprobada vigente. Los casos imposibles (por ejemplo `cpu::count` no recibe entrada o `status()` no tiene anomalía) se declaran en `tests/coverage-waivers.toml` con `symbol`, `reason`, `owner` y `until`; CI falla si vencen. Los casos se etiquetan para que el runner pueda verificar cada dimensión.

| Hito | M1 — presencia | M2 — contrato |
|-------|----------------|----------------|
| F1 | ≥ 40 % | — |
| F4 | ≥ 70 % | — |
| F6 (gate) | 100 % | ≥ 60 % |
| F8 (gate final) | 100 % | ≥ 90 %, con ≤ 10 % de waivers justificados |

No se usa cobertura de líneas ni `llvm-cov` como criterio de aceptación. El recolector de atributos, inventario de exports y validador de expiración se implementan en F0.

---

## 12. Reset de repositorios

> **Regla:** fuera artefactos generados, caches, binarios, fixtures de test en raíz y stubs de una línea. Dentro: fuente, docs, tests, scripts, `owl.toml`, `owl.lock`, licencias.

### 12.1 Inventario de la raíz

**[verificado]** Conteo de archivos excluyendo `.git`:

| Directorio | Archivos | Git | Decisión |
|------------|----------|-----|----------|
| `avenys/` | 16 132 | Sí | Mantener fuente/docs/tests/bench; borrar `target/`, `bin/`, `src/.cobject_cache/` |
| `bin/` | 0 | — | **Borrar**; `.gitignore` |
| `blu/` | 28 | Sí | Mover a `examples/blu` o repo aparte |
| `docker-tests/` | 6 457 | No | **Revisar**: copias vendorizadas; probable borrado |
| `docs/` | 0 | — | **Borrar** (solo `docs/avenys` vacío) |
| `grammar/` | 7 | Sí | Mantener |
| `ilogical/` | 133 | No | Mantener; **inicializar git** |
| `kioto/` | 425 | Sí | Mantener; borrar junk dirs + `bin/` |
| `libs/` | 11 | Sí | Mantener; sacar tarballs |
| `linguist/` | 4 275 | Sí | Mantener como tooling |
| `mire/` | 87 | Sí | Mantener; borrar `code/`, caches |
| `MireData/` | 355 | Sí | Mantener como fixture; borrar `bin/` |
| `mire-lang/` | 0 | No | **Borrar** |
| `mire-lang.github.io/` | 12 | Sí | Mantener |
| `mireLSP/` | 0 | No | **Borrar** o inicializar |
| `owl/` | 803 | Sí | Mantener; borrar `bin/`, `.owl/` |
| `registry/` | 10 | Sí | Mantener |
| `sdl/` | 304 | Sí | Mantener; borrar `bin/` |
| `sqlite/` | 61 | Sí | Mantener; borrar `code/test_*.mire` |
| `stress/` | 1 095 | Sí | Mantener; borrar `backups/` |
| `testproj/` | 0 | No | **Borrar** |
| `token/` | 203 | Sí | Mantener; borrar `bin/` |
| `wasm/` | 11 | Sí | Mantener; borrar `bin/`, config TOML |
| `websocket-chat/` | 146 | Sí | Mantener; borrar `chat.db`, datos |

### 12.2 `kioto/` — detalle

| Elemento | Acción |
|----------|--------|
| `modules/`, `src/`, `docs/`, `scripts/`, `tests/` | **Mantener** |
| `owl.toml`, `owl.lock`, `README.md`, `CHANGELOG.md` | Mantener |
| `bin/` | **Borrar** + `.gitignore` |
| `kioto-fsw-empty/` | **Borrar** (basura de test) |
| `kioto-fsw-ok/` | **Borrar** |
| `reading_through_an_escaping_symlink_returns_nothing/` | **Borrar** (symlink a `/etc`) |
| `relative_paths_inside_the_root_are_allowed/` | **Borrar** |
| `symlink_pointing_out_of_the_root_is_denied/` | **Borrar** |
| `symlink_that_stays_inside_the_root_is_allowed/` | **Borrar** |
| `todo.md` | Revisar; archivar en `docs/` o borrar |
| `modules/math/{complex,decimal,hyperbolic,special}/` | **Borrar** (stubs, §9.12) |
| `modules/json/`, `modules/toml/` | **Borrar** (D5, §9.14) |
| `kioto = { path = "." }` en `owl.toml` | **Eliminar** |

**Origen de la basura [verificado]:** `kioto/tests/fs/sandbox/mod.mr` crea los fixtures bajo el `cwd`, que al ejecutar `owl test` es la raíz del repo. El fix está en §11.2.

### 12.3 `mire/` — detalle

| Elemento | Acción |
|----------|--------|
| `src/alloc/`, `src/std/`, `src/aux/`, `src/lib.mire` | Mantener |
| `tests/` | Mantener; recuperar/o eliminar `json_test.mire` |
| `bin/`, `src/alloc/bin/.cache/` | **Borrar** + `.gitignore` |
| `code/` completo (`json_parser.c`, `toml_parser.c`, `mod.mire`) | **Borrar** tras §8 |
| `[c]` en `owl.toml` | **Borrar** |
| `src/aux/pal.mire` | Migrar (§7.3) |

### 12.4 `avenys/` — detalle

| Elemento | Acción |
|----------|--------|
| `src/` (excepto lo listado) | Mantener |
| `tests/`, `bench/`, `examples/`, `docs/`, `scripts/`, `install/` | Mantener |
| `src/.cobject_cache/` | **Borrar**; mover a `bin/.cache/cobjects` |
| `bin/`, `target/` | **Borrar** + `.gitignore` |
| `src/pal/core/pal_optional.c` | **Borrar** |
| `src/pal/core/*.c`, `src/pal/linux/pal_linux.c` | Migrar a Mire (§5.5) |

### 12.5 `owl/` — detalle

| Elemento | Acción |
|----------|--------|
| `src/`, `docs/`, `scripts/`, `tests/`, `native/`, `owl/` | Mantener |
| `bin/`, `.owl/` | **Borrar** |
| `src/lockfile` duplicado | Refactor (§6.2) |
| `src/_externs` | Migrar a `mire::pal` |
| `src/{server,trust,upgrade}` | Revisar alcance; documentar o mover |

### 12.6 `.gitignore` mínimo por repo

```gitignore
/bin/
/target/
/.cache/
**/bin/.cache/
**/fixtures/tmp/
**/logs/*.log
**/logs/metrics.json
*.o
*.so
*.a
!libs/**/*.a
mire-config-*.toml
*.manifest.json
```

### 12.7 Script de reset

Un script `scripts/reset-tree.sh` (dry-run por defecto) implementa §12 con confirmación explícita:

```bash
scripts/reset-tree.sh --dry-run       # lista qué borraría
scripts/reset-tree.sh --apply kioto   # borra solo kioto
scripts/reset-tree.sh --apply all
```

Nunca toca `.git/`, `src/`, `docs/`, `tests/`, `scripts/`.

---

## 13. Diagnósticos, errores, warnings y ayuda

> Este es el capítulo de mayor inversión del plan. Un compilador se juzga por cómo explica lo que no entiende. Hoy Avenys tiene una base decente (códigos, spans, cajas Unicode) pero con drift, cobertura parcial y sin formatos machine-readable. El objetivo es pasar de “mensaje que señala” a **“mensaje que enseña y corrige”**.

### 13.1 Auditoría del estado actual

| Componente | Archivo | Estado | Problema |
|------------|---------|--------|----------|
| Códigos | `error/diagnostic.rs:30-98` | Enum plano | Sin metadatos; `category()` parcial; sin deprecación |
| Nombres | `diagnostic.rs:name()` | snake_case | Bien, pero no se muestran al usuario en errores |
| Severidad | `Severity` | 4 variantes | Bien |
| Span | `error/mod.rs:15-52` | `line, column` | **Sin byte offsets**, sin longitud, sin archivo |
| Labels | `diagnostic.rs:Label` | `span + length` | Solo una línea; sin multi-línea; sin multi-archivo |
| Render | `error/format.rs` | Caja Unicode | Hand-rolled; sin tabs, sin ancho Unicode, sin truncado |
| Fuente | `format.rs:52-65` | Lee el archivo en cada render | Sin `SourceMap`; falla si el path cambió |
| Help | `error/kind.rs:default_help_for_code` | Solo E0005, E0014, E01xx | La mayoría sin ayuda |
| Ownership | `error/mss.rs` | 8 mensajes genéricos | “MSS Error: Use after move - value was moved”; sin variable ni segundo sitio |
| CLI help | `cli/mod.rs:44-106` | `println!` | Pared de texto; sin `mire help <cmd>`; sin exit codes |
| Warnings | `cli/warnings.rs` | Resumen por nombre | Sin grupos, sin niveles, sin JSON |
| Docs | `docs/errors/README.md` | 498 líneas | **Drift**: E0008 documentado como “Cannot Infer Type”, implementado como `MultipleMutableRefs` |
| Tests | `error/mod.rs` tests | `contains("25:12")` | No validan estructura ni calidad |
| JSON | — | No existe | Sin integración con editores/CI |

**Drift documentado [verificado]:**

| Código | Doc dice | Código hace |
|--------|----------|-------------|
| E0008 | “Cannot Infer Type” | `MultipleMutableRefs` (ownership) |
| E0009 | “Expected Expression” | `MutationWhileShared` |
| Formato | `error[E0005] Type Error` | `error[E0005] ── Type Error` con caja |

Eso no es cosmético: un usuario que busca E0008 en los docs recibe la explicación equivocada.

### 13.2 Objetivos

| # | Objetivo | Métrica |
|---|----------|---------|
| O1 | Todo error tiene código, span y ayuda | 100 % de los códigos con `help` no vacío |
| O2 | Todo error apunta al sitio exacto | spans de byte, no línea/columna |
| O3 | Todo error dice **qué** y **por qué** | mensaje + nota de causa |
| O4 | Todo error ofrece **cómo arreglarlo** | ≥ 60 % con fix-it machine-applicable |
| O5 | El usuario puede profundizar | `mire explain <code>` |
| O6 | Las herramientas pueden consumirlo | `--error-format json` estable |
| O7 | La ayuda no miente | generada del mismo spec que el parser |
| O8 | Los warnings son accionables | grupos, niveles, `@[allow]` |
| O9 | Un ICE es reportable | código E09xx + `mire bug` |
| O10 | El texto es consistente | guía de estilo + lint de mensajes |

### 13.3 Modelo de datos

#### 13.3.1 De `Span(line, column)` a `ByteSpan`

```rust
// error/source.rs
pub struct FileId(pub u32);

pub struct ByteSpan { pub start: u32, pub end: u32 }

pub struct SourceFile {
    pub id: FileId,
    pub path: PathBuf,
    pub text: Arc<str>,
    pub line_starts: Vec<u32>,   // offsets, calculados una vez
}

pub struct SourceMap {
    files: Vec<SourceFile>,
    by_path: HashMap<PathBuf, FileId>,
}

impl SourceMap {
    pub fn add(&mut self, path: PathBuf, text: String) -> FileId;
    pub fn lookup(&self, file: FileId, offset: u32) -> LineCol; // 1-based
    pub fn line_text(&self, file: FileId, line: u32) -> &str;
    pub fn span_text(&self, file: FileId, span: ByteSpan) -> &str;
}
```

Reglas:

- Todo diagnóstico nuevo usa `ByteSpan`.
- `Span(line, column)` se conserva como tipo de compatibilidad y se calcula desde `ByteSpan` al renderizar.
- Los spans de un solo punto (`end == start`) son válidos y se renderizan con un caret.
- Los spans multilínea se renderizan con bloque.

#### 13.3.2 `Diagnostic` enriquecido

```rust
pub struct Diagnostic {
    pub severity: Severity,
    pub code: DiagnosticCode,
    pub message: String,             // “expected `i64`, found `str`”
    pub labels: Vec<Label>,          // 1+ primary, N secondary
    pub notes: Vec<Note>,            // causa, historia, macro
    pub help: Option<Help>,          // cómo arreglarlo
    pub fixes: Vec<Fix>,             // edits aplicables
    pub origin: Option<Origin>,      // cadena de macro/load
    pub docs: Option<&'static str>,  // slug de explicación
}

pub struct Label {
    pub file: FileId,
    pub span: ByteSpan,
    pub style: LabelStyle,           // Primary | Secondary
    pub message: String,
}

pub struct Note { pub message: String, pub kind: NoteKind }
pub enum NoteKind { Cause, Expansion, History, Compatibility }

pub struct Help {
    pub message: String,
    pub code: Option<String>,        // ejemplo de código
}

pub struct Fix {
    pub applicability: Applicability,
    pub message: String,
    pub edits: Vec<Edit>,
}

pub enum Applicability {
    MachineApplicable,   // seguro de aplicar
    MaybeIncorrect,      // heurístico
    HasPlaceholders,
    Unspecified,
}

pub struct Edit {
    pub file: FileId,
    pub span: ByteSpan,
    pub replacement: String,
}

pub struct Origin {
    pub kind: OriginKind,            // Macro | Load | Derive | Generated
    pub file: FileId,
    pub span: ByteSpan,
    pub name: String,
}
```

#### 13.3.3 `SourceMap` global

Hoy `format.rs` hace `std::fs::read_to_string` por cada diagnóstico. Eso:

- Rompe si el archivo se movió.
- Es lento con muchos warnings.
- No puede resolver `#include`/`load` de macros.

`SourceMap` se construye al inicio del build y se pasa a todos los emisores. El renderer nunca toca el filesystem.

### 13.4 Taxonomía de códigos

#### 13.4.1 Reglas

1. **Un código nunca se reutiliza.** Se deprecia, no se reasigna.
2. **Un código = una causa.** No `E0015` para todo runtime.
3. **El nombre es estable** (`snake_case`) y se muestra en warnings: `warning[W0001 unused_variables]`.
4. **Un código tiene una explicación** con ejemplos malo/bueno.
5. **Un código tiene al menos un test** que lo dispara.

#### 13.4.2 Rangos

| Rango | Dominio | Estado |
|-------|---------|--------|
| `E0001–E0099` | Léxico, sintaxis, parser | Existente; completar |
| `E0100–E0199` | Tipos y conversiones | Existente |
| `E0200–E0299` | Ownership / MSS | **Nuevo**; alias de E0007–E0013 |
| `E0300–E0399` | Backend / codegen | **Nuevo** |
| `E0400–E0499` | Enlazado / linker | **Nuevo** |
| `E0500–E0599` | Cache / incremental | **Nuevo** |
| `E0600–E0699` | Paquetes / resolución | **Nuevo** |
| `E0700–E0799` | CLI / uso | **Nuevo** |
| `E0800–E0899` | Runtime / PAL | **Nuevo** |
| `E0900–E0999` | ICE | **Nuevo** |
| `W0001–W0049` | Lints existentes | Existente |
| `W0050–W0099` | Lints nuevos | **Nuevo** |

#### 13.4.3 Catálogo nuevo (preliminar)

**Ownership (E02xx)**

| Código | Nombre | Cuándo |
|--------|--------|--------|
| E0201 | `use_after_move` | Alias de E0007; añade “moved here” |
| E0202 | `multiple_mutable_borrows` | Alias de E0008 |
| E0203 | `mutate_while_borrowed` | Alias de E0009 |
| E0204 | `move_while_borrowed` | Alias de E0010 |
| E0205 | `drop_while_borrowed` | Alias de E0011 |
| E0206 | `borrow_outlives_owner` | Alias de E0013 |
| E0207 | `return_local_reference` | Devolver `&` a local |
| E0208 | `partial_move` | Mover un campo y usar el todo |

**Backend (E03xx)**

| Código | Nombre |
|--------|--------|
| E0301 | `unsupported_construct` |
| E0302 | `target_unsupported` |
| E0303 | `codegen_failed` |
| E0304 | `cpu_feature_unsupported` |

**Link (E04xx)**

| Código | Nombre | Caso real |
|--------|--------|-----------|
| E0401 | `undefined_symbol` | Símbolo no encontrado; cita paquete/archivo |
| E0402 | `library_not_found` | `-l sodium` sin `.so`/`.a` |
| E0403 | `library_hash_mismatch` | `-H` no coincide |
| E0404 | `native_dependency_missing` | **libsodium** |
| E0405 | `duplicate_symbol` | Dos libs exportan lo mismo |
| E0406 | `abi_mismatch` | Runtime vs stdlib |

**Cache (E05xx)**

| Código | Nombre | Acción |
|--------|--------|--------|
| E0501 | `cache_schema_mismatch` | Cuarentena automática (warning) |
| E0502 | `cache_corrupt` | Cuarentena del blob |
| E0503 | `cache_write_failed` | Permisos/FS |
| E0504 | `cache_lock_timeout` | Otro proceso |

**Paquetes (E06xx)**

| Código | Nombre |
|--------|--------|
| E0601 | `package_not_found` |
| E0602 | `version_conflict` |
| E0603 | `lockfile_out_of_sync` |
| E0604 | `registry_unreachable` |
| E0605 | `checksum_mismatch` |
| E0606 | `circular_dependency` |

**CLI (E07xx)**

| Código | Nombre |
|--------|--------|
| E0701 | `unknown_flag` |
| E0702 | `missing_value` |
| E0703 | `invalid_value` |
| E0704 | `conflicting_flags` |
| E0705 | `unknown_subcommand` |
| E0706 | `test_polluted_workspace` |

**Runtime/PAL (E08xx)**

| Código | Nombre |
|--------|--------|
| E0801 | `syscall_failed` |
| E0802 | `sandbox_denied` |
| E0803 | `pal_unavailable` |
| E0804 | `out_of_memory` |

**ICE (E09xx)**

| Código | Nombre |
|--------|--------|
| E0901 | `internal_compiler_error` | Panic Rust |
| E0902 | `ice_assertion` | Assertion interna |
| E0903 | `worker_panic` | Worker hijo terminó por panic/señal |
| E0904 | `fatal_signal` | SIGSEGV/SIGBUS/SIGILL directa |
| E0905 | `stack_overflow` | Guard page / stack agotado |

**Warnings nuevos (W005x)**

| Código | Nombre | Grupo |
|--------|--------|-------|
| W0050 | `unused_must_use` | correctness |
| W0051 | `unused_result` | correctness |
| W0052 | `unreachable_code` | correctness |
| W0053 | `unused_extern` | unused |
| W0054 | `stale_security_allowlist` | correctness |
| W0055 | `clock_regression` | suspicious |
| W0056 | `shell_invocation` | security |
| W0057 | `redundant_clone` | performance |
| W0058 | `needless_borrow` | style |
| W0059 | `deprecated_api` | deprecated |
| W0060 | `implicit_narrowing` | correctness |
| W0061 | `legacy_link_flag` | deprecated |
| W0062 | `cache_schema_migrated` | note |
| W0063 | `shadowed_builtin` | suspicious |
| W0064 | `unsafe_block` | security |

### 13.5 Renderizado humano

#### 13.5.1 Anatomía

```text
error[E0005]: expected `i64`, found `str`
  --> src/main.mire:7:18
   │
 6 │ pub fn main: () {
 7 │     set x :i64 = "hola"
   │                  ^^^^^^ expected `i64`
   │
   = note: `str` cannot be coerced to `i64` implicitly
   = help: use `str::to::i64` and handle the error
   = fix: replace `"hola"` with `str::to::i64("hola")` (machine-applicable)

error: could not compile `main` due to 1 previous error
```

Reglas de composición:

1. **Cabecera:** `severity[CODE]` para errores; `severity[CODE name]` para warnings/lints.
2. **Ubicación:** `--> file:line:col` con la columna del **primer byte** del span.
3. **Gutter:** ancho = dígitos de la línea máxima mostrada.
4. **Contexto:** 2 líneas antes y 2 después del span; si el span es multilínea, se muestra completo con elipsis si > 12 líneas.
5. **Marcadores:**
   - Primario: `^` (o `~` en spans multilínea).
   - Secundario: `-`.
   - Longitud exacta del span; mínimo 1.
6. **Etiquetas:** el mensaje va a la derecha del marcador; si hay varias en la misma línea, se apilan.
7. **Notas:** `= note: …`.
8. **Ayuda:** `= help: …`.
9. **Fix:** `= fix: … (machine-applicable)` solo si existe.
10. **Resumen final:** una línea, no un párrafo.

#### 13.5.2 Tabs, Unicode y líneas largas

| Caso | Regla |
|------|-------|
| Tab | Expandir a 4 espacios; dibujar `→` tenue en la guía |
| Ancho Unicode | Usar `unicode-width` para el caret; un emoji ocupa 2 |
| Caracteres de control | Escapar como `\xNN` en el render, mantener el byte en el span |
| Línea > 200 columnas | Truncar con `…` a los lados del span y nota `(column 4312)` |
| CRLF | Tratar `\r\n` como un salto; no contar `\r` |
| BOM | Saltar; no desplazar columnas |
| Archivo vacío | `--> file:1:1` + `(empty file)` |

#### 13.5.3 Multi-línea

```text
error[E0201]: use of moved value: `data`
   --> src/main.mire:12:9
    │
 10 │     set data = vec::new() :vec[i64]
    │         ---- `data` moved here
 11 │     consume(data)
 12 │     use data
    │         ^^^^ value used after move
    │
    = help: borrow instead: `consume(&data)`
```

Implementación: si `span.start_line != span.end_line`, se dibuja:

```text
 10 │     set data = vec::new() :vec[i64]
    │     ╭───────────────┘
 11 │     consume(data)
    │     ──────────────
 12 │     use data
    │     ╰── value used after move
```

#### 13.5.4 Multi-archivo

```text
error[E0401]: undefined symbol `kioto::strings::trim`
   --> src/main.mire:3:14
    │
  3 │ load kioto::strings
    │      ^^^^^^^^^^^^^^ module resolved to kioto@3.0.0
    │
   = note: `trim` exists in kioto@3.1.0 at modules/strings/mod.mr:14
   = help: update the lockfile: `owl update kioto`
```

Regla: el label primario está en el archivo del usuario; los secundarios pueden estar en dependencias, con su propio `-->` si están en otro archivo.

#### 13.5.5 Color

| Elemento | Color |
|----------|-------|
| `error` | rojo negrita |
| `warning` | amarillo negrita |
| `note` | azul |
| `help` | verde |
| `fix` | cian |
| Ubicación/gutter | cian tenue |
| Marcador primario | rojo/amarillo según severidad |

Política:

- `--color auto` (default): color si `isatty(stderr)` y `TERM != dumb` y `NO_COLOR` no está.
- `--color always` / `never`.
- `CLICOLOR_FORCE=1` fuerza.
- `NO_COLOR` (cualquier valor) desactiva.
- En CI (`CI=true`) → `never` salvo `--color always`.

#### 13.5.6 Deduplicación y orden

- Diagnósticos idénticos (mismo código, span, mensaje) se deduplican.
- Orden: `(archivo, línea, columna, código)`.
- Los errores van antes que los warnings del mismo archivo.
- `--error-limit N` corta y añade: `... 47 more errors not shown (use --error-limit 0 for all)`.
- `--error-limit 0` = sin límite.

### 13.6 Formatos de salida

#### 13.6.1 `human` (default)

El de §13.5.

#### 13.6.2 `short`

Para editores y CI:

```text
src/main.mire:7:18: error[E0005]: expected `i64`, found `str`
src/main.mire:12:9: error[E0201]: use of moved value: `data`
```

Una línea por diagnóstico, sin caja, sin notas.

#### 13.6.3 `json`

Esquema estable y versionado:

```json
{
  "schema_version": 1,
  "tool": {
    "name": "mire",
    "version": "4.5.0",
    "target": "x86_64-unknown-linux-gnu",
    "runtime": "base"
  },
  "diagnostics": [
    {
      "code": "E0005",
      "name": "type_mismatch",
      "severity": "error",
      "message": "expected `i64`, found `str`",
      "labels": [
        {
          "file": "src/main.mire",
          "span": { "start": 120, "end": 126 },
          "line": 7,
          "column": 18,
          "end_line": 7,
          "end_column": 24,
          "style": "primary",
          "message": "expected `i64`"
        }
      ],
      "notes": [
        { "kind": "cause", "message": "`str` cannot be coerced to `i64` implicitly" }
      ],
      "help": {
        "message": "use `str::to::i64` and handle the error",
        "code": "str::to::i64(\"hola\")"
      },
      "fixes": [
        {
          "applicability": "machine-applicable",
          "message": "wrap the literal in `str::to::i64`",
          "edits": [
            { "file": "src/main.mire", "start": 120, "end": 126, "replacement": "str::to::i64(\"hola\")" }
          ]
        }
      ],
      "url": "https://mire-lang.dev/errors/E0005"
    }
  ],
  "summary": { "errors": 1, "warnings": 0, "notes": 0 }
}
```

Reglas:

- Los offsets son **bytes** del archivo, no caracteres.
- `line`/`column` son 1-based y se incluyen por comodidad.
- `url` es estable y apunta a la explicación.
- `schema_version` se incrementa solo con cambios incompatibles.
- Un consumidor debe ignorar campos desconocidos.

#### 13.6.4 `json-rendered-ansi`

Igual que `json` más `"rendered": "<texto con ANSI>"`. Para IDEs que quieren mostrar el texto tal cual.

#### 13.6.5 `sarif`

Para GitHub Code Scanning / Sonar:

```json
{
  "$schema": "https://json.schemastore.org/sarif-2.1.0.json",
  "version": "2.1.0",
  "runs": [{
    "tool": { "driver": { "name": "mire", "rules": [{ "id": "E0005", "shortDescription": { "text": "type mismatch" } }] } },
    "results": [{
      "ruleId": "E0005",
      "level": "error",
      "message": { "text": "expected `i64`, found `str`" },
      "locations": [{ "physicalLocation": { "artifactLocation": { "uri": "src/main.mire" }, "region": { "startLine": 7, "startColumn": 18, "endColumn": 24 } } }]
    }]
  }]
}
```

#### 13.6.6 `--message-format`

Alias compatible de `--error-format` para no romper scripts. Si ambos se pasan, gana el último y se emite `W0065 duplicate_flag`.

### 13.7 Fix-its

#### 13.7.1 Niveles de aplicabilidad

| Nivel | Significado | `mire fix` |
|-------|-------------|------------|
| `MachineApplicable` | Cambio seguro y completo | Se aplica |
| `MaybeIncorrect` | Heurístico; puede no compilar | Se muestra, no se aplica sin `--apply-maybe` |
| `HasPlaceholders` | Falta información (`_:T`) | Se muestra, no se aplica |
| `Unspecified` | Solo sugerencia textual | Se muestra |

#### 13.7.2 Comandos

```bash
mire fix                  # dry-run: muestra diff unificado
mire fix --apply          # aplica solo MachineApplicable
mire fix --apply --code E0201
mire fix --apply --all    # incluye MaybeIncorrect (pide confirmación)
mire fix --list           # lista fixes disponibles
```

Salida del dry-run:

```diff
--- src/main.mire
+++ src/main.mire
@@ -9,3 +9,3 @@
-    consume(data)
+    consume(&data)
```

Reglas:

- Nunca aplica sobre un archivo con cambios sin commitear salvo `--force`.
- Escribe con temp + rename atómico.
- Un fix que solapa a otro se descarta y se reporta.
- Tras aplicar, recompila y reporta si quedan errores.

#### 13.7.3 Fix-its por categoría

| Categoría | Ejemplos |
|-----------|----------|
| Tipo | Añadir `:i64`, envolver en `str::to::i64`, cast explícito |
| Ownership | `&x` en lugar de `x`, `clone()` |
| Import | Añadir `load`, corregir ruta, `use!` faltante |
| Deprecación | Reemplazar símbolo viejo por nuevo |
| CLI | `--relese` → `--release` |
| Allowlist | Añadir `rt_*` faltante |

### 13.8 `mire explain`

```bash
mire explain E0005
mire explain E0201 --examples
mire explain --list
mire explain --search ownership
mire build --explain E0201      # imprime la explicación tras el error
```

Formato:

```text
E0005 — type_mismatch

Un valor de un tipo se usa donde se espera otro, y no existe una coerción
implícita entre ellos.

Mire no convierte entre tipos automáticamente porque una conversión oculta
puede perder información (por ejemplo, i64 → i32).

Mal:
    set x :i64 = "42"

Bien:
    set x :i64 = str::to::i64("42")!

Variantes:
    E0100 precision_loss
    E0105 sign_mismatch
    E0110 numeric_kind_mismatch

Ver también:
    str::to::i64, i64::from
```

Reglas:

- El contenido vive en `avenys/docs/errors/E0005.md` y se **genera** al registro.
- `mire explain <code>` lee del binario (embebido) para funcionar offline.
- Un test de CI verifica que cada código tiene archivo y que el archivo parsea.

### 13.9 Sistema de ayuda

#### 13.9.1 Niveles

| Comando | Contenido |
|---------|-----------|
| `mire` | Resumen de comandos + “use `mire help <cmd>`” |
| `mire -h` | Ayuda corta del comando actual |
| `mire --help` | Ayuda larga (incluye ejemplos y exit codes) |
| `mire help` | Índice de temas |
| `mire help build` | Igual que `mire build --help` |
| `mire help errors` | Cómo leer diagnósticos + enlace a `explain` |
| `mire help targets` | Tabla de targets |
| `mire help cache` | Esquema y comandos |
| `mire help env` | Variables de entorno |
| `mire help exit-codes` | Tabla de códigos de salida |

#### 13.9.2 Contenido mínimo de `--help`

Toda ayuda de comando incluye:

1. Descripción en una línea.
2. `Usage:`.
3. Flags agrupados (`Profile`, `Dependencies`, `Output`, `Diagnostics`, …).
4. Ejemplos (2-3).
5. `Exit codes:`.
6. `See also:`.

#### 13.9.3 Anti-drift

- `FlagSpec` único (§5.2.4).
- Test: `mire <cmd> --help` contra golden.
- Test: todo flag del parser aparece en la ayuda y viceversa.
- `mire help --check` en CI falla si hay drift.

#### 13.9.4 Man y completions

```bash
mire man > mire.1
mire completions bash > mire.bash
mire completions zsh  > _mire
mire completions fish > mire.fish
```

Generados del mismo `FlagSpec`.

### 13.10 Códigos de salida

| Código | Significado |
|--------|-------------|
| `0` | Éxito |
| `1` | Error de compilación |
| `2` | Error de uso / CLI |
| `3` | Error de E/S |
| `4` | Error de resolución de paquetes |
| `5` | Tests fallidos |
| `6` | Error de cache irrecuperable |
| `7` | Error de enlace / dependencia nativa |
| `101` | ICE |
| `130` | Interrumpido (SIGINT) |

Reglas:

- Un error de CLI **nunca** devuelve `1` (se confunde con error de compilación).
- `mire test` devuelve `5` si hay tests fallidos, `1` si no compila.
- Un ICE devuelve `101` y el mensaje incluye cómo reportar.
- `owl` propaga el código del compilador, salvo que el suyo sea más específico.

### 13.11 Warnings y lints

#### 13.11.1 Niveles

| Nivel | Flag | Efecto |
|-------|------|--------|
| `allow` | `-A <lint>` | No se emite |
| `warn` | `-W <lint>` | Se emite (default) |
| `deny` | `-D <lint>` | Se convierte en error |
| `forbid` | `-F <lint>` | Como deny, pero no se puede anular con `@[allow]` |

Sintaxis de lint: `nombre` o `grupo` o `nombre=allow`.

#### 13.11.2 Grupos

| Grupo | Incluye |
|-------|---------|
| `correctness` | `unused_must_use`, `unreachable_code`, `stale_security_allowlist`, `implicit_narrowing` |
| `unused` | `unused_variables`, `dead_code`, `unused_extern` |
| `style` | `non_snake_case_*`, `needless_borrow`, `redundant_bool_compare` |
| `complexity` | `long_function`, `deep_loop_nesting`, `deeply_nested_if` |
| `performance` | `redundant_clone`, `string_concat_in_loop` |
| `suspicious` | `self_comparison`, `clock_regression`, `shadowed_builtin` |
| `deprecated` | `deprecated_syntax`, `deprecated_api`, `legacy_link_flag` |
| `security` | `shell_invocation`, `unsafe_block` |
| `pedantic` | Opt-in; no en `warn` por defecto |

#### 13.11.3 Atributos por ítem

```mire
@[allow(unused_variables)]
pub fn f: () { … }

@[deny(implicit_narrowing)]
pub fn g: () { … }

@[expect(deprecated_api)]      // falla si el warning NO ocurre
pub fn h: () { … }
```

`@[expect]` es clave para migraciones: permite escribir código nuevo que use una API deprecada y saber cuándo dejar de hacerlo.

#### 13.11.4 `--cap-lints`

- `--cap-lints allow` en dependencias: no se muestran sus warnings.
- `--cap-lints warn`: sus `deny` no rompen el build.
- Owl pasa `--cap-lints allow` para paquetes de `~/.owl/libs`, y sin cap para el proyecto del usuario.

#### 13.11.5 Baseline de warnings

Para CI con código legacy:

```bash
mire build --lint-baseline baseline.json          # escribe
mire build --lint-baseline baseline.json --check   # falla solo con warnings nuevos
```

El baseline guarda `(archivo, línea, código, hash del mensaje)`; si el código se mueve, no se considera nuevo.

#### 13.11.6 Presentación

```text
warning[W0001 unused_variables]: variable `tmp` is never used
   --> src/main.mire:14:9
    │
 14 │     set tmp = 42
    │         ^^^ help: remove it or prefix with `_`: `_tmp`
```

Reglas:

- El nombre del lint va **siempre** en la cabecera.
- `help` con fix-it cuando sea posible.
- Al final: `warning: 3 warnings emitted. Use -A <lint> to silence, -D <lint> to deny.`

### 13.12 Mensajes de ownership (MSS)

Hoy:

```text
error[E0007] ── Ownership Error
╭─[ main.mire:12:9 ]
...
╰─ MSS Error: Use after move - value was moved
```

Problemas: “MSS” es jerga interna; no dice **qué** valor; no dice **dónde** se movió; no ofrece arreglo.

Objetivo:

```text
error[E0201]: use of moved value: `data`
   --> src/main.mire:12:9
    │
 10 │     set data = vec::new() :vec[i64]
    │         ---- `data` moved here
 11 │     consume(data)
 12 │     use data
    │         ^^^^ value used after move
    │
    = note: `vec[i64]` is not `Copy`; passing it by value transfers ownership
    = help: borrow instead: `consume(&data)`
    = fix: replace `consume(data)` with `consume(&data)` (machine-applicable)
```

Implementación:

1. `MssError` deja de ser una unidad y pasa a llevar contexto:
   ```rust
   pub struct MssError {
       pub kind: MssKind,
       pub value: String,          // nombre del símbolo
       pub move_site: Option<ByteSpan>,
       pub use_site: ByteSpan,
   }
   ```
2. `Dim`/`Display` ya no se usa para el mensaje de usuario.
3. Cada `MssKind` tiene:
   - `title` (“use of moved value”),
   - `message` (con el nombre),
   - `note` (por qué),
   - `help` (cómo),
   - `fix` (si aplica).

Tabla de mensajes:

| Kind | Título | Note | Help |
|------|--------|------|------|
| `UseAfterMove` | use of moved value: `x` | `T` is not `Copy` | borrow with `&x` |
| `MultipleMutableRefs` | cannot borrow `x` as mutable more than once | first borrow here | end the first borrow before the second |
| `MutationWhileShared` | cannot mutate `x` while it is borrowed | shared borrow here | use `x` after the borrow ends |
| `MoveWhileBorrowed` | cannot move `x` while it is borrowed | borrow here | clone or wait |
| `DropWhileBorrowed` | cannot drop `x` while it is borrowed | borrow here | drop the borrow first |
| `BorrowOutOfScope` | borrow of `x` outlives its owner | owner dropped here | return by value or extend the owner |
| `InvalidMove` | cannot move out of `x` | `x` is borrowed/`&` | clone or restructure |
| `UnsafeViolation` | unsafe operation outside `unsafe` | — | wrap in `unsafe { … }` |

### 13.13 Errores de resolución, enlace y cache

#### 13.13.1 Import no resuelto

```text
error[E0025]: unresolved import `kioto::jsn`
   --> src/main.mire:3:14
    │
  3 │ load kioto::jsn
    │      ^^^^^^^^^^ no module `jsn` in kioto@3.1.0
    │
    = help: did you mean `kioto::json`? (edit distance 1)
    = note: available modules: strings, time, fs, env, proc, async, mem, cpu, math, net, log, cli, crypto
    = fix: replace `kioto::jsn` with `kioto::json` (machine-applicable)
```

Reglas:

- Sugerencia por distancia de edición (Damerau-Levenshtein) sobre los módulos reales.
- Si el paquete no está instalado: `E0601` con `owl add kioto`.
- Si la versión no tiene el módulo: `E0602` con `owl update kioto`.

#### 13.13.2 Símbolo indefinido en enlace

```text
error[E0401]: undefined symbol `rt_crypto_sha256_hex`
   --> kioto/modules/crypto/hash/_externs.mr:5:13
    │
  5 │ extern fn rt_crypto_sha256_hex: (s :&str) :str lib "c"
    │           ^^^^^^^^^^^^^^^^^^^^ not provided by any linked object
    │
    = note: searched: libmire_runtime.a, libkioto.a, libc.so
    = note: this symbol is part of the C runtime; it was removed in the `experimental` PAL
    = help: build with `--runtime base` or add the PAL implementation
    = help: if this is a native dependency, run `mire doctor`
```

#### 13.13.3 Dependencia nativa faltante (libsodium)

```text
error[E0404]: native dependency `sodium` not found
   --> <linker>
    │
    = note: `kioto::crypto` requires libsodium for `rt_crypto_*`
    = note: searched: /usr/lib, /usr/local/lib, /lib
    = help: install it:
        Arch:   sudo pacman -S libsodium
        Debian: sudo apt install libsodium-dev
        Fedora: sudo dnf install libsodium-devel
    = help: install the missing archive, or accept a dynamic link with `--bundle dynamic`
    = help: use `--bundle static-strict` in CI to require static archives
    = help: or avoid crypto: build with `-A deprecated_api` is not enough; remove the `load kioto::crypto`
```

`mire doctor` produce el mismo diagnóstico sin compilar.

#### 13.13.4 Hash mismatch

```text
error[E0403]: hash mismatch for library `kioto`
   --> <lockfile>
    │
    = expected: sha256:ab12…
    = found:    sha256:cd34…
    = note: the library content changed after the lockfile was written
    = help: if the change is intentional: `owl update kioto`
    = help: if not: restore the pinned copy or run `owl install --locked`
```

#### 13.13.5 Cache corrupta

```text
warning[E0501 cache_schema_mismatch]: cache schema 2 is not compatible with 3
   --> bin/.cache/version.json
    │
    = note: the cache was moved to bin/.cache/quarantine/1763000000/
    = note: the next build will be a full rebuild
```

Nunca se le pide al usuario `rm -rf .cache`.

### 13.14 Recuperación del parser y “did you mean”

#### 13.14.1 Recuperación

Hoy un error de sintaxis detiene el parser en el primer token inesperado. Objetivo:

1. **Sincronización:** al fallar una sentencia, saltar hasta `;`, `}` o el siguiente `pub fn`/`module`/`load`.
2. **Múltiples errores:** reportar hasta `--error-limit` en una pasada.
3. **Nunca** producir un AST inválido que cause un ICE posterior: se marca el nodo como `Error` y se sigue.

#### 13.14.2 Mensajes de expectativa

```text
error[E0003]: expected `}` to close the block opened at 3:12
   --> src/main.mire:9:1
    │
  3 │ pub fn main: () {
    │                - block opened here
  9 │ pub fn next: () {
    │ ^^^ expected `}` before this item
    │
    = help: add `}` at the end of line 8
```

Reglas:

- Siempre citar el sitio de apertura.
- Si el token inesperado es un keyword de ítem, decir “expected `}` before this item”, no “unexpected token”.
- Para `} else {` en línea nueva: `error[E0003]` con nota de estilo y fix-it (ver §14, S3).

#### 13.14.3 Sugerencias

| Situación | Sugerencia |
|-----------|------------|
| Identificador desconocido | Candidatos por distancia ≤ 2 |
| Flag desconocida | Igual sobre el `FlagSpec` |
| Módulo desconocido | Igual sobre `[exports]` |
| Campo de struct desconocido | Igual sobre los campos |
| Tipo desconocido | Igual sobre tipos en scope |

Formato: `help: did you mean `X`?` + fix-it si hay un único candidato.

### 13.15 Errores de runtime y PAL

Los errores de runtime ocurren **en el binario**, no en el compilador. Deben ser igual de claros:

```text
error[E0802]: sandbox denied write to `/etc/passwd`
   --> fs::write
    │
    = root: /home/user/project
    = requested: /etc/passwd
    = reason: path escapes the sandbox root
    = help: use `fs::unrestricted` only if you trust this code
```

Requisitos:

- `pal_*` devuelve `result[T Error]`, no `-1`.
- Cada `Error` tiene `code` estable (`pal/eperm`, `pal/enoent`, `pal/sandbox_denied`).
- El binario imprime el mismo formato de caja que el compilador (reutiliza `mire::diag`).
- `KIOTO_BACKTRACE=1` añade la pila Mire, no la de C.

### 13.16 ICE: internal compiler error

```text
error[E0901]: internal compiler error
   --> src/main.mire:1:1
    │
    = thread: main
    = version: mire 4.5.0 (abc1234, 2026-11-13)
    = target: x86_64-unknown-linux-gnu
    = panic: called `Option::unwrap()` on a `None` value
    │
    = note: this is a bug in the compiler, not in your code
    = help: please report it:
        https://github.com/mire-lang/avenys/issues/new?template=ice.yml
        attach the file below and the output of `mire --version`
    = file: /tmp/mire-ice-1763000000/main.mire
    = log:  /tmp/mire-ice-1763000000/ice.log
```

Reglas:

1. El perfil release actual tiene `panic = "abort"`: `catch_unwind` no recupera con abort y nunca captura señales. Un panic Rust usa `panic::set_hook`, informa E0901 y sale con 101; en debug se usa `panic = "unwind"` y `catch_unwind` solo donde sea seguro.
2. SIGSEGV/SIGBUS/SIGILL usan un handler async-signal-safe: descriptor preabierto, buffer estático, `write()` mínimo y `_exit(101)`. Instalarlo con `sigaltstack`/`SA_ONSTACK` por hilo y guard page para detectar stack overflow. No asigna memoria ni formatea texto.
3. El proceso supervisor construye el informe completo. Un worker que termina por panic o señal se reporta como E0903 `worker_panic`; señal fatal directa es E0904 `fatal_signal`; stack overflow es E0905 `stack_overflow`.
4. Copiar el archivo fuente y los flags a `/tmp/mire-ice-<ts>/`.
5. Redactar rutas de usuario (`/home/<user>` → `$HOME`) en el log.
6. `mire bug` genera el reporte y abre el navegador (opcional).
7. `MIRE_BACKTRACE=1` muestra la pila Rust.
8. Nunca “segmentation fault” sin explicación.

### 13.17 Contrato de errores en las librerías Mire

El compilador no es el único que emite errores. Las librerías también. Para que los mensajes sean consistentes:

```mire
// mire/src/error/mod.mire
pub struct Error {
    code :str           // "fs/enoent", "json/out_of_range"
    message :str        // legible, en inglés
    cause :maybe[&Error] // nunca inline; arena de la operación vive con el result
    span :maybe[Span]   // si la librería lo conoce
}
```

Reglas:

1. Toda función pública falible devuelve `result[T Error]`.
2. `code` es estable y namespaced: `<lib>/<motivo>`.
3. `message` no repite el código.
4. `cause` nunca se almacena inline: preferir `maybe[&Error]` con arena cuya vida acompaña al result. Antes de fijar la firma, comprobar que el result conserva el arena; alternativa `maybe[Box[Error]]` solo si Collections/Box confirma heap ownership real. Si ninguna aplica, usar un handle estable.
5. Un `Error` se puede imprimir con `use std::eprintln(e)` y sale formateado.
6. Nunca `0`, `-1`, `""` o `null` como error silencioso.

Esto es lo que permite que `i64::input` del ejemplo canónico muestre un error claro en vez de un `0`.

### 13.18 Guía de estilo de mensajes

| Regla | Bien | Mal |
|-------|------|-----|
| Minúscula inicial | `expected i64, found str` | `Expected I64, Found Str` |
| Sin punto final | `cannot move out of borrowed content` | `Cannot move … .` |
| Imperativo en help | `use …` | `You should use …` |
| Nombrar el símbolo | `use of moved value: \`data\`` | `use of moved value` |
| Citar el sitio | `moved here` + span | solo “elsewhere” |
| Sin jerga interna | `ownership error` | `MSS Error` |
| Un concepto por frase | `expected X, found Y` | `type mismatch while checking …` |
| No culpabilizar | `this value is not Copy` | `you made a mistake` |
| Inglés estable | mensajes en inglés | mezclar idiomas |
| Código en backticks | `` `i64` `` | `i64` sin marcar |

Se implementa un lint de mensajes en CI: `cargo test --test message_style` que revisa que ningún `format!` de diagnóstico termine en `.` ni empiece en mayúscula.

### 13.19 Documentación generada

**Problema [verificado]:** `avenys/docs/errors/README.md` tiene 498 líneas y ya divergió del código (E0008/E0009).

Solución: **una sola fuente de verdad**.

```rust
// error/registry.rs
pub struct DiagnosticMeta {
    pub code: DiagnosticCode,
    pub name: &'static str,
    pub category: Category,
    pub default_severity: Severity,
    pub explanation: &'static str,   // include_str!("../../docs/errors/E0005.md")
    pub introduced: &'static str,
    pub deprecated: Option<&'static str>,
    pub replaces: Option<DiagnosticCode>,
}
```

Generación:

```bash
mire docs errors --output docs/errors/     # README + una página por código
mire docs errors --check                   # falla si hay drift
```

CI:

1. Todo `DiagnosticCode` tiene `DiagnosticMeta`.
2. Todo `DiagnosticMeta` tiene archivo `.md`.
3. Todo archivo `.md` está referenciado.
4. `mire docs errors --check` sin cambios.

### 13.20 Tests de diagnósticos

#### 13.20.1 Golden tests

```text
avenys/tests/diagnostics/
├── cases/
│   ├── E0005/
│   │   ├── literal_mismatch.mire
│   │   ├── literal_mismatch.stderr
│   │   └── literal_mismatch.json
│   ├── E0201/
│   │   ├── move_then_use.mire
│   │   ├── move_then_use.stderr
│   │   └── move_then_use.json
│   └── …
└── runner.rs
```

El runner:

1. Ejecuta `mire build --color never --error-format human <caso>.mire`.
2. Compara `.stderr` exacto.
3. Ejecuta con `--error-format json` y valida el esquema + campos.
4. Verifica que el `code`, `span` y `fixes` son los esperados.
5. Actualiza con `--snapshot`.

#### 13.20.2 Cobertura de códigos

```bash
mire docs errors --check-coverage
# error: E0023 has no test case
# error: W0044 has no test case
```

Gate de CI: 100 % de códigos con al menos un caso.

#### 13.20.3 Tests de la ayuda

- `mire --help` → golden.
- `mire build --help` → golden.
- `mire help errors` → golden.
- Todo flag del parser aparece en la ayuda.
- Toda flag de la ayuda es parseable.
- `--help` no accede a red ni a `~/.owl`.

#### 13.20.4 Tests de fix-its

- `mire fix --apply` sobre un caso → compila.
- Fix solapado → se descarta con nota.
- Fix sobre archivo con cambios → se niega sin `--force`.

### 13.21 Migración y compatibilidad

| Paso | Cambio | Compatibilidad |
|------|--------|----------------|
| 1 | Añadir `ByteSpan`/`SourceMap` en paralelo a `Span` | Total |
| 2 | Migrar emisores a `Diagnostic` nuevo | Total |
| 3 | Añadir `--error-format` | Total |
| 4 | Añadir códigos E02xx–E09xx | Total (códigos nuevos) |
| 5 | Alias E0007→E0201 con `deprecated` | Un release |
| 6 | Eliminar alias | 5.0 |
| 7 | Regenerar docs desde registry | Total |
| 8 | Sustituir `println!` de ayuda por `FlagSpec` | Total |
| 9 | `-L` semántica nueva | Shim §1.1 |

Regla: **un código nunca cambia de significado**. Si se necesita otro, se crea.

### 13.22 Rúbrica de calidad de un mensaje

Cada diagnóstico se puntúa 0-2 en 8 ejes:

| Eje | 0 | 1 | 2 |
|-----|---|---|---|
| Código | No tiene | Tiene | Tiene + nombre + doc |
| Ubicación | No tiene | Línea | Span exacto |
| Qué | Vago | Concreto | Concreto + símbolo |
| Por qué | No dice | Insinúa | Note explícita |
| Cómo | No dice | Sugerencia | Fix-it aplicable |
| Contexto | Ninguno | 1 label | 2+ labels / origen |
| Consistencia | Jerga | Aceptable | Sigue guía |
| Accionable | No | Parcial | Se arregla en un paso |

Objetivo: media ≥ **1.8** en errores, ≥ **1.5** en warnings. Se mide con una muestra de 50 diagnósticos reales por release.

---

## 14. Propuestas de sintaxis

> Todas son **compatibles hacia atrás** o marcadas como deprecación con ciclo. Ninguna toca el parser de forma incompatible; las que sí lo tocan son aditivas.

### 14.1 Compatibilidad y alcance

| Tipo | Definición |
|------|------------|
| **Aditiva** | Se acepta además de lo actual; no rompe nada. |
| **Deprecación** | Se acepta con warning durante un release; luego se elimina. |
| **Convención** | No cambia el parser; cambia cómo escribimos las librerías. |

### 14.2 Tabla de propuestas

| # | Problema | Propuesta | Tipo | Impacto |
|---|----------|-----------|------|---------|
| S1 | `dasu`/`ireru` opacos | Alias `std::print` / `std::input`; mantener japoneses como compat | Aditiva | Alto (UX) |
| S2 | `set` declara y reasigna | `let` para declaración inmutable; `set` solo mutación | Aditiva | Medio |
| S3 | `} else {` obligatorio en la misma línea | Permitir `else` en línea nueva con warning + fix-it | Deprecación inversa | Alto |
| S4 | `(Type field: val)` para structs | Aceptar `Type { field: val }` como alternativa | Aditiva | Alto |
| S5 | `self` implícito en pipeline | Hacer explícito `x \|> f(it)`; `self` sigue válido | Aditiva | Medio |
| S6 | `load`/`load!`/`use!` confusos | Mantener (D8); mejorar docs y diagnósticos | Convención | Medio |
| S7 | Named args solo en llamada | Permitir en definición: `fn greet: (name: &str, age: i64)` | Aditiva | Bajo |
| S8 | Sin `Parse` estándar | `skill Parse[T] { fn parse: (s :&str) :result[T Error] }` | Aditiva | Medio |
| S9 | Conversiones devuelven `0` | `to::i64` → `result[i64 Error]`; `to::i64::or` explícito | Deprecación | **Alto** |
| S10 | Macros son funciones con `!` | Documentar como “call sugar”; no prometer metaprogramación | Convención | Bajo |
| S11 | `status()` gigante por bug del optimizador | Arreglar el reordenamiento de `+`; luego construir normal | Bugfix | Medio |
| S12 | Sin `Duration`/`Instant` | Añadir en `mire::time` | Aditiva | Medio |
| S13 | Sin `for` indexado | `for (i, x) in items` o `seq::enumerate` | Aditiva | Medio |
| S14 | `?` no documentado en `maybe` | Documentar `?` para `result` y `maybe`; tests | Docs | Bajo |
| S15 | `cons` no usable por path | `const` con path o funciones | Aditiva | Bajo |
| S16 | Sin `must_use` | `@[must_use]` + warning `W0050` | Aditiva | Alto |
| S17 | Sin deprecación de API | `@[deprecated(since, note)]` + warning `W0059` + fix-it | Aditiva | Alto |
| S18 | Sin `expect` | `@[expect(lint)]` para migraciones | Aditiva | Medio |
| S19 | Errores de librería inconsistentes | `mire::error::Error` + `result[T Error]` | Convención | Alto |
| S20 | `use!` obligatorio sin explicación | Diagnóstico E0026 con fix-it que lo añade | Bugfix | Alto |

### 14.3 Detalle de las propuestas de mayor impacto

#### S1 — `std::print` / `std::input`

```mire
// Hoy
use dasu("hola")
set nombre = ireru("Nombre: ")

// Propuesto (alias exacto, no copia)
use std::println("hola")
set nombre = std::input("Nombre: ")
```

`dasu`/`ireru` siguen funcionando; `mire::std` los re-exporta. Un lint `W0066 deprecated_japanese_alias` (opcional) sugiere el alias inglés en código nuevo.

#### S3 — `else` en línea nueva

```mire
// Hoy: error
if cond {
    …
}
else {
    …
}

// Propuesto: warning + fix-it
warning[W0067 else_on_new_line]: `else` should be on the same line as `}`
help: move `else` to the previous line
fix: replace `}\nelse {` with `} else {` (machine-applicable)
```

Esto elimina una de las fricciones más reportadas sin romper la gramática.

#### S4 — Struct literal con llaves

```mire
// Hoy
set p = (Point x: 1, y: 2)

// Propuesto (ambas válidas)
set p = Point { x: 1, y: 2 }
```

Regla de desambiguación: `Type {` solo es struct literal si `Type` está en el scope de tipos; si no, se interpreta como bloque (comportamiento actual).

#### S8 / S9 — `Parse` y conversiones

```mire
pub skill Parse[T] {
    fn parse: (s :&str) :result[T Error]
}

impl Parse[i64] {
    fn parse: (s :&str) :result[i64 Error] { … }
}

pub fn input[T: Parse]: (msg :&str) :T { … }
```

Con esto, `i64::input` puede ser genérico de verdad, sin `T` mágico.

#### S16 / S17 / S18 — Atributos

```mire
@[must_use]
pub fn parse: (s :&str) :result[Value Error] { … }

@[deprecated(since = "3.2.0", note = "use ilogical::format::json")]
pub fn parse_json: (s :&str) :Value { … }

@[expect(deprecated_api)]
pub fn legacy_path: () { … }
```

- `must_use` → `W0050 unused_must_use` si se ignora el `result`.
- `deprecated` → `W0059 deprecated_api` con fix-it al reemplazo.
- `expect` → falla si el lint **no** ocurre (útil para limpiar migraciones).

#### S13 — `enumerate`

```mire
for (i, x) in vec::enumerate(items) {
    …
}
```

Si el `for` con tupla no es viable en el parser actual, se entrega como `seq::enumerate(items) :vec[(i64, T)]` y se documenta.

#### S20 — Diagnóstico de `use!`

```text
error[E0026]: `trim` is ambiguous between `kioto::strings` and `mire::str`
   --> src/main.mire:8:5
    │
  8 │     trim(x)
    │     ^^^^ ambiguous
    │
    = help: disambiguate with `use!`:
        use! kioto::strings::trim
    = fix: insert `use! kioto::strings::trim` after the last `load` (machine-applicable)
```

Hoy `use!` es obligatorio pero el error no explica cómo añadirlo. Este fix-it convierte una fricción en un paso.

### 14.4 Convenciones de librería (`mire::case`)

Además de los renombres de §3.4:

1. **Un módulo = un namespace.** No `kioto_strings`; sí `kioto::strings`.
2. **Los tipos viven en el namespace del módulo:** `kioto::net::Socket`, no `KiotoSocket`.
3. **Las funciones sobre un tipo cuelgan del tipo:** `Socket::connect`, no `socket_connect`.
4. **Los errores cuelgan de la operación:** `fs/open::not_found`, no `FS_ENOENT`.
5. **Los tests espejan el namespace:** `tests/fs/symbols/open.mire`.

Un lint `W0068 non_mire_case` detecta `pub fn` con `_` que debería ser namespace, solo en paquetes del ecosistema (configurable en `owl.toml [lint] mire_case = true`).

---

## 15. Fases de ejecución

> Estimaciones en semanas de una persona. La columna “depende” marca bloqueos duros.

### F0 — Reset y baseline (1 semana)

| Tarea | Entregable |
|-------|------------|
| Aplicar §12 (dry-run + apply) | Repos limpios |
| `.gitignore` por repo | Sin artefactos versionados |
| Congelar `avenys/bench/BASELINE.md` | 30+ kernels medidos |
| `scripts/reset-tree.sh` | Reutilizable |
| Runner de tests con aislamiento `fixtures/tmp` | `mire test` no ensucia |
| Inventario de exports, recolector `@[covers]`, etiquetas de casos y expiración de waivers | Métricas M1/M2 (§11.7) |

**Gate:** `git status --porcelain` vacío tras `owl test` en cada repo.

### F1 — Owl lockfile + parser real (2 semanas) **CRÍTICO**

| Tarea | Depende |
|-------|---------|
| `ilogical::format::toml` con tests | — |
| `lockfile::{in_sync,generate,all}` sin parseo manual | TOML |
| `ensure_project_dependencies` offline-first | lockfile |
| Tests: proyecto fresco, local, lock ausente/desincronizado/corrupto | — |

**Gate:** `owl build` en proyecto fresco con deps instaladas: **< 2 s, 0 red**.

### F2 — Protocolo Owl↔Avenys (2 semanas)

| Tarea | Depende |
|-------|---------|
| `CompileRequest` v2 + goldens | F1 |
| `--proto` en Avenys | — |
| Owl produce `CompileRequest` | v2 |
| Eliminar `--config`, `write_mire_config`, `append_c_and_security_flags` | v2 |
| Resolución por símbolo + `owl oracle` | v2 |

**Gate:** `owl build` no escribe TOML temporal; Avenys no lee `owl.toml`.

### F3 — Cache nueva (2 semanas)

| Tarea | Depende |
|-------|---------|
| Esquema `file::bin/` + perfiles + `struc/` | F2 |
| Build key completa | — |
| Publicación atómica + manifest sidecar | — |
| Cuarentena automática | — |
| `mire cache verify\|gc\|doctor` | — |
| Tests de cache (§5.4.7) | — |

**Gate:** cambiar un byte en una dependencia actualiza el binario sin borrar cache.

### F4 — PAL C → Mire (3 semanas) **CRÍTICO**

| Tarea | Depende |
|-------|---------|
| `syscall6` / `asm` intrínseco | — |
| `mire::pal::{core,linux,sandbox,dispatch}` | — |
| Migrar `mire::aux::pal`, kioto externs, owl `_externs` | PAL |
| Bench sin regresión > 2 % | PAL |

**Gate:** `--runtime experimental` compila y pasa los 30+ kernels; ningún kernel puede quedar por debajo de 1.2× respecto a su baseline acordado y se informa por kernel.

### F5 — Identidad Mire + codecs (2 semanas)

| Tarea | Depende |
|-------|---------|
| `ilogical::format::{json,toml}` puro Mire | F1 |
| Borrar `mire/code/` y `[c]` | codecs |
| Eliminar `kioto::{json,toml}` | codecs |
| Tests por símbolo de codecs | codecs |

**Gate:** `mire` compila sin `mire/code/`; `kioto` no tiene wrappers C.

### F6 — Tests por símbolo (3 semanas, paralelizable)

| Tarea | Depende |
|-------|---------|
| `_support/{harness,fixture,log}` | F0 |
| Suites de kioto, mire, ilogical, sqlite, sdl, wasm | F5 |
| M1=100 % y M2≥60 % (§11.7) | — |

**Gate:** M1=100 % y M2≥60 %; los casos imposibles deben tener waiver vigente (§11.7).

### F7 — Diagnósticos (4 semanas) **INVERSIÓN PRINCIPAL**

| Semana | Tarea |
|--------|-------|
| 1 | `ByteSpan`, `SourceMap`, `Diagnostic` nuevo, renderer |
| 2 | `--error-format` human/short/json/sarif, `mire explain` |
| 3 | Fix-its, `mire fix`, help declarativo, exit codes |
| 4 | Lints, ownership, ICE, docs generadas, goldens |

**Gate:** rúbrica §13.22 ≥ 1.8; 100 % de códigos con test y doc.

### F8 — Sintaxis y UX (2 semanas)

| Orden | Propuesta |
|-------|-----------|
| 1 | S1, S3, S4, S9, S16, S17, S20 |
| 2 | S2, S5, S8, S13 |
| 3 | S7, S12, S14, S15 |

**Gate:** cada propuesta con tests y guía de migración.

### F9 — Binarios autosuficientes (1 semana)

| Tarea |
|-------|
| `--bundle static` default, `static-strict` para releases |
| Sacar libsodium del runtime base |
| `mire doctor` |
| En `scratch`: `owl --version`, `mire --version`; build de hello-world con `load kioto::crypto` usando Owl y Mire |

**Gate:** los dos binarios arrancan y los dos builds hello-world funcionan en `scratch` con `static-strict` y sin libs dinámicas instaladas. Esto no valida el enlace LLVM 22 de Avenys.

### 15.1 Camino crítico

```text
F0 → F1 → F2 → F3 → F5
              ↘ F4
F2 → F7 (en paralelo desde la semana 3)
F6 depende de F0 y F5
F9 depende de F4
```

| Escenario | Semanas | Supuesto |
|-----------|---------|----------|
| Secuencial puro | 22 | Sin solape; con 15 % de integración/revisión, ≈25 |
| 1 persona con solape F6/F5 | 21 | F6 inicia al cerrar los codecs necesarios |
| 2 personas | 19 | F4 ∥ F7; no comparten archivos |
| 3 personas | 16–17 | F4 ∥ F6 ∥ F7; F6 depende de F5 solo para codecs |

Camino crítico con 2 personas: **F0 → F1 → F2 → F3 → F5 → F6 → F7 → F8 → F9 = 19 semanas**; F4 corre en paralelo y solo bloquea F9. Con 3 personas, F6 y F7 también se solapan.

---

## 16. Métricas de aceptación

| Métrica | Baseline | Target | Fase |
|---------|----------|--------|------|
| `owl build` proyecto fresco con deps | bloquea/red | < 2 s, 0 red | F1 |
| Compilar `main` vacío sin red | ? | siempre | F1 |
| Archivos compilados para 1 símbolo | módulo completo | 1-3 archivos | F2 |
| Benchmarks kernels | baseline | regresión ≤ 2 % | F4 |
| Objetos C de runtime | 5 archivos | 0 (solo `pal_abi.h`) | F4 |
| Directorios con solo `owl.toml` | 20 (ilogical) | 0 | F5 |
| Stubs de 1-2 líneas | 4 (math) | 0 | F0 |
| Código duplicado lockfile | ~150 líneas | 0 | F1 |
| `[build]` duplicado en config | 2 veces | 0 | F2 |
| M1 — presencia de símbolos públicos | ~15 % | 100 % | F6 |
| M2 — contrato por símbolo | sin medir | ≥ 60 % F6; ≥ 90 % F8 (waivers ≤10 %) | F6/F8 |
| Tests que ensucian la raíz | 6 directorios | 0 | F0 |
| `--config` / `MIRE_CONFIG` | presente | eliminado | F2 |
| Fallo por libsodium en binarios | sí | no | F9 |
| Códigos con doc | parcial + drift | 100 % | F7 |
| Códigos con test | parcial | 100 % | F7 |
| Errores con span exacto | línea/columna | byte span | F7 |
| Errores con fix-it | ~0 % | ≥ 60 % | F7 |
| Rúbrica de mensaje (errores) | ~0.9 | ≥ 1.8 | F7 |
| Warnings con nombre de lint | no | sí | F7 |
| `--error-format json` | no | estable | F7 |
| Exit codes documentados | no | sí | F7 |
| Cache exorcism | sí | no | F3 |

---

## 17. Riesgos

| Riesgo | Prob. | Impacto | Mitigación |
|--------|-------|---------|------------|
| Migración PAL rompe ABI | Media | Alto | Bench por paso; shim `syscall6` temporal; `--runtime base` default |
| Protocolo v2 rompe v1 | Baja | Alto | Campos ignorables; goldens v1+v2 |
| Resolución por símbolo pierde símbolos | Media | Alto | Fallback `eager`; `E0401` cita paquete/archivo |
| Cache nueva introduce regresiones | Media | Alto | Manifest sidecar + `cache verify` + cuarentena |
| `mire::case` rompe API | Alta | Medio | Alias deprecados un release |
| Tests por símbolo son caros | Alta | Medio | Paralelizar; cache de test acotada; `--filter` |
| Parser TOML puro Mire es lento | Media | Medio | Benchmark vs C; optimizar; fallback temporal |
| `.a` ausente con `--bundle static` | Media | Medio | Si hay `.so`, W0069 y `static-partial`; si no hay ninguna, E0404; `static-strict` siempre falla |
| Diagnósticos nuevos cambian goldens | Alta | Bajo | `--snapshot` y revisión por PR |
| ICE por `ByteSpan` mal calculado | Media | Alto | Tests de offsets con UTF-8/tabs/CRLF |
| `-L` cambia de significado | Alta | Medio | Shim §1.1 + warning + docs |
| libsodium sigue en runtime | Media | Alto | F9 explícita; gate en `scratch` |

---

## 18. Preguntas abiertas

Solo quedan las que dependen de decisiones de producto, no de ingeniería:

1. **`--bundle static` y licencias:** ¿se permite embeber `.a` con licencias no compatibles (p. ej. GPL) en binarios de usuario? Propuesta: sí, pero `mire doctor --licenses` lo reporta.
2. **`mire fix` en CI:** ¿debe existir `--check` que falle si hay fixes aplicables? Propuesta: sí, opt-in.
3. **Idioma de los mensajes:** ¿inglés siempre, o traducción de `explain`? Propuesta: inglés en el compilador; `explain` traducible por archivo.
4. **`symbols/` vs `symbols/` por módulo:** ¿un archivo por símbolo o un archivo por grupo? Propuesta: uno por símbolo, con `mod.mr` que los agrega.
5. **Baseline de warnings:** ¿se versiona `baseline.json` en el repo? Propuesta: sí, en `ci/`.
6. **`--error-format sarif`:** ¿es requisito para el primer release de F7 o puede ir en F8? Propuesta: F7, porque GitHub lo consume directo.

---

## Anexo A — Catálogo de códigos

### A.1 Errores existentes (sin cambio de significado)

| Código | Nombre | Dominio |
|--------|--------|---------|
| E0001 | `lexical_error` | Léxico |
| E0003 | `syntax_error` | Parser |
| E0005 | `type_mismatch` | Tipos |
| E0007 | `use_after_move` | Ownership (alias E0201) |
| E0008 | `multiple_mutable_borrows` | Ownership (alias E0202) |
| E0009 | `mutation_while_shared` | Ownership (alias E0203) |
| E0010 | `move_while_borrowed` | Ownership (alias E0204) |
| E0011 | `drop_while_borrowed` | Ownership (alias E0205) |
| E0013 | `borrow_out_of_scope` | Ownership (alias E0206) |
| E0014 | `backend_limitation` | Backend |
| E0015 | `runtime_error` | Runtime |
| E0017 | `cli_error` | CLI |
| E0022 | `extern_not_allowed` | Seguridad |
| E0100-E0110 | `precision_loss` … `numeric_kind_mismatch` | Tipos |

### A.2 Errores nuevos

| Código | Nombre | Fase |
|--------|--------|------|
| E0025 | `unresolved_import` | F2 |
| E0026 | `ambiguous_import` | F2 |
| E0201-E0208 | Ownership detallado | F7 |
| E0301-E0304 | Backend | F7 |
| E0401-E0406 | Link | F4/F9 |
| E0501-E0504 | Cache | F3 |
| E0601-E0606 | Paquetes | F1 |
| E0701-E0706 | CLI | F7 |
| E0801-E0804 | Runtime/PAL | F4 |
| E0901-E0905 | ICE/fallo fatal | F7 |

### A.3 Warnings

| Código | Nombre | Grupo |
|--------|--------|-------|
| W0001 | `unused_variables` | unused |
| W0002 | `dead_code` | unused |
| W0004-W0009 | tipos/lógica | correctness |
| W0010 | `deprecated_syntax` | deprecated |
| W0011-W0014 | complejidad/estilo | complexity/style |
| W0017-W0019 | lógica | logic |
| W0021 | `negative_index` | correctness |
| W0024-W0025 | literales | style/memory |
| W0034-W0049 | estilo/lógica | varios |
| W0050 | `unused_must_use` | correctness |
| W0051 | `unused_result` | correctness |
| W0052 | `unreachable_code` | correctness |
| W0053 | `unused_extern` | unused |
| W0054 | `stale_security_allowlist` | correctness |
| W0055 | `clock_regression` | suspicious |
| W0056 | `shell_invocation` | security |
| W0057 | `redundant_clone` | performance |
| W0058 | `needless_borrow` | style |
| W0059 | `deprecated_api` | deprecated |
| W0060 | `implicit_narrowing` | correctness |
| W0061 | `legacy_link_flag` | deprecated |
| W0062 | `cache_schema_migrated` | note |
| W0063 | `shadowed_builtin` | suspicious |
| W0064 | `unsafe_block` | security |
| W0065 | `duplicate_flag` | CLI |
| W0066 | `deprecated_japanese_alias` | deprecated |
| W0067 | `else_on_new_line` | style |
| W0068 | `non_mire_case` | style |
| W0069 | `static_bundle_fallback` | linker |

---

## Anexo B — Checklist por release

Antes de cada release del ecosistema:

- [ ] `scripts/reset-tree.sh --dry-run` no reporta nada inesperado.
- [ ] `git status --porcelain` vacío tras `owl test` en cada repo.
- [ ] `mire docs errors --check` sin drift.
- [ ] `mire docs errors --check-coverage` 100 %.
- [ ] `mire help --check` sin drift.
- [ ] `mire test --release -O3` verde.
- [ ] `avenys/bench/run.sh` sin regresión > 2 %.
- [ ] `owl build` en proyecto fresco: 0 red, < 2 s.
- [ ] `mire cache verify` sin errores.
- [ ] `mire doctor` sin dependencias faltantes en el entorno de CI.
- [ ] Binario release de `owl` corre en contenedor `scratch`.
- [ ] Ningún `test_*.mire` fuera de `tests/`.
- [ ] Ningún directorio con solo `owl.toml`.
- [ ] Ningún `extern` fuera de la allowlist (`kioto/tests/security/allowlist.mire`).
- [ ] `CHANGELOG.md` con sección de breaking changes y migración.
- [ ] Los códigos deprecados siguen funcionando y avisan.

---

## Cierre

Este plan no pide una migración absoluta. Pide, en este orden:

1. **Parar de acumular deuda:** borrar stubs y fósiles (F0).
2. **Arreglar lo que bloquea:** lockfile y resolución offline (F1).
3. **Definir la frontera:** Owl resuelve, Avenys compila, nadie parsea TOML de más (F2, F5).
4. **Hacer que la cache sea confiable:** sin “cache exorcism” (F3).
5. **Sacar el C del runtime:** PAL en Mire, con benchmarks como juez (F4).
6. **Medir cada símbolo:** tests unitarios reales (F6).
7. **Explicar cada error:** diagnósticos que enseñan y corrigen (F7).
8. **Mejorar la sintaxis sin romper:** aditivo primero, deprecación después (F8).
9. **Distribuir binarios que arrancan:** `--bundle static` y `mire doctor` (F9).

El criterio de éxito no es “todo está reescrito”. Es que un usuario nuevo pueda:

```bash
owl new hola
cd hola
owl run
```

…sin red, sin borrar cache, sin instalar libsodium, y con un mensaje claro si algo falla.
