---
title: Pure Core, Effectful Edges
tags: [rust, principle, effects, testability]
summary: Push I/O, time, randomness, and device access to the system edges; keep the center pure and deterministic for testability.
keywords: [functional core imperative shell, side effects, hexagonal, dependency injection, deterministic logic, side effect free]
---

*The core is pure functions over data; effects live at the edges so the interesting logic stays deterministic and easy to test.*

# Pure Core, Effectful Edges

Isolate file I/O, env vars, clocks, randomness, networking, and device access at the **edges** of the system. Keep the **center** pure and deterministic.

## The shape

```
[ filesystem ] ─┐
[   network  ] ─┤
[    clock   ] ─┼─▶ adapters ─▶  PURE CORE  ─▶ adapters ─▶ [ filesystem ]
[   devices  ] ─┤                                          [    stdout    ]
[ env / args ] ─┘
```

- `main` parses args, sets up logging, wires dependencies.
- `lib` (or core modules) holds the actual behavior — pure functions, deterministic transformations.
- Adapter modules at the edge talk to the OS, devices, network, etc.

## Why this is the highest-value habit in Rust

Tests that depend on the filesystem, the network, the clock, or external devices are slow, flaky, hard to set up, and hard to reason about. Tests that depend on **pure functions** are fast, cheap, and boring — the best kind.

By moving logic out of `main` (and out of effect-laden modules) into pure code, you don't just make it testable. You make refactors safer, parallel execution trivial, and reasoning local.

## Per-crate expression

This principle applies recursively. A workspace tends to settle into:

- A pure **core** library crate (or several) — domain logic, deterministic transformations.
- One or more **adapter** crates that wrap external effects (devices, network clients, OS APIs).
- A thin **binary** crate (CLI, service entry point) that wires args, I/O, and adapters into the core.

The same shape recurs **inside** each crate too. A crate is allowed to talk to one effect (e.g. an adapter crate that wraps a device API) but should still keep its internal logic — orchestration, planning — as pure as possible, with the actual side-effecting calls in a thin layer.

## Diagnostic questions

Use these in code review:

- Does this function read the clock or env var directly? Could it take the value as a parameter?
- Does this function open a file path, or could it accept a `Read` / `Write`?
- Does this function call into an external system directly, or could it produce a description that an adapter executes?
- Could I test this function with no setup at all?

The closer the answer is to "yes, no setup," the further left along [[Push Correctness Left]] you've pushed it.

## Related

- [[Traits as Seams]]
- [[Tests Without Heroics]]
- [[Headless First]]
