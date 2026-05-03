---
title: Cargo Feature Gating
tags: [pattern, cargo, configuration]
summary: How to use Cargo features to express capability slices — additive, well-named, defaulted carefully, and never used for internal implementation switches.
keywords: [conditional compilation, optional dependencies, feature flags, additive features, default features, cross-crate features]
---

*The Cargo expression of capability-slices-vs-implementation-switches: features are additive public capabilities; internal toggles become conditional code or separate modules.*

# Cargo Feature Gating

The universal principle lives at [[Engineering Philosophy/Principles/Capability Slices vs Implementation Switches]]. This note covers Cargo-specific mechanics.

## Additive features

Cargo features are **additive** by design — enabling a feature must not break other consumers' builds. This rules out using features as alternatives:

- ❌ `runtime-tokio` vs `runtime-async-std` as mutually-exclusive features (a transitive consumer enabling both breaks compilation)
- ✓ separate sibling crates if the runtimes are mutually exclusive
- ✓ a single feature `tokio` that pulls in tokio when on, with a stub or alternate path when off

If you need alternatives, you need separate crates, not features.

## Default features

Default to **off** for anything that:
- adds heavy dependencies (UI, GUI, telemetry runtimes)
- changes runtime cost meaningfully
- is only useful in a specific environment (dev, CI, profiling)

Default to **on** for things that are part of the normal expected experience.

A common shape:

```toml
[features]
default = []
ui = ["dep:viewer"]
trace = ["dep:tracing", "dep:tracing-subscriber"]
profiling = ["dep:tracy-client"]
```

`default = []` (rather than omitting it) is explicit — the empty default is a deliberate choice.

## Cross-crate feature plumbing

Feature in a binary crate that turns on a UI crate:

```toml
# crates/cli/Cargo.toml
[features]
default = []
ui = ["dep:ui"]

[dependencies]
ui = { path = "../ui", optional = true }
```

In `cli` source:

```rust
#[cfg(feature = "ui")]
mod ui_mode;
```

When you need a feature in crate A to enable a feature in dependency B:

```toml
[features]
trace = ["tracing", "core/trace"]    # also enables `trace` in the core crate
```

## Anti-pattern: testing every feature combination

If `N` features compose freely, you have `2^N` build configurations. Keep features orthogonal, well-scoped, and few. CI should test the meaningful combinations (default; default + ui; minimal; full), not the cartesian product.

For comprehensive coverage, `cargo hack --feature-powerset` exists — use it sparingly.

## When features are not the right tool

If you're considering a Cargo feature, first check:

- Is this an *internal* toggle (CPU vs GPU, fast-path vs safe-path)? → Use conditional code or a sibling module, not a feature.
- Are the alternatives mutually exclusive? → Separate crates, not features.
- Is this just a developer convenience? → Often a `[dev-dependencies]` block or a separate test binary is cleaner than a feature.

The bar for adding a feature is: *would a downstream consumer or operator ever have reason to consciously flip this?* If no, don't expose it.

## Related

- [[Engineering Philosophy/Principles/Capability Slices vs Implementation Switches]]
- [[Languages/Rust/Workspace/Cargo Workspace Configuration]]
- [[Languages/Rust/Workspace/Patterns/Cargo Binary Strategy]]
- [[Languages/Rust/Workspace/Patterns/Headless First Cargo Wiring]]
