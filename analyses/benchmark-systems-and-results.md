# Benchmark Systems and Results

**Date:** 2026-03-18
**Verus commit:** `f04abf70` (toolchain 1.94.0, `--features singular`)

## Results

glibc default vs jemalloc `narenas:2` on Linux, 6 Verus projects, quiet machine.
macOS not yet benchmarked. Not applicable on Windows.

| # | Project | Verified | glibc MB | jemal MB | RSS Δ | glibc s | jemal s | Time Δ |
|---|---------|----------|----------|----------|-------|---------|---------|--------|
| 1 | human-eval | 292 | 614 | 537 | -13% | 5.4s | 5.1s | -4% |
| 2 | ironkv | 319 | 783 | 706 | -10% | 5.4s | 5.1s | -5% |
| 3 | node-replication | 254 | 957 | 854 | -11% | 6.5s | 5.5s | -14% |
| 4 | vest | 496 | 648 | 565 | -13% | 5.7s | 5.2s | -9% |
| 5 | memory-allocator | 731 | 1,629 | 1,416 | -13% | 38.7s | 35.2s | -9% |
| 6 | APAS-VERUS | 4,265 | 5,822 | 5,143 | -12% | 59.2s | 53.2s | -10% |
| | **Peak** | | | | **-14%** | | | **-14%** |
| | **Average** | | | | **-12%** | | | **-9%** |

Per-project raw data: `results/benchmark-all-20260318-123432/`

## Benchmark Systems

All fixtures are shallow clones (`--depth 1`) in `tests/fixtures/`.
APAS-VERUS counted with `-e experiments -e attic`.

| # | Project | Repo | Spec (LOS) | Proof (LOP) | Exec (LOE) | Rust (LOR) | Total | Files | Verified |
|---|---------|------|------|-------|------|------|-------|-------|----------|
| 1 | human-eval | secure-foundations/human-eval-verus | 1,694 | 1,000 | 4,574 | 7,092 | 22,498 | 265 | 292 |
| 2 | ironkv | verus-lang/verified-ironkv | 2,650 | 2,347 | 2,993 | 367 | 10,924 | 34 | 319 |
| 3 | node-replication | verus-lang/verified-node-replication | 2,696 | 1,773 | 2,741 | 850 | 11,483 | 18 | 254 |
| 4 | memory-allocator | verus-lang/verified-memory-allocator | 5,290 | 5,801 | 5,709 | 1,670 | 24,381 | 28 | 731 |
| 5 | vest | secure-foundations/vest | 1,721 | 1,863 | 1,567 | 256 | 10,720 | 23 | 496 |
| 6a | verified-storage/pmemlog | microsoft/verified-storage | 1,178 | 667 | 505 | 79 | 3,228 | 9 | 81 |
| 6b | verified-storage/multilog | microsoft/verified-storage | 2,590 | 1,037 | 2,701 | 2,501 | 13,244 | 24 | 167 |
| 6c | verified-storage/capybarakv | microsoft/verified-storage | 9,333 | 4,048 | 6,762 | 3,328 | 35,153 | 96 | 725 |
| 7 | APAS-VERUS | briangmilnes/APAS-VERUS | 22,640 | 29,968 | 53,120 | 9,664 | 151,464 | 303 | 4,265 |

verified-storage (6a/6b/6c) requires `cargo verus focus` which spawns `rust_verify` as
a child process. `/usr/bin/time -v` reports peak RSS across the entire process tree, so
we cannot isolate `rust_verify`'s memory usage from cargo's build processes.

**APAS-VERUS size notes:** APAS-VERUS is a teaching tool where each algorithm is implemented
in up to four variants: single-threaded persistent, single-threaded ephemeral, multi-threaded
persistent, and multi-threaded ephemeral. Both iterative and recursive proofs of many
algorithms will be released. The current LOC will be trimmed before release; the project is
in active development using AI agents, and `veracity-minimize-lib` has not yet been run to
remove dead code, unused modules, or duplicated proof infrastructure.

## Totals

|   | Spec (LOS) | Proof (LOP) | Exec (LOE) | Rust (LOR) | Total | Files | Verified |
|---|------|-------|------|------|-------|-------|----------|
| All fixtures | 49,792 | 48,504 | 80,672 | 25,807 | 283,095 | 800 | 7,330 |

## Line Categories

Counted by `veracity-count-loc -l Verus`. Each line is classified into exactly one category
(no double counting). Lines not matching any category (blanks, comments, imports, attributes)
are uncategorized and included only in the Total column.

| Abbrev | Column | What it counts |
|--------|--------|----------------|
| LOS | Spec | `spec fn` bodies, `open spec fn` bodies, `requires`, `ensures`, `recommends`, `invariant`, `decreases` clauses |
| LOP | Proof | `proof fn` bodies (excluding their requires/ensures), `proof { }` blocks inside exec functions, `assert`, `assume`, `reveal`, `let ghost`, `let tracked` |
| LOE | Exec | `fn` bodies (excluding requires/ensures/proof blocks) — executable code |
| LOR | Rust | Lines outside `verus! { }` blocks — plain Rust code not under Verus verification |

## Validation Status

All fixtures verified clean against Verus `f04abf70` (1.94.0).

| # | Project | Invocation | Result |
|---|---------|------------|--------|
| 1 | human-eval | `rust_verify tasks/lib.rs --crate-type=lib` | 292 verified, 0 errors |
| 2 | ironkv | `rust_verify ironsht/src/lib.rs --crate-type=lib` | 319 verified, 0 errors |
| 3 | node-replication | `rust_verify .../src/lib.rs --crate-type=dylib` | 254 verified, 0 errors |
| 4 | memory-allocator | `rust_verify verus-mimalloc/lib.rs --extern libc=... --rlimit 240` | 731 verified, 0 errors |
| 5 | vest | `rust_verify vest/src/lib.rs --crate-type=lib` | 496 verified, 0 errors |
| 6a | verified-storage/pmemlog | `cargo verus focus` | 81 verified, 0 errors |
| 6b | verified-storage/multilog | `cargo verus focus` | 167 verified, 0 errors |
| 6c | verified-storage/capybarakv | `cargo verus focus` | 725 verified, 0 errors |
| 7 | APAS-VERUS | `rust_verify --crate-type=lib src/lib.rs --num-threads 8` | 4,265 verified, 0 errors |

## Prerequisites

- **rustc 1.94.0**: `rustup install 1.94.0` (for cargo verus + published vstd)
- **Singular**: `sudo apt install singular` (for integer_ring proofs)
- **libpmem-dev**: `sudo apt install libpmem-dev` (for verified-storage pmem bindings)
- **clang**: `sudo apt install clang` (for bindgen in verified-storage build.rs)

Verus built with: `vargo build --release --features singular`

