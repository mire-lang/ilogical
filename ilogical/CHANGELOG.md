# Changelog

All notable changes to the ilogical library will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [0.1.0] - 2026-11-06

### Added
- **ilogical::base** - Core dynamic Value type with:
  - Value constructors: `null()`, `bool()`, `int()`, `float()`, `string()`, `error()`, `array()`, `object()`, `blob()`
  - Type inspection: `kind()`, `is_*()`, `type_name()`
  - Accessors: `as_bool()`, `as_int()`, `as_float()`, `as_string()`, `as_bytes_len()`, `as_bytes_ptr()`
  - Array operations: `array_len()`, `array_get()`, `array_push()` with custom dynamic buffers
  - Object operations: `object_len()`, `object_has()`, `object_get()`, `object_key_at()`, `object_value_at()`, `object_set()` with custom dynamic buffers
  - Clone and equality: `clone()`, `equals()` with deep comparison
  - Debug representation: `to_string()` for all types
  - Custom array/object buffers using `pal::ptr_add` to avoid Mire's `bytes::write_u64` bug at offsets 8, 24, 40, 56

- **ilogical::format** - Format codecs:
  - JSON: `json_encode()`, `json_decode()`, `json_encode_pretty()`
  - TOML: `toml_encode()`, `toml_decode()`
  - Binary (MessagePack-inspired): `bin_encode()`, `bin_decode()`, `bin_encode_to_ptr()`
  - Round-trip conversion between JSON/TOML/Binary without intermediate strings

- **ilogical::schema** - Schema definition and validation:
  - Schema types: `any()`, `type_schema()`, `object_schema()`, `array_schema()`, `union_schema()`, `custom_schema()`
  - Object field definitions with required/optional flags
  - Recursive validation with `validate()`
  - Support for type, object, array, union, and custom validators

- **ilogical::encoding** - Encoding/decoding:
  - Base64: `base64_encode()`, `base64_decode()`, `base64_url_encode()`, `base64_url_decode()`
  - Hex: `hex_encode()`, `hex_decode()`
  - Percent/URL: `percent_encode()`, `percent_decode()`

- **ilogical::url** - URL handling:
  - Parsing: `parse()`, `parse_with_base()`
  - Building: `build()`, `set_scheme()`, `set_host()`, `set_port()`, `set_path()`, `set_query()`, `set_fragment()`
  - Query parameters: `query_parse()`, `query_stringify()`, `query_get()`, `query_set()`, `query_remove()`
  - Resolution: `resolve()`, `resolve_str()`
  - Serialization: `to_string()`, getters for all components

- **ilogical::web** - HTTP/Web types:
  - Methods: GET, POST, PUT, DELETE, PATCH, HEAD, OPTIONS
  - Headers: `headers_new()`, `headers_set()`, `headers_get()`, `headers_has()`
  - Request/Response: `request_new()`, `response_new()` with full builder APIs
  - Client: `client_new()`, `client_get()`, `client_post()`, `client_put()`, `client_delete()`
  - Cookies: `cookie_jar_new()`, `cookie_parse()`, `cookie_serialize()`

- **kioto::net::ws** - WebSocket implementation:
  - Client: `connect()` with automatic handshake
  - Server: `serve()`, `accept()` with server-side handshake
  - Frames: text, binary, ping, pong, close
  - Opcodes: continuation, text, binary, close, ping, pong
  - Masking for client-to-server frames
  - Close codes per RFC 6455

- **owl serve** - Native WebSocket/HTTP server command:
  - `owl serve <IP:PORT> [--ws] [--http] [--dir <DIR>] [--cert <FILE>] [--key <FILE>]`
  - WebSocket upgrade support
  - Static file serving
  - TLS support

### Testing
- Comprehensive test suite covering all modules
- Tests follow Avenys test patterns with measurable assertions
- Console output via `dasu()` for test visibility
- Test modules: `tests/base`, `tests/format`, `tests/schema`, `tests/encoding`, `tests/url`, `tests/web`

### Known Limitations
- `object_set()` has a known issue with buffer reallocation in the base module
- Object operations work correctly in isolated tests but may crash in certain contexts
- Runtime PAL implementations for format/schema/encoding/url/web are stubs

### Internal
- Uses `pal::ptr_add` and `pal::read_u64`/`pal::write_u64` for safe pointer arithmetic
- Avoids Mire's `bytes::write_u64` bug at specific offsets
- Custom dynamic buffers for arrays and objects instead of `mire::vec`