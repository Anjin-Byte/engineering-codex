---
title: Satisfies for Configuration Tables
tags: [type-driven-design, typescript, satisfies, configuration]
summary: Use `satisfies` (TypeScript 4.9+) for shape-checking config tables and lookup maps without erasing literal-type inference.
keywords: [satisfies operator, config validation, literal types, type narrowing, lookup maps, shape checking]
---

*`satisfies` lets you assert "this thing has the right shape" without forcing the inferred type to widen to that shape. Useful exactly for config tables.*

# Satisfies for Configuration Tables

> **Rule:** when a value must conform to a target type but you also want to keep the precise inferred types of its members, use `satisfies` instead of a type annotation. The annotation widens; `satisfies` validates without widening.

The `satisfies` operator was added in [TypeScript 4.9](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-4-9.html). It validates that an expression conforms to a target type and otherwise leaves the expression's inferred type alone. That difference matters for two specific patterns: configuration tables and lookup maps.

## The problem `satisfies` solves

Without `satisfies`, you have two unsatisfying options for typing a config table.

### Option A — type annotation

```ts
type Config = Record<string, { url: string; timeoutMs: number }>;

const services: Config = {
  auth: { url: "https://auth.example", timeoutMs: 5000 },
  billing: { url: "https://billing.example", timeoutMs: 10000 },
};

services.auth;     // type: { url: string; timeoutMs: number } | undefined
services.unknown;  // type: { url: string; timeoutMs: number } | undefined  ← compiles!
```

The annotation forced `services` to be a `Record<string, ...>`, so the compiler now thinks every key might or might not be present, and downstream code can no longer rely on `services.auth` being defined at compile time. Worse, accessing `services.unknown` is allowed.

### Option B — no annotation

```ts
const services = {
  auth: { url: "https://auth.example", timeoutMs: 5000 },
  billing: { url: "https://billing.example", timeoutMs: 10000 },
};
```

Now `services.auth` and `services.billing` are precisely typed, but if a third entry has a typo (`timeoutMS` instead of `timeoutMs`), the compiler doesn't catch it because there's no target shape to validate against.

### `satisfies` is the synthesis

```ts
type ServiceConfig = { url: string; timeoutMs: number };

const services = {
  auth: { url: "https://auth.example", timeoutMs: 5000 },
  billing: { url: "https://billing.example", timeoutMs: 10000 },
} satisfies Record<string, ServiceConfig>;

services.auth;     // type: { url: string; timeoutMs: number }  ← precise
services.unknown;  // ✗ Property 'unknown' does not exist
```

The compiler validates each entry against `ServiceConfig` (catching typos), refuses to widen the keys to a generic `string` index (so unknown access errors), and preserves the literal types of the values.

## Where to reach for it

`satisfies` earns its keep in:

- **Configuration objects** where keys are known at compile time and values must conform to a schema.
- **Route tables, dispatch tables, command registries** — same reason.
- **`as const` arrays of literals** that need to satisfy a target tuple or union, without losing the literal narrowness.
- **Default value tables** where the *literal types* of the defaults inform downstream inference.

Where it doesn't help:

- **Truly dynamic objects** populated from user input or external sources. Those need [[Languages/TypeScript/Practices/Error Handling/Runtime Validation at Boundaries]], not `satisfies`. `satisfies` is a compile-time check; it has no runtime presence.
- **Fields that genuinely *should* widen** to a common type (e.g., a list whose element types you want unified). A type annotation is correct there; `satisfies` would be the wrong tool.

## Combine with `as const` for literal preservation

```ts
const httpStatus = {
  ok: 200,
  notFound: 404,
  serverError: 500,
} as const satisfies Record<string, number>;

type Status = (typeof httpStatus)[keyof typeof httpStatus];
// type: 200 | 404 | 500  ← still literal
```

`as const` freezes the values to literal types; `satisfies` validates them against the schema. Together they give you a config table that is type-safe, key-safe, *and* whose values can drive downstream inference (e.g. `Status` above is a literal union, not a generic `number`).

## A common mistake

Don't write both annotation *and* `satisfies` on the same expression:

```ts
// ✗ Redundant — the annotation already widens, so the satisfies adds nothing useful
const services: Record<string, ServiceConfig> = {
  auth: { url: "https://auth.example", timeoutMs: 5000 },
} satisfies Record<string, ServiceConfig>;
```

The annotation wins; the precise inference is gone. Use `satisfies` *instead of* an annotation, not in addition to it.

## Composing with other practices

- [[Languages/TypeScript/Practices/Type-Driven Design/Branded Types for Domain Identifiers]] — `satisfies Record<OrderId, OrderConfig>` keeps the brand on each key in the resulting type.
- [[Languages/TypeScript/Practices/Type-Driven Design/Discriminated Unions and Exhaustive Handling]] — a config table whose values are members of a discriminated union benefits doubly: the schema is enforced and the dispatch on each value's `kind` field stays exhaustive.
- [[Languages/TypeScript/Practices/Tooling and Quality/Reference TSConfig]] — assumes `target: ES2022` or later, where `satisfies` is supported.

## Related

- [[Languages/TypeScript/Practices/Type-Driven Design/Discriminated Unions and Exhaustive Handling]]
- [[Languages/TypeScript/Practices/Type-Driven Design/Branded Types for Domain Identifiers]]
- [[Languages/TypeScript/Practices/Foundational/Strict Compiler Profile]]
- [[Languages/TypeScript/Practices/Tooling and Quality/Reference TSConfig]]
