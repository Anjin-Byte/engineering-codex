---
title: Feature Gating
tags: [pattern, cargo, configuration]
summary: Cargo features should represent real capability slices a downstream user would toggle, not internal implementation switches.
keywords: [conditional compilation, optional dependencies, feature flags, additive features, public api surface, build configuration]
---

*Features are a public capability vocabulary; if no consumer would ever want to flip the flag, it does not belong in features.*

# Feature Gating

> **Rule:** Cargo features represent **real capability slices**, not random internal toggles.

## Good features

- `ui` — pulls in interactive front-end and any UI deps
- `trace` — wires up `tracing` exporters / spans for production diagnostics
- `profiling` — enables backend profilers and timestamps
- `hot-reload` — dev-only watcher that rebuilds artifacts on change

Each one corresponds to a thing a user or developer would consciously turn on.

## Bad features (avoid)

- `use-fast-path` — internal toggles that should just be conditional code
- `experimental` — too vague; what does enabling it actually do?
- `legacy` — usually a sign the old code should just be deleted
- one feature per minor switch — feature soup that nobody understands

## Default features

Default to **off** for anything that:
- adds heavy dependencies (UI, GUI, telemetry runtimes)
- changes runtime cost meaningfully
- is only useful in a specific environment (dev, CI, profiling)

Default to **on** for things that are part of the normal expected experience.

## Cross-crate feature plumbing

Feature in a binary crate that turns on a UI:

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

If you go with [[Binary Strategy]] Option B, you may not need a feature flag at all — the front-end is just a separate binary.

## Anti-pattern: testing every feature combination

If `N` features compose freely, you have `2^N` build configurations. Keep features orthogonal, well-scoped, and few. CI should test the meaningful combinations, not the cartesian product.

## Related

- [[Cargo Workspace Configuration]]
- [[Binary Strategy]]
- [[Headless First]]
