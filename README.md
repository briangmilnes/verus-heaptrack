# verus-heaptrack

Memory profiling and allocator benchmarking for [Verus](https://github.com/verus-lang/verus) (`rust_verify`).

## What worked

**jemalloc** (2 lines changed) — compile jemalloc as global allocator:

- avg **-10.0% RSS**, avg **-11.6% wall time** vs glibc malloc

**jemalloc + DefId path cache** (25 lines changed) — cache `def_id_to_vir_path_option` results in a thread-local HashMap:

- avg **-13.9% RSS**, avg **-12.8% wall time** vs glibc malloc
- avg **-3.7% RSS**, avg **-1.4% wall time** incremental over jemalloc alone

All numbers are averages across 6 open-source Verus projects (human-eval,
ironkv, node-replication, vest, memory-allocator, apas-verus), 3 timed runs
per variant on a quiescent host.

macOS is not yet benchmarked. jemalloc is not available on Windows.

## What didn't work

**Eager drop of SST and pruned VIR** (10 lines changed) — drop per-bucket
`krate_sst` and `pruned_krate` immediately after `OpGenerator` consumes them,
rather than holding them through Z3 solving:

- **No measurable effect.** Per-bucket data is tiny relative to total RSS.

**Pre-compute SST before spawning threads** (plan only, not implemented) —
estimated 1-3% savings for 165 lines of medium-risk code. Not worth it
given diminishing returns after jemalloc + defid cache.

**heaptrack profiling** confirmed no large single allocation target remains.
~445 MB of RSS is rustc-internal memory (DroplessArena, mmap) invisible to
malloc profiling. The tracked heap (510 MB on node-replication) is spread
across 251K+ call sites with no dominant hotspot.

## Results

- [glibc vs jemalloc vs jemalloc+defid](analyses/glibc-vs-jemalloc-vs-jemalloc-plus-defid-varia-systems.md) — full 3-way benchmark with per-project tables
- [Benchmark systems and results](analyses/benchmark-systems-and-results.md) — LOC counts, validation status, invocation details
- [Allocator comparison](analyses/experiment-allocator-compare.md) — jemalloc-only benchmark analysis

## Analyses

- [Eager drop SST/VIR](analyses/eager-drop-sst-and-pruned-vir.md) — no measurable effect (negative result)
- [heaptrack node-replication](analyses/heaptrack-node-replication-glibc.md) — heap profiling, no large target remains
- [heaptrack findings](analyses/heaptrack-findings.md) — earlier heap profiling of rust_verify
- [Drop VIR Krate early](analyses/experiment-drop-vir-krate-early.md) — no effect due to Arc refcounting with `--num-threads 8`

## Plans

| Plan | Status | Description |
|------|--------|-------------|
| [DefId path cache](plans/plan-intern-paths.md) | **Shipped** | Cache `def_id_to_vir_path_option` lookups (23 lines, -3.7% RSS) |
| [Eager drop SST/VIR](plans/plan-eager-drop-sst-and-pruned-vir.md) | **Failed** | Drop per-bucket SST/VIR before Z3 solving (no measurable effect) |
| [Drop VIR Krate early](plans/plan-drop-vir-krate-early.md) | **Failed** | Free VIR Krate after SST conversion (no effect with Arc refcounting) |
| [Pre-compute SST](plans/plan-precompute-sst-before-spawning.md) | **Not worth it** | Build SST before spawning threads (~1% estimated, too much risk) |
| [Intern paths](plans/plan-intern-paths.md) | **Open** | Replace `Ident = Arc<String>` with interned `Ident(u32)` (~5% estimated, ~30 files) |
