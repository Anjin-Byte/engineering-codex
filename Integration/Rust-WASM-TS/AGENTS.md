---
title: AGENTS — Rust↔WASM↔TS Integration Scope
tags: [meta, agent-entry-point, integration, placeholder]
summary: Placeholder for the Rust↔WASM↔TS integration scope. Populated in a future release; empty in the current version.
---

*Empty in this release. Notes for cross-language integration patterns (Rust compiled to WebAssembly and consumed from TypeScript) will land here in a future release.*

# AGENTS — Rust↔WASM↔TS Integration Scope

This scope is **empty** in the current vault release. It exists as a placeholder so the routing in [[AGENTS]] (root) is structurally complete for projects that span Rust, WebAssembly, and TypeScript.

When this scope is populated, agents working on integration tasks will load this file alongside [[Languages/Rust/AGENTS]] and (once it exists) `Languages/TypeScript/AGENTS.md`.

## Expected scope (future)

Topic areas that will live under `Integration/Rust-WASM-TS/` once authored:

- When Rust↔WASM is justified vs not (boundary-crossing cost; profiler-first rule)
- `wasm-bindgen` / `wasm-pack` interop boundary patterns
- Generated TypeScript surfaces from Rust (`.d.ts` shape, ABI stability)
- Workers and transferable buffers as the *first* parallelism move before WASM
- Build pipeline integration

These topics are sourced from research documented in `reference_reports_not_apart_of_vault/` and will be authored note-by-note following the standard vault contribution process ([[CONTRIBUTING]]).

## Status

**Until populated**: an agent on a Rust↔WASM↔TS integration project should load [[Languages/Rust/AGENTS]] (and the universal [[Engineering Philosophy/AGENTS]]) and rely on cross-language judgment. Note any gaps that surface as issues per [[ISSUES]] so they can inform the eventual content.
