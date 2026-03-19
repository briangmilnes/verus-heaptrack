# Plan: Eagerly Drop SST and Pruned VIR During Verification

**Goal:** Free `krate_sst` (SST) and `pruned_krate` (pruned VIR) as soon as
their data has been consumed, rather than holding them alive through Z3 solving.

**Risk:** Low — structural refactor, no algorithmic changes.
**Complexity:** ~30 lines changed across 2 files.

## Current Extents (the problem)

```
verify_bucket_outer(krate: &Krate, ...)
│
│  pruned_krate ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┓
│  │  used: prune (1956), log (1995), ast_to_sst (1998) ┃
│  │  dead after line 2003                              ┃
│  │                                                    ┃
│  krate_sst ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┓ ┃
│  │  used: poly (2013), passed to verify_bucket (2016)┃ ┃
│  │                                                   ┃ ┃
│  │  verify_bucket(&krate_sst, ...)                   ┃ ┃
│  │  │  traits_to_air       (1382)  ─ reads krate     ┃ ┃
│  │  │  datatypes_to_air    (1401)  ─ reads krate     ┃ ┃
│  │  │  func_decls_to_air   (1453)  ─ reads krate     ┃ ┃
│  │  │  OpGenerator::new    (1475)  ─ clones into map ┃ ┃
│  │  │  ┄┄┄┄ krate_sst dead here ┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄  ┃ ┃
│  │  │  reveal_groups lookup (1706) ─ reads krate (!) ┃ ┃
│  │  │  ... Z3 solving ...                            ┃ ┃
│  │  └─ return                                        ┃ ┃
│  │                                                   ┃ ┃
│  └─ return                     krate_sst freed ━━━━━━┛ ┃
│                                pruned_krate freed ━━━━━┛
```

`pruned_krate` is dead after line 2003 but freed at ~2034.
`krate_sst` is dead after line 1475 (except for one field access at 1706)
but freed at ~2034. Both sit in memory through all of Z3 solving for the
bucket.

## Proposed Extents

```
verify_bucket_outer(krate: &Krate, ...)
│
│  pruned_krate ━━━━━━━━━━━━━━━━┓
│  │  prune, log, ast_to_sst    ┃
│  │  drop(pruned_krate)  ━━━━━━┛  ← freed here
│  │
│  krate_sst ━━━━━━━━━━━━━━━━━━━┓
│  │  poly, pass to verify_bucket┃
│  │                             ┃
│  │  verify_bucket(krate_sst)   ┃  ← takes ownership, not &borrow
│  │  │  traits_to_air           ┃
│  │  │  datatypes_to_air        ┃
│  │  │  func_decls_to_air       ┃
│  │  │  extract reveal_group_names: HashSet<Fun>
│  │  │  OpGenerator::new        ┃
│  │  │  drop(krate_sst)  ━━━━━━┛  ← freed here
│  │  │  ... Z3 solving (krate_sst gone) ...
│  │  └─ return
│  └─ return
```

## Changes

### Change 1: Drop `pruned_krate` after SST conversion

In `verify_bucket_outer`, after `ast_to_sst_krate` and optional logging
(~line 2012):

```rust
drop(pruned_krate);
```

This is safe because `pruned_krate` is not read after line 2003. The logging
at line 1995 and SST conversion at line 1998 are the last uses.

### Change 2: Change `verify_bucket` to take `KrateSst` by value

Current signature (line 1322):
```rust
fn verify_bucket(
    &mut self,
    reporter: &impl Diagnostics,
    krate: &vir::sst::KrateSst,   // borrowed
    ...
```

New signature:
```rust
fn verify_bucket(
    &mut self,
    reporter: &impl Diagnostics,
    krate: vir::sst::KrateSst,    // owned
    ...
```

`KrateSst` is `Arc<KrateSstX>`, so passing by value is just moving an Arc
(no data copy). The caller in `verify_bucket_outer` (line 2016) changes
from `&krate_sst` to `krate_sst`.

### Change 3: Extract `reveal_groups` before dropping SST

Before `OpGenerator::new` (after all AIR declarations are built, ~line 1470):

```rust
let reveal_group_names: HashSet<Fun> = krate
    .reveal_groups
    .iter()
    .map(|g| g.x.name.clone())
    .collect();
```

Then replace the `krate.reveal_groups` usage at line 1706-1710:

```rust
// Before:
let is_reveal_group = krate
    .reveal_groups
    .iter()
    .find(|g| &g.x.name == funx)
    .is_some();

// After:
let is_reveal_group = reveal_group_names.contains(funx);
```

This is also a performance improvement — O(1) HashSet lookup vs O(n) linear
scan.

### Change 4: Drop `krate` inside `verify_bucket`

After `OpGenerator::new` and the `reveal_group_names` extraction:

```rust
drop(krate);  // SST fully consumed; free before Z3 solving
```

## Why This Is Safe

1. **OpGenerator clones what it needs.** `OpGenerator::new` (commands.rs:83-96)
   iterates `krate.functions` and `krate.trait_impls`, cloning each into its
   own `HashMap`. After construction, it never touches the original `krate`.

2. **Op values are self-contained.** Each `Op` returned by `OpGenerator::next()`
   owns its data via `Arc`. No references back to SST or VIR.

3. **expand_errors uses cloned `FuncCheckSst`.** The `func_check_sst` field in
   `OpKind::Query` is `Option<Arc<FuncCheckSst>>` — a clone, not a reference.

4. **AIR commands are independent of SST.** `sst_to_air` converts VIR types to
   AIR strings. The resulting AIR expressions contain no VIR/SST references.

5. **Spans are already Arc-cloned** into `CommandContext`. They don't reference
   back to VIR structures.

6. **`pruned_krate` is only used for logging and SST conversion.** Both happen
   before `verify_bucket` is called.

## What We Don't Know Yet

**How much memory is in `krate_sst` and `pruned_krate`?**

We know they contain the full SST and pruned VIR for one module (bucket).
For a large module in apas-verus, this could be significant. A heaptrack
profile would quantify the savings. The upper bound is: total RSS minus
rustc's base footprint minus AIR commands minus the full (unpruned) VIR krate.

The full (unpruned) VIR `krate` in `verify_crate_inner` is NOT addressed by
this plan — it lives across all buckets. That is the subject of the existing
plan-drop-vir-krate-early.md.

## Interaction With Other Plans

- **plan-drop-vir-krate-early.md:** Complementary. That plan drops the full
  unpruned VIR `krate` held in `verify_crate_inner`. This plan drops the
  per-bucket `pruned_krate` and `krate_sst`. Both can be done independently.

- **DefId cache:** No interaction. The cache is populated during VIR
  construction (before any of this code runs).

- **jemalloc:** Complementary. jemalloc returns freed pages to the OS more
  aggressively, so eager drops translate more directly to RSS reduction.
