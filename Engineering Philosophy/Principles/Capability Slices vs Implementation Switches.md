---
title: Capability Slices vs Implementation Switches
tags: [principle, architecture, configuration]
summary: Build-time configuration flags should represent real capability slices a downstream user would toggle, not internal implementation switches.
keywords: [conditional compilation, optional dependencies, feature flags, additive features, public api surface, build configuration]
---

*Configuration flags are a public capability vocabulary; if no consumer would ever want to flip the flag, it does not belong as a flag.*

# Capability Slices vs Implementation Switches

> **Rule:** Build-time configuration flags (Cargo features, npm exports, build constants, etc.) represent **real capability slices**, not random internal toggles.

A flag is a public-facing dial. If no user, operator, or developer ever has reason to consciously turn it on, it should not exist as a flag — it should be conditional code, a separate module, or simply removed.

## Good flags

- `ui` — pulls in interactive front-end and any UI dependencies
- `trace` — wires up tracing exporters / spans for production diagnostics
- `profiling` — enables backend profilers and timestamps
- `hot-reload` — dev-only watcher that rebuilds artifacts on change

Each one corresponds to a thing a user or developer would consciously turn on.

## Bad flags (avoid)

- `use-fast-path` — internal toggles that should just be conditional code
- `experimental` — too vague; what does enabling it actually do?
- `legacy` — usually a sign the old code should just be deleted
- one flag per minor switch — flag soup that nobody understands

## Default flag state

Default to **off** for anything that:
- adds heavy dependencies (UI, GUI, telemetry runtimes)
- changes runtime cost meaningfully
- is only useful in a specific environment (dev, CI, profiling)

Default to **on** for things that are part of the normal expected experience.

## Anti-pattern: testing every flag combination

If `N` flags compose freely, you have `2^N` build configurations. Keep flags orthogonal, well-scoped, and few. CI should test the meaningful combinations, not the cartesian product.

## The deciding question

> **Would a downstream consumer or operator ever have reason to flip this flag, with a clear understanding of what changes?**

If yes — it's a capability slice; expose it.
If no — it's an implementation detail; bury it. The flag surface is part of the public API of the project; treat it with the same restraint.

## Related

- [[Engineering Philosophy/Principles/One Binary or Many]]
- [[Engineering Philosophy/Principles/Headless First]]
- [[Engineering Philosophy/Principles/Architectural Core Principles]]
- [[Languages/Rust/Workspace/Patterns/Cargo Feature Gating]]
