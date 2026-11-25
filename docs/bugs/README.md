# Known Bugs & Workarounds

## BUG-001: object_buf_push Buffer Reallocation Crash

**Severity**: Critical  
**Module**: `ilogical::base` (object operations)  
**Status**: Open

### Description

When `object_buf_push` needs to grow the object buffer (exceeds capacity), it allocates a new buffer, copies existing entries, and returns the new buffer pointer. However, the original buffer pointer stored in the Value record becomes stale, causing SIGSEGV on subsequent access.

### Reproduction

```mire
set o = base::object() :i64 mut
set i = 0 :i64 mut
while i < 5 {  // Exceeds initial capacity of 4
    set key = "k" + base::to_string(base::int(i))
    set o = base::object_set(o, key, base::int(i))
    set i = i + 1
}
```

### Root Cause

In `object_buf_push` (base/mod.mire:204-234):
1. New buffer allocated with `bytes::alloc`
2. Existing entries copied to new buffer
3. New entry added
4. New buffer returned

But the caller (`object_set`) only updates the Value record if `new_buf != obj_buf`. The comparison may fail due to pointer aliasing, or the old buffer is freed while still referenced.

### Workaround

Avoid exceeding initial capacity (4):
```mire
// Pre-size by setting many fields at once
set o = base::object() :i64 mut
set o = base::object_set(o, "a", base::int(1))
set o = base::object_set(o, "b", base::int(2))
set o = base::object_set(o, "c", base::int(3))
set o = base::object_set(o, "d", base::int(4))
// 5th field will crash
```

### Fix Required

In `object_buf_push`:
1. Don't free old buffer until Value record updated
2. Or use `mire::vec` with proper ownership semantics
3. Or implement copy-on-write semantics

## BUG-002: clone_object Crashes

**Severity**: Critical  
**Module**: `ilogical::base`  
**Status**: Open

### Description

`clone` calls `clone_object` which uses `object_buf_clone`. This crashes when cloning objects with >4 fields due to same buffer issue as BUG-001.

## BUG-003: Nested Array Clone Crash

**Severity**: High  
**Module**: `ilogical::base`  
**Status**: Open

### Description

Cloning arrays containing other arrays crashes because `clone_array` calls `clone` recursively on child arrays, which eventually calls `clone_object` if the array contains objects.

## BUG-004: Mire bytes::write_u64 Offset Bug

**Severity**: High  
**Module**: Mire stdlib  
**Status**: Workaround Applied

### Description

`bytes::write_u64` corrupts the base pointer when writing at offsets 8, 24, 40, 56. This is why ilogical uses `pal::ptr_add` + `pal::write_u64` directly.

### Workaround

All record field access uses:
```mire
fn write_u64_off: (base :ptr, offset :i64, value :i64) {
    set p = pal::ptr_add(base, offset)
    pal::write_u64(p, value)
}
```

## BUG-005: Format/Encoding/URL/Web Stubs

**Severity**: Medium  
**Module**: Multiple  
**Status**: Planned

### Description

`format`, `encoding`, `url`, `web` modules are skeleton implementations delegating to PAL runtime functions that don't exist yet (`rt_json_*`, `rt_base64_*`, `rt_url_parse`, `rt_client_request`, etc.).

### Required PAL Functions

- `rt_json_encode`, `rt_json_decode`, `rt_json_encode_pretty`
- `rt_toml_encode`, `rt_toml_decode`
- `rt_bin_encode`, `rt_bin_decode`
- `rt_base64_encode`, `rt_base64_decode`, `rt_base64_url_encode`, `rt_base64_url_decode`
- `rt_hex_encode`, `rt_hex_decode`
- `rt_percent_encode`, `rt_percent_decode`
- `rt_url_parse`, `rt_url_parse_with_base`, `rt_url_resolve`, `rt_url_resolve_str`, `rt_url_to_string`, `rt_query_parse`, `rt_query_stringify`
- `rt_client_request`, `rt_cookie_parse`, `rt_cookie_serialize`
- `rt_ws_encode_frame`, `rt_ws_decode_frame`
- `pal_random_byte`

## BUG-006: Test Cyclic Load

**Severity**: Low  
**Module**: Test infrastructure  
**Status**: Workaround Applied

### Description

`load ilogical::tests::base` causes cyclic load detection because `tests/mod.mr` loads `ilogical::base` and test modules load `ilogical::tests::*`.

### Workaround

Tests embedded directly in test runner file instead of separate modules.
