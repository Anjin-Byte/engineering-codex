---
title: TypeScript Practices MOC
tags: [moc, typescript, practices]
summary: Map of content for TypeScript code-level practices that push correctness left into types, runtime validation, async hygiene, and typed linting.
keywords: [typescript practices, navigation hub, project structure, design index, code level idioms]
---

*A non-trivial TypeScript codebase earns its safety from compile-time strictness, runtime validation at boundaries, and async control-flow hygiene; this MOC indexes the practices that enforce all three.*

# TypeScript Practices — Map of Content

Entry point for understanding **what** the load-bearing practices for TypeScript code are, and **where** to look for any specific code-level decision.

## The thesis

> TypeScript is a *compile-time contract language* layered over JavaScript. Its strictness gives you compile-time guarantees about *type soundness*; it tells you nothing about runtime data or behavioral correctness. Both gaps must be closed deliberately: runtime validation at every external boundary, typed linting for the patterns the compiler can't see, async hygiene for the failure modes the runtime tolerates silently. The practices below close those gaps.

## Foundational reading order

1. [[Languages/TypeScript/Practices/Foundational/Strict Compiler Profile]] — what to enable in `tsconfig.json` and why.
2. [[Languages/TypeScript/Practices/Foundational/Runtime Suitability Boundary]] — when TypeScript / Node fits a problem and when it doesn't.
3. [[Languages/TypeScript/Practices/Tooling and Quality/Typed Linting with typescript-eslint]] — the lint companion to the strictness profile.
4. [[Languages/TypeScript/Practices/Error Handling/Runtime Validation at Boundaries]] — every external input is `unknown` until a schema parses it.

## Topic clusters

### Foundational
- [[Languages/TypeScript/Practices/Foundational/Strict Compiler Profile]] — strict + the eight additional flags.
- [[Languages/TypeScript/Practices/Foundational/Runtime Suitability Boundary]] — Node fits high-consequence backends; not hard real-time.

### Type-driven design
- [[Languages/TypeScript/Practices/Type-Driven Design/Discriminated Unions and Exhaustive Handling]] — tagged unions + `never`-based exhaustiveness.
- [[Languages/TypeScript/Practices/Type-Driven Design/Branded Types for Domain Identifiers]] — opaque `unique symbol` branding for nominal typing.
- [[Languages/TypeScript/Practices/Type-Driven Design/Satisfies for Configuration Tables]] — shape-check without erasing inference.

### Error handling
- [[Languages/TypeScript/Practices/Error Handling/Result Pattern for Expected Failures]] — `Result<T, E>` for expected failures.
- [[Languages/TypeScript/Practices/Error Handling/Process-Fatal vs Domain-Recoverable]] — invariant violations crash the process.
- [[Languages/TypeScript/Practices/Error Handling/Runtime Validation at Boundaries]] — schema-parse `unknown` at every external boundary.

### Async and concurrency
- [[Languages/TypeScript/Practices/Async and Concurrency/Floating Promise Hygiene]] — typed lint enforces no-floating, no-misused, exhaustive switches.
- [[Languages/TypeScript/Practices/Async and Concurrency/Event Loop as Correctness Signal]] — ELU and event-loop delay are first-class signals.
- [[Languages/TypeScript/Practices/Async and Concurrency/Async Context Propagation]] — `AsyncLocalStorage` for request-scoped state.
- [[Languages/TypeScript/Practices/Async and Concurrency/Deadline Propagation with AbortSignal]] — every outbound IO has a deadline; fan-out is bounded.

### Testing
- [[Languages/TypeScript/Practices/Testing/TypeScript Test Tooling]] — which tool implements which rung of the universal Evidence Ladder.

### Tooling and quality
- [[Languages/TypeScript/Practices/Tooling and Quality/Typed Linting with typescript-eslint]] — flat-config typed lint; bundles + explicit overrides.
- [[Languages/TypeScript/Practices/Tooling and Quality/Reference TSConfig]] — drop-in `tsconfig.json` template.
- [[Languages/TypeScript/Practices/Tooling and Quality/Reference ESLint Config]] — drop-in `eslint.config.js` template.

### Performance (advanced / optional)
- [[Languages/TypeScript/Practices/Performance/Allocation Hygiene]] — minimize per-iteration allocation, avoid retention in closures and caches, profile before tuning.
- [[Languages/TypeScript/Practices/Performance/Object Shape Stability]] — V8 hidden-class optimization rewards stable object shapes on hot paths.

## Cross-cutting universal principles (always relevant)

These live under [[Engineering Philosophy/Principles/Principles Index]] and apply to TypeScript code as much as anywhere:

- [[Engineering Philosophy/Principles/Specification Risk vs Type Soundness]] — types prove structure, not behavior.
- [[Engineering Philosophy/Principles/Layered Risk Categories]] — six risk layers; every control is tagged with what it covers.
- [[Engineering Philosophy/Principles/Evidence Ladder for Testing]] — what each test layer proves.
- [[Engineering Philosophy/Principles/Sharp Oracles]] — every test, every layer, needs a sharp oracle.
- [[Engineering Philosophy/Principles/Tests Should Make Programs Fail]] — adversarial posture is layer-independent.

## The condensed rules

1. Strict compiler profile is the floor, not the ceiling.
2. External data is `unknown` until parsed by an executable schema.
3. Discriminated unions and `never`-based exhaustiveness over Boolean flags.
4. Brand domain identifiers; primitives are not domain types.
5. Expected failures are `Result<T, E>`; invariant violations crash the process.
6. Typed lint catches what the compiler can't see — promise hygiene, exhaustiveness, `any` containment.
7. Every outbound IO has a deadline; fan-out is bounded.
8. Event-loop delay and ELU are correctness signals on critical paths.
9. Async context propagates via `AsyncLocalStorage`, not function arguments.
10. Test tooling follows the Evidence Ladder, not framework convenience.

## Related

- [[Languages/TypeScript/AGENTS]] — TypeScript scope agent entry point
- [[Languages/TypeScript/Practices/TypeScript Practices Checklist]] — condensed PR-self-check
- [[Engineering Philosophy/Engineering Philosophy MOC]] — universal principles MOC
- [[Languages/Rust/Practices/Rust Practices MOC]] — sibling per-language MOC
