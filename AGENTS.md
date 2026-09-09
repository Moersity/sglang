# AGENTS.md

## Cursor Cloud specific instructions

### What this repo is, and what runs on this VM
SGLang is a GPU-first monorepo (LLM/multimodal serving runtime + CUDA kernels + a
Rust gateway + docs/tests/benchmarks). The Cursor Cloud VM has **no GPU** (CPU-only).

- The main Python package (`python/`, the `sglang` runtime) and the CUDA kernels
  (`sgl-kernel/`) require an NVIDIA CUDA GPU. Their default install pulls
  GPU-only deps (`flashinfer`, `cuda-python`, `sglang-kernel`), so they cannot be
  built or served here. Do not attempt to `pip install -e python` or serve models
  on this VM.
- The **fully CPU-runnable product is `sgl-model-gateway/`** (a Rust HTTP/gRPC
  model router / gateway). This is the component to build, lint, test, and run
  here. See `sgl-model-gateway/README.md` and `sgl-model-gateway/Makefile`.
- Repo-wide lint (`.pre-commit-config.yaml`) also runs CPU-only.

### Rust toolchain (important)
- The gateway is pinned to Rust **1.90** here (matches CI:
  `scripts/ci/cuda/ci_install_gateway_dependencies.sh`). `rustup default` is set
  to `1.90.0`. Do NOT `rustup update stable` — a newer stable/clippy (e.g. 1.97)
  introduces new lints that make `cargo clippy -- -D warnings` fail even though
  the code is clean at 1.90.
- `nightly` is installed only for `cargo +nightly fmt` (CI checks formatting with
  nightly rustfmt).

### Gateway build caveat (why plain `cargo build` works here)
The system default `cc`/`c++` is **clang**, which (a) cannot find the libstdc++
`<cstdint>` header needed by C++ build-deps like `esaxx-rs`, and (b) trips
rustc's `rust-lld` with `unable to find library -lstdc++`. This is worked around
globally in `$CARGO_HOME/config.toml` (`/usr/local/cargo/config.toml`):

```toml
[env]
CC = "gcc"
CXX = "g++"
[target.x86_64-unknown-linux-gnu]
linker = "g++"
```

So plain `cargo build` / `cargo test` / `cargo clippy` just work. If that file is
ever missing and you see `'cstdint' file not found` or `unable to find library
-lstdc++`, recreate it (or build with `CC=gcc CXX=g++ RUSTFLAGS="-C linker=g++"`).

### Build / lint / test / run the gateway
Run everything from `sgl-model-gateway/`:

- Build (dev): `cargo build --bin sgl-model-gateway` (first build is slow: large
  dep tree incl. wasmtime; ~6–10 min on this 4-core VM).
- Lint (mirrors CI): `cargo clippy --all-targets --all-features -- -D warnings`
  and `cargo +nightly fmt -- --check`.
- Tests: `cargo test` (see flaky note below). `redis-server` should be running
  for the Redis/mesh paths (`redis-cli ping` → `PONG`; start with
  `sudo service redis-server start`).
- Run (no GPU workers needed): IGW mode starts with zero workers and serves the
  control/data plane:
  `./target/debug/sgl-model-gateway launch --host 127.0.0.1 --port 30000 --enable-igw`

Pure-Rust endpoints that work without any backend worker:
`/parse/reasoning` (e.g. `{"text":"<think>..</think>answer","reasoning_parser":"deepseek_r1"}`)
and `/parse/function_call`. Health: `/liveness`, `/health`.

Worker control plane: `POST /workers` with `{"url":..,"model_id":..}` registers a
worker in the background; the gateway probes the worker's `/health`, `/server_info`,
and `/model_info` (with `/get_*` fallbacks). `GET /workers` and `/v1/models` show
registered state. To exercise data-plane routing without a GPU, point it at a small
mock HTTP worker implementing those endpoints plus `/v1/chat/completions` and
`/generate`.

### Flaky test to ignore
`policies::tree::tests::test_tree_structure_integrity_after_stress` is a randomized
8-thread concurrency stress test that occasionally fails
("Tenant … should have positive size"); it passes reliably in isolation
(`cargo test --lib test_tree_structure_integrity_after_stress`). Do not treat a
single failure of only this test as a real regression.

### Repo-wide lint (Python/etc.)
`pre-commit` is installed in a venv at `~/.venvs/precommit` (the `sglang` package
itself is NOT importable here). Run: `~/.venvs/precommit/bin/pre-commit run
--all-files` (use `SKIP=no-commit-to-branch` when on a feature branch). Hook
environments are cached under `~/.cache/pre-commit`.
