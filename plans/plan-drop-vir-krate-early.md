# Plan 1: Drop VIR Krate After SST Conversion

**Estimated savings:** ~300 MB (paths 173M + expressions 129M held through Z3)
**Risk:** Low — purely lifecycle management, no algorithmic changes
**Complexity:** ~10 lines changed in `verifier.rs`

## Problem

The VIR `Krate` (an `Arc<KrateX>` containing the entire verified IR AST) is
held alive during the Z3 verification phase even though its data was fully
converted to `KrateSst` before Z3 starts. The `verify_bucket()` function takes
`&KrateSst`, never the original `Krate`.

## Changes

All changes are in `rust_verify/src/verifier.rs`.

### Change 1: Clear `self.vir_crate` after cloning to local

After line 2046 where the local `krate` is created:

```rust
let krate = self.vir_crate.clone().expect("vir_crate should be initialized");
self.vir_crate = None;  // Release Verifier's reference; local `krate` has it
```

This prevents `from_self()` (line 554) from cloning vir_crate into each
thread_verifier. Thread workers don't read `self.vir_crate` — they receive
`&krate` as a parameter to `verify_bucket_outer`.

### Change 2: Drop `krate` after thread spawning (multi-threaded path)

After the thread spawn loop (after line ~2200, before the receiver loop):

```rust
drop(krate);  // Each thread has its own Arc clone via thread_krate
```

### Change 3: Drop `krate` after bucket loop (single-threaded path)

After line 2554 (end of the single-threaded bucket loop):

```rust
drop(krate);  // All buckets processed; SST has the data
```

### Change 4: Drop `pruned_krate` after SST conversion in `verify_bucket_outer`

After line 2003 (after `ast_to_sst_krate` returns):

```rust
drop(pruned_krate);  // SST conversion complete; pruned VIR no longer needed
```

## Verification

1. The only read of `self.vir_crate` is line 2046 (the clone to local). After
   that, all code uses the local `krate` or `thread_krate`.

2. `from_self()` clones `self.vir_crate` (line 554) but thread workers never
   read their copy — they receive `&thread_krate` as a parameter.

3. `verify_bucket()` (line 1322) takes `&KrateSst`, not `&Krate`. Z3 queries
   are built from AIR commands derived from SST, never from VIR directly.

4. `pruned_krate` in `verify_bucket_outer` is used only for logging (line 1995)
   and SST conversion (line 1998-2003). After that it's dead.

## What this does NOT fix

- The 153 MB in rustc `DroplessArena` (unfree-able by design)
- Path duplication (4.97M calls creating identical paths)
- Per-node Arc overhead throughout VIR/AIR

## Patch

See `patches/drop-vir-krate-early.patch`.
