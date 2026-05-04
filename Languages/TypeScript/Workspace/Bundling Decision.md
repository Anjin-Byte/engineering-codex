---
title: Bundling Decision
tags: [workspace, typescript, bundle, esbuild, deployment]
summary: Long-lived services should not bundle. Reach for esbuild only when single-file artifacts, cold-start time, or browser-shipping constraints justify it; source maps and debugger UX are the hidden cost teams underestimate.
keywords: [bundling, esbuild, webpack, rollup, source maps, debugging, single file, cold start, tree shaking]
---

*Don't bundle long-lived services. Bundling is right for browsers and short-lived workers; for everything else, the debugging and source-map cost rarely justifies the artifact savings.*

# Bundling Decision

> **Rule:** the default for a Node service is to ship the `tsc`-emitted JavaScript directly, not pre-bundled. Reach for a bundler (esbuild typically) only when a specific concrete benefit applies — single-file artifact, cold-start optimization, or browser/client distribution. Each of those has a counterweight cost in source-map fidelity and debugging surface.

The bundling debate is often framed as "fast vs correct" or "modern vs legacy." Neither framing is useful. The actual decision is operational: does the runtime environment benefit from a single bundled artifact, or does it not?

## When *not* to bundle (most Node services)

A long-running Node service in a containerized or VM-based deployment has none of the bundling benefits:

- The container ships its `node_modules/` already; bundling doesn't reduce the artifact.
- Cold start is once per container restart; saving 50ms there doesn't matter against a 5-minute uptime.
- Tree-shaking unused dependencies is a build-time win; for a long-lived process, the unused code costs nothing at runtime.
- Module resolution at runtime is fast in modern Node; the overhead bundling avoids isn't material.

Meanwhile bundling adds:

- **Source-map indirection.** Every stack trace must resolve through a source map to point at real source. The trace that says `bundle.js:1:284910` is harder to reason about than `dist/api/handler.js:42`.
- **Debugger friction.** Setting a breakpoint in source means the debugger must map to the bundle. Most modern debuggers handle it; some don't.
- **Build-time cost.** Bundling isn't free. `tsc --build` is already in the pipeline; bundling is an extra step.
- **Bundler-specific quirks.** Each bundler has edge cases around dynamic imports, CommonJS interop, native modules, and `__dirname`. None are showstoppers, all are noise.

The default for a Node service: don't bundle.

## When to bundle

The benefits accrue in specific cases:

### Browser / client distribution

A frontend SPA, a browser library, a UI bundle delivered over HTTP — bundling is mandatory because the runtime is a browser, the distribution is over the network, and tree-shaking + minification meaningfully reduce download size. esbuild, rspack, rolldown, Vite's underlying tooling all serve this case.

This is the case the bundling ecosystem was built for. The trade-offs reverse: source-map fidelity is a problem you actively manage, not a benefit you give up.

### Single-file artifact for portability

Some operational requirements expect a single distributable file: serverless functions where the deploy unit is a zip, edge runtimes (Cloudflare Workers, Deno Deploy), CLI tools shipped as a single binary via something like `pkg` or `bun build --compile`. Here the artifact format dictates bundling.

### Cold-start optimization in serverless

A Lambda or similar function whose performance budget is dominated by cold-start may benefit from bundling because module-resolution time at cold start is significant. Measure first; the gain is real but not always large enough to justify the debugging cost.

### Library publishing

A library publishing to npm typically ships:

- The `tsc` `.js` output for module consumers.
- The `.d.ts` declarations for type consumers.

Bundling is generally inappropriate for libraries — consumers' bundlers do their own tree-shaking, and a pre-bundled library defeats that. The exception is libraries with native or very large internal dependencies that benefit from being pre-bundled into a single file. Most libraries: don't bundle.

## When in doubt, esbuild

If you're going to bundle, esbuild is the canonical choice for TypeScript projects:

- Native TypeScript support without a separate transformer.
- 10–100× faster than webpack/rollup on most projects.
- Small enough configuration surface that the build is easy to reason about.
- Reasonable defaults; minimal config required.

Webpack remains common in legacy projects and has the broadest plugin ecosystem; new projects rarely need that ecosystem and esbuild's simplicity wins. Rollup is excellent for libraries; esbuild is acceptable for libraries too. Vite uses esbuild + Rollup under the hood and is the right choice when the project is already Vite-shaped.

## Bundling and the rest of the build

Bundling sits *after* `tsc --build` in the pipeline:

```
tsc --build  →  dist/*.js  →  esbuild bundle  →  bundle.js
        (type-check)        (transform/minify)
```

Don't replace `tsc` with esbuild's TypeScript handling. esbuild strips types but does not type-check. Run both: `tsc --noEmit` for the type check, esbuild for the bundle. (Same point as [[Languages/TypeScript/Workspace/TypeScript Execution]].)

## Source maps when you do bundle

If you bundle, source maps are non-negotiable for any code that runs in production:

- `--sourcemap` (esbuild) or equivalent.
- Source maps are deployed alongside the bundle (in production, behind authentication if you don't want them publicly accessible — many error-tracking services accept uploaded source maps separately).
- Stack traces in error reporting (Sentry, OpenTelemetry, custom logger) resolve through the source map.

A bundle with no production source map is a bundle whose stack traces are useless.

## Composing with other practices

- [[Languages/TypeScript/Workspace/TypeScript Execution]] — bundling sits after the `tsc` build; it doesn't replace it.
- [[Languages/TypeScript/Workspace/Project Layout]] — what the build produces depends on whether the project is single-package or multi-package.
- [[Languages/TypeScript/Practices/Foundational/Runtime Suitability Boundary]] — the runtime environment (server vs edge vs browser) shapes the bundling decision.
- [[Engineering Philosophy/Principles/Architectural Core Principles]] — the deployable-unit principle decides what counts as a single artifact.

## Related

- [[Languages/TypeScript/Workspace/TypeScript Workspace Architecture MOC]]
- [[Languages/TypeScript/Workspace/TypeScript Execution]]
- [[Languages/TypeScript/Workspace/Project Layout]]
- [[Languages/TypeScript/Practices/Foundational/Runtime Suitability Boundary]]
