# Plan 3: Pre-compute SST Before Spawning Threads

**Estimated savings:** ~150 MB (VIR expression/statement nodes only)
**Risk:** Medium — touches core verification pipeline (prune, Ctx, thread spawning)
**Complexity:** Moderate refactor of `verify_crate_inner` and `verify_bucket_outer`

## Motivation

The "drop VIR Krate early" patch (Plan 1) was ineffective because Arc
refcounting keeps the Krate alive until the last worker thread finishes.
This plan attacks the same problem from a different angle: move the
Krate-dependent work (prune + SST conversion) out of the threads entirely,
so the Krate can be dropped before threads start.

## Current Architecture

```
Main thread:
  krate = Arc<KrateX>

  for each of 8 threads:
    thread_krate = krate.clone()       // Arc refcount++
    spawn thread:
      loop (pull buckets from work queue):
        pruned = prune(&thread_krate)  // filter functions/datatypes
        ctx = Ctx::new(&pruned)        // build HashMaps
        sst = ast_to_sst(&ctx, &pruned) // convert VIR→SST
        verify_bucket(&sst)            // Z3 queries (the slow part)
```

Threads hold Arc clones of krate for the entire run. `drop(krate)` on the
main thread only decrements the refcount.

## Proposed Architecture

```
Main thread:
  krate = Arc<KrateX>

  for each bucket (sequentially):
    pruned = prune(&krate)
    ctx = Ctx::new(&pruned)
    sst = ast_to_sst(&ctx, &pruned)
    push (bucket_id, sst, ctx) into bounded channel

  drop(krate)  // last reference — actually frees VIR data

  for each of 8 threads:
    spawn thread:
      loop (pull prepared results from channel):
        verify_bucket(&sst, &ctx)      // Z3 queries only
```

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

## The Ctx Problem

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

Each Ctx is self-contained after construction. Currently, at most 8 exist
simultaneously (one per thread). Pre-computing all ~200 buckets means ~200
Ctx objects alive at once, which could **increase** peak memory.

### Mitigation: Bounded Channel

Use a bounded channel (capacity 16-32) instead of pre-computing everything:

1. Main thread prepares buckets one at a time, blocking when the channel is
   full.
2. Worker threads pull prepared results and run Z3.
3. At most `channel_capacity + 8` prepared results exist simultaneously.
4. After the last bucket is prepared, the main thread drops the krate.

This limits the Ctx explosion while still allowing the krate to be dropped
once all buckets are prepared.

## Performance Impact

### Wall time

Prune + SST conversion is sequentialized onto the main thread. Currently
8 threads do this work in parallel. However, Z3 queries dominate total
time (~90%+). For APAS-VERUS (55-58s wall time), the prune+SST phase is
likely 3-5s total across all buckets. Sequentializing it adds at most a
few seconds to wall time — well under 10%.

With the bounded channel variant, worker threads start Z3 work as soon
as the first bucket is prepared, so the prune+SST and Z3 phases overlap.
Net impact on wall time: negligible.

### Memory

- Freed: ~150 MB of VIR expression/statement nodes
- Still held: paths (173 MB), types, datatypes, traits (via SST Arc refs)
- Ctx objects: bounded by channel capacity, not all 200+ at once
- Net reduction: ~100-150 MB (2-2.5% of 5.97 GB peak RSS)

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

### 2. Change `verify_crate_inner` multi-threaded path

The existing code uses `Arc<Mutex<VecDeque>>` as the work queue (line 2173).
Keep that same pattern — just change what's in the queue:

```rust
// Phase 1: Prepare all buckets on main thread
let mut tasks = VecDeque::with_capacity(bucket_ids.len());
for (i, bucket_id) in bucket_ids.iter().enumerate() {
    let reporter = QueuedReporter::new(i, sender.clone());
    let prepared = self.prepare_bucket(&reporter, &krate, bucket_id, ...)?;
    tasks.push_back((i, prepared, reporter));
}
drop(krate); // all prune+SST done; VIR expressions freed

// Phase 2: Workers pull prepared buckets from the same Mutex<VecDeque>
let taskq = Arc::new(Mutex::new(tasks));
for _tid in 0..self.num_threads {
    let mut thread_verifier = self.from_self();
    let thread_taskq = taskq.clone();
    // No thread_krate needed — prepared buckets have everything
    spawn(move || {
        loop {
            let elm = { thread_taskq.lock().unwrap().pop_front() };
            if let Some((_i, prepared, reporter)) = elm {
                thread_verifier.run_bucket(&reporter, prepared, None)?;
                reporter.done();
            } else {
                break;
            }
        }
    });
}
```

### 3. Work queue contents change

Currently each work queue entry is `(i, bucket_id, global_ctx, reporter)`
and threads access `&thread_krate` captured in their closure. The new
entries are `(i, PreparedBucket, reporter)` — self-contained, no krate
reference needed. The `Mutex<VecDeque>` pattern, the `QueuedReporter`,
and the mpsc channel for reporter messages all stay the same.

## Estimated Lines of Code

| Component | Lines | Notes |
|-----------|-------|-------|
| `PreparedBucket` struct | ~15 | New struct holding krate_sst, ctx, bucket_id, timing, reporter |
| `prepare_bucket` fn | ~70 | Extracted from verify_bucket_outer (lines 1944-2015) |
| `run_bucket` fn | ~30 | Extracted from verify_bucket_outer (lines 2017-2036) |
| Multi-threaded path rewrite | ~60 | Replace thread_krate + work-stealing with bounded channel |
| Single-threaded path rewrite | ~30 | Sequential prepare loop + run loop with krate drop between |
| Imports and type aliases | ~5 | PreparedBucket definition |
| **Total new/modified** | **~210** | In `verifier.rs` only |
| Lines removed | ~100 | Current verify_bucket_outer body, thread_krate cloning |
| **Net delta** | **~+110** | |

For comparison:
- Plan 2 (path interning): ~20-30 lines in `rust_to_vir_base.rs`
- Plan 1 (drop krate early): 4 lines in `verifier.rs`

## Comparison With Other Plans

| Plan | Savings | Risk | Effort | Threading-dependent? |
|------|---------|------|--------|---------------------|
| 1: Drop Krate early | 0 MB* | Low | Tiny | Yes — defeated by Arc |
| 2: Intern paths | ~170 MB | Low | Small | No |
| 3: Pre-compute SST | ~150 MB | Medium | Moderate | No (by design) |
| 1+2: Drop + intern | ~170 MB | Low | Small | Partially |
| 2+3: Intern + pre-compute | ~320 MB | Medium | Moderate | No |

*Plan 1 saves ~150 MB with `--num-threads 1`, 0 MB with `--num-threads 8`.

## Recommendation

**Do Plan 2 (path interning) first.** It delivers comparable savings (~170
MB) with a fraction of the code change — a `HashMap<DefId, Arc<PathX>>`
cache in one function. Zero architectural risk. Threading-independent.

Plan 3 is the natural follow-up if more memory reduction is needed after
interning. The two target different memory pools (paths vs VIR expressions)
and their savings are additive. Together they would reclaim ~320 MB of the
1.38 GB peak heap — a 23% reduction.

Plan 3 should not be attempted without Plan 2 in place first. The path
interning reduces the amount of shared data between VIR and SST, which
increases the fraction of memory that Plan 2b can actually free.
