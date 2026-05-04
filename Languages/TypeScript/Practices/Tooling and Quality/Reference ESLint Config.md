---
title: Reference ESLint Config
tags: [tooling, typescript, eslint, lint, template]
summary: A drop-in ESLint flat-config template for TypeScript projects, using typescript-eslint typed linting with `recommendedTypeChecked`, `strictTypeChecked`, and `stylisticTypeChecked` plus explicit safety overrides.
keywords: [eslint flat config, typescript-eslint, typed linting, projectService, strictTypeChecked, drop-in template]
---

*A starting `eslint.config.js` that ships with typed linting on, the right rule bundles selected, and the most-frequent overrides explicit. Adjust project-specific globs; keep the rule core.*

# Reference ESLint Config

> **Rule:** every TypeScript project starts from this template. It enables typed linting, applies the strict-typed bundle, includes the explicit overrides every critical-path project ends up adding anyway, and partitions test-file rules from production-code rules.

The full reasoning behind the rule choices lives at [[Languages/TypeScript/Practices/Tooling and Quality/Typed Linting with typescript-eslint]] and [[Languages/TypeScript/Practices/Async and Concurrency/Floating Promise Hygiene]]. This note is the *complete* config — what to drop in.

## The template

```js
// eslint.config.js (ESM flat config; place at project root)
import eslint from "@eslint/js";
import tseslint from "typescript-eslint";
import globals from "globals";

export default tseslint.config(
  {
    ignores: ["dist/**", "build/**", "coverage/**", "**/*.cjs", "**/.tsbuildinfo"],
  },

  // Base recommended for plain JS files (rare in a TS project, but covers config files)
  eslint.configs.recommended,

  // Type-aware bundles for TypeScript
  ...tseslint.configs.strictTypeChecked,
  ...tseslint.configs.stylisticTypeChecked,

  {
    files: ["**/*.ts", "**/*.tsx"],
    languageOptions: {
      parserOptions: {
        projectService: true,
        tsconfigRootDir: import.meta.dirname,
      },
      globals: {
        ...globals.node,
      },
    },
    rules: {
      // --- Promise / async hygiene (typed) ---
      "@typescript-eslint/no-floating-promises": "error",
      "@typescript-eslint/no-misused-promises": "error",
      "@typescript-eslint/require-await": "error",
      "@typescript-eslint/prefer-promise-reject-errors": "error",

      // --- Exhaustiveness ---
      "@typescript-eslint/switch-exhaustiveness-check": ["error", { considerDefaultExhaustiveForUnions: true }],

      // --- Containment of any (typed) ---
      // These are in strictTypeChecked already; listed for visibility / override flexibility.
      "@typescript-eslint/no-unsafe-assignment": "error",
      "@typescript-eslint/no-unsafe-call": "error",
      "@typescript-eslint/no-unsafe-member-access": "error",
      "@typescript-eslint/no-unsafe-return": "error",
      "@typescript-eslint/no-unsafe-argument": "error",
      "@typescript-eslint/no-explicit-any": "error",

      // --- Type-correct coercions / comparisons ---
      "@typescript-eslint/no-base-to-string": "error",
      "@typescript-eslint/restrict-template-expressions": ["error", { allowNumber: true, allowBoolean: true }],
      "@typescript-eslint/restrict-plus-operands": "error",
      "eqeqeq": ["error", "always"],

      // --- Security-adjacent ---
      "no-eval": "error",
      "no-implied-eval": "error",
      "@typescript-eslint/no-implied-eval": "error",

      // --- Style / readability ---
      "@typescript-eslint/consistent-type-imports": ["error", { prefer: "type-imports", fixStyle: "inline-type-imports" }],
      "@typescript-eslint/no-unused-vars": ["error", {
        argsIgnorePattern: "^_",
        varsIgnorePattern: "^_",
        caughtErrorsIgnorePattern: "^_",
      }],
    },
  },

  // Test-file overrides — relax rules pragmatic for test ergonomics
  {
    files: ["**/*.test.ts", "**/*.spec.ts", "test/**/*.ts", "tests/**/*.ts"],
    rules: {
      "@typescript-eslint/no-unused-expressions": "off",
      "@typescript-eslint/no-non-null-assertion": "off",
      "@typescript-eslint/unbound-method": "off",
      "no-console": "off",
    },
  },

  // Plain-JS config-file override (vite, eslint, build configs themselves)
  {
    files: ["**/*.config.{js,mjs,cjs}", "**/*.config.{ts,mts,cts}"],
    languageOptions: {
      globals: {
        ...globals.node,
      },
    },
    rules: {
      "@typescript-eslint/no-explicit-any": "off",
    },
  },
);
```

Save as `eslint.config.js` (ESM) at the project root. The flat-config format is the typescript-eslint recommended path going forward; the older `.eslintrc.js` format is still supported but deprecated.

## What's in this template that needs explanation

### `projectService: true`

The modern (typescript-eslint v8+) auto-discovery mechanism for `tsconfig.json` files. Replaces the older `project: true` / `project: ["./tsconfig.json"]` pattern. It auto-finds tsconfigs for each linted file, including monorepos with multiple tsconfigs.

For typescript-eslint v7 or older, use:

```js
parserOptions: {
  project: "./tsconfig.json",
  tsconfigRootDir: import.meta.dirname,
}
```

### `consistent-type-imports`

Forces type-only imports to use the `import type` form (or `import { type Foo }` inline). With `verbatimModuleSyntax: true` in tsconfig (see [[Languages/TypeScript/Practices/Tooling and Quality/Reference TSConfig]]), the compiler errors on import drift; this lint rule autofixes it.

### Test-file overrides

A few rules trade safety for test ergonomics:
- `no-unused-expressions` — testing libraries often use bare expressions for assertions.
- `no-non-null-assertion` — `expect(value!).toBe(...)` is common when the test setup has guaranteed the value.
- `unbound-method` — Jest / Vitest mock setup sometimes needs unbound method references.
- `no-console` — debugging tests with `console.log` is normal.

If you're not using a test framework that needs these relaxations, drop them.

### Config-file override

Build tool config files (`vite.config.ts`, `eslint.config.js`, etc.) are typically allowed `any` for the framework's option types where the framework doesn't export precise types.

## Plugin extensions worth considering

Beyond the base typescript-eslint setup, project-specific plugins:

- **`eslint-plugin-import`** — import / export linting, ordering. Useful but slow under typed-resolution; consider `eslint-plugin-import-x` (faster fork).
- **`eslint-plugin-vitest`** or **`eslint-plugin-jest`** — test-framework-specific rules.
- **`eslint-plugin-react`** / **`eslint-plugin-react-hooks`** — for React projects.
- **`eslint-plugin-security`** — security-pattern checks (debated value; some rules are noisy).
- **`eslint-plugin-unicorn`** — opinionated stylistic rules; use selectively if at all.

The vault doesn't prescribe any of these. Add them if the project's domain calls for them; resist adding them by default, because each plugin has a maintenance and lint-time cost.

## Companion: Prettier vs Biome for formatting

ESLint can format, but it's slow and overlapping responsibility-with-rules causes friction. The two reasonable choices for formatting:

- **Prettier** with `eslint-config-prettier` to disable conflicting ESLint rules. Mature, ubiquitous.
- **Biome** for formatting only (faster than Prettier; no formatting-related rule conflict with typescript-eslint).

Pick one project-wide. Don't use both.

## Composing with other practices

- [[Languages/TypeScript/Practices/Tooling and Quality/Typed Linting with typescript-eslint]] — the rule rationale.
- [[Languages/TypeScript/Practices/Tooling and Quality/Reference TSConfig]] — the compiler-config sibling; both are needed.
- [[Languages/TypeScript/Practices/Async and Concurrency/Floating Promise Hygiene]] — the rule cluster this config operationalizes.
- [[Languages/Rust/Practices/Tooling and Quality/Clippy as Discipline]] — same posture, different tool.
- [[Languages/Rust/Practices/Tooling and Quality/CI Quality Bar]] — typed lint runs in CI as a quality gate.

## Related

- [[Languages/TypeScript/Practices/Tooling and Quality/Typed Linting with typescript-eslint]]
- [[Languages/TypeScript/Practices/Tooling and Quality/Reference TSConfig]]
- [[Languages/TypeScript/Practices/Async and Concurrency/Floating Promise Hygiene]]
- [[Languages/Rust/Practices/Tooling and Quality/Clippy as Discipline]]
