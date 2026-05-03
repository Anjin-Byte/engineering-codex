---
title: Cargo Workspace Configuration
tags: [architecture, cargo, configuration]
summary: Use a virtual workspace root that owns members, shared dependencies, lints, and profiles but ships no package of its own.
keywords: [virtual manifest, monorepo, multi crate, workspace inheritance, dependency hoisting, toml setup, root manifest]
---

*The root Cargo.toml is an orchestrator, not a package — versions, lints, and profiles centralize there so members stay uniform.*

# Cargo Workspace Configuration

Use a **virtual workspace** root. The root `Cargo.toml` carries no `[package]` of its own — it only orchestrates members, shared dependencies, lints, and profiles.

## Root `Cargo.toml`

```toml
[workspace]
members = ["crates/*", "tools/*"]
default-members = ["crates/cli"]
resolver = "3"

[workspace.package]
edition = "2024"
version = "0.1.0"
license = "MIT OR Apache-2.0"
rust-version = "1.85"

[workspace.dependencies]
anyhow = "1"
thiserror = "2"
tracing = "0.1"
tracing-subscriber = "0.3"
serde = { version = "1", features = ["derive"] }
clap = { version = "4", features = ["derive"] }
```

## Why each piece matters

### `resolver = "3"`
Cargo requires virtual workspaces to set the resolver explicitly — it does not inherit a default from any member.

### `default-members = ["crates/cli"]`
Without this, root-level Cargo commands operate on **all** members in a virtual workspace. That is noisy and slow. Pinning a primary binary as the default makes `cargo run`, `cargo build`, `cargo test` at the root behave the way humans expect.

### `[workspace.package]`
Lets member crates inherit `edition`, `version`, `license`, and `rust-version` via `edition.workspace = true` etc. Avoids drift between crates.

### `[workspace.dependencies]`
The right place to pin shared versions for libraries used across crates (serde, clap, error/log libraries, any heavy adapter library). Member crates inherit with:

```toml
[dependencies]
serde = { workspace = true, features = ["derive"] }
```

But: see [[Shared Dependencies]] — **don't** make every crate depend on everything just because it's listed. Pure core crates should stay free of heavy adapter dependencies.

## Profiles

Profiles are **only** recognized in the root manifest, not in member manifests. Always configure them centrally:

```toml
[profile.dev]
opt-level = 1
debug = 2

[profile.release]
lto = "thin"
codegen-units = 1
debug = 1
```

Reasoning:
- `dev` with `opt-level = 0` makes timing and behavior feel misleading on math-heavy code paths.
- `release` should still preserve enough symbols (`debug = 1`) to debug crashes and profile sanely.

See also: [[Workspace Lints and Profiles]]

## Lints

Lint configuration is also centralized — see [[Workspace Lints and Profiles]].

## Related

- [[Workspace Layout]]
- [[Shared Dependencies]]
- [[Languages/Rust/Workspace/Patterns/Cargo Feature Gating]]
