# Changelog

All notable changes to the ilogical library will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [0.1.0] - 2026-11-06

### Added

- **ilogical::base** — the dynamic `Value` type shared by every codec:
  - Constructors: `null()`, `bool()`, `int()`, `float()`, `string()`, `error()`, `array()`, `object()`, `blob()`
  - Kind constants and inspection: `K_*`, `kind()`, `is_*()`, `type_name()`
  - Accessors: `as_bool()`, `as_int()`, `as_float()`, `as_string()`, `as_bytes_ptr()`, `as_bytes_len()`
  - Arrays: `array_len()`, `array_get()`, `array_push()`
  - Objects: `object_len()`, `object_has()`, `object_get()`, `object_key_at()`, `object_value_at()`, `object_remove()`, `object_set()`
  - `clone()`, `equals()` and `to_string()`
- **ilogical::buffer** — growable byte buffer used by the encoders: `new()`, `len()`, `data()`, `reserve()`, `push_u8()`, `push_str()`, `push_i64()`, `push_f64()`, `to_str()`, `clear()`, `free()`.
- **ilogical::format** — JSON, TOML and binary codecs in pure Mire:
  - JSON: `json_decode()`, `json_encode()`, `json_encode_pretty()`
  - TOML: `toml_decode()`, `toml_encode()`
  - Binary (MessagePack-inspired): `bin_encode()`, `bin_encode_to_ptr()`, `bin_decode()`
  - Every decoder produces the same `base::Value`, so a document can be converted between formats without an intermediate string.
- **ilogical::schema** — structural schemas and validation:
  - `any()`, `type_schema()`, `object_schema()`, `array_schema()`, `union_schema()`
  - Field metadata: `object_field()`, `object_field_count()`, `object_field_required()`
  - `validate()` with recursive object, array and union checks
- **ilogical::encoding** — `base64_encode()`, `base64_decode()`, the URL-safe base64 variants, `hex_encode()`, `hex_decode()`, `percent_encode()`, `percent_decode()`.
- **ilogical::url** — `parse()`, `parse_with_base()`, `build()`, component setters and getters, `query_*()` helpers, `resolve()`, `resolve_str()` and `to_string()`.
- **ilogical::web** — HTTP message types and client configuration:
  - Methods and `method_name()` / `method_from_str()`
  - Headers: `headers_new()`, `headers_set()`, `headers_get()`, `headers_has()`, `headers_remove()`, `headers_count()`, `headers_key_at()`, `headers_value_at()`
  - Request and response builders
  - Client configuration: `client_new()`, `client_base_url()`, `client_timeout()`, `client_set_timeout()`, `client_set_default_header()`
  - Cookies: `cookie_new()`, `cookie_name()`, `cookie_value()`, `cookie_parse()`, `cookie_serialize()`, `cookie_jar_*()`

### Changed

- The library is pure Mire. The C JSON/TOML codecs were removed, the `[c]`
  section is gone, and the package depends only on `mire`.
- `ilogical` is the single owner of JSON and TOML in the ecosystem; Kioto no
  longer ships `modules/json` or `modules/toml`.

### Testing

- 154 per-symbol cases under `tests/<module>/symbols/*.mire`, covering success,
  limit and anomaly behaviour. Run them with:

  ```bash
  mire test tests --lib-dir "$PWD/.."
  ```

### Known limitations

- `web` provides message types and client configuration only. It has no
  transport; use `kioto::net` for sockets.
- Custom schema validators are not supported; schemas are structural.
- TOML documents are tables. `toml_encode()` on a non-object value emits an
  inline fragment, and `toml_decode()` rejects that fragment because it is not
  a document. Wrap the value in an object to round-trip it.
