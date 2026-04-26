---
title: Core Principles
tags: [architecture, principles]
summary: "Four load-bearing rules that drive workspace decisions: pure core, narrow adapters, isolated UI, root-owned configuration."
keywords: [hexagonal architecture, ports and adapters, separation of concerns, dependency direction, layering, design smells]
---

*Four principles govern any decision about workspace shape; treat any future change that violates one of them as a smell to revisit.*

# Core Principles

Four principles drive workspace decisions. If a future change violates one of these, treat that as a smell and revisit the design.

## 1. Keep external-system boundaries narrow

Core algorithms should not know about the types of any external system (a device API, a windowing library, a heavy third-party SDK) unless absolutely necessary.

**Bad shape:** application logic directly manipulates external-system types, configuration, window state, CLI args, and file I/O all in one crate.

**Better shape:** split by responsibility:

- **core crate(s)** — domain types, algorithms, deterministic transformations
- **adapter crate(s)** — one per external system being wrapped, with its own narrow public API
- **cli crate** — argument parsing, file I/O, logging, exit codes
- **optional UI crate** — interactive front-end, isolated from core

That way, most logic can be tested without the external systems present at all.

See: [[Headless First]]

## 2. Treat headless as the default, interactive UI as an add-on

A library-first design lets the same code be embedded in CLIs, services, batch jobs, CI pipelines, and tests. Interactive UI is layered on top — never required for the core to work. See: [[Headless First]].

## 3. Make the workspace reflect deployable units

Rule of thumb: **if something can be built, tested, or shipped independently, it deserves its own crate.**

Do not split crates just to look sophisticated. A workspace with five meaningful crates is better than one with twelve decorative gourds.

See: [[Workspace Layout]]

## 4. Always keep a reference implementation alongside any optimized path

For any algorithm with an optimized implementation (SIMD, GPU, FFI, hand-rolled), maintain a reference implementation in pure, deterministic code.

Reasons:
- correctness oracle
- testability in CI without the optimized environment
- easier debugging
- deterministic comparisons

Workspace consequence:
- **core**: canonical, deterministic behavior
- **optimized adapter**: accelerated implementation

This saves you from the classic optimization debugging experience: *"it is very fast and very wrong."*

See: [[CPU Reference Path]]
