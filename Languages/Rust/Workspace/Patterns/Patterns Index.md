---
title: Patterns Index
tags: [index, patterns, cargo]
summary: "Index of Cargo workspace patterns: each entry shows how a universal architectural principle is realized in a Cargo workspace."
keywords: [table of contents, navigation, cargo wiring, design rules, architectural decisions, catalog]
---

*A flat list of recurring Cargo workspace patterns; each is a Cargo expression of a universal architectural principle.*

# Patterns Index

Cargo workspace patterns. Each entry pairs a Cargo-specific rule with the universal principle it expresses.

## Architectural patterns (Cargo expressions of universal principles)

- [[Languages/Rust/Workspace/Patterns/Headless First Cargo Wiring]] — separate binary crates or feature-gated UI; expresses [[Engineering Philosophy/Principles/Headless First]]
- [[Languages/Rust/Workspace/Patterns/Reference Path Cargo Wiring]] — core crate vs optimized adapter crate; expresses [[Engineering Philosophy/Principles/Reference Implementation as Oracle]]
- [[Languages/Rust/Workspace/Patterns/Cargo Binary Strategy]] — sibling binary crates vs feature-gated binary; expresses [[Engineering Philosophy/Principles/One Binary or Many]]
- [[Languages/Rust/Workspace/Patterns/Cargo Feature Gating]] — additive features representing real capabilities; expresses [[Engineering Philosophy/Principles/Capability Slices vs Implementation Switches]]

## Cargo / build mechanics

- [[Languages/Rust/Workspace/Patterns/Shared Dependencies]] — what belongs in `[workspace.dependencies]`
- [[Languages/Rust/Workspace/Patterns/Workspace Lints and Profiles]] — root-only configuration

## Related

- [[Languages/Rust/Workspace/Workspace Architecture MOC]]
- [[Languages/Rust/Workspace/Core Principles]]
- [[Engineering Philosophy/Principles/Architectural Core Principles]]
