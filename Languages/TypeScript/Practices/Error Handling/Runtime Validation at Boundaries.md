---
title: Runtime Validation at Boundaries
tags: [error-handling, typescript, validation, schema, boundary]
summary: Every external input is validated from `unknown` against an executable schema before entering the typed core — HTTP bodies, message-bus payloads, environment variables, CLI arguments, persisted state.
keywords: [parse don't validate, schema validation, ajv, zod, valibot, arktype, json schema, boundary parsing, unknown first]
---

*Types are a compile-time guarantee. The runtime sees no types — only bytes, JSON, env strings. Convert bytes to typed values explicitly, at the boundary, with a schema.*

# Runtime Validation at Boundaries

> **Rule:** every external input enters the program as `unknown` and is parsed against an executable schema into a typed value before the typed core sees it. The typed core then never deals with raw external data. This applies to HTTP bodies, message-bus payloads, environment variables, CLI arguments, feature-flag payloads, file imports, persisted state on read-back, and anything else that originates outside the running program.

The compiler can prove that values of type `User` always have the fields `User` declares. It cannot prove that what `JSON.parse(req.body)` returns is a `User`. The two facts are independent, and conflating them is the most common defect class in real-world TypeScript codebases.

The discipline is small but absolute: external bytes → `unknown` → schema parse → typed value. Skipping the parse and casting `as User` produces a value that the type system claims is a `User` but the runtime may have given a string, a partial object, a string-encoded number, or a deliberately hostile payload.

## The pattern

```ts
type User = { id: UserId; email: Email; createdAt: Date };
const userSchema: Schema<User> = /* ... */;

async function getUser(req: Request): Promise<Result<User, ParseError>> {
  const raw: unknown = await req.json();
  const parsed = userSchema.parse(raw);
  if (!parsed.ok) return err({ kind: "invalid_payload", details: parsed.error });
  return ok(parsed.value);
}
```

The shape of `parse` depends on the schema library, but the contract is the same: take `unknown`, return `Result<T, ParseError>`. Throwing on parse failure is also acceptable when the failure is clearly a programmer or hostile-input bug rather than an expected outcome — see [[Languages/TypeScript/Practices/Error Handling/Process-Fatal vs Domain-Recoverable]].

This pattern composes with [[Languages/TypeScript/Practices/Type-Driven Design/Branded Types for Domain Identifiers]]: the schema for `User` produces a branded `UserId` and `Email`, so the typed core gets domain-typed values, not raw strings. It composes with [[Languages/TypeScript/Practices/Error Handling/Result Pattern for Expected Failures]]: parse failure is exactly the kind of expected failure that belongs in `Result`.

## Why "unknown first"

TypeScript provides `unknown` as the safer counterpart to `any`. `any` lets you do anything to the value without compiler complaint; `unknown` forces narrowing before any access. Externally-sourced data should be `unknown` for exactly that reason: the type system should refuse to let any code touch the value until the program has *proven* what it is.

A common anti-pattern:

```ts
// ✗ The type assertion is a lie
async function badGetUser(req: Request): Promise<User> {
  const body = await req.json() as User;
  return body;
}
```

The cast tells the type system "trust me, this is a `User`." The runtime is under no obligation to honor that. If the request body is `{ "email": null }`, the cast still succeeds, the function returns "a `User`," and `user.email.split('@')` throws at the next step. The type system prevented none of this.

The corrected version above forces the schema to do the proof.

## Choosing a schema library

Several mature options exist. The vault treats this as a project decision, not a vault-prescribed one, but the trade-offs are real.

### Ajv — JSON Schema and JTD

[Ajv](https://ajv.js.org/) is the most-used Node.js JSON Schema validator. It supports JSON Schema drafts 04 / 06 / 07 / 2019-09 / 2020-12, plus JSON Type Definition (RFC 8927). It compiles schemas into optimized JavaScript validators.

**Pick Ajv when:**
- The contract must be cross-language (other services in other languages consume the same schema).
- The contract is published externally (to OpenAPI, AsyncAPI, customer SDKs).
- Throughput matters and the validator is on a hot path.
- You want a standards-based artifact rather than a TypeScript-only DSL.

### Zod — TypeScript-first ergonomics

[Zod](https://zod.dev/) is a TypeScript-first validator with a fluent builder API and first-class type inference (`z.infer<typeof Schema>`). It is mature, widely adopted, and integrates with most TS tooling.

**Pick Zod when:**
- The schema is internal (only the TS service reads it).
- Developer ergonomics matter more than wire-format standardization.
- You want the schema and the type to live in one place, with the type derived from the schema.

### Valibot — tree-shakeable

[Valibot](https://valibot.dev/) is a TypeScript-first validator built around modular, tree-shakeable function composition. Smaller client-bundle impact than Zod, at the cost of a less-fluent API.

**Pick Valibot when:**
- The schema runs in the browser and bundle size matters.
- You're comfortable with a function-composition style over a method-chain style.

### ArkType — TS-syntax inference

[ArkType](https://arktype.io/) parses TypeScript-syntax type strings at runtime to produce both the runtime validator and the static type, optimizing for type-system fidelity and parser performance.

**Pick ArkType when:**
- You want schemas to read like TypeScript type literals.
- You need the validator to handle TS-style edge cases (intersections, complex unions) faithfully.
- The smaller ecosystem (vs Zod) is acceptable.

### General guidance

For services with external contracts, default to **Ajv + JSON Schema**. The cross-language interoperability is rarely something you outgrow.

For internal services, default to **Zod** for the ecosystem maturity, with **Valibot** as the fallback when bundle size becomes a constraint and **ArkType** when type fidelity dominates.

What you should *not* do is have the same project use three different schema libraries because each subsystem made an independent choice. Schema discipline is a system property; pick one and stick.

## Boundaries to validate

Sources of `unknown`-shaped data:

- **HTTP request bodies and query parameters** — every framework's "this is a `Foo`" type is a fiction until parsed.
- **Message-bus payloads** — Kafka, NATS, RabbitMQ. All bytes.
- **Environment variables** — `process.env.X` is `string | undefined`, even when the value is supposed to be a number, a URL, or a boolean.
- **CLI arguments** — same.
- **Feature-flag payloads** — JSON delivered by an external system.
- **Persisted state on read-back** — the schema may have changed since the row was written; explicit re-validation is the only safe path.
- **File imports** — `import data from './data.json'` types the data structurally but doesn't validate that the JSON file matches the type.
- **WebSocket frames, gRPC payloads decoded at the edge, multipart-form data** — all bytes.

Any data crossing the program's outer boundary belongs in this list.

## Cite-able primary source

The OWASP [Input Validation Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Input_Validation_Cheat_Sheet.html) is the canonical industry reference for this discipline. Its core recommendation — validate syntactic and semantic correctness of all external input as early as possible using allowlists — is exactly what schema-at-the-boundary implements.

## Composing with other practices

- [[Languages/TypeScript/Practices/Type-Driven Design/Branded Types for Domain Identifiers]] — schema parsers produce branded values directly.
- [[Languages/TypeScript/Practices/Error Handling/Result Pattern for Expected Failures]] — parse failure is `Result<T, ParseError>`.
- [[Languages/TypeScript/Practices/Foundational/Strict Compiler Profile]] — `useUnknownInCatchVariables` forces the same `unknown`-first discipline at catch sites.
- [[Engineering Philosophy/Principles/Layered Risk Categories]] — runtime validation covers the *type-model* layer at the *runtime* boundary; without it, the type-model layer can't be trusted at runtime.
- [[Engineering Philosophy/Principles/Specification Risk vs Type Soundness]] — the schema is the runtime presence of the specification; tests, type checks, and runtime schemas are the three places the same invariant should live.

## Related

- [[Languages/TypeScript/Practices/Type-Driven Design/Branded Types for Domain Identifiers]]
- [[Languages/TypeScript/Practices/Error Handling/Result Pattern for Expected Failures]]
- [[Languages/TypeScript/Practices/Error Handling/Process-Fatal vs Domain-Recoverable]]
- [[Languages/TypeScript/Practices/Foundational/Strict Compiler Profile]]
- [[Engineering Philosophy/Principles/Specification Risk vs Type Soundness]]
- [[Languages/Rust/Practices/Error Handling/Domain Errors at Boundaries]]
