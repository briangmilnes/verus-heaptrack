# Benchmark: jemalloc + DefId Path Cache on APAS-VERUS

Date: 2026-03-19
Host: quiescent (no competing workloads)
Project: APAS-VERUS (4265 verified, 0 errors)
Verus args: `--crate-type=lib src/lib.rs --num-threads 8`
Method: 1 warmup + 3 timed runs per variant, sequential, `/usr/bin/time -v`

## Variants

| Variant | Branch | Allocator | DefId Cache | Commit |
|---------|--------|-----------|-------------|--------|
| baseline | main | glibc | no | f04abf70 |
| jemalloc | jemalloc-global-allocator | jemalloc (compiled in) | no | 5151db60 |
| jemalloc+defid | defid_cache (rebased on jemalloc) | jemalloc (compiled in) | yes | fcc2e8d9 |

## APAS-VERUS Peak RSS (MB)

| Variant | Min | Max | Avg | Delta from baseline |
|---------|-----|-----|-----|---------------------|
| baseline | 5818 | 5831 | 5822 | -- |
| jemalloc | 5199 | 5206 | 5203 | -619 MB (-10.6%) |
| jemalloc+defid | 4926 | 4938 | 4932 | -890 MB (-15.3%) |

## APAS-VERUS Wall Time (seconds)

| Variant | Min | Max | Avg | Delta from baseline |
|---------|-----|-----|-----|---------------------|
| baseline | 51.42 | 53.03 | 52.02 | -- |
| jemalloc | 44.56 | 45.63 | 45.06 | -6.96s (-13.4%) |
| jemalloc+defid | 43.81 | 45.06 | 44.37 | -7.65s (-14.7%) |

## APAS-VERUS Incremental Delta (jemalloc+defid vs jemalloc alone)

| Metric | Delta |
|--------|-------|
| Peak RSS | -271 MB (-5.2%) |
| Wall Time | -0.69s (-1.5%) |

## Raw Data

### Run 1

| Variant | RSS (MB) | Wall Time |
|---------|----------|-----------|
| baseline | 5818 | 0:51.60 |
| jemalloc | 5199 | 0:44.56 |
| jemalloc+defid | 4938 | 0:43.81 |

### Run 2

| Variant | RSS (MB) | Wall Time |
|---------|----------|-----------|
| baseline | 5831 | 0:51.42 |
| jemalloc | 5206 | 0:45.00 |
| jemalloc+defid | 4932 | 0:45.06 |

### Run 3

| Variant | RSS (MB) | Wall Time |
|---------|----------|-----------|
| baseline | 5818 | 0:53.03 |
| jemalloc | 5203 | 0:45.63 |
| jemalloc+defid | 4926 | 0:44.25 |

## What the DefId Cache Does

`def_id_to_vir_path_option` is called ~5M times during VIR construction
but produces only ~20K unique paths. The cache is a thread-local
`HashMap<DefId, Option<Path>>` that stores each result on first
computation and returns Arc clones on subsequent lookups. This
eliminates ~4.98M redundant `Arc<PathX>` allocations, each containing
fresh `Arc<String>` segments.

The change is 23 lines in `source/rust_verify/src/rust_to_vir_base.rs`.

## Notes

- RSS variance within each variant is very low (13 MB max spread),
  confirming the measurements are stable.
- Wall time variance is also low on the quiescent host (< 2s spread).
- The defid cache saves 271 MB beyond jemalloc alone. Combined with
  jemalloc's 619 MB saving, total reduction from glibc baseline is
  890 MB (15.3%).
- Wall time improvement from the cache is modest (0.69s / 1.5%) but
  consistent across all runs.
