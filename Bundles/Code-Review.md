---
title: Code Review Bundle
tags: [bundle, code-review]
summary: Pre-bundled review checks: Rust practices checklist plus the testing principles that decide whether a diff is genuinely defended.
source_trigger: "Reviewing a PR or diff"
bundles: [Rust Practices Checklist, Tests Should Make Programs Fail, Sharp Oracles, Regression Discipline]
---

*The through-line: a PR is acceptable when its code shape passes the Rust checklist and its tests are adversarial, sharply judged, and preserve every prior bug.*

# Code Review Bundle

Read this bundle when reviewing a pull request or diff. It bundles the condensed Rust practices checklist together with the three testing principles that decide whether the change is genuinely defended: tests must try to break the program, must judge results with sharp oracles, and must permanently encode every bug ever found. Together these answer the two review questions: "is the code shaped well?" and "would this suite catch regressions?"

---

## Rust Practices Checklist

*The compressed form of the Rust practices — a single checklist a reviewer or author can run through before opening a PR.*

### Rust Practices — Short Checklist

The compressed version. Use this in code review and as a self-check before opening a PR.

- [ ] **Model invalid states out of existence** — see [[Make Invalid States Unrepresentable]]
- [ ] **Isolate effects at the edges** — see [[Pure Core Effectful Edges]]
- [ ] **Keep functions small and contracts crisp** — see [[Small Functions Narrow Contracts]]
- [ ] **Use precise error types at boundaries** — see [[Domain Errors at Boundaries]]
- [ ] **Minimize shared mutable state** — see [[Minimize Shared Mutability]]
- [ ] **Test at unit, integration, and doc levels** — see [[Three Levels of Tests]]
- [ ] **Let Clippy nag you before production does** — see [[Clippy as Discipline]]
- [ ] **Favor predictable APIs over clever ones** — see [[Predictable APIs]]

That is the Rust version of building a house with fewer trapdoors.

### When to consult the long form

If a PR review surfaces friction that doesn't fit the checklist, the corresponding long-form note in [[Rust Practices MOC]] usually has the deeper guidance.

*Source: [[Rust Practices/Rust Practices Checklist]]*

---

## Tests Should Make Programs Fail

*Write tests that try to break the system; a green suite that never failed is a suite that never tested anything important.*

> The purpose of testing is to make the program fail until the important bugs are found.

This is the offensive posture. Tests are not validation theatre — they are an adversary that wants the program to break, so that the bugs it finds are removed before users find them.

### What this looks like in practice

Create tests that are **intentionally unlike ordinary usage**. Specifically:

- **Combine edge cases in unpleasant ways.** Empty input AND maximum-size flag. First-byte-corrupt AND otherwise valid. Zero AND negative AND NaN.
- **Exercise limits.** Off-by-one is the eternal classic, but also: integer overflow, capacity boundaries, recursion depth, allocation failure, file size near filesystem limits, buffer size near hardware limits.
- **Feed malformed inputs.** Truncated headers, null bytes in the middle of strings, unicode normalization differences, mixed line endings, BOM in unexpected places.
- **Test boundary values.** `0`, `1`, `MAX-1`, `MAX`, `MAX+1` (where applicable). The interesting bugs cluster here.
- **Provoke degenerate states.** A graph with one node. A "polygon" with two vertices. A texture that is 1×1 or 0×N.
- **Pass contradictory options.** `--quiet --verbose`. `--input file --stdin`. `--output -` with `--format binary`. Either reject coherently or pick deterministically — but never silently choose one and discard the other.
- **Use unusual orderings.** Operations the API doesn't forbid but didn't anticipate. Calling `flush` before `write`. Iterating concurrently with mutation (in languages where that's even a question).
- **Repeat operations.** Initialize twice. Close twice. Submit the same job back-to-back. Idempotence violations are common and silent.
- **Exercise empty cases.** Most code paths assume nonempty input somewhere. Find the assumption.
- **Exercise maximal cases.** Most code paths assume "reasonable" size somewhere. Find that assumption.
- **Attempt invalid transitions.** Use the type system if you can ([[State Transition Types]]); test them at runtime if you can't.

### The slogan

> **Prefer tests that would embarrass a weak design.**

A test passes either because:

(a) the design is robust enough to handle that case, or
(b) the test was too gentle to expose the weakness.

Aim for (a) by writing tests that would expose (b). Tests that no one would feel embarrassed by failing are not earning their place in the suite.

### The relationship between this principle and design

Adversarial tests **inform design**. When a test reveals an awkward corner, the answer is often not just "add a guard." The answer is sometimes:

- This case should be made unrepresentable by the type system.
- This combination of options should not exist as a configuration.
- This API should not expose the operation in this state.

In other words: failed adversarial tests sometimes point toward [[Make Invalid States Unrepresentable]] rather than toward more runtime validation. Use them as design feedback, not just as bug-finding tools.

### The relationship between this principle and oracles

A test that "exercises" a case but cannot say whether the result is right is a **smoke test**, not an adversarial one. The adversarial value comes from a **sharp verdict**: this output is wrong, this state is invalid, this invariant was broken. See [[Sharp Oracles]].

### When the suite has caught up

A useful sign that the suite is doing its job: **adding new adversarial tests stops finding new bugs** for a while. That doesn't mean stop. It means the suite has tightened around the current design. The next round of bugs will arrive with the next round of changes.

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

A test that asserts only that something happened (a function returned, no exception was thrown, the result is non-null) cannot distinguish a working implementation from one that returns garbage. It will pass on the buggy version. The test does not actually defend the property — it defends the existence of an output.

This is how silent corruption ships. The test suite is green; the system is broken.

### Oracle techniques in practice

#### Exact output

```rust
assert_eq!(parse("8080").unwrap(), Port::new(8080).unwrap());
```

The simplest and strongest. Use whenever the output is deterministic and small.

#### Invariants

```rust
let result = compute(&input);
// Property: output is sorted ascending
assert!(result.windows(2).all(|w| w[0] <= w[1]));
// Property: output preserves length
assert_eq!(result.len(), input.len());
// Property: output is a permutation of input
let mut a = input.clone(); a.sort();
let mut b = result.clone(); b.sort();
assert_eq!(a, b);
```

Useful when the exact output depends on implementation choices but properties of it must always hold.

#### Differential check vs. reference

When you have an optimized implementation alongside a slower reference (see [[CPU Reference Path]]):

```rust
let reference = core::reference::process(&input);
let optimized = optimized::process(&input);
assert_close(&reference, &optimized, tolerance);
```

The optimized implementation is being compared to a known-correct reference. The oracle is **another implementation**, not a guess.

#### Reference oracle (gold file)

For complex outputs (rendered images, serialized data, parser ASTs), check against a stored reference:

```rust
let actual = serialize(&input);
let expected = include_bytes!("../tests/golden/input_42.bin");
assert_eq!(actual.as_slice(), expected);
```

Pair this with a tooling task that regenerates gold files **only on deliberate command** — never automatically. Auto-regeneration turns the oracle into a rubber stamp.

#### Precise failure conditions

```rust
assert!(matches!(parse(""), Err(ParseError::Empty)));
assert!(matches!(parse("00"), Err(ParseError::LeadingZero)));
```

Don't merely assert "an error happened." Assert *which* error. A bug that returns the wrong error variant is still a bug, and a vague oracle hides it.

### Tolerances and floating point

For numeric output, exact equality is often the wrong tool — different summation orders and FMA fusion produce equivalent-but-not-bitwise-identical results. Use tolerances:

- **Absolute tolerance** for small magnitudes near zero.
- **Relative tolerance** for larger values.
- **ULP-based comparison** when both can be misleading.

Document the chosen tolerance in the test and explain *why* that value. A magic ε with no justification is a tolerance that will eventually be loosened to "make the test pass."

### Anti-patterns

- `assert!(result.is_ok())` — only proves an error did not occur; says nothing about correctness.
- `assert!(!output.is_empty())` — proves something was returned; not what.
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
4. **Generalize when you can.** If the bug arose from a class of input (e.g., empty inputs, max-sized values, repeated operations), add tests for nearby members of that class. The bug you found is rarely alone.

### Why this matters more than it looks

Three reasons.

#### 1. Bugs come back

Code is rewritten. Refactors happen. Optimizations replace working code with cleverer working code, except the cleverer version handles the empty case differently. A regression test catches this **the first time it happens**, not the third time it ships.

#### 2. Bugs are sample points of weakness

Each bug found is evidence of a class of failure the design did not protect against. Recording the bug as a test makes the weakness **legible**. Future maintainers — and future tools — can see the shape of past failures and reason about whether new changes are likely to violate the same invariants.

#### 3. Tests preserve hard-won knowledge

A bug found takes hours, sometimes days, to root-cause. The fix is often one line. The reasoning behind the fix lives only in the engineer's head, the commit message, and (if you're lucky) a comment.

A regression test makes that reasoning **executable**. Anyone who breaks the invariant in the future learns immediately, without needing to re-discover what the original engineer learned.

### What a good regression test looks like

- **Named for what it defends.** `truncated_header_returns_typed_error` is better than `bug_42` (even if the comment links to issue #42).
- **Annotated with the failure mode.** A short doc comment naming what would happen if the regression returned. This is what makes the test legible to a future reader.
- **Targets the property, not the byte-for-byte behavior.** A regression test for "off-by-one on empty input" should test the empty case generally, not just the specific input that revealed the bug.
- **Not deleted "because it duplicates an existing test."** It might. The duplication is the point — it documents that this case is load-bearing.

### What NOT to do

- **Don't fix the code without a failing test first.** You'll never know if the fix actually fixes the reported bug, vs. some adjacent thing.
- **Don't delete regression tests during refactors** because they "no longer match the structure." If they no longer compile, port them. The behavior they defend is still required.
- **Don't replace adversarial regression tests with weaker ones.** "We replaced the brittle test with a smoke test" usually means "we removed the part that caught the regression."
- **Don't bury regressions in a `#[ignore]` block** because they're flaky. Flakiness is its own bug; investigate root causes. An ignored test is a deleted test with extra paperwork.

### Bug history as a knowledge source

The accumulation of regression tests, taken together, is a **map of the system's historical weaknesses**. It tells future contributors:

- Which types of inputs have caused trouble.
- Which APIs have been misused.
- Which invariants were once silently violated.
- Which platform/environment edge cases bit us.

Reading the regression suite is one of the fastest ways to learn what's actually fragile in a codebase. Curate it like the asset it is.

### Composing with adversarial testing

When [[Tests Should Make Programs Fail|adversarial tests]] reveal a bug, they too produce a regression test — not just the fix, but the originating adversarial case becomes part of the permanent record. Over time, the regression suite becomes a library of attacks the system has been hardened against.

*Source: [[Engineering Philosophy/Principles/Regression Discipline]]*

---

## Related bundles

- [[Bundles/Test-Design]] — the constructive side of the same testing principles, for authoring rather than reviewing
- [[Bundles/Error-Handling]] — review checks for the error-handling slice of a diff
- [[Bundles/Module-Design]] — when a review surfaces structural problems, jump here to redesign
