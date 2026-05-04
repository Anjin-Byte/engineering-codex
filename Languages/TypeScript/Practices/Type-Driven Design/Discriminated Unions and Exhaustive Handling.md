---
title: Discriminated Unions and Exhaustive Handling
tags: [type-driven-design, typescript, narrowing, discriminated-union]
summary: Prefer discriminated unions over Boolean flags or shape-overlap; require `never`-based exhaustiveness on every union narrowing site so the compiler tracks every case.
keywords: [tagged union, narrowing, never type, exhaustive switch, kind field, assertNever]
---

*Tag every union with a literal discriminator; check exhaustiveness with `never`. The compiler now tells you when a new variant lands and you forgot to handle it.*

# Discriminated Unions and Exhaustive Handling

> **Rule:** any place the code needs to dispatch on "what kind of thing this is," express the kinds as a discriminated union and check exhaustiveness with the `never` type. The compiler then catches every place a new variant was not handled.

A discriminated union pairs a *common literal-type field* (the discriminator, conventionally `kind` or `type`) across each variant with variant-specific shape. With control-flow narrowing, TypeScript will refine the type inside each branch automatically, and an unhandled branch can be turned into a compile error.

This is the [TypeScript handbook's canonical pattern for narrowing](https://www.typescriptlang.org/docs/handbook/2/narrowing.html), including the exhaustiveness check via `never`.

## Shape

```ts
type Shape =
  | { kind: "circle"; radius: number }
  | { kind: "rectangle"; width: number; height: number }
  | { kind: "triangle"; base: number; height: number };

function area(shape: Shape): number {
  switch (shape.kind) {
    case "circle":
      return Math.PI * shape.radius ** 2;
    case "rectangle":
      return shape.width * shape.height;
    case "triangle":
      return 0.5 * shape.base * shape.height;
    default: {
      const _exhaustive: never = shape;
      return _exhaustive;
    }
  }
}
```

The `_exhaustive: never = shape` line is the load-bearing one. Inside the `default` branch, `shape` should have been narrowed to `never` if every case was handled. If a future commit adds `{ kind: "ellipse"; ... }` to `Shape` and forgets to handle it in `area`, the assignment fails to type-check and the build breaks. That is the protection.

## Why Boolean flags are worse

Compare:

```ts
// Worse — two booleans, four states, two of which are nonsensical
interface Order {
  isPaid: boolean;
  isShipped: boolean;
}
```

The four combinations include `{ isPaid: false, isShipped: true }` (shipped without payment) — a state the domain forbids but the type permits. Every consumer must write defensive code for it, and someone eventually won't.

The discriminated-union form makes the illegal state inexpressible:

```ts
// Better — the type lists exactly the legal states
type Order =
  | { kind: "draft" }
  | { kind: "awaiting_payment"; lineItems: LineItem[] }
  | { kind: "paid"; lineItems: LineItem[]; paidAt: Date }
  | { kind: "shipped"; lineItems: LineItem[]; paidAt: Date; shippedAt: Date };
```

Now the compiler enforces both the *set of legal states* and the *fields that must accompany each state*. There is no `Order` value where `shippedAt` exists but `paidAt` does not.

This is the TypeScript application of [[Languages/Rust/Practices/Type-Driven Design/Make Invalid States Unrepresentable]] — the same principle, expressed in TypeScript's structural type system rather than Rust's algebraic data types.

## When to add a discriminator

The pattern is right when:

- The code dispatches on which variant something is (a `switch`, an `if/else if` chain, a method-table lookup).
- Different variants carry different fields.
- Adding a new variant in the future is plausible, and forgetting to handle it would be a bug rather than a no-op.

It is overkill when:

- The variants are pure constants with no associated data and no dispatch logic. A plain `'small' | 'medium' | 'large'` literal type is enough; the discriminator is the value itself.
- The "kinds" are a polymorphism boundary the project deliberately uses class-based subtyping for (rare in TypeScript, but it happens).

## The `assertNever` helper

Many projects extract the exhaustiveness check into a helper:

```ts
export function assertNever(_: never): never {
  throw new Error("Unreachable: non-exhaustive switch");
}
```

Use site:

```ts
default:
  return assertNever(shape);
```

The runtime `throw` is defensive — if the type system is bypassed (a value with a `kind` the compiler didn't know about, perhaps from JSON), the code fails fast at the dispatch site rather than producing wrong output. This composes with [[Languages/TypeScript/Practices/Error Handling/Process-Fatal vs Domain-Recoverable]]: an unreachable code path *should* be process-fatal because it indicates the type system's invariants have been violated.

## Composing with other practices

- [[Languages/TypeScript/Practices/Type-Driven Design/Branded Types for Domain Identifiers]] — when a discriminator is itself an opaque domain type rather than a free string, the dispatch is even more constrained.
- [[Languages/TypeScript/Practices/Error Handling/Result Pattern for Expected Failures]] — `Result<T, E>` is itself a discriminated union; the same exhaustiveness discipline applies to consumers of `Result`.
- [[Languages/TypeScript/Practices/Tooling and Quality/Typed Linting with typescript-eslint]] — the `switch-exhaustiveness-check` rule enforces this discipline automatically across the codebase.
- [[Engineering Philosophy/Principles/Specification Risk vs Type Soundness]] — the discriminated union *encodes* the spec for "which states are legal"; the type system then proves the code matches that encoding. The encoding's correctness is still your responsibility.

## Related

- [[Languages/TypeScript/Practices/Type-Driven Design/Branded Types for Domain Identifiers]]
- [[Languages/TypeScript/Practices/Error Handling/Result Pattern for Expected Failures]]
- [[Languages/TypeScript/Practices/Tooling and Quality/Typed Linting with typescript-eslint]]
- [[Languages/Rust/Practices/Type-Driven Design/Make Invalid States Unrepresentable]]
- [[Languages/Rust/Practices/Type-Driven Design/State Transition Types]]
