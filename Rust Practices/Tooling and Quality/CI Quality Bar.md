---
title: CI Quality Bar
tags: [rust, ci, tooling, quality]
summary: CI enforces format, lint, build, test, and doc-test on every change; checks here are the floor below which work cannot ship.
keywords: [continuous integration, github actions, pipeline checks, pre merge gates, automation, fmt check, cargo check]
---

*CI is the project's quality bar made executable; whatever you do not check here will eventually leave a developer's machine broken.*

# CI Quality Bar

CI should reflect the quality bar of the project. Every check below stops bugs from leaving a developer's machine.

## The baseline checks

```sh
# Formatting — no debate, no diffs
cargo fmt --all -- --check

# Lints — see [[Clippy as Discipline]]
cargo clippy --workspace --all-targets --all-features -- -D warnings

# Tests — unit + integration + doc
cargo test --workspace --all-targets
cargo test --workspace --doc
```

Each is fast on its own and parallelizable.

## Why each check exists

- **`cargo fmt --check`** — formatting drift in PRs is noise. Settling it once via `rustfmt` removes a category of debate forever.
- **`cargo clippy -D warnings`** — turns lint warnings into hard failures. Without `-D warnings`, "I'll fix it later" wins. With it, the warning blocks merge.
- **`cargo test --all-targets`** — runs unit tests, integration tests, and tests inside examples/benches. Without `--all-targets`, examples can rot.
- **`cargo test --doc`** — runs doc tests. The default `cargo test` runs them, but explicit invocation makes it visible in CI logs and lets it be split out for parallelism.

## Splitting for speed

For larger workspaces, a useful split:

| Job | Purpose |
|---|---|
| `fmt` | `cargo fmt --check` (~seconds) |
| `clippy` | `cargo clippy -D warnings` (cached, ~minutes first time) |
| `test-unit` | `cargo test --workspace --lib --bins` |
| `test-integration` | `cargo test --workspace --test '*'` |
| `test-doc` | `cargo test --workspace --doc` |

Run them in parallel. The slowest job sets the wall-clock time for the PR.

## What to add later, when it pays off

- **`cargo deny`** — supply-chain hygiene (license, advisory, duplicate-version checks).
- **`cargo audit`** — known-vulnerability scan against `RustSec`.
- **`cargo machete`** or **`cargo udeps`** — find unused dependencies.
- **`cargo miri test`** — undefined-behavior detection for code with `unsafe`. See [[Unsafe Quarantine]].
- **Smoke tests on real environments** — for adapter crates that wrap external systems, a CI runner with the real environment (or a software fallback) that exercises a minimal end-to-end flow.

These are valuable but **add them when there's a payoff**, not because a checklist says to.

## What CI is NOT

- A substitute for local discipline. Developers should run `cargo fmt && cargo clippy && cargo test` locally before pushing. CI is the *backstop*.
- A place to put builds that "might fail flakily." Flakiness erodes trust in CI; investigate root causes.
- A replacement for code review. CI catches mechanical issues; humans catch design issues.

## Failure policy

- A red CI blocks merge. No exceptions for "it's just a warning."
- Flakiness is a bug filed against CI itself, not "rerun until green."
- New lints land as `warn`, get cleaned up across the codebase, then promoted to `deny`.

## Related

- [[Clippy as Discipline]]
- [[Three Levels of Tests]]
- [[Workspace Lints and Profiles]]
- [[Documentation as Truth]]
