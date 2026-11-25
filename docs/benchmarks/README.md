# Performance Benchmarks

## Environment

- CPU: AMD Ryzen (tested on)
- Runtime: Mire minimal tier
- Library: ilogical v0.1.0

## Array Operations

| Operation | Count | Time | Per-op |
|-----------|-------|------|--------|
| `array_push` | 10,000 | ~10.7ms | ~1.07 µs/op |
| `array_get` | 10,000 | ~0.8ms | ~0.08 µs/op |
| `array_len` | 10,000 | ~0.3ms | ~0.03 µs/op |

Array push uses custom dynamic buffer with 2x growth strategy.

## Object Operations

| Operation | Count | Time | Status |
|-----------|-------|------|--------|
| `object_set` | 1,000 | N/A | **CRASH** |

**Object operations crash (SIGSEGV)** due to `object_buf_push` buffer reallocation bug.

## Clone & Equality

| Operation | Count | Time | Per-op |
|-----------|-------|------|--------|
| `clone` (array, 100 items) | 1,000 | ~5ms | ~5 µs/op |
| `equals` (array, 100 items) | 1,000 | ~3ms | ~3 µs/op |

Deep structural equality with recursive comparison.

## Serialization

| Format | Operation | Status |
|--------|-----------|--------|
| JSON | encode/decode | Stub (PAL) |
| TOML | encode/decode | Stub (PAL) |
| Binary | encode/decode | Stub (PAL) |

Runtime PAL implementations needed.

## Memory

- Value handle: 8 bytes (i64)
- Record: 64 bytes fixed
- Array buffer: 16 bytes header + 8 bytes/item
- Object buffer: 16 bytes header + 16 bytes/item (key ptr + value)

## Test Suite Performance

| Test Suite | Tests | Time |
|------------|-------|------|
| base (working subset) | 5 | <10ms |
| base (full) | 26 | Crash on objects |

## Recommendations

1. Fix `object_buf_push` buffer reallocation
2. Add PAL implementations for format/encoding
3. Consider arena allocation for high-frequency operations
