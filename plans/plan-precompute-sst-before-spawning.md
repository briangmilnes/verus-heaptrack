# Plan 3: Pre-compute SST Before Running Z3

**Estimated savings:** ~150 MB (VIR expression/statement nodes only)
**Risk:** Medium — touches core verification pipeline (prune, Ctx, thread spawning)
**Complexity:** Small refactor of `verify_crate_inner` and `verify_bucket_outer`

## Motivation

The "drop VIR Krate early" patch (Plan 1) was ineffective because Arc
refcounting keeps the Krate alive until the last worker thread finishes.
Currently each thread holds an `Arc<KrateX>` for the entire run — including
the Z3 phase that dominates 90%+ of wall time. The krate is only freed
when all threads exit, i.e. when the program is essentially done.

This plan splits each thread's work into two phases so that all Arc
clones are dropped before the expensive Z3 phase.

## Current Architecture

```
Main thread:
  krate = Arc<KrateX>                   // refcount = 1

  for each of 8 threads:
    thread_krate = krate.clone()        // refcount++
    spawn thread:
      loop (pull buckets from work queue):
        pruned = prune(&thread_krate)   // needs krate
        ctx = Ctx::new(&pruned)         // needs krate
        sst = ast_to_sst(&ctx, &pruned) // needs krate
        verify_bucket(&sst)             // Z3 queries — does NOT need krate
      drop(thread_krate)                // refcount-- (only at thread exit)

  drop(krate)                           // refcount-- (main thread)
  // krate freed only when last thread exits — after ALL Z3 work is done
```

## Proposed Architecture: Two-Phase Threads

No channels. No barriers. Each thread simply does its prepare work first,
drops the krate, then does Z3 work:

```
Main thread:
  krate = Arc<KrateX>                   // refcount = 1
  drop(main_krate)                      // main thread doesn't need it after spawn

  for each of 8 threads:
    thread_krate = krate.clone()        // refcount++
    spawn thread:

      // Phase 1: Prepare (needs krate)
      let mut prepared = Vec::new();
      loop (pull buckets from work queue):
        pruned = prune(&thread_krate)
        ctx = Ctx::new(&pruned)
        sst = ast_to_sst(&ctx, &pruned)
        prepared.push((sst, ctx))

      drop(thread_krate)                // refcount--
      // Last thread to finish Phase 1 triggers deallocation:
      //   Arc 8 → 7 → ... → 0, krate freed

      // Phase 2: Z3 (no krate reference held anywhere)
      for (sst, ctx) in prepared:
        verify_bucket(&sst, &ctx)
```

The key insight: Arc already handles deallocation correctly. When the last
thread finishes preparing and drops its clone, Arc hits 0 and the krate is
freed. No channel or synchronization needed — each thread independently
transitions from Phase 1 to Phase 2.

The existing `Mutex<VecDeque>` work queue stays the same. Threads still
pull bucket IDs from the shared queue. The only change is that each thread
accumulates prepared results locally, drops its Arc, then runs Z3 on the
accumulated results.

## What Would Actually Be Freed

`KrateSstX` (sst.rs:391) is built by Arc-cloning datatypes, traits,
trait_impls, etc. from the pruned krate:

```rust
pub struct KrateSstX {
    pub functions: Vec<FunctionSst>,          // NEW — converted from VIR
    pub datatypes: Vec<crate::ast::Datatype>, // Arc-cloned from pruned krate
    pub traits: Vec<crate::ast::Trait>,       // Arc-cloned from pruned krate
    pub trait_impls: Vec<crate::ast::TraitImpl>, // Arc-cloned
    pub assoc_type_impls: Vec<crate::ast::AssocTypeImpl>, // Arc-cloned
    pub reveal_groups: Vec<crate::ast::RevealGroup>,      // Arc-cloned
    ...
}
```

After dropping the krate, the SSTs' Arc references keep datatypes, traits,
paths, and types alive. Only data unique to VIR (not referenced by SST) is
freed:

| Data | Peak size | Freed? | Why |
|------|-----------|--------|-----|
| VIR expression nodes (`SpannedTyped<ExprX>::new`) | 129 MB | Yes | SST uses `sst::Exp`, a different type |
| VIR statement nodes | ~20 MB | Yes | SST uses `sst::Stm`, different type |
| Krate container structs, modules, external_fns | ~10 MB | Yes | Not copied to SST |
| Path nodes (`Arc<PathX>`) | 173 MB | No | Referenced by SST datatypes/traits |
| Datatype/trait/trait_impl data | ~50 MB | No | Arc-cloned into KrateSst |

**Realistic savings: ~150 MB** (the VIR expression/statement layer).
Not the full ~300 MB estimated in Plan 1, because paths and types are shared.

## The Ctx Accumulation Problem

`Ctx::new` (context.rs:748) builds its own HashMaps by cloning functions,
datatypes, and traits from the pruned krate:

```rust
pub struct Ctx {
    pub functions: Vec<Function>,
    pub func_map: HashMap<Fun, Function>,
    pub func_sst_map: HashMap<Fun, FunctionSst>,
    pub datatype_map: HashMap<Dt, Datatype>,
    pub trait_map: HashMap<Path, Trait>,
    // ... ~15 more fields
}
```

Currently at most 8 Ctx objects exist (one per thread). With the two-phase
approach, all ~200 Ctx objects are alive simultaneously during the gap
between Phase 1 and Phase 2. This could **increase** peak memory.

### Mitigation

This is likely acceptable:

1. Each Ctx contains Arc clones, not deep copies. The underlying data is
   shared — 200 Ctx objects don't hold 200x the data, they hold 200 sets
   of HashMap overhead pointing to shared Arc'd data.
2. The HashMap overhead per Ctx is modest (pointers + hash table metadata).
3. The prepared SST results would exist transiently in any design that
   separates prepare from Z3.
4. The ~200 Ctx objects are distributed across 8 threads' `prepared`
   vectors, not one giant allocation.

If this does prove problematic, a simple fallback is to keep the work queue
but split it into two queues (prepare-queue and run-queue) with a cap on
how far ahead preparation can get. But try the simple approach first.

## Fragmentation: Will RSS Actually Drop?

Even when Arc hits 0 and the krate's destructor frees all VIR nodes,
the freed memory may not reduce RSS. The VIR expression nodes are many
small allocations interleaved with still-live objects on the same pages:

```
Page: [VIR expr][SST exp][Arc<PathX>][VIR expr][type node][VIR stmt]
         freed     live      live      freed      live      freed
```

A page with any live object cannot be returned to the OS. The freed slots
become allocator free-list entries — available for reuse but not reducing
RSS.

**Best case:** VIR expression nodes were allocated in a burst during
`ast_to_sst` and happen to occupy mostly-contiguous pages. Freeing them
empties entire pages, and the allocator returns them via `munmap` or
`madvise(MADV_DONTNEED)`. RSS drops by close to 150 MB.

**Worst case:** VIR nodes are scattered across pages shared with live SST
and path data. No pages can be reclaimed. RSS doesn't change, but the
freed slots are reused by Z3's working allocations, potentially preventing
RSS from growing *further*.

**Likely outcome:** Somewhere in between. Some pages reclaimed, some not.
RSS reduction of 50-100 MB rather than the full 150 MB.

## Performance Impact

### Wall time

The two-phase approach does NOT sequentialize prune+SST onto the main
thread. All 8 threads still do their prepare work in parallel — they just
finish all of it before starting Z3. Since threads pull from the same
work queue, fast threads prepare more buckets and slow threads prepare
fewer. Load balancing is preserved.

The only potential slowdown: a thread that finishes Phase 1 early must
wait for its prepared results to all be ready before starting Phase 2.
But since all threads draw from the same queue, they'll finish Phase 1
at roughly the same time. The Z3 phase starts almost immediately after
preparation finishes.

Net impact on wall time: negligible.

### Memory

- Freed: ~150 MB of VIR expression/statement nodes (subject to fragmentation)
- Still held: paths (173 MB), types, datatypes, traits (via SST Arc refs)
- Ctx objects: ~200 at peak (up from 8), but mostly Arc pointer overhead
- Net RSS reduction: ~50-150 MB (1-2.5% of 5.97 GB peak RSS)

## Implementation Sketch

Changes in `rust_verify/src/verifier.rs`:

### 1. Split `verify_bucket_outer` into prepare and run

```rust
// New: prepare phase (needs &krate)
fn prepare_bucket(
    &mut self,
    reporter: &impl Diagnostics,
    krate: &Krate,
    bucket_id: &BucketId,
    global_ctx: GlobalCtx,
) -> Result<PreparedBucket, VirErr> {
    // prune, Ctx::new, ast_to_sst_krate, poly — everything before verify_bucket
}

// New: run phase (needs only SST + Ctx)
fn run_bucket(
    &mut self,
    reporter: &impl Diagnostics,
    prepared: PreparedBucket,
    source_map: Option<&SourceMap>,
) -> Result<GlobalCtx, VirErr> {
    // verify_bucket + timing + stats
}
```

### 2. Change each thread's work loop

```rust
for _tid in 0..self.num_threads {
    let mut thread_verifier = self.from_self();
    let thread_krate = krate.clone();
    let thread_taskq = taskq.clone();
    spawn(move || {
        // Phase 1: Prepare all assigned buckets
        let mut prepared_work = Vec::new();
        loop {
            let elm = { thread_taskq.lock().unwrap().pop_front() };
            if let Some((i, bucket_id, global_ctx, reporter)) = elm {
                let prepared = thread_verifier.prepare_bucket(
                    &reporter, &thread_krate, &bucket_id, global_ctx,
                )?;
                prepared_work.push((i, prepared, reporter));
            } else {
                break;
            }
        }

        // Drop krate reference — last thread to reach here frees VIR data
        drop(thread_krate);

        // Phase 2: Run Z3 on prepared results
        for (i, prepared, reporter) in prepared_work {
            thread_verifier.run_bucket(&reporter, prepared, None)?;
            reporter.done();
        }
        Ok(())
    });
}
```

### 3. Work queue stays the same

The existing `Arc<Mutex<VecDeque>>` work queue is unchanged. Each entry
is still `(i, bucket_id, global_ctx, reporter)`. Threads still pull
entries one at a time until the queue is empty. The only difference is
what threads do with each entry: prepare and accumulate rather than
prepare-and-immediately-run.

## Estimated Lines of Code

| Component | Lines | Notes |
|-----------|-------|-------|
| `PreparedBucket` struct | ~15 | New struct holding krate_sst, ctx, bucket_id, timing |
| `prepare_bucket` fn | ~70 | Extracted from verify_bucket_outer (lines 1944-2015) |
| `run_bucket` fn | ~30 | Extracted from verify_bucket_outer (lines 2017-2036) |
| Thread loop restructure | ~30 | Two-phase loop replacing single-phase loop |
| Single-threaded path update | ~20 | Same two-phase pattern, one thread |
| **Total new/modified** | **~165** | In `verifier.rs` only |
| Lines removed | ~100 | Current verify_bucket_outer body |
| **Net delta** | **~+65** | |

For comparison:
- Plan 2 (path interning): ~20-30 lines in `rust_to_vir_base.rs`
- Plan 1 (drop krate early): 4 lines in `verifier.rs`

## Comparison With Other Plans

| Plan | Savings | Risk | Effort | Threading-dependent? |
|------|---------|------|--------|---------------------|
| 1: Drop Krate early | 0 MB* | Low | Tiny | Yes — defeated by Arc |
| 2: Intern paths | ~170 MB | Low | Small | No |
| 3: Pre-compute SST | ~50-150 MB | Medium | Small-Moderate | No (by design) |
| 1+2: Drop + intern | ~170 MB | Low | Small | Partially |
| 2+3: Intern + pre-compute | ~220-320 MB | Medium | Moderate | No |

*Plan 1 saves ~150 MB with `--num-threads 1`, 0 MB with `--num-threads 8`.

Plan 3 savings shown as a range (50-150 MB) to reflect fragmentation
uncertainty. Actual RSS reduction depends on allocation layout.

## Recommendation

**Do Plan 2 (path interning) first.** It delivers comparable savings (~170
MB) with a fraction of the code change — a `HashMap<DefId, Arc<PathX>>`
cache in one function. Zero architectural risk. Threading-independent.
No fragmentation concerns (it prevents allocations rather than freeing
them after the fact).

Plan 3 is the natural follow-up if more memory reduction is needed after
interning. The two target different memory pools (paths vs VIR expressions)
and their savings are additive. Together they would reclaim ~220-320 MB
of the 1.38 GB peak heap — a 16-23% reduction.

Plan 3 should not be attempted without Plan 2 in place first. The path
interning reduces the amount of shared data between VIR and SST, which
increases the fraction of memory that Plan 3 can actually free.
