---
title: Result Pattern for Expected Failures
tags: [error-handling, typescript, result, expected-failure]
summary: Expected failures return `Result<T, E>` (or library equivalent); `throw` is reserved for invariant violations. The compiler then forces the caller to handle the failure case.
keywords: [result type, either type, typed errors, expected failures, invariant violations, error as value]
---

*Failures the spec lists belong in the type system. The compiler can force every caller to handle them. `throw` is for failures the spec never anticipated.*

# Result Pattern for Expected Failures

> **Rule:** if the spec says "this can fail with X, Y, or Z," the function returns a `Result<T, E>` (or `Either<E, T>`) with `E` enumerating those failure modes. The compiler then refuses to let any caller forget to handle them. `throw` is reserved for situations the spec did not anticipate — invariant violations, programmer errors, broken assumptions.

This is the same partition Rust draws between `Result` and `panic!`, expressed in TypeScript. TypeScript has no built-in `Result` type, so the discipline is conventional rather than language-enforced — but the convention pays for itself the first time a downstream caller silently ignores a now-typed error.

## The pattern

```ts
type Ok<T> = { ok: true; value: T };
type Err<E> = { ok: false; error: E };
type Result<T, E> = Ok<T> | Err<E>;

const ok = <T>(value: T): Ok<T> => ({ ok: true, value });
const err = <E>(error: E): Err<E> => ({ ok: false, error });
```

A function that can fail with a known set of reasons returns one:

```ts
type TransferError =
  | { kind: "insufficient_funds"; available: Cents; required: Cents }
  | { kind: "account_frozen"; account: AccountId }
  | { kind: "amount_invalid"; amount: Cents };

function transferFunds(
  from: AccountId, to: AccountId, amount: Cents,
): Result<Receipt, TransferError> {
  if (amount <= 0) return err({ kind: "amount_invalid", amount });
  // ...
  return ok(receipt);
}
```

Every caller is now compelled by the type system to inspect `result.ok` before reading `result.value`. The discriminated-union pattern from [[Languages/TypeScript/Practices/Type-Driven Design/Discriminated Unions and Exhaustive Handling]] applies directly: a switch on `error.kind` will exhaustiveness-check, so adding a new failure variant becomes a *compile error* at every consumer that didn't handle it.

## Why this beats throw-and-catch

A throw-based design has three structural problems:

1. **The signature lies.** `function transferFunds(...): Receipt` looks total. A caller has to read the body or the docs to learn it can fail. The type system is providing zero information. With `Result<Receipt, TransferError>`, the failure shape is part of the signature.
2. **The catch site loses the type.** `catch (e)` is `unknown` (with `useUnknownInCatchVariables`) or `any` (without). The catch handler must guard-narrow before reading any field — and if a new error type is thrown that the handler didn't anticipate, it gets caught silently and treated as a generic error.
3. **Forgetting to catch is invisible.** A function that throws and a caller that doesn't catch produces an exception at the *next* `await` boundary, possibly far from the failure site, possibly resulting in an unhandled rejection that crashes the process. A function that returns `Result<T, E>` and a caller that ignores it produces a compile-time warning under typed linting.

## When to throw anyway

Throws still belong, but only for *invariant violations* and *programmer errors*:

- **Unreachable code.** The `assertNever` helper (see [[Languages/TypeScript/Practices/Type-Driven Design/Discriminated Unions and Exhaustive Handling]]) throws because being there means the type system's invariants have been violated.
- **Required preconditions failing.** A constructor receiving an obviously-invalid value (a negative `Cents` that should have been validated upstream) can throw. The point is: getting here means the caller violated the contract, not that a normal failure occurred.
- **Resource-exhaustion conditions** the spec says are unrecoverable (out-of-memory, etc.).

These should be process-fatal — see [[Languages/TypeScript/Practices/Error Handling/Process-Fatal vs Domain-Recoverable]]. The whole point of throwing is to stop the process before it produces wrong output.

## Result libraries

Several libraries provide canonical `Result` / `Either` implementations:

- **`neverthrow`** — TypeScript-first, fluent (`.map`, `.andThen`, `.match`).
- **`ts-results`** — Rust-flavored API.
- **`fp-ts` / `effect`** — full functional ecosystems, with `Either` as one piece.

The vault does not prescribe a specific library. The hand-rolled `Ok | Err` discriminated union above is sufficient for projects that want to keep dependencies minimal; library adoption is appropriate when the project benefits from richer combinators.

What matters is the *discipline*: expected failures return `Result`, not throws.

## Promise-based handling

For async functions, the same pattern applies — return `Promise<Result<T, E>>`:

```ts
async function transferFunds(...): Promise<Result<Receipt, TransferError>> {
  // ...
}

const result = await transferFunds(...);
if (!result.ok) {
  // typed handler
  return;
}
useReceipt(result.value);
```

This composes cleanly with [[Languages/TypeScript/Practices/Async and Concurrency/Floating Promise Hygiene]] — the linter's `no-floating-promises` rule catches forgotten `await`, and the `Result` discipline catches forgotten error handling. Both protections are needed; neither subsumes the other.

## Composing with other practices

- [[Languages/TypeScript/Practices/Type-Driven Design/Discriminated Unions and Exhaustive Handling]] — `Result<T, E>` is itself a discriminated union; `E` is itself often one too.
- [[Languages/TypeScript/Practices/Error Handling/Process-Fatal vs Domain-Recoverable]] — the partition of throws-vs-results.
- [[Languages/TypeScript/Practices/Error Handling/Runtime Validation at Boundaries]] — schema validation produces a `Result<TypedT, ValidationError>` at the system boundary, so the typed core never sees raw input.
- [[Languages/Rust/Practices/Error Handling/Result vs Panic]] — same principle, native to Rust.
- [[Languages/Rust/Practices/Error Handling/Domain Errors at Boundaries]] — typed enum errors at library boundaries; the same discipline applies in TS.

## Related

- [[Languages/TypeScript/Practices/Error Handling/Process-Fatal vs Domain-Recoverable]]
- [[Languages/TypeScript/Practices/Error Handling/Runtime Validation at Boundaries]]
- [[Languages/TypeScript/Practices/Type-Driven Design/Discriminated Unions and Exhaustive Handling]]
- [[Languages/Rust/Practices/Error Handling/Result vs Panic]]
- [[Languages/Rust/Practices/Error Handling/Domain Errors at Boundaries]]
