---
title: Headless First
tags: [pattern, architecture]
summary: Build the system to run to completion with no UI or event loop; any interactive front-end is an opt-in adapter.
keywords: [no gui, ci friendly, batch processing, library first, daemon mode, server side, automation]
---

*The default execution path is headless and library-first; UI is an outer layer that the core never depends on.*

# Headless First

> **Rule:** the system runs to completion with no UI, no window, and no event loop. Any interactive front-end is opt-in.

The general form of the rule: **library-first, binaries thin**. Core logic lives in libraries that can be driven from anywhere — a CLI, a service, a test, a notebook, a batch job. UI and interactivity are an additional, optional layer.

## Why this is non-negotiable

A non-headless design gets used in places where windows can't or shouldn't open:
- CI pipelines
- Headless servers
- Containerized batch jobs
- Automated regression suites
- Remote SSH sessions

If the headless path is an afterthought, all of the above become awkward — extra flags, display server stubs, suppressed exceptions, dummy event loops, etc.

## Architectural consequence

- Core and adapter crates must function without any UI dependency.
- The CLI crate must not import any UI / windowing library.
- Any interactive front-end is either a separate binary (preferred) or behind a Cargo feature gate.

See [[Binary Strategy]] and [[Feature Gating]].

## What "interactive UI" means here

- A window opens; the user can watch results and interact (rotate, inspect, scrub a parameter).
- Or: a TUI / web UI that requires its own runtime loop.

It is **not**:
- Required for any computational output.
- A debugging substitute for proper logging or snapshot tests.

## The general principle

The same logic applies to any concern that adds runtime requirements: profilers, telemetry exporters, debug overlays, hot-reload systems. Each should be opt-in (feature flag or separate binary), never load-bearing for the core path.

## Related

- [[Core Principles]]
- [[Binary Strategy]]
- [[Feature Gating]]
