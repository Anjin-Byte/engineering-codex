---
title: Floating Promise Hygiene
tags: [async, typescript, eslint, promise, lint]
summary: Typed linting must enforce `no-floating-promises`, `no-misused-promises`, and `switch-exhaustiveness-check` — these rules need full type information and untyped linting cannot catch them.
keywords: [floating promise, misused promise, eslint, typescript-eslint, async hygiene, unhandled rejection, type-aware lint]
---

*Forgetting `await` is silent in JavaScript. The compiler can't help; only typed linting can. Pay the slowdown to get the protection.*

# Floating Promise Hygiene

> **Rule:** every TypeScript project enables typed linting via [typescript-eslint](https://typescript-eslint.io/getting-started/typed-linting/) and turns on at minimum `no-floating-promises`, `no-misused-promises`, and `switch-exhaustiveness-check`. These rules require the type-checker to run and are slower than untyped linting; the slowdown is the cost of the protection. There is no equivalent at the compiler level.

JavaScript silently accepts code that promises something but never awaits it. A function returning a `Promise` whose result is discarded compiles cleanly, runs without warning, and may produce wrong-state outcomes minutes later when the promise eventually rejects unhandled. The compiler can't see the bug; the runtime tolerates it. The only place the discipline can be enforced is the linter, and only the typed linter has enough information to detect it.

## The three load-bearing rules

### `no-floating-promises`

Catches promises whose result is discarded:

```ts
// ✗ Floating — the result is unhandled
sendEmail(user, "welcome");

// ✓ Awaited
await sendEmail(user, "welcome");

// ✓ Explicitly fire-and-forget (the rule allows this when the intent is annotated)
void sendEmail(user, "welcome");
```

Forgotten `await` is one of the highest-frequency Node defects in production. The rule eliminates a category. The `void` operator is the documented escape for the rare case where you genuinely don't want to wait for the result — but using it forces the author to declare the intent, and a reviewer can ask whether it was the right call.

### `no-misused-promises`

Catches places where a promise is being passed to something that doesn't expect one:

```ts
// ✗ if expects a boolean; the promise is always truthy
if (isAuthorized(user)) { /* ... */ }

// ✗ forEach can't await — the loop completes before the promises do
items.forEach(async item => { await process(item); });

// ✗ event handler returns Promise; emitter ignores it
emitter.on("data", async (chunk) => { await save(chunk); });
```

In each case the program executes, but the semantics are wrong. The first case treats every async authorization as authorized. The second loses ordering and never waits. The third loses errors entirely (the emitter has no notion of awaiting).

### `switch-exhaustiveness-check`

Pairs with [[Languages/TypeScript/Practices/Type-Driven Design/Discriminated Unions and Exhaustive Handling]]: when a switch dispatches on a discriminated-union tag, the rule errors if any variant is missed. The compiler's `never`-based exhaustiveness check is opt-in (you must write the `_exhaustive: never = ...` line); the linter's check is automatic across the codebase. Both are useful — the linter as a default safety net, the explicit `never` assertion as a clearer signal at the dispatch site.

## Why typed linting

Untyped linters (the default ESLint without `parserOptions.project`) can lint syntax. The three rules above all need to know whether an expression is a `Promise`, which only the TypeScript type-checker knows. Enabling typed linting requires:

```js
// eslint.config.js (flat config)
import tseslint from "typescript-eslint";

export default tseslint.config(
  tseslint.configs.strictTypeChecked,
  tseslint.configs.stylisticTypeChecked,
  {
    languageOptions: {
      parserOptions: {
        projectService: true,
      },
    },
  }
);
```

The `projectService: true` flag (or the older explicit `project: true`) is what makes the type-checker available to the rules. The trade-off is slower lint runs — typed linting can be 5–20× slower than untyped — but for critical-path projects the rules it enables justify the cost.

A drop-in flat config is in [[Languages/TypeScript/Practices/Tooling and Quality/Reference ESLint Config]].

## Other rules worth pairing

The `recommendedTypeChecked` and `strictTypeChecked` config bundles include many additional type-aware rules. Worth highlighting:

- **`no-unsafe-assignment`, `no-unsafe-call`, `no-unsafe-member-access`, `no-unsafe-return`** — catch values typed `any` flowing through safe-looking code paths. Particularly useful at trust boundaries.
- **`require-await`** — flags `async` functions with no `await`, often a sign of an incomplete refactor.
- **`prefer-promise-reject-errors`** — every rejection should be an `Error`, not a string or arbitrary value.
- **`no-base-to-string`** — catches accidental `${obj}` string interpolation that produces `[object Object]`.
- **`switch-exhaustiveness-check`** (covered above).

A project's strictness is roughly the union of strict-typed-lint plus the rules above.

## What this rule does *not* do

Floating-promise lint catches a *pattern* failure; it does not catch a *content* failure. A function that awaits an upstream call but ignores the rejected `Result<T, E>` returned by it has not floated a promise — the promise was awaited — but it has still ignored an expected failure. That is what [[Languages/TypeScript/Practices/Error Handling/Result Pattern for Expected Failures]] addresses, and the linter cannot enforce it without a *project-specific* rule that knows your `Result` type.

Similarly, the linter cannot tell you whether a promise *should* be fire-and-forget. The `void` escape works because the alternative — making every async call mandatory-await — would be impractical in real systems with legitimate fire-and-forget use cases (kicking off a background task, queueing telemetry, scheduling cleanup). The rule traps the common bug without trying to be perfect.

## Composing with other practices

- [[Languages/TypeScript/Practices/Error Handling/Process-Fatal vs Domain-Recoverable]] — unhandled rejections are process-fatal in modern Node, which is the correct behavior. Floating-promise lint catches the *cause* before runtime; the runtime fail-fast catches anything that slipped through.
- [[Languages/TypeScript/Practices/Error Handling/Result Pattern for Expected Failures]] — once the lint forces you to `await`, the next discipline is to handle the `Result` the await yields.
- [[Languages/TypeScript/Practices/Type-Driven Design/Discriminated Unions and Exhaustive Handling]] — `switch-exhaustiveness-check` enforces what the explicit `never`-based check makes opt-in.
- [[Languages/TypeScript/Practices/Tooling and Quality/Typed Linting with typescript-eslint]] — the parent topic; this note is one rule cluster within it.
- [[Languages/TypeScript/Practices/Tooling and Quality/Reference ESLint Config]] — the drop-in template.

## Related

- [[Languages/TypeScript/Practices/Tooling and Quality/Typed Linting with typescript-eslint]]
- [[Languages/TypeScript/Practices/Tooling and Quality/Reference ESLint Config]]
- [[Languages/TypeScript/Practices/Error Handling/Process-Fatal vs Domain-Recoverable]]
- [[Languages/TypeScript/Practices/Error Handling/Result Pattern for Expected Failures]]
- [[Languages/TypeScript/Practices/Type-Driven Design/Discriminated Unions and Exhaustive Handling]]
