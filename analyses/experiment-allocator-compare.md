# Experiment: Allocator Comparison

**Date:** 2026-03-18
**Verus commit:** `76e69b81` (unpatched, via briangmilnes/verus-lang fork)
**APAS-VERUS commit:** `e3a5c74d`
**Machine load:** Quiet — no other significant processes running.

## Motivation

heaptrack reports 1.38 GB peak heap for rust_verify, but `top` shows
~5.7 GB RSS. The 4.3 GB gap is glibc malloc arena overhead on
rust_verify's allocation pattern (5M+ small Arc objects, 12M+ temporary
Vec growths).

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

jemalloc with default arenas reduces peak RSS by 624 MB (10.3%) and wall
time by 7.6 seconds (13%). With `narenas:2`, the savings increase to
714 MB (11.9%) with no additional time penalty. glibc `MALLOC_ARENA_MAX=2`
saves only 52 MB of RSS but adds 8.2 seconds (14%) to wall time due to
lock contention.

## Additional code-level optimizations (on top of jemalloc)

Starting from jemalloc narenas:2 as baseline (5.06 GB RSS, 52.9s wall time):

| # | Approach | Additional RSS saved | Wall time | Code changes |
|---|----------|---------------------|-----------|-------------|
| 1 | Intern paths | ~176 MB heap + fragmentation | neutral | ~20 lines |
| 2 | Pre-compute SST | ~150 MB heap + fragmentation | neutral | ~110 lines |
| 3 | Both | ~305 MB heap + fragmentation | neutral | ~130 lines |

Path interning eliminates 5M small allocations, so the RSS savings would
exceed the 176 MB heap reduction.

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

## Cross-Project Validation

Benchmarked 6 verita fixture projects to confirm the effect is not APAS-VERUS-specific.
Each project ran 5 configs (warmup, glibc, glibc arena=2, jemalloc, jemalloc narenas:2).
Verus `f04abf70` (toolchain 1.94.0, `--features singular`). All runs on a quiet machine.

| # | Project | Verified | glibc MB | jemal MB | RSS Δ | glibc s | jemal s | Time Δ |
|---|---------|----------|----------|----------|-------|---------|---------|--------|
| 1 | human-eval | 292 | 614 | 537 | -13% | 5.4s | 5.1s | -4% |
| 2 | ironkv | 319 | 783 | 706 | -10% | 5.4s | 5.1s | -5% |
| 3 | node-replication | 254 | 957 | 854 | -11% | 6.5s | 5.5s | -14% |
| 4 | vest | 496 | 648 | 565 | -13% | 5.7s | 5.2s | -9% |
| 5 | memory-allocator | 731 | 1,629 | 1,416 | -13% | 38.7s | 35.2s | -9% |
| 6 | APAS-VERUS | 4,265 | 5,822 | 5,143 | -12% | 59.2s | 53.2s | -10% |
| | **Peak** | | | | **-14%** | | | **-14%** |
| | **Average** | | | | **-12%** | | | **-9%** |

The effect is consistent across all project sizes (537 MB to 5.8 GB). jemalloc narenas:2
wins on both RSS and wall time in every case.

Per-project results: `results/benchmark-all-20260318-123432/`

## Raw Data

- Clean run: `results/allocator-compare-20260318-095817.txt`
- Earlier noisy runs (for RSS reproducibility):
  `results/allocator-compare-20260318-085859.txt`,
  `results/allocator-compare-20260318-094610.txt`

## TODO

- [ ] Measure jemalloc + path interning together
- [ ] Test `mimalloc` as a third allocator option
- [ ] Verify jemalloc results on a different machine
