---
title: AGENTS — TypeScript Scope
tags: [meta, agent-entry-point, typescript]
summary: TypeScript-specific triggers and full note index for Languages/TypeScript/. Loaded only when the project uses TypeScript.
---

*TypeScript-specific guidance: code-level practices for projects whose primary language is TypeScript. Load alongside the universal Engineering Philosophy scope.*

# AGENTS — TypeScript Scope

This file routes agents through the TypeScript portion of the vault: code-level practices under [[Languages/TypeScript/Practices/TypeScript Practices MOC]]. Triggers below cross-reference universal philosophy notes (always loaded alongside this scope) where the principle is universal even though the application is TypeScript-specific.

## When to consult what (TypeScript triggers)

Group by trigger.

- **Reviewing a PR or diff** → [[Languages/TypeScript/Practices/TypeScript Practices Checklist]]; [[Engineering Philosophy/Principles/Tests Should Make Programs Fail]]; [[Engineering Philosophy/Principles/Sharp Oracles]]; [[Engineering Philosophy/Principles/Regression Discipline]]; [[Engineering Philosophy/Principles/Evidence Ladder for Testing]]. Bundle: [[Bundles/TypeScript/Code-Review]].
- **Designing a new module or service** → [[Languages/TypeScript/Practices/Foundational/Strict Compiler Profile]]; [[Engineering Philosophy/Principles/Architectural Core Principles]]; [[Engineering Philosophy/Principles/Layered Risk Categories]]; [[Languages/TypeScript/Practices/Type-Driven Design/Discriminated Unions and Exhaustive Handling]]; [[Languages/TypeScript/Practices/Error Handling/Runtime Validation at Boundaries]]. Bundle: [[Bundles/TypeScript/Module-Design]].
- **Writing tests** → [[Languages/TypeScript/Practices/Testing/TypeScript Test Tooling]]; [[Engineering Philosophy/Principles/Evidence Ladder for Testing]]; [[Engineering Philosophy/Principles/Sharp Oracles]]; [[Engineering Philosophy/Principles/Tests Should Make Programs Fail]]; [[Engineering Philosophy/Principles/Normal Usage Is Not Testing]]. Bundle: [[Bundles/TypeScript/Test-Design]].
- **Handling errors and runtime validation** → [[Languages/TypeScript/Practices/Error Handling/Result Pattern for Expected Failures]]; [[Languages/TypeScript/Practices/Error Handling/Process-Fatal vs Domain-Recoverable]]; [[Languages/TypeScript/Practices/Error Handling/Runtime Validation at Boundaries]]; [[Engineering Philosophy/Principles/Specification Risk vs Type Soundness]]. Bundle: [[Bundles/TypeScript/Error-Handling]].
- **Modeling a domain value or state machine** → [[Languages/TypeScript/Practices/Type-Driven Design/Discriminated Unions and Exhaustive Handling]]; [[Languages/TypeScript/Practices/Type-Driven Design/Branded Types for Domain Identifiers]]; [[Languages/TypeScript/Practices/Type-Driven Design/Satisfies for Configuration Tables]]
- **Async control-flow hygiene** → [[Languages/TypeScript/Practices/Async and Concurrency/Floating Promise Hygiene]]; [[Languages/TypeScript/Practices/Async and Concurrency/Async Context Propagation]]; [[Languages/TypeScript/Practices/Async and Concurrency/Deadline Propagation with AbortSignal]]; [[Languages/TypeScript/Practices/Async and Concurrency/Event Loop as Correctness Signal]]
- **Configuring TypeScript or ESLint** → [[Languages/TypeScript/Practices/Tooling and Quality/Reference TSConfig]]; [[Languages/TypeScript/Practices/Tooling and Quality/Reference ESLint Config]]; [[Languages/TypeScript/Practices/Foundational/Strict Compiler Profile]]; [[Languages/TypeScript/Practices/Tooling and Quality/Typed Linting with typescript-eslint]]
- **Designing a public API** → [[Languages/TypeScript/Practices/Type-Driven Design/Branded Types for Domain Identifiers]]; [[Languages/TypeScript/Practices/Type-Driven Design/Discriminated Unions and Exhaustive Handling]]; [[Languages/TypeScript/Practices/Error Handling/Result Pattern for Expected Failures]]; [[Languages/TypeScript/Practices/Error Handling/Runtime Validation at Boundaries]]
- **Refactoring a long function or tangled module** → [[Languages/TypeScript/Practices/Type-Driven Design/Discriminated Unions and Exhaustive Handling]]; [[Languages/TypeScript/Practices/Error Handling/Result Pattern for Expected Failures]]; [[Engineering Philosophy/Principles/Architectural Core Principles]]
- **Evaluating runtime suitability for a project** → [[Languages/TypeScript/Practices/Foundational/Runtime Suitability Boundary]]; [[Languages/TypeScript/Practices/Async and Concurrency/Event Loop as Correctness Signal]]; [[Engineering Philosophy/Principles/Layered Risk Categories]]; [[Engineering Philosophy/Principles/Error Budgets and Reliability Policy]]
- **Observability and operations on a TS service** → [[Languages/TypeScript/Practices/Async and Concurrency/Event Loop as Correctness Signal]]; [[Languages/TypeScript/Practices/Async and Concurrency/Async Context Propagation]]; [[Engineering Philosophy/Principles/Error Budgets and Reliability Policy]]; [[Engineering Philosophy/Principles/Staged Canary Deployment]]
- **Configuring TypeScript project layout / monorepo / build** → [[Languages/TypeScript/Workspace/TypeScript Workspace Architecture MOC]]; [[Languages/TypeScript/Workspace/Project Layout]]; [[Languages/TypeScript/Workspace/TypeScript Execution]]; [[Languages/TypeScript/Workspace/Bundling Decision]]; [[Engineering Philosophy/Principles/Architectural Core Principles]]
- **Setting up a release pipeline / supply-chain controls** → [[Languages/TypeScript/Workspace/Release Engineering/Release Gate Pipeline]]; [[Languages/TypeScript/Workspace/Release Engineering/npm Lockfile and Install Discipline]]; [[Languages/TypeScript/Workspace/Release Engineering/Provenance and SBOM]]; [[Languages/TypeScript/Workspace/Release Engineering/Permission Model Adoption]]; [[Engineering Philosophy/Principles/Staged Canary Deployment]]; [[Engineering Philosophy/Principles/Error Budgets and Reliability Policy]]; [[Engineering Philosophy/Principles/Configuration as Code Review]]
- **Performance tuning on a hot path (advanced / optional)** → [[Languages/TypeScript/Practices/Performance/Allocation Hygiene]]; [[Languages/TypeScript/Practices/Performance/Object Shape Stability]]; [[Languages/TypeScript/Practices/Async and Concurrency/Event Loop as Correctness Signal]]; [[Engineering Philosophy/Principles/Reference Implementation as Oracle]]

## Full note index (Languages/TypeScript/)

Every note in the TypeScript scope, grouped by folder, with its `summary:` text.

### Languages/TypeScript/Practices/

- [[Languages/TypeScript/Practices/TypeScript Practices MOC]] — Map of content for TypeScript code-level practices that push correctness left into types, runtime validation, async hygiene, and typed linting.
- [[Languages/TypeScript/Practices/TypeScript Practices Checklist]] — Condensed code-review checklist of every TypeScript practice in the vault, suitable as a PR self-check.

### Languages/TypeScript/Practices/Foundational/

- [[Languages/TypeScript/Practices/Foundational/Strict Compiler Profile]] — Enable `strict` plus the additional flags that close type-system loopholes a strict baseline doesn't catch — each flag is a class of bug the compiler can prevent for free.
- [[Languages/TypeScript/Practices/Foundational/Runtime Suitability Boundary]] — TypeScript / Node fits high-consequence backend and control-plane services; it is unsuitable for hard real-time actuation loops unless deadline compliance is independently proven under worst-case load.

### Languages/TypeScript/Practices/Type-Driven Design/

- [[Languages/TypeScript/Practices/Type-Driven Design/Discriminated Unions and Exhaustive Handling]] — Prefer discriminated unions over Boolean flags or shape-overlap; require `never`-based exhaustiveness on every union narrowing site so the compiler tracks every case.
- [[Languages/TypeScript/Practices/Type-Driven Design/Branded Types for Domain Identifiers]] — Wrap primitive types in `unique symbol`-branded types whenever a value carries domain semantics — `OrderId`, `UserId`, `Cents`, `Email` — so the compiler enforces the distinction.
- [[Languages/TypeScript/Practices/Type-Driven Design/Satisfies for Configuration Tables]] — Use `satisfies` (TypeScript 4.9+) for shape-checking config tables and lookup maps without erasing literal-type inference.

### Languages/TypeScript/Practices/Error Handling/

- [[Languages/TypeScript/Practices/Error Handling/Result Pattern for Expected Failures]] — Expected failures return `Result<T, E>` (or library equivalent); `throw` is reserved for invariant violations. The compiler then forces the caller to handle the failure case.
- [[Languages/TypeScript/Practices/Error Handling/Process-Fatal vs Domain-Recoverable]] — Invariant violations and unhandled async failures should crash the process (fail-fast); expected failures are typed and handled (fail-safe). Process-fatal is a stability mechanism, not a cleanliness one.
- [[Languages/TypeScript/Practices/Error Handling/Runtime Validation at Boundaries]] — Every external input is validated from `unknown` against an executable schema before entering the typed core — HTTP bodies, message-bus payloads, environment variables, CLI arguments, persisted state.

### Languages/TypeScript/Practices/Async and Concurrency/

- [[Languages/TypeScript/Practices/Async and Concurrency/Floating Promise Hygiene]] — Typed linting must enforce `no-floating-promises`, `no-misused-promises`, and `switch-exhaustiveness-check` — these rules need full type information and untyped linting cannot catch them.
- [[Languages/TypeScript/Practices/Async and Concurrency/Event Loop as Correctness Signal]] — For high-consequence Node services, treat event-loop delay and event-loop utilization as first-class correctness signals — not just performance metrics. A saturated loop means deadlines, timers, and cancellations stop being honored.
- [[Languages/TypeScript/Practices/Async and Concurrency/Async Context Propagation]] — Use `AsyncLocalStorage` for request-scoped state — correlation IDs, authorization context, tenancy, idempotency keys — so context survives `await` boundaries without being threaded through every signature.
- [[Languages/TypeScript/Practices/Async and Concurrency/Deadline Propagation with AbortSignal]] — Every outbound IO accepts an `AbortSignal` and runs under a deadline; unbounded fan-out is forbidden — bounded concurrency is mandatory.

### Languages/TypeScript/Practices/Testing/

- [[Languages/TypeScript/Practices/Testing/TypeScript Test Tooling]] — Each rung of the universal Evidence Ladder for Testing has a canonical TypeScript tool — node:test for unit, Testcontainers for integration, Playwright for E2E, fast-check for property-based, Jazzer.js for fuzzing, Stryker for mutation.

### Languages/TypeScript/Practices/Tooling and Quality/

- [[Languages/TypeScript/Practices/Tooling and Quality/Typed Linting with typescript-eslint]] — Adopt typescript-eslint flat config with `recommendedTypeChecked`, `strictTypeChecked`, and `stylisticTypeChecked` plus explicit rules for promise hygiene and exhaustiveness; the slowdown is the cost of catching what untyped lint cannot.
- [[Languages/TypeScript/Practices/Tooling and Quality/Reference TSConfig]] — A drop-in `tsconfig.json` template for critical-path TypeScript projects, with every option justified — strict compiler profile, deterministic emit, no-emit-on-error, modern target.
- [[Languages/TypeScript/Practices/Tooling and Quality/Reference ESLint Config]] — A drop-in ESLint flat-config template for TypeScript projects, using typescript-eslint typed linting with `recommendedTypeChecked`, `strictTypeChecked`, and `stylisticTypeChecked` plus explicit safety overrides.

### Languages/TypeScript/Practices/Performance/ (advanced / optional)

- [[Languages/TypeScript/Practices/Performance/Allocation Hygiene]] — JavaScript is garbage-collected; the only performance lever is allocation behavior. For hot paths, minimize per-iteration allocation, avoid retention in closures or caches, and measure the heap before changing flags.
- [[Languages/TypeScript/Practices/Performance/Object Shape Stability]] — V8's hidden-class optimization rewards objects whose property set, order, and types are stable across instances. Polymorphic shape on hot paths costs measurable performance — most teams will never need to optimize for this; the few that do should know the model.

### Languages/TypeScript/Workspace/

- [[Languages/TypeScript/Workspace/TypeScript Workspace Architecture MOC]] — Map of content for shaping a TypeScript project — directory layout, execution mode, bundling decisions, and the release-engineering pipeline that ships it.
- [[Languages/TypeScript/Workspace/Project Layout]] — A non-trivial TypeScript project's directory shape encodes its deployable-unit boundaries — single-package, npm workspaces, or project-references monorepo. Pick by where the build, test, and ship boundaries actually fall.
- [[Languages/TypeScript/Workspace/TypeScript Execution]] — Precompile to JavaScript with `tsc` for production. Reserve `ts-node` and `--experimental-transform-types` for local scripts. Shipping TypeScript through a runtime stripper in production is a stability risk that compile-then-deploy avoids.
- [[Languages/TypeScript/Workspace/Bundling Decision]] — Long-lived services should not bundle. Reach for esbuild only when single-file artifacts, cold-start time, or browser-shipping constraints justify it; source maps and debugger UX are the hidden cost teams underestimate.

### Languages/TypeScript/Workspace/Release Engineering/

- [[Languages/TypeScript/Workspace/Release Engineering/Release Gate Pipeline]] — A critical-path TypeScript service merges to production only when typecheck, typed lint, tests, fuzzing regression, SAST, dependency audit, provenance, SBOM, canary, and post-canary SLO watch are all green — the pipeline is the policy.
- [[Languages/TypeScript/Workspace/Release Engineering/npm Lockfile and Install Discipline]] — Commit and review the lockfile; use `npm ci` (not `npm install`) in CI; gate on `npm audit` with a tiered policy and an explicit override process — installs must be deterministic and attributable.
- [[Languages/TypeScript/Workspace/Release Engineering/Provenance and SBOM]] — Critical-path packages publish with npm provenance attestations and emit an SBOM. CycloneDX is operational/security-first; SPDX is compliance/licensing-first with ISO standing — many teams emit both.
- [[Languages/TypeScript/Workspace/Release Engineering/Permission Model Adoption]] — Node's `--permission` flag restricts filesystem, network, child-process, worker, native-addon, WASI, and inspector access at the process level. Stable as of Node v22.13.0 / v23.5.0; defense in depth, not a sandbox against malicious code.

### Bundles/TypeScript/

- [[Bundles/TypeScript/Code-Review]] — Reviewing a PR or diff (TypeScript Practices Checklist; Tests Should Make Programs Fail; Sharp Oracles; Regression Discipline; Evidence Ladder for Testing).
- [[Bundles/TypeScript/Test-Design]] — Writing tests (TypeScript Test Tooling; Evidence Ladder for Testing; Sharp Oracles; Tests Should Make Programs Fail; Normal Usage Is Not Testing).
- [[Bundles/TypeScript/Module-Design]] — Designing a new module or service (TypeScript Practices MOC; Strict Compiler Profile; Architectural Core Principles; Layered Risk Categories; Discriminated Unions and Exhaustive Handling; Runtime Validation at Boundaries).
- [[Bundles/TypeScript/Error-Handling]] — Handling errors and runtime validation (Result Pattern for Expected Failures; Process-Fatal vs Domain-Recoverable; Runtime Validation at Boundaries).

## How to read a note

- Each note opens with an italicized one-line thesis. If you only need the gist, read that.
- Frontmatter `summary:` is the same idea in a different form (suitable for indexing or RAG).
- "Related" sections at the bottom of notes are the canonical next-hops.
