# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

@AGENTS.md

## Project shape

NautilusTrader v2 is a Rust-native trading engine (Cargo workspace under `crates/`) with a thin Python
package (`python/nautilus_trader/`) projected from Rust through PyO3. There is no Cython: the legacy v1
Cython package lives on the `develop_v1` branch (see `MIGRATION_V2.md`).

- `crates/pyo3` builds the single extension module `nautilus_trader._libnautilus`. Its `src/lib.rs`
  submodule list is the public Python API allowlist, enforced by pre-commit hooks.
- Most `python/nautilus_trader/**` modules re-export bindings. The `.pyi` stubs there and the wrapper
  doc comments under `crates/**/src/python/` are generated: change the Rust item, its
  `pyo3_stub_gen` annotation, or its core doc comment, then run `make py-stubs` and commit the
  regenerated files. `make check-generated-drift` verifies them.
- Each crate keeps its PyO3 projection in `src/python/` behind the `python` feature. Only
  `nautilus-core` and `nautilus-model` have an `ffi` feature (C ABI via cbindgen).
- Fixed-point values use 64-bit (standard) or 128-bit (`high-precision` feature) raw storage.
  Propagate `high-precision` to any dependent crate that stores or constructs those values.

## Architecture

Backtest, sandbox, and live nodes share one `NautilusKernel` (`crates/system`). The kernel owns the
`MessageBus`, `Cache`, `DataEngine`, `RiskEngine`, `ExecutionEngine`, `Portfolio`, `Trader`, and
clock; only the environment context (data source and execution setting) differs. Read
`docs/concepts/architecture.md` for data and execution flow traces.

- Market data: adapter data client → channel → `DataEngine` → `Cache` → `MessageBus` publish →
  actor/strategy handlers.
- Trading commands: strategy → `RiskEngine` → `ExecutionEngine` → execution client. Venue reports
  and fills flow back as events on the bus to the portfolio, risk engine, and cache.
- Live reconciliation: `ExecutionManager` reconciles venue reports and `LiveNode` owns recurring
  checks, deadlines, and cancellation (both in `crates/live`).
- Venue adapters live in `crates/adapters/<venue>` (`common/`, `http/`, `websocket/`, `data.rs`,
  `execution.rs`, `factories.rs`, `python/`). A new adapter crate must also be added to the workspace
  `Cargo.toml`, `ADAPTER_CRATES` in the `Makefile`, `crates/pyo3/Cargo.toml`, and
  `crates/pyo3/src/lib.rs`. See `docs/developer_guide/adapters.md`.
- Async code that should run under deterministic simulation (DST, madsim) must route time, tasks,
  and signals through `nautilus_common::live::dst` and `nautilus_core::time`, not `tokio` or
  `SystemTime` directly. See the DST readiness checklist in `docs/developer_guide/testing.md`.

Repository-specific conventions are enforced by scripts in `.pre-commit-hooks/` (Cargo layout, PyO3,
tokio usage, DST, error names, logging, rustdoc, test style). The source-of-truth table at the top of
`docs/developer_guide/rust.md` maps each concern to its hook.

## Commands

Requires GNU Make, the pinned Rust toolchain (`rust-toolchain.toml`), uv, and the tools installed by
`make install-tools`. `make help` lists every target.

Environment (Linux/macOS), needed before Rust tests that touch PyO3:

```bash
make sync                                    # Python deps into python/.venv
export PYO3_PYTHON="$PWD/python/.venv/bin/python"
export PYTHONHOME="$("$PYO3_PYTHON" -c 'import sys; print(sys.base_prefix)')"
# Linux only:
export LD_LIBRARY_PATH="$("$PYO3_PYTHON" -c 'import sysconfig; print(sysconfig.get_config_var("LIBDIR"))')${LD_LIBRARY_PATH:+:$LD_LIBRARY_PATH}"
```

Build and lint:

```bash
make build-debug          # regenerate stubs if needed, then maturin develop (nextest profile)
make py-stubs             # regenerate .pyi stubs and wrapper docs
make format               # cargo +nightly fmt and ruff format
make check-code           # clippy -D warnings (lib/tests) and ruff --fix
make pre-commit           # all prek hooks on all files
cargo check -p nautilus-model --features python   # fast single-crate check
```

Rust tests (always use nextest: the suite relies on per-test process isolation, so plain
`cargo test --workspace` is not a supported gate):

```bash
make cargo-test                  # full workspace
make cargo-test-core-local       # core crates only, faster local compile
make cargo-test-doc              # doctests (nextest cannot run them)
cargo nextest run --cargo-profile nextest -p nautilus-model --lib                 # one crate
cargo nextest run --cargo-profile nextest -p nautilus-model -E 'test(test_name)'  # one test
```

Add `--features "$(bash scripts/cargo-features.bash)"` to match the Makefile's feature set. Some
tests need the large fixtures described in `docs/developer_guide/test_datasets.md`.

Python tests (require a current extension build):

```bash
make pytest                                              # builds debug first, then runs all
cd python && uv run --no-sync pytest tests/unit/path/test_file.py::test_name   # one test
```

`make pre-flight` is the broad local validation suite (format, clippy, Rust and Python tests,
generated drift, security audit). It requires staged changes and strips adapter environment variables.

## Agent skills

### Issue tracker

Issues and specs live in GitHub Issues on `nextlevelcyber/nautilus`. See `docs/agents/issue-tracker.md`.

### Triage labels

Default labels: `needs-triage`, `needs-info`, `ready-for-agent`, `ready-for-human`, `wontfix`. See
`docs/agents/triage-labels.md`.

### Domain docs

Single-context: root `CONTEXT.md` and `docs/adr/`, created lazily. See `docs/agents/domain.md`.
