---
title: When Rust-WASM Is Justified
tags: [integration, rust, wasm, typescript, decision]
summary: Rust/WASM is justified only when profiling shows a real CPU-bound kernel or when an existing Rust ecosystem provides clear leverage; reflexive adoption costs more than it saves.
keywords: [rust wasm, webassembly, profile first, decision rule, cpu bound, ffi cost, kernel adoption]
---

*Profile first. Workers second. Rust/WASM third — only when the kernel is compute-heavy enough to amortize the FFI boundary cost over meaningful work.*

# When Rust-WASM Is Justified

> **Rule:** every new Rust/WASM proposal answers two design-review questions before any code is written: *what does profiling show this kernel costs?* and *why won't TypeScript-plus-workers solve it?* Without those answers, the project should not start.

The pattern of "let's add Rust/WASM to make this faster" is one of the most common ways to spend a quarter and produce no measurable improvement. The runtime overhead of crossing the JS/WASM boundary, the build complexity of maintaining two toolchains, and the hiring/staffing implications of a Rust-fluent team all compound. WASM wins when the math works out; it loses when the bottleneck is somewhere the language switch can't reach.

## The decision rule

A new Rust/WASM kernel is justified when **at least one** of the following is true and *measured*, not asserted:

1. **Profiling identifies a CPU-bound hot path** that cannot be eliminated by algorithmic change, that runs frequently enough to matter, and whose work is coarse-grained enough that boundary-crossing cost amortizes.
2. **A mature Rust crate provides leverage** that re-implementing in TypeScript would cost weeks or quarters — codecs, parsers, scientific libraries, image/audio kernels, simulation engines.
3. **Memory or determinism constraints** require a model TypeScript cannot provide — bit-exact numerical computation, deterministic floating-point semantics, manual memory layout.

If none of those is true, Rust/WASM is the wrong tool. The right tool is probably TypeScript with workers and `TypedArray` (see [[Integration/Rust-WASM-TS/Decision and Architecture/Workers Before WASM]]) or an algorithmic improvement to the existing TypeScript code.

## Where Rust/WASM is a strong fit

Concrete cases where the math works out:

- **Codecs.** Image encoding/decoding (JPEG XL, WebP, AVIF), audio (Opus), video (AV1) — pure compute over byte buffers, decade-mature Rust crates.
- **Binary parsers and decoders.** Wire-format parsers, protocol decoders, file-format readers. Coarse-grained input (a buffer), coarse-grained output (a parsed structure), CPU-bound work.
- **Numerical kernels.** Linear algebra, FFT, signal processing, statistical computation. Hot inner loops where SIMD and tight memory layout produce material wins.
- **Image and audio transforms.** Filters, color-space conversions, audio effects. Buffer in, buffer out.
- **Simulation engines.** Physics, particle systems, agent simulations. Run for many milliseconds per call; one boundary crossing per frame.
- **Cryptographic primitives.** Constant-time properties, audited Rust implementations, side-channel resistance.
- **Domain-specific solvers and rules engines.** Constraint solvers, rules evaluators, deterministic decision engines.

In each case the workload is *coarse* (one boundary crossing per substantial unit of work) and *CPU-bound* (the cost is computation, not orchestration).

## Where Rust/WASM is a poor fit

The cases where the math does not work:

- **DOM-heavy orchestration.** UI state, event wiring, layout. The DOM is JS-native; bridging it through WASM adds friction that nothing pays back. See [[Integration/Rust-WASM-TS/Error Handling and DOM/DOM Ownership Boundary]].
- **High-frequency chatty boundaries.** A call that crosses the boundary 10,000 times per second to do tiny work. The boundary cost dominates; you've added overhead, not removed it.
- **I/O-bound workloads.** If the bottleneck is network, storage, or the database, Rust/WASM doesn't help. A faster local computation that waits on the same blocking I/O is no faster overall.
- **Code whose actual cost is GC or allocation in JS.** Sometimes the right answer is allocation discipline in TypeScript (see [[Languages/TypeScript/Practices/Performance/Allocation Hygiene]]), not a language switch.
- **Application orchestration.** Lifecycle, routing, auth, telemetry. These belong in the host (TypeScript). See [[Integration/Rust-WASM-TS/Decision and Architecture/TypeScript Shell Rust Kernel Architecture]].

The shared property: in each poor-fit case, the *boundary* dominates the *kernel* — either because the kernel is tiny, the boundary is hot, or the bottleneck is outside both.

## The required design-review evidence

Before approving a Rust/WASM proposal, the design review demands four pieces of evidence:

1. **A flame graph or equivalent profile** showing the hot path that motivates the work. Hand-wave performance arguments fail this gate.
2. **A boundary-call frequency estimate.** How many times per second does the proposed JS↔WASM boundary cross? What is the per-call work? See [[Integration/Rust-WASM-TS/Boundary Design/Boundary Crossing Cost]] for why this matters.
3. **A TypeScript-only baseline.** Could the same hot path be solved with workers + transferable buffers + algorithmic improvements? If yes, why isn't it?
4. **An owner.** Rust/WASM kernels have a long maintenance tail. Someone is on the hook for the toolchain, the build pipeline, the audit trail, and the eventual upgrade. If no one wants to own it, it should not exist.

The evidence is not a formality. The most common failure of Rust/WASM proposals is missing evidence #1 or #3, and the project ships a faster kernel that improves the user-visible workload by an unmeasurable amount because the bottleneck was somewhere else.

## What "wins" actually looks like

A successful Rust/WASM kernel typically shows:

- **Wall-clock improvement** of multiple ×, not single-digit percent. If the win is 5%, the boundary and complexity overhead probably eat it.
- **Tail-latency improvement** that matters. p99 dropping from 200ms to 30ms changes user experience; p50 dropping from 5ms to 4ms doesn't.
- **Throughput at high load** that previously required scale-out. If the same machine now handles 5× the requests, that's real.
- **A library you'd otherwise have to write.** Avoiding 6 months of compression-codec implementation by using a Rust crate is real leverage.

When the post-implementation measurement shows none of these, the project should be revisited honestly, not declared a success because the kernel exists.

## Composing with other principles

- [[Engineering Philosophy/Principles/Reference Implementation as Oracle]] — when adding an optimized WASM kernel alongside a slower reference, keep both. The slower reference is the correctness oracle.
- [[Engineering Philosophy/Principles/Architectural Core Principles]] — rule 4 ("reference path alongside optimization") applies directly.
- [[Engineering Philosophy/Principles/Layered Risk Categories]] — Rust/WASM adoption changes the runtime layer and the supply-chain layer simultaneously. Both need controls.
- [[Languages/TypeScript/Practices/Performance/Allocation Hygiene]] and [[Languages/TypeScript/Practices/Performance/Object Shape Stability]] — the TS-side performance options to exhaust before reaching for Rust.
- [[Integration/Rust-WASM-TS/Decision and Architecture/Workers Before WASM]] — the escalation rule.

## Related

- [[Integration/Rust-WASM-TS/Decision and Architecture/TypeScript Shell Rust Kernel Architecture]]
- [[Integration/Rust-WASM-TS/Decision and Architecture/Workers Before WASM]]
- [[Integration/Rust-WASM-TS/Boundary Design/Boundary Crossing Cost]]
- [[Engineering Philosophy/Principles/Reference Implementation as Oracle]]
- [[Languages/TypeScript/Practices/Foundational/Runtime Suitability Boundary]]
