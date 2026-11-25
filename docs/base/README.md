# ilogical::base — Core Value Type

The foundation of ilogical: a single dynamic `Value` type that represents all JSON-compatible data plus binary blobs.

## Type System

`Value` is an `i64` handle to a 64-byte heap record. The record layout:

| Offset | Field | Description |
|--------|-------|-------------|
| 0 | kind | Type tag (K_NULL=0, K_BOOL=1, K_INT=2, K_FLOAT=3, K_STRING=4, K_ARRAY=5, K_OBJECT=6, K_BYTES=7, K_ERROR=8) |
| 8 | integer | Boolean (0/1) or i64 value |
| 16 | float | f64 bit pattern |
| 24 | string | Pointer to managed string |
| 32 | array | Pointer to custom array buffer |
| 40 | object keys | Pointer to custom object buffer |
| 48 | object vals | Same as keys (shared buffer) |
| 56 | length | Byte length for blobs, element count for arrays/objects |

## Constructors

| Function | Signature | Description |
|----------|-----------|-------------|
| `null` | `() :i64` | Null value |
| `bool` | `(b :bool) :i64` | Boolean |
| `int` | `(n :i64) :i64` | Signed 64-bit integer |
| `float` | `(f :f64) :i64` | IEEE 754 double |
| `string` | `(s :str) :i64` | UTF-8 string |
| `error` | `(msg :str) :i64` | Error with message |
| `array` | `() :i64` | Empty array |
| `object` | `() :i64` | Empty object |
| `blob` | `(data :&ptr, len :i64) :i64` | Binary blob |

## Type Inspection

```mire
pub fn kind: (h :i64) :i64
pub fn is_null: (h :i64) :bool
pub fn is_bool: (h :i64) :bool
pub fn is_int: (h :i64) :bool
pub fn is_float: (h :i64) :bool
pub fn is_string: (h :i64) :bool
pub fn is_array: (h :i64) :bool
pub fn is_object: (h :i64) :bool
pub fn is_bytes: (h :i64) :bool
pub fn is_error: (h :i64) :bool
pub fn type_name: (h :i64) :str
```

## Accessors

```mire
pub fn as_bool: (h :i64) :bool
pub fn as_int: (h :i64) :i64
pub fn as_float: (h :i64) :f64
pub fn as_string: (h :i64) :str
pub fn as_bytes_ptr: (h :i64) :ptr
pub fn as_bytes_len: (h :i64) :i64
```

## Array Operations

Arrays use custom dynamic buffers (not `mire::vec`) to avoid Mire's `bytes::write_u64` bug at offsets 8, 24, 40, 56.

```mire
pub fn array_len: (h :i64) :i64
pub fn array_get: (h :i64, index :i64) :i64
pub fn array_push: (h :i64, item :i64) :i64
```

Array buffer layout: `[capacity: i64][length: i64][data: i64 * capacity]`

Growth strategy: 2x capacity when full (minimum 4).

## Object Operations

Objects use custom dynamic buffers with separate keys/values arrays.

```mire
pub fn object_len: (h :i64) :i64
pub fn object_has: (h :i64, key :&str) :bool
pub fn object_get: (h :i64, key :&str) :i64
pub fn object_key_at: (h :i64, index :i64) :str
pub fn object_value_at: (h :i64, index :i64) :i64
pub fn object_set: (h :i64, key :str, item :i64) :i64
```

Object buffer layout: `[capacity][length][keys: ptr * cap][values: i64 * cap]`

## Clone & Equality

```mire
pub fn clone: (h :i64) :i64
pub fn equals: (a :i64, b :i64) :bool
```

Deep structural equality with type coercion for int/float.

## Debug Representation

```mire
pub fn to_string: (h :i64) :str
```

Outputs JSON-like format: `null`, `true`/`false`, numbers, strings, `[...]`, `{...}`, `<N bytes>`.

## Known Issues

- **object_set buffer reallocation**: When `object_buf_push` allocates a new buffer, the old buffer pointer in the Value record may become stale. Workaround: avoid rapid sequential `object_set` calls on the same object, or pre-size with many fields at once.

## Implementation Notes

- Uses `pal::ptr_add` + `pal::read_u64`/`pal::write_u64` instead of `bytes::read_u64`/`bytes::write_u64` to avoid Mire's bug at offsets 8, 24, 40, 56.
- All operations take `i64` handles directly (not `&Value`) to avoid Mire's struct field access bug with references.
- Values are Copy (just i64) so ownership is simple.
