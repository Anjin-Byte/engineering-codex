---
title: Workspace Architecture MOC
tags: [moc, architecture]
summary: Map of content for shaping a Cargo workspace whose crate boundaries reflect responsibilities that evolve at different speeds.
keywords: [crate boundaries, module organization, change rate, navigation hub, project structure, design index]
---

*A non-trivial Rust application is a set of crates whose seams track responsibility, not aesthetics; this MOC indexes the rules and patterns that enforce that.*

# Workspace Architecture — Map of Content

Entry point for understanding **why** a Rust workspace is shaped a particular way, and **where** to look for any specific architectural decision.

## The thesis

> A non-trivial Rust application should be modeled as a **set of crates whose boundaries reflect responsibilities that evolve at different speeds** — fast-moving CLI / orchestration code, slow-moving domain logic, narrowly-scoped adapters for external systems. The workspace shape is what keeps these concerns from fusing.

## Foundational reading order

1. [[Engineering Philosophy/Principles/Architectural Core Principles]] — the universal load-bearing rules
2. [[Languages/Rust/Workspace/Core Principles]] — how those rules apply to a Cargo workspace
3. [[Languages/Rust/Workspace/Workspace Layout]] — directory structure
4. [[Languages/Rust/Workspace/Cargo Workspace Configuration]] — root manifest
5. [[Languages/Rust/Workspace/Patterns/Patterns Index]] — recurring Cargo workspace patterns

## Patterns (Cargo wiring for universal principles)

- [[Languages/Rust/Workspace/Patterns/Headless First Cargo Wiring]] — Cargo expression of [[Engineering Philosophy/Principles/Headless First]]
- [[Languages/Rust/Workspace/Patterns/Reference Path Cargo Wiring]] — Cargo expression of [[Engineering Philosophy/Principles/Reference Implementation as Oracle]]
- [[Languages/Rust/Workspace/Patterns/Cargo Binary Strategy]] — Cargo expression of [[Engineering Philosophy/Principles/One Binary or Many]]
- [[Languages/Rust/Workspace/Patterns/Cargo Feature Gating]] — Cargo expression of [[Engineering Philosophy/Principles/Capability Slices vs Implementation Switches]]
- [[Languages/Rust/Workspace/Patterns/Shared Dependencies]] — what belongs in `workspace.dependencies`
- [[Languages/Rust/Workspace/Patterns/Workspace Lints and Profiles]] — root-only configuration

## The condensed rules

1. Core logic must survive without its optional adapters.
2. External-system adapters live behind narrow crate boundaries.
3. Optional UI / interactive concerns are isolated and opt-in.
4. The workspace root owns versions, profiles, and lints.
5. Default workspace commands target the primary binary, not every crate.
6. Crates reflect responsibilities, not aesthetics.
7. For any optimized implementation, keep a reference implementation as oracle.
