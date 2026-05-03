---
title: One Binary or Many
tags: [principle, architecture, deployment]
summary: Decide between one executable with subcommands or modes versus several based on whether the surfaces share lifecycle, dependencies, and audience.
keywords: [executable layout, multi binary, monolithic cli, subcommand dispatch, packaging, distribution targets, entry points]
---

*Pick the executable count by responsibility shape: one when surfaces share lifecycle, several when they diverge in dependencies or audience.*

# One Binary or Many

How to ship the application: as one executable, or several?

## Option A — single executable with feature flags or subcommands

```
myapp           # supports --ui or similar opt-in modes, or `myapp serve` / `myapp tool`
```

**Good when:**
- The optional mode is a convenience the same user occasionally wants.
- A single distributable simplifies install and documentation.
- The dependency cost of dragging optional libraries into headless invocations is acceptable.

## Option B — separate executables

```
myapp-cli       # headless, lean, no UI dependency
myapp-viewer    # interactive front-end
```

**Good when:**
- Headless builds (server, CI) should have **zero** UI / heavy-runtime dependencies.
- Deployment for batch / server use should be lean.
- The interactive front-end grows richer (panels, inspectors, tools) and benefits from independent evolution.
- The audiences are different — operators run the headless build, end users run the viewer.

For most non-trivial systems, Option B ages better. Server and CI use should not require UI libraries to be present.

## Default recommendation

Start **headless-only** with no UI dependency. When/if interactivity becomes useful, prefer **Option B** — a separate executable that depends on the same core and adapter modules as the headless one.

## What both options share

- Core and domain-logic modules are consumed by both.
- Adapter modules that wrap external systems are consumed by both.
- Neither executable contains domain logic itself — both orchestrate.

## The deciding question

> **Do the surfaces share lifecycle and dependencies, or do they diverge?**

If shared — one binary with feature flags or subcommands is fine; users get a single install. If divergent — split, because one will start carrying the other's dependency cost.

## Related

- [[Engineering Philosophy/Principles/Headless First]]
- [[Engineering Philosophy/Principles/Capability Slices vs Implementation Switches]]
- [[Engineering Philosophy/Principles/Architectural Core Principles]]
- [[Languages/Rust/Workspace/Patterns/Cargo Binary Strategy]]
