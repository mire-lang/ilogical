# Performance benchmarks

These numbers come from a probe built against the working tree, not from a
microbenchmark harness. They are indicative, not a contract: they were measured
on one machine with a debug-free build, and the compiler's codegen changes
between releases.

## Method

- Machine: x86-64, 8 logical CPUs, Linux
- Compiler: Avenys v4.4.44, `--release` profile
- Runtime: Mire minimal tier
- Package: ilogical 0.1.0
- Each case times a loop with `pal::time_monotonic_ns()` around it; the
  per-operation column divides the loop time by the operation count.

The probe lives outside the repository so it can be re-run without adding a
build target. The shape is:

```mire
load ilogical::base
load mire::aux::pal

set t0 = pal::time_monotonic_ns()
// ... loop ...
set t1 = pal::time_monotonic_ns()
```

## Results

| Operation | Count | Total | Per operation |
|---|---:|---:|---:|
| `array_push` (growing to 10,000) | 10,000 | 4.3 ms | 0.43 µs |
| `array_get` | 10,000 | 1.5 ms | 0.15 µs |
| `object_set` (growing to 1,000) | 1,000 | 21.9 ms | 21.9 µs |
| `clone` (array of 10,000) | 1,000 | 336 ms | 336 µs |
| `equals` (array of 100, same handle) | 1,000 | 20.9 ms | 20.9 µs |
| `json_encode` (small object) | 10,000 | 34.5 ms | 3.45 µs |
| `json_decode` (small object) | 10,000 | 86.2 ms | 8.62 µs |
| `toml_decode` (small object) | 10,000 | 104.5 ms | 10.45 µs |
| `percent_encode` (48-byte string) | 10,000 | 53.2 ms | 5.32 µs |
| `json_decode` (array of 10,000) | 1,000 | 11.3 s | 11.3 ms |

`json_decode` on a large array is the dominant case because it allocates one
`Value` record per element; the per-element cost is roughly `array_push` plus
parsing. `object_set` is slower than `array_push` because keys are hashed and
stored, and because growth copies both the key and value arrays.

## Memory layout

- `Value` handle: 8 bytes (an `i64`)
- Record: 64 bytes fixed
- Array buffer: 16-byte header + 8 bytes per item
- Object buffer: 16-byte header + 8 bytes per key pointer + 8 bytes per value

## Reproducing

Build a probe with the package as a library:

```sh
mire run bench.mire --release --runtime minimal --lib-dir /path/to/Arch
```

`--lib-dir` points at the directory that contains the `ilogical/` package.
