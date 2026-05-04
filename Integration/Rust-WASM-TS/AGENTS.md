---
title: AGENTS — Rust↔WASM↔TS Integration Scope
tags: [meta, agent-entry-point, integration]
summary: Cross-language triggers and full note index for Integration/Rust-WASM-TS. Loaded only when the project compiles Rust to WebAssembly and consumes it from TypeScript.
---

*Cross-language guidance for projects spanning Rust, WebAssembly, and TypeScript. Load alongside the universal Engineering Philosophy scope, the Languages/Rust scope, and the Languages/TypeScript scope.*

# AGENTS — Rust↔WASM↔TS Integration Scope

This file routes agents through the integration portion of the vault: the boundary discipline between a TypeScript shell and a Rust/WASM kernel. The notes here cross-reference both [[Languages/Rust/AGENTS]] and [[Languages/TypeScript/AGENTS]] heavily — the kernel internals belong to Rust scope, the shell internals belong to TS scope, and the *seam* belongs here.

When working on integration tasks, an agent typically loads:

- [[Engineering Philosophy/AGENTS]] (universal, always)
- [[Languages/Rust/AGENTS]] (Rust scope)
- [[Languages/TypeScript/AGENTS]] (TS scope)
- This file (Integration scope)

## When to consult what (Integration triggers)

Group by trigger.

- **Evaluating whether to add Rust/WASM** → [[Integration/Rust-WASM-TS/Decision and Architecture/When Rust-WASM Is Justified]]; [[Integration/Rust-WASM-TS/Decision and Architecture/Workers Before WASM]]; [[Integration/Rust-WASM-TS/Boundary Design/Boundary Crossing Cost]]; [[Engineering Philosophy/Principles/Reference Implementation as Oracle]]
- **Designing a Rust/WASM kernel + TS shell** → [[Integration/Rust-WASM-TS/Decision and Architecture/TypeScript Shell Rust Kernel Architecture]]; [[Integration/Rust-WASM-TS/Boundary Design/Boundary Crossing Cost]]; [[Integration/Rust-WASM-TS/Boundary Design/Serialization Choice]]; [[Integration/Rust-WASM-TS/Boundary Design/Generated TypeScript Surfaces]]
- **Setting up wasm-bindgen / wasm-pack toolchain** → [[Integration/Rust-WASM-TS/Tooling and Build/wasm-bindgen and wasm-pack]]; [[Integration/Rust-WASM-TS/Tooling and Build/WASM Build and Loading]]; [[Integration/Rust-WASM-TS/Boundary Design/Generated TypeScript Surfaces]]
- **Picking a boundary serialization format** → [[Integration/Rust-WASM-TS/Boundary Design/Serialization Choice]]; [[Integration/Rust-WASM-TS/Boundary Design/Boundary Crossing Cost]]
- **Error handling across the WASM boundary** → [[Integration/Rust-WASM-TS/Error Handling and DOM/Error Propagation Across Boundary]]; [[Languages/TypeScript/Practices/Error Handling/Process-Fatal vs Domain-Recoverable]]; [[Languages/Rust/Practices/Error Handling/Result vs Panic]]
- **Deciding whether Rust should touch the DOM** → [[Integration/Rust-WASM-TS/Error Handling and DOM/DOM Ownership Boundary]]; [[Integration/Rust-WASM-TS/Decision and Architecture/TypeScript Shell Rust Kernel Architecture]]
- **Testing a Rust/WASM module end-to-end** → [[Integration/Rust-WASM-TS/Testing/Cross-Boundary Testing]]; [[Engineering Philosophy/Principles/Evidence Ladder for Testing]]; [[Languages/TypeScript/Practices/Testing/TypeScript Test Tooling]]; [[Languages/Rust/Practices/Testing/Three Levels of Tests]]
- **Performance optimization for WASM** → [[Integration/Rust-WASM-TS/Tooling and Build/WASM Build and Loading]]; [[Integration/Rust-WASM-TS/Boundary Design/Boundary Crossing Cost]]; [[Integration/Rust-WASM-TS/Decision and Architecture/Workers Before WASM]]
- **Setting up the WASM release pipeline** → [[Integration/Rust-WASM-TS/Tooling and Build/WASM Build and Loading]]; [[Integration/Rust-WASM-TS/Boundary Design/Generated TypeScript Surfaces]]; [[Languages/TypeScript/Workspace/Release Engineering/Release Gate Pipeline]]

## Full note index (Integration/Rust-WASM-TS/)

Every note in the Integration scope, grouped by folder, with its `summary:` text.

### Integration/Rust-WASM-TS/

- [[Integration/Rust-WASM-TS/Integration MOC]] — Map of content for the Rust↔WASM↔TS integration scope — when to adopt Rust/WASM, how to design the boundary, what tooling to use, how to handle errors and DOM, and how to test across the seam.

### Integration/Rust-WASM-TS/Decision and Architecture/

- [[Integration/Rust-WASM-TS/Decision and Architecture/When Rust-WASM Is Justified]] — Rust/WASM is justified only when profiling shows a real CPU-bound kernel or when an existing Rust ecosystem provides clear leverage; reflexive adoption costs more than it saves.
- [[Integration/Rust-WASM-TS/Decision and Architecture/TypeScript Shell Rust Kernel Architecture]] — TypeScript owns lifecycle, DOM, routing, auth, observability; Rust owns pure compute and narrow data transforms behind a small, deliberately stable API. The shell decides; the kernel computes.
- [[Integration/Rust-WASM-TS/Decision and Architecture/Workers Before WASM]] — Before introducing Rust/WASM, exhaust the TypeScript-plus-workers-plus-transferable-buffers escalation. Workers may suffice for medium-hot work; WASM wins only when the kernel is heavy enough to amortize boundary cost.

### Integration/Rust-WASM-TS/Boundary Design/

- [[Integration/Rust-WASM-TS/Boundary Design/Boundary Crossing Cost]] — WASM and JS live in separate memory; object-heavy crossings are slower than byte-oriented. Split a control plane from a data plane; keep APIs coarse.
- [[Integration/Rust-WASM-TS/Boundary Design/Serialization Choice]] — Three boundary-serialization choices — `serde-wasm-bindgen` (ergonomic default), FlatBuffers (zero-copy hot binary), Protobuf (cross-service schemas). Pick by ergonomics, throughput, or schema longevity.
- [[Integration/Rust-WASM-TS/Boundary Design/Generated TypeScript Surfaces]] — wasm-bindgen's generated `.d.ts` is a public API; review its diff on every change and design Rust exports to produce a usable TS surface.

### Integration/Rust-WASM-TS/Tooling and Build/

- [[Integration/Rust-WASM-TS/Tooling and Build/wasm-bindgen and wasm-pack]] — `wasm-bindgen` generates the JS glue and TS declarations; `wasm-pack` runs the build and packaging; `wasm-bindgen-futures` bridges Rust `Future`s and JS `Promise`s.
- [[Integration/Rust-WASM-TS/Tooling and Build/WASM Build and Loading]] — Maintain two release profiles (speed and size); run `wasm-opt` post-link; prefer `instantiateStreaming` for browser loading; cache the compiled `Module` and transfer it to workers.

### Integration/Rust-WASM-TS/Error Handling and DOM/

- [[Integration/Rust-WASM-TS/Error Handling and DOM/Error Propagation Across Boundary]] — Rust returns `Result<T, E>`; `Err` becomes a JS exception caught at containment boundaries. Async via `wasm-bindgen-futures`. Panic strategy is profile-specific.
- [[Integration/Rust-WASM-TS/Error Handling and DOM/DOM Ownership Boundary]] — TypeScript owns DOM mutation, event wiring, and UI state. Rust may compute UI data but does not own the DOM by default; `web-sys` exists but rarely pays back.

### Integration/Rust-WASM-TS/Testing/

- [[Integration/Rust-WASM-TS/Testing/Cross-Boundary Testing]] — Test in three layers — native Rust + cargo-fuzz, then `wasm-bindgen-test` in Node and headless browsers, then TS-side integration against the generated package.

## How to read a note

- Each note opens with an italicized one-line thesis. If you only need the gist, read that.
- Frontmatter `summary:` is the same idea in a different form (suitable for indexing or RAG).
- "Related" sections at the bottom of notes are the canonical next-hops, often spanning into the per-language scopes.
