---
title: Strict Compiler Profile
tags: [foundational, typescript, compiler, type-system]
summary: Enable `strict` plus the additional flags that close type-system loopholes a strict baseline doesn't catch — each flag is a class of bug the compiler can prevent for free.
keywords: [tsconfig, strict mode, compiler options, type safety, noUncheckedIndexedAccess, exactOptionalPropertyTypes, useUnknownInCatchVariables]
---

*The strict baseline is not enough on its own. Add the flags that catch the categories of bug `strict` does not.*

# Strict Compiler Profile

> **Rule:** every TypeScript project enables `strict: true` *plus* the eight flags below. Each one closes a class of bug the strict baseline still permits, and each costs almost nothing — a one-line `tsconfig.json` change in exchange for compile-time prevention of an entire failure mode.

The TypeScript team explicitly says of [`strict`](https://www.typescriptlang.org/tsconfig/strict.html) that it "enables a wide range of type checking behavior that results in stronger guarantees of program correctness," and reserves the right to add stricter checks under it in future versions. That covers the foundational class of issues — implicit `any`, missing null checks, unsafe `this`, and so on. It does not cover the remaining set below.

## The required additions

### `noUncheckedIndexedAccess`

> [Adds `undefined` to any un-declared field in the type](https://www.typescriptlang.org/tsconfig/noUncheckedIndexedAccess.html) so that indexed access on objects with index signatures is recognized as possibly missing.

Without this flag, `myMap[someKey]` returns the value type as if it were always present. With it, the result is `T | undefined` and the caller is forced to handle the missing case. This prevents a large class of silent `undefined`-propagation bugs, particularly around lookup tables, environment maps, and parsed JSON payloads.

### `exactOptionalPropertyTypes`

> [Treats `?`-marked properties strictly](https://www.typescriptlang.org/tsconfig/exactOptionalPropertyTypes.html) so that "missing" and "present but `undefined`" are no longer interchangeable.

Without this flag, `{ name?: string }` permits `{ name: undefined }` and `{}` interchangeably. Code that round-trips this object through serialization, ORMs, or partial-update APIs cannot reliably tell whether the field was explicitly cleared or never present. With the flag, those two states are distinct types and the code must choose.

### `useUnknownInCatchVariables`

> [Defaults the type of catch-clause variables to `unknown`](https://www.typescriptlang.org/tsconfig/useUnknownInCatchVariables.html) rather than `any`, forcing narrowing before the caught value is used.

A `catch (e)` block where `e` is typed `any` lets every property access compile, including `e.message`, `e.code`, and any fictional property. With `unknown`, the catch handler is forced to narrow (`if (e instanceof Error)`, schema-validate, or assert) before reading anything. This is the simplest possible fix for an entire class of "the error handler crashed" production incidents.

### `noImplicitOverride`

> [Requires the `override` keyword on subclass methods](https://www.typescriptlang.org/tsconfig/noImplicitOverride.html) that override a base class member, preventing silent drift if the base method is renamed or removed.

When a base class renames a method, an unmarked subclass method that thought it was overriding stops overriding silently — and starts being a new method that no caller ever invokes. The flag turns this into a compile error.

### `noFallthroughCasesInSwitch`

> [Errors on non-empty switch cases that lack `break`, `return`, or `throw`](https://www.typescriptlang.org/tsconfig/noFallthroughCasesInSwitch.html), which "won't accidentally ship a case fallthrough bug."

Switch fallthrough is occasionally intended, but the intentional case is rare enough that requiring an explicit `// fall through` comment (or restructuring as a discriminated-union match) is cheaper than the bug it prevents.

### `noEmitOnError`

> [Do not emit compiler output files like JavaScript source code, source-maps or declarations if any errors were reported](https://www.typescriptlang.org/tsconfig/noEmitOnError.html), preventing partially-broken output from being generated.

In CI, with the build feeding into a release pipeline, a `tsc` run that emits *some* JavaScript despite errors is dangerous: subsequent build steps may run successfully against partial output and the pipeline may keep moving. With `noEmitOnError`, a type error stops the pipeline cold, where it should.

### `skipLibCheck` — handle with care

> [Skip type checking of declaration files](https://www.typescriptlang.org/tsconfig/skipLibCheck.html), which trades type-system accuracy for compile speed and is mainly a workaround for inconsistent third-party `.d.ts` definitions.

This is the one flag in this set that is *recommended off* for a critical-path project, but is widely turned on in practice because the third-party `.d.ts` ecosystem still ships internally inconsistent typings. If the project turns it on, it should know that doing so trades correctness for speed and should periodically run a `skipLibCheck: false` build to catch drift.

The strict-profile recommendation: turn it **off** in CI's release-gate build; turn it **on** in the watch / dev build if necessary. The release gate is the place where the cost of being wrong is highest.

## Other flags worth considering

The following are not in the verified-source set above but are commonly part of strict-profile recommendations. They should be evaluated per project:

- `noImplicitReturns` — every code path in a function returns a value (or none does).
- `noPropertyAccessFromIndexSignature` — distinguish dot access (`o.prop`, declared) from bracket access (`o["prop"]`, index-signature) at the call site.
- `isolatedModules` — every file is independently transpilable, ruling out features that require whole-program awareness.
- `forceConsistentCasingInFileNames` — case-sensitive imports, which avoids cross-platform drift between case-insensitive (macOS, Windows) and case-sensitive (Linux) filesystems.
- `verbatimModuleSyntax` — emits exactly the import/export form the source uses, removing ambiguity about type-only imports.

A drop-in template incorporating these decisions lives at [[Languages/TypeScript/Practices/Tooling and Quality/Reference TSConfig]].

## What this profile does *not* do

Strict-mode discipline closes type-soundness loopholes. It does not address [[Engineering Philosophy/Principles/Specification Risk vs Type Soundness]] — the question of whether the types you wrote down are the right invariants. For that, you need runtime validation at boundaries, property-based tests, and observability. The compiler can prevent the type from being violated; it cannot tell you the type was wrong to begin with.

## Related

- [[Languages/TypeScript/Practices/Tooling and Quality/Reference TSConfig]]
- [[Languages/TypeScript/Practices/Error Handling/Runtime Validation at Boundaries]]
- [[Languages/TypeScript/Practices/Type-Driven Design/Discriminated Unions and Exhaustive Handling]]
- [[Engineering Philosophy/Principles/Specification Risk vs Type Soundness]]
- [[Languages/Rust/Practices/Foundational/Push Correctness Left]]
