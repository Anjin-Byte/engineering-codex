---
title: Changelog
tags: [meta, changelog, versioning]
summary: Public record of vault changes by version. Resolved issues feed the Issues Resolved category. Versioning policy lives in CONTRIBUTING.md §11.
---

*The public record of vault changes. Each released version captures what was added, changed, removed, fixed, and which tracked issues closed.*

# Changelog

All notable changes to the Engineering Codex are recorded in this file.

The format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/) with one extension: an **Issues Resolved** category that links to resolved issue files in `Issues/`. Versioning follows the policy in [CONTRIBUTING.md §11](./CONTRIBUTING.md#11-versioning-and-the-changelog).

> **Pre-1.0:** the vault is currently in the `v0.x` series. Until `v1.0`, MINOR releases may include breaking changes (renamed/moved notes, trigger-group renames) — projects pinning to a specific version are recommended during this period.

---

## [Unreleased]

### Added
- Seven universal principles lifted into [`Engineering Philosophy/Principles/`](./Engineering%20Philosophy/Principles/) from the cross-language content of the TypeScript report (Phase B1 of the multilingual restructure): [[Engineering Philosophy/Principles/Layered Risk Categories]], [[Engineering Philosophy/Principles/Specification Risk vs Type Soundness]], [[Engineering Philosophy/Principles/Evidence Ladder for Testing]], [[Engineering Philosophy/Principles/Formal Modeling for Distributed State]], [[Engineering Philosophy/Principles/Error Budgets and Reliability Policy]], [[Engineering Philosophy/Principles/Staged Canary Deployment]], [[Engineering Philosophy/Principles/Configuration as Code Review]].
- New "Delivery and reliability principles" section in [[Engineering Philosophy/Principles/Principles Index]].
- Six new universal trigger groups in [[Engineering Philosophy/AGENTS]] (risk evaluation, test-strategy decisions, distributed-protocol verification, SLO/error-budget design, deployment planning, configuration review).
- `Languages/TypeScript/` scope created with 18 practice notes plus AGENTS file (Phase B2 of the multilingual restructure): Strict Compiler Profile, Runtime Suitability Boundary, Discriminated Unions and Exhaustive Handling, Branded Types for Domain Identifiers, Satisfies for Configuration Tables, Result Pattern for Expected Failures, Process-Fatal vs Domain-Recoverable, Runtime Validation at Boundaries, Floating Promise Hygiene, Event Loop as Correctness Signal, Async Context Propagation, Deadline Propagation with AbortSignal, TypeScript Test Tooling, Typed Linting with typescript-eslint, Reference TSConfig, Reference ESLint Config, plus the TypeScript Practices MOC and Checklist.
- New `Languages/TypeScript/Practices/Async and Concurrency/` folder for Node-specific async hygiene with no Rust analog.
- TypeScript routing pointer added to root [[AGENTS]] router and [[CLAUDE]] entry-point list.
- `Languages/TypeScript/Workspace/` scope created with 4 notes (Phase B3): Workspace Architecture MOC, Project Layout, TypeScript Execution, Bundling Decision.
- `Languages/TypeScript/Workspace/Release Engineering/` subfolder with 4 notes: Release Gate Pipeline, npm Lockfile and Install Discipline, Provenance and SBOM, Permission Model Adoption.
- `Languages/TypeScript/Practices/Performance/` folder with 2 advanced/optional notes: Allocation Hygiene, Object Shape Stability.
- `Bundles/TypeScript/` scope with 4 task bundles: Code-Review, Test-Design, Module-Design, Error-Handling.
- Three new universal trigger groups in [[Languages/TypeScript/AGENTS]] (project layout / monorepo / build, release pipeline / supply-chain controls, performance tuning).
- CONTRIBUTING.md §5 meta-file exception extended to require path-qualified wikilinks for bundle basenames (`Code-Review.md` etc. now exist under multiple `Bundles/<scope>/` paths).
- `Integration/Rust-WASM-TS/` scope populated (Phase C) with 11 content notes plus Integration MOC, replacing the placeholder AGENTS.md with the full scope file. Five subfolders: Decision and Architecture (3 notes), Boundary Design (3 notes), Tooling and Build (2 notes), Error Handling and DOM (2 notes), Testing (1 note).
- New cross-references in [[Languages/Rust/AGENTS]] (Wiring an optimized algorithm trigger now points at Integration scope) and [[Languages/TypeScript/AGENTS]] (new "Considering Rust/WASM for a hot path" trigger).
- Root [[AGENTS]] router and [[CLAUDE]] entry-point list updated to reflect that the Integration scope is populated.

### Changed
- [[Languages/Rust/AGENTS]] "Reviewing a PR or diff", "Designing a new module or crate", and "Writing tests" trigger groups now cross-reference the relevant new universal principles.

### Removed
-

### Fixed
-

### Issues Resolved
-

---

## [1.0.0] — 2026-05-03

The multilingual restructure. The vault now separates universal philosophy from per-language disciplines, so an agent on a Rust-only project never loads context for a language it does not need (and vice versa once additional languages land). This is a **major version bump with breaking changes** — every wikilink, MOC reference, and consumer-template path that pointed at `Rust Practices/`, `Architecture/`, or flat `Bundles/<task>.md` is gone.

The full design log lives outside the vault at [`reference_reports_not_apart_of_vault/restructure_plan.md`](./reference_reports_not_apart_of_vault/restructure_plan.md); the executable phase plan lived at the agent's plan file during execution.

### Added

- **`Languages/<lang>/` scope axis.** Top-level `Languages/Rust/Practices/` and `Languages/Rust/Workspace/` host all Rust-specific content. Future language scopes (TypeScript, etc.) slot in as `Languages/<lang>/` siblings.
- **`Integration/Rust-WASM-TS/AGENTS.md`** placeholder — empty scope for cross-language interop, populated in a future release.
- **Per-scope AGENTS files.** [`Engineering Philosophy/AGENTS.md`](./Engineering%20Philosophy/AGENTS.md) holds universal triggers; [`Languages/Rust/AGENTS.md`](./Languages/Rust/AGENTS.md) holds Rust triggers. The root [`AGENTS.md`](./AGENTS.md) is now a thin router (~2 KB) that selects which deeper AGENTS files to load by scope.
- **Five lifted universal architectural principles** under `Engineering Philosophy/Principles/`: [[Engineering Philosophy/Principles/Architectural Core Principles]], [[Engineering Philosophy/Principles/Headless First]], [[Engineering Philosophy/Principles/Reference Implementation as Oracle]], [[Engineering Philosophy/Principles/One Binary or Many]], [[Engineering Philosophy/Principles/Capability Slices vs Implementation Switches]]. These travel to any language.
- **Four Cargo-application patterns** under `Languages/Rust/Workspace/Patterns/`: Headless First Cargo Wiring, Reference Path Cargo Wiring, Cargo Binary Strategy, Cargo Feature Gating. Each is the Rust expression of its universal principle.
- **`Bundles/Universal/` and `Bundles/Rust/`.** Bundles are now scoped per language, so loading a bundle only ever costs in-scope context.
- **CONTRIBUTING.md generalization-test extension** — a language-scope dimension now decides whether a lesson belongs under `Engineering Philosophy/` or under a per-language scope.
- **AUDIT.md scope-aware checks** — orphan, structural-integrity, bundle-freshness, folder-fit, and trigger-group checks now operate across multiple AGENTS files and per-language MOCs.

### Changed

- **`Rust Practices/` → `Languages/Rust/Practices/`** (24 notes; subtree moved via `git mv`).
- **`Architecture/` → `Languages/Rust/Workspace/`** (7 notes moved; 4 patterns lifted out — see below).
- **Bundles split per scope:** `Bundles/Code-Review.md` → `Bundles/Rust/Code-Review.md`, similarly for Test-Design, Module-Design, Error-Handling, API-Design; `Bundles/Written-Analysis.md` → `Bundles/Universal/Written-Analysis.md`.
- **Root `AGENTS.md`** rewritten as a router (no per-language triggers; scope-detection guidance + vault-meta triggers only).
- **[`README.md`](./README.md), [`CLAUDE.md`](./CLAUDE.md), [`CONTRIBUTING.md`](./CONTRIBUTING.md), [`AUDIT.md`](./AUDIT.md), [`ISSUES.md`](./ISSUES.md), [`.github/pull_request_template.md`](./.github/pull_request_template.md), [`consumer-template/README.md`](./consumer-template/README.md), [`consumer-template/CLAUDE.md`](./consumer-template/CLAUDE.md)** — all references and folder taxonomies updated for the new structure.
- **Languages/Rust/Workspace/Core Principles.md** is now a thin Cargo-application note pointing at the universal [[Engineering Philosophy/Principles/Architectural Core Principles]]; the four universal rules moved to philosophy.
- **CONTRIBUTING.md §5** documents the meta-file exception: bare `[[AGENTS]]` / `[[README]]` / `[[CLAUDE]]` resolves to the vault-root copy; non-root copies must be path-qualified.
- **CONTRIBUTING.md §6 / §10.2 / §11.3** make the no-contributor-disclosure rule explicit and ban `Co-Authored-By:` trailers in commit metadata. (Originally entered in `[Unreleased]` ahead of this release; merged into 1.0.0.)

### Removed

- **`Architecture/` folder entirely.** All contents either moved under `Languages/Rust/Workspace/` or lifted into `Engineering Philosophy/Principles/`.
- **The four pattern notes at `Architecture/Patterns/`** (Headless First, CPU Reference Path, Binary Strategy, Feature Gating) are deleted as files. Their content now lives across one universal note plus one Cargo-application note per pattern (except Feature Gating's universal lift, where the Cargo expression has its own note: Cargo Feature Gating).

### Breaking

This release breaks every wikilink and consumer-template path that referenced the old layout:

- `[[Rust Practices/...]]` → `[[Languages/Rust/Practices/...]]` (or bare-name; filenames remain vault-unique)
- `[[Architecture/...]]` → `[[Languages/Rust/Workspace/...]]` for moved notes; lifted notes use new names (CPU Reference Path → Reference Implementation as Oracle, Binary Strategy → One Binary or Many, Feature Gating → Capability Slices vs Implementation Switches; "Headless First" keeps its name but moves to `Engineering Philosophy/Principles/`).
- `[[Bundles/<task>]]` → `[[Bundles/Rust/<task>]]` or `[[Bundles/Universal/<task>]]`.
- The single root `AGENTS.md` is now a router; agents that hard-coded reading just the root file should additionally read the per-scope AGENTS files.
- Consumer-template snippets updated to reference the per-scope AGENTS pattern.

Pre-1.0 consumers should pin to a `v0.x` tag if they need stability against this change.

---

## [0.1.0] — 2026-04-26

Initial release. The vault contains 46 content notes across three sections (Engineering Philosophy, Rust Practices, Architecture), six pre-bundled task contexts, and a complete set of meta-files governing addition, refinement, and audit.

### Added
- 8 Engineering Philosophy principles (Sharp Oracles, Tests Should Make Programs Fail, Reason About Correctness, Reasoning Beside Code, Normal Usage Is Not Testing, Regression Discipline, Explicit Not Magical, Human Understanding First) plus the Output Format note and section MOC.
- 23 Rust Practices notes across Foundational, Type-Driven Design, Effect Isolation, Error Handling, Ownership and Mutation, Functions and Data, Testing, and Tooling and Quality, plus the Practices Checklist and section MOC.
- 11 Architecture notes covering Workspace Layout, Cargo Workspace Configuration, Core Principles, six cross-cutting Patterns (Headless First, Feature Gating, Binary Strategy, Shared Dependencies, Workspace Lints and Profiles, CPU Reference Path), the Patterns Index, and section MOC.
- 6 task bundles in `Bundles/` (Code-Review, Test-Design, Module-Design, Error-Handling, API-Design, Written-Analysis) inlining each trigger group's notes for single-read agent consumption.
- Frontmatter standards on every note: `title`, `tags`, `summary`, `keywords` (4–8 retrieval terms per note), italic TL;DR line preceding the body.
- Vault-root meta-files governing process: [README.md](./README.md), [AGENTS.md](./AGENTS.md) (agent runbook with 16 trigger groups + full note index), [CLAUDE.md](./CLAUDE.md) (Claude Code entry pointer), [CONTRIBUTING.md](./CONTRIBUTING.md) (note authoring standards + GitHub workflow + versioning policy), [ISSUES.md](./ISSUES.md) (use-time refinement process for existing notes), [AUDIT.md](./AUDIT.md) (drift-check runbook).
- `consumer-template/` with drop-in CLAUDE.md snippet and README covering three integration paths (vendored, submodule, symlink) for consuming projects.
- `Issues/` folder structure (lifecycle: open → triaged → in-progress → resolved/rejected/duplicate; status tracked in frontmatter).

### Notes

This initial release was assembled by generalizing lessons drawn from prior project work. All domain-specific nouns were stripped during the port; the resulting principles and Rust idioms travel to any project.

---

[Unreleased]: ./
[1.0.0]: ./
[0.1.0]: ./
