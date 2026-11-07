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

All functions delegate to PAL runtime (`rt_base64_*`, `rt_hex_*`, `rt_percent_*`). Runtime implementations needed.
