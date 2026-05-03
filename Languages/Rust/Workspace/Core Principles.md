---
title: Core Principles
tags: [architecture, principles, cargo]
summary: How the four universal architectural principles map to a Cargo workspace — crate-shaped boundaries, headless binary crate, deployable-unit crate split, core/optimized adapter twin.
keywords: [hexagonal architecture, ports and adapters, separation of concerns, crate boundaries, workspace topology, design smells]
---

*The four universal architectural principles, applied to Cargo workspace topology.*

# Core Principles (Cargo Application)

The four principles that drive project shape are universal; they live at [[Engineering Philosophy/Principles/Architectural Core Principles]]. This note covers how a Cargo workspace expresses them. Read the universal note first.

## How each principle maps to Cargo

### 1. Narrow external-system boundaries → core / adapter / cli / ui crate split

- **`core` crate(s)** — domain types, algorithms, deterministic transformations. No external-system dependencies, no UI, no I/O.
- **Adapter crates** — one per external system (a device API, a windowing library, a heavy SDK), with its own narrow public API.
- **`cli` crate** — argument parsing, file I/O, logging, exit codes. Orchestrates; contains no domain logic.
- **`ui` crate** (optional) — interactive front-end, depends on `core` but not vice versa.

The dependency graph runs *inward*: `cli` and `ui` depend on `core` and adapters; `core` depends on nothing project-specific.

### 2. Headless default → headless binary crate has no UI dependency

The headless binary crate's `Cargo.toml` lists no UI / windowing dependency at any feature level. Verify with `cargo tree --no-default-features`. See [[Languages/Rust/Workspace/Patterns/Headless First Cargo Wiring]] for the two Cargo expressions (separate binary crates vs feature-gated UI).

### 3. Workspace reflects deployable units → crate-per-deployable-unit

If something can be built, tested, or shipped independently, it deserves its own crate. The workspace member list is the deployable-unit list.

Five meaningful crates beats twelve decorative ones. Conversely, a single monolith crate that contains parts with genuinely different lifecycles (a CLI binary intermixed with library code) eventually needs splitting.

### 4. Reference path alongside optimized → core crate vs optimized adapter crate

For any optimized algorithm (SIMD, GPU, FFI, hand-rolled), the canonical implementation lives in the core crate; the accelerated implementation lives in an adapter crate (e.g. `accel-gpu`, `accel-simd`). Tests cross-validate. See [[Languages/Rust/Workspace/Patterns/Reference Path Cargo Wiring]].

## Workspace shape that satisfies all four

```
crates/
  core/                # rule 1, 4 — pure domain, canonical reference
  adapters/
    storage/           # rule 1 — narrow external-system wrapper
    accel-gpu/         # rule 4 — optimized adapter for the reference in core
  cli/                 # rule 2 — headless binary, zero UI deps
  viewer/              # rule 2 — optional UI as separate binary crate
```

Five crates, four principles satisfied. Each crate's reason for existing is one of the four rules.

## Related

- [[Engineering Philosophy/Principles/Architectural Core Principles]]
- [[Languages/Rust/Workspace/Workspace Layout]]
- [[Languages/Rust/Workspace/Patterns/Headless First Cargo Wiring]]
- [[Languages/Rust/Workspace/Patterns/Reference Path Cargo Wiring]]
- [[Languages/Rust/Workspace/Patterns/Cargo Binary Strategy]]
