# ilogical::url — URL Parsing & Manipulation

## Parsing

```mire
pub fn parse: (s :str) :i64
pub fn parse_with_base: (s :str, base :i64) :i64
```

Returns a `URL` struct handle.

## Building

```mire
pub fn build: () :i64
pub fn set_scheme: (url :i64, scheme :str) :i64
pub fn set_host: (url :i64, host :str) :i64
pub fn set_port: (url :i64, port :i64) :i64
pub fn set_path: (url :i64, path :str) :i64
pub fn set_query: (url :i64, query :i64) :i64
pub fn set_fragment: (url :i64, fragment :str) :i64
```

## Query Parameters

```mire
pub fn query_parse: (s :str) :i64
pub fn query_stringify: (params :i64) :str
pub fn query_get: (params :i64, key :str) :str
pub fn query_set: (params :i64, key :str, value :str) :i64
pub fn query_remove: (params :i64, key :str) :i64
```

Query params are `base::Value` objects.

## Resolution

```mire
pub fn resolve: (base :i64, relative :i64) :i64
pub fn resolve_str: (base :str, relative :str) :str
```

RFC 3986 compliant relative resolution.

## Serialization

```mire
pub fn to_string: (url :i64) :str
pub fn scheme: (url :i64) :str
pub fn host: (url :i64) :str
pub fn port: (url :i64) :i64
pub fn path: (url :i64) :str
pub fn query: (url :i64) :i64
pub fn fragment: (url :i64) :str
```

## URL Struct

```
offset 0: scheme ptr
offset 8: host ptr
offset 16: port (i64)
offset 24: path ptr
offset 32: query ptr (base::Value object)
offset 40: fragment ptr
```
