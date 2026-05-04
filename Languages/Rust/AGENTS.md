---
title: AGENTS — Rust Scope
tags: [meta, agent-entry-point, rust]
summary: Rust-specific triggers and full note index for Languages/Rust/. Loaded only when the project uses Rust.
---

*Rust-specific guidance: code-level practices and Cargo workspace architecture. Load alongside the universal Engineering Philosophy scope when working on a Rust project.*

# AGENTS — Rust Scope

This file routes agents through the Rust portion of the vault: code-level practices under [[Languages/Rust/Practices/Rust Practices MOC]] and Cargo workspace architecture under [[Languages/Rust/Workspace/Workspace Architecture MOC]]. Triggers below cross-reference universal philosophy notes (always loaded alongside this scope) where the principle is universal even though the application is Rust-specific.

## When to consult what (Rust triggers)

Group by trigger.

- **Reviewing a PR or diff** → [[Languages/Rust/Practices/Rust Practices Checklist]]; [[Engineering Philosophy/Principles/Tests Should Make Programs Fail]]; [[Engineering Philosophy/Principles/Sharp Oracles]]; [[Engineering Philosophy/Principles/Regression Discipline]]; [[Engineering Philosophy/Principles/Evidence Ladder for Testing]]. Bundle: [[Bundles/Rust/Code-Review]].
- **Designing a new module or crate** → [[Languages/Rust/Workspace/Workspace Architecture MOC]]; [[Languages/Rust/Workspace/Core Principles]]; [[Languages/Rust/Workspace/Workspace Layout]]; [[Languages/Rust/Practices/Foundational/Type System as Design Tool]]; [[Languages/Rust/Practices/Foundational/Push Correctness Left]]; [[Engineering Philosophy/Principles/Architectural Core Principles]]; [[Engineering Philosophy/Principles/Layered Risk Categories]]. Bundle: [[Bundles/Rust/Module-Design]].
- **Writing tests** → [[Languages/Rust/Practices/Testing/Three Levels of Tests]]; [[Languages/Rust/Practices/Testing/Edge Cases and Properties]]; [[Languages/Rust/Practices/Effect Isolation/Tests Without Heroics]]; [[Engineering Philosophy/Principles/Sharp Oracles]]; [[Engineering Philosophy/Principles/Normal Usage Is Not Testing]]; [[Engineering Philosophy/Principles/Evidence Ladder for Testing]]. Bundle: [[Bundles/Rust/Test-Design]].
- **Handling errors** → [[Languages/Rust/Practices/Error Handling/Result vs Panic]]; [[Languages/Rust/Practices/Error Handling/Domain Errors at Boundaries]]. Bundle: [[Bundles/Rust/Error-Handling]].
- **Modeling a domain value or state machine** → [[Languages/Rust/Practices/Type-Driven Design/Make Invalid States Unrepresentable]]; [[Languages/Rust/Practices/Type-Driven Design/Newtypes and Domain Types]]; [[Languages/Rust/Practices/Type-Driven Design/State Transition Types]]; [[Languages/Rust/Practices/Type-Driven Design/Checked Constructors and Builders]]
- **Touching unsafe or FFI** → [[Languages/Rust/Practices/Ownership and Mutation/Unsafe Quarantine]]; [[Languages/Rust/Practices/Ownership and Mutation/Obvious Ownership]]; [[Languages/Rust/Practices/Ownership and Mutation/Minimize Shared Mutability]]
- **Adding a feature flag** → [[Engineering Philosophy/Principles/Capability Slices vs Implementation Switches]]; [[Languages/Rust/Workspace/Patterns/Cargo Feature Gating]]; [[Languages/Rust/Workspace/Patterns/Cargo Binary Strategy]]; [[Languages/Rust/Workspace/Patterns/Headless First Cargo Wiring]]
- **Wiring an optimized algorithm (GPU, SIMD, parallel)** → [[Engineering Philosophy/Principles/Reference Implementation as Oracle]]; [[Languages/Rust/Workspace/Patterns/Reference Path Cargo Wiring]]; [[Engineering Philosophy/Principles/Sharp Oracles]]; [[Languages/Rust/Practices/Effect Isolation/Pure Core Effectful Edges]]
- **Configuring the Cargo workspace** → [[Languages/Rust/Workspace/Cargo Workspace Configuration]]; [[Languages/Rust/Workspace/Patterns/Shared Dependencies]]; [[Languages/Rust/Workspace/Patterns/Workspace Lints and Profiles]]
- **Setting up CI or tooling** → [[Languages/Rust/Practices/Tooling and Quality/CI Quality Bar]]; [[Languages/Rust/Practices/Tooling and Quality/Clippy as Discipline]]; [[Languages/Rust/Practices/Tooling and Quality/Documentation as Truth]]
- **Refactoring a long function or tangled module** → [[Languages/Rust/Practices/Functions and Data/Small Functions Narrow Contracts]]; [[Languages/Rust/Practices/Functions and Data/Boring Data Layouts]]; [[Languages/Rust/Practices/Effect Isolation/Pure Core Effectful Edges]]; [[Languages/Rust/Practices/Effect Isolation/Traits as Seams]]
- **Designing a public API** → [[Languages/Rust/Practices/Foundational/Predictable APIs]]; [[Languages/Rust/Practices/Type-Driven Design/Checked Constructors and Builders]]; [[Languages/Rust/Practices/Tooling and Quality/Documentation as Truth]]. Bundle: [[Bundles/Rust/API-Design]].

## Full note index (Languages/Rust/)

Every note in the Rust scope, grouped by folder, with its `summary:` text.

### Languages/Rust/Practices/

- [[Languages/Rust/Practices/Rust Practices MOC]] — Map of content for Rust code-level practices that push correctness left into types, construction, ownership, and tests.
- [[Languages/Rust/Practices/Rust Practices Checklist]] — Condensed code-review checklist of every Rust practice in the vault, suitable as a PR self-check.

### Languages/Rust/Practices/Foundational/

- [[Languages/Rust/Practices/Foundational/Push Correctness Left]] — The meta-principle behind every Rust practice: encode constraints into types, construction, and APIs so fewer bugs survive to runtime.
- [[Languages/Rust/Practices/Foundational/Type System as Design Tool]] — Treat the type system as a design tool that encodes invariants, not merely as a compiler obstacle to satisfy.
- [[Languages/Rust/Practices/Foundational/Predictable APIs]] — Optimize public APIs for predictability and consistency over cleverness, in line with the Rust API Guidelines.

### Languages/Rust/Practices/Type-Driven Design/

- [[Languages/Rust/Practices/Type-Driven Design/Make Invalid States Unrepresentable]] — Encode invariants in the type so the bad value cannot exist, instead of writing runtime checks for it later.
- [[Languages/Rust/Practices/Type-Driven Design/Newtypes and Domain Types]] — Wrap primitives in newtypes whenever a value carries semantic meaning, so the type system enforces the distinction.
- [[Languages/Rust/Practices/Type-Driven Design/State Transition Types]] — Model lifecycle phases as distinct types so the compiler tracks where a value is in its sequence of states.
- [[Languages/Rust/Practices/Type-Driven Design/Checked Constructors and Builders]] — Push validation into construction with checked constructors and builders rather than post-init mutation and reassertion.

### Languages/Rust/Practices/Effect Isolation/

- [[Languages/Rust/Practices/Effect Isolation/Pure Core Effectful Edges]] — Push I/O, time, randomness, and device access to the system edges; keep the center pure and deterministic for testability.
- [[Languages/Rust/Practices/Effect Isolation/Traits as Seams]] — Use traits where you genuinely need to inject behavior or substitute implementations in tests, not as default abstraction.
- [[Languages/Rust/Practices/Effect Isolation/Tests Without Heroics]] — If testing requires elaborate setup or fakes, the design is fighting testability; treat heroic tests as a design smell.

### Languages/Rust/Practices/Error Handling/

- [[Languages/Rust/Practices/Error Handling/Result vs Panic]] — Use Result for recoverable failure expected in normal operation; reserve panic for bugs and broken invariants.
- [[Languages/Rust/Practices/Error Handling/Domain Errors at Boundaries]] — Library boundaries expose typed enum errors via thiserror; anyhow-style wrappers belong only at the application boundary.

### Languages/Rust/Practices/Ownership and Mutation/

- [[Languages/Rust/Practices/Ownership and Mutation/Obvious Ownership]] — Treat borrow-checker friction as a design signal; redesign for clear ownership rather than fighting the checker.
- [[Languages/Rust/Practices/Ownership and Mutation/Minimize Shared Mutability]] — Rust makes mutation safe but not simple; localize and make explicit any mutation, especially across threads.
- [[Languages/Rust/Practices/Ownership and Mutation/Unsafe Quarantine]] — Confine unsafe to small, audited modules with explicit safety contracts; never let it leak through public APIs.

### Languages/Rust/Practices/Functions and Data/

- [[Languages/Rust/Practices/Functions and Data/Small Functions Narrow Contracts]] — Each function should have one job and a crisp contract; smaller scope means fewer ways to be wrong in one place.
- [[Languages/Rust/Practices/Functions and Data/Boring Data Layouts]] — Prefer boring, explicit data shapes over clever encodings; verbosity in modeling buys reliability over years.

### Languages/Rust/Practices/Testing/

- [[Languages/Rust/Practices/Testing/Three Levels of Tests]] — Use unit, integration, and doc tests together because each catches a class of mistake the others cannot.
- [[Languages/Rust/Practices/Testing/Edge Cases and Properties]] — Test the inputs where humans improvise badly: empties, boundaries, overflows; pair example tests with property-based ones.

### Languages/Rust/Practices/Tooling and Quality/

- [[Languages/Rust/Practices/Tooling and Quality/Clippy as Discipline]] — Run Clippy aggressively on every build with intentional configuration; treat it as a discipline, not an occasional cleanup.
- [[Languages/Rust/Practices/Tooling and Quality/CI Quality Bar]] — CI enforces format, lint, build, test, and doc-test on every change; checks here are the floor below which work cannot ship.
- [[Languages/Rust/Practices/Tooling and Quality/Documentation as Truth]] — Document invariants and examples next to the code; Rust doc tests turn documentation into executable, verified truth.

### Languages/Rust/Workspace/

- [[Languages/Rust/Workspace/Workspace Architecture MOC]] — Map of content for shaping a Cargo workspace whose crate boundaries reflect responsibilities that evolve at different speeds.
- [[Languages/Rust/Workspace/Core Principles]] — Four load-bearing rules that drive workspace decisions: pure core, narrow adapters, isolated UI, root-owned configuration.
- [[Languages/Rust/Workspace/Workspace Layout]] — Recommended directory layout for a non-trivial Rust workspace, separating core, adapters, binaries, and reference implementations.
- [[Languages/Rust/Workspace/Cargo Workspace Configuration]] — Use a virtual workspace root that owns members, shared dependencies, lints, and profiles but ships no package of its own.

### Languages/Rust/Workspace/Patterns/

- [[Languages/Rust/Workspace/Patterns/Patterns Index]] — Index of Cargo workspace patterns: each entry is a Cargo expression of a universal architectural principle.
- [[Languages/Rust/Workspace/Patterns/Headless First Cargo Wiring]] — How to express the headless-first principle in a Cargo workspace — separate binary crates for UI vs headless, or feature-gated optional UI.
- [[Languages/Rust/Workspace/Patterns/Reference Path Cargo Wiring]] — Core crate holds the canonical reference, optimized adapter holds the accelerated path, tests cross-validate.
- [[Languages/Rust/Workspace/Patterns/Cargo Binary Strategy]] — Sibling binary crates vs feature-gated binary; how to express the one-binary-or-many decision in Cargo.
- [[Languages/Rust/Workspace/Patterns/Cargo Feature Gating]] — Additive features representing real capabilities; Cargo-specific mechanics for the capability-slices principle.
- [[Languages/Rust/Workspace/Patterns/Shared Dependencies]] — Hoist dependencies into workspace.dependencies only when truly shared; keep crate-local deps local to preserve crate independence.
- [[Languages/Rust/Workspace/Patterns/Workspace Lints and Profiles]] — Profiles and lints belong only in the root manifest; member crates inherit so the quality bar stays uniform.

### Bundles/Rust/

- [[Bundles/Rust/Code-Review]] — Reviewing a PR or diff (Rust Practices Checklist; Tests Should Make Programs Fail; Sharp Oracles; Regression Discipline).
- [[Bundles/Rust/Test-Design]] — Writing tests (Three Levels of Tests; Edge Cases and Properties; Tests Without Heroics; Sharp Oracles; Normal Usage Is Not Testing).
- [[Bundles/Rust/Module-Design]] — Designing a new module or crate (Workspace Architecture MOC; Core Principles; Workspace Layout; Type System as Design Tool; Push Correctness Left).
- [[Bundles/Rust/Error-Handling]] — Handling errors (Result vs Panic; Domain Errors at Boundaries).
- [[Bundles/Rust/API-Design]] — Designing a public API (Predictable APIs; Checked Constructors and Builders; Documentation as Truth).

## How to read a note

- Each note opens with an italicized one-line thesis. If you only need the gist, read that.
- Frontmatter `summary:` is the same idea in a different form (suitable for indexing or RAG).
- "Related" sections at the bottom of notes are the canonical next-hops.
