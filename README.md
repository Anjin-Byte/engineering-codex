---
title: Engineering Codex
tags: [index, vault-root]
summary: "Engineering standards for multi-language projects: language-agnostic philosophy, per-language practices and architecture, and cross-language integration patterns."
---

*A multi-scope vault — universal philosophy, per-language disciplines (Rust today; more as they land), and cross-language integration — covering how to reason, code, and shape a project toward long-term reliability.*

# Engineering Codex

A general-purpose set of engineering standards organized into orthogonal scopes. The Engineering Philosophy scope is language-agnostic and applies anywhere. Per-language scopes hold code-level practices and project/workspace architecture for each language the vault covers (Rust today). The Integration scope holds patterns for projects that cross language boundaries (Rust↔WASM↔TS, etc.). An agent or reader loads only the scopes that apply.

## Repository format

This repository **is an [Obsidian](https://obsidian.md) vault** — clone it and open the folder as a vault to get the full experience: working `[[wikilinks]]` between notes, the graph view, tag search across frontmatter, and the folder structure used as the navigation spine. Every `.md` file uses YAML frontmatter (`title`, `tags`, `summary`, sometimes `keywords`) and Obsidian-style `[[Note Name]]` links rather than relative Markdown links.

It also reads fine as a plain GitHub repo — files render as Markdown and frontmatter is shown as a table. The shared `.obsidian/` config (graph, appearance, core plugins) is committed so cloners get a sensible default; per-machine state (`workspace.json`) is gitignored.

You don't need Obsidian to *use* the standards — agents and humans can read the files directly — but `[[wikilinks]]` won't be clickable on GitHub. If you want the navigation to work, use Obsidian or any editor with wikilink support.

## Start here

- [[Engineering Philosophy/Engineering Philosophy MOC]] — how to reason about correctness and attack programs with adversarial tests (language-agnostic, always relevant)
- [[Languages/Rust/Practices/Rust Practices MOC]] — code-level idioms for writing Rust that favors stability and testability
- [[Languages/Rust/Workspace/Workspace Architecture MOC]] — how to shape a Cargo workspace for a non-trivial Rust application

## Folder map

- **Engineering Philosophy/** — Knuth-style principles for reasoning, design, and adversarial testing. Applies to any project, any language. Always relevant.
  - [[Engineering Philosophy/Engineering Philosophy MOC]] — the entry-point map
  - [[Engineering Philosophy/Output Format]] — the four-section template (correctness model → invariants → design → test strategy)
  - [[Engineering Philosophy/Principles/Principles Index]] — the principles
- **Languages/** — per-language disciplines. Each language is its own self-contained scope; agents load only the languages that apply.
  - **Languages/Rust/Practices/** — code-level idioms for Rust.
    - [[Languages/Rust/Practices/Rust Practices MOC]] — the entry-point map
    - [[Languages/Rust/Practices/Rust Practices Checklist]] — condensed code-review form
    - Subfolders: Foundational, Type-Driven Design, Effect Isolation, Error Handling, Ownership and Mutation, Functions and Data, Testing, Tooling and Quality
  - **Languages/Rust/Workspace/** — Cargo workspace structure, crate boundaries, and Cargo-specific architectural patterns.
    - [[Languages/Rust/Workspace/Workspace Architecture MOC]] — the entry-point map
    - [[Languages/Rust/Workspace/Core Principles]] — the load-bearing rules
    - [[Languages/Rust/Workspace/Patterns/Patterns Index]] — Cargo-specific patterns
  - **Languages/TypeScript/Practices/** — code-level idioms for TypeScript.
    - [[Languages/TypeScript/Practices/TypeScript Practices MOC]] — the entry-point map
    - [[Languages/TypeScript/Practices/TypeScript Practices Checklist]] — condensed code-review form
    - Subfolders: Foundational, Type-Driven Design, Error Handling, Async and Concurrency, Testing, Tooling and Quality, Performance (advanced/optional)
  - **Languages/TypeScript/Workspace/** — TypeScript project layout, build configuration, and release engineering.
    - [[Languages/TypeScript/Workspace/TypeScript Workspace Architecture MOC]] — the entry-point map
    - Subfolder: Release Engineering (release gate pipeline, npm lockfile + audit, provenance + SBOM, Permission Model adoption)
- **Integration/** — patterns for projects that cross language boundaries.
  - **Integration/Rust-WASM-TS/** — projects compiling Rust to WebAssembly and consuming it from TypeScript: when it's justified, boundary design, tooling, error handling, DOM ownership, layered testing.
    - [[Integration/Rust-WASM-TS/Integration MOC]] — the entry-point map
    - Subfolders: Decision and Architecture, Boundary Design, Tooling and Build, Error Handling and DOM, Testing
- **Bundles/** — pre-materialized task contexts, scoped per language so loading a bundle never costs cross-scope context.
  - **Bundles/Universal/** — bundles drawing only from Engineering Philosophy.
  - **Bundles/Rust/** — bundles for Rust tasks (code review, test design, module design, error handling, API design).
  - **Bundles/TypeScript/** — bundles for TypeScript tasks (code review, test design, module design, error handling).

## Adding to the vault

See [[CONTRIBUTING]] for note standards, the generalization test that gates project lessons, folder taxonomy, linking conventions, and the required index updates.

To check the vault for drift, prompt an agent to follow [[AUDIT]] — a self-contained runbook that compares vault state to CONTRIBUTING and emits a severity-tagged report.

If you (or an agent) hit a problem with an existing note during real use — wrong, vague, contradictory, or missing a case — file it per [[ISSUES]]. Refinements accumulate as tracked issue files in `Issues/`, with triage and resolution recorded in place.

For the GitHub workflow (branches, PRs, reviews, the lane mapping for what kind of PR you're opening) see [[CONTRIBUTING]] §10. For versioning policy and how releases are cut, see [[CONTRIBUTING]] §11. The public record of vault changes lives in [[CHANGELOG]].

Agents should read [[AGENTS]] first. It covers when to consult what, the full note index, and conventions for reading and citing notes.

## Using this vault from a project

To wire this vault into another project's repo so its AI tools (Claude Code, Codex, etc.) consult it on engineering tasks, see [[consumer-template/README]]. Three integration paths are documented (vendored copy, git submodule, symlink), along with a drop-in `CLAUDE.md` snippet at [[consumer-template/CLAUDE]] that the consuming project pastes into its own root.

For high-frequency tasks, `Bundles/<scope>/<task>.md` provides pre-materialized single-file contexts so an agent can grab one bundle instead of resolving multiple wikilinks. Bundles are scoped per language; see the relevant scope's AGENTS file for the bundles that apply.

## How the scopes relate

| Scope | Answers |
|---|---|
| **Engineering Philosophy** (universal) | How do we *reason and test* toward correctness? |
| **Languages/`<lang>`/Practices** | What does good code look like at the line level *in this language*? |
| **Languages/`<lang>`/Workspace** | What does a project of this language look like in the large? |
| **Integration/** | How do we cross language boundaries safely? |

The per-language workspace scope defines what good looks like in the large. The per-language practices scope defines what good looks like at the line level. The philosophy defines how we know we've achieved either, regardless of language.

A worked example of how the scopes interlock for Rust: a universal architectural principle like keeping external-system boundaries narrow ([[Engineering Philosophy/Principles/Architectural Core Principles]], applied to Cargo at [[Languages/Rust/Workspace/Core Principles]]) is realized at the line level by [[Languages/Rust/Practices/Effect Isolation/Pure Core Effectful Edges]]; both are *defended* by the adversarial testing posture of [[Engineering Philosophy/Principles/Tests Should Make Programs Fail]] and the differential oracles of [[Engineering Philosophy/Principles/Sharp Oracles]] (e.g. comparing an optimized implementation to a reference via [[Engineering Philosophy/Principles/Reference Implementation as Oracle]] and its Cargo wiring at [[Languages/Rust/Workspace/Patterns/Reference Path Cargo Wiring]]).

## Adapting this vault

The Engineering Philosophy scope needs no adaptation — it is already general. Per-language scopes assume their language but make no project-specific assumptions inside that scope. The Integration scope is for projects that genuinely cross language boundaries.

If you are using these notes on a project whose language doesn't yet have its own scope in the vault, the Engineering Philosophy scope is the part that travels — the rest is opt-in.

## License

Prose is licensed under [CC BY 4.0](LICENSE); code samples and `consumer-template/` are licensed under [MIT](LICENSE).
