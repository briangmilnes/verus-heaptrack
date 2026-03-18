<style>
body { max-width: 98% !important; width: 98% !important; margin: 0 !important; padding: 1em !important; }
.markdown-body { max-width: 98% !important; width: 98% !important; }
.container, .container-lg, .container-xl, main, article { max-width: 98% !important; width: 98% !important; }
table { width: 100% !important; table-layout: fixed; }
</style>

# Setting Up a Verus Fork for Memory Patches

Step-by-step instructions to set up `briangmilnes/verus-lang` (a fork of
`verus-lang/verus`), apply the memory patch, build, and benchmark.

## Pinned versions

| Component | Commit / Version | Notes |
|-----------|-----------------|-------|
| verus-lang/verus | `76e69b81` | Base commit for the patch |
| APAS-VERUS | `e3a5c74d` | Benchmark target (R36 prompts) |
| Rust toolchain | `1.93.1-x86_64-unknown-linux-gnu` | Pinned in verus's `rust-toolchain.toml` |

The verus commit and APAS-VERUS commit must stay in sync. APAS-VERUS at
`e3a5c74d` was built and verified against verus at `76e69b81`. Using a
different verus commit will likely produce compilation errors (e.g., the
`HashSet` type parameter change on newer main).

## Baseline measurements (unpatched, commit 76e69b81)

Measured 2026-03-17 on APAS-VERUS at `e3a5c74d`:

```
verified:    4265
wall time:   0:58.52 / 0:58.60  (two runs)
peak RSS:    5,979,512 KB / 5,983,340 KB  (~5.7 GB)
user time:   207.70s / 209.67s
```

## Part 1: Reset the fork on GitHub

The fork `briangmilnes/verus-lang` already exists. To reset it to match
upstream:

1. Go to https://github.com/briangmilnes/verus-lang
2. Click **Sync fork** → **Update branch** (or **Discard commits** if ahead)

Or from CLI:

```bash
gh api repos/briangmilnes/verus-lang/merge-upstream -f branch=main
```

## Part 2: Clone locally

```bash
cd ~/projects
git clone https://github.com/briangmilnes/verus-lang.git
cd verus-lang
git remote add upstream https://github.com/verus-lang/verus.git
```

## Part 3: Check out the pinned commit and create a branch

```bash
git checkout 76e69b81
git checkout -b heaptrack-memory-fix
```

## Part 4: Get Z3 and build (unpatched baseline)

```bash
cd source
./tools/get-z3.sh
source ../tools/activate
vargo build --release
```

If rustup says the toolchain is missing:

```bash
rustup toolchain install    # reads rust-toolchain.toml automatically
```

Binaries land in `~/projects/verus-lang/source/target-verus/release/`.

## Part 5: Benchmark the unpatched build

From the APAS-VERUS checkout at `e3a5c74d`:

```bash
cd ~/projects/APAS-VERUS

/usr/bin/time -v env \
  LD_LIBRARY_PATH=$HOME/.rustup/toolchains/1.93.1-x86_64-unknown-linux-gnu/lib \
  VSTD_KIND=Imported \
  VERUS_Z3_PATH=$HOME/projects/verus-lang/source/target-verus/release/z3 \
  RUSTC_BOOTSTRAP=1 \
  $HOME/projects/verus-lang/source/target-verus/release/rust_verify \
    --crate-type=lib src/lib.rs \
    --multiple-errors 20 --expand-errors --num-threads 8
```

Confirm it matches the baseline numbers above (4265 verified, ~58s, ~5.7 GB).

## Part 6: Apply the patch

```bash
cd ~/projects/verus-lang
git apply ~/projects/verus-heaptrack/patches/drop-vir-krate-early.patch
```

If the patch doesn't apply cleanly, the changes are 4 insertions in one file,
`source/rust_verify/src/verifier.rs`:

**After line 2046** (`let krate = self.vir_crate.clone()...`), add:
```rust
        self.vir_crate = None;
```

**After line 2003** (`let krate_sst = vir::ast_to_sst_crate::ast_to_sst_krate(...)?;`), add:
```rust
        drop(pruned_krate);
```

**After line ~2222** (end of the thread-spawn `for` loop, before `let multi_progress`), add:
```rust
            drop(krate);
```

**After line ~2554** (end of the single-threaded `for bucket_id` loop), add:
```rust
            drop(krate);
```

## Part 7: Commit, rebuild, and push

```bash
git add -A
git commit -m "Drop VIR Krate after SST conversion to reduce peak memory"

cd source
source ../tools/activate
vargo build --release

git push origin heaptrack-memory-fix
```

## Part 8: Benchmark the patched build

Same command as Part 5 — the binaries at
`~/projects/verus-lang/source/target-verus/release/` are now patched.

```bash
cd ~/projects/APAS-VERUS

/usr/bin/time -v env \
  LD_LIBRARY_PATH=$HOME/.rustup/toolchains/1.93.1-x86_64-unknown-linux-gnu/lib \
  VSTD_KIND=Imported \
  VERUS_Z3_PATH=$HOME/projects/verus-lang/source/target-verus/release/z3 \
  RUSTC_BOOTSTRAP=1 \
  $HOME/projects/verus-lang/source/target-verus/release/rust_verify \
    --crate-type=lib src/lib.rs \
    --multiple-errors 20 --expand-errors --num-threads 8
```

Compare peak RSS and wall time against the baseline. Expected: ~300 MB
reduction in peak RSS, negligible change in wall time.

## Part 9: Run verus's own tests

```bash
cd ~/projects/verus-lang/source
source ../tools/activate
vargo test -p rust_verify_test --release
```

At commit `76e69b81`, all tests pass (0 failed). The suite runs ~129 test files
covering the verifier, syntax, traits, closures, bitvectors, etc.

## Part 10: Open a PR (when results look good)

```bash
cd ~/projects/verus-lang
gh pr create --repo verus-lang/verus \
  --base main \
  --head briangmilnes:heaptrack-memory-fix \
  --title "Drop VIR Krate after SST conversion to reduce peak memory" \
  --body "$(cat <<'EOF'
## Summary

- Release the VIR `Krate` (`Arc<KrateX>`) as soon as SST conversion completes,
  rather than holding it through the entire Z3 verification phase.
- Clear `self.vir_crate` after cloning to local in `verify_crate_inner`.
- Drop `pruned_krate` after `ast_to_sst_krate` in `verify_bucket_outer`.
- Drop local `krate` after thread spawning (multi-threaded) or after the
  bucket loop (single-threaded).

## Motivation

heaptrack profiling on a large Verus crate (4265 verified functions) shows:

- Peak heap: 1.36 GB, of which 1.36 GB is freed only at process exit
- Peak RSS: 5.7 GB
- The VIR Krate (~300 MB of `Arc<PathX>`, `Arc<SpannedTyped<ExprX>>`, etc.)
  is held alive during the entire Z3 verification phase even though all data
  was converted to `KrateSst` before Z3 starts
- `verify_bucket()` takes `&KrateSst`, never the original `Krate`

Dropping the Krate early saves ~300 MB of peak memory.

## Changes

4 insertions in `source/rust_verify/src/verifier.rs`:
- `self.vir_crate = None` after cloning to local (line 2047)
- `drop(pruned_krate)` after `ast_to_sst_krate` (line 2004)
- `drop(krate)` after thread spawn loop (line 2223)
- `drop(krate)` after single-threaded bucket loop (line 2555)

## Test plan

- [ ] `vargo test --release` passes
- [ ] Peak RSS measured before/after on a large crate
EOF
)"
```

## Cleanup

To start over: `rm -rf ~/projects/verus-lang`

Your original `~/projects/verus` is never modified.
