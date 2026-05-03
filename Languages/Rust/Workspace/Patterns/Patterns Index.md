---
title: Patterns Index
tags: [index, patterns]
summary: "Index of cross-cutting workspace patterns: each entry is a rule with its rationale."
keywords: [table of contents, navigation, design rules, architectural decisions, catalog, recurring solutions]
---

*A flat list of recurring architectural patterns the workspace follows, each pairing a hard rule with the reasoning behind it.*

# Patterns Index

Cross-cutting design patterns the workspace follows. Each one is a rule with a rationale.

## Architectural rules

- [[Headless First]] — library-first; binaries and UIs are thin
- [[CPU Reference Path]] — every optimized algorithm has a reference oracle
- [[Binary Strategy]] — one binary vs. several

## Cargo / build

- [[Feature Gating]] — features represent real capability slices
- [[Shared Dependencies]] — what belongs in `[workspace.dependencies]`
- [[Workspace Lints and Profiles]] — root-only configuration

## Related

- [[Workspace Architecture MOC]]
- [[Core Principles]]
