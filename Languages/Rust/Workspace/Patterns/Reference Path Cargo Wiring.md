---
title: Reference Path Cargo Wiring
tags: [pattern, cargo, testing]
summary: How to express the reference-implementation-as-oracle pattern in a Cargo workspace — core crate holds the canonical reference, optimized adapter holds the accelerated path, tests cross-validate.
keywords: [naive implementation, reference oracle, simd fallback, gpu fallback, ffi parity, test crate, cfg test]
---

*The Cargo expression of reference-implementation-as-oracle: the core crate owns the deterministic reference, the optimized adapter crate owns the accelerated path, and shared tests cross-validate.*

# Reference Path Cargo Wiring

The universal principle lives at [[Engineering Philosophy/Principles/Reference Implementation as Oracle]]. This note covers how a Cargo workspace expresses it.

## Crate topology

- **`core` crate** — holds the canonical, deterministic, testable implementation. No optional features that would change the algorithm; no FFI, no GPU. The reference computes exactly the answer the optimized version is supposed to compute.
- **Optimized adapter crate** (e.g. `accel-gpu`, `accel-simd`, `accel-ffi`) — holds the accelerated implementation, depends on the optimized environment.
- **Tests** — exercise both implementations on shared fixtures. Lives either in a dedicated test crate or under `tests/` at the workspace root for cross-crate flows.

## Where the reference can live when you don't want to ship it

The reference does not have to be in the production binary's dependency closure:

- Behind a `#[cfg(test)]` block in the optimized adapter — useful for unit-level cross-checks.
- In a sibling crate consumed only by integration tests.
- Behind a Cargo feature like `reference` (default off) — see [[Languages/Rust/Workspace/Patterns/Cargo Feature Gating]].

What matters is that the reference is *runnable in CI* on the same fixtures the optimized path consumes.

## Differential test pattern

Typical workspace-root test that exercises both:

```rust
#[test]
fn gpu_matches_reference() {
    let input = fixture::checkerboard_512();
    let reference_out = core::compute(&input);
    let optimized_out = accel_gpu::compute(&input);
    assert_close(&reference_out, &optimized_out, /* abs */ 1e-6, /* rel */ 1e-4);
}
```

Use absolute + relative tolerance, not bitwise equality (FMA fusion and summation order will diverge harmlessly). For non-numeric outputs (indices, topology), structural equality.

## When to require both paths in CI

If the optimized environment is available in CI (typical for SIMD), run both on every PR. If it requires hardware that CI lacks (GPU, specialized accelerators), at minimum keep the reference path running on every PR; gate the optimized path behind an environment-aware job.

## Related

- [[Engineering Philosophy/Principles/Reference Implementation as Oracle]]
- [[Engineering Philosophy/Principles/Sharp Oracles]]
- [[Languages/Rust/Workspace/Core Principles]]
- [[Languages/Rust/Workspace/Patterns/Cargo Feature Gating]]
