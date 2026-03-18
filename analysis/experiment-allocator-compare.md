# Experiment: Allocator Comparison

**Date:** 2026-03-18
**Verus commit:** `76e69b81` (unpatched, via briangmilnes/verus-lang fork)
**APAS-VERUS commit:** `e3a5c74d`
**Machine load:** Quiet — no other significant processes running.

## Motivation

heaptrack reports 1.38 GB peak heap for rust_verify, but `top` shows the
process at ~5.7 GB RSS. The 4.3 GB gap is caused by glibc malloc
fragmentation: rust_verify makes 5M+ small Arc allocations (16-400 bytes)
intermixed with 12M+ temporary Vec growth allocations, leaving glibc's
per-thread arenas full of freed-but-resident pages. glibc does not return
these pages to the OS.

## Method

Five runs of the same verification (1 warmup + 4 measured), differing only
in allocator configuration. A warmup run primes filesystem caches so all
tests start equally warm.

```bash
cd ~/projects/APAS-VERUS
~/projects/verus-heaptrack/scripts/allocator-compare \
    --crate-type=lib src/lib.rs --multiple-errors 20 --expand-errors --num-threads 8
```

All runs: 4282 verified, 0 errors.

## Results

| # | Allocator | Peak RSS (KB) | Peak RSS (GB) | RSS delta | Wall time | Time delta |
|---|-----------|---------------|---------------|-----------|-----------|------------|
| W | warmup (glibc) | 6,023,044 | 5.74 | — | 1:00.67 | — |
| 1 | glibc malloc (default) | 6,020,900 | 5.74 | — | 1:00.02 | — |
| 2 | glibc malloc (MALLOC_ARENA_MAX=2) | 5,968,284 | 5.69 | -52 MB (-0.9%) | 1:08.21 | +8.2s (+14%) |
| 3 | jemalloc (default arenas) | 5,397,196 | 5.15 | -624 MB (-10.3%) | 0:52.47 | -7.6s (-13%) |
| 4 | jemalloc (narenas:2) | 5,306,864 | 5.06 | -714 MB (-11.9%) | 0:52.90 | -7.1s (-12%) |

## Analysis

### jemalloc saves 624-714 MB and is 13% faster

jemalloc with default arenas reduces peak RSS by 624 MB (10.3%) and wall
time by 7.6 seconds (13%). With `narenas:2`, the savings increase to
714 MB (11.9%) with no additional time penalty.

jemalloc wins on both axes because its size-class segregation handles the
millions of small Arc allocations more efficiently than glibc:

- **Less fragmentation**: freed objects return to size-class slabs, not
  general-purpose free lists. Pages can be recycled sooner.
- **Better cache locality**: the allocator's own data structures are
  smaller and more cache-friendly, reducing overhead on the hot allocation
  path.

### jemalloc narenas:2 saves an extra 90 MB over default jemalloc

Limiting jemalloc to 2 arenas (vs the default, typically 4x CPU count)
saves an additional 90 MB without hurting wall time. With only 2 arenas,
there is less per-arena overhead and less memory held in per-arena caches.
jemalloc's lock-free fast path means the reduced arena count does not
cause contention — unlike glibc.

### glibc MALLOC_ARENA_MAX=2 is counterproductive

Limiting glibc to 2 arenas saves only 52 MB of RSS but adds 8.2 seconds
(14%) to wall time. glibc's arena locking does not have a fast path — fewer
arenas means more lock contention. The minimal memory savings are not worth
the performance cost.

### The fragmentation problem quantified

glibc malloc: 1.38 GB live heap in 5.74 GB resident pages = **24% utilization**.
jemalloc narenas:2: 1.38 GB live heap in 5.06 GB resident pages = **27% utilization**.

Even jemalloc cannot fully close the gap. The remaining 3.7 GB of RSS
overhead comes from:

- Allocator metadata and page-level fragmentation (unavoidable with
  millions of small objects)
- mmap'd shared libraries (librustc_driver alone is 146 MB on disk)
- Thread stacks (8 workers + main)
- Kernel page table overhead

Reducing the number of small allocations (Plan 2: path interning eliminates
5M allocations) would improve utilization under either allocator.

## Comparison with code-level optimizations

| # | Approach | RSS saved | Wall time | Code changes |
|---|----------|-----------|-----------|-------------|
| 1 | jemalloc narenas:2 | 714 MB | -7.1s (-12%) | 0 lines |
| 2 | Plan 2: intern paths | ~176 MB heap + fragmentation | neutral | ~20 lines |
| 3 | Plan 3: pre-compute SST | ~150 MB heap + fragmentation | neutral | ~110 lines |
| 4 | Plans 2+3 | ~305 MB heap + fragmentation | neutral | ~130 lines |
| 5 | jemalloc + Plans 2+3 | ~1 GB (estimated) | -12% | ~130 lines |

Path interning (Plan 2) would have an outsized effect on RSS because
eliminating 5M small allocations reduces fragmentation, not just live heap.
The RSS savings could be 2-3x the raw 176 MB heap savings. Combined with
jemalloc narenas:2, the total reduction could approach 1 GB.

## Recommendations

### Immediate (zero code changes)

Add `LD_PRELOAD=/usr/lib/x86_64-linux-gnu/libjemalloc.so.2` to the verus
invocation. Optionally add `MALLOC_CONF=narenas:2`. This saves 714 MB of
RSS and 7 seconds of wall time immediately. Could be documented or added
as a default in the verus launcher script.

### For a Verus PR

Link jemalloc as the global allocator. Two lines in `rust_verify/src/main.rs`:

```rust
#[global_allocator]
static GLOBAL: tikv_jemallocator::Jemalloc = tikv_jemallocator::Jemalloc;
```

Plus a `Cargo.toml` dependency on `tikv-jemallocator`. This benefits all
users without requiring LD_PRELOAD. The crate is widely used in the Rust
ecosystem (TiKV, Servo, ripgrep, Deno, etc.) and is well-maintained.

The `narenas:2` tuning could be set via `_rjem_malloc_conf` (a link-time
symbol) or left to users via `MALLOC_CONF`.

## Raw Data

- Clean run: `results/allocator-compare-20260318-095817.txt`
- Earlier noisy runs (for RSS reproducibility):
  `results/allocator-compare-20260318-085859.txt`,
  `results/allocator-compare-20260318-094610.txt`

## TODO

- [ ] Measure jemalloc + path interning together
- [ ] Test `mimalloc` as a third allocator option
- [ ] Verify jemalloc results on a different machine
