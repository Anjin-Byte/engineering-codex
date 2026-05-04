---
title: Formal Modeling for Distributed State
tags: [principle, testing, formal-methods, distributed-systems, philosophy]
summary: For workflows whose distributed-state logic is hard to reason about, invest in lightweight formal modeling early — the bugs found in code review or test are the easy ones.
keywords: [tla+, alloy, model checking, distributed correctness, exactly-once, leader election, lease renewal, idempotency, safety, liveness]
---

*Tests find the bugs your imagination produced. Formal modeling finds the bugs your imagination missed. For distributed state, the second category dominates.*

# Formal Modeling for Distributed State

> **Rule:** when a workflow's correctness depends on state transitions across multiple actors over time — leader election, exactly-once delivery, lease renewal, compensation, sequencing, idempotency — invest in a lightweight formal model. Cheaper to find a design bug in TLA+ than in production.

For most code, tests are sufficient evidence. The set of inputs is small enough that a thoughtful engineer can enumerate the dangerous ones, write tests for them, and trust the suite. For distributed state, this breaks down. The "input" is not a function argument; it is an *interleaving of concurrent operations across multiple processes, possibly with failures*. The space of interleavings is combinatorially larger than what tests can cover, and the dangerous ones are not the ones a thoughtful engineer would imagine.

A formal model — TLA+, Alloy, P, or similar — is a small mathematical specification of the protocol that a tool can check exhaustively over a bounded state space. It catches design bugs, not implementation bugs. The implementation can still be wrong. But if the *design* is wrong, the implementation cannot be right.

## When the cost is justified

Formal modeling is not free. Writing a useful TLA+ spec for a non-trivial protocol takes engineering days to weeks. The justification is the cost of being wrong:

- **Blast radius is large.** A consensus algorithm that splits brain corrupts data for every consumer downstream.
- **Concurrent state changes are central, not incidental.** A simple request-response service usually doesn't need a formal model. A workflow engine, a distributed lock service, an exactly-once message bus, a leader-elected scheduler does.
- **Bugs are extremely expensive to reproduce.** A race condition that manifests once a month under specific load patterns is essentially unfixable from production logs. Catching it in a model saves the equivalent of months of incident response.
- **The protocol must hold invariants under failure.** Network partitions, process crashes, message duplication, message reordering. If "what if the leader crashes mid-write?" is a load-bearing question, the model is the place to ask it.

The famous public examples — the AWS S3 / DynamoDB TLA+ work, Microsoft's Azure Cosmos DB modeling, MongoDB's WiredTiger transactional model — all share these properties. They are exactly the systems where the cost of the model paid for itself many times over.

## What a model proves and what it doesn't

Two complementary kinds of property:

- **Safety properties.** "Nothing bad happens": no two leaders simultaneously, no double-debit, no lost message, no stale read after a committed write. Formal modeling is excellent at these.
- **Liveness properties.** "Something good eventually happens": every submitted operation eventually completes, every leader-less period eventually ends. Formal modeling supports these but with caveats (you usually need fairness assumptions to prove liveness).

What it does *not* prove:
- **The implementation matches the model.** The model is an abstraction. A correct model with a buggy implementation produces buggy production behavior. This is why formal modeling sits *alongside* the [[Engineering Philosophy/Principles/Evidence Ladder for Testing]], not on top of it.
- **The model captures the right invariants.** A model that proves "no two leaders" can still have a spec bug if the *real* requirement was "no two leaders accepting writes" and the model conflates leadership with write-acceptance. See [[Engineering Philosophy/Principles/Specification Risk vs Type Soundness]] — the model has the same risk as the type system.
- **The world matches the model's failure assumptions.** A model that assumes message loss but not Byzantine corruption may miss a class of failure that occurs in adversarial conditions.

## How to invest at lower cost

Full TLA+ adoption is a significant investment. There are middle paths:

- **Property-based tests with state-machine generators** (e.g. fast-check `commands`, Hypothesis `RuleBasedStateMachine`). These explore concurrent action sequences against an executable model. Cheaper than TLA+; less exhaustive; closer to the implementation.
- **Lightweight invariant checks at runtime in production.** The system continually verifies its invariants against actual state. A leader-election service that asserts "current term ≥ all observed terms" catches violations without modeling them ahead.
- **Carefully written informal specifications.** Even without a tool, the act of writing the protocol down precisely — every message, every state transition, every timeout — surfaces bugs. The model is the act of writing, not the language.

The point is not "use TLA+ for everything" but "raise the level of rigor proportional to the cost of being wrong." A protocol that handles money or coordinates real-world physical actuation deserves more rigor than a protocol that displays a notification.

## When the answer is still no

Some teams shouldn't adopt formal modeling, even when their system would benefit:

- **Skill gap.** TLA+ has a learning curve. If no one on the team has used it, the first attempt will be worse than no attempt — the model will be wrong and trusted. Better path: hire a consultant for the first model, learn from it, then build internal capacity.
- **The protocol is changing weekly.** A model only pays off when the protocol has stabilized enough that re-modeling on every change isn't a tax. Model the protocol after design freeze, not before.
- **Smaller blast radius than maintenance cost.** Don't model a system whose worst-case incident is smaller than the engineering cost of maintaining the model.

These are real constraints. The principle is "use formal modeling when its expected value is positive" — not "use formal modeling whenever it's technically possible."

## Composing with other principles

- [[Engineering Philosophy/Principles/Evidence Ladder for Testing]] places formal modeling at the top rung — strongest claim, narrowest scope.
- [[Engineering Philosophy/Principles/Specification Risk vs Type Soundness]] applies to models too: the model's correctness is the model's *spec's* correctness.
- [[Engineering Philosophy/Principles/Reason About Correctness]] is the universal habit; formal modeling is the deepest tool for it.
- [[Engineering Philosophy/Principles/Layered Risk Categories]] places this at the concurrency / context layer.

## Related

- [[Engineering Philosophy/Principles/Evidence Ladder for Testing]]
- [[Engineering Philosophy/Principles/Specification Risk vs Type Soundness]]
- [[Engineering Philosophy/Principles/Reason About Correctness]]
- [[Engineering Philosophy/Principles/Layered Risk Categories]]
- [[Engineering Philosophy/Principles/Sharp Oracles]]
