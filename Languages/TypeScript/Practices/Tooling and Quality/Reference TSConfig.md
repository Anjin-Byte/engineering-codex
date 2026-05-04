---
title: Reference TSConfig
tags: [tooling, typescript, tsconfig, template]
summary: A drop-in `tsconfig.json` template for critical-path TypeScript projects, with every option justified — strict compiler profile, deterministic emit, no-emit-on-error, modern target.
keywords: [tsconfig template, compiler options, strict mode, target es2022, module nodenext, isolatedModules, verbatimModuleSyntax]
---

*A starting `tsconfig.json` that ships with every option enabled for a reason. Adjust the project-specific bits; keep the strictness bits.*

# Reference TSConfig

> **Rule:** every TypeScript project starts from this template. Project-specific paths, target, and entry points adjust; the strictness profile and the emit settings do not.

The full reasoning for each strictness flag lives at [[Languages/TypeScript/Practices/Foundational/Strict Compiler Profile]]. This note is the *complete* template — strictness, module behavior, emit, source maps, isolation, and editor-experience settings.

## The template

```jsonc
{
  "compilerOptions": {
    /* --- Type-checking strictness --- */
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "exactOptionalPropertyTypes": true,
    "useUnknownInCatchVariables": true,
    "noImplicitOverride": true,
    "noFallthroughCasesInSwitch": true,
    "noImplicitReturns": true,
    "noPropertyAccessFromIndexSignature": true,

    /* --- Emit safety --- */
    "noEmitOnError": true,
    "skipLibCheck": false,                   // Turn off in CI; on for dev iteration only.
    "forceConsistentCasingInFileNames": true,

    /* --- Language target --- */
    "target": "ES2022",                       // satisfies operator (4.9+), top-level await, Error.cause
    "lib": ["ES2022"],                        // backend; add "DOM" for browser builds
    "useDefineForClassFields": true,

    /* --- Module behavior --- */
    "module": "NodeNext",                     // for Node services / libraries
    "moduleResolution": "NodeNext",
    "verbatimModuleSyntax": true,             // type-only imports stay type-only
    "isolatedModules": true,                  // every file independently transpilable
    "esModuleInterop": true,
    "resolveJsonModule": true,

    /* --- Output --- */
    "outDir": "./dist",
    "rootDir": "./src",
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true,
    "inlineSources": false,                   // sources adjacent, not embedded
    "removeComments": false,                  // doc comments stay in declarations

    /* --- Editor / project --- */
    "incremental": true,
    "tsBuildInfoFile": "./.tsbuildinfo"
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist"]
}
```

Save as `tsconfig.json` at the project root.

## What each section does

### Type-checking strictness

The eight strictness flags collectively catch a wide class of bug at compile time. Each one is justified individually in [[Languages/TypeScript/Practices/Foundational/Strict Compiler Profile]]; the short version:

- `strict` — umbrella flag for the strict family (no implicit `any`, strict null checks, strict function types, etc.).
- `noUncheckedIndexedAccess` — index access returns `T | undefined`.
- `exactOptionalPropertyTypes` — distinguishes "missing" from "present but `undefined`."
- `useUnknownInCatchVariables` — caught errors are `unknown`, not `any`.
- `noImplicitOverride` — `override` keyword required on overriding methods.
- `noFallthroughCasesInSwitch` — switch fallthrough must be explicit.
- `noImplicitReturns` — every code path either returns or doesn't.
- `noPropertyAccessFromIndexSignature` — distinguishes declared from index-signature access.

### Emit safety

- `noEmitOnError: true` — type errors stop the build cold; no partial output reaches downstream pipeline steps.
- `skipLibCheck: false` — type-check declaration files in CI to catch drift in third-party `.d.ts`. Some projects keep this `true` for dev speed and run a `false` build only in the release gate; that's acceptable as long as the release gate exists.
- `forceConsistentCasingInFileNames: true` — case-sensitive imports, avoiding cross-platform drift between case-insensitive (macOS, Windows) and case-sensitive (Linux) filesystems.

### Language target

- `target: "ES2022"` — gives `satisfies` (TS 4.9+), top-level `await`, `Error.cause`, native class fields.
- `lib: ["ES2022"]` — backend default; add `"DOM"` for browser builds, `"WebWorker"` for worker contexts.
- `useDefineForClassFields: true` — class fields use the standardized *Define semantics* rather than the older *Set semantics*. Required for ES2022-compliant emit.

### Module behavior

- `module: "NodeNext"` — for Node services and libraries published to npm. (For browser-only builds via a bundler, `"ESNext"` plus `"moduleResolution": "Bundler"` is more appropriate.)
- `moduleResolution: "NodeNext"` — must match `module`.
- `verbatimModuleSyntax: true` — preserves the exact import syntax, removing ambiguity about type-only imports. Catches accidental value imports of types and vice versa.
- `isolatedModules: true` — each file must be independently transpilable. Required for tools like esbuild, swc, and Babel; disallows features that require whole-program awareness (`const enum`, namespace re-exports without `type`).
- `esModuleInterop: true` — `import x from "y"` works for CommonJS modules.
- `resolveJsonModule: true` — `import config from "./config.json"` works.

### Output

- `outDir: "./dist"`, `rootDir: "./src"` — clean separation between source and build.
- `declaration: true`, `declarationMap: true` — emit `.d.ts` and source maps for declarations. Essential for libraries; useful for monorepo project references.
- `sourceMap: true`, `inlineSources: false` — source maps adjacent (not embedded), keeping production output small while still enabling debug.
- `removeComments: false` — JSDoc comments survive into declarations, where editor tooltips can show them.

### Editor / project

- `incremental: true`, `tsBuildInfoFile: "./.tsbuildinfo"` — caches compiler state for faster subsequent builds. Add `.tsbuildinfo` to `.gitignore`.

## Variants

### Browser / DOM

```diff
-    "lib": ["ES2022"],
+    "lib": ["ES2022", "DOM", "DOM.Iterable"],
-    "module": "NodeNext",
-    "moduleResolution": "NodeNext",
+    "module": "ESNext",
+    "moduleResolution": "Bundler",
```

### Library publishing

Add:

```jsonc
"composite": true,                  // enables project references
"declarationMap": true,
"stripInternal": true               // omits @internal declarations
```

And ensure `package.json` has `"types": "./dist/index.d.ts"`.

### Monorepo with project references

Each package has its own `tsconfig.json` with `composite: true`. The root has a thin `tsconfig.json` with `references: [{ path: "./packages/foo" }, ...]`. See the [TS handbook on project references](https://www.typescriptlang.org/docs/handbook/project-references.html) (verified URL).

### Test files

Test files participate in the same `tsconfig.json`. Some projects use a separate `tsconfig.test.json` extending the main one to relax a few rules for test files specifically; that's acceptable, but the relaxations should be named (e.g. allow `any` only in test setup files).

## What this template does *not* do

- **It is not a replacement for typed linting.** The strictness profile catches type-soundness bugs; typed linting catches semantic bugs (promise hygiene, exhaustiveness). Both are required. See [[Languages/TypeScript/Practices/Tooling and Quality/Typed Linting with typescript-eslint]].
- **It does not address bundling decisions.** `module: "NodeNext"` is the right choice for Node; bundler-emit projects should use `module: "ESNext"`. The deeper bundling-vs-no-bundling decision is deferred to a future workspace-shape note.
- **It does not address project layout.** Where `src/` lives, how monorepos are structured, and how packages depend on each other is a workspace decision.

## Composing with other practices

- [[Languages/TypeScript/Practices/Foundational/Strict Compiler Profile]] — the strictness reasoning.
- [[Languages/TypeScript/Practices/Tooling and Quality/Typed Linting with typescript-eslint]] — the lint companion to this config.
- [[Languages/TypeScript/Practices/Tooling and Quality/Reference ESLint Config]] — the drop-in lint config.
- [[Languages/Rust/Workspace/Cargo Workspace Configuration]] — same posture, different toolchain.

## Related

- [[Languages/TypeScript/Practices/Foundational/Strict Compiler Profile]]
- [[Languages/TypeScript/Practices/Tooling and Quality/Typed Linting with typescript-eslint]]
- [[Languages/TypeScript/Practices/Tooling and Quality/Reference ESLint Config]]
- [[Languages/TypeScript/Practices/Type-Driven Design/Satisfies for Configuration Tables]]
