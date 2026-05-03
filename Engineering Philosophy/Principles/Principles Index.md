---
title: Principles Index
tags: [index, philosophy]
summary: Index of the eight Knuth-style engineering principles in their canonical reading order.
keywords: [reading list, navigation, table of contents, sequence, doctrine, curated entry points]
---

*An ordered list of the eight principles, each linked, so a reader can traverse the philosophy in its intended sequence.*

# Principles Index

The eight Knuth-style engineering principles, in the order they appear in the [[Engineering Philosophy MOC]].

## Reasoning principles (how we think about correctness)

- [[Engineering Philosophy/Principles/Human Understanding First]] — narratable design, exposed invariants
- [[Engineering Philosophy/Principles/Reason About Correctness]] — argue for it; don't hope for it
- [[Engineering Philosophy/Principles/Explicit Not Magical]] — clear control flow, debuggable inspection
- [[Engineering Philosophy/Principles/Reasoning Beside Code]] — produce the surrounding argument

## Testing principles (how we attack the program)

- [[Engineering Philosophy/Principles/Normal Usage Is Not Testing]] — typical use only walks the obvious paths
- [[Engineering Philosophy/Principles/Tests Should Make Programs Fail]] — adversarial posture finds important bugs
- [[Engineering Philosophy/Principles/Sharp Oracles]] — exact, decisive correctness criteria
- [[Engineering Philosophy/Principles/Regression Discipline]] — every bug strengthens the suite, permanently

## Architectural principles (how we shape the system)

- [[Engineering Philosophy/Principles/Architectural Core Principles]] — four load-bearing rules (narrow external boundaries, headless default, deployable-unit modularity, reference path alongside optimization)
- [[Engineering Philosophy/Principles/Headless First]] — library-first; UI is opt-in
- [[Engineering Philosophy/Principles/Reference Implementation as Oracle]] — keep a slow correct twin alongside any optimized path
- [[Engineering Philosophy/Principles/One Binary or Many]] — choose by lifecycle and audience, not aesthetics
- [[Engineering Philosophy/Principles/Capability Slices vs Implementation Switches]] — flags are public capabilities, not internal toggles

## Related

- [[Engineering Philosophy/Engineering Philosophy MOC]]
- [[Engineering Philosophy/Output Format]]
