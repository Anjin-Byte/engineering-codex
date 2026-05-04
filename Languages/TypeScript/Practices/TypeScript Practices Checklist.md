---
title: TypeScript Practices Checklist
tags: [checklist, typescript, code-review, practices]
summary: Condensed code-review checklist of every TypeScript practice in the vault, suitable as a PR self-check.
keywords: [pr checklist, code review, typescript, self check, gate, review checklist]
---

*The compressed form of the TypeScript practices — a single checklist a reviewer or author can run through before opening a PR.*

# TypeScript Practices — Short Checklist

Use this in code review and as a self-check before opening a PR. Each item links to the long-form note for the deeper guidance.

## Foundational
- [ ] **Strict compiler profile applied** — see [[Languages/TypeScript/Practices/Foundational/Strict Compiler Profile]]
- [ ] **Runtime suitability acknowledged** — Node fits this workload's deadlines, or the project has measured worst-case latency. See [[Languages/TypeScript/Practices/Foundational/Runtime Suitability Boundary]]

## Type-driven design
- [ ] **Discriminated unions for state, with exhaustive narrowing** — see [[Languages/TypeScript/Practices/Type-Driven Design/Discriminated Unions and Exhaustive Handling]]
- [ ] **Branded types for domain identifiers** — `OrderId`, `UserId`, `Cents`, `Email` are not bare primitives. See [[Languages/TypeScript/Practices/Type-Driven Design/Branded Types for Domain Identifiers]]
- [ ] **`satisfies` for config tables, not annotations** — see [[Languages/TypeScript/Practices/Type-Driven Design/Satisfies for Configuration Tables]]

## Error handling
- [ ] **Expected failures return `Result<T, E>`** — not throws. See [[Languages/TypeScript/Practices/Error Handling/Result Pattern for Expected Failures]]
- [ ] **Invariant violations crash the process** — fail-fast at process; fail-safe at product. See [[Languages/TypeScript/Practices/Error Handling/Process-Fatal vs Domain-Recoverable]]
- [ ] **External inputs are validated from `unknown`** — schema-parsed at the boundary, before the typed core. See [[Languages/TypeScript/Practices/Error Handling/Runtime Validation at Boundaries]]

## Async and concurrency
- [ ] **No floating promises; every promise awaited or explicit `void`** — see [[Languages/TypeScript/Practices/Async and Concurrency/Floating Promise Hygiene]]
- [ ] **Event-loop delay and ELU exposed as metrics on critical-path services** — see [[Languages/TypeScript/Practices/Async and Concurrency/Event Loop as Correctness Signal]]
- [ ] **Request-scoped state propagated via `AsyncLocalStorage`** — not threaded through every signature. See [[Languages/TypeScript/Practices/Async and Concurrency/Async Context Propagation]]
- [ ] **Every outbound IO has a deadline; fan-out is bounded** — see [[Languages/TypeScript/Practices/Async and Concurrency/Deadline Propagation with AbortSignal]]

## Testing
- [ ] **Test tooling follows the Evidence Ladder** — `node:test` / Vitest for unit, Testcontainers for integration, Playwright for E2E, fast-check for property-based. See [[Languages/TypeScript/Practices/Testing/TypeScript Test Tooling]] and [[Engineering Philosophy/Principles/Evidence Ladder for Testing]]
- [ ] **Adversarial tests, not just happy paths** — see [[Engineering Philosophy/Principles/Tests Should Make Programs Fail]]
- [ ] **Sharp oracles** — exact equality, invariants, or differential refs; never "looks right." See [[Engineering Philosophy/Principles/Sharp Oracles]]

## Tooling and quality
- [ ] **Typed linting via typescript-eslint flat config** — `recommendedTypeChecked` + `strictTypeChecked`. See [[Languages/TypeScript/Practices/Tooling and Quality/Typed Linting with typescript-eslint]]
- [ ] **Reference templates adopted** — TSConfig and ESLint config match (or justifiably diverge from) [[Languages/TypeScript/Practices/Tooling and Quality/Reference TSConfig]] and [[Languages/TypeScript/Practices/Tooling and Quality/Reference ESLint Config]]

## When to consult the long form

If a PR review surfaces friction that doesn't fit the checklist, the corresponding long-form note in [[Languages/TypeScript/Practices/TypeScript Practices MOC]] usually has the deeper guidance. Universal philosophy notes ([[Engineering Philosophy/Principles/Principles Index]]) cover the language-agnostic principles.

## Related

- [[Languages/TypeScript/Practices/TypeScript Practices MOC]]
- [[Languages/TypeScript/AGENTS]]
- [[Languages/Rust/Practices/Rust Practices Checklist]]
- [[Engineering Philosophy/Principles/Principles Index]]
