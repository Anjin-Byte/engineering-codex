---
title: Branded Types for Domain Identifiers
tags: [type-driven-design, typescript, nominal-typing, branding]
summary: Wrap primitive types in `unique symbol`-branded types whenever a value carries domain semantics — `OrderId`, `UserId`, `Cents`, `Email` — so the compiler enforces the distinction.
keywords: [opaque type, nominal type, brand, unique symbol, newtype, type tagging, domain modeling]
---

*A `string` is not a name, an email, an ID, a path, or a token. The type system can be made to know the difference — and should be, because the difference is exactly where bugs live.*

# Branded Types for Domain Identifiers

> **Rule:** any primitive value that carries domain meaning — an identifier, an amount, an email, a path, a token, a unit-bearing quantity — gets its own branded type. The compiler then refuses to silently substitute one for another, even when both are `string` or `number` underneath.

TypeScript is structurally typed: a function expecting `{ x: number }` accepts any object with the right shape. That structural discipline is excellent for most code, but it actively works against you for domain identifiers, because two distinct domain concepts that happen to share a primitive shape become interchangeable. `OrderId` and `UserId` are both strings; `transferFunds(orderIdByMistake, userId)` compiles cleanly, runs, and silently corrupts data.

Branding emulates nominal typing — the discipline that says "two types with the same shape are still distinct" — by attaching a phantom marker to a primitive type that no other code can produce.

## The pattern

```ts
declare const OrderIdBrand: unique symbol;
export type OrderId = string & { readonly [OrderIdBrand]: "OrderId" };

export function asOrderId(s: string): OrderId {
  // Validate as appropriate for the domain (e.g. UUID format)
  return s as OrderId;
}
```

The `unique symbol` and the `readonly` indexed-property field make `OrderId` a strict subtype of `string` that no plain `string` literal satisfies. A function `getOrder(id: OrderId)` will reject a bare string at the call site:

```ts
const id: string = "abc-123";
getOrder(id);                // ✗ Type 'string' is not assignable to 'OrderId'
getOrder(asOrderId(id));     // ✓
getOrder("abc-123" as OrderId); // ✓ but explicit; reviewer can flag
```

The `as OrderId` escape hatch is intentional — branding doesn't try to be a runtime guarantee, it tries to be a *type-system* guarantee that catches accidental misuse. Deliberate casts are still allowed and visible at the call site, where they can be reviewed.

This pattern is community-canonical in TypeScript. It is not built into the language and there is no official TS-team-blessed syntax for it; the [Narrowing handbook chapter](https://www.typescriptlang.org/docs/handbook/2/narrowing.html) covers narrowing and never-based exhaustiveness but does not formalize branding. The vault adopts the pattern despite that, because the alternative (free-string IDs) has a long track record of producing the bugs branding prevents.

## When to brand

Brand a value type when:

- **It identifies something.** `OrderId`, `UserId`, `SessionToken`, `IdempotencyKey`. The type system should refuse to mix them.
- **It carries a unit.** `Cents`, `Milliseconds`, `Bytes`. Mixing a `Cents` value into a `Dollars`-expecting function is the kind of bug branding catches.
- **It carries a constrained format.** `Email`, `Url`, `RFC3339Timestamp`. Branding combined with a checked constructor (see below) means downstream code can trust the format.
- **It carries a security boundary.** `RawHtml` versus `EscapedHtml`. Branding makes "you must escape this before rendering" a compile-time rule.

## Checked constructors

The branding is most powerful when it pairs with validation at construction. The `asOrderId` function above is a degenerate constructor; a real one validates before branding:

```ts
export function asEmail(s: string): Email {
  if (!/^[^@\s]+@[^@\s]+\.[^@\s]+$/.test(s)) {
    throw new Error(`Invalid email: ${s}`);
  }
  return s as Email;
}
```

Now anywhere downstream that accepts `Email` can trust that the string passed validation. This is the TypeScript expression of the Rust pattern in [[Languages/Rust/Practices/Type-Driven Design/Checked Constructors and Builders]] and [[Languages/Rust/Practices/Type-Driven Design/Newtypes and Domain Types]] — the principle is universal; the mechanism differs.

## When *not* to brand

Branding has a real readability cost. Don't apply it to:

- **Internal-only values that no other module sees.** A loop counter that's `number` is fine.
- **Boolean flags.** Branding `true` does nothing useful; if you need nominal booleans, you almost always want a discriminated union (see [[Languages/TypeScript/Practices/Type-Driven Design/Discriminated Unions and Exhaustive Handling]]).
- **Throwaway data that crosses one or two function boundaries.** The cost of declaring the brand outweighs the safety on short-lived values.

## Composing with other practices

- [[Languages/TypeScript/Practices/Error Handling/Runtime Validation at Boundaries]] — when external data enters the system, the schema validator can produce branded values directly. `Email` only exists in the typed core if a schema accepted it.
- [[Languages/TypeScript/Practices/Type-Driven Design/Discriminated Unions and Exhaustive Handling]] — branded types are excellent discriminators (the kind field is itself a domain type).
- [[Languages/TypeScript/Practices/Foundational/Strict Compiler Profile]] — `noUncheckedIndexedAccess` and `exactOptionalPropertyTypes` strengthen branding by closing the loopholes where a primitive could leak in.
- [[Engineering Philosophy/Principles/Specification Risk vs Type Soundness]] — branding closes a class of *type-soundness* bug. The *spec* still needs separate evidence (validation, tests, runtime checks).

## Related

- [[Languages/TypeScript/Practices/Type-Driven Design/Discriminated Unions and Exhaustive Handling]]
- [[Languages/TypeScript/Practices/Error Handling/Runtime Validation at Boundaries]]
- [[Languages/TypeScript/Practices/Foundational/Strict Compiler Profile]]
- [[Languages/Rust/Practices/Type-Driven Design/Newtypes and Domain Types]]
- [[Languages/Rust/Practices/Type-Driven Design/Checked Constructors and Builders]]
