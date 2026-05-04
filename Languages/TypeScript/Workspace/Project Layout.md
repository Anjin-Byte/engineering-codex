---
title: Project Layout
tags: [workspace, typescript, monorepo, project-references]
summary: A non-trivial TypeScript project's directory shape encodes its deployable-unit boundaries — single-package, npm workspaces, or project-references monorepo. Pick by where the build, test, and ship boundaries actually fall.
keywords: [monorepo, npm workspaces, project references, package layout, src structure, tsconfig hierarchy]
---

*Make the layout reflect the deployable units. If two parts ship together, package them together; if they ship apart, separate them — even when they live in one repo.*

# Project Layout

> **Rule:** the directory shape of a TypeScript project encodes which parts are independently buildable, testable, and shippable. Three shapes cover almost every case: single package, npm workspaces, and project-references monorepo. The right one is the one whose boundaries match where the project's deployable units actually fall.

This is the TypeScript expression of [[Engineering Philosophy/Principles/Architectural Core Principles]] rule 3 ("Make the project shape reflect deployable units"). The principle is universal; the mechanism is `package.json` and `tsconfig.json` arrangement.

## Three canonical shapes

### Shape 1 — Single package

```
project-root/
├── package.json
├── tsconfig.json
├── eslint.config.js
├── src/
│   ├── index.ts
│   ├── domain/
│   ├── adapters/
│   └── api/
├── test/
└── dist/                    # gitignored, build output
```

**Use when:** the project ships as a single npm package, a single binary, or a single service. One `package.json`, one `tsconfig.json`, one build target. This is the default; reach for the more complex shapes only when the project's deployable units genuinely diverge.

### Shape 2 — npm workspaces

```
project-root/
├── package.json             # root, declares workspaces
├── tsconfig.json            # root, optional shared compiler base
├── packages/
│   ├── core/
│   │   ├── package.json
│   │   ├── tsconfig.json
│   │   └── src/
│   ├── api-server/
│   │   ├── package.json
│   │   ├── tsconfig.json
│   │   └── src/
│   └── cli/
│       ├── package.json
│       ├── tsconfig.json
│       └── src/
└── pnpm-workspace.yaml      # if using pnpm; otherwise the root package.json's "workspaces" field
```

The root `package.json` declares the member packages:

```jsonc
{
  "name": "project-root",
  "private": true,
  "workspaces": ["packages/*"]
}
```

**Use when:** several packages live in one repo and depend on each other locally during development, but ship as independent npm artifacts. Each package has its own `package.json`, its own version, its own published artifact. Cross-package imports during development resolve to the local path; at publish time they resolve to the published version.

The three major package managers (npm, pnpm, Yarn) all support workspaces; the configuration syntax differs slightly. Pick one project-wide.

### Shape 3 — Project references (monorepo with TypeScript compilation graph)

```
project-root/
├── package.json
├── tsconfig.json            # root, references the member tsconfigs
├── packages/
│   ├── core/
│   │   ├── package.json
│   │   ├── tsconfig.json    # composite: true
│   │   └── src/
│   ├── api-server/
│   │   ├── package.json
│   │   ├── tsconfig.json    # composite: true; references ["../core"]
│   │   └── src/
│   └── cli/
│       ├── package.json
│       ├── tsconfig.json    # composite: true; references ["../core"]
│       └── src/
```

The root `tsconfig.json`:

```jsonc
{
  "files": [],
  "references": [
    { "path": "./packages/core" },
    { "path": "./packages/api-server" },
    { "path": "./packages/cli" }
  ]
}
```

Each member's `tsconfig.json` has `"composite": true` and its own `references` field listing dependency packages.

**Use when:** the project has a TypeScript compilation graph between packages where one depends on another's *types* directly, and you want incremental builds across the graph. Project references make `tsc --build` aware of the dependency order, build only what changed, and emit `.d.ts` files at package boundaries. The [TypeScript handbook on project references](https://www.typescriptlang.org/docs/handbook/project-references.html) is the canonical reference.

Project references compose with npm workspaces: most monorepos use both. Workspaces handle package linkage; project references handle the TypeScript compilation graph.

## When to add structure

The progression is:

1. **Start with Shape 1 (single package).** Move to a more complex shape only when forced.
2. **Move to Shape 2 (workspaces) when** a clear second deployable unit emerges — a separate published package, a separate service, a separate CLI tool that should version independently.
3. **Move to Shape 3 (project references) when** the workspace has 3+ packages with TypeScript dependencies between them and `tsc` build times become a developer-experience problem.

Premature monorepo structure is a real cost. It complicates dependency management, doubles the tooling configuration surface, and makes "just publish this fix" harder. Don't reach for it until the project's deployable units actually diverge.

## Where source lives within a package

Within any single package, the convention is:

- `src/` — TypeScript source. The only thing the compiler reads.
- `dist/` (or `build/`, `out/`) — emit target. Gitignored. Published if the package is published.
- `test/` (or co-located `*.test.ts` next to source) — test files.
- `tools/` — scripts, codemods, build helpers.

The `tsconfig.json` `rootDir` and `outDir` enforce the `src/` → `dist/` mapping (see [[Languages/TypeScript/Practices/Tooling and Quality/Reference TSConfig]]).

## Shared compiler config in workspaces

When multiple packages share a strictness profile, factor it out:

```jsonc
// tsconfig.base.json (root)
{
  "compilerOptions": {
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "exactOptionalPropertyTypes": true,
    /* ... see Reference TSConfig ... */
  }
}
```

Each package's `tsconfig.json`:

```jsonc
{
  "extends": "../../tsconfig.base.json",
  "compilerOptions": {
    "outDir": "./dist",
    "rootDir": "./src",
    "composite": true                  // if using project references
  },
  "include": ["src/**/*"]
}
```

This keeps the strictness story uniform without duplicating the profile across every package.

## What to *not* do

- **Do not put TypeScript source files at the package root.** `src/` is the convention; flat layouts make `tsc` configuration harder and confuse tooling.
- **Do not commit `dist/`.** Build output is reproducible; commit only source.
- **Do not mix CommonJS and ESM emit in one package without explicit dual-publish setup.** Pick `module: "NodeNext"` and stick.
- **Do not skip `package.json` for internal-only packages in a workspace.** The package.json is what makes them a workspace member.

## Composing with other practices

- [[Languages/TypeScript/Practices/Tooling and Quality/Reference TSConfig]] — the per-package compiler config.
- [[Languages/TypeScript/Workspace/TypeScript Execution]] — what the layout produces when built.
- [[Languages/TypeScript/Workspace/Bundling Decision]] — whether the dist/ output goes into a bundler or ships directly.
- [[Engineering Philosophy/Principles/Architectural Core Principles]] — the deployable-unit principle this layout encodes.
- [[Engineering Philosophy/Principles/One Binary or Many]] — the principle behind splitting into multiple packages.

## Related

- [[Languages/TypeScript/Workspace/TypeScript Workspace Architecture MOC]]
- [[Languages/TypeScript/Workspace/TypeScript Execution]]
- [[Languages/TypeScript/Workspace/Bundling Decision]]
- [[Languages/TypeScript/Practices/Tooling and Quality/Reference TSConfig]]
- [[Engineering Philosophy/Principles/Architectural Core Principles]]
- [[Languages/Rust/Workspace/Workspace Layout]]
