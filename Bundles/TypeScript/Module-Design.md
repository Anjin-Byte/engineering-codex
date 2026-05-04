---
title: Module Design Bundle
tags: [bundle, architecture, typescript]
summary: "Pre-bundled module design for TypeScript: the Practices MOC, strict compiler profile, the universal architectural core, the risk surface, plus the type-driven and runtime-validation disciplines that decide what each boundary guarantees."
source_trigger: "Designing a new module or service"
bundles: [TypeScript Practices MOC, Strict Compiler Profile, Architectural Core Principles, Layered Risk Categories, Discriminated Unions and Exhaustive Handling, Runtime Validation at Boundaries]
---

*The through-line: a new TypeScript module earns its boundary by reflecting a deployable unit, with strict compile-time discipline, runtime validation at its edges, and types that make illegal states unrepresentable.*

# Module Design Bundle (TypeScript)

Read this bundle when designing a new TypeScript module or service. It bundles the Practices MOC for orientation, the strict compiler profile for compile-time discipline, the universal architectural principles for shape, the layered risk model for what controls cover what, plus discriminated unions and runtime validation as the type-driven and runtime expressions of "make illegal states unrepresentable."

---

## TypeScript Practices MOC (orientation)

*A non-trivial TypeScript codebase earns its safety from compile-time strictness, runtime validation at boundaries, and async control-flow hygiene; this MOC indexes the practices that enforce all three.*

> TypeScript is a *compile-time contract language* layered over JavaScript. Its strictness gives you compile-time guarantees about *type soundness*; it tells you nothing about runtime data or behavioral correctness. Both gaps must be closed deliberately.

### Foundational reading order

1. [[Languages/TypeScript/Practices/Foundational/Strict Compiler Profile]] — what to enable in `tsconfig.json` and why.
2. [[Languages/TypeScript/Practices/Foundational/Runtime Suitability Boundary]] — when TS/Node fits a problem.
3. [[Languages/TypeScript/Practices/Tooling and Quality/Typed Linting with typescript-eslint]] — the lint companion.
4. [[Languages/TypeScript/Practices/Error Handling/Runtime Validation at Boundaries]] — schema-parse `unknown` at every boundary.

### Topic clusters

- **Foundational:** Strict Compiler Profile, Runtime Suitability Boundary.
- **Type-driven design:** Discriminated Unions, Branded Types, Satisfies for Configuration Tables.
- **Error handling:** Result Pattern, Process-Fatal vs Domain-Recoverable, Runtime Validation at Boundaries.
- **Async and concurrency:** Floating Promise Hygiene, Event Loop as Correctness Signal, Async Context Propagation, Deadline Propagation with AbortSignal.
- **Testing:** TypeScript Test Tooling.
- **Tooling and quality:** Typed Linting, Reference TSConfig, Reference ESLint Config.

*Source: [[Languages/TypeScript/Practices/TypeScript Practices MOC]]*

---

## Strict Compiler Profile

*The strict baseline is not enough on its own. Add the flags that catch the categories of bug `strict` does not.*

> Every TypeScript project enables `strict: true` plus eight additional flags. Each one closes a class of bug `strict` still permits, costs almost nothing, and prevents an entire failure mode at compile time.

The required additions: `noUncheckedIndexedAccess`, `exactOptionalPropertyTypes`, `useUnknownInCatchVariables`, `noImplicitOverride`, `noFallthroughCasesInSwitch`, `noEmitOnError`, `noImplicitReturns`, `noPropertyAccessFromIndexSignature`. Plus `skipLibCheck: false` for CI.

Each is justified by the canonical TSConfig handbook page. The combined effect: indexed access becomes safe, optional properties distinguish "missing" from "undefined," catch variables narrow before use, override drift is a compile error, switch fallthrough is explicit, type errors halt the build cold.

The drop-in template: [[Languages/TypeScript/Practices/Tooling and Quality/Reference TSConfig]].

### What this profile does *not* do

Strict-mode discipline closes type-soundness loopholes. It does not address [[Engineering Philosophy/Principles/Specification Risk vs Type Soundness]] — whether the types you wrote down are the right invariants. For that, you need runtime validation at boundaries (see below), property-based tests, and observability.

*Source: [[Languages/TypeScript/Practices/Foundational/Strict Compiler Profile]]*

---

## Architectural Core Principles

*Four rules govern any decision about project shape; treat any future change that violates one of them as a smell to revisit.*

Four principles drive project-shape decisions regardless of language. Apply them to TypeScript modules and services.

### 1. Keep external-system boundaries narrow

Core algorithms should not know about the types of any external system (a database driver, a heavy SDK, a third-party HTTP client) unless absolutely necessary.

**Better shape:**
- **core module(s)** — domain types, algorithms, deterministic transformations
- **adapter module(s)** — one per external system, with its own narrow public API
- **edge module** — argument parsing, file I/O, request/response handling
- **optional UI module** — interactive front-end, isolated from core

### 2. Treat headless as the default, interactive UI as an add-on

A library-first design lets the same code be embedded in CLIs, services, batch jobs, CI pipelines, and tests. Interactive UI is layered on top — never required for the core to work.

### 3. Make the project shape reflect deployable units

If something can be built, tested, or shipped independently, it deserves its own module/package boundary. In TypeScript, this is npm packages, project references, or workspace members. See [[Languages/TypeScript/Workspace/Project Layout]].

### 4. Always keep a reference implementation alongside any optimized path

For any algorithm with an optimized implementation (SIMD, GPU, FFI, hand-rolled), maintain a reference implementation in pure deterministic code as the correctness oracle.

*Source: [[Engineering Philosophy/Principles/Architectural Core Principles]]*

---

## Layered Risk Categories

*Six risk layers: specification, type model, runtime, concurrency, deployment, supply chain. Every control is tagged with the layer it mitigates, or it is decoration.*

> The risk surface of any non-trivial system can be exhausted by six layers. A standard, audit, or control set that does not say *which layer* it covers is not earning its place.

The six layers:

1. **Specification** — does the system do the right thing in principle?
2. **Type model** — does the code structurally express the specification?
3. **Runtime** — does the running program behave as the code says, given the runtime's actual semantics?
4. **Concurrency / context** — does the program preserve correctness across concurrent operations and async boundaries?
5. **Deployment** — does the change reach production safely, with rollback?
6. **Supply chain** — is the code that runs in production the code we wrote?

A control earns its place by being tagged with the layer or layers it mitigates. *Untagged controls are decoration*: they pass audits and catch nothing.

When designing a new TypeScript module, walk the layers explicitly:

- Specification: what does this module promise? Where is that promise documented?
- Type model: do the types encode the promise, or just allow it?
- Runtime: does the implementation honor the types under realistic load (event-loop, GC, async hygiene)?
- Concurrency: how does request-scoped context propagate? Are deadlines honored?
- Deployment: how is this module released? Canary? SLO-gated?
- Supply chain: what does this module depend on? Is the lockfile under review?

*Source: [[Engineering Philosophy/Principles/Layered Risk Categories]]*

---

## Discriminated Unions and Exhaustive Handling

*Tag every union with a literal discriminator; check exhaustiveness with `never`.*

> Any place the code needs to dispatch on "what kind of thing this is," express the kinds as a discriminated union and check exhaustiveness with the `never` type.

```ts
type Shape =
  | { kind: "circle"; radius: number }
  | { kind: "rectangle"; width: number; height: number };

function area(shape: Shape): number {
  switch (shape.kind) {
    case "circle":    return Math.PI * shape.radius ** 2;
    case "rectangle": return shape.width * shape.height;
    default: {
      const _exhaustive: never = shape;
      return _exhaustive;
    }
  }
}
```

Adding a new variant becomes a *compile error* at every consumer that didn't handle it. That is the protection.

This is the TypeScript application of [[Languages/Rust/Practices/Type-Driven Design/Make Invalid States Unrepresentable]] — the same principle, expressed in TypeScript's structural type system.

### Why Boolean flags are worse

`{ isPaid: boolean; isShipped: boolean }` permits four combinations including `{ isPaid: false, isShipped: true }` (shipped without payment) — a state the domain forbids. Discriminated unions make illegal states inexpressible.

```ts
type Order =
  | { kind: "draft" }
  | { kind: "awaiting_payment"; lineItems: LineItem[] }
  | { kind: "paid"; lineItems: LineItem[]; paidAt: Date }
  | { kind: "shipped"; lineItems: LineItem[]; paidAt: Date; shippedAt: Date };
```

Now there is no `Order` value where `shippedAt` exists but `paidAt` does not.

*Source: [[Languages/TypeScript/Practices/Type-Driven Design/Discriminated Unions and Exhaustive Handling]]*

---

## Runtime Validation at Boundaries

*Types are a compile-time guarantee. The runtime sees no types — only bytes, JSON, env strings. Convert bytes to typed values explicitly, at the boundary, with a schema.*

> Every external input enters the program as `unknown` and is parsed against an executable schema into a typed value before the typed core sees it. Applies to HTTP bodies, message-bus payloads, environment variables, CLI arguments, persisted state, feature-flag payloads, file imports.

The compiler can prove that values of type `User` always have the fields `User` declares. It cannot prove that what `JSON.parse(req.body)` returns is a `User`. The two facts are independent.

```ts
const userSchema: Schema<User> = /* ... */;

async function getUser(req: Request): Promise<Result<User, ParseError>> {
  const raw: unknown = await req.json();
  const parsed = userSchema.parse(raw);
  if (!parsed.ok) return err({ kind: "invalid_payload", details: parsed.error });
  return ok(parsed.value);
}
```

### Library landscape

- **Ajv** (JSON Schema, cross-language) — pick when contracts must be cross-language, published externally, or throughput-critical.
- **Zod** (TS-first, fluent) — pick for internal services where developer ergonomics matter most.
- **Valibot** (tree-shakeable) — pick when bundle size matters.
- **ArkType** (TS-syntax inference) — pick when type fidelity dominates.

For external contracts, default to Ajv. For internal services, default to Zod. Don't use three different libraries in one project.

### Why "unknown first"

Externally-sourced data should be `unknown` because the type system should refuse to let any code touch the value until the program has *proven* what it is. Casting `as User` is a lie the runtime is under no obligation to honor.

*Source: [[Languages/TypeScript/Practices/Error Handling/Runtime Validation at Boundaries]]*

---

## Related bundles

- [[Bundles/TypeScript/Error-Handling]] — what errors the new module's boundary returns
- [[Bundles/TypeScript/Test-Design]] — how the new module gets its test coverage without heroics
- [[Bundles/TypeScript/Code-Review]] — review checks for the new module's PR
- [[Bundles/Universal/Written-Analysis]] — the design doc that should accompany a non-trivial new module
