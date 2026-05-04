---
title: Test Design Bundle
tags: [bundle, testing, typescript]
summary: "Pre-bundled test design for TypeScript: the Evidence Ladder, the canonical TS tool per rung, the adversarial posture, sharp oracles, and the rule that normal usage isn't testing."
source_trigger: "Writing tests"
bundles: [TypeScript Test Tooling, Evidence Ladder for Testing, Sharp Oracles, Tests Should Make Programs Fail, Normal Usage Is Not Testing]
---

*The through-line: tests prove specific claims at specific layers, with sharp oracles, against adversarial inputs — running the program in normal use is not testing.*

# Test Design Bundle (TypeScript)

Read this bundle when authoring tests for a TypeScript project. It bundles the universal Evidence Ladder, the canonical TypeScript tool per rung, the adversarial posture, sharp-oracle discipline, and the explicit framing that normal-usage validation is not the same thing as testing.

---

## TypeScript Test Tooling

*The universal Evidence Ladder picks which layers to test at. This note picks which TypeScript tools implement each layer.*

> The test strategy follows [[Engineering Philosophy/Principles/Evidence Ladder for Testing]]; the tool selection follows from which rungs the project commits to. Picking *which* rungs is a project decision; picking *what* implements each rung should not be.

### The mapping

| Rung | Canonical TS tool | Notes |
|---|---|---|
| Unit | [`node:test`](https://nodejs.org/api/test.html) (built-in) | Stable since v20; mocking, watch, coverage. |
| Unit (alt) | Vitest | Fast watch ergonomics; choose when toolchain is Vite. |
| Integration | `node:test` + Testcontainers | Real ephemeral dependencies; replaces brittle mocks. |
| End-to-end | Playwright | Browser-context isolation, auto-waiting, network interception. |
| Property-based | fast-check | Deterministic seeded runs, automatic shrinking. |
| Fuzzing | Jazzer.js | Coverage-guided libFuzzer harness; for parsers and protocol code. |
| Mutation | Stryker.js | `thresholds` config; the *target* score is your policy. |
| Formal modeling | TLA+ / Alloy / P | See [[Engineering Philosophy/Principles/Formal Modeling for Distributed State]]. |

### Per-rung discipline

**Unit (`node:test`).** Tests pure domain logic, type invariants, edge cases. No real time, no network, no filesystem. Deterministic with fixed seeds. Coverage thresholds are policy choices, applied per-layer not per-line.

**Integration (Testcontainers).** Real Postgres, real Kafka, real Redis spun up per test or per suite. Replaces brittle mocks for everything except true nondeterminism (real time, randomness, external services with no test instance) and rare fault injection.

**End-to-end (Playwright).** Browser-context isolation per test means tests don't contaminate each other. Cover golden paths plus safety/failure paths. Don't try to cover every UI state with E2E; use unit tests for that.

**Property-based (fast-check).** Mandatory for parsers, transforms, pricing/routing/scheduling rules, idempotency logic. Fixed seeds in CI. Stored failure seeds in a regression corpus.

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
    { seed: 42, numRuns: 1000 }
  );
});
```

**Fuzzing (Jazzer.js).** For untrusted-input surfaces only — parsers, protocol decoders, auth/session/token handling. Run on a separate cadence (nightly) since runtime is too long for per-PR.

**Mutation (Stryker).** Verifies the test suite actually exercises the code. The specific target scores are *policy*, not vendor recommendation. Stryker exposes a generic `thresholds` mechanism but does not endorse particular cutoffs.

### What test runner to *not* configure

A common anti-pattern is running tests with multiple frameworks because different parts of the project picked different ones. Don't. The runner choice should be project-wide. Greenfield projects should pick `node:test` for backend or Vitest for Vite-aligned, and not pick Jest.

*Source: [[Languages/TypeScript/Practices/Testing/TypeScript Test Tooling]]*

---

## Evidence Ladder for Testing

*Each test layer proves a distinct claim. Pick a layer because of the claim it can support, not because the next one was harder to set up.*

> Tests are organized by *what they can prove*, not by where they live in the build. The strongest test suites use multiple layers because each layer answers a question the others cannot.

A useful framing: a test suite is a portfolio of evidence. Each piece of evidence supports a specific claim about the system. A team's testing strategy is choosing which claims to support, at what cost, with which evidence type.

### The ladder, weakest to strongest claim

| Layer | Claim it supports |
|---|---|
| Unit | This function behaves per-contract for the inputs we listed. |
| Integration | These components, wired together, don't break each other. |
| End-to-end | A real user-flow produces the intended outcome. |
| Property-based | An invariant holds across a family of inputs. |
| Fuzzing | The system survives malformed inputs without crashing or corrupting. |
| Fault injection / chaos | The system behaves correctly when dependencies fail. |
| Mutation testing | The test suite itself detects deliberate corruption. |
| Formal modeling | An invariant holds for all reachable states of an abstract model. |

### Reading the ladder

The layers are *independent*: each proves something the others can't, and a gap at one layer is not closed by piling up evidence at another.

The layers don't replace each other. A property-based test does not subsume a unit test, because the unit test's hand-picked example may be the one the property generator never produces.

Strength of claim is not strength of evidence. Formal methods prove the strongest claim, but the claim is about an *abstraction*, not the running code.

Coverage thresholds belong on per-layer claims, not on lines of code. "All public domain functions have at least one property test, and the property test exercises the failure modes the spec describes" tells you something specific. Threshold the claim, not the percentage.

### Choosing layers by criticality

Not every system needs every rung. The selection is a budget decision:

- **Low-stakes / iterative.** Unit + targeted integration. Skip property/fuzz/mutation unless cost-of-being-wrong justifies them.
- **High-stakes service.** Unit + integration + property-based on critical paths + fuzzing on parser/boundary surfaces + chaos on dependency failures.
- **Distributed / state-rich.** All of the above + a formal model for the parts where wrong-state-machine bugs are catastrophic.

The decision criterion is the cost of a missed bug at that layer × the probability the bug exists.

*Source: [[Engineering Philosophy/Principles/Evidence Ladder for Testing]]*

---

## Sharp Oracles

*An assertion is only as strong as its oracle; replace looks-right checks with exact equality, invariants, or differential references.*

> Tests must have **sharp oracles**. A test without a precise verdict is a smoke test, not a check.

For any test:

- **Define concrete expected behavior whenever possible.** Exact outputs over plausibility.
- **Prefer in this order:**
  1. Exact output equality
  2. Invariant checks
  3. Differential checks (reference implementation)
  4. Reference oracles (gold answer)
  5. Precise failure conditions (specific error variant)
- **Avoid** "looks right" as a standard of correctness.

### Why "looks right" fails

A test that asserts only that something happened cannot distinguish a working implementation from one that returns garbage. It will pass on the buggy version. The test does not actually defend the property — it defends the existence of an output. This is how silent corruption ships.

### Oracle techniques in TypeScript

#### Exact output

```ts
assert.deepStrictEqual(parse("8080"), Result.ok(asPort(8080)));
```

#### Invariants via property tests

```ts
fc.assert(fc.property(fc.array(fc.integer()), (input) => {
  const result = sort(input);
  for (let i = 1; i < result.length; i++) {
    if (result[i - 1] > result[i]) return false;
  }
  return [...result].sort().join(",") === [...input].sort().join(",");
}));
```

#### Precise failure conditions

```ts
const result = parse("");
assert.strictEqual(result.ok, false);
assert.strictEqual(result.error.kind, "empty_input");
```

### Anti-patterns

- `assert.ok(result)` — proves only that a value was returned.
- "Visual inspection" of test output — not a test, an opinion.
- A `Result` test that doesn't distinguish error variants.

*Source: [[Engineering Philosophy/Principles/Sharp Oracles]]*

---

## Tests Should Make Programs Fail

*Write tests that try to break the system; a green suite that never failed is a suite that never tested anything important.*

> The purpose of testing is to make the program fail until the important bugs are found.

Tests are not validation theatre — they are an adversary that wants the program to break.

### What this looks like in practice

Create tests that are **intentionally unlike ordinary usage**:

- Combine edge cases in unpleasant ways.
- Exercise limits — off-by-one, integer overflow, capacity boundaries.
- Feed malformed inputs — truncated, null bytes, unicode normalization.
- Boundary values — `0`, `1`, `MAX-1`, `MAX`, `MAX+1`.
- Provoke degenerate states.
- Pass contradictory options.
- Use unusual orderings.
- Repeat operations.
- Empty cases. Maximal cases.
- Invalid transitions.

### The slogan

> Prefer tests that would embarrass a weak design.

A test passes either because the design handles it, or because the test was too gentle. Aim for tests that would expose weakness.

*Source: [[Engineering Philosophy/Principles/Tests Should Make Programs Fail]]*

---

## Normal Usage Is Not Testing

*Running the program in the normal way exercises the happy path; real testing requires deliberate adversarial inputs and edge cases.*

> Real testing means going **outside** the normal range of inputs to find weakness. Normal use exercises only the obvious paths.

The temptation: the program "works" because exercising it produces correct output, so the tests just confirm that fact. The trap: the inputs you reach for under casual use are typical inputs. Casual use exercises the happy path *and only the happy path*.

### Why testing means leaving the comfort zone

A function may produce correct output for inputs you'd ever realistically construct, *and* produce wrong output for inputs you wouldn't think of. The set of inputs that reveals bugs is rarely the set of inputs that look natural to write — it's the inputs you'd never consider writing because they look weird.

### What this looks like in practice

- Don't substitute a "test" for a "demo run." A demo proves the program ran end-to-end; that is barely a smoke test.
- Don't conclude the test pass means correctness. It means the inputs in the test, with their oracles, did not show the bug.
- Don't confuse "test coverage went up" with "more bugs would be caught." Coverage is a necessary signal, not a sufficient one.

### Composing with adversarial testing

This principle is the precondition for [[Engineering Philosophy/Principles/Tests Should Make Programs Fail]]. Adversarial tests *go where normal use does not*. The two principles work together: stop using normal use as the standard of evidence; start using adversarial use as the standard of evidence.

*Source: [[Engineering Philosophy/Principles/Normal Usage Is Not Testing]]*

---

## Related bundles

- [[Bundles/TypeScript/Code-Review]] — review checks that depend on these testing principles
- [[Bundles/TypeScript/Module-Design]] — when tests need heroics, redesign the module
- [[Bundles/TypeScript/Error-Handling]] — error-test discipline for the error-handling slice
- [[Bundles/Universal/Written-Analysis]] — for the test-strategy section of a design doc
