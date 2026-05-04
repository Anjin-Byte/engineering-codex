---
title: TypeScript Test Tooling
tags: [testing, typescript, node, tooling]
summary: Each rung of the universal Evidence Ladder maps to a canonical TS tool — node:test, Testcontainers, Playwright, fast-check, Jazzer.js, Stryker.
keywords: [node test, vitest, testcontainers, playwright, fast-check, jazzer, stryker, mutation testing, property based testing]
---

*The universal Evidence Ladder picks which layers to test at. This note picks which TypeScript tools implement each layer.*

# TypeScript Test Tooling

> **Rule:** the test strategy follows [[Engineering Philosophy/Principles/Evidence Ladder for Testing]]; the tool selection follows from which rungs the project commits to. The mapping below is the canonical TypeScript toolchain for each rung. Picking *which* rungs is a project decision; picking *what* implements each rung should not be.

The Evidence Ladder (universal) lists test layers from weakest claim (a unit example) to strongest (formal modeling). Each layer proves something the others can't. This note shows what tool each rung uses in TypeScript and why.

## The mapping

| Rung | Canonical TS tool | Notes |
|---|---|---|
| Unit | [`node:test`](https://nodejs.org/api/test.html) (built-in) | Stable since v20; mocking, watch, coverage, multiple reporters. |
| Unit (alt) | Vitest | Fast watch ergonomics; Vite-aligned; choose when toolchain is already Vite. |
| Integration | `node:test` + Testcontainers | Real ephemeral databases / brokers / browsers; replaces brittle mocks. |
| End-to-end | Playwright | Browser-context isolation per test, auto-waiting, network interception. |
| Property-based | fast-check | Deterministic seeded runs, automatic shrinking. |
| Fuzzing | Jazzer.js | Coverage-guided libFuzzer harness; for parsers and protocol code. |
| Mutation | Stryker.js | `thresholds` config for `high`/`low`/`break`; the *target* score is your policy. |
| Formal modeling | TLA+ / Alloy / P | See [[Engineering Philosophy/Principles/Formal Modeling for Distributed State]]. Not TS-specific. |

## Per-rung notes

### Unit — `node:test`

[Built into Node](https://nodejs.org/api/test.html), stability 2 — Stable since v20.0.0. Provides subtests, `it.skip` / `it.todo`, function / module / property / timer mocking, snapshot testing, multiple reporters (spec, tap, dot, junit, lcov), watch mode, coverage, and global setup / teardown hooks. No third-party install needed.

**Pick `node:test` when:**
- The project is backend / library code in modern Node.
- You want to minimize the dev-dependency footprint.
- You're already using Node's coverage tooling.

**Pick Vitest when:**
- The project is browser / Vite-aligned (frontend, full-stack with Vite).
- Watch-mode ergonomics are a primary developer-experience concern.
- You want HMR-driven test re-runs.

What you should *not* use for new projects: Jest. Jest works, but Vitest is faster and `node:test` is built-in; Jest's transform pipeline adds CI cost without proportional benefit. Existing Jest projects can keep it; greenfield projects should not pick it.

### Integration — `node:test` + Testcontainers

The unit-test framework runs the test; [Testcontainers](https://node.testcontainers.org/) spins up real ephemeral dependencies. The pattern:

```ts
import { test, before, after } from "node:test";
import { PostgreSqlContainer } from "@testcontainers/postgresql";

let container, db;

before(async () => {
  container = await new PostgreSqlContainer().start();
  db = await connect(container.getConnectionUri());
  await migrate(db);
});

after(async () => {
  await db.end();
  await container.stop();
});

test("transferFunds debits source and credits destination", async () => {
  // Real Postgres, real schema, real transactions
});
```

The reason to prefer real ephemeral containers over mocks: a mocked database doesn't have constraints, doesn't deadlock, doesn't enforce serializability, doesn't reproduce migration drift. Tests against the real engine catch a class of bug mocks structurally cannot. The cost is per-test container startup; Testcontainers' reuse / pooling features mitigate it.

Mocks are still appropriate for *true nondeterminism* (real time, randomness, external services with no test instance) and for *fault injection* (forcing a failure mode the real dependency rarely exhibits). Everything else: real ephemeral.

### End-to-end — Playwright

[Playwright](https://playwright.dev/) is Microsoft's cross-browser E2E testing framework. Its load-bearing feature is **browser-context isolation per test**: each test starts in a fresh, cookies-empty, storage-empty browser context, so tests do not contaminate each other. This eliminates a class of flaky-test failure that plagued earlier-generation E2E tools.

Other features that earn its place: auto-waiting (locators retry until the element exists or times out), trace viewer (visual replay of failed runs), network interception, parallel execution.

The E2E layer should cover **golden paths plus safety / failure paths** — happy-path login, happy-path checkout, plus the cases where the UI must degrade safely. Don't try to cover every UI state with E2E; use unit tests for that. E2E is expensive and slow.

### Property-based — fast-check

[fast-check](https://fast-check.dev/) is the canonical TypeScript property-based testing library. Deterministic seeded runs, automatic shrinking on counterexamples, integrates with `node:test`, Jest, Vitest, and Mocha.

Property-based testing is mandatory for code where the input space is too large to enumerate by example: parsers, transforms, pricing / routing / scheduling rules, idempotency logic, serialization round-trips. The pattern:

```ts
import { test } from "node:test";
import * as fc from "fast-check";

test("serialize/deserialize roundtrip", () => {
  fc.assert(
    fc.property(orderArbitrary, (order) => {
      const bytes = serialize(order);
      const back = deserialize(bytes);
      return deepEqual(order, back);
    }),
    { seed: 42, numRuns: 1000 } // deterministic, reproducible
  );
});
```

Two practices that earn fast-check its place:

- **Fixed seeds in CI.** A property test with a random seed is non-reproducible. Pin the seed; CI runs are deterministic; failures are reproducible.
- **Stored failure seeds.** When fast-check finds a counterexample, save the seed in a regression corpus. Future runs include that seed, ensuring the bug stays caught.

Property tests *complement* example tests; they don't replace them. An example test asserts the function works for the case you thought of; a property test asserts an *invariant* holds across many cases.

### Fuzzing — Jazzer.js

[Jazzer.js](https://github.com/CodeIntelligenceTesting/jazzer.js) is the coverage-guided fuzzing harness for Node, built on libFuzzer. For untrusted-input surfaces — parsers, protocol decoders, auth / session / token handling, deserialization — fuzzing finds bugs that property-based tests don't, because the fuzzer drives input toward unexplored code paths.

Fuzzing is expensive and slow. The discipline:

- Fuzz only the trust-boundary surfaces. Don't fuzz internal utility functions.
- Run fuzzing in a separate CI lane (nightly or on-demand), not on every PR.
- Maintain a regression corpus checked into the repo.

A note on adoption: fuzzing has a real setup cost. For projects without untrusted-input surfaces, the cost rarely justifies the benefit.

### Mutation — Stryker.js

[Stryker.js](https://stryker-mutator.io/) systematically mutates source code (changing operators, deleting statements, etc.) and reports the percentage of mutants the test suite kills. A high mutation score means the suite actually exercises the code; a low score means the suite has coverage but doesn't *check* very much.

Stryker exposes a `thresholds` configuration with `high` / `low` / `break` keys for reporting bands and CI failure cutoffs. **The specific target scores are policy, not vendor recommendation.** Stryker does not publish target numbers. If your standard says "≥70% overall, ≥85% for domain kernels," that is your project's policy choice — defensible, but not endorsed by Stryker or by mutation-testing literature in any specific number form.

The literature *does* support the underlying claim that line/branch coverage alone is insufficient (Andrews/Briand/Labiche 2005, Inozemtseva/Holmes 2014). The mutation-score thresholds you adopt rest on that broader claim, not on a vendor-published target.

### Formal modeling — out of scope for the toolchain

If the project's risk profile justifies formal modeling, the right tool is TLA+ or Alloy or P, not a TypeScript library. See [[Engineering Philosophy/Principles/Formal Modeling for Distributed State]] for the cost / benefit analysis.

## What test runner to *not* configure

A common anti-pattern is running tests with multiple frameworks because different parts of the project picked different ones. Don't. The runner choice should be project-wide. The cost is the friction of one-time migration; the benefit is one set of fixtures, one CI lane, one mocking convention. If the project is large enough that team A picked Vitest and team B picked Jest, the benefit is having team A and team B converge.

## Composing with other practices

- [[Engineering Philosophy/Principles/Evidence Ladder for Testing]] — picks which rungs to invest in.
- [[Engineering Philosophy/Principles/Sharp Oracles]] — every test, regardless of rung, needs a sharp oracle.
- [[Engineering Philosophy/Principles/Tests Should Make Programs Fail]] — adversarial posture is layer-independent.
- [[Engineering Philosophy/Principles/Regression Discipline]] — every confirmed bug becomes a permanent test, ideally at the rung it would have been caught earliest.
- [[Languages/TypeScript/Practices/Tooling and Quality/Reference TSConfig]] — test files participate in the same TS strictness profile.

## Related

- [[Engineering Philosophy/Principles/Evidence Ladder for Testing]]
- [[Engineering Philosophy/Principles/Sharp Oracles]]
- [[Engineering Philosophy/Principles/Tests Should Make Programs Fail]]
- [[Engineering Philosophy/Principles/Regression Discipline]]
- [[Engineering Philosophy/Principles/Formal Modeling for Distributed State]]
- [[Languages/Rust/Practices/Testing/Three Levels of Tests]]
- [[Languages/Rust/Practices/Testing/Edge Cases and Properties]]
