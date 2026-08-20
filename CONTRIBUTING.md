# Contributing to LoonFS

While the project is in early development, we are only accepting contributions from core
maintainers.

This guide covers workflow: building, testing, spec-locked artifacts, and git conventions.
`STYLE.md` covers how code is written. `docs/specs/` is normative for durable formats and the API —
when code and spec disagree, one of them is wrong and the PR fixes both together.

## Toolchain and first build

`rust-toolchain.toml` pins stable with `rustfmt` and `clippy`; MSRV is the workspace
`rust-version` (1.83). No other system dependencies are required — the local-fs provider backs all
default tests.

```bash
cargo build -p loonfs-server        # build the server binary first —
cargo test --all                    # the CLI suite drives target/debug/loonfs-server
cargo fmt --all --check
cargo clippy --all-targets --all-features -- -D warnings
```

## Crate map

Dependency direction is strictly downward; a type lives in the lowest crate whose vocabulary it
belongs to (see `STYLE.md` → Crates and modules).

| Crate | Role | Spec-locked? |
| --- | --- | --- |
| `loonfs-api` | Ids, paths, wire shapes, durable-format codecs | Yes — golden fixtures + `format.md`/`api.md` |
| `loonfs-objectstore` | `ObjectStore` contract, key layout, providers, `StoreConfig` | Yes — key table pinned to `format.md` |
| `loonfs-core` | Protocol engine: metadata, commits, replay, maintenance | Via api's formats |
| `loonfs` | Runtime: `FsWriter`/`FsReader`/`FsAdmin`, caching, background work, publisher | No |
| `loonfs-server` | Reference HTTP server (v0 API) | Yes — `openapi.json` + `api.md` tables |
| `loonfs-client` | HTTP client for a LoonFS server | No |
| `loonfs-cli` | `loon` binary; embedded and remote profiles | CLI JSON output snapshot-pinned |
| `loonfs-model` | Independent reference oracle for differential tests (`publish = false`) | — |
| `loonfs-sim` | Harness contract for the out-of-repo simulator (`publish = false`) | — |

Two hard boundaries:

- `loonfs-server` never depends on `loonfs-core` in production code — everything the server needs
  is (re-)exported through `loonfs`. If a core type is missing from the seam, widen the seam.
- The deterministic simulator and the benchmark harness live in a separate private repo as a
  deliberate structure — this repo carries only the hooks they consume (`loonfs-sim`, core's
  `inspection` feature, injection seams such as `MonotonicTimer`). Keep the hooks coherent and
  update them alongside repo refactors; breaking changes are fine — the external driver follows
  this repo, not the reverse, and is amended afterward. Do not delete the hooks or grow
  compatibility fallbacks for their sake.

## Running tests

`cargo test --all` runs everything hermetic (~600+ tests). Details by tier (full table in
`STYLE.md` → Testing):

- **Golden wire fixtures** (`crates/loonfs-api/tests/golden/`): fail on byte divergence and print
  the regen command. `UPDATE_GOLDEN=1 cargo test -p loonfs-api` regenerates; updating fixtures is
  **expected** for deliberate format changes and must be called out in the PR description.
- **OpenAPI**: `openapi_static_file_is_current` fails when handlers drift from
  `docs/specs/openapi.json` and prints the regen command
  (`cargo run -p loonfs-server --features openapi --bin loonfs-openapi -- docs/specs/openapi.json`).
- **Spec tables**: the `api.md` error-status table and the `format.md` key table are test-enforced
  (`error_status_mapping_matches_the_api_spec_table`,
  `standard_key_patterns_match_format_spec_table`). Change doc and code together.
- **insta snapshots**: review with `cargo insta review`; never hand-edit `.snap` files.
- **CLI suite** (`loonfs-cli/tests/cli.rs`): drives the real `loon` and `loonfs-server` binaries —
  build `loonfs-server` before running it in isolation.
- **Live provider conformance** (`loonfs-objectstore/tests/`, `direct_put_real_provider`): skipped
  unless `LOONFS_TEST_{S3,R2,GCS,ABS}_*` credentials are set; see
  `crates/loonfs-objectstore/tests/provider-conformance.env.example`. These are not CI gates —
  run them when touching provider code.
- **Differential suite** (`loonfs-core/tests/it/differential.rs`): replays scenarios through core and
  the `loonfs-model` oracle and compares. A visibility-semantics change that doesn't touch both is
  suspicious.

Never hand-edit a spec-locked artifact to silence a test — regenerate it through the printed
command, or the change is wrong.

## Changing a wire or durable format

Backwards compatibility is currently a non-goal (no users, no deployments) — break formats the
clean way rather than the compatible way. The mechanical checklist:

1. Change the type in `loonfs-api` (tag `kind`, snake_case, `_id` suffixes, tolerant decoding —
   see `STYLE.md` → Serde).
2. Non-additive payload change? Bump that family's `format_version` — and only that family's.
3. `UPDATE_GOLDEN=1 cargo test -p loonfs-api`; commit the regenerated fixtures.
4. Update the `format.md` / `api.md` tables and examples the sync tests point at.
5. Regenerate `openapi.json` if HTTP shapes changed.
6. If metadata semantics changed, mirror it in `loonfs-model` (independently — no shared code).
7. Sweep every consumer (core, server, client, CLI render) — exhaustive matches will tell you.
8. Say "wire format change; goldens regenerated" in the PR description.

## Determinism rules

`clippy.toml` bans ambient `SystemTime::now`, `Instant::now`, `thread::sleep`,
`tokio::time::sleep`, and `rand::random`. New time or randomness enters through a named boundary
function with a function-scoped `#[allow(clippy::disallowed_methods)]` and a reason comment —
never file-scoped, in prod or tests. CLI output goes through `crates/loonfs-cli/src/render.rs`
(that is how the `print_stdout` lint stays satisfied); no bare `println!`.

## Git conventions

- **Branches**: `<author>/<kebab-description>` (e.g. `conor/error-style-sweep`). Never include
  "claude", "ai", or any agent identifier in branch names, commits, or PRs.
- **Commits**: one line, `type(scope): lowercase summary`. Types in use: `feat`, `fix`, `refactor`,
  `test`, `docs`, `ci`, `chore`. Scopes are crate short-names or subsystems (`api`, `core`, `cli`,
  `server`, `client`, `objectstore`, `runtime`, `wal`, `manifest`, `storage`, `specs`); omit the
  scope for workspace-wide sweeps. No bodies, no bullet changelogs.
- **No AI attribution** — no `Co-Authored-By: Claude`, no "Generated with" trailers, anywhere.
  Check `git log -1` before pushing; amend if tooling injected anything.
- **PRs**: 1–4 sentences of rationale-first prose (why the change exists). No section templates,
  no diff restatement. Base on `main`; if stacking is unavoidable, name the base branch in the
  description. PRs merge by squash, so the PR title follows the commit convention.

## Definition of done

Every PR, before review:

1. `cargo fmt --all --check`
2. `cargo clippy --all-targets --all-features -- -D warnings`
3. `cargo test --all` (build `loonfs-server` first) and, when server shapes changed,
   `cargo test -p loonfs-server --features openapi`
4. Spec-locked artifacts regenerated deliberately (goldens, `openapi.json`, spec tables, insta) —
   with the change acknowledged in the PR description.
5. If the PR changes a convention, sweep it to 100% — grep for stragglers before finishing.
   Half-migrations are worse than no migration.

## Where decisions live

- `docs/specs/` — the normative durable format and API (start with its `README.md` reading guide).
- `STYLE.md` — code conventions and their rationale.
- `docs/specs/glossary.md` — the domain vocabulary; use its terms exactly.
- `AUDIT_PR_PLANS.md` — the pre-release cleanup ledger (executed round-1 plans and the deferred
  round-2 queue); useful history for why a canon exists.
