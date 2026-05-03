---
title: Rust Practices Checklist
tags: [rust, checklist, summary]
summary: Condensed code-review checklist of every Rust practice in the vault, suitable as a PR self-check.
keywords: [pr checklist, review template, pre merge, quality gate, idiom audit, self review]
---

*The compressed form of the Rust practices — a single checklist a reviewer or author can run through before opening a PR.*

# Rust Practices — Short Checklist

The compressed version. Use this in code review and as a self-check before opening a PR.

- [ ] **Model invalid states out of existence** — see [[Make Invalid States Unrepresentable]]
- [ ] **Isolate effects at the edges** — see [[Pure Core Effectful Edges]]
- [ ] **Keep functions small and contracts crisp** — see [[Small Functions Narrow Contracts]]
- [ ] **Use precise error types at boundaries** — see [[Domain Errors at Boundaries]]
- [ ] **Minimize shared mutable state** — see [[Minimize Shared Mutability]]
- [ ] **Test at unit, integration, and doc levels** — see [[Three Levels of Tests]]
- [ ] **Let Clippy nag you before production does** — see [[Clippy as Discipline]]
- [ ] **Favor predictable APIs over clever ones** — see [[Predictable APIs]]

That is the Rust version of building a house with fewer trapdoors.

## When to consult the long form

If a PR review surfaces friction that doesn't fit the checklist, the corresponding long-form note in [[Rust Practices MOC]] usually has the deeper guidance.
