---
title: TypeScript Execution
tags: [workspace, typescript, build, tsc, ts-node]
summary: Precompile to JavaScript with `tsc` for production. Reserve `ts-node` and `--experimental-transform-types` for local scripts. Shipping TypeScript through a runtime stripper in production is a stability risk that compile-then-deploy avoids.
keywords: [tsc, ts-node, tsx, experimental transform types, runtime stripping, production deploy, build target]
---

*Compile TypeScript at build time, run JavaScript in production. Runtime type-stripping is a developer-loop convenience; in production it is a stability risk that introduces no benefit the build can't already provide.*

# TypeScript Execution

> **Rule:** production deployments run JavaScript that was compiled from TypeScript at build time. `tsc --build` (or an equivalent emit step) is the canonical path. `ts-node`, `tsx`, and Node's `--experimental-transform-types` are appropriate for local scripts and developer iteration; they are not appropriate for production critical-path code.

The reasoning is structural, not religious: the build step is where the type checker runs, where `noEmitOnError` halts on a type error, where the artifact you ship is the artifact you tested, and where runtime startup is fast because there's nothing to strip. Each of those properties is given up when execution becomes "load TypeScript, transform, run."

## The four execution modes

### `tsc` (precompile, run JavaScript) — production canonical

```sh
tsc --build       # build all referenced projects
node dist/index.js
```

The compiled output is plain JavaScript. The runtime is unmodified Node. The artifact you deploy is the artifact your tests ran against. This is the canonical production path.

### `ts-node`, `tsx`, `bun` (transform on load) — developer convenience

`ts-node` and `tsx` transform TypeScript at module load time. They're useful for:

- Quick scripts where the compile step would be friction.
- REPL exploration.
- Test runners that haven't committed to a separate build step.
- Editor-driven workflows where saving and running should be immediate.

The trade-off:

- **Type-checking is decoupled from execution.** Most "TypeScript loaders" prioritize speed over type-checking; they strip types and run, but they don't reject on type errors. You're running TS that may not type-check. CI must run a separate `tsc --noEmit` pass to catch type errors.
- **Startup cost is paid every run.** Production servers pay this on every restart and every worker spawn.
- **Runtime semantics may differ from `tsc` emit.** Most loaders use a different transformer (esbuild, swc) with subtly different decoupling of type-only vs runtime constructs. The differences are small but real.

For local-loop work, the convenience justifies the trade-offs. For production, it doesn't.

### Node's `--experimental-transform-types` — emerging, not yet production

Modern Node (since v22.6 with `--experimental-strip-types`, broader transformation in v22.7+ behind `--experimental-transform-types`) can run `.ts` files directly. The flag is *experimental* — Node explicitly marks it so. Adopting it for production critical paths today means depending on a feature whose semantics may shift across Node minor versions.

The experimental nature is the deciding factor. Watch the feature graduate to stable; until then, treat it like any other experimental flag — acceptable for local scripts, unsuitable for production builds where you've committed to running for years.

### `bun` runtime — separate concern

`bun` is a separate runtime that natively understands TypeScript. If your project has chosen `bun` over Node, that's a different runtime-suitability decision (see [[Languages/TypeScript/Practices/Foundational/Runtime Suitability Boundary]]) and `bun`'s native TS handling is part of that runtime's identity. The vault doesn't take a position on Node vs Bun, but a project that has chosen Bun is, by definition, not running compiled-then-deployed JavaScript on Node — different rules apply.

## What "precompile" actually requires

For `tsc --build` to be a production-grade execution path, the build pipeline must:

1. **Run `tsc` (not just a type-strip transformer).** Only `tsc` checks types. esbuild, swc, ts-loader-without-type-check do not.
2. **Set `noEmitOnError: true`.** Type errors halt the build. See [[Languages/TypeScript/Practices/Tooling and Quality/Reference TSConfig]].
3. **Produce source maps.** Production stack traces should resolve back to TS source lines. `sourceMap: true`, `inlineSources: false`.
4. **Include declaration emit when publishing.** Libraries publish `.d.ts` files; services don't need to.
5. **Cache between builds.** `incremental: true` and `tsBuildInfoFile` make subsequent builds fast.

## Speed vs correctness in CI

The temptation in CI is to use a fast transformer (esbuild, swc) for build and run `tsc --noEmit` separately for type-checking. This is fine *as long as both steps run* and the build fails on either. Skipping `tsc --noEmit` because "the transformer succeeded" produces deployable artifacts whose type-correctness was never verified.

The pragmatic split:

- **Build artifact**: esbuild or swc for speed.
- **Type check**: `tsc --noEmit` running in parallel as a required CI step.
- **Both must pass** before merge.

Don't conflate "fast transformer" with "checks types"; they're separate concerns.

## When you actually want `ts-node` in CI

Two narrow cases:

- **Database migration scripts** that run TypeScript directly because a build-then-run cycle would be friction in a deploy script. The migration runner is short-lived; the trade-off is acceptable.
- **One-off operational scripts** in `tools/` or `scripts/` that aren't part of the deployed artifact.

In both cases, the `ts-node` invocation is for scripts whose footprint is small and whose lifetime is bounded. It is not for the service that handles production traffic.

## Composing with other practices

- [[Languages/TypeScript/Practices/Tooling and Quality/Reference TSConfig]] — sets the compiler options that make `tsc` build production-ready output.
- [[Languages/TypeScript/Workspace/Bundling Decision]] — the next decision after compile: do you bundle the `.js` output or ship it directly?
- [[Languages/TypeScript/Workspace/Project Layout]] — workspace shape affects whether `tsc --build` operates on one package or a project-references graph.
- [[Languages/TypeScript/Workspace/Release Engineering/Release Gate Pipeline]] — the pipeline that runs `tsc` and rejects on type error.
- [[Engineering Philosophy/Principles/Layered Risk Categories]] — execution-mode choice is a runtime-layer concern.

## Related

- [[Languages/TypeScript/Workspace/TypeScript Workspace Architecture MOC]]
- [[Languages/TypeScript/Workspace/Project Layout]]
- [[Languages/TypeScript/Workspace/Bundling Decision]]
- [[Languages/TypeScript/Practices/Tooling and Quality/Reference TSConfig]]
- [[Languages/TypeScript/Practices/Foundational/Runtime Suitability Boundary]]
