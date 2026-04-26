---
title: Test Design Bundle
tags: [bundle, testing]
summary: Bundle for authoring tests: three test levels, edge and property tactics, design-for-testability, sharp oracles, and the adversarial posture.
source_trigger: "Writing tests"
bundles: [Three Levels of Tests, Edge Cases and Properties, Tests Without Heroics, Sharp Oracles, Normal Usage Is Not Testing]
---

*The through-line: real tests are adversarial, sharply judged, run cheaply because the design supports it, and span all three Rust test levels.*

# Test Design Bundle

Read this bundle when writing tests. It bundles the structural rules (which level a test belongs at, what tactics apply to edges and properties, how to keep tests cheap), with the philosophical posture that distinguishes real testing from automated normal use, and with the oracle discipline that makes a test verdict mean something.

---

## Three Levels of Tests

*Three test levels exist for three different failure classes; strong projects use all of them rather than picking one.*

Rust distinguishes **unit tests**, **integration tests**, and **doc tests**. Strong projects use all three because they catch different classes of mistake.

### The split

#### Unit tests
- Live alongside the code they test, in `#[cfg(test)] mod tests { use super::*; }`.
- Have access to private items in the same module.
- Target small pure functions, type invariants, edge cases.
- Should be fast and independent.

#### Integration tests
- Live in the crate's `tests/` directory.
- See only the **public API** of the crate — exactly what external users see.
- Target end-to-end flows through the public surface.
- Catch API drift and accidental breakage of public behavior.

#### Doc tests
- Live inside `///` doc comments as fenced ` ```rust ... ``` ` blocks.
- Compiled and run by `cargo test` automatically.
- Keep documentation **honest** — examples can't lie if they're executed.

### Why all three

| Test kind | Catches |
|---|---|
| Unit | Logic bugs, off-by-one, invariant violations, branch coverage |
| Integration | API regressions, wiring mistakes, real-world flow bugs |
| Doc | Documentation drifting from reality |

Each level catches things the others can't. Skipping a level means a class of bug has nowhere to be caught.

### Where to put what

A pure function on a domain type? **Unit test** it.

A scenario like "load this config file, run the operation, write this output"? **Integration test** in `tests/`.

A worked example in a public type's docs? **Doc test**, so it stays compiling.

### Doc tests in practice

```rust
/// Constructs a [`Port`] from a non-zero `u16`.
///
/// # Examples
///
/// ```
/// use mycrate::Port;
/// let p = Port::new(8080).expect("nonzero");
/// assert_eq!(p.get(), 8080);
/// ```
///
/// Returns `None` for the zero value:
///
/// ```
/// use mycrate::Port;
/// assert!(Port::new(0).is_none());
/// ```
pub fn new(value: u16) -> Option<Self> { /* ... */ }
```

Both blocks compile and run as part of `cargo test`. If the API changes and the example breaks, CI catches it before the docs become a liar.

### Doc tests that intentionally don't compile

Use code-block annotations:

- ` ```ignore ` — present in docs, not compiled.
- ` ```no_run ` — compiled but not executed (for examples that need a real device, network, etc.).
- ` ```compile_fail ` — must fail to compile (useful for documenting type-system safety).

Use these sparingly — the whole point is that the example matches reality.

### How this layers across a workspace

- **Pure core crates**: lots of unit tests + doc tests. Easy because the crate is pure.
- **Adapter crates** (those that wrap external effects): unit tests for layout/planning logic; integration tests for end-to-end flows (some `no_run` doc examples for snippets that require a real environment).
- **Binary crates**: integration tests in `tests/` exercising the binary's argument parsing and orchestration.
- **Cross-crate** integration tests: at the workspace root in `/tests/` for flows that span crates.

*Source: [[Rust Practices/Testing/Three Levels of Tests]]*

---

## Edge Cases and Properties

*Happy-path tests are necessary but rarely sufficient; pursue edges and properties where bugs actually live.*

Test the places where humans improvise badly. Happy-path tests are necessary but rarely sufficient.

### What to actually target

The high-value test categories for stability:

- **Boundary values** — zero, one, max, max+1, empty, full.
- **Empty inputs** — empty slice, empty string, empty iterator. Many bugs live here.
- **Malformed inputs** — wrong types, wrong order, garbage bytes, partial reads.
- **Repeated application / idempotence** — does `f(f(x)) == f(x)` when it should?
- **Ordering assumptions** — does the function depend on input order? Does it preserve order?
- **State transition mistakes** — calling methods in the wrong sequence; double-init, double-close.
- **Concurrency boundaries** — two operations racing; cancellation mid-flight (where applicable).

### A useful question

For every public function, ask:

> **What inputs would an annoyed future version of me accidentally pass here?**

Then write *that* test now, while current-you is still cooperative. The annoyed-future-you scenarios are exactly where bugs land in production.

### Specific Rust patterns

#### Boundary tests for integer types

```rust
#[test]
fn handles_zero() { assert!(Port::new(0).is_none()); }
#[test]
fn handles_one() { assert!(Port::new(1).is_some()); }
#[test]
fn handles_max() { assert!(Port::new(u16::MAX).is_some()); }
```

#### Empty-collection tests

```rust
#[test]
fn empty_input_returns_empty_output() {
    assert_eq!(transform(&[]), vec![]);
}
```

#### Malformed-input tests

```rust
#[test]
fn rejects_truncated_header() {
    let bad = &[0x00, 0x01]; // valid headers are 8 bytes
    assert!(matches!(parse(bad), Err(ParseError::TruncatedHeader)));
}
```

#### Idempotence

```rust
#[test]
fn validate_is_idempotent() {
    let v1 = validate(input.clone()).unwrap();
    let v2 = validate(input).unwrap();
    assert_eq!(v1, v2);
}
```

### Property-based testing

For algorithms with rich input spaces, consider `proptest` or `quickcheck`:

```rust
proptest! {
    #[test]
    fn round_trip_serialization(input in any::<Config>()) {
        let bytes = serialize(&input).unwrap();
        let back = deserialize(&bytes).unwrap();
        prop_assert_eq!(input, back);
    }
}
```

Properties worth looking for:
- Optimized result ≈ reference result within tolerance (see [[CPU Reference Path]])
- Serialization round-trips
- Idempotent transformations
- Invariants preserved by every operation on a type

### What NOT to over-test

- Trivial getters/setters — testing them tests the language, not your code.
- Implementation details — tests that break on every refactor are noise.
- Behavior already enforced by the type system — if `Port` can't be 0, no test needs to check `Port != 0`.

The goal is tests that catch real defects without rotting on every change. Test the **behavior**, not the **implementation**.

*Source: [[Rust Practices/Testing/Edge Cases and Properties]]*

---

## Tests Without Heroics

*When a test needs heroics to run, fix the design, not the test; testability follows from where effects and dependencies live.*

> **Heuristic:** if testing requires heroics, the *design* is fighting testability — not the test framework.

### Symptoms of a fighting design

- Temp directories created and torn down for every case.
- Thread sleeps to "wait for" something to happen.
- Network mocks or local server processes spun up per test.
- Giant fixture files duplicated across cases.
- Half the crate marked `pub` purely so tests can reach in.
- `#[cfg(test)]` shims in production modules to "make this testable."

When you see these, the cause is usually upstream: a function that mixes effects with logic, or a type that hides behavior the test needs to observe.

### Healthier alternatives

- **Dependency injection through traits or function parameters.** Pass the clock, filesystem, or external client in. See [[Traits as Seams]].
- **Constructors that accept collaborators.** A type that takes its dependencies in `new()` is testable; one that constructs them internally is not.
- **Explicit config objects.** Beats long parameter lists, and tests can build a known-good config once.
- **Pure helpers factored out of orchestrators.** The orchestrator stays effectful and is integration-tested; the helpers become unit-testable.

### The "moved-out-of-main" pattern

The Rust Book's classic CLI example demonstrates this: code stuck in `main` is hard to test, so move it into a library with explicit inputs. The same move applies recursively — code stuck in any effectful function is hard to test, so move the pure part out.

### Avoid `pub(crate)` as a workaround

Marking things `pub(crate)` "so tests can see them" is a smell. The right moves are:

1. Test through the public API instead. If that's hard, the public API needs adjusting.
2. Move the pure logic to a place it can be tested at the unit level without exposing internals.
3. Use `#[cfg(test)] mod tests { use super::*; }` for tests that legitimately need private access — that's idiomatic and doesn't widen the public surface.

### When heroics are unavoidable

Some integration tests genuinely need a real filesystem, a real device, or a real subprocess. That's fine — but those should be **few**, **clearly labeled** (consider a separate `tests/integration_*.rs` file or a feature gate), and not the only way the system is tested. Most tests should still be cheap.

*Source: [[Rust Practices/Effect Isolation/Tests Without Heroics]]*

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

## Normal Usage Is Not Testing

*Normal usage is not testing — it only proves the path you already took works; real tests require chosen adversarial inputs.*

> Do not confuse normal usage with real testing.

### The claim

- **Typical user behavior exercises only the obvious paths.** Most users do not stress-test; they accomplish their goal and leave.
- **The hardest and least-traveled code paths are the ones most likely to hide defects.** They have not been worn smooth by use.
- Therefore, **tests must reach the rare, awkward, pathological, composite, and adversarial cases** — because nothing else will.

### Why this matters

A program that "works in normal use" has been validated against approximately the easiest 10–20% of the input space. The remaining 80–90% — empty inputs, max-sized inputs, simultaneous edge conditions, malformed-yet-superficially-valid data, contradictory option combinations, repeated invalid transitions — has been **observed not at all**.

If the only tests in the suite reflect the same shape as normal use, the suite is just **automated normal use**. It catches regressions in the well-traveled paths. It does not catch lurking defects.

The well-traveled paths are not where bugs hide. They are where bugs have been *found and fixed*. The unvisited paths are where bugs **still are**.

### Where the lurking defects live

Categories of code that historically conceal the most bugs:

- **Error paths.** They run rarely, are tested less, and often have their own subtle assumptions.
- **Resource cleanup paths.** Drop, close, abort, cancel. Frequently exercised in tests only on the happy path.
- **Empty cases.** Empty slice, zero-length string, empty iterator, no input at all. Often a separate code path that nobody walked through carefully.
- **Maximum cases.** Max-int, max-capacity, max-recursion-depth. Frequently fail in subtle ways (overflow, allocation failure, stack growth).
- **Concurrent operations.** Two operations interleaved in an order no one anticipated.
- **Repeated operations.** Calling `init` twice, `close` twice, `flush` after `close`.
- **State-machine edge transitions.** Going from a rare state to another rare state.
- **Cross-feature interactions.** Two features that work alone but interact badly.

Each of these is a place ordinary usage will not visit, and where bugs accumulate quietly.

### Implication for the testing strategy

Normal-shaped tests (the "ordinary behavioral" class in [[Output Format]]) are a baseline, not the goal. The goal is to **also** systematically attack the categories above. See:

- [[Tests Should Make Programs Fail]] — the offensive posture
- [[Sharp Oracles]] — the precision of the verdict
- [[Edge Cases and Properties]] — concrete tactical patterns
- [[Tests Without Heroics]] — keep the design testable so the attacks can be cheap

### Honest framing

You will not eliminate all bugs this way. But you will eliminate the **important** bugs that hide in the neglected half of the system — the ones that, left alone, become production incidents. The cost is upfront test design effort. The payoff is the bugs your users do not file.

*Source: [[Engineering Philosophy/Principles/Normal Usage Is Not Testing]]*

---

## Related bundles

- [[Bundles/Code-Review]] — the same testing principles applied to reviewing rather than authoring
- [[Bundles/Module-Design]] — when tests need heroics, redesign the module using this bundle
- [[Bundles/Written-Analysis]] — for the test-strategy section of a design doc
