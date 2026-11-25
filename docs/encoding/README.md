# ilogical::encoding — Base64, Hex, Percent Encoding

## Base64

```mire
pub fn base64_encode: (data :&ptr, len :i64) :str
pub fn base64_decode: (s :str) :i64
pub fn base64_decode_to_ptr: (s :str, out :&mut ptr, out_len :&mut i64)
pub fn base64_url_encode: (data :&ptr, len :i64) :str
pub fn base64_url_decode: (s :str) :i64
```

Standard and URL-safe (RFC 4648) variants.

## Hex

```mire
pub fn hex_encode: (data :&ptr, len :i64) :str
pub fn hex_decode: (s :str) :i64
pub fn hex_decode_to_ptr: (s :str, out :&mut ptr, out_len :&mut i64)
```

Uppercase hex output.

## Percent (URL) Encoding

```mire
pub fn percent_encode: (s :str) :str
pub fn percent_decode: (s :str) :str
```

RFC 3986 compliant. Encodes all except alphanumeric, `-`, `.`, `_`, `~`.

## Implementation

Pure Mire, over `mire::bytes` and `mire::aux::pal`. There is no PAL codec call:
the base64, hex and percent alphabets are built in `ilogical` itself, so the
result is the same in every runtime tier.

Decoding returns an `i64` handle to a byte value; read it with
`base::as_bytes_ptr` and `base::as_bytes_len`. Use the `_to_ptr` variants
(`base64_decode_to_ptr`, `base64_url_decode_to_ptr`, `hex_decode_to_ptr`) when
the caller wants the raw pointer and length without wrapping them in a value.
