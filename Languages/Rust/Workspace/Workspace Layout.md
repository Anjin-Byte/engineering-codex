---
title: Workspace Layout
tags: [architecture, layout]
summary: Recommended directory layout for a non-trivial Rust workspace, separating core, adapters, binaries, and reference implementations.
keywords: [folder structure, repository layout, multi crate organization, project skeleton, file tree, module placement]
---

*Lay the workspace out so the file tree itself shows where responsibilities live and which crates depend on which.*

# Workspace Layout

A recommended directory shape for a non-trivial Rust workspace.

## Generic full layout

```
project_root/
  Cargo.toml              # virtual workspace manifest
  .cargo/
    config.toml
  crates/
    core/                 # pure domain logic (no external effects)
      Cargo.toml
      src/
    adapter-x/            # adapter wrapping an external system / library
      Cargo.toml
      src/
    cli/                  # argument parsing, file I/O, orchestration
      Cargo.toml
      src/
    ui/                   # OPTIONAL: interactive front-end
      Cargo.toml
      src/
  tests/                  # cross-crate integration tests
  reference/              # primary-source reference documents
  tools/                  # build / automation tasks
    automation/
      Cargo.toml
      src/
```

The names are placeholders — pick names that reflect responsibility in your domain. The shape matters more than the labels.

## Why this shape

- **Virtual workspace root** — see [[Cargo Workspace Configuration]] for why and how.
- **`crates/*` glob** — every member sits in one place; trivial to add or move.
- **Adapter crates per external system** — each external boundary (a device API, a network protocol, a heavy third-party library) gets its own crate so its types do not leak into core logic.
- **Build automation as a sibling to `crates/`** — it is build infrastructure, not a library, but it is a Cargo workspace member and benefits from shared lints/deps.
- **`tests/` at root** — for cross-crate integration tests; per-crate unit tests live in their own `tests/` or inline `#[cfg(test)]`.
- **`reference/` at root** — primary-source PDFs, specifications, datasets. Not a crate.

## Phasing the workspace

A useful pattern: start with the pure core, add adapters and binaries as their need is justified.

### Phase 1 — pure core

```
project_root
├── crates/core
└── tools/automation
```

Establish the domain model, types, and reference algorithms with no external dependencies. The core can be tested anywhere, and serves as the oracle for any future optimized implementation. See [[Push Correctness Left]] and [[Engineering Philosophy/Principles/Reference Implementation as Oracle]].

### Phase 2 — add adapters and a binary

```
project_root
├── crates/core
├── crates/adapter-x        ← optimized / external-system path
├── crates/cli              ← orchestration, I/O
└── tools/automation
```

At this point the system is shippable as a binary.

### Phase 3 — optional interactive front-end

Add a UI crate only when interactive use is actually needed. See [[Headless First]].

## Related

- [[Core Principles]]
- [[Languages/Rust/Workspace/Patterns/Cargo Binary Strategy]]
- [[Cargo Workspace Configuration]]
