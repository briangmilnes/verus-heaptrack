# verus-heaptrack

Memory profiling and allocator benchmarking for the [Verus](https://github.com/verus-lang/verus) verification tool.

## Summary

Switching `rust_verify` from glibc malloc to jemalloc saves **10-13% peak RSS** and
**4-14% wall time** with zero code changes. The effect is consistent across 6 open-source
Verus projects ranging from 537 MB to 5.8 GB peak RSS.

## Cross-Project Results

glibc default vs jemalloc `narenas:2`, Verus `f04abf70` (rustc 1.94.0, `--features singular`):

| Project | Verified | glibc MB | jemal MB | RSS Δ | glibc s | jemal s | Time Δ |
|---------|----------|----------|----------|-------|---------|---------|--------|
| human-eval | 292 | 614 | 537 | -13% | 5.4s | 5.1s | -4% |
| ironkv | 319 | 783 | 706 | -10% | 5.4s | 5.1s | -5% |
| node-replication | 254 | 957 | 854 | -11% | 6.5s | 5.5s | -14% |
| vest | 496 | 648 | 565 | -13% | 5.7s | 5.2s | -9% |
| memory-allocator | 731 | 1,629 | 1,416 | -13% | 38.7s | 35.2s | -9% |
| APAS-VERUS | 4,265 | 5,822 | 5,143 | -12% | 59.2s | 53.2s | -10% |
| **Average** | | | | **-12%** | | | **-9%** |

## Why

`rust_verify` makes millions of small Arc allocations (16-400 bytes) intermixed with
temporary Vec growth allocations. glibc's per-thread arenas hold freed-but-resident pages
that cannot be reused across threads. jemalloc's size-class slab allocator recycles these
more efficiently, reducing both fragmentation and allocator overhead.

## Quick Start

For any Verus invocation on Linux:

```bash
LD_PRELOAD=/usr/lib/x86_64-linux-gnu/libjemalloc.so.2 \
MALLOC_CONF=narenas:2 \
rust_verify your_crate.rs
```

Requires `sudo apt install libjemalloc2`.

## For a Verus PR

Link jemalloc as the global allocator in `rust_verify/src/main.rs`:

```rust
#[global_allocator]
static GLOBAL: tikv_jemallocator::Jemalloc = tikv_jemallocator::Jemalloc;
```

Plus a `Cargo.toml` dependency on `tikv-jemallocator`. Benefits all users without
requiring `LD_PRELOAD`.

## Repository Contents

| Path | Description |
|------|-------------|
| `analyses/` | Detailed writeups: heaptrack findings, allocator comparison, benchmark systems |
| `scripts/` | Profiling and benchmarking scripts |
| `results/` | Raw benchmark output |
| `plans/` | Proposed Verus patches (path interning, SST pre-computation) |
| `patches/` | Experimental patches for Verus |
| `tests/fixtures/` | Cloned Verus projects used as benchmarks (not in git) |

## Benchmark Projects

The 6 benchmarked projects are drawn from Parno's [verita](https://github.com/verus-lang/verus/pull/2250)
fixture set. See [analyses/benchmark-systems-and-results.md](analyses/benchmark-systems-and-results.md)
for LOC counts, validation status, and invocation details.
