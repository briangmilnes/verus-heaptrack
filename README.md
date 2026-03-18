# verus-heaptrack

Memory profiling and allocator benchmarking for [Verus](https://github.com/verus-lang/verus) (`rust_verify`).

Switching from glibc malloc to jemalloc saves **12% peak RSS** and **9% wall time**
on average across 6 open-source Verus projects, with zero code changes.

## Results

- [Allocator comparison](analyses/experiment-allocator-compare.md) — full benchmark analysis
- [Benchmark systems and results](analyses/benchmark-systems-and-results.md) — LOC counts, validation status, invocation details

## Analyses

- [heaptrack findings](analyses/heaptrack-findings.md) — heap profiling of rust_verify
- [Drop VIR Krate early](analyses/experiment-drop-vir-krate-early.md) — no effect due to Arc refcounting with `--num-threads 8`

## Plans

| Plan | Description |
|------|-------------|
| [Intern paths](plans/plan-intern-paths.md) | Deduplicate 5M VIR path allocations to ~20K unique interned symbols |
| [Pre-compute SST](plans/plan-precompute-sst-before-spawning.md) | Build SST before spawning worker threads to reduce per-thread duplication |
| [Drop VIR Krate early](plans/plan-drop-vir-krate-early.md) | Free VIR Krate after SST conversion (tested: no effect) |
