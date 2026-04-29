# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository overview

AWS Lambda Web Adapter is a Rust binary that runs as a Lambda extension. It receives Lambda events from the Runtime API, converts them into local HTTP requests against a child web app (Express, FastAPI, Spring Boot, ASP.NET, etc.) listening on `127.0.0.1:$PORT`, and translates the HTTP responses back into Lambda responses. The same image runs unchanged on EC2/Fargate/locally because the adapter only activates inside Lambda.

The build artifact is published two ways: an OCI image at `public.ecr.aws/awsguru/aws-lambda-adapter` (copied into user Dockerfiles via `COPY --from=...`), and as a managed Lambda Layer (`LambdaAdapterLayerX86` / `LambdaAdapterLayerArm64`) that pairs with `layer/bootstrap` to wrap a non-container handler.

## Common commands

```bash
cargo build                              # Build (host target — does NOT produce a Lambda artifact)
cargo nextest run                        # Run all unit + integration tests
cargo nextest run <test_name>            # Run a single test by name (substring match)
cargo nextest run --profile ci           # CI profile: no fail-fast, JUnit output
cargo clippy --all-targets -- -Dwarnings # Lint (treat warnings as errors)
cargo fmt --all -- --check               # Format check (CI-equivalent; use `cargo fmt` to fix)
cargo doc --no-deps                      # Build rustdoc

make build-image-x86                     # fmt + lint + test, then cargo-lambda + docker build (x86_64)
make build-image-arm64                   # same, arm64
```

The Makefile's `build-image-*` targets require `cargo-lambda` (`pip install cargo-lambda`), Docker, and the musl targets (`rustup target add {x86_64,aarch64}-unknown-linux-musl`). On macOS, also install `messense/homebrew-macos-cross-toolchains` — `.cargo/config.toml` already points at `*-unknown-linux-musl-gcc` linkers.

To produce a Lambda Layer artifact (CI path), use SAM:

```bash
sam build --template template-x86_64.yaml --parameter-overrides CargoPkgVersion=$(cargo metadata --no-deps --format-version=1 | jq -r '.packages[0].version')
```

Layer builds are dispatched through Makefile targets `build-LambdaAdapterLayerX86` / `build-LambdaAdapterLayerArm64`, which SAM invokes via `BuildMethod: makefile` in the templates.

## Architecture

### Process model

`src/main.rs` is the binary entry point. It does one critical thing **before** starting the Tokio runtime: calls `Adapter::apply_runtime_proxy_config()`. That function may set `AWS_LAMBDA_RUNTIME_API` via `std::env::set_var`, which is unsound once threads exist. Anything that calls `set_var` must run on the single pre-runtime thread.

Inside the Tokio runtime: build `AdapterOptions` from env, construct an `Adapter`, call `register_default_extension()` (POSTs to `/2020-01-01/extension/register`, then long-polls `/extension/event/next` to keep the extension slot alive), `check_init_health()` (loop GET on the readiness URL until healthy), then `run()` which dispatches into `lambda_http::run_concurrent` or `run_with_streaming_response_concurrent` based on `invoke_mode`.

### Module layout

- `src/lib.rs` — Single-file library. Public API: `Adapter`, `AdapterOptions`, `Protocol`, `LambdaInvokeMode`, plus re-exports `tracing` and `Error` from `lambda_http`. The `Adapter` implements `tower::Service<Request>` so the request transformation lives in `call()` / its helpers.
- `src/main.rs` — Binary glue (~30 lines). Don't add logic here; route through the library.
- `src/readiness.rs` — `Checkpoint` struct that paces "app not ready after Nms" log lines on an exponential schedule during the readiness wait.
- `tests/integ_tests/` — In-process tests that build an `Adapter` against an `httpmock::MockServer`. Many mutate process env vars and rely on **nextest's per-test process isolation** — they will race or fail under `cargo test`.
- `tests/e2e_tests/` — Tests marked `#[ignore]`; they hit a real deployed Lambda URL and require `API_ENDPOINT`, `API_AUTH_TYPE`, and AWS creds. Run explicitly with `cargo nextest run --run-ignored only`.
- `benches/e2e_body_forwarding.rs` — Criterion benchmark, registered in `Cargo.toml` under `[[bench]]`.
- `layer/bootstrap` — Two-line shell wrapper (`exec -- "${LAMBDA_TASK_ROOT}/${_HANDLER}"`) shipped inside the Layer; users set `AWS_LAMBDA_EXEC_WRAPPER=/opt/bootstrap` to invoke their handler script.

### Configuration surface

All knobs are environment-variable driven. Canonical names use the `AWS_LWA_` prefix (see `ENV_*` constants in `src/lib.rs`). Older non-prefixed names (`HOST`, `READINESS_CHECK_*`, `REMOVE_BASE_PATH`, `ASYNC_INIT`) are read via `get_env_with_deprecation` / `get_optional_env_with_deprecation` and emit a `tracing::warn!` — they will be removed in 2.0. **Exception:** `PORT` is *not* deprecated and remains a permanent fallback for `AWS_LWA_PORT`.

`AWS_LWA_READINESS_CHECK_MIN_UNHEALTHY_STATUS` was removed in 1.0; use `AWS_LWA_READINESS_CHECK_HEALTHY_STATUS` (comma-separated codes and ranges, parsed by `parse_status_codes`).

### Notable runtime behaviors

- **SnapStart**: when `AWS_LAMBDA_INITIALIZATION_TYPE=snap-start`, hyper's connection pool is disabled (`pool_max_idle_per_host(0)`). Reason: `CLOCK_MONOTONIC` can jump backwards on restore, so the pool may hand out dead connections (hyper#3810). Don't "fix" this back to a normal pool.
- **Compression × streaming**: `compression=true` plus `invoke_mode=ResponseStream` is silently downgraded to no-compression with a warning. The streaming path can't go through `tower_http::CompressionLayer` because it buffers.
- **Async init**: `async_init=true` caps `check_init_health` at 9.8s (Lambda's init phase is hard-killed at 10s). The `ready_at_init` flag tracks whether the readiness check passed, and the first request handler re-checks before forwarding if it didn't.

## Conventions

- Edition 2021. Rustfmt: `max_width=120`, `reorder_imports=true` (`.rustfmt.toml`).
- Commits follow Conventional Commits and are enforced by commitlint. Allowed types: `feat`, `fix`, `docs`, `example`/`examples`, `chore`, `refactor`, `perf`, `test`, `ci`, `revert`. Header max length 120.
- Changelog is generated by `git-cliff` from `cliff.toml`; the type → group mapping there is the source of truth for release notes.
- Tests that need to mutate env vars don't need a mutex — they rely on nextest spawning a fresh process per test. If you add such a test, do not switch to plain `cargo test` to verify it.
- The `examples/` tree is excluded from the published crate (`exclude = ["examples"]` in `Cargo.toml`) and skipped in PR CI (`paths-ignore`); changes there don't trigger the Rust test pipeline.
