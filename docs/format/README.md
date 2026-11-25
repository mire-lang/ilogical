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

## Errors and limits

- **JSON** is written and parsed here, in Mire. A decode failure returns an
  error `Value` carrying the position, line and column; test it with
  `base::is_error`. A trailing token after the document is an error, not
  ignored. `json_encode` of an error value writes `null`.
- **TOML** documents are tables. `toml_encode` of a non-object writes an inline
  fragment, which `toml_decode` will reject; wrap scalars and arrays in an
  object when the output must round-trip. See
  [known limits](../bugs/README.md#toml-documents-are-tables).
- **Binary** is a MessagePack-inspired format private to this package. It is
  versioned by the first byte and is not compatible with MessagePack decoders;
  `bin_decode` returns an error `Value` on a version or tag mismatch.
- None of the codecs stream: the input must be a complete `str` or buffer.
