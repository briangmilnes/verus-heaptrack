# Eager Drop of SST and Pruned VIR — No Measurable Effect

Date: 2026-03-19
Host: quiescent
Method: 1 warmup + 3 timed runs per variant, sequential, `/usr/bin/time -v`
Branch: `eager-drop-sst` (off `defid_cache`, which is off `jemalloc-global-allocator`)

## Hypothesis

The per-bucket `pruned_krate` (pruned VIR) and `krate_sst` (SST) are held
alive through Z3 solving even though their data has been fully consumed.
Dropping them eagerly — immediately after `OpGenerator::new` clones what it
needs — should reduce peak RSS.

## Changes (10 lines in `verifier.rs`)

1. Changed `verify_bucket` to take `KrateSst` by value (move) instead of `&KrateSst`.
2. Extracted `reveal_group_names: HashSet<Fun>` before `OpGenerator::new`
   (the only post-construction use of `krate`).
3. `drop(krate)` after `OpGenerator::new`.
4. `drop(pruned_krate)` after `ast_to_sst_krate` in `verify_bucket_outer`.

## Results — jemalloc+defid+eager-drop vs jemalloc+defid

Compared against previous quiescent benchmark of `defid_cache` (jemalloc+defid,
no eager-drop). RSS differences are within measurement noise.

| Project | defid RSS (MB) | eager-drop RSS (MB) | Δ (MB) |
|---------|---------------|---------------------|-------:|
| human-eval | 567 | 563 | -4 |
| ironkv | 704 | 702 | -2 |
| node-replication | 839 | 837 | -2 |
| vest | 564 | 562 | -2 |
| memory-allocator | 490 | 489 | -1 |
| apas-verus | 4934 | 4925 | -9 |

Wall time differences are also within noise (cross-run comparison unreliable).

## Full Benchmark — jemalloc vs jemalloc+defid+eager-drop

This single-run comparison confirms the combined defid+eager-drop effect
matches defid alone (i.e., eager-drop adds nothing).

| Project | jemalloc Avg RSS (MB) | eager-drop Avg RSS (MB) | Δ RSS (MB) | Δ RSS (%) |
|---------|----------------------|------------------------|----------:|----------:|
| human-eval | 575 | 563 | -12 | -2.1% |
| ironkv | 718 | 702 | -16 | -2.2% |
| node-replication | 861 | 837 | -24 | -2.8% |
| vest | 572 | 562 | -10 | -1.7% |
| memory-allocator* | 533 | 489 | -44 | -8.3% |
| apas-verus | 5198 | 4925 | -273 | -5.3% |

These match the previous defid_cache-only results within noise, confirming
the eager-drop adds no measurable savings.

## Why It Didn't Work

The per-bucket `pruned_krate` and `krate_sst` are small relative to total
RSS. The large allocations are:

1. **Full unpruned VIR `krate`** — lives across all buckets in
   `verify_crate_inner`. Not addressed by this change.
2. **rustc internal structures** (TyCtxt, MIR, metadata) — permanent,
   not under verus's control.
3. **OpGenerator's cloned data** — `func_map` and `trait_impl_map` are
   cloned from SST during construction. Same lifetime either way.
4. **AIR commands and Z3 solver state** — accumulate during verification.

The pruned VIR and SST for a single module are a small fraction of the
program's total data. Dropping them a few milliseconds earlier frees
almost nothing.

## Conclusion

Not worth shipping. The branch `eager-drop-sst` is preserved for the
record but should not be merged.
