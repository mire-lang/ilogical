# ilogical::web — HTTP Client & Types

## HTTP Methods

```mire
pub fn METHOD_GET: () :i64
pub fn METHOD_POST: () :i64
pub fn METHOD_PUT: () :i64
pub fn METHOD_DELETE: () :i64
pub fn METHOD_PATCH: () :i64
pub fn METHOD_HEAD: () :i64
pub fn METHOD_OPTIONS: () :i64

pub fn method_name: (m :i64) :str
pub fn method_from_str: (s :str) :i64
```

The methods are nullary functions, not `cons` constants: Mire cannot read a
`cons` through a dotted path.

## Headers

```mire
pub fn headers_new: () :i64
pub fn headers_set: (h :i64, name :str, value :str) :i64
pub fn headers_get: (h :i64, name :str) :str
pub fn headers_has: (h :i64, name :str) :bool
pub fn headers_remove: (h :i64, name :str) :i64
pub fn headers_count: (h :i64) :i64
pub fn headers_key_at: (h :i64, index :i64) :str
pub fn headers_value_at: (h :i64, index :i64) :str
```

Headers are `base::Value` objects (string-to-string map).

## Request

```mire
pub fn request_new: (method :i64, url :i64) :i64
pub fn request_method: (req :i64) :i64
pub fn request_url: (req :i64) :i64
pub fn request_headers: (req :i64) :i64
pub fn request_body: (req :i64) :i64
pub fn request_set_headers: (req :i64, h :i64) :i64
pub fn request_set_body: (req :i64, body :i64) :i64
pub fn request_set_header: (req :i64, name :str, value :str) :i64
```

Request struct: `[method][url][headers][body]`

## Response

```mire
pub fn response_new: (status :i64) :i64
pub fn response_status: (resp :i64) :i64
pub fn response_headers: (resp :i64) :i64
pub fn response_body: (resp :i64) :i64
pub fn response_set_headers: (resp :i64, h :i64) :i64
pub fn response_set_body: (resp :i64, body :i64) :i64
pub fn response_set_header: (resp :i64, name :str, value :str) :i64
```

Response struct: `[status][headers][body]`

## Client

```mire
pub fn client_new: (base_url :str) :i64
pub fn client_base_url: (client :i64) :i64
pub fn client_timeout: (client :i64) :i64
pub fn client_set_timeout: (client :i64, millis :i64) :i64
pub fn client_set_default_header: (client :i64, name :str, value :str) :i64
```

A client carries configuration only. This package does not send requests;
transport lives in `kioto::net`, so a program builds the request here and hands
it to `kioto` to execute. See
[known limits](../bugs/README.md#no-http-transport).

## Cookies

```mire
pub fn cookie_parse: (header :str) :i64
pub fn cookie_serialize: (cookie :i64) :str
pub fn cookie_jar_new: () :i64
pub fn cookie_jar_add: (jar :i64, cookie :i64) :i64
pub fn cookie_jar_get: (jar :i64, name :str) :i64
pub fn cookie_jar_remove: (jar :i64, name :str) :i64
```

A cookie is `[name][value][attributes]`; attributes are a string-to-string
object. `cookie_parse` accepts a `Cookie:` header value and returns a cookie
value.

## Implementation

Everything on this page is pure Mire over `base::Value`. The structures are
plain objects, so a caller can inspect them with `base::object_get` or
serialize them with `ilogical::format`. There is no PAL call and no transport.
