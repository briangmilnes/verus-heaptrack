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

## Alternative: String Interner for Idents

### What string interning is

String interning stores one canonical copy of each unique string in a
global table, replacing all duplicates with a cheap handle (typically a
`u32` index). Two interned strings can be compared for equality or hashed
by comparing/hashing their indices — O(1) instead of O(n) in string
length.

This is how rustc handles identifiers internally. Its `Symbol` type is a
`u32` index into a global `Interner` table. Every identifier in the AST
is a `Symbol`, not an `Arc<String>`. Equality is `==` on two `u32` values.

### Current state in verus

VIR defines (`vir/src/ast.rs:21`):

```rust
pub type Ident = Arc<String>;
pub type Idents = Arc<Vec<Ident>>;
```

Every identifier in VIR is a separately heap-allocated `Arc<String>`.
Construction happens in ~109 call sites across VIR, primarily via:

```rust
// air/src/ast_util.rs:58
pub fn str_ident(x: &str) -> Ident {
    Arc::new(x.to_string())
}

// And direct construction throughout def.rs, ast_to_sst.rs, etc.:
Arc::new("some_string".to_string())
```

No deduplication occurs. The string `"bool"` may exist as hundreds of
separate heap allocations across the VIR tree.

### How interning would work for verus

Replace the `Ident` type alias with a newtype wrapping a `u32` index:

```rust
// vir/src/ast.rs — replace the type alias
#[derive(Copy, Clone, PartialEq, Eq, Hash, PartialOrd, Ord)]
pub struct Ident(u32);

// Global interner (single-threaded phase, so no synchronization needed)
use std::cell::RefCell;

struct Interner {
    strings: Vec<String>,            // index → string
    lookup: HashMap<String, u32>,    // string → index
}

thread_local! {
    static INTERNER: RefCell<Interner> = RefCell::new(Interner::new());
}

impl Ident {
    pub fn intern(s: &str) -> Ident {
        INTERNER.with(|interner| {
            let mut interner = interner.borrow_mut();
            if let Some(&idx) = interner.lookup.get(s) {
                return Ident(idx);
            }
            let idx = interner.strings.len() as u32;
            interner.strings.push(s.to_string());
            interner.lookup.insert(s.to_string(), idx);
            Ident(idx)
        })
    }

    pub fn as_str(&self) -> &str {
        // Must borrow from thread-local; returns require care
        // In practice, resolve to String for display
        INTERNER.with(|interner| {
            let interner = interner.borrow();
            &interner.strings[self.0 as usize]
        })
    }
}
```

The `as_str` lifetime is tricky with `thread_local!` — the borrow cannot
escape the closure. Practical alternatives:

1. Return `String` (clone on access — acceptable since display/print is
   infrequent compared to comparison/hashing)
2. Use a `&'static str` approach where interned strings are leaked into
   `&'static str` (never freed, but the interner lives for the process)
3. Use an arena allocator that outlives all uses

### Performance impact

**Equality:** Currently `Arc<String>` equality compares string bytes —
O(n) in string length. With interning, equality is `self.0 == other.0` —
O(1). This benefits every HashMap keyed on `Ident` or `Path`:

- `HashMap<Ident, Fun>` (context.rs:102)
- `HashMap<Path, Trait>` (context.rs:110)
- `HashMap<Path, OpaqueType>` (context.rs:103)
- `HashMap<Ident, u64>` (ast_to_sst.rs:87)
- `HashMap<Fun, Function>` — `Fun` contains a `Path` which contains `Idents`
- At least 8 more maps in `Ctx` alone

**Hashing:** Currently hashing an `Ident` hashes the full string bytes.
With interning, hashing the `u32` index is O(1). Every HashMap lookup and
insertion on paths becomes faster.

**Memory:** Each `Arc<String>` costs at minimum 48 bytes (16 bytes Arc
refcount + 16 bytes String header + 8 bytes allocation overhead + string
data). An `Ident(u32)` is 4 bytes — stored inline, no heap allocation.
For ~5 million Ident instances with ~20K unique strings, this saves
roughly:

- Eliminated: 5M × 48 bytes = ~240 MB of Arc<String> allocations
- Added: 20K × ~60 bytes = ~1.2 MB for interner table
- Net: ~239 MB saved

**Copy semantics:** `Ident(u32)` is `Copy`, eliminating all `.clone()`
calls on Ident values. No more refcount increments/decrements.

### What would break

**1. Serialization (~30 files)**

`PathX` derives `Serialize, Deserialize` (`ast.rs:26`). With `Ident` as
a `u32`, serialization would emit indices instead of strings. Fix: custom
`Serialize`/`Deserialize` impls that resolve the index to/from a string.
Alternatively, implement `serde::Serialize` on the `Ident` newtype itself
so it serializes transparently as a string.

```rust
impl Serialize for Ident {
    fn serialize<S: Serializer>(&self, s: S) -> Result<S::Ok, S::Error> {
        self.to_string().serialize(s)
    }
}

impl<'de> Deserialize<'de> for Ident {
    fn deserialize<D: Deserializer<'de>>(d: D) -> Result<Self, D::Error> {
        let s = String::deserialize(d)?;
        Ok(Ident::intern(&s))
    }
}
```

**2. Display/printing (~20 call sites)**

Code that formats Idents (e.g. `def.rs:319` `path_to_string()` joining
segments with `"."`) currently dereferences the `Arc<String>` directly.
These would need `ident.as_str()` or `ident.to_string()` calls. The
`Display` impl on the newtype handles most cases:

```rust
impl fmt::Display for Ident {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        f.write_str(&self.to_string())
    }
}
```

**3. String construction patterns (~109 call sites)**

Every `Arc::new(x.to_string())` becomes `Ident::intern(&x)`. Every
`str_ident("foo")` becomes `Ident::intern("foo")`. These are mechanical
search-and-replace changes but span 27+ source files across `vir/src/`
and `air/src/`.

**4. Pattern matching and mutation**

Code in `def.rs` that builds new Idents by string concatenation:

```rust
// def.rs:341 — appends a suffix to an Ident
let x = segments.last().expect("segment").to_string() + TRAIT_DEFAULT_SEPARATOR;
*segments.last_mut().unwrap() = Arc::new(x);
```

Becomes:

```rust
let x = segments.last().expect("segment").to_string() + TRAIT_DEFAULT_SEPARATOR;
*segments.last_mut().unwrap() = Ident::intern(&x);
```

Functionally identical. The new concatenated string gets interned (or
found in the table if it already exists).

**5. `VarIdent` wrapper (`ast.rs:77`)**

```rust
pub struct VarIdent(pub Ident, pub VarIdentDisambiguate);
```

This struct wraps an `Ident`. It derives `Serialize, Deserialize, Hash,
Eq`. With interned `Ident`, its derives continue to work — `Hash` and
`Eq` on a `u32` are free. Its `Display` impl (`ast_util.rs:1072`) calls
`self.into()` to convert to String, which would call `Ident::to_string()`.

**6. The `Idents` type**

```rust
pub type Idents = Arc<Vec<Ident>>;
```

Changes from `Arc<Vec<Arc<String>>>` (24 bytes per element) to
`Arc<Vec<u32>>` (4 bytes per element). A path with 5 segments shrinks
from ~120 bytes to ~20 bytes plus the Arc/Vec overhead. This compounds
with path deduplication.

### Scope of the refactor

| Area | Files affected | Nature of change |
|------|---------------|------------------|
| Type definition | `vir/src/ast.rs` | Replace alias with newtype + interner |
| Construction | ~27 files | `Arc::new(s.to_string())` → `Ident::intern(&s)` |
| `str_ident` helper | `air/src/ast_util.rs` | Body changes to `Ident::intern(x)` |
| Serialization | `ast.rs` + any custom impls | Custom Serialize/Deserialize on newtype |
| Display/formatting | ~20 call sites | `.as_str()` or `.to_string()` where needed |
| HashMap key types | No changes needed | `Ident` still implements `Hash + Eq` |
| `VarIdent` | No changes needed | Inner `Ident` type change is transparent |
| Tests | Unknown | May need interner initialization |
| **Estimated total** | **~30 files, ~200-300 LOC** | Mostly mechanical |

### Comparison: DefId cache vs string interner

| Aspect | DefId cache | String interner |
|--------|-------------|-----------------|
| Memory saved | ~167 MB (path dedup) | ~239 MB (all Ident dedup) |
| Equality speedup | None | O(n) → O(1) per comparison |
| Hashing speedup | None | O(n) → O(1) per hash |
| Copy semantics | No (still Arc clone) | Yes (u32 is Copy) |
| Files changed | 1 | ~30 |
| Risk | Low | Medium-high |
| Incremental | Yes (do first) | Yes (do second, replaces cache) |

### Recommendation

The DefId cache and the string interner are not mutually exclusive but
the interner subsumes the cache. If the interner is done, the DefId
cache becomes unnecessary — interned Idents already deduplicate, so
`def_id_to_vir_path` naturally returns shared Ident values without a
separate cache.

However, the DefId cache is still the right first step:

1. It delivers ~167 MB savings in 10 lines with near-zero risk
2. It validates the assumption that path deduplication is safe
3. The string interner can be attempted later as a larger project
4. If the interner is never done, the cache still captures most of the win

## Recommendation

Start with the DefId cache (10 lines, low risk, gets ~90% of the savings).
Defer Ident interning to a separate effort if the cache proves insufficient.

## No patch provided

This plan is more speculative. A patch would need validation against the full
verus test suite, and the cache approach needs sign-off from verus maintainers
on thread-safety and cache-lifetime assumptions.
