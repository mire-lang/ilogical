# ilogical::format — Serialization Formats

Codec implementations for JSON, TOML, and binary formats. All decode to the common `base::Value` representation.

## JSON

```mire
pub fn json_encode: (v :i64) :str
pub fn json_decode: (s :str) :i64
pub fn json_encode_pretty: (v :i64, indent :str) :str
```

## TOML

```mire
pub fn toml_encode: (v :i64) :str
pub fn toml_decode: (s :str) :i64
```

## Binary (MessagePack-inspired)

```mire
pub fn bin_encode: (v :i64) :i64
pub fn bin_decode: (data :&ptr, len :i64) :i64
pub fn bin_encode_to_ptr: (v :i64, out :&mut ptr, out_len :&mut i64)
```

## Round-trip Conversion

Since all formats decode to the same `base::Value`, you can convert between formats without intermediate strings:

```mire
set json = format::json_encode(value)
set decoded = format::json_decode(json)
set toml = format::toml_encode(decoded)
set bin = format::bin_encode(decoded)
```

## Implementation Status

- JSON: Stub (delegates to PAL `rt_json_*`)
- TOML: Stub (delegates to PAL `rt_toml_*`)
- Binary: Stub (delegates to PAL `rt_bin_*`)

Runtime implementations needed in PAL.
