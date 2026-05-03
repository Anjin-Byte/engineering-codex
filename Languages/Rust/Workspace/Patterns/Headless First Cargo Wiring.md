---
title: Headless First Cargo Wiring
tags: [pattern, cargo, architecture]
summary: How to express the headless-first principle in a Cargo workspace — separate binary crates for UI versus headless, or feature-gated optional UI.
keywords: [no gui, ci friendly, batch processing, library first, cargo features, binary crate, workspace topology]
---

*The Cargo expression of headless-first: the headless binary crate has zero UI dependencies, and the UI is either a sibling binary crate or a non-default Cargo feature.*

# Headless First Cargo Wiring

The universal principle lives at [[Engineering Philosophy/Principles/Headless First]]. This note covers how a Cargo workspace expresses it.

## Two Cargo expressions

### Sibling binary crates (preferred)

The headless binary and the interactive front-end are two separate binary crates in the workspace. Both depend on the same core and adapter crates. Neither binary contains domain logic.

```
crates/
  core/           # pure domain logic; no UI deps
  adapters/...    # external-system adapters
  cli/            # bin: headless executable
  viewer/         # bin: optional interactive front-end
```

Server / CI users build only `cli`. Local users who want interactivity build `viewer`. The dependency graphs do not overlap.

### Feature-gated optional UI

A single binary crate where UI is a non-default Cargo feature:

```toml
# crates/cli/Cargo.toml
[features]
default = []
ui = ["dep:viewer"]

[dependencies]
viewer = { path = "../viewer", optional = true }
```

```rust
#[cfg(feature = "ui")]
mod ui_mode;
```

This is acceptable when the UI dependency is light or the audiences overlap. For substantial UI dependencies, sibling binary crates age better — see [[Languages/Rust/Workspace/Patterns/Cargo Binary Strategy]].

## Concrete invariants

- The `core` crate's `Cargo.toml` lists no UI / windowing dependency at any feature level.
- Adapter crates that wrap external systems list no UI dependency at any feature level.
- The headless binary crate's default-feature dependency closure has no UI / windowing crate in it (`cargo tree --no-default-features` should confirm).
- Profilers, telemetry exporters, debug overlays, and hot-reload watchers each live behind their own feature flag — see [[Languages/Rust/Workspace/Patterns/Cargo Feature Gating]].

## Related

- [[Engineering Philosophy/Principles/Headless First]]
- [[Languages/Rust/Workspace/Core Principles]]
- [[Languages/Rust/Workspace/Patterns/Cargo Binary Strategy]]
- [[Languages/Rust/Workspace/Patterns/Cargo Feature Gating]]
