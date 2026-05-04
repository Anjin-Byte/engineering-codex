---
title: Specification Risk vs Type Soundness
tags: [principle, type-system, reasoning, philosophy]
summary: Types prove structural claims; tests, schemas, monitors, and formal models prove behavioral claims — the wrong invariant typed correctly is still wrong.
keywords: [specification, type system limits, semantic correctness, behavioral verification, hoare logic, design intent]
---

*Types prove the shape of your invariant; they do not prove that the invariant is the right one. The wrong invariant typed correctly is still wrong.*

# Specification Risk vs Type Soundness

> **Rule:** the type system can prove that your code matches the invariant you wrote down. It cannot prove that the invariant you wrote down is the one you needed.

There are two distinct risks for any program with a non-trivial domain. **Type-soundness risk** is the risk that the code does not match its declared types — that some path produces a value the type does not allow. **Specification risk** is the risk that the declared types and invariants do not capture what the system is actually supposed to do. Strong type systems annihilate the first; they say almost nothing about the second.

A function `transferFunds(from: AccountId, to: AccountId, amount: Cents): Result<Receipt, TransferError>` may be sound in the strictest type system. It can also silently allow negative amounts, transfers from an account to itself, or transfers in violation of a holding-period rule that nobody encoded. The compiler is satisfied; the spec is not.

## Why this confusion is structural

Strong type discipline rewards thinking *inside the type system*. Once a team is fluent in discriminated unions, branded types, and exhaustive matches, the temptation is to treat type-checking as the proof of correctness. It isn't. The type checker proves that *if* the types describe the truth about the domain, the code respects them. The "if" is doing more work than it looks.

Three places where this gap lives:

1. **The type expresses a weaker invariant than the spec demands.** `amount: Cents` permits `Cents(-100)`. The spec says `amount > 0`. The type does not.
2. **The type expresses the wrong invariant.** A status field `'active' | 'archived'` looks well-modeled until the spec adds `'frozen-but-accruing-interest'`. The compiler will tell you about the missing case after you remember to add it; it cannot tell you the case was missing in the first place.
3. **The type is correct but the implementation lies.** A function returning `Result<T, E>` whose body silently catches and swallows an error returns `Ok` for inputs the spec says should fail. The signature is honest; the body is not.

Each of these failures *passes* a strict type-check.

## Four evidence layers that complement types

The layers below are not a hierarchy of strength — they are a hierarchy of *what they prove*. A program that takes specification correctness seriously uses several of them, on the parts where wrong-spec consequences justify the cost.

1. **Tests** — examples and properties demonstrating that the behavior matches what the spec describes. Property-based tests are particularly powerful because they articulate the spec as a checkable invariant.
2. **Runtime schemas at the boundary** — refuse data the spec doesn't allow before it enters the typed core. A `Cents` validator that rejects negatives turns the spec-layer rule into a runtime guarantee.
3. **Observability and assertions** — invariants checked at runtime in production. A daily reconciliation job that asserts "sum of debits equals sum of credits" is a spec-layer check that no type system will perform.
4. **Formal models** — a TLA+ or Alloy spec proves invariants and liveness for distributed protocols and concurrency-rich logic. See [[Engineering Philosophy/Principles/Formal Modeling for Distributed State]].

Each layer answers a different question: types say "the code matches the contract"; tests say "the contract matches behavior on examples and properties"; runtime checks say "the contract holds for actual production inputs"; formal models say "the contract is internally consistent."

## What this rule changes in practice

- **Stop treating a green type-check as a correctness signal.** It is a non-correctness signal: the absence of one class of bug. The other classes are still on the table.
- **For invariants the system depends on, encode them in two places.** The type system *and* a runtime check at the boundary, *or* the type system and a property test that exercises the invariant. The redundancy is the point.
- **Treat the type as a hypothesis, not a conclusion.** The hypothesis is "this type captures the invariant." Tests, runtime checks, and observability either confirm or refute it.

This is the rule [[Engineering Philosophy/Principles/Reason About Correctness]] makes universal: invariants must be argued for, not merely typed.

## Related

- [[Engineering Philosophy/Principles/Reason About Correctness]]
- [[Engineering Philosophy/Principles/Layered Risk Categories]]
- [[Engineering Philosophy/Principles/Sharp Oracles]]
- [[Engineering Philosophy/Principles/Formal Modeling for Distributed State]]
- [[Engineering Philosophy/Principles/Tests Should Make Programs Fail]]
