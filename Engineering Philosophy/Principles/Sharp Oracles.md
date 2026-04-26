---
title: Sharp Oracles
tags: [philosophy, principle, testing]
summary: Test against an independent reference (closed-form, alternate algorithm, or simpler implementation) so failures point at the bug, not the assertion.
keywords: [test oracles, differential testing, golden tests, exact equality, smoke test, reference implementation]
---

*An assertion is only as strong as its oracle; replace looks-right checks with exact equality, invariants, or differential references.*

# Sharp Oracles

> Tests must have **sharp oracles**. A test without a precise verdict is a smoke test, not a check.

## The principle

For any test:

- **Define concrete expected behavior whenever possible.** Exact outputs over plausibility.
- **Prefer** in this order:
  1. **Exact output equality** (the function returns this specific value)
  2. **Invariant checks** (this property must hold over the result)
  3. **Differential checks** (the result equals a reference implementation's, possibly within a tolerance)
  4. **Reference oracles** (the result matches a precomputed gold answer)
  5. **Precise failure conditions** (this specific error variant must be returned for this input)
- **Avoid** "looks right" as a standard of correctness. If a test passes when the output is plausible but wrong, it does not catch wrong outputs.

## Why "looks right" fails

A test that asserts only that something happened (a function returned, no exception was thrown, the result is non-null) cannot distinguish a working implementation from one that returns garbage. It will pass on the buggy version. The test does not actually defend the property — it defends the existence of an output.

This is how silent corruption ships. The test suite is green; the system is broken.

## Oracle techniques in practice

### Exact output

```rust
assert_eq!(parse("8080").unwrap(), Port::new(8080).unwrap());
```

The simplest and strongest. Use whenever the output is deterministic and small.

### Invariants

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

### Differential check vs. reference

When you have an optimized implementation alongside a slower reference (see [[CPU Reference Path]]):

```rust
let reference = core::reference::process(&input);
let optimized = optimized::process(&input);
assert_close(&reference, &optimized, tolerance);
```

The optimized implementation is being compared to a known-correct reference. The oracle is **another implementation**, not a guess.

### Reference oracle (gold file)

For complex outputs (rendered images, serialized data, parser ASTs), check against a stored reference:

```rust
let actual = serialize(&input);
let expected = include_bytes!("../tests/golden/input_42.bin");
assert_eq!(actual.as_slice(), expected);
```

Pair this with a tooling task that regenerates gold files **only on deliberate command** — never automatically. Auto-regeneration turns the oracle into a rubber stamp.

### Precise failure conditions

```rust
assert!(matches!(parse(""), Err(ParseError::Empty)));
assert!(matches!(parse("00"), Err(ParseError::LeadingZero)));
```

Don't merely assert "an error happened." Assert *which* error. A bug that returns the wrong error variant is still a bug, and a vague oracle hides it.

## Tolerances and floating point

For numeric output, exact equality is often the wrong tool — different summation orders and FMA fusion produce equivalent-but-not-bitwise-identical results. Use tolerances:

- **Absolute tolerance** for small magnitudes near zero.
- **Relative tolerance** for larger values.
- **ULP-based comparison** when both can be misleading.

Document the chosen tolerance in the test and explain *why* that value. A magic ε with no justification is a tolerance that will eventually be loosened to "make the test pass."

## Anti-patterns

- `assert!(result.is_ok())` — only proves an error did not occur; says nothing about correctness.
- `assert!(!output.is_empty())` — proves something was returned; not what.
- "Visual inspection" of test output to decide if it's right — not a test, an opinion.
- A `Result` test that doesn't distinguish error variants.
- A floating-point test with a tolerance picked to make the test pass on this machine.

## Related

- [[Tests Should Make Programs Fail]]
- [[Regression Discipline]]
- [[CPU Reference Path]]
- [[Edge Cases and Properties]]
