---
title: Principles Index
tags: [index, philosophy]
summary: Index of the engineering principles, grouped by reasoning, testing, architectural, and delivery / reliability concerns.
keywords: [reading list, navigation, table of contents, sequence, doctrine, curated entry points]
---

*A grouped index of the principles, so a reader can traverse the philosophy by concern.*

# Principles Index

Engineering principles grouped by the kind of decision they govern. Each group is independent — a reader concerned with one (e.g. how to attack programs with adversarial tests) can read just that section. The [[Engineering Philosophy/Engineering Philosophy MOC]] is the human entry point.

## Reasoning principles (how we think about correctness)

- [[Engineering Philosophy/Principles/Human Understanding First]] — narratable design, exposed invariants
- [[Engineering Philosophy/Principles/Reason About Correctness]] — argue for it; don't hope for it
- [[Engineering Philosophy/Principles/Explicit Not Magical]] — clear control flow, debuggable inspection
- [[Engineering Philosophy/Principles/Reasoning Beside Code]] — produce the surrounding argument
- [[Engineering Philosophy/Principles/Specification Risk vs Type Soundness]] — types prove structure, not behavior

## Testing principles (how we attack the program)

- [[Engineering Philosophy/Principles/Normal Usage Is Not Testing]] — typical use only walks the obvious paths
- [[Engineering Philosophy/Principles/Tests Should Make Programs Fail]] — adversarial posture finds important bugs
- [[Engineering Philosophy/Principles/Sharp Oracles]] — exact, decisive correctness criteria
- [[Engineering Philosophy/Principles/Regression Discipline]] — every bug strengthens the suite, permanently
- [[Engineering Philosophy/Principles/Evidence Ladder for Testing]] — each test layer proves a distinct claim
- [[Engineering Philosophy/Principles/Formal Modeling for Distributed State]] — TLA+-grade rigor for protocols whose bugs cost more than the model

## Architectural principles (how we shape the system)

- [[Engineering Philosophy/Principles/Architectural Core Principles]] — four load-bearing rules (narrow external boundaries, headless default, deployable-unit modularity, reference path alongside optimization)
- [[Engineering Philosophy/Principles/Headless First]] — library-first; UI is opt-in
- [[Engineering Philosophy/Principles/Reference Implementation as Oracle]] — keep a slow correct twin alongside any optimized path
- [[Engineering Philosophy/Principles/One Binary or Many]] — choose by lifecycle and audience, not aesthetics
- [[Engineering Philosophy/Principles/Capability Slices vs Implementation Switches]] — flags are public capabilities, not internal toggles
- [[Engineering Philosophy/Principles/Layered Risk Categories]] — six risk layers; every control is tagged with the layer it covers

## Delivery and reliability principles (how we ship and operate)

- [[Engineering Philosophy/Principles/Error Budgets and Reliability Policy]] — reliability is a budget enforced by policy, not negotiation
- [[Engineering Philosophy/Principles/Staged Canary Deployment]] — partial rollout against a control, gated on SLO health
- [[Engineering Philosophy/Principles/Configuration as Code Review]] — config changes go through the same pipeline as code

## Related

- [[Engineering Philosophy/Engineering Philosophy MOC]]
- [[Engineering Philosophy/Output Format]]
