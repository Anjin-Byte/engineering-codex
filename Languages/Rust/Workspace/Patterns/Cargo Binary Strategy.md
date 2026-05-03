---
title: Cargo Binary Strategy
tags: [pattern, cargo, deployment]
summary: How to express the one-binary-or-many decision in a Cargo workspace — separate binary crates versus a single binary crate with feature-gated modes.
keywords: [bin table, binary crate, feature-gated binary, workspace member, multi binary, subcommand]
---

*The Cargo expression of one-binary-or-many: prefer separate binary crates as workspace members when surfaces diverge; reach for feature-gated binaries only when the deployment story is genuinely shared.*

# Cargo Binary Strategy

The universal principle lives at [[Engineering Philosophy/Principles/One Binary or Many]]. This note covers how a Cargo workspace expresses the choice.

## Separate binary crates (preferred when surfaces diverge)

Each binary is its own workspace member:

```
crates/
  core/
  cli/        # bin: headless executable
  viewer/     # bin: optional interactive front-end
  service/    # bin: long-running server
```

Each binary crate has its own `Cargo.toml`, its own dependency set, its own `main.rs`. The workspace member layout is the deployable-unit layout from [[Engineering Philosophy/Principles/Architectural Core Principles]] rule 3.

`cargo build -p cli` builds exactly the headless executable, with exactly its dependency closure. No UI crate gets dragged in.

## Feature-gated binary (use sparingly)

A single binary crate that turns capabilities on via Cargo features:

```toml
# crates/myapp/Cargo.toml
[features]
default = []
ui = ["dep:viewer"]

[dependencies]
viewer = { path = "../viewer", optional = true }
```

Acceptable when:
- The audiences overlap heavily.
- The optional dependency is light.
- A single distributable simplifies installation.

Becomes painful when:
- The optional dependency is heavy (UI runtimes, telemetry stacks).
- Different audiences want different default builds.
- The binary's dependency closure starts being asymmetric across feature combinations.

When the pain shows up, split into separate binary crates. See [[Languages/Rust/Workspace/Patterns/Cargo Feature Gating]] for the decision rule on whether a Cargo feature is even the right mechanism.

## What both options share

- The `core` crate and adapter crates are workspace members consumed by every binary.
- Each binary's `main.rs` is thin: argument parsing, configuration, orchestration. No domain logic.
- `[workspace]` configuration (lints, profiles, shared dependency hoisting) lives at the root manifest — see [[Languages/Rust/Workspace/Cargo Workspace Configuration]] and [[Languages/Rust/Workspace/Patterns/Workspace Lints and Profiles]].

## Related

- [[Engineering Philosophy/Principles/One Binary or Many]]
- [[Engineering Philosophy/Principles/Headless First]]
- [[Languages/Rust/Workspace/Core Principles]]
- [[Languages/Rust/Workspace/Patterns/Headless First Cargo Wiring]]
- [[Languages/Rust/Workspace/Patterns/Cargo Feature Gating]]
