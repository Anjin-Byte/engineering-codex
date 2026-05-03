---
title: Claude Code Entry Point
tags: [meta, agent-entry-point]
summary: Pointer file for Claude Code and similar tools that look for CLAUDE.md by convention; defers to AGENTS.md as the authoritative agent router.
---

*This vault is a set of engineering standards. Read [[AGENTS]] first; the root AGENTS.md routes by language scope.*

# CLAUDE.md

This file exists because Claude Code (and other AI coding tools) look for `CLAUDE.md` at a project root by convention. The authoritative agent guidance for this vault lives in `AGENTS.md` — read that first. The root AGENTS.md is a router: it points to per-scope AGENTS files so an agent loads only the scope its task needs.

## Vault purpose

Engineering standards for multi-language projects: a language-agnostic philosophy scope, per-language code-level practices and architecture (Rust today; other languages as they land), and integration patterns for cross-language interop. The contents apply across projects; project-specific knowledge belongs elsewhere.

## Entry points

- [[AGENTS]] — root router: detects scope and points at the relevant deeper-AGENTS files. **Start here.**
- [[Engineering Philosophy/AGENTS]] — universal scope; always relevant.
- [[Languages/Rust/AGENTS]] — Rust scope; load when the project uses Rust.
- [[Integration/Rust-WASM-TS/AGENTS]] — Rust↔WASM↔TS interop scope (placeholder until populated).
- [[README]] — human-oriented orientation.
- [[CONTRIBUTING]] — how to add a new note (only for additions; existing-note refinements go through ISSUES).
- [[ISSUES]] — how to file an issue against an existing note when use surfaces a problem.
- [[AUDIT]] — runbook an agent follows when prompted to audit the vault for drift.
- Bundles — pre-materialized task contexts, scoped per language: [[Bundles/Universal/Written-Analysis]] for universal tasks; [[Bundles/Rust/Code-Review]], [[Bundles/Rust/Test-Design]], [[Bundles/Rust/Module-Design]], [[Bundles/Rust/Error-Handling]], [[Bundles/Rust/API-Design]] for Rust tasks. One read instead of orchestrating wikilink resolution across multiple notes.

## Using this vault from a project

See [[consumer-template/README]] for how to wire this vault into your own project's `CLAUDE.md`. Three paths supported: vendored copy, git submodule, symlink.
