---
title: Engineering Codex
tags: [index, vault-root]
summary: Engineering standards for Rust projects: language-agnostic philosophy, Rust-specific code idioms, and Cargo workspace architecture.
---

*A three-section vault — philosophy, Rust practices, architecture — covering how to reason, code, and shape a workspace toward long-term reliability.*

# Engineering Codex

A general-purpose set of engineering standards for any Rust project. The philosophy section is language-agnostic and applies anywhere; the workspace-architecture and Rust-practices sections target Rust workspaces specifically.

## Start here

- [[Engineering Philosophy MOC]] — how to reason about correctness and attack programs with adversarial tests (language-agnostic)
- [[Rust Practices MOC]] — code-level idioms for writing Rust that favors stability and testability
- [[Workspace Architecture MOC]] — how to shape a Cargo workspace for a non-trivial application

## Folder map

- **Engineering Philosophy/** — eight Knuth-style principles for reasoning, design, and adversarial testing. Applies to any project, any language.
  - [[Engineering Philosophy MOC]] — the entry-point map
  - [[Output Format]] — the four-section template (correctness model → invariants → design → test strategy)
  - [[Principles Index]] — the eight principles
- **Rust Practices/** — principles for writing the Rust code inside crates.
  - [[Rust Practices MOC]] — the entry-point map
  - [[Rust Practices Checklist]] — condensed code-review form
  - Subfolders: Foundational, Type-Driven Design, Effect Isolation, Error Handling, Ownership and Mutation, Functions and Data, Testing, Tooling and Quality
- **Architecture/** — Cargo workspace structure, crate boundaries, and recurring architectural patterns.
  - [[Workspace Architecture MOC]] — the entry-point map
  - [[Core Principles]] — the load-bearing rules
  - [[Patterns Index]] — cross-cutting design patterns

## Adding to the vault

See [[CONTRIBUTING]] for note standards, the generalization test that gates project lessons, folder taxonomy, linking conventions, and the required index updates.

To check the vault for drift, prompt an agent to follow [[AUDIT]] — a self-contained runbook that compares vault state to CONTRIBUTING and emits a severity-tagged report.

If you (or an agent) hit a problem with an existing note during real use — wrong, vague, contradictory, or missing a case — file it per [[ISSUES]]. Refinements accumulate as tracked issue files in `Issues/`, with triage and resolution recorded in place.

For the GitHub workflow (branches, PRs, reviews, the lane mapping for what kind of PR you're opening) see [[CONTRIBUTING]] §10. For versioning policy and how releases are cut, see [[CONTRIBUTING]] §11. The public record of vault changes lives in [[CHANGELOG]].

Agents should read [[AGENTS]] first. It covers when to consult what, the full note index, and conventions for reading and citing notes.

## Using this vault from a project

To wire this vault into another project's repo so its AI tools (Claude Code, Codex, etc.) consult it on engineering tasks, see [[consumer-template/README]]. Three integration paths are documented (vendored copy, git submodule, symlink), along with a drop-in `CLAUDE.md` snippet at [[consumer-template/CLAUDE]] that the consuming project pastes into its own root.

For high-frequency tasks, the `Bundles/` folder contains pre-materialized single-file contexts so an agent can grab one bundle instead of resolving multiple wikilinks. See the AGENTS.md "Bundles/" index for the current list.

## How the three sections relate

| Section | Answers |
|---|---|
| **Engineering Philosophy** | How do we *reason and test* toward correctness? |
| **Rust Practices** | What does good code look like at the line level? |
| **Architecture** | What does a Rust workspace look like in the large? |

The architecture defines what good looks like in the large. The Rust practices define what good looks like at the line level. The philosophy defines how we know we've achieved either.

A worked example of how the three interlock: an architectural rule like keeping external-system boundaries narrow ([[Core Principles]]) is realized at the line level by [[Pure Core Effectful Edges]]; both are *defended* by the adversarial testing posture of [[Tests Should Make Programs Fail]] and the differential oracles of [[Sharp Oracles]] (e.g. comparing an optimized implementation to a reference via the [[CPU Reference Path]]).

## Adapting this vault

The philosophy section needs no adaptation — it is already general. The Rust practices section assumes Rust but is otherwise project-agnostic. The architecture section assumes a Cargo workspace; the patterns generalize to any application that wants pure core, narrow adapters, and a thin binary layer.

If you are using these notes on a non-Rust project, the Engineering Philosophy section is the part that travels.

## License

Prose is licensed under [CC BY 4.0](LICENSE); code samples and `consumer-template/` are licensed under [MIT](LICENSE).
