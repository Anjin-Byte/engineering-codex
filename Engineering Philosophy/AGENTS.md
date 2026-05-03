---
title: AGENTS — Engineering Philosophy Scope
tags: [meta, agent-entry-point, philosophy]
summary: Universal triggers and full philosophy note index. Always loaded by agents regardless of project language.
---

*The principles in this scope travel to any language and any project. Always relevant; load alongside any per-language scope.*

# AGENTS — Engineering Philosophy Scope

This file routes agents through the universal portion of the vault. The principles here apply regardless of project language. If the agent is also working in a per-language scope, it should additionally load that scope's AGENTS file (e.g. [[Languages/Rust/AGENTS]]).

## When to consult what (universal triggers)

Universal triggers — task types whose guidance does not depend on the project's language.

- **Producing a written analysis or design doc** → [[Engineering Philosophy/Output Format]]; [[Engineering Philosophy/Principles/Reason About Correctness]]; [[Engineering Philosophy/Principles/Reasoning Beside Code]]; [[Engineering Philosophy/Principles/Explicit Not Magical]]; [[Engineering Philosophy/Principles/Human Understanding First]]. Bundle: [[Bundles/Universal/Written-Analysis]].

Most other engineering triggers (review, test design, module design, error handling, API design, refactoring) are language-conditional and live in per-language AGENTS files. Those triggers cross-reference universal notes from this scope as needed.

## Full note index (Engineering Philosophy/)

Every philosophy note with its `summary:` text.

### Engineering Philosophy/

- [[Engineering Philosophy/Engineering Philosophy MOC]] — Map of content for the Knuth-style engineering principles: how to reason, design, and adversarially test software in any language.
- [[Engineering Philosophy/Output Format]] — Four-section deliverable template: correctness model, invariants, design, test strategy — each catches a class of failure the others miss.

### Engineering Philosophy/Principles/

- [[Engineering Philosophy/Principles/Principles Index]] — Index of the engineering principles, grouped by reasoning, testing, and architectural concerns.
- [[Engineering Philosophy/Principles/Reason About Correctness]] — Treat correctness as something to be reasoned about with explicit invariants and proofs of fit, not merely hoped for through tests passing.
- [[Engineering Philosophy/Principles/Tests Should Make Programs Fail]] — The purpose of testing is to make the program fail until the important bugs are found, not to confirm the happy path.
- [[Engineering Philosophy/Principles/Sharp Oracles]] — Test against an independent reference (closed-form, alternate algorithm, or simpler implementation) so failures point at the bug, not the assertion.
- [[Engineering Philosophy/Principles/Regression Discipline]] — Every discovered bug should produce or strengthen a test so past failures become permanent members of the suite.
- [[Engineering Philosophy/Principles/Normal Usage Is Not Testing]] — Running the program in the normal way exercises the happy path; real testing requires deliberate adversarial inputs and edge cases.
- [[Engineering Philosophy/Principles/Reasoning Beside Code]] — When producing code, also produce the surrounding reasoning so readers can follow why it is shaped this way.
- [[Engineering Philosophy/Principles/Explicit Not Magical]] — Prefer clear control flow, stable interfaces, and obvious data movement over cleverness that obscures reasoning.
- [[Engineering Philosophy/Principles/Human Understanding First]] — Structure code so a reader can reconstruct the reasoning behind it; design must be narratable, not just functional.
- [[Engineering Philosophy/Principles/Architectural Core Principles]] — Four load-bearing rules that drive project shape regardless of language: narrow external boundaries, headless default, deployable-unit modularity, reference path alongside optimization.
- [[Engineering Philosophy/Principles/Headless First]] — Build the system to run to completion with no UI or event loop; any interactive front-end is an opt-in adapter.
- [[Engineering Philosophy/Principles/Reference Implementation as Oracle]] — Every algorithm with an optimized implementation must keep a simpler reference implementation alongside it as the correctness oracle.
- [[Engineering Philosophy/Principles/One Binary or Many]] — Decide between one executable with subcommands or modes versus several based on whether the surfaces share lifecycle, dependencies, and audience.
- [[Engineering Philosophy/Principles/Capability Slices vs Implementation Switches]] — Build-time configuration flags should represent real capability slices a downstream user would toggle, not internal implementation switches.

### Bundles/Universal/

- [[Bundles/Universal/Written-Analysis]] — Producing a written analysis or design doc (Output Format; Reason About Correctness; Reasoning Beside Code; Explicit Not Magical; Human Understanding First).

## How to read a note

- Each note opens with an italicized one-line thesis. If you only need the gist, read that.
- Frontmatter `summary:` is the same idea in a different form (suitable for indexing or RAG).
- "Related" sections at the bottom of notes are the canonical next-hops.
