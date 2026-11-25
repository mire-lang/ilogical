# ilogical::web — HTTP Client & Types

## HTTP Methods

```mire
cons METHOD_GET = 0
cons METHOD_POST = 1
cons METHOD_PUT = 2
cons METHOD_DELETE = 3
cons METHOD_PATCH = 4
cons METHOD_HEAD = 5
cons METHOD_OPTIONS = 6

pub fn method_name: (m :i64) :str
pub fn method_from_str: (s :str) :i64
```

## Headers

```mire
pub fn headers_new: () :i64
pub fn headers_set: (h :i64, name :str, value :str) :i64
pub fn headers_get: (h :i64, name :str) :str
pub fn headers_has: (h :i64, name :str) :bool
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
```

Response struct: `[status][headers][body]`

## Client

```mire
pub fn client_new: (base_url :str) :i64
pub fn client_request: (client :i64, req :i64) :i64
pub fn client_get: (client :i64, url :str) :i64
pub fn client_post: (client :i64, url :str, body :i64) :i64
pub fn client_put: (client :i64, url :str, body :i64) :i64
pub fn client_delete: (client :i64, url :str) :i64
```

## Cookies

```mire
pub fn cookie_parse: (header :str) :i64
pub fn cookie_serialize: (cookie :i64) :str
pub fn cookie_jar_new: () :i64
pub fn cookie_jar_add: (jar :i64, cookie :i64) :i64
pub fn cookie_jar_get: (jar :i64, name :str) :i64
```

## Implementation Status

All functions are stubs delegating to PAL runtime (`rt_client_request`, `rt_cookie_*`). Runtime implementations needed.
