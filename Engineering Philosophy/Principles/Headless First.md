---
title: Headless First
tags: [principle, architecture, philosophy]
summary: Build the system to run to completion with no UI or event loop; any interactive front-end is an opt-in adapter.
keywords: [no gui, ci friendly, batch processing, library first, daemon mode, server side, automation]
---

*The default execution path is headless and library-first; UI is an outer layer that the core never depends on.*

# Headless First

> **Rule:** the system runs to completion with no UI, no window, and no event loop. Any interactive front-end is opt-in.

The general form of the rule: **library-first, executables thin**. Core logic lives in libraries that can be driven from anywhere — a CLI, a service, a test, a notebook, a batch job. UI and interactivity are an additional, optional layer.

## Why this is non-negotiable

A non-headless design gets used in places where windows can't or shouldn't open:
- CI pipelines
- Headless servers
- Containerized batch jobs
- Automated regression suites
- Remote SSH sessions

If the headless path is an afterthought, all of the above become awkward — extra flags, display server stubs, suppressed exceptions, dummy event loops, etc.

## Architectural consequence

- Core and adapter modules must function without any UI dependency.
- The CLI / service module must not import any UI / windowing library.
- Any interactive front-end is either a separate executable (preferred) or behind a build-time / configuration switch.

The language-specific application of this principle (e.g. how to express it in Cargo features or as a separate npm package) lives in each language's workspace patterns. See [[Languages/Rust/Workspace/Patterns/Headless First Cargo Wiring]] for the Rust application.

## What "interactive UI" means here

- A window opens; the user can watch results and interact (rotate, inspect, scrub a parameter).
- Or: a TUI / web UI that requires its own runtime loop.

It is **not**:
- Required for any computational output.
- A debugging substitute for proper logging or snapshot tests.

## The general principle

The same logic applies to any concern that adds runtime requirements: profilers, telemetry exporters, debug overlays, hot-reload systems. Each should be opt-in (configuration switch or separate executable), never load-bearing for the core path.

## Related

- [[Engineering Philosophy/Principles/Architectural Core Principles]]
- [[Engineering Philosophy/Principles/One Binary or Many]]
- [[Engineering Philosophy/Principles/Capability Slices vs Implementation Switches]]
- [[Languages/Rust/Workspace/Patterns/Headless First Cargo Wiring]]
