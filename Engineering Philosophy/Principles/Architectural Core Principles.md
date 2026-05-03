---
title: Architectural Core Principles
tags: [principle, architecture, philosophy]
summary: "Four load-bearing rules that drive project shape regardless of language: narrow external boundaries, headless default, deployable-unit modularity, reference path alongside optimization."
keywords: [hexagonal architecture, ports and adapters, separation of concerns, dependency direction, layering, design smells]
---

*Four rules govern any decision about project shape; treat any future change that violates one of them as a smell to revisit.*

# Architectural Core Principles

Four principles drive project-shape decisions regardless of language. If a future change violates one of these, treat that as a smell and revisit the design. Each rule applies at every scale: a single library, a multi-package workspace, or a service boundary.

## 1. Keep external-system boundaries narrow

Core algorithms should not know about the types of any external system (a device API, a windowing library, a heavy third-party SDK, a database driver) unless absolutely necessary.

**Bad shape:** application logic directly manipulates external-system types, configuration, window state, CLI args, and file I/O all in one module.

**Better shape:** split by responsibility:

- **core module(s)** — domain types, algorithms, deterministic transformations
- **adapter module(s)** — one per external system being wrapped, with its own narrow public API
- **edge module** — argument parsing, file I/O, logging, exit codes, request/response handling
- **optional UI module** — interactive front-end, isolated from core

That way, most logic can be tested without the external systems present at all.

See: [[Engineering Philosophy/Principles/Headless First]]

## 2. Treat headless as the default, interactive UI as an add-on

A library-first design lets the same code be embedded in CLIs, services, batch jobs, CI pipelines, and tests. Interactive UI is layered on top — never required for the core to work.

See: [[Engineering Philosophy/Principles/Headless First]].

## 3. Make the project shape reflect deployable units

Rule of thumb: **if something can be built, tested, or shipped independently, it deserves its own module/package boundary.**

Do not split modules just to look sophisticated. Five meaningful units beats twelve decorative ones. The reverse is also true: a single monolith that contains parts with genuinely different lifecycles will eventually need splitting under pressure.

Each language's project layout encodes this principle differently — Cargo workspaces, npm workspaces, Go modules, Python packages — but the principle is the same: deployable-unit boundaries should be visible in the project structure.

## 4. Always keep a reference implementation alongside any optimized path

For any algorithm with an optimized implementation (SIMD, GPU, FFI, hand-rolled, distributed), maintain a reference implementation in pure, deterministic code.

Reasons:
- correctness oracle
- testability in CI without the optimized environment
- easier debugging
- deterministic comparisons

Project shape consequence:
- canonical, deterministic behavior lives in the most-portable layer
- accelerated implementation lives in an adapter / extension layer

This saves you from the classic optimization debugging experience: *"it is very fast and very wrong."*

See: [[Engineering Philosophy/Principles/Reference Implementation as Oracle]]

## Related

- [[Engineering Philosophy/Principles/Headless First]]
- [[Engineering Philosophy/Principles/Reference Implementation as Oracle]]
- [[Engineering Philosophy/Principles/One Binary or Many]]
- [[Engineering Philosophy/Principles/Capability Slices vs Implementation Switches]]
- [[Languages/Rust/Workspace/Core Principles]]
