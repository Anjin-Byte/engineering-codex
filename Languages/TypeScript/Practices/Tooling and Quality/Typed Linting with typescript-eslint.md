---
title: Typed Linting with typescript-eslint
tags: [tooling, typescript, eslint, lint, type-aware]
summary: Adopt typescript-eslint flat config with `recommendedTypeChecked`, `strictTypeChecked`, and `stylisticTypeChecked` plus explicit rules for promise hygiene and exhaustiveness; the slowdown is the cost of catching what untyped lint cannot.
keywords: [typescript-eslint, flat config, typed linting, type-aware lint, recommendedTypeChecked, strictTypeChecked, stylisticTypeChecked, projectService]
---

*Untyped lint catches syntax. Typed lint catches semantics. The distinction is large; the cost is real; the answer for a critical-path project is typed.*

# Typed Linting with typescript-eslint

> **Rule:** every TypeScript project enables typed linting via the [typescript-eslint typed-linting setup](https://typescript-eslint.io/getting-started/typed-linting/) using flat-config bundles `recommendedTypeChecked`, `strictTypeChecked`, and `stylisticTypeChecked`. Untyped linting is acceptable for non-critical projects but cannot enforce promise hygiene, no-unsafe rules, exhaustiveness, or any other rule that requires type information.

The typescript-eslint project provides three layers of configuration with progressive strictness, each with a typed counterpart:

| Bundle | Type-aware? | Purpose |
|---|---|---|
| `recommended` / `recommendedTypeChecked` | ✓ when typed | Baseline correctness and bug-prevention rules. |
| `strict` / `strictTypeChecked` | ✓ when typed | Stricter correctness; catches subtler bugs. |
| `stylistic` / `stylisticTypeChecked` | ✓ when typed | Stylistic consistency rules that need type info. |

The `*TypeChecked` variants require the type-checker. The non-`TypeChecked` variants are syntax-only.

## The drop-in flat config

The full template lives at [[Languages/TypeScript/Practices/Tooling and Quality/Reference ESLint Config]]. The structure is:

```js
// eslint.config.js
import tseslint from "typescript-eslint";
import eslint from "@eslint/js";

export default tseslint.config(
  eslint.configs.recommended,
  tseslint.configs.strictTypeChecked,
  tseslint.configs.stylisticTypeChecked,
  {
    languageOptions: {
      parserOptions: {
        projectService: true,        // auto-discover tsconfigs
        tsconfigRootDir: import.meta.dirname,
      },
    },
    rules: {
      // Project-specific overrides go here
    },
  },
);
```

The `projectService: true` flag (since typescript-eslint v8) is the modern equivalent of the older explicit `project: true` and is what makes the type-checker available to type-aware rules. It auto-discovers `tsconfig.json` files for each linted file.

## What typed lint enables that untyped lint cannot

The four rule families that require type information:

### Promise hygiene
- `no-floating-promises` — every promise is awaited or explicitly `void`-ed.
- `no-misused-promises` — promises don't flow into places that don't await them (event handlers, `if` conditions, `forEach`).
- `require-await` — `async` functions actually await something (catches incomplete refactors).
- `prefer-promise-reject-errors` — every rejection is an `Error`, not a string or arbitrary value.

These all need to know which expressions return `Promise`. Covered in detail at [[Languages/TypeScript/Practices/Async and Concurrency/Floating Promise Hygiene]].

### `any` containment
- `no-unsafe-assignment` — flags `any` flowing into typed code.
- `no-unsafe-call` — flags calling `any` (a common consequence of forgotten generics).
- `no-unsafe-member-access` — flags `.x` on `any`.
- `no-unsafe-return` — flags returning `any` from a typed function.
- `no-unsafe-argument` — flags passing `any` into a typed parameter.

Together these contain the contagion that `any` causes once it enters a type. They need full type-flow analysis.

### Exhaustiveness
- `switch-exhaustiveness-check` — every switch on a union type either handles all cases or has a default. The lint-level enforcement complements [[Languages/TypeScript/Practices/Type-Driven Design/Discriminated Unions and Exhaustive Handling]]'s explicit `never`-based check.

### Type-correct coercions
- `no-base-to-string` — catches `${obj}` interpolation that produces `[object Object]` because `obj` lacks a meaningful `toString`.
- `restrict-template-expressions` — only certain types are allowed in template literals.
- `restrict-plus-operands` — `+` only on compatible types.

## Beyond the bundles — rules worth enabling explicitly

The `strictTypeChecked` bundle is comprehensive but a few rules earn an explicit override:

- `eqeqeq: ["error", "always"]` — `===` not `==`. Catches a class of coercion bug.
- `no-eval: "error"` — eval is rarely correct and always a security risk.
- `no-implied-eval: "error"` — string-as-callback in `setTimeout`, etc.
- `no-console` (project policy) — depending on whether console is your logger or a forgotten debugging artifact.

Disable selectively for test files where some rules are pragmatic to relax:

```js
{
  files: ["**/*.test.ts", "**/*.spec.ts"],
  rules: {
    "@typescript-eslint/no-unused-expressions": "off",  // expectations
    "no-console": "off",                                 // test debugging
  },
}
```

## Performance trade-off

Typed linting is slower. Reports of 5–20× slowdown vs syntax-only lint are common. The slowdown is real and should be planned for:

- **In CI, run typed lint as a quality gate** — accept the slowdown there, where the cost of a wrong merge dominates the cost of a slow build.
- **In editor / on-save, configure depending on machine** — fast machines can typed-lint inline; slower machines may want syntax-only on save and full lint on pre-commit / CI.
- **Pre-commit hooks**: typed lint of changed files only is usually fast enough; full-project typed lint is too slow for pre-commit.

The slowdown is not a reason to skip typed linting. It is a reason to plan when to run it.

## Why not Biome instead?

[Biome](https://biomejs.dev/) is a fast, all-in-one toolchain written in Rust. For projects where speed is paramount and the rule set Biome covers is sufficient, it's a reasonable choice. For projects where typed linting matters — and that's any critical-path TypeScript project — Biome currently does not match typescript-eslint's rule set, particularly for promise hygiene and unsafe-`any` containment. The recommendation:

- Use **typescript-eslint** for safety rules (priority).
- Use **Biome** for formatting (faster than Prettier, no rule conflict with typescript-eslint).
- Don't use Biome to *replace* typescript-eslint until its typed-rule set matches.

Re-evaluate every six to twelve months — the gap is narrowing.

## Composing with other practices

- [[Languages/TypeScript/Practices/Async and Concurrency/Floating Promise Hygiene]] — the rule cluster this note operationalizes.
- [[Languages/TypeScript/Practices/Type-Driven Design/Discriminated Unions and Exhaustive Handling]] — `switch-exhaustiveness-check` enforces this discipline.
- [[Languages/TypeScript/Practices/Foundational/Strict Compiler Profile]] — typed lint complements the compiler-strictness story; both enforce different classes of rule.
- [[Languages/TypeScript/Practices/Tooling and Quality/Reference ESLint Config]] — the drop-in template.
- [[Languages/Rust/Practices/Tooling and Quality/Clippy as Discipline]] — same posture, different tool.

## Related

- [[Languages/TypeScript/Practices/Tooling and Quality/Reference ESLint Config]]
- [[Languages/TypeScript/Practices/Async and Concurrency/Floating Promise Hygiene]]
- [[Languages/TypeScript/Practices/Foundational/Strict Compiler Profile]]
- [[Languages/TypeScript/Practices/Type-Driven Design/Discriminated Unions and Exhaustive Handling]]
- [[Languages/Rust/Practices/Tooling and Quality/Clippy as Discipline]]
