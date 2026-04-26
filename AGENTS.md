# AGENTS.md — How agents should use this vault

## Purpose

This vault is a set of engineering standards for any Rust project: a language-agnostic philosophy for reasoning about correctness, code-level Rust idioms, and a Cargo workspace architecture. The philosophy section travels to any language; the rest assumes Rust and Cargo. An agent should treat the contents as authoritative guidance — when a task overlaps a topic covered here, the relevant note's rules and rationale apply by default, and any deviation should be justified explicitly.

## When to consult what

Group by trigger, not by folder.

- **Reviewing a PR or diff** → [[Rust Practices/Rust Practices Checklist]]; [[Engineering Philosophy/Principles/Tests Should Make Programs Fail]]; [[Engineering Philosophy/Principles/Sharp Oracles]]; [[Engineering Philosophy/Principles/Regression Discipline]]
- **Designing a new module or crate** → [[Architecture/Workspace Architecture MOC]]; [[Architecture/Core Principles]]; [[Architecture/Workspace Layout]]; [[Rust Practices/Foundational/Type System as Design Tool]]; [[Rust Practices/Foundational/Push Correctness Left]]
- **Writing tests** → [[Rust Practices/Testing/Three Levels of Tests]]; [[Rust Practices/Testing/Edge Cases and Properties]]; [[Rust Practices/Effect Isolation/Tests Without Heroics]]; [[Engineering Philosophy/Principles/Sharp Oracles]]; [[Engineering Philosophy/Principles/Normal Usage Is Not Testing]]
- **Handling errors** → [[Rust Practices/Error Handling/Result vs Panic]]; [[Rust Practices/Error Handling/Domain Errors at Boundaries]]
- **Modeling a domain value or state machine** → [[Rust Practices/Type-Driven Design/Make Invalid States Unrepresentable]]; [[Rust Practices/Type-Driven Design/Newtypes and Domain Types]]; [[Rust Practices/Type-Driven Design/State Transition Types]]; [[Rust Practices/Type-Driven Design/Checked Constructors and Builders]]
- **Touching unsafe or FFI** → [[Rust Practices/Ownership and Mutation/Unsafe Quarantine]]; [[Rust Practices/Ownership and Mutation/Obvious Ownership]]; [[Rust Practices/Ownership and Mutation/Minimize Shared Mutability]]
- **Adding a feature flag** → [[Architecture/Patterns/Feature Gating]]; [[Architecture/Patterns/Binary Strategy]]; [[Architecture/Patterns/Headless First]]
- **Wiring an optimized algorithm (GPU, SIMD, parallel)** → [[Architecture/Patterns/CPU Reference Path]]; [[Engineering Philosophy/Principles/Sharp Oracles]]; [[Rust Practices/Effect Isolation/Pure Core Effectful Edges]]
- **Configuring the Cargo workspace** → [[Architecture/Cargo Workspace Configuration]]; [[Architecture/Patterns/Shared Dependencies]]; [[Architecture/Patterns/Workspace Lints and Profiles]]
- **Setting up CI or tooling** → [[Rust Practices/Tooling and Quality/CI Quality Bar]]; [[Rust Practices/Tooling and Quality/Clippy as Discipline]]; [[Rust Practices/Tooling and Quality/Documentation as Truth]]
- **Producing a written analysis or design doc** → [[Engineering Philosophy/Output Format]]; [[Engineering Philosophy/Principles/Reason About Correctness]]; [[Engineering Philosophy/Principles/Reasoning Beside Code]]; [[Engineering Philosophy/Principles/Explicit Not Magical]]; [[Engineering Philosophy/Principles/Human Understanding First]]
- **Refactoring a long function or tangled module** → [[Rust Practices/Functions and Data/Small Functions Narrow Contracts]]; [[Rust Practices/Functions and Data/Boring Data Layouts]]; [[Rust Practices/Effect Isolation/Pure Core Effectful Edges]]; [[Rust Practices/Effect Isolation/Traits as Seams]]
- **Designing a public API** → [[Rust Practices/Foundational/Predictable APIs]]; [[Rust Practices/Type-Driven Design/Checked Constructors and Builders]]; [[Rust Practices/Tooling and Quality/Documentation as Truth]]
- **Adding a note to this vault** → [[CONTRIBUTING]] (read first; covers the generalization test, note shape, where it goes, and required index updates)
- **Auditing the vault for drift** → [[AUDIT]] (the on-demand runbook: mechanical + semantic checks against CONTRIBUTING, with severity-tagged findings and a repair policy)
- **Observing a problem with an existing note (use-time)** → [[ISSUES]] (file a tracked issue in `Issues/`; do not edit the target note autonomously; do not silently work around)
- **Pulling task context as a single read** → see `Bundles/` — pre-materialized contexts inlining the notes for high-frequency tasks (code review, test design, module design, error handling, API design, written analysis). One read instead of resolving multiple wikilinks.
- **Wiring this vault into a project** → [[consumer-template/README]] (three integration paths — vendored, submodule, symlink — plus the project-side CLAUDE.md snippet)
- **Opening a pull request** → [[CONTRIBUTING]] §10 (lanes, branch and commit conventions, PR template at `.github/pull_request_template.md`, vault-Issues vs GitHub-Issues distinction)
- **Cutting a release / understanding versions** → [[CONTRIBUTING]] §11 (semver-ish policy with knowledge-vault interpretation; pre-1.0 caveat; release flow); [[CHANGELOG]] (the public record)

## Full note index

Every note in the vault, grouped by folder, with its `summary:` text.

### Vault root

- [[README]] — Engineering standards for Rust projects: language-agnostic philosophy, Rust-specific code idioms, and Cargo workspace architecture.
- [[CLAUDE]] — Pointer file for Claude Code and similar tools that look for CLAUDE.md by convention; defers to AGENTS.md as the authoritative agent guide.
- [[CONTRIBUTING]] — Standards for adding, organizing, and linking notes — including the generalization test that gates project lessons into general knowledge.
- [[AUDIT]] — Step-by-step procedure an LLM follows when prompted to audit this vault for drift against CONTRIBUTING.md.
- [[ISSUES]] — How humans and agents file, track, and resolve issues against existing notes when use surfaces a problem.
- [[CHANGELOG]] — Public record of vault changes, organized by version. Resolved issues feed the `Issues Resolved` category. Versioning policy in CONTRIBUTING.md §11.

### Bundles/

Pre-materialized contexts that inline several notes into a single file, one per high-frequency task type. Read the bundle once instead of orchestrating wikilink resolution across multiple notes.

- [[Bundles/Code-Review]] — Reviewing a PR or diff (Rust Practices Checklist; Tests Should Make Programs Fail; Sharp Oracles; Regression Discipline).
- [[Bundles/Test-Design]] — Writing tests (Three Levels of Tests; Edge Cases and Properties; Tests Without Heroics; Sharp Oracles; Normal Usage Is Not Testing).
- [[Bundles/Module-Design]] — Designing a new module or crate (Workspace Architecture MOC; Core Principles; Workspace Layout; Type System as Design Tool; Push Correctness Left).
- [[Bundles/Error-Handling]] — Handling errors (Result vs Panic; Domain Errors at Boundaries).
- [[Bundles/API-Design]] — Designing a public API (Predictable APIs; Checked Constructors and Builders; Documentation as Truth).
- [[Bundles/Written-Analysis]] — Producing a written analysis or design doc (Output Format; Reason About Correctness; Reasoning Beside Code; Explicit Not Magical; Human Understanding First).

### consumer-template/

- [[consumer-template/README]] — Wiring guide for projects that want to consume this vault (vendored, submodule, or symlink integration paths).
- [[consumer-template/CLAUDE]] — Drop-in CLAUDE.md snippet for the consuming project's repo.

### Architecture/

- [[Architecture/Workspace Architecture MOC]] — Map of content for shaping a Cargo workspace whose crate boundaries reflect responsibilities that evolve at different speeds.
- [[Architecture/Core Principles]] — Four load-bearing rules that drive workspace decisions: pure core, narrow adapters, isolated UI, root-owned configuration.
- [[Architecture/Workspace Layout]] — Recommended directory layout for a non-trivial Rust workspace, separating core, adapters, binaries, and reference implementations.
- [[Architecture/Cargo Workspace Configuration]] — Use a virtual workspace root that owns members, shared dependencies, lints, and profiles but ships no package of its own.

### Architecture/Patterns/

- [[Architecture/Patterns/Patterns Index]] — Index of cross-cutting workspace patterns: each entry is a rule with its rationale.
- [[Architecture/Patterns/Headless First]] — Build the system to run to completion with no UI or event loop; any interactive front-end is an opt-in adapter.
- [[Architecture/Patterns/CPU Reference Path]] — Every algorithm with an optimized implementation must keep a simpler reference implementation in the core crate as its correctness oracle.
- [[Architecture/Patterns/Binary Strategy]] — Decide between one binary with subcommands and several binaries based on whether the surfaces share lifecycle and dependencies.
- [[Architecture/Patterns/Feature Gating]] — Cargo features should represent real capability slices a downstream user would toggle, not internal implementation switches.
- [[Architecture/Patterns/Shared Dependencies]] — Hoist dependencies into workspace.dependencies only when truly shared; keep crate-local deps local to preserve crate independence.
- [[Architecture/Patterns/Workspace Lints and Profiles]] — Profiles and lints belong only in the root manifest; member crates inherit so the quality bar stays uniform.

### Engineering Philosophy/

- [[Engineering Philosophy/Engineering Philosophy MOC]] — Map of content for the Knuth-style engineering principles: how to reason, design, and adversarially test software in any language.
- [[Engineering Philosophy/Output Format]] — Four-section deliverable template: correctness model, invariants, design, test strategy — each catches a class of failure the others miss.

### Engineering Philosophy/Principles/

- [[Engineering Philosophy/Principles/Principles Index]] — Index of the eight Knuth-style engineering principles in their canonical reading order.
- [[Engineering Philosophy/Principles/Reason About Correctness]] — Treat correctness as something to be reasoned about with explicit invariants and proofs of fit, not merely hoped for through tests passing.
- [[Engineering Philosophy/Principles/Tests Should Make Programs Fail]] — The purpose of testing is to make the program fail until the important bugs are found, not to confirm the happy path.
- [[Engineering Philosophy/Principles/Sharp Oracles]] — Test against an independent reference (closed-form, alternate algorithm, or simpler implementation) so failures point at the bug, not the assertion.
- [[Engineering Philosophy/Principles/Regression Discipline]] — Every discovered bug should produce or strengthen a test so past failures become permanent members of the suite.
- [[Engineering Philosophy/Principles/Normal Usage Is Not Testing]] — Running the program in the normal way exercises the happy path; real testing requires deliberate adversarial inputs and edge cases.
- [[Engineering Philosophy/Principles/Reasoning Beside Code]] — When producing code, also produce the surrounding reasoning so readers can follow why it is shaped this way.
- [[Engineering Philosophy/Principles/Explicit Not Magical]] — Prefer clear control flow, stable interfaces, and obvious data movement over cleverness that obscures reasoning.
- [[Engineering Philosophy/Principles/Human Understanding First]] — Structure code so a reader can reconstruct the reasoning behind it; design must be narratable, not just functional.

### Rust Practices/

- [[Rust Practices/Rust Practices MOC]] — Map of content for Rust code-level practices that push correctness left into types, construction, ownership, and tests.
- [[Rust Practices/Rust Practices Checklist]] — Condensed code-review checklist of every Rust practice in the vault, suitable as a PR self-check.

### Rust Practices/Foundational/

- [[Rust Practices/Foundational/Push Correctness Left]] — The meta-principle behind every Rust practice: encode constraints into types, construction, and APIs so fewer bugs survive to runtime.
- [[Rust Practices/Foundational/Type System as Design Tool]] — Treat the type system as a design tool that encodes invariants, not merely as a compiler obstacle to satisfy.
- [[Rust Practices/Foundational/Predictable APIs]] — Optimize public APIs for predictability and consistency over cleverness, in line with the Rust API Guidelines.

### Rust Practices/Type-Driven Design/

- [[Rust Practices/Type-Driven Design/Make Invalid States Unrepresentable]] — Encode invariants in the type so the bad value cannot exist, instead of writing runtime checks for it later.
- [[Rust Practices/Type-Driven Design/Newtypes and Domain Types]] — Wrap primitives in newtypes whenever a value carries semantic meaning, so the type system enforces the distinction.
- [[Rust Practices/Type-Driven Design/State Transition Types]] — Model lifecycle phases as distinct types so the compiler tracks where a value is in its sequence of states.
- [[Rust Practices/Type-Driven Design/Checked Constructors and Builders]] — Push validation into construction with checked constructors and builders rather than post-init mutation and reassertion.

### Rust Practices/Effect Isolation/

- [[Rust Practices/Effect Isolation/Pure Core Effectful Edges]] — Push I/O, time, randomness, and device access to the system edges; keep the center pure and deterministic for testability.
- [[Rust Practices/Effect Isolation/Traits as Seams]] — Use traits where you genuinely need to inject behavior or substitute implementations in tests, not as default abstraction.
- [[Rust Practices/Effect Isolation/Tests Without Heroics]] — If testing requires elaborate setup or fakes, the design is fighting testability; treat heroic tests as a design smell.

### Rust Practices/Error Handling/

- [[Rust Practices/Error Handling/Result vs Panic]] — Use Result for recoverable failure expected in normal operation; reserve panic for bugs and broken invariants.
- [[Rust Practices/Error Handling/Domain Errors at Boundaries]] — Library boundaries expose typed enum errors via thiserror; anyhow-style wrappers belong only at the application boundary.

### Rust Practices/Ownership and Mutation/

- [[Rust Practices/Ownership and Mutation/Obvious Ownership]] — Treat borrow-checker friction as a design signal; redesign for clear ownership rather than fighting the checker.
- [[Rust Practices/Ownership and Mutation/Minimize Shared Mutability]] — Rust makes mutation safe but not simple; localize and make explicit any mutation, especially across threads.
- [[Rust Practices/Ownership and Mutation/Unsafe Quarantine]] — Confine unsafe to small, audited modules with explicit safety contracts; never let it leak through public APIs.

### Rust Practices/Functions and Data/

- [[Rust Practices/Functions and Data/Small Functions Narrow Contracts]] — Each function should have one job and a crisp contract; smaller scope means fewer ways to be wrong in one place.
- [[Rust Practices/Functions and Data/Boring Data Layouts]] — Prefer boring, explicit data shapes over clever encodings; verbosity in modeling buys reliability over years.

### Rust Practices/Testing/

- [[Rust Practices/Testing/Three Levels of Tests]] — Use unit, integration, and doc tests together because each catches a class of mistake the others cannot.
- [[Rust Practices/Testing/Edge Cases and Properties]] — Test the inputs where humans improvise badly: empties, boundaries, overflows; pair example tests with property-based ones.

### Rust Practices/Tooling and Quality/

- [[Rust Practices/Tooling and Quality/Clippy as Discipline]] — Run Clippy aggressively on every build with intentional configuration; treat it as a discipline, not an occasional cleanup.
- [[Rust Practices/Tooling and Quality/CI Quality Bar]] — CI enforces format, lint, build, test, and doc-test on every change; checks here are the floor below which work cannot ship.
- [[Rust Practices/Tooling and Quality/Documentation as Truth]] — Document invariants and examples next to the code; Rust doc tests turn documentation into executable, verified truth.

## How to read a note

- Each note opens with an italicized one-line thesis. If you only need the gist, read that.
- Frontmatter `summary:` is the same idea in a different form (suitable for indexing or RAG).
- "Related" sections at the bottom of notes are the canonical next-hops.

## Vault conventions

- Plain CommonMark only — no Obsidian-specific syntax.
- Wikilinks resolve by filename; filenames are unique across the vault.
- The three MOCs ([[Engineering Philosophy/Engineering Philosophy MOC]], [[Rust Practices/Rust Practices MOC]], [[Architecture/Workspace Architecture MOC]]) are the human entry points; AGENTS.md is the agent entry point.
