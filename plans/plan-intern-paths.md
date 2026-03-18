# Plan 2: Intern VIR Path Symbols

**Estimated savings:** ~167 MB (96% of path memory)
**Risk:** Medium — changes a core type's allocation pattern; may affect
  downstream Arc semantics, serialization, equality assumptions
**Complexity:** ~50-100 lines across `rust_to_vir_base.rs` and possibly
  `vir/src/ast.rs`

## Problem

`def_path_to_vir_path` creates a fresh `Arc<PathX>` with fresh `Arc<String>`
segments on every call — 4.97 million times. No caching. The same DefId is
converted to the same Path repeatedly. Estimated ~20,000 unique paths vs
4.97M allocations = ~250x duplication.

Each Path allocation includes:
- `Arc<PathX>` (16 bytes Arc overhead + 16 bytes PathX)
- `Arc<Vec<Arc<String>>>` for segments
- `Arc<String>` per segment (~50-60 bytes each)
- Total: ~200-400 bytes per path depending on segment count

## Proposed approach: DefId-keyed cache

The simplest fix caches at the `def_id_to_vir_path_option` level, keyed
by `DefId` (which is `Copy` and cheap to hash):

```rust
// In rust_to_vir_base.rs, add a thread-local or function-local cache:
use std::cell::RefCell;

thread_local! {
    static PATH_CACHE: RefCell<HashMap<DefId, Path>> = RefCell::new(HashMap::new());
}

pub(crate) fn def_id_to_vir_path_option<'tcx>(
    tcx: TyCtxt<'tcx>,
    verus_items: Option<&crate::verus_items::VerusItems>,
    def_id: DefId,
) -> Option<Path> {
    // Check cache first.
    if let Some(cached) = PATH_CACHE.with(|c| c.borrow().get(&def_id).cloned()) {
        return Some(cached);
    }

    // Existing logic...
    let path = /* existing code */;

    // Cache the result.
    if let Some(ref p) = path {
        PATH_CACHE.with(|c| c.borrow_mut().insert(def_id, p.clone()));
    }
    path
}
```

## Concerns (why the user is doubtful)

### 1. Cache invalidation

VIR paths are derived from rustc DefPaths, which are stable within a single
compilation. The cache would be valid for the entire `crate_to_vir` phase.
But if rust_verify is ever used incrementally (multiple crates in one process),
the cache would need clearing between crates. Currently verus runs one crate
per process, so this is not an issue today.

### 2. Arc identity semantics

Some code might rely on `Arc::ptr_eq` for identity (distinct Arc pointers for
distinct logical instances). With interning, all references to the same path
share the same Arc, which could cause aliasing issues if anyone mutates through
`Arc::make_mut`. However, VIR paths appear to be immutable after construction.

Note: `typ_path_and_ident_to_vir_path` (line 74) calls `Arc::make_mut` on the
segments. This creates a new Arc if refcount > 1, so interning would cause
copy-on-write there instead of in-place mutation. This is functionally correct
but could change performance characteristics.

### 3. Ident interning (deeper change)

The path segments are `Ident = Arc<String>`. Full interning would also
deduplicate these, but that's a broader change affecting all Ident usage
across VIR, not just paths. The cache approach above gets most of the benefit
without touching the Ident type.

### 4. Thread safety

`def_path_to_vir_path` is called during VIR construction which appears to be
single-threaded (before the multi-threaded verification phase). A thread-local
cache is safe. If construction ever becomes multi-threaded, a `DashMap` or
mutex-protected cache would be needed.

### 5. Memory accounting

The cache itself would hold ~20K entries of `(DefId, Arc<PathX>)` = ~640 KB.
This is negligible compared to the 167 MB saved.

## Alternative: String interner for Idents

A more invasive but more thorough approach replaces `Ident = Arc<String>` with
an interned symbol type (like rustc's `Symbol`). This would:

- Deduplicate all string allocations across all VIR nodes
- Make equality checks O(1) (integer comparison)
- Make hashing O(1) (hash the index)
- Benefit all 8+ HashMaps that key on Path

But it changes a fundamental type used throughout VIR and AIR, affecting
serialization, printing, and every place that constructs or pattern-matches
on Idents. This is a major refactor.

## Recommendation

Start with the DefId cache (10 lines, low risk, gets ~90% of the savings).
Defer Ident interning to a separate effort if the cache proves insufficient.

## No patch provided

This plan is more speculative. A patch would need validation against the full
verus test suite, and the cache approach needs sign-off from verus maintainers
on thread-safety and cache-lifetime assumptions.
