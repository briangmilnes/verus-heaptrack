# Heaptrack Profile: node-replication (glibc baseline)

Date: 2026-03-19
Binary: verus-lang-baseline (main, glibc allocator)
Project: node-replication (254 verified, 0 errors)
Tool: heaptrack + heaptrack_print, symbols demangled with rustfilt

## Summary

| Metric | Value |
|--------|-------|
| Total RSS (including heaptrack overhead) | 1.59 GB |
| Peak heap consumption (malloc-tracked) | 510 MB |
| Untracked memory (arenas, mmap, stack) | ~445 MB |
| Total allocation calls | 25.5M |
| Temporary allocations | 3.5M |

heaptrack only tracks malloc/free. Roughly **half the RSS is invisible** to
it — rustc's `DroplessArena`, mmap'd metadata, and other non-malloc memory.

## Top Heap Consumers

| Peak (MB) | Calls | Source |
|-----------|-------|--------|
| 29.4 | 450K | `Verifier::from_self` cloning `Vec<(Span, (Mode, Mode))>` for thread spawning |
| 14.6 | 2.5M | `import_crates` → bincode deserializing `Arc<String>`, `Arc<PathX>`, `Arc<FunX>` from VSTD |
| 1.1 | 18K | `GlobalCtx::from_self_with_log` cloning `Vec<Span>` |
| 0.8 | 28K | `prune_krate_for_module_or_krate` cloning `Vec<(PathX, String)>` |
| ~350 | ~22M | Long tail: "1376371 from 251503 other places" — thousands of small sites |

The top 4 identified sites account for ~46 MB. The remaining ~464 MB of
tracked heap is spread across 251K+ call sites, none individually large.

## Note on heaptrack + jemalloc

heaptrack intercepts libc malloc/free via LD_PRELOAD. When the binary uses
jemalloc as the global allocator (compiled in), heaptrack sees zero
allocations. Must use the glibc baseline binary for profiling.

## What This Tells Us

1. **No single large target remains.** After jemalloc (-10% RSS) and defid
   cache (-4% RSS), the remaining heap is dominated by a long tail of small
   allocations. There is no 100 MB function waiting to be optimized.

2. **`from_self` cloning is the largest single site** (29 MB) but it's 3%
   of total RSS. It deep-clones the Verifier struct for each worker thread.
   Reducing this (e.g., sharing immutable data via Arc instead of cloning)
   would save ~29 MB — measurable but not transformative.

3. **VSTD deserialization** allocates 2.5M objects (14.6 MB peak) via
   bincode. String interning at deserialization time would deduplicate these,
   but the peak is only 14.6 MB — not the big win we hoped for.

4. **~445 MB is not heap at all.** It's rustc's DroplessArena, mmap'd
   files, thread stacks, and other memory that malloc never touches. We
   cannot reduce this from verus code.

## Conclusion

The heaptrack profile confirms that jemalloc + defid_cache captured the
accessible wins. The remaining memory is either:
- In a long tail of thousands of small allocation sites (no single target)
- In rustc internals we cannot control (~445 MB)
- In `from_self` cloning (~29 MB, 3% of RSS)

String interning (Plan 2) would help with the long tail by reducing
per-string overhead, but the heaptrack data suggests the total savings
would be modest (~5% at best) for a ~30-file refactor.

Further RSS reduction requires either architectural changes to rustc
(not practical) or a fundamentally different verification architecture
(not practical either).
