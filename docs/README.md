# ilogical Documentation

Modular documentation for the ilogical library.

## Table of Contents

| Module | Description |
|--------|-------------|
| [base](./base/) | Core Value type, constructors, arrays, objects, cloning |
| [format](./format/) | JSON, TOML, Binary serialization |
| [schema](./schema/) | Schema definition and validation |
| [encoding](./encoding/) | Base64, Hex, Percent encoding |
| [url](./url/) | URL parsing, building, resolution |
| [web](./web/) | HTTP message types: methods, headers, request/response, client config, cookies |
| [benchmarks](./benchmarks/) | Performance benchmarks |
| [bugs](./bugs/) | Known limits and deliberate omissions |

## Quick Start

```mire
load ilogical::base

pub fn main: () {
    set arr = base::array() :i64 mut
    set arr = base::array_push(arr, base::int(1))
    set arr = base::array_push(arr, base::int(2))
    use dasu(base::to_string(arr))  // "[1, 2]"
}
```

## Testing

```bash
# from this repository
mire test tests --lib-dir "$PWD/.."
```

`--lib-dir` names a directory of packages, so it points at the parent of this
repository. The suites live under `tests/`, one directory per module, with a
`symbols/` subdirectory holding the per-symbol cases.

## Known limits

See [bugs](./bugs/) for the deliberate omissions: no HTTP transport, TOML
documents are tables, no custom schema validator, and no streaming codecs.
