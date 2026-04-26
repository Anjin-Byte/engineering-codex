---
title: Shared Dependencies
tags: [pattern, cargo, dependencies]
summary: Hoist dependencies into workspace.dependencies only when truly shared; keep crate-local deps local to preserve crate independence.
keywords: [version pinning, dependency management, transitive dependencies, monorepo deps, semver alignment, third party libraries]
---

*workspace.dependencies is for genuinely shared third-party crates; using it as a default registry blurs crate boundaries.*

# Shared Dependencies

What belongs in `[workspace.dependencies]` — and what doesn't.

## Belongs at workspace level

Pin these once in the root `Cargo.toml` so all crates use the same version:

- `serde` (with `derive`)
- `tracing`, `tracing-subscriber`
- `anyhow`, `thiserror`
- `clap` (with `derive`)
- Any heavy adapter library used in more than one crate (a database driver, an HTTP client, a GPU library, a parser framework)
- Any async runtime chosen for the project

A version drift between crates on a heavy dependency would be painful — these often ship breaking changes meaningfully often.

## Member crate inheritance

```toml
# crates/some-adapter/Cargo.toml
[dependencies]
serde = { workspace = true, features = ["derive"] }
```

Note: features can be **added** at the member level on top of the workspace baseline.

## Does NOT belong everywhere

Just because something is in `[workspace.dependencies]` does not mean every crate should pull it in.

Critical examples:
- Pure core crates should **not** depend on heavy adapter libraries. That's the whole point of narrow boundaries.
- The CLI crate should **not** depend on UI / windowing libraries. That's the UI crate's job.
- Build-automation crates should **not** depend on heavy runtime libraries unless a specific automation task genuinely needs them.

## Why discipline here matters

Each unnecessary dependency:
- slows clean builds
- expands the attack/maintenance surface
- couples crates that should evolve independently
- makes "what does this crate actually do" harder to read at a glance

A dependency that doesn't appear in a crate's `Cargo.toml` is one you cannot accidentally use.

## Related

- [[Cargo Workspace Configuration]]
- [[Core Principles]]
