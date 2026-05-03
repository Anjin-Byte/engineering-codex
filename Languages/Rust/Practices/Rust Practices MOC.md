---
title: Rust Practices MOC
tags: [moc, rust, practices]
summary: Map of content for Rust code-level practices that push correctness left into types, construction, ownership, and tests.
keywords: [navigation hub, idiom catalog, table of contents, code level guide, conventions, principles index]
---

*An entry-point map indexing the Rust idioms that move correctness as far left as possible — into types and boundaries, before runtime.*

# Rust Practices — Map of Content

Principles and idioms for writing Rust that favors stability and testability. Entry point for understanding **how to write Rust here** and **why these choices favor long-term reliability**.

## The thesis (one sentence)

> Push correctness as far left as possible — into types, construction, ownership, module boundaries, and tests — so fewer bugs are left to runtime chance.

See [[Push Correctness Left]].

## Foundational

- [[Push Correctness Left]] — the meta-principle behind everything else
- [[Type System as Design Tool]] — types are not just a compiler obstacle
- [[Predictable APIs]] — predictability over cleverness

## Type-Driven Design

- [[Make Invalid States Unrepresentable]]
- [[Newtypes and Domain Types]]
- [[State Transition Types]]
- [[Checked Constructors and Builders]]

## Effect Isolation

- [[Pure Core Effectful Edges]]
- [[Traits as Seams]]
- [[Tests Without Heroics]]

## Error Handling

- [[Result vs Panic]]
- [[Domain Errors at Boundaries]]

## Ownership and Mutation

- [[Obvious Ownership]]
- [[Minimize Shared Mutability]]
- [[Unsafe Quarantine]]

## Functions and Data

- [[Small Functions Narrow Contracts]]
- [[Boring Data Layouts]]

## Testing

- [[Three Levels of Tests]] — unit, integration, doc
- [[Edge Cases and Properties]]

## Tooling and Quality

- [[Clippy as Discipline]]
- [[CI Quality Bar]]
- [[Documentation as Truth]]

## The condensed checklist

See [[Rust Practices Checklist]] for the short-form version.

## Relationship to architecture

These practices are how individual crates are written. The workspace shape that hosts them is documented in [[Workspace Architecture MOC]]. The two should be read together — for example, [[Pure Core Effectful Edges]] is the per-crate expression of architectural rules about effect isolation between crates.
