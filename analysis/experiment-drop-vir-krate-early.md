# Experiment: Drop VIR Krate Early

**Date:** 2026-03-17
**Verus commit:** `76e69b81` (verus-lang/verus)
**APAS-VERUS commit:** `e3a5c74d`
**Fork:** `briangmilnes/verus-lang` branch `heaptrack-memory-fix`
**Toolchain:** `1.93.1-x86_64-unknown-linux-gnu`

## Hypothesis

Dropping the VIR `Krate` (`Arc<KrateX>`) after SST conversion would reduce peak
memory by ~300 MB. The Krate contains paths (173M), expressions (129M), and other
AST nodes that are dead weight during the Z3 verification phase.

## Patch

Four insertions in `source/rust_verify/src/verifier.rs`:

1. `self.vir_crate = None` after cloning to local `krate` (line 2047)
2. `drop(pruned_krate)` after `ast_to_sst_krate` returns (line 2004)
3. `drop(krate)` after the thread-spawn `for` loop (line 2223)
4. `drop(krate)` after the single-threaded bucket `for` loop (line 2555)

Applied manually (the patch file had hunk ordering issues). See
`plans/plan-drop-vir-krate-early.md` for detailed rationale.

## Verification

Patched build passes the full `vargo test -p rust_verify_test --release` suite:
0 failures across all 129 test files. Log at
`logs/patching-building-and-testing-verus-76e69b81.log`.

APAS-VERUS verifies identically: 4265 verified, 0 errors.

## Benchmark Results

All runs: `--crate-type=lib src/lib.rs --multiple-errors 20 --expand-errors --num-threads 8`

### /usr/bin/time -v (no heaptrack overhead)

| Metric            | Unpatched       | Patched         | Delta        |
|-------------------|-----------------|-----------------|--------------|
| Verified          | 4265            | 4265            | 0            |
| Errors            | 0               | 0               | 0            |
| Wall time         | 55.48s          | 57.53s          | +2.05s (+4%) |
| Peak RSS          | 5,976,064 KB    | 5,966,916 KB    | -9,148 KB    |
| Peak RSS (GB)     | 5.70 GB         | 5.69 GB         | -0.15%       |

### heaptrack (instrumented)

| Metric                    | Unpatched | Patched | Delta |
|---------------------------|-----------|---------|-------|
| Peak heap consumption     | 1.38G     | 1.38G   | 0     |
| Peak RSS (with overhead)  | 3.93G     | 3.93G   | 0     |
| Total memory leaked       | 1.38G     | 1.38G   | 0     |

## Result: No measurable effect

The patch produced no meaningful change in peak memory. Peak RSS dropped by
9 MB out of 5.97 GB — 0.15%, well within run-to-run noise.

## Why It Failed

The patch was designed around a single-threaded mental model, but APAS-VERUS
runs with `--num-threads 8`. The multi-threaded verification path defeats all
four drop points:

### Arc refcounting preserves the data

The `krate` local in `verify_crate_inner` is an `Arc<KrateX>`. The thread-spawn
loop (lines 2160-2220) does this for each bucket:

```rust
let thread_krate = krate.clone();  // Arc::clone — refcount++, no data copy
```

With 8 threads, the refcount is at least 9 (1 main + 8 threads). When the main
thread executes `drop(krate)`, the refcount decrements from 9 to 8. The
underlying `KrateX` data stays alive because 8 threads still hold references.
The data is only freed when the last thread finishes — which is effectively
process exit.

### self.vir_crate = None is similarly defeated

Setting `self.vir_crate = None` releases the Verifier's `Option<Arc<KrateX>>`
reference. But the local `krate` (created one line earlier via `.clone()`) already
holds its own Arc reference. And each thread clones from that local. The Verifier's
reference was already redundant.

### drop(pruned_krate) in verify_bucket_outer has no effect

Inside `verify_bucket_outer`, `pruned_krate` is created by pruning the full krate.
But the parent thread still holds `thread_krate` (the Arc clone of the full krate).
Dropping the pruned subset doesn't help because:

- The pruned krate shares Arc nodes with the full krate (pruning doesn't deep-copy)
- The full krate's refcount is unchanged by dropping the pruned version
- Other threads' `thread_krate` references keep everything alive anyway

### The single-threaded path would work, but isn't used

With `--num-threads 1`, there are no thread clones — `drop(krate)` after the
bucket loop would actually release the last reference and free ~300 MB. But
APAS-VERUS (and most large Verus projects) uses `--num-threads 8` for
acceptable verification times. Optimizing only the single-threaded path
doesn't help real workloads.

## What Would Actually Work

### For the Krate-lifetime problem

The fundamental issue is that `krate.clone()` is an Arc clone: all threads share
the same underlying data via refcounting. To actually release the Krate early,
the architecture would need to change so that threads receive only what they
need (the already-computed `KrateSst` and bucket metadata), not a full Krate
reference.

Specifically, the SST conversion (`ast_to_sst_krate`) currently happens inside
each thread's `verify_bucket_outer` call. If it were hoisted to the main thread
— converting all buckets to SST before spawning threads — the Krate could be
dropped before any threads start. This is a larger refactor: the main thread
would need to pre-compute all `KrateSst` objects and distribute them to workers.

### For the path duplication problem (Plan 2)

The path interning approach (`plan-intern-paths.md`) remains viable and is
unaffected by threading. `def_path_to_vir_path` creates 4.97M `Arc<PathX>`
allocations with an estimated ~20K unique paths — ~250x duplication consuming
176.6M at peak. A simple `HashMap<DefId, Arc<PathX>>` cache in
`Context::def_path_to_vir_path` would eliminate the duplicates regardless of
how many threads later reference the interned paths. This is the more promising
avenue.

### For the arena problem

The 153 MB in rustc's `DroplessArena` is unfree-able by design. No patch
within Verus can reclaim it.

## Raw Data

All data is in `results/`:

| File | Size | Contents |
|------|------|----------|
| `verus-76e69b81-apas-verus-e3a5c74d-heaptrack.txt` | 966 KB | Full heaptrack_print analysis (unpatched) |
| `verus-76e69b81-apas-verus-e3a5c74d-without-heaptrack.txt` | 56 KB | Verification output + /usr/bin/time (unpatched) |
| `verus-76e69b81-apas-verus-e3a5c74d-heaptrack.rust_verify.2187638.zst` | 311 MB | Compressed heaptrack trace (unpatched) |
| `verus-76e69b81-patched-apas-verus-e3a5c74d-heaptrack.txt` | 966 KB | Full heaptrack_print analysis (patched) |
| `verus-76e69b81-patched-apas-verus-e3a5c74d-without-heaptrack.txt` | 56 KB | Verification output + /usr/bin/time (patched) |
| `verus-76e69b81-patched-apas-verus-e3a5c74d-heaptrack.rust_verify.2309684.zst` | 311 MB | Compressed heaptrack trace (patched) |

## Conclusion

The "drop VIR Krate early" patch is correct (all tests pass, verification
unchanged) but ineffective for multi-threaded verification. Arc refcounting
means the Krate data stays alive until the last worker thread finishes,
regardless of when the main thread drops its reference.

To reduce Verus's memory footprint on large crates, the more promising
approaches are:

1. **Path interning** — cache `def_path_to_vir_path` results to eliminate ~250x
   duplication. Saves ~170 MB regardless of threading. Low risk, localized change.

2. **Pre-compute SST before spawning** — hoist `ast_to_sst_krate` out of per-thread
   work, drop the Krate before threads start. Saves ~300 MB but requires
   architectural refactoring of the verification pipeline.

3. **Both** — the savings are additive. Together they would reclaim ~470 MB
   of the 1.38 GB peak heap.
