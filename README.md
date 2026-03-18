<style>
body { max-width: 98% !important; width: 98% !important; margin: 0 !important; padding: 1em !important; }
.markdown-body { max-width: 98% !important; width: 98% !important; }
.container, .container-lg, .container-xl, main, article { max-width: 98% !important; width: 98% !important; }
table { width: 100% !important; table-layout: fixed; }
</style>

# verus-heaptrack

Memory profiling toolkit for the Verus verification tool.

## Problem

Verus (rust_verify) uses ~1.36 GB of heap memory when verifying large crates
like APAS-VERUS (4076 verified functions). Virtually none of it is freed before
process exit. This project contains the heaptrack analysis, proposed patches,
and scripts for profiling and benchmarking.

## Installation

### heaptrack

```bash
# Ubuntu/Debian
sudo apt install heaptrack heaptrack-gui

# Fedora
sudo dnf install heaptrack

# Arch
sudo pacman -S heaptrack
```

### rustfilt (for demangling Rust symbols)

```bash
cargo install rustfilt
```

### GNU time

```bash
# Usually pre-installed. Verify:
/usr/bin/time --version
# If missing:
sudo apt install time
```

### Verus (baseline)

Assumes Verus is built at `~/projects/verus/source/target-verus/release/`.
The toolchain (currently `1.93.1-x86_64-unknown-linux-gnu`) must be installed
via rustup.

## Quick start

### 1. Profile with heaptrack

```bash
cd ~/projects/APAS-VERUS
~/projects/verus-heaptrack/scripts/verus-heaptrack src/lib.rs \
    --multiple-errors 20 --expand-errors --num-threads 8
```

### 2. Benchmark baseline (no heaptrack overhead)

```bash
cd ~/projects/APAS-VERUS
~/projects/verus-heaptrack/scripts/verus-benchmark src/lib.rs \
    --multiple-errors 20 --expand-errors --num-threads 8
```

### 3. Set up patched verus fork and compare

```bash
# Clone verus, apply the drop-vir-krate-early patch, and build:
~/projects/verus-heaptrack/scripts/setup-verus-fork --apply-patch

# Run baseline vs patched comparison (3 runs each, timed, peak RSS):
cd ~/projects/APAS-VERUS
~/projects/verus-heaptrack/scripts/verus-compare src/lib.rs \
    --multiple-errors 20 --expand-errors --num-threads 8
```

Results are saved to `results/`.

## Scripts

| Script | Purpose |
|--------|---------|
| `scripts/verus-heaptrack` | Run heaptrack on rust_verify (bypasses verus launcher) |
| `scripts/verus-benchmark` | Measure wall time + peak RSS with `/usr/bin/time -v` (no heaptrack) |
| `scripts/verus-compare` | Run baseline and patched benchmarks back-to-back |
| `scripts/setup-verus-fork` | Clone verus-lang/verus, optionally apply patch, build |

### Environment variables

| Variable | Default | Used by |
|----------|---------|---------|
| `VERUS_ROOT` | `~/projects/verus/source/target-verus/release` | All scripts |
| `VERUS_TOOLCHAIN` | `1.93.1-x86_64-unknown-linux-gnu` | All scripts |
| `VERUS_ORIGINAL` | `~/projects/verus/source/target-verus/release` | `verus-compare` |
| `VERUS_PATCHED` | `~/projects/verus-patched/source/target-verus/release` | `verus-compare` |
| `BENCHMARK_LABEL` | `run` | `verus-benchmark` |
| `BENCHMARK_RUNS` | `3` | `verus-benchmark`, `verus-compare` |

## Setting up a verus fork

### Fresh clone + patch + build

```bash
# Clone, apply patch, build:
~/projects/verus-heaptrack/scripts/setup-verus-fork --apply-patch
```

This creates `~/projects/verus-patched/` with branch `heaptrack-memory-fix`.

### Manual setup

```bash
git clone https://github.com/verus-lang/verus.git ~/projects/verus-patched
cd ~/projects/verus-patched
git checkout 76e69b81   # Pinned commit matching analysis
git checkout -b heaptrack-memory-fix

# Apply the patch:
git apply ~/projects/verus-heaptrack/patches/drop-vir-krate-early.patch
git add -A && git commit -m "Drop VIR Krate after SST conversion"

# Build (vargo must be built first):
cd source
cd tools/vargo && cargo build --release && cd ../..
./tools/vargo/target/release/vargo build --release
```

### Verify correctness

After building the patched version, run the verus test suite:

```bash
cd ~/projects/verus-patched/source
./tools/vargo/target/release/vargo test --release
```

## Why heaptrack on verus doesn't work out of the box

The `verus` binary (4.1 MB) is a thin launcher that exec's:

    rustup run <toolchain> -- rust_verify <args>

On Unix, `exec()` replaces the process, which resets LD_PRELOAD. heaptrack's
LD_PRELOAD-based interception dies at the exec boundary, producing ~70 KB of
trace (just the launcher's allocations) instead of the real data.

The fix: invoke `rust_verify` directly with `LD_LIBRARY_PATH` pointing to the
toolchain's lib directory. That's what `scripts/verus-heaptrack` does.

## Proposed patches

See `plans/` for two proposed changes:

1. **plan-drop-vir-krate-early.md** — Drop the VIR Krate after SST conversion.
   **Tested 2026-03-17: no measurable effect** with `--num-threads 8`.
   Arc refcounting keeps the data alive until all worker threads finish.
   See `analysis/experiment-drop-vir-krate-early.md` for full results.

2. **plan-intern-paths.md** — Intern VIR path symbols to deduplicate 4.97M
   allocations down to ~20K unique paths. Saves ~167 MB. Not yet tested.
   Unaffected by threading — the more promising approach.

## Analysis

- `analysis/heaptrack-findings.md` — Full heaptrack analysis: binary architecture,
  allocation layers, peak memory consumers, VIR Krate lifetime trace
- `analysis/experiment-drop-vir-krate-early.md` — Experiment results showing why
  the Krate drop patch is ineffective with multi-threaded verification, and what
  approaches would actually reduce memory
