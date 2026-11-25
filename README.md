# ilogical

Logical data processing for [Mire](https://github.com/mire-lang/Avenys-rust),
in pure Mire.

`ilogical` is the ecosystem's dynamic value model and its codecs. A `Value` is a
single `i64` handle that can hold null, bool, int, float, string, bytes, array,
object or error. Arrays, objects, JSON, TOML, the binary format and the URL and
web types all build on it.

Version **0.1.0** — [CHANGELOG](CHANGELOG.md) ·
[Documentation](docs/README.md).

## Modules

| Module | Contents |
|--------|----------|
| `ilogical::base` | `Value` constructors, inspection, accessors, arrays, objects, `clone`, `equals`, `to_string` |
| `ilogical::buffer` | Growable byte buffer used by the encoders |
| `ilogical::format` | JSON, TOML and a MessagePack-inspired binary format, all pure Mire |
| `ilogical::schema` | Structural schemas: `any`, `type`, `object`, `array`, `union` |
| `ilogical::encoding` | Base64 (standard and URL-safe), hex and percent encoding |
| `ilogical::url` | RFC 3986 parsing, building, component access, query helpers, resolution |
| `ilogical::web` | HTTP message types: methods, headers, request, response, client config, cookies |

`ilogical::format` is the single owner of JSON and TOML in the ecosystem. Kioto
no longer ships `modules/json` or `modules/toml`.

## Quick start

```mire
load ilogical::base
load ilogical::format

pub fn main: () {
    set doc = base::object() :i64 mut
    set doc = base::object_set(doc, "name", base::string("ilogical"))
    set doc = base::object_set(doc, "stable", base::bool(false))

    set text = format::json_encode(doc)
    use dasu(text)                      // {"name":"ilogical","stable":false}

    set back = format::json_decode(text)
    use dasu(base::as_string(base::object_get(back, "name")))
}
```

## Requirements

- Avenys 4.4 or newer, default `minimal` runtime
- No C dependency: the package links no codec library and declares no native
  sources

## Building and testing

```sh
# from this repository
mire test tests --lib-dir "$PWD/.."
```

`--lib-dir` names a directory of packages, so it points at the parent of this
repository. The suites live under `tests/`, one directory per module, with a
`symbols/` subdirectory holding per-symbol cases; each case is a success, a
limit or an anomaly.

## Known limits

- `ilogical::web` defines message types and client configuration only; there is
  no transport. Use `kioto::net` for sockets.
- TOML documents are tables. `toml_encode` on a scalar or an array writes an
  inline fragment that `toml_decode` rejects; wrap it in an object to
  round-trip.
- Schemas are structural. There is no custom validator callback.
- The codecs are not streaming: input must be a complete string or buffer.

See [docs/bugs](docs/bugs/README.md) for the details.

## Layout

```
src/
  base/      Value model, arrays, objects
  buffer/    byte buffer
  format/    JSON, TOML, binary
  schema/    structural schemas
  encoding/  base64, hex, percent
  url/       URL and query handling
  web/       HTTP message types
tests/       per-module suites with symbols/ cases
docs/        module guides
```

## License

See the repository for the license text.
