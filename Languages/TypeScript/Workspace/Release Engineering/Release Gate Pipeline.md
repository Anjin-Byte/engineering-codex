---
title: Release Gate Pipeline
tags: [release-engineering, typescript, ci, pipeline, quality-gate]
summary: A critical-path TypeScript service merges only when every gate is green — typecheck, typed lint, tests, SAST, audit, provenance, SBOM, canary, SLO watch.
keywords: [ci pipeline, quality gates, codeql, semgrep, sast, npm audit, canary, slo watch, merge gate]
---

*The release gate is the place the policy gets enforced. If a check is recommended but not enforced by the pipeline, it doesn't exist.*

# Release Gate Pipeline

> **Rule:** a critical-path TypeScript service does not merge to `main` (and does not deploy to production) until *every* gate below has passed. The gates are not advisory; the pipeline blocks. Manual override is allowed only with an explicit, audited exception process — the override is itself a gate.

This is the TypeScript-specific operationalization of [[Engineering Philosophy/Principles/Staged Canary Deployment]] and [[Engineering Philosophy/Principles/Error Budgets and Reliability Policy]]. Those principles define *what* the release should do; this note defines *how* a TypeScript pipeline does it.

## The pipeline

In rough order, the gates a critical-path TS service merges through:

```
PR open
  │
  ├─ Typecheck (tsc --noEmit)
  ├─ Typed lint (typescript-eslint with strictTypeChecked)
  ├─ Unit tests (node:test or Vitest)
  ├─ Integration tests (Testcontainers + node:test)
  ├─ Property-based tests (fast-check, with stored failure seeds)
  ├─ Security scanning (CodeQL + Semgrep, project-specific rules)
  ├─ Dependency review (changed dependencies vs allowlist)
  ├─ Lockfile integrity (`npm ci` passes; lockfile committed)
  ├─ Audit (npm audit with tiered policy; see npm Lockfile note)
  ├─ Provenance attestation (for published packages)
  ├─ SBOM generation (CycloneDX and/or SPDX)
  │
  ▼ All required checks green → reviewer approves
  │
Merge to main
  │
  ├─ Build artifact (tsc + bundling if applicable)
  ├─ Deploy to staging
  ├─ Smoke tests
  │
  ▼
Canary deploy to production (1% → 10% → 50% → 100%)
  │
  ├─ Compare canary metrics vs control population
  ├─ Auto-rollback on SLO regression
  │
  ▼
Promote to 100% → post-canary SLO watch (24h–7d) → release closed
```

Fuzzing (Jazzer.js) and mutation testing (Stryker) typically run on a separate cadence — nightly or weekly — because their runtime is too long for per-PR gating. Both feed back into the next PR's evidence.

## Per-gate detail

### Typecheck

`tsc --noEmit` runs the type checker over the entire project (or all referenced projects in a project-references monorepo). With [[Languages/TypeScript/Practices/Foundational/Strict Compiler Profile]] enabled and `noEmitOnError`, a type error is a hard fail. This is the cheapest gate by orders of magnitude; it should be the first one the pipeline runs.

### Typed lint

[typescript-eslint](https://typescript-eslint.io/getting-started/typed-linting/) with `recommendedTypeChecked` + `strictTypeChecked` + `stylisticTypeChecked` plus the explicit safety overrides in [[Languages/TypeScript/Practices/Tooling and Quality/Reference ESLint Config]]. Gate on zero errors. Warnings can be allowed but should also be tracked over time.

### Unit and integration tests

`node:test` (or Vitest) runs unit tests; Testcontainers spins up real ephemeral databases / brokers / browsers for integration tests. See [[Languages/TypeScript/Practices/Testing/TypeScript Test Tooling]] for tool selection. Coverage thresholds are policy choices — line coverage ≥80% is a common starting point but is project-specific.

### Property-based tests

[fast-check](https://fast-check.dev/) with fixed CI seeds and a checked-in regression corpus. The regression corpus contains seeds for every previously-found counterexample; new PRs run against the corpus to ensure regressions don't reintroduce.

### Security scanning

Two complementary tools:

- **[CodeQL](https://docs.github.com/en/code-security/code-scanning/introduction-to-code-scanning/about-code-scanning-with-codeql)** — GitHub's semantic-analysis engine with first-party JS/TS query packs covering taint tracking, injection, prototype pollution, and broad CWE coverage. Runs natively in GitHub Actions.
- **[Semgrep](https://semgrep.dev/)** — rule-based static analysis with custom-rule support. Faster to author project-specific rules than CodeQL's QL.

Both as required gates. CodeQL covers breadth; Semgrep covers project-specific policy.

### Dependency review

For PRs that change `package.json` or the lockfile, a dependency-review check that:

- Flags new dependencies for review.
- Cross-checks against an allowlist or block-list (project-specific).
- Reads OpenSSF Scorecard signals where available (see [[Languages/TypeScript/Workspace/Release Engineering/Provenance and SBOM]]).

### Lockfile and audit

See [[Languages/TypeScript/Workspace/Release Engineering/npm Lockfile and Install Discipline]]. The pipeline runs `npm ci`, fails on lockfile drift, and runs `npm audit` against a tiered policy. High and critical vulnerabilities block; medium and low generate findings tracked over time.

### Provenance and SBOM

For packages that publish to npm, the publish workflow generates [npm provenance attestations](https://docs.npmjs.com/generating-provenance-statements) signed via Sigstore from the public CI workflow. SBOM generation produces CycloneDX or SPDX (or both) — see [[Languages/TypeScript/Workspace/Release Engineering/Provenance and SBOM]].

For services (not published packages), provenance is optional but SBOM is still useful for vulnerability response.

### Canary deploy + SLO watch

The deploy half of the pipeline implements [[Engineering Philosophy/Principles/Staged Canary Deployment]]:

- Canary at 1%, 10%, 50%, 100%.
- Compare canary vs control on error rate, latency p99, event-loop delay (see [[Languages/TypeScript/Practices/Async and Concurrency/Event Loop as Correctness Signal]]), and any service-specific SLOs.
- Auto-rollback on regression.
- Post-canary SLO watch (24 hours minimum for high-traffic services; up to 7 days for lower-traffic) before closing the release as successful.

## Per-tier rigor

Not every TypeScript service needs every gate. The bar scales with consequence-of-failure:

- **Critical (production payment, security-relevant, irrecoverable side effects)**: every gate above.
- **High (production user-facing, recoverable)**: every gate except possibly the property/fuzz/mutation cadence may be lighter.
- **Standard (internal tools, low-traffic, easily-rebuilt)**: typecheck + lint + tests + dependency audit; canary may simplify to staged manual rollout.

The point isn't to gold-plate every project's pipeline; it's to make the gating deliberate. A service that *should* be in the Critical tier but runs the Standard pipeline is exactly the kind of mismatch [[Engineering Philosophy/Principles/Layered Risk Categories]] exists to flag.

## Override and exception

A Critical-tier pipeline that allows manual override without audit is a fiction. The discipline:

- Override requires an explicit reason, recorded in the PR.
- Override requires an approver who is not the author.
- Override is logged centrally and reviewed periodically (rate of overrides is itself a metric).
- Some gates (security, provenance) may be configured to allow no override, or only with a separate security-team approval path.

If overrides are rare, the gate is real. If overrides are constant, the gate is theater and either the policy needs to change or the pipeline needs to be tuned.

## Composing with other practices

- [[Engineering Philosophy/Principles/Staged Canary Deployment]] — what the deploy half operationalizes.
- [[Engineering Philosophy/Principles/Error Budgets and Reliability Policy]] — the SLO watch's budget.
- [[Engineering Philosophy/Principles/Layered Risk Categories]] — which tier a service is in determines which gates apply.
- [[Languages/TypeScript/Workspace/Release Engineering/npm Lockfile and Install Discipline]] — the install-time gate.
- [[Languages/TypeScript/Workspace/Release Engineering/Provenance and SBOM]] — the publish-time gate.
- [[Languages/TypeScript/Practices/Tooling and Quality/Typed Linting with typescript-eslint]] — the lint gate.
- [[Languages/TypeScript/Practices/Testing/TypeScript Test Tooling]] — the test-rung tooling.

## Related

- [[Languages/TypeScript/Workspace/TypeScript Workspace Architecture MOC]]
- [[Languages/TypeScript/Workspace/Release Engineering/npm Lockfile and Install Discipline]]
- [[Languages/TypeScript/Workspace/Release Engineering/Provenance and SBOM]]
- [[Languages/TypeScript/Workspace/Release Engineering/Permission Model Adoption]]
- [[Engineering Philosophy/Principles/Staged Canary Deployment]]
- [[Engineering Philosophy/Principles/Error Budgets and Reliability Policy]]
