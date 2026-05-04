---
title: Evidence Ladder for Testing
tags: [principle, testing, philosophy]
summary: Tests form an evidence ladder — unit, integration, end-to-end, property-based, fuzzing, fault injection, mutation, formal modeling — and each layer proves a distinct claim.
keywords: [test pyramid, evidence layers, test strategy, property tests, fuzzing, mutation testing, fault injection, formal methods]
---

*Each test layer proves a distinct claim. Pick a layer because of the claim it can support, not because the next one was harder to set up.*

# Evidence Ladder for Testing

> **Rule:** tests are organized by *what they can prove*, not by where they live in the build. The strongest test suites use multiple layers because each layer answers a question the others cannot.

A useful framing: a test suite is a portfolio of evidence. Each piece of evidence supports a specific claim about the system. A team's testing strategy is choosing which claims to support, at what cost, with which evidence type.

The ladder, weakest claim to strongest:

| Layer | Claim it supports |
|---|---|
| **Unit** | This function, in isolation, behaves as its contract states for the inputs we listed. |
| **Integration** | These components, wired together against real dependencies, do not break each other. |
| **End-to-end** | A real user-flow, end to end, produces the intended outcome under intended conditions. |
| **Property-based** | An invariant holds across a *family* of inputs the test runner generates and shrinks. |
| **Fuzzing** | The system survives malformed, hostile, or unexpected inputs without crashing or corrupting state. |
| **Fault injection / chaos** | The system behaves correctly when its dependencies fail in plausible ways. |
| **Mutation testing** | The test suite *itself* detects deliberate corruption — a meta-claim about the suite, not the code. |
| **Formal modeling** | An invariant or liveness property holds for *all* reachable states of an abstract model. |

Each layer is *independent*: each proves something the others can't, and a gap at one layer is not closed by piling up evidence at another.

## Reading the ladder

Three things to notice.

**First — the layers don't replace each other.** A property-based test does not subsume a unit test, because the unit test's hand-picked example may be the one the property generator never produces. A formal model does not subsume property tests, because the model abstracts away the implementation. A team that "moved up" the ladder by deleting the lower rungs has lost coverage, not improved it.

**Second — strength of the claim is not strength of evidence.** Formal methods prove the strongest claim, but the claim is about an *abstraction*, not the running code. A formal proof plus zero unit tests is not a stronger position than a formal proof plus comprehensive unit tests; it's a different position with the same blind spot (the gap between model and code).

**Third — coverage thresholds belong on per-layer claims, not on lines of code.** "90% line coverage" tells you almost nothing. "All public domain functions have at least one property test, and the property test exercises the failure modes the spec describes" tells you something specific. Threshold the claim, not the percentage.

## Choosing layers by criticality

Not every system needs every rung. The selection is a budget decision:

- **Low-stakes / iterative.** Unit + targeted integration. Skip property-based, fuzzing, mutation unless the cost of being wrong justifies them.
- **High-stakes service.** Unit + integration + property-based on critical paths + fuzzing on parser / boundary surfaces + chaos on dependency failures.
- **Distributed / concurrent / state-rich.** All of the above + a formal model for the parts where wrong-state-machine bugs are catastrophic. See [[Engineering Philosophy/Principles/Formal Modeling for Distributed State]].

The decision criterion is the cost of a missed bug at that layer × the probability the bug exists. A payment processor invests in fuzzing the amount-parsing layer; a static-site generator probably doesn't.

## Layers and oracles compose

A test from any layer is only as strong as its oracle. A property-based test that asserts "the function returns *something*" without checking *what* is just an expensive smoke test. See [[Engineering Philosophy/Principles/Sharp Oracles]]: the oracle decides the verdict; the layer decides what's being checked.

Similarly, [[Engineering Philosophy/Principles/Tests Should Make Programs Fail]] applies at every rung. A unit test that exercises only happy paths is weak unit-layer evidence. A fuzzer that doesn't actually generate hostile inputs is weak fuzz-layer evidence. The adversarial posture is layer-independent.

## Composing with other principles

- [[Engineering Philosophy/Principles/Sharp Oracles]] — the oracle quality multiplier on every rung.
- [[Engineering Philosophy/Principles/Tests Should Make Programs Fail]] — the adversarial posture, applied per layer.
- [[Engineering Philosophy/Principles/Normal Usage Is Not Testing]] — why "we ran it and it worked" is not a layer.
- [[Engineering Philosophy/Principles/Regression Discipline]] — every confirmed bug becomes a permanent test, somewhere on the ladder, ideally at the layer it would have been caught earliest.
- [[Engineering Philosophy/Principles/Layered Risk Categories]] — each test rung typically maps to one or more risk layers.

## Related

- [[Engineering Philosophy/Principles/Sharp Oracles]]
- [[Engineering Philosophy/Principles/Tests Should Make Programs Fail]]
- [[Engineering Philosophy/Principles/Normal Usage Is Not Testing]]
- [[Engineering Philosophy/Principles/Regression Discipline]]
- [[Engineering Philosophy/Principles/Formal Modeling for Distributed State]]
- [[Engineering Philosophy/Principles/Layered Risk Categories]]
