# Known limits

This page records what `ilogical` deliberately does not do. It is not a bug
list: the objects-grow and clone crashes that earlier revisions of this file
described were fixed, and the suites under `tests/base/` cover objects past the
initial capacity and nested clones.

## No HTTP transport

`ilogical::web` defines the message types — methods, headers, request,
response, client options and cookies — as `base::Value` structures. It does not
open sockets. Transport belongs to `kioto::net`; a program composes a request
with `web`, sends it with `kioto`, and builds a response type from the result.

This is why there are no `client_request`, `client_get` or `client_post`
functions. `client_new`, `client_base_url`, `client_timeout`,
`client_set_timeout` and `client_set_default_header` only carry configuration.

## TOML documents are tables

`toml_encode` on an object writes a TOML document. On any other value it writes
an inline TOML fragment (`42`, `[1, 2]`, `"text"`) followed by a newline, which
is not a valid TOML document and `toml_decode` rejects. Wrap a scalar or an
array in an object before encoding it if the result must round-trip.

Dotted keys, tables, arrays of tables, `[[...]]`, basic and literal strings
(single and multi-line), integers in decimal/hex/octal/binary with underscores,
floats, booleans, dates and times are accepted on decode.

## No custom schema validator

`ilogical::schema` validates `any`, `type`, `object`, `array` and `union`
schemas. There is no `custom_schema` function: a validator callback was never
implemented in the pure-Mire surface, so it was removed rather than exposed as
a stub. Validate application-specific rules by walking the value with
`base::*` after the structural check.

## Values are copied, not shared

Every `Value` is an owned record. `clone` deep-copies arrays and objects, so
cloning a large structure costs time proportional to its size. There is no
reference counting and no copy-on-write; `array_push` and `object_set` return a
new handle and may reallocate the underlying buffer.

## String decoding returns a handle

`base64_decode`, `base64_url_decode` and `hex_decode` return an `i64` handle to
a byte value, not a `str`. Use the `_to_ptr` forms when the caller wants the
raw bytes and length, or read the handle through the byte accessors.

## No streaming

The codecs take a complete `str` or byte buffer. There is no pull parser and no
incremental encoder, so a document must fit in memory.
