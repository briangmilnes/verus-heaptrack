# Plan 4: Benchmark All Verita Projects

**Goal:** Measure peak RSS under glibc vs jemalloc for every active Verus
project in verita's configuration, not just APAS-VERUS.

**Motivation:** Our allocator comparison only tested one project (APAS-VERUS,
167 kLOC). Different codebases have different allocation profiles — a project
with fewer small Arc allocations might see less jemalloc benefit, while one
with deeper recursive types might see more. Testing all 6 active projects
(plus APAS-VERUS) gives a representative picture of the jemalloc win across
the Verus ecosystem, which strengthens the case for a `#[global_allocator]`
PR.

## Fixtures

Pin each project at a known-good commit in `tests/fixtures/`. Each fixture
is a shallow clone (~1 depth) to keep disk usage low.

```
tests/fixtures/
├── README.md               # Documents pinned commits, dates, how to update
├── fetch-fixtures.sh        # Clones/updates all fixtures
├── ironkv/                  # verus-lang/verified-ironkv
├── node-replication/        # verus-lang/verified-node-replication
├── verified-storage/        # microsoft/verified-storage (3 crates)
├── memory-allocator/        # verus-lang/verified-memory-allocator
├── vest/                    # secure-foundations/vest
├── human-eval/              # secure-foundations/human-eval-verus
└── apas-verus/              # briangmilnes/APAS-VERUS (pinned)
```

Ignored projects (anvil, page-table) are excluded — they don't verify
against current Verus so benchmarking them is pointless.

### fetch-fixtures.sh

Clones each repo at a pinned commit. Runs prepare_script where needed
(memory-allocator needs liblibc.rlib, vest needs `cargo install vest`).
Idempotent — skips repos already cloned at the right commit.

```bash
#!/usr/bin/env bash
set -euo pipefail

FIXTURES_DIR="$(cd "$(dirname "$0")" && pwd)"

declare -A REPOS=(
  [ironkv]="https://github.com/verus-lang/verified-ironkv.git"
  [node-replication]="https://github.com/verus-lang/verified-node-replication.git"
  [verified-storage]="https://github.com/microsoft/verified-storage.git"
  [memory-allocator]="https://github.com/verus-lang/verified-memory-allocator.git"
  [vest]="https://github.com/secure-foundations/vest.git"
  [human-eval]="https://github.com/secure-foundations/human-eval-verus.git"
  [apas-verus]="https://github.com/briangmilnes/APAS-VERUS.git"
)

# Pinned commits — update these when re-baselining.
# Each must be a commit where the project verifies cleanly against
# Verus 76e69b81. TODO: pin to specific SHAs after first successful run.
declare -A COMMITS=(
  [ironkv]="main"
  [node-replication]="main"
  [verified-storage]="main"
  [memory-allocator]="main"
  [vest]="main"
  [human-eval]="main"
  [apas-verus]="e3a5c74d"
)

for name in "${!REPOS[@]}"; do
  dir="$FIXTURES_DIR/$name"
  url="${REPOS[$name]}"
  ref="${COMMITS[$name]}"

  if [ -d "$dir/.git" ]; then
    echo "SKIP $name (already cloned)"
    # TODO: verify commit matches pinned SHA
    continue
  fi

  echo "CLONE $name from $url at $ref"
  git clone --depth 1 --branch "$ref" "$url" "$dir"
done

# Prepare scripts (match verita config)
if [ -d "$FIXTURES_DIR/memory-allocator" ]; then
  echo "PREPARE memory-allocator: building liblibc.rlib"
  pushd "$FIXTURES_DIR/memory-allocator"
  pushd test_libc && cargo clean && cargo build --release && popd
  LIBC_RLIB=$(find ./test_libc/target/release/deps/ -name 'liblibc-*.rlib' | head -1)
  mkdir -p build && cp "$LIBC_RLIB" build/liblibc.rlib
  popd
fi

if [ -d "$FIXTURES_DIR/vest" ]; then
  echo "PREPARE vest: installing vest binary"
  pushd "$FIXTURES_DIR/vest"
  cargo install vest
  popd
fi

echo "All fixtures ready."
```

### .gitignore for fixtures

Fixtures are too large for git. Add to `.gitignore`:

```
tests/fixtures/*/
!tests/fixtures/README.md
!tests/fixtures/fetch-fixtures.sh
```

## Benchmark Script

New script: `scripts/benchmark-all-projects`. For each fixture, runs
`allocator-compare` with the project-specific verus args (matching verita's
`crate_roots` and `extra_args`).

### Project invocation table

How each project maps to a rust_verify / verus command:

| # | Project | cargo_verus | Invocation |
|---|---------|-------------|------------|
| 1 | ironkv | yes | `cargo verus focus` in `ironsht/` |
| 2 | node-replication | no | `rust_verify --crate-type=dylib src/lib.rs` |
| 3 | verified-storage | yes | `cargo verus focus` in each of 3 dirs |
| 4 | memory-allocator | no | `rust_verify verus-mimalloc/lib.rs --extern libc=... --rlimit 240` |
| 5 | vest | yes | `cargo verus focus` in `vest/` and `vest-examples/tls_dsl/` |
| 6 | human-eval | no | `rust_verify tasks/lib.rs --crate-type=lib` |
| 7 | apas-verus | no | `rust_verify --crate-type=lib src/lib.rs --num-threads 8` |

### cargo_verus complication

Our current `allocator-compare` script calls `rust_verify` directly with
`LD_PRELOAD` to swap the allocator. For `cargo_verus` projects, we need to
set `LD_PRELOAD` in the environment of the `cargo verus focus` command
instead. This still works because `cargo verus` ultimately spawns
`rust_verify` as a child process, and `LD_PRELOAD` is inherited.

The script needs two modes:
1. **Direct mode**: `rust_verify <args>` (current behavior)
2. **Cargo-verus mode**: `cargo verus focus -- <args>` (new)

### Output

Results go to `results/benchmark-all-YYYYMMDD-HHMMSS/` with one file per
project:

```
results/benchmark-all-20260318-120000/
├── summary.txt               # Combined table
├── ironkv.txt
├── node-replication.txt
├── verified-storage-capybarakv.txt
├── verified-storage-multilog.txt
├── verified-storage-pmemlog.txt
├── memory-allocator.txt
├── vest-vest.txt
├── vest-tls_dsl.txt
├── human-eval.txt
└── apas-verus.txt
```

### Estimated run time

Each project runs 5 times (warmup + 4 allocator configs). Rough estimates
based on typical Verus verification times:

| # | Project | Est. per-run | x5 | Cumulative |
|---|---------|-------------|-----|------------|
| 1 | ironkv | ~30s | 2.5m | 2.5m |
| 2 | node-replication | ~20s | 1.7m | 4.2m |
| 3 | verified-storage (3) | ~45s each | 11m | 15m |
| 4 | memory-allocator | ~60s | 5m | 20m |
| 5 | vest (2) | ~90s each | 15m | 35m |
| 6 | human-eval | ~15s | 1.3m | 36m |
| 7 | apas-verus | ~60s | 5m | 41m |

**Total estimate: ~40-50 minutes** on a quiet machine. Must be run
sequentially (no parallel verification).

## Deliverable

A summary table like:

| # | Project | Rust kLOC | glibc RSS | jemal RSS | RSS saved | glibc time | jemal time | Time saved |
|---|---------|-----------|-----------|-----------|-----------|------------|------------|------------|
| 1 | ironkv | 11 | ? | ? | ? | ? | ? | ? |
| ... | | | | | | | | |
| 7 | apas-verus | 167 | 5.74 GB | 5.06 GB | 714 MB (12%) | 60.0s | 52.9s | 7.1s (12%) |

This table, covering 7 projects and ~460 kLOC of verified Rust, would be
strong evidence for a `#[global_allocator]` PR to verus-lang.

## Steps

1. Write `tests/fixtures/fetch-fixtures.sh` and `.gitignore`
2. Clone all 6 external fixtures, set up APAS-VERUS symlink
3. Run prepare_scripts for memory-allocator and vest
4. Extend `allocator-compare` to support cargo-verus mode (or write a
   wrapper that sets LD_PRELOAD and delegates to `cargo verus focus`)
5. Write `scripts/benchmark-all-projects`
6. Run on a quiet machine
7. Write up results in `analysis/experiment-benchmark-all-projects.md`

## Risks

- **Fixture staleness**: Projects track `main`, which may not verify against
  our pinned Verus (76e69b81). Pin to specific SHAs after first successful
  run.
- **cargo-verus version**: Our Verus build must include `cargo-verus`. Check
  that it exists at `target-verus/release/cargo-verus`.
- **vest prepare_script**: `cargo install vest` builds from crates.io, which
  may not match the pinned repo version. May need to build from source.
- **Machine time**: 40-50 minutes of exclusive CPU. Must be quiet — no
  agents, no other builds.
