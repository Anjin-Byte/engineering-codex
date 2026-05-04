---
title: Workspace Architecture MOC
tags: [moc, workspace, typescript]
summary: Map of content for shaping a TypeScript project — directory layout, execution mode, bundling decisions, and the release-engineering pipeline that ships it.
keywords: [workspace, project layout, navigation hub, build configuration, release engineering, typescript]
---

*A non-trivial TypeScript project is shaped by four decisions — layout, execution, bundling, release pipeline — each independent and each load-bearing.*

# Workspace Architecture — Map of Content

Entry point for understanding **how** a TypeScript project is shaped at the directory and pipeline level, and **where** to look for any specific architectural decision.

## The thesis

> A TypeScript project's outer shape is a small set of decisions about *deployable-unit boundaries*, *execution mode*, *bundling*, and *release pipeline*. The decisions interact: project layout determines what `tsc --build` operates on; execution mode determines whether bundling matters; release pipeline gates them all on quality signals. The notes below treat each decision in isolation, then their composition.

## Foundational reading order

1. [[Languages/TypeScript/Workspace/Project Layout]] — the deployable-unit shape.
2. [[Languages/TypeScript/Workspace/TypeScript Execution]] — `tsc` precompile vs ts-node vs experimental loaders.
3. [[Languages/TypeScript/Workspace/Bundling Decision]] — bundle the output or ship it directly.
4. [[Languages/TypeScript/Workspace/Release Engineering/Release Gate Pipeline]] — what gates exist before production.

## Workspace decisions

- [[Languages/TypeScript/Workspace/Project Layout]] — single-package, npm workspaces, or project-references monorepo.
- [[Languages/TypeScript/Workspace/TypeScript Execution]] — precompile to JavaScript with `tsc`; reserve `ts-node` and runtime stripping for local scripts.
- [[Languages/TypeScript/Workspace/Bundling Decision]] — don't bundle long-lived services; do bundle for browser, single-file artifacts, and cold-start-sensitive serverless.

## Release engineering

- [[Languages/TypeScript/Workspace/Release Engineering/Release Gate Pipeline]] — typecheck, typed lint, tests, security scanning, audit, provenance, canary, SLO watch.
- [[Languages/TypeScript/Workspace/Release Engineering/npm Lockfile and Install Discipline]] — committed lockfile, `npm ci`, audit policy.
- [[Languages/TypeScript/Workspace/Release Engineering/Provenance and SBOM]] — npm provenance attestations, CycloneDX vs SPDX SBOMs.
- [[Languages/TypeScript/Workspace/Release Engineering/Permission Model Adoption]] — Node's `--permission` flag as defense in depth.

## Cross-cutting universal principles

The workspace shape and release pipeline are the TS application of universal principles:

- [[Engineering Philosophy/Principles/Architectural Core Principles]] — deployable-unit modularity drives Project Layout.
- [[Engineering Philosophy/Principles/One Binary or Many]] — same principle, applied to npm packages and binaries.
- [[Engineering Philosophy/Principles/Staged Canary Deployment]] — what the release gate guards.
- [[Engineering Philosophy/Principles/Error Budgets and Reliability Policy]] — the budget the canary observes.
- [[Engineering Philosophy/Principles/Configuration as Code Review]] — applies to lockfiles, CI configuration, and release-gate definitions.

## The condensed rules

1. Layout reflects deployable units; don't reach for monorepo until the units actually diverge.
2. Compile TypeScript at build time; run JavaScript in production.
3. Don't bundle long-lived services; bundle for browser, single-file, and cold-start cases.
4. Lockfiles are committed; `npm ci` runs in CI; audits are gates, not warnings.
5. Critical-path packages publish with provenance attestations and SBOMs.
6. Releases gate on the full pipeline, not on a subset chosen for speed.

## Related

- [[Languages/TypeScript/AGENTS]]
- [[Languages/TypeScript/Practices/TypeScript Practices MOC]]
- [[Languages/Rust/Workspace/Workspace Architecture MOC]]
- [[Engineering Philosophy/Principles/Principles Index]]
