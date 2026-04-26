---
title: Binary Strategy
tags: [pattern, architecture, deployment]
summary: Decide between one binary with subcommands and several binaries based on whether the surfaces share lifecycle and dependencies.
keywords: [executable layout, multi binary crate, monolithic cli, subcommand dispatch, packaging, distribution targets, entry points]
---

*Pick the binary count by responsibility shape: one binary when surfaces share lifecycle, several when they diverge in deps or audience.*

# Binary Strategy

How to ship the application: one binary, or several?

## Option A — single binary with feature flags

```
myapp           # supports --ui or similar opt-in modes
```

**Good when:**
- The optional mode is a convenience the same user occasionally wants.
- A single distributable simplifies install/docs.
- The dependency cost of dragging optional libraries into headless invocations is acceptable.

**Implementation:** the binary crate gains a Cargo feature that pulls in the optional adapter or UI crate. See [[Feature Gating]].

## Option B — separate binaries

```
myapp-cli       # headless, lean, no UI dependency
myapp-viewer    # interactive front-end
```

**Good when:**
- Headless builds (server, CI) should have **zero** UI / heavy-runtime dependencies.
- Deployment for batch / server use should be lean.
- The interactive front-end grows richer (panels, inspectors, tools) and benefits from independent evolution.

For most non-trivial systems, Option B ages better. Server and CI use should not require UI libraries to be present.

## Default recommendation

Start **headless-only** with no UI crate. When/if interactivity becomes useful, prefer **Option B** — a separate binary in its own crate that depends on the same core and adapter crates as the headless binary.

## What both options share

- Core and domain-logic crates are consumed by both.
- Adapter crates that wrap external systems are consumed by both.
- Neither binary contains domain logic itself — both orchestrate.

## Related

- [[Headless First]]
- [[Feature Gating]]
