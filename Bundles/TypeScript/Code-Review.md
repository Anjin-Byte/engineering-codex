---
title: Code Review Bundle
tags: [bundle, code-review, typescript]
summary: "Pre-bundled review checks: TypeScript practices checklist plus the testing principles that decide whether a diff is genuinely defended."
source_trigger: "Reviewing a PR or diff"
bundles: [TypeScript Practices Checklist, Tests Should Make Programs Fail, Sharp Oracles, Regression Discipline, Evidence Ladder for Testing]
---

*The through-line: a TypeScript PR is acceptable when its code shape passes the checklist and its tests are adversarial, sharply judged, evidence-laddered, and preserve every prior bug.*

# Code Review Bundle (TypeScript)

Read this bundle when reviewing a TypeScript pull request or diff. It bundles the condensed TypeScript practices checklist together with four testing principles that decide whether the change is genuinely defended: tests must try to break the program, must judge results with sharp oracles, must permanently encode every bug ever found, and must use the right evidence layer for the claim. Together these answer the two review questions: "is the code shaped well?" and "would this suite catch regressions?"

---

## TypeScript Practices Checklist

*The compressed form of the TypeScript practices — a single checklist a reviewer or author can run through before opening a PR.*

Use this in code review and as a self-check before opening a PR.

### Foundational
- [ ] **Strict compiler profile applied** — see [[Languages/TypeScript/Practices/Foundational/Strict Compiler Profile]]
- [ ] **Runtime suitability acknowledged** — Node fits this workload, or worst-case latency has been measured. See [[Languages/TypeScript/Practices/Foundational/Runtime Suitability Boundary]]

### Type-driven design
- [ ] **Discriminated unions for state with exhaustive narrowing** — see [[Languages/TypeScript/Practices/Type-Driven Design/Discriminated Unions and Exhaustive Handling]]
- [ ] **Branded types for domain identifiers** — see [[Languages/TypeScript/Practices/Type-Driven Design/Branded Types for Domain Identifiers]]
- [ ] **`satisfies` for config tables, not annotations** — see [[Languages/TypeScript/Practices/Type-Driven Design/Satisfies for Configuration Tables]]

### Error handling
- [ ] **Expected failures return `Result<T, E>`** — not throws. See [[Languages/TypeScript/Practices/Error Handling/Result Pattern for Expected Failures]]
- [ ] **Invariant violations crash the process** — fail-fast at process; fail-safe at product. See [[Languages/TypeScript/Practices/Error Handling/Process-Fatal vs Domain-Recoverable]]
- [ ] **External inputs validated from `unknown`** — schema-parsed at the boundary. See [[Languages/TypeScript/Practices/Error Handling/Runtime Validation at Boundaries]]

### Async and concurrency
- [ ] **No floating promises** — see [[Languages/TypeScript/Practices/Async and Concurrency/Floating Promise Hygiene]]
- [ ] **Event-loop delay and ELU exposed as metrics on critical-path services** — see [[Languages/TypeScript/Practices/Async and Concurrency/Event Loop as Correctness Signal]]
- [ ] **Request-scoped state via `AsyncLocalStorage`** — see [[Languages/TypeScript/Practices/Async and Concurrency/Async Context Propagation]]
- [ ] **Every outbound IO has a deadline; fan-out bounded** — see [[Languages/TypeScript/Practices/Async and Concurrency/Deadline Propagation with AbortSignal]]

### Testing and tooling
- [ ] **Test tooling follows the Evidence Ladder** — see [[Languages/TypeScript/Practices/Testing/TypeScript Test Tooling]] and [[Engineering Philosophy/Principles/Evidence Ladder for Testing]]
- [ ] **Adversarial tests, not just happy paths** — see [[Engineering Philosophy/Principles/Tests Should Make Programs Fail]]
- [ ] **Sharp oracles** — exact equality, invariants, or differential refs. See [[Engineering Philosophy/Principles/Sharp Oracles]]
- [ ] **Typed linting via typescript-eslint flat config** — see [[Languages/TypeScript/Practices/Tooling and Quality/Typed Linting with typescript-eslint]]
- [ ] **Reference templates adopted** — TSConfig and ESLint match (or justifiably diverge from) [[Languages/TypeScript/Practices/Tooling and Quality/Reference TSConfig]] and [[Languages/TypeScript/Practices/Tooling and Quality/Reference ESLint Config]]

### When to consult the long form

If a PR review surfaces friction that doesn't fit the checklist, the corresponding long-form note in [[Languages/TypeScript/Practices/TypeScript Practices MOC]] usually has the deeper guidance.

*Source: [[Languages/TypeScript/Practices/TypeScript Practices Checklist]]*

---

## Tests Should Make Programs Fail

*Write tests that try to break the system; a green suite that never failed is a suite that never tested anything important.*

> The purpose of testing is to make the program fail until the important bugs are found.

This is the offensive posture. Tests are not validation theatre — they are an adversary that wants the program to break, so that the bugs it finds are removed before users find them.

### What this looks like in practice

Create tests that are **intentionally unlike ordinary usage**. Specifically:

- **Combine edge cases in unpleasant ways.** Empty input AND maximum-size flag. First-byte-corrupt AND otherwise valid. Zero AND negative AND NaN.
- **Exercise limits.** Off-by-one is the eternal classic, but also: integer overflow, capacity boundaries, recursion depth, allocation failure.
- **Feed malformed inputs.** Truncated headers, null bytes in the middle of strings, unicode normalization differences, mixed line endings.
- **Test boundary values.** `0`, `1`, `MAX-1`, `MAX`, `MAX+1`. The interesting bugs cluster here.
- **Provoke degenerate states.** A request with one item. A "list" with two elements. An empty body that "shouldn't" be empty.
- **Pass contradictory options.** Conflicting query-string params. Two flags that both claim authority. Either reject coherently or pick deterministically — never silently choose one and discard the other.
- **Use unusual orderings.** Operations the API doesn't forbid but didn't anticipate. Calling `flush` before `write`. Iterating concurrently with mutation.
- **Repeat operations.** Initialize twice. Close twice. Submit the same job back-to-back. Idempotence violations are common and silent.
- **Exercise empty cases.** Most code paths assume nonempty input somewhere. Find the assumption.
- **Exercise maximal cases.** Most code paths assume "reasonable" size somewhere. Find that assumption.
- **Attempt invalid transitions.** Use the type system if you can ([[Languages/TypeScript/Practices/Type-Driven Design/Discriminated Unions and Exhaustive Handling]]); test them at runtime if you can't.

### The slogan

> **Prefer tests that would embarrass a weak design.**

A test passes either because (a) the design is robust enough to handle that case, or (b) the test was too gentle to expose the weakness. Aim for (a) by writing tests that would expose (b). Tests that no one would feel embarrassed by failing are not earning their place in the suite.

### The relationship between this principle and oracles

A test that "exercises" a case but cannot say whether the result is right is a **smoke test**, not an adversarial one. The adversarial value comes from a **sharp verdict**: this output is wrong, this state is invalid, this invariant was broken. See [[Engineering Philosophy/Principles/Sharp Oracles]].

*Source: [[Engineering Philosophy/Principles/Tests Should Make Programs Fail]]*

---

## Sharp Oracles

*An assertion is only as strong as its oracle; replace looks-right checks with exact equality, invariants, or differential references.*

> Tests must have **sharp oracles**. A test without a precise verdict is a smoke test, not a check.

### The principle

For any test:

- **Define concrete expected behavior whenever possible.** Exact outputs over plausibility.
- **Prefer** in this order:
  1. **Exact output equality** (the function returns this specific value)
  2. **Invariant checks** (this property must hold over the result)
  3. **Differential checks** (the result equals a reference implementation's, possibly within a tolerance)
  4. **Reference oracles** (the result matches a precomputed gold answer)
  5. **Precise failure conditions** (this specific error variant must be returned for this input)
- **Avoid** "looks right" as a standard of correctness. If a test passes when the output is plausible but wrong, it does not catch wrong outputs.

### Why "looks right" fails

A test that asserts only that something happened (a function returned, no exception was thrown, the result is non-null) cannot distinguish a working implementation from one that returns garbage. It will pass on the buggy version. The test does not actually defend the property — it defends the existence of an output. This is how silent corruption ships.

### Oracle techniques in TypeScript

#### Exact output

```ts
assert.deepStrictEqual(parse("8080"), Result.ok(asPort(8080)));
```

The simplest and strongest. Use whenever the output is deterministic and small.

#### Invariants (often via fast-check property)

```ts
import * as fc from "fast-check";

fc.assert(fc.property(fc.array(fc.integer()), (input) => {
  const result = sort(input);
  // Property: output is sorted ascending
  for (let i = 1; i < result.length; i++) {
    if (result[i - 1] > result[i]) return false;
  }
  // Property: output is a permutation of input
  return [...result].sort().join(",") === [...input].sort().join(",");
}));
```

#### Precise failure conditions

```ts
const result = parse("");
assert.strictEqual(result.ok, false);
assert.strictEqual(result.error.kind, "empty_input");
```

Don't merely assert "an error happened." Assert *which* error. A bug that returns the wrong error variant is still a bug, and a vague oracle hides it. This is the [[Languages/TypeScript/Practices/Type-Driven Design/Discriminated Unions and Exhaustive Handling]] discipline applied to error testing.

### Anti-patterns

- `assert.ok(result)` — only proves a value was returned; says nothing about correctness.
- `assert.notStrictEqual(output, null)` — proves something was returned; not what.
- "Visual inspection" of test output to decide if it's right — not a test, an opinion.
- A `Result` test that doesn't distinguish error variants.
- A floating-point test with a tolerance picked to make the test pass on this machine.

*Source: [[Engineering Philosophy/Principles/Sharp Oracles]]*

---

## Regression Discipline

*Bugs are engineering knowledge; capture each one as a regression test so the same mistake cannot return silently.*

> Every discovered bug should produce or strengthen a test. Tricky past failures become **permanent members** of the suite. Treat bug history as engineering knowledge.

### The principle

When a bug is found:

1. **Reproduce it as a failing test first.** Before fixing the code. The failing test is the proof you understand the bug; without it, the "fix" is speculative.
2. **Fix the code.** Make the test pass.
3. **Leave the test in the suite forever.** Even — especially — if the bug is "embarrassing" or "obviously won't recur."
4. **Generalize when you can.** If the bug arose from a class of input (empty, max, repeated), add tests for nearby members of that class.

### Why this matters more than it looks

Three reasons.

**Bugs come back.** Code is rewritten. Refactors happen. Optimizations replace working code with cleverer working code, except the cleverer version handles the empty case differently. A regression test catches this **the first time it happens**, not the third time it ships.

**Bugs are sample points of weakness.** Each bug found is evidence of a class of failure the design did not protect against. Recording the bug as a test makes the weakness **legible**.

**Tests preserve hard-won knowledge.** A bug found takes hours, sometimes days, to root-cause. The fix is often one line. The regression test makes the reasoning **executable** — anyone who breaks the invariant in the future learns immediately, without needing to re-discover what the original engineer learned.

### What a good regression test looks like

- **Named for what it defends.** `truncated_payload_returns_typed_error` is better than `bug_42`.
- **Annotated with the failure mode.** A short comment naming what would happen if the regression returned.
- **Targets the property, not the byte-for-byte behavior.** A regression test for "off-by-one on empty input" should test the empty case generally.
- **Not deleted "because it duplicates an existing test."** The duplication is the point.

### Anti-patterns

- **Don't fix the code without a failing test first.**
- **Don't delete regression tests during refactors** because they "no longer match the structure."
- **Don't replace adversarial regression tests with weaker ones.**
- **Don't bury regressions in `it.skip` / `it.todo`** because they're flaky. Flakiness is its own bug.

*Source: [[Engineering Philosophy/Principles/Regression Discipline]]*

---

## Evidence Ladder for Testing

*Each test layer proves a distinct claim. Pick a layer because of the claim it can support, not because the next one was harder to set up.*

> Tests are organized by *what they can prove*, not by where they live in the build. The strongest test suites use multiple layers because each layer answers a question the others cannot.

### The ladder

| Layer | Claim it supports | TypeScript tool |
|---|---|---|
| **Unit** | This function, in isolation, behaves as its contract states for the inputs we listed. | `node:test` / Vitest |
| **Integration** | These components, wired together against real dependencies, do not break each other. | `node:test` + Testcontainers |
| **End-to-end** | A real user-flow, end to end, produces the intended outcome under intended conditions. | Playwright |
| **Property-based** | An invariant holds across a *family* of inputs the test runner generates and shrinks. | fast-check |
| **Fuzzing** | The system survives malformed, hostile, or unexpected inputs without crashing or corrupting state. | Jazzer.js |
| **Mutation** | The test suite *itself* detects deliberate corruption — a meta-claim about the suite, not the code. | Stryker.js |
| **Formal modeling** | An invariant or liveness property holds for *all* reachable states of an abstract model. | TLA+ / Alloy |

### Reading the ladder

The layers don't replace each other. A property-based test does not subsume a unit test, because the unit test's hand-picked example may be the one the property generator never produces. A formal model does not subsume property tests, because the model abstracts away the implementation. A team that "moved up" the ladder by deleting the lower rungs has lost coverage, not improved it.

Strength of claim is not strength of evidence. Formal methods prove the strongest claim, but the claim is about an *abstraction*, not the running code. A formal proof plus zero unit tests is not a stronger position.

Coverage thresholds belong on per-layer claims, not on lines of code. "All public domain functions have at least one property test, and the property test exercises the failure modes the spec describes" tells you something specific. Threshold the claim, not the percentage.

### What to gate at each layer (review checklist)

For a TypeScript PR, each layer the project commits to should be visible in the diff or in the unchanged-but-still-passing suite:

- **Unit:** new functions have at least one example test plus boundary cases.
- **Integration:** changed boundaries (HTTP routes, message-bus consumers, DB queries) have integration tests against real ephemeral dependencies.
- **Property-based:** new parsers, transforms, idempotency-sensitive code paths have a property test, with a fixed seed and stored failure seeds in the regression corpus.
- **Mutation:** if the project uses Stryker, run mutation testing on changed files; check that the score didn't regress.

*Source: [[Engineering Philosophy/Principles/Evidence Ladder for Testing]]*

---

## Related bundles

- [[Bundles/TypeScript/Test-Design]] — the constructive side of the same testing principles, for authoring rather than reviewing
- [[Bundles/TypeScript/Error-Handling]] — review checks for the error-handling slice of a diff
- [[Bundles/TypeScript/Module-Design]] — when a review surfaces structural problems, jump here to redesign
- [[Bundles/Universal/Written-Analysis]] — for the design-doc / analysis side of a non-trivial change
