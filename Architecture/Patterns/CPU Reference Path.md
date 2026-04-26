---
title: CPU Reference Path
tags: [pattern, architecture, testing]
summary: Every algorithm with an optimized implementation must keep a simpler reference implementation in the core crate as its correctness oracle.
keywords: [naive implementation, golden path, differential testing, accelerated code, simd fallback, ffi parity, slow but correct]
---

*Optimized code is only trustworthy when a slower, obviously-correct twin exists alongside it to act as a differential oracle.*

# CPU Reference Path

> **Rule:** for every algorithm with an optimized implementation, a simpler reference implementation exists in the core crate and is the correctness oracle.

The name is historical (CPU-vs-GPU is the canonical case), but the pattern is general: **whenever you have an optimized path — SIMD, FFI, hand-rolled, hardware-accelerated, distributed — keep a slower, deterministic reference implementation as the oracle.**

## Why

Optimization debugging without an oracle has a notorious failure mode: *"it is very fast and very wrong."* You get plausible-looking output, you have no idea whether it's correct, and bisecting is miserable because the optimized pipeline is opaque (or runs on hardware you can't step through).

A reference implementation solves this by giving you:

1. **Correctness oracle.** Compare optimized output to reference output on the same input. Diffs above a small numerical tolerance are bugs.
2. **CI testability.** Tests of the algorithm itself run anywhere, including on machines without the optimized environment available.
3. **Easier debugging.** Standard debuggers and prints work. You can step through.
4. **Deterministic comparisons.** Floating-point order of operations is controlled.

## Workspace consequence

- **core crate** holds the canonical, deterministic, testable implementation.
- **optimized adapter crate** holds the accelerated implementation.
- A test crate (or `tests/` at the workspace root) exercises both with shared fixtures.

This is the structural payoff for [[Core Principles]] #4.

## Tolerances

Don't compare optimized and reference outputs with bitwise equality — different summation orders, FMA fusion, or rounding modes will diverge harmlessly. Use:

- absolute tolerance for small magnitudes
- relative tolerance for larger ones
- structural equality for non-numeric outputs (indices, topology)

## When the reference path can lag behind

Pragmatically, the reference implementation doesn't have to ship in the binary. It can be:

- behind a `#[cfg(test)]` feature
- in a sibling crate consumed only by tests
- a slower, more obviously correct version of the algorithm

What matters is that **for any optimized output, there exists a reference computation that produces the expected answer**.

## Beyond GPU

The pattern shows up wherever an optimization is opaque or hard to step through:

- A SIMD-vectorized inner loop versus a scalar reference.
- An FFI call to a third-party numerical library versus a pure-Rust reimplementation.
- A distributed / parallel algorithm versus a single-threaded version.
- A cached / incremental path versus a "recompute from scratch" path.

In each case, the same discipline applies: write the boring version first; use it as the oracle that the fast version answers to.

## Related

- [[Core Principles]]
- [[Sharp Oracles]]
