
# Heaptrack Analysis: Verus rust_verify

**Date:** 2026-03-17
**Verus commit:** `76e69b81` (verus-lang/verus)
**Target crate:** APAS-VERUS at `e3a5c74d` (4265 verified functions, ~60 chapters)
**Trace files:** See `results/` directory for all traces and benchmarks

## Summary

```
peak heap memory consumption: 1.36G
peak RSS (including heaptrack overhead): 3.88G
total memory leaked: 1.36G
suppressed leaks: 94.74K
```

Virtually nothing is freed before process exit. The entire VIR AST is built
as `Arc<T>` nodes, held alive for the full verification run, and dropped only
at exit.

## Binary Architecture

The `verus` binary (4.1 MB) is a thin launcher that exec's:

    rustup run 1.93.1-x86_64-unknown-linux-gnu -- rust_verify <args>

`rust_verify` (19.7 MB) is the real compiler plugin. It dynamically loads
`librustc_driver` (146 MB) at startup. Both use glibc malloc (no jemalloc).

heaptrack's LD_PRELOAD dies at the exec boundary, which is why naive
`heaptrack verus ...` produces only ~70 KB of trace. Must invoke
`heaptrack rust_verify ...` directly.

## Allocation Architecture

Two allocation layers in the process:

1. **rustc layer** (HIR, type checking): `DroplessArena` — 115 large chunks
   via posix_memalign, bump-allocated internally. 153 MB at peak.

2. **VIR/AIR layer** (rust_verify's verification code): Everything is `Arc<T>`.
   Every AST node — `Expr`, `Stmt`, `Typ`, `Ident`, `Path`, `Function` — is
   individually heap-allocated via standard malloc. This is the bulk of memory.

## Peak Memory Consumers

| # | Peak | Calls | Source | What |
|---|------|-------|--------|------|
| 1 | 173.6M | 4,970,000 | `def_path_to_vir_path` | `Arc<PathX>` for every DefPath->VIR conversion |
| 2 | 153.1M | 115 | `DroplessArena::grow` | rustc arena chunks (by design, never freed) |
| 3 | 129.4M | 1,331,000 | `SpannedTyped<ExprX>::new` | Every VIR expression node |
| 4 | various | various | VIR visitor rewrites | `demote_external_traits`, `cleanup_span_ids`, etc. |

## Temporary Allocations

79.8% of all allocations are temporary (allocated and freed during the run).
The top temporary allocator is `RawVecInner::finish_grow` in
`libverus_builtin_macros.so` — the proc macro expansion phase. These are
`Vec` growth allocations during `verus_syn::rejoin_tokens`.

## Root Causes

### 1. VIR Krate held too long

The VIR `Krate` object (containing all paths, expressions, types, statements)
is held alive during the entire Z3 verification phase, even though:

- All data has been converted to SST (`KrateSst`) before Z3 starts
- Z3's `verify_bucket()` takes `&KrateSst`, never the original `Krate`
- The `Krate` is dead weight during the slowest phase

Held alive by:
- `self.vir_crate: Option<Krate>` in the `Verifier` struct (never set to None)
- `krate` local in `verify_crate_inner` (lives until function exit)
- `thread_krate` clones per worker thread (lives until thread exit)
- `thread_verifier.vir_crate` clones via `from_self()` (lives until thread exit)

### 2. No path interning

`def_path_to_vir_path` creates a fresh `Arc<PathX>` with fresh `Arc<String>`
segments on every call — 4.97 million times. No caching or deduplication.
The same DefId is converted to the same Path repeatedly throughout VIR
translation. Estimated ~20,000 unique paths vs 4.97M allocations = ~250x
duplication.

### 3. Arc<T> for immutable AST nodes

VIR uses `Arc<T>` for everything (Expr, Stmt, Typ, Path, etc.) even though
the AST is single-threaded during construction and read-only during
verification. The refcount overhead and per-node heap allocation are
unnecessary — a typed arena with indices would eliminate both.

## Verification Pipeline (showing data lifetimes)

```
Phase                  Input          Output          Krate alive?
-----                  -----          ------          ------------
HIR -> VIR             Rust HIR       Krate           created
VIR merge              Krate[]        merged Krate    alive
Validation             Krate          checked Krate   alive
GlobalCtx build        &Krate         extracted maps  alive (data copied out)
Simplify               &Krate         new Krate       alive (replaced)
Bucket extraction      &Krate         Vec<BucketId>   alive
Prune (per bucket)     &Krate         pruned Krate    alive
VIR -> SST             pruned Krate   KrateSst        alive (LAST USE)
Poly expansion         KrateSst       KrateSst        alive but unused
SST -> AIR             KrateSst       Commands        alive but unused
Z3 queries             Commands       results         alive but unused  <-- waste
Results + reporting    -              -               alive until exit
```

## Experiment Results

### Drop VIR Krate early (tested 2026-03-17)

**Result: No measurable effect** with `--num-threads 8`.

The patch adds 4 explicit `drop()` calls to release the Krate after SST conversion.
All Verus tests pass and verification output is identical. But peak memory is
unchanged because `krate.clone()` in the thread-spawn loop is an Arc clone — each
of 8 threads holds a reference, keeping the underlying data alive until the last
thread finishes.

See `experiment-drop-vir-krate-early.md` for full benchmark data, the root cause
analysis, and what approaches would actually work.

### Path interning (not yet tested)

The most promising remaining target. `def_path_to_vir_path` creates 4.97M
`Arc<PathX>` allocations for an estimated ~20K unique paths. A
`HashMap<DefId, Arc<PathX>>` cache would save ~170 MB regardless of threading.
See `../plans/plan-intern-paths.md`.
