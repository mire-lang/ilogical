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
| [web](./web/) | HTTP client, request/response, headers, cookies |
| [ws](./ws/) | WebSocket client/server implementation |
| [serve](./serve/) | `owl serve` command reference |
| [benchmarks](./benchmarks/) | Performance benchmarks |
| [bugs](./bugs/) | Known issues and workarounds |

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
mire test tests/ --lib-dir /path/to/ilogical
```

## Known Issues

See [bugs](./bugs/) for current limitations.
