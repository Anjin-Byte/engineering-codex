---
title: Integration MOC
tags: [moc, integration, rust, wasm, typescript]
summary: Map of content for the Rust↔WASM↔TS integration scope — when to adopt, how to design the boundary, what tooling to use, how to test across the seam.
keywords: [navigation hub, rust wasm typescript integration, scope index, project structure, design index]
---

*A non-trivial Rust↔WASM↔TS project earns its complexity by treating the boundary as a designed artifact — coarse-grained API, validated inputs, profile-driven build, layered testing. This MOC indexes the decisions that get there.*

# Integration — Map of Content (Rust ↔ WASM ↔ TS)

Entry point for understanding **how** to design, build, and test a project where Rust compiled to WebAssembly is consumed from a TypeScript host, and **where** to look for any specific integration decision.

## The thesis

> The TS shell owns the world; the Rust kernel owns pure compute; the boundary between them is a small, deliberate, reviewed API. The Integration scope is for the *seam* — what crosses, how often, under what discipline. Everything else lives in the per-language scopes.

Integration notes deliberately cross-reference both [[Languages/Rust/AGENTS]] and [[Languages/TypeScript/AGENTS]] rather than duplicate their content. The value here is the boundary discipline; the kernel internals and the shell internals belong to their respective scopes.

## Foundational reading order

1. [[Integration/Rust-WASM-TS/Decision and Architecture/When Rust-WASM Is Justified]] — should this project even exist?
2. [[Integration/Rust-WASM-TS/Decision and Architecture/Workers Before WASM]] — the escalation order: TS → workers → Rust/WASM.
3. [[Integration/Rust-WASM-TS/Decision and Architecture/TypeScript Shell Rust Kernel Architecture]] — the canonical pattern.
4. [[Integration/Rust-WASM-TS/Boundary Design/Boundary Crossing Cost]] — why the boundary is a designed artifact.
5. [[Integration/Rust-WASM-TS/Tooling and Build/wasm-bindgen and wasm-pack]] — the toolchain.
6. [[Integration/Rust-WASM-TS/Testing/Cross-Boundary Testing]] — how to verify the whole thing works.

## Decision and architecture

- [[Integration/Rust-WASM-TS/Decision and Architecture/When Rust-WASM Is Justified]] — profile-first decision rule; design-review evidence required.
- [[Integration/Rust-WASM-TS/Decision and Architecture/TypeScript Shell Rust Kernel Architecture]] — TS owns lifecycle/DOM/I/O; Rust owns pure compute.
- [[Integration/Rust-WASM-TS/Decision and Architecture/Workers Before WASM]] — exhaust the TS-plus-workers ladder before reaching for Rust.

## Boundary design

- [[Integration/Rust-WASM-TS/Boundary Design/Boundary Crossing Cost]] — control plane vs data plane; coarse-grained APIs.
- [[Integration/Rust-WASM-TS/Boundary Design/Serialization Choice]] — `serde-wasm-bindgen` (default), FlatBuffers (zero-copy), Protobuf (cross-service).
- [[Integration/Rust-WASM-TS/Boundary Design/Generated TypeScript Surfaces]] — `.d.ts` is the public API; gate on its diff.

## Tooling and build

- [[Integration/Rust-WASM-TS/Tooling and Build/wasm-bindgen and wasm-pack]] — what each tool does; the standard build pipeline.
- [[Integration/Rust-WASM-TS/Tooling and Build/WASM Build and Loading]] — speed vs size profiles; `wasm-opt`; streaming compilation; module caching.

## Error handling and DOM

- [[Integration/Rust-WASM-TS/Error Handling and DOM/Error Propagation Across Boundary]] — `Result<T, E>` ↔ JS exceptions; `JsError`; panic strategy is profile-specific.
- [[Integration/Rust-WASM-TS/Error Handling and DOM/DOM Ownership Boundary]] — TS owns DOM mutation by default; Rust computes UI data.

## Testing

- [[Integration/Rust-WASM-TS/Testing/Cross-Boundary Testing]] — three layers: native Rust + cargo-fuzz, `wasm-bindgen-test`, TS-side integration.

## Cross-cutting universal principles

These live under [[Engineering Philosophy/Principles/Principles Index]] and apply to integration projects directly:

- [[Engineering Philosophy/Principles/Architectural Core Principles]] — narrow boundaries, deployable units, reference path alongside optimization.
- [[Engineering Philosophy/Principles/Reference Implementation as Oracle]] — the slower TypeScript implementation is the correctness oracle for the Rust/WASM kernel.
- [[Engineering Philosophy/Principles/Layered Risk Categories]] — Rust/WASM adoption changes the runtime, supply-chain, and concurrency layers simultaneously.
- [[Engineering Philosophy/Principles/Evidence Ladder for Testing]] — applied per-layer in [[Integration/Rust-WASM-TS/Testing/Cross-Boundary Testing]].
- [[Engineering Philosophy/Principles/Capability Slices vs Implementation Switches]] — Cargo features and TS feature flags both apply.

## Cross-references into per-language scopes

The Integration scope leans heavily on per-language work:

- **Rust side:** [[Languages/Rust/Workspace/Patterns/Reference Path Cargo Wiring]], [[Languages/Rust/Practices/Foundational/Predictable APIs]], [[Languages/Rust/Practices/Error Handling/Result vs Panic]], [[Languages/Rust/Practices/Testing/Three Levels of Tests]].
- **TypeScript side:** [[Languages/TypeScript/Practices/Foundational/Runtime Suitability Boundary]], [[Languages/TypeScript/Practices/Async and Concurrency/Event Loop as Correctness Signal]], [[Languages/TypeScript/Practices/Performance/Allocation Hygiene]], [[Languages/TypeScript/Practices/Error Handling/Process-Fatal vs Domain-Recoverable]], [[Languages/TypeScript/Practices/Testing/TypeScript Test Tooling]].

An agent working on integration tasks loads this scope alongside [[Languages/Rust/AGENTS]] and [[Languages/TypeScript/AGENTS]] (and the universal [[Engineering Philosophy/AGENTS]]).

## The condensed rules

1. Profile before adopting Rust/WASM; not every hot path needs it.
2. Exhaust workers + `TypedArray` first; WASM is the last escalation step.
3. TS owns the shell; Rust owns the kernel; nothing crosses unless it must.
4. Boundary calls are coarse; control plane is structured, data plane is bytes.
5. Generated `.d.ts` is a public API; gate on its diff.
6. Pick serialization by the boundary's character: ergonomic, hot-binary, or cross-service.
7. Two build profiles (speed and size); `wasm-opt` is mandatory.
8. Stream compile in browsers; cache the `WebAssembly.Module`; transfer it to workers.
9. `Result<T, E>` ↔ JS exception; map at the adapter layer to the application's typed errors.
10. TS owns DOM; Rust computes UI data, doesn't manipulate the DOM.
11. Test in three layers; native Rust catches most logic; wasm-bindgen-test catches toolchain drift; TS integration catches surface drift.
12. Test the exact panic strategy and profile you ship.

## Related

- [[Integration/Rust-WASM-TS/AGENTS]]
- [[Languages/Rust/Practices/Rust Practices MOC]]
- [[Languages/TypeScript/Practices/TypeScript Practices MOC]]
- [[Languages/Rust/Workspace/Workspace Architecture MOC]]
- [[Languages/TypeScript/Workspace/TypeScript Workspace Architecture MOC]]
- [[Engineering Philosophy/Engineering Philosophy MOC]]
