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

1. [[Core Principles]] — the load-bearing rules
2. [[Workspace Layout]] — directory structure
3. [[Cargo Workspace Configuration]] — root manifest
4. [[Patterns Index]] — recurring design patterns

## Patterns

- [[Headless First]] — library-first; binaries are thin
- [[CPU Reference Path]] — keep a reference implementation alongside any optimized one
- [[Binary Strategy]] — one binary vs. several
- [[Feature Gating]] — when to use Cargo features
- [[Shared Dependencies]] — what belongs in `workspace.dependencies`
- [[Workspace Lints and Profiles]] — root-only configuration

## The condensed rules

1. Core logic must survive without its optional adapters.
2. External-system adapters live behind narrow crate boundaries.
3. Optional UI / interactive concerns are isolated and opt-in.
4. The workspace root owns versions, profiles, and lints.
5. Default workspace commands target the primary binary, not every crate.
6. Crates reflect responsibilities, not aesthetics.
7. For any optimized implementation, keep a reference implementation as oracle.
