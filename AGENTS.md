---
title: AGENTS — Vault Router
tags: [meta, agent-entry-point, router]
summary: Router for agents using this vault. Routes to per-scope AGENTS files by language scope and to vault-meta tasks.
---

*Detect the project's scope; load only the AGENTS files that apply. The vault separates universal philosophy from per-language practice so an agent never carries context for a scope it does not need.*

# AGENTS — Vault Router

This file is the front door. It routes agents to the right deeper-AGENTS file based on the project's scope. The vault separates content into three orthogonal scopes:

- **Universal** — engineering philosophy that travels to any language. Always relevant.
- **Per-language** — Rust today; other languages as they land. Only relevant when the project uses that language.
- **Integration** — Rust↔WASM↔TS interop and similar cross-scope concerns. Only relevant when the project crosses language boundaries.

## Routing

For any engineering task:

1. **Always load** [[Engineering Philosophy/AGENTS]] — universal triggers and the philosophy note index.
2. **If the project's language matches a per-language scope, also load** that scope's AGENTS file:
   - Rust → [[Languages/Rust/AGENTS]]
   - TypeScript → [[Languages/TypeScript/AGENTS]]
3. **If the project integrates Rust↔WASM↔TS, also load** [[Integration/Rust-WASM-TS/AGENTS]] — boundary discipline for projects that compile Rust to WebAssembly and consume it from TypeScript.

Do not load scopes that don't apply. Each AGENTS file is self-contained for its scope; loading more than the task requires wastes context.

## Vault-meta triggers

These tasks are about the vault itself, not about a project's code. They route to vault-root files regardless of language scope.

- **Adding a note to this vault** → [[CONTRIBUTING]] (read first; covers the generalization test, note shape, where it goes, and required index updates)
- **Auditing the vault for drift** → [[AUDIT]] (the on-demand runbook: mechanical + semantic checks against CONTRIBUTING, with severity-tagged findings and a repair policy)
- **Observing a problem with an existing note (use-time)** → [[ISSUES]] (file a tracked issue in `Issues/`; do not edit the target note autonomously; do not silently work around)
- **Pulling task context as a single read** → see `Bundles/<scope>/<task>.md` — pre-materialized contexts inlining the relevant notes, scoped per language so no cross-scope context is loaded. The relevant scope's AGENTS file lists which bundles cover its triggers.
- **Wiring this vault into a project** → [[consumer-template/README]] (three integration paths — vendored, submodule, symlink — plus the project-side CLAUDE.md snippet)
- **Opening a pull request** → [[CONTRIBUTING]] §10 (lanes, branch and commit conventions, PR template at `.github/pull_request_template.md`, vault-Issues vs GitHub-Issues distinction)
- **Cutting a release / understanding versions** → [[CONTRIBUTING]] §11 (semver-ish policy with knowledge-vault interpretation; pre-1.0 caveat; release flow); [[CHANGELOG]] (the public record)

## How to read a note

- Each note opens with an italicized one-line thesis. If you only need the gist, read that.
- Frontmatter `summary:` is the same idea in a different form (suitable for indexing or RAG).
- "Related" sections at the bottom of notes are the canonical next-hops.

## Vault conventions

- Plain CommonMark only — no Obsidian-specific syntax.
- Wikilinks resolve by filename; filenames are unique across the vault. The vault uses path-qualified wikilinks for unambiguous resolution when read as raw markdown (see [[CONTRIBUTING]] §5).
- Each scope has its own MOC (the human entry point) and AGENTS.md (the agent entry point); cross-scope wikilinks are normal where a topic genuinely crosses scopes.
