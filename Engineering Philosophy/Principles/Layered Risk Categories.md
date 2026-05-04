---
title: Layered Risk Categories
tags: [principle, architecture, risk, philosophy]
summary: Engineering risk for any non-trivial system splits into six layers — specification, type model, runtime, concurrency, deployment, and supply chain — and a control either mitigates one layer or it does not earn its place.
keywords: [risk surface, control mapping, defense in depth, threat modeling, ssdf, security layers, failure classes]
---

*Six risk layers: specification, type model, runtime, concurrency, deployment, supply chain. Every control is tagged with the layer it mitigates, or it is decoration.*

# Layered Risk Categories

> **Rule:** the risk surface of any non-trivial system can be exhausted by six layers. A standard, audit, or control set that does not say *which layer* it covers is not earning its place.

The six layers, listed once and then defended individually:

1. **Specification** — does the system do the right thing in principle?
2. **Type model** — does the code structurally express the specification it claims to implement?
3. **Runtime** — does the running program behave as the code says, given the runtime's actual semantics (memory, scheduling, resource limits)?
4. **Concurrency / context** — does the program preserve correctness across concurrent operations, async boundaries, and lost or shared context?
5. **Deployment** — does the change reach production safely, with observability and the ability to roll back?
6. **Supply chain** — is the code that runs in production the code we wrote, built from sources we vetted?

A control (a test, a typing rule, a runtime check, a CI gate, a review process) earns its place by being tagged with the layer or layers it mitigates. *Untagged controls are decoration*: they pass audits and catch nothing.

## Why this layering and not some other

The layers are chosen so that no two are reducible to each other. A perfect specification cannot save you from a runtime that ignores promise rejections. A flawless type system cannot prevent a deployment that forgot the migration. A bulletproof CI pipeline cannot vouch for a transitive dependency that was hijacked yesterday. Each layer fails independently, so each must be defended independently.

Other framings (CIA triad, OWASP top 10, NIST SSDF practice groups) are valid for their purposes but cut the surface differently. The six-layer model is engineering-first: it indexes the questions a designer should ask while building, not after.

## What goes wrong at each layer

The point of naming the layers is to give failure modes a home.

- **Specification failures.** The wrong invariant typed correctly is still wrong. A function called `transferFunds` that meets its type signature but allows negative transfers has a spec-layer bug. (See [[Engineering Philosophy/Principles/Specification Risk vs Type Soundness]].)
- **Type-model failures.** The right invariant exists in the spec but the type system does not encode it. `string` instead of `Email`. `number` instead of `Cents`. `boolean` flag instead of a discriminated union. The compiler cannot help.
- **Runtime failures.** The code is correct under its language's *abstract* semantics but breaks under its runtime's *actual* semantics. Event-loop blocking. GC pauses crossing a deadline. Heap exhaustion under load. An async function whose rejection is silently swallowed.
- **Concurrency / context failures.** Lost trace context across an `await`. Two operations racing on a shared cache. A request handler whose authorization context evaporates when work is moved to a worker thread. A workflow whose at-least-once delivery silently mutates twice.
- **Deployment failures.** The new code is fine; the rollout is broken. A migration that ran before the code that needed it. A canary that promoted on a stale metric. A feature flag that was "off" in the wrong environment. (See [[Engineering Philosophy/Principles/Staged Canary Deployment]].)
- **Supply-chain failures.** The artifact in production is not the artifact you reviewed. A typo-squatted dependency. A maintainer compromise that landed a malicious release between `npm install` runs. A build server whose cache poisoned every downstream image.

## The rule of attribution

For every control in the system — a strict compiler flag, a test suite, a CI gate, a runtime monitor, a release-pipeline step — name the layer or layers it covers. A control covering more layers is not necessarily better; a control covering *no* layer is necessarily worse.

Two practical consequences:

1. **Controls cluster, gaps don't move.** Teams over-invest where it's easy (the type model has dozens of strict flags) and under-invest where it's hard (concurrency-layer property tests are rare). A layered audit makes the imbalance visible.
2. **"We have lots of controls" is not the same as "we cover the risk surface."** A standards document that lists 200 rules without attributing each to a layer is uninspectable. The layering is the inspection criterion.

## How this composes with other principles

- [[Engineering Philosophy/Principles/Architectural Core Principles]] addresses the project's *shape*; this note addresses the project's *risks*. They cross-reference: a narrow external-system boundary (Core Principles rule 1) reduces type-model and runtime risk by limiting where untyped data enters.
- [[Engineering Philosophy/Principles/Specification Risk vs Type Soundness]] is the deepest cut of the spec-vs-type-model distinction.
- [[Engineering Philosophy/Principles/Evidence Ladder for Testing]] is the test-strategy layer of this risk model: each test layer proves something against one or more risk layers.
- [[Engineering Philosophy/Principles/Configuration as Code Review]] covers a common deployment-layer hole.
- [[Engineering Philosophy/Principles/Staged Canary Deployment]] and [[Engineering Philosophy/Principles/Error Budgets and Reliability Policy]] cover the deployment layer.

## Related

- [[Engineering Philosophy/Principles/Architectural Core Principles]]
- [[Engineering Philosophy/Principles/Specification Risk vs Type Soundness]]
- [[Engineering Philosophy/Principles/Evidence Ladder for Testing]]
- [[Engineering Philosophy/Principles/Configuration as Code Review]]
- [[Engineering Philosophy/Principles/Staged Canary Deployment]]
