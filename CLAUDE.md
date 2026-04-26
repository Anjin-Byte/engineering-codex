---
title: Claude Code Entry Point
tags: [meta, agent-entry-point]
summary: Pointer file for Claude Code and similar tools that look for CLAUDE.md by convention; defers to AGENTS.md as the authoritative agent guide.
---

*This vault is a set of engineering standards. Read [[AGENTS]] first; everything routes from there.*

# CLAUDE.md

This file exists because Claude Code (and other AI coding tools) look for `CLAUDE.md` at a project root by convention. The authoritative agent guidance for this vault lives in `AGENTS.md` — read that first.

## Vault purpose

Engineering standards for any Rust project: a language-agnostic philosophy section, code-level Rust practices, and Cargo workspace architecture. The contents apply across projects; project-specific knowledge belongs elsewhere.

## Entry points

- [[AGENTS]] — when to consult what; full note index; reading conventions. **Start here.**
- [[README]] — human-oriented orientation.
- [[CONTRIBUTING]] — how to add a new note (only for additions; existing-note refinements go through ISSUES).
- [[ISSUES]] — how to file an issue against an existing note when use surfaces a problem.
- [[AUDIT]] — runbook an agent follows when prompted to audit the vault for drift.
- [[Bundles/Code-Review]], [[Bundles/Test-Design]], [[Bundles/Module-Design]], [[Bundles/Error-Handling]], [[Bundles/API-Design]], [[Bundles/Written-Analysis]] — pre-bundled task contexts; one read instead of orchestrating wikilink resolution across multiple notes.

## Using this vault from a project

See [[consumer-template/README]] for how to wire this vault into your own project's `CLAUDE.md`. Three paths supported: vendored copy, git submodule, symlink.
