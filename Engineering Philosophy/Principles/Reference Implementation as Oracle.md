---
title: Reference Implementation as Oracle
tags: [principle, testing, philosophy]
summary: Every algorithm with an optimized implementation must keep a simpler reference implementation alongside it as the correctness oracle.
keywords: [naive implementation, golden path, differential testing, accelerated code, simd fallback, ffi parity, slow but correct]
---

*Optimized code is only trustworthy when a slower, obviously-correct twin exists alongside it to act as a differential oracle.*

# Reference Implementation as Oracle

> **Rule:** for every algorithm with an optimized implementation, a simpler reference implementation exists alongside it and is the correctness oracle.

Whenever you have an optimized path — SIMD, GPU, FFI, hand-rolled, hardware-accelerated, distributed, cached, incremental — keep a slower, deterministic reference implementation as the oracle.

## Why

Optimization debugging without an oracle has a notorious failure mode: *"it is very fast and very wrong."* You get plausible-looking output, you have no idea whether it's correct, and bisecting is miserable because the optimized pipeline is opaque (or runs on hardware you can't step through).

A reference implementation solves this by giving you:

1. **Correctness oracle.** Compare optimized output to reference output on the same input. Diffs above a small numerical tolerance are bugs.
2. **CI testability.** Tests of the algorithm itself run anywhere, including on machines without the optimized environment available.
3. **Easier debugging.** Standard debuggers and prints work. You can step through.
4. **Deterministic comparisons.** Floating-point order of operations is controlled.

## The structural pattern

- The **canonical, deterministic, testable implementation** lives in the language's most-portable layer (a pure library, a core module, the language's standard runtime).
- The **accelerated implementation** lives in an adapter or extension layer that depends on the optimized environment.
- A test layer exercises both with shared fixtures.

The language-specific application of this principle (where the reference lives in a Cargo workspace, how `#[cfg(test)]` or feature flags are used) is described per language. See [[Languages/Rust/Workspace/Patterns/Reference Path Cargo Wiring]] for the Rust application.

## Tolerances

Don't compare optimized and reference outputs with bitwise equality — different summation orders, FMA fusion, or rounding modes will diverge harmlessly. Use:

- absolute tolerance for small magnitudes
- relative tolerance for larger ones
- structural equality for non-numeric outputs (indices, topology)

## When the reference path can lag behind

Pragmatically, the reference implementation doesn't have to ship in the production binary. It can be:

- behind a test-only compile flag
- in a sibling module/package consumed only by tests
- a slower, more obviously correct version of the algorithm

What matters is that **for any optimized output, there exists a reference computation that produces the expected answer**.

## Where the pattern shows up

The pattern shows up wherever an optimization is opaque or hard to step through:

- A SIMD-vectorized inner loop versus a scalar reference.
- A GPU kernel versus a CPU reimplementation.
- An FFI call to a third-party numerical library versus a pure-language reimplementation.
- A distributed / parallel algorithm versus a single-threaded version.
- A cached / incremental path versus a "recompute from scratch" path.

In each case, the same discipline applies: write the boring version first; use it as the oracle that the fast version answers to.

## Related

- [[Engineering Philosophy/Principles/Sharp Oracles]]
- [[Engineering Philosophy/Principles/Architectural Core Principles]]
- [[Engineering Philosophy/Principles/Tests Should Make Programs Fail]]
- [[Languages/Rust/Workspace/Patterns/Reference Path Cargo Wiring]]
