<style>
body { max-width: 98% !important; width: 98% !important; margin: 0 !important; padding: 1em !important; }
.markdown-body { max-width: 98% !important; width: 98% !important; }
.container, .container-lg, .container-xl, main, article { max-width: 98% !important; width: 98% !important; }
table { width: 100% !important; table-layout: fixed; }
</style>

# Prompt: Fix veracity-count-loc

## Context

`veracity-count-loc` is a Verus LOC counter in `~/projects/veracity/`. It classifies lines in `.rs` files
into four categories: Spec, Proof, Exec, Rust (outside `verus!`). The tool is invoked like:

```bash
veracity-count-loc -l Verus -d src           # count lines in src/
veracity-count-loc -l Verus -c               # count src/ + tests/ + benches/
veracity-count-loc -l Verus -f src/Foo.rs    # single file
```

## Bugs to fix

### 1. Proof lines are massively double-counted

The sum of Spec+Proof+Exec+Rust should never exceed the total line count of the files. Currently it does,
often by 2x or more. Evidence:

```
File: MappingStEph.rs (664 lines)
  Spec: 417, Proof: 1253, Exec: 0, Rust: 29 → sum = 1699 (2.56x actual)

File: WeightedDirGraphStEphU64.rs (444 lines)
  Spec: 2, Proof: 850, Exec: 0, Rust: 23 → sum = 875 (1.97x actual)

Full APAS-VERUS src/ (168,209 lines):
  Spec: 77,636, Proof: 160,417, Exec: 50,389, Rust: 10,640 → sum = 299,082 (1.78x actual)
```

The invariant `spec + proof + exec + rust + uncategorized == total_lines` must hold for every file. Every
line belongs to exactly one category (or is uncategorized: blank, comment, import, attribute).

### 2. requires/ensures on exec functions should count as Spec, not Exec

In Verus, `requires` and `ensures` clauses are spec-mode code even when attached to exec functions. They
should be counted as Spec lines. Currently unclear if these are double-counted as both Spec and Exec — if
so, that's part of bug #1. The classification rule should be:

- **Spec**: `spec fn` bodies, `open spec fn` bodies, `requires` clauses, `ensures` clauses, `recommends`
  clauses, `invariant` clauses, `decreases` clauses — regardless of whether they appear on spec, proof, or
  exec functions.
- **Proof**: `proof fn` bodies (excluding their requires/ensures), `proof { }` blocks inside exec
  functions, `assert`, `assume`, `reveal`, `let ghost`, `let tracked`, `Ghost(...)` parameters.
- **Exec**: `fn` bodies (excluding requires/ensures/proof blocks), `let`, `if`, `match`, `while`, `for`,
  `return`, assignments, function calls — the executable code.
- **Rust**: Lines outside `verus! { }` blocks.
- **Uncategorized**: Blank lines, comments, `use` statements, attributes, `mod` declarations, section
  headers.

### 3. Add `-e`/`--exclude` flag

Add support for excluding directories by name:

```bash
veracity-count-loc -l Verus -d src -e experiments -e standards
```

This should exclude any directory whose name matches (at any depth). Multiple `-e` flags allowed.

### 4. Default to src-only, not src+tests+benches

When called with `-c` or no arguments, the default should analyze only `src/`. Tests and benches are a
different concern — they inflate line counts and have different proof-to-exec ratios. Add `-a`/`--all` to
explicitly include tests and benches:

```bash
veracity-count-loc -l Verus -c              # src/ only (new default)
veracity-count-loc -l Verus -c -a           # src/ + tests/ + benches/
```

## Testing

After fixes, verify these invariants:

1. For every file: `spec + proof + exec + rust + uncategorized == wc -l file`
2. For APAS-VERUS `src/` (excluding experiments, standards): the sum of all categories should be close to
   `wc -l` total, never exceeding it.
3. Single-file test on `src/Chap05/MappingStEph.rs` (664 lines) — categories must sum to <= 664.
4. Single-file test on `src/Chap05/SetStEph.rs` (933 lines) — should show no double counting (it currently
   doesn't, so this is a regression test).

## Do not change

- The output format (Spec/Proof/Exec/Rust columns, per-file rows, totals line).
- The `-f`, `-d`, `-m`, `-r` flags.
- The `-p APAS` project-specific behavior.
- The `-l Verus` vs `-l Rust` distinction.
