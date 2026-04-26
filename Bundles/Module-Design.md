---
title: Module Design Bundle
tags: [bundle, architecture]
summary: Bundle for designing a new module or crate: workspace shape, core principles, layout, and the type-system mindset that pushes correctness left.
source_trigger: "Designing a new module or crate"
bundles: [Workspace Architecture MOC, Core Principles, Workspace Layout, Type System as Design Tool, Push Correctness Left]
---

*The through-line: a new module or crate earns its boundary by reflecting a responsibility that evolves at its own speed, with types doing the load-bearing correctness work.*

# Module Design Bundle

Read this bundle when designing a new module or crate. It bundles the workspace-architecture rules (when a crate is justified, what shape the tree takes, what each kind of crate is for) with the foundational Rust mindset (types are a design tool; correctness moves leftward through types, constructors, and module boundaries). The architecture chooses the seams; the foundational practices decide what each seam guarantees.

---

## Workspace Architecture MOC

*A non-trivial Rust application is a set of crates whose seams track responsibility, not aesthetics; this MOC indexes the rules and patterns that enforce that.*

Entry point for understanding **why** a Rust workspace is shaped a particular way, and **where** to look for any specific architectural decision.

### The thesis

> A non-trivial Rust application should be modeled as a **set of crates whose boundaries reflect responsibilities that evolve at different speeds** — fast-moving CLI / orchestration code, slow-moving domain logic, narrowly-scoped adapters for external systems. The workspace shape is what keeps these concerns from fusing.

### Foundational reading order

1. [[Core Principles]] — the load-bearing rules
2. [[Workspace Layout]] — directory structure
3. [[Cargo Workspace Configuration]] — root manifest
4. [[Patterns Index]] — recurring design patterns

### Patterns

- [[Headless First]] — library-first; binaries are thin
- [[CPU Reference Path]] — keep a reference implementation alongside any optimized one
- [[Binary Strategy]] — one binary vs. several
- [[Feature Gating]] — when to use Cargo features
- [[Shared Dependencies]] — what belongs in `workspace.dependencies`
- [[Workspace Lints and Profiles]] — root-only configuration

### The condensed rules

1. Core logic must survive without its optional adapters.
2. External-system adapters live behind narrow crate boundaries.
3. Optional UI / interactive concerns are isolated and opt-in.
4. The workspace root owns versions, profiles, and lints.
5. Default workspace commands target the primary binary, not every crate.
6. Crates reflect responsibilities, not aesthetics.
7. For any optimized implementation, keep a reference implementation as oracle.

*Source: [[Architecture/Workspace Architecture MOC]]*

---

## Core Principles

*Four principles govern any decision about workspace shape; treat any future change that violates one of them as a smell to revisit.*

Four principles drive workspace decisions. If a future change violates one of these, treat that as a smell and revisit the design.

### 1. Keep external-system boundaries narrow

Core algorithms should not know about the types of any external system (a device API, a windowing library, a heavy third-party SDK) unless absolutely necessary.

**Bad shape:** application logic directly manipulates external-system types, configuration, window state, CLI args, and file I/O all in one crate.

**Better shape:** split by responsibility:

- **core crate(s)** — domain types, algorithms, deterministic transformations
- **adapter crate(s)** — one per external system being wrapped, with its own narrow public API
- **cli crate** — argument parsing, file I/O, logging, exit codes
- **optional UI crate** — interactive front-end, isolated from core

That way, most logic can be tested without the external systems present at all.

See: [[Headless First]]

### 2. Treat headless as the default, interactive UI as an add-on

A library-first design lets the same code be embedded in CLIs, services, batch jobs, CI pipelines, and tests. Interactive UI is layered on top — never required for the core to work. See: [[Headless First]].

### 3. Make the workspace reflect deployable units

Rule of thumb: **if something can be built, tested, or shipped independently, it deserves its own crate.**

Do not split crates just to look sophisticated. A workspace with five meaningful crates is better than one with twelve decorative gourds.

See: [[Workspace Layout]]

### 4. Always keep a reference implementation alongside any optimized path

For any algorithm with an optimized implementation (SIMD, GPU, FFI, hand-rolled), maintain a reference implementation in pure, deterministic code.

Reasons:
- correctness oracle
- testability in CI without the optimized environment
- easier debugging
- deterministic comparisons

Workspace consequence:
- **core**: canonical, deterministic behavior
- **optimized adapter**: accelerated implementation

This saves you from the classic optimization debugging experience: *"it is very fast and very wrong."*

See: [[CPU Reference Path]]

*Source: [[Architecture/Core Principles]]*

---

## Workspace Layout

*Lay the workspace out so the file tree itself shows where responsibilities live and which crates depend on which.*

A recommended directory shape for a non-trivial Rust workspace.

### Generic full layout

```
project_root/
  Cargo.toml              # virtual workspace manifest
  .cargo/
    config.toml
  crates/
    core/                 # pure domain logic (no external effects)
      Cargo.toml
      src/
    adapter-x/            # adapter wrapping an external system / library
      Cargo.toml
      src/
    cli/                  # argument parsing, file I/O, orchestration
      Cargo.toml
      src/
    ui/                   # OPTIONAL: interactive front-end
      Cargo.toml
      src/
  tests/                  # cross-crate integration tests
  reference/              # primary-source reference documents
  tools/                  # build / automation tasks
    automation/
      Cargo.toml
      src/
```

The names are placeholders — pick names that reflect responsibility in your domain. The shape matters more than the labels.

### Why this shape

- **Virtual workspace root** — see [[Cargo Workspace Configuration]] for why and how.
- **`crates/*` glob** — every member sits in one place; trivial to add or move.
- **Adapter crates per external system** — each external boundary (a device API, a network protocol, a heavy third-party library) gets its own crate so its types do not leak into core logic.
- **Build automation as a sibling to `crates/`** — it is build infrastructure, not a library, but it is a Cargo workspace member and benefits from shared lints/deps.
- **`tests/` at root** — for cross-crate integration tests; per-crate unit tests live in their own `tests/` or inline `#[cfg(test)]`.
- **`reference/` at root** — primary-source PDFs, specifications, datasets. Not a crate.

### Phasing the workspace

A useful pattern: start with the pure core, add adapters and binaries as their need is justified.

#### Phase 1 — pure core

```
project_root
├── crates/core
└── tools/automation
```

Establish the domain model, types, and reference algorithms with no external dependencies. The core can be tested anywhere, and serves as the oracle for any future optimized implementation. See [[Push Correctness Left]] and [[CPU Reference Path]].

#### Phase 2 — add adapters and a binary

```
project_root
├── crates/core
├── crates/adapter-x        ← optimized / external-system path
├── crates/cli              ← orchestration, I/O
└── tools/automation
```

At this point the system is shippable as a binary.

#### Phase 3 — optional interactive front-end

Add a UI crate only when interactive use is actually needed. See [[Headless First]].

*Source: [[Architecture/Workspace Layout]]*

---

## Type System as Design Tool

*Types are a design surface; use them to express what the program is allowed to do, not just to placate the compiler.*

The deepest mindset shift in writing strong Rust: **the type system is a design tool, not a compiler obstacle.**

### The reframe

When the borrow checker or type checker pushes back on a design, the instinct from other languages is to **placate** the compiler — `clone`, `Rc`, `RefCell`, `Box<dyn Any>`, `unsafe`, whatever makes the red squiggle go away.

In Rust, that pushback is usually **information**. The compiler is saying "the shape you've drawn doesn't actually fit the constraints you've claimed." The right move is often not to placate but to **redesign**.

### Diagnostic questions

When a Rust design feels like a fight, run through these:

- Should this be **two types** instead of one? (e.g. `Draft` vs `Validated`)
- Should this state be an **enum** instead of two booleans?
- Should this field be **private** with a checked constructor in front?
- Should this dependency be **passed in** rather than looked up globally?
- Should this owned value really be **borrowed**, or this borrow really be **owned**?
- Should this trait object be a **generic**, or vice versa?

Most "Rust is hard" moments resolve when the answer to one of these flips.

### What this looks like in practice

A signal that the type system is doing useful design work:

- The compiler refuses an operation, you change the *types* (not the operation), and the operation becomes unnecessary.
- A bug class disappears from your test plan because it can no longer be expressed.
- Refactors propagate through the codebase by following compile errors instead of running every test.

A signal you're fighting the type system instead of using it:

- Lots of `.clone()` calls that exist only to dodge a borrow.
- Frequent `Rc<RefCell<_>>` in code that isn't actually shared between owners.
- `unsafe` blocks added to express something the type system "doesn't get."
- Long chains of generic parameters whose only job is to stay generic.

*Source: [[Rust Practices/Foundational/Type System as Design Tool]]*

---

## Push Correctness Left

*Move correctness leftward — into types, constructors, and module shape — until what remains for runtime testing is genuinely small.*

The meta-principle behind every other Rust practice in this codex.

> **Push correctness as far left as possible** — into types, construction, ownership, module boundaries, and tests — so fewer bugs are left to runtime chance.

### "Left" means earlier in the bug's lifecycle

Imagine the timeline of how a defect becomes user-visible:

1. **Type system** rejects it (best — bug never compiles)
2. **Constructor / validation** rejects it at object birth
3. **Module boundary** rejects it at API entry
4. **Unit test** catches it before merge
5. **Integration test** catches it before release
6. **Production** catches it (worst — bug becomes incident)

The further left you push the catch, the cheaper, faster, and louder the failure. Rust gives unusually good tools at the leftmost three slots — the type system, construction patterns, and module privacy — and we should use them.

### Why this principle leads everything

Every other practice in [[Rust Practices MOC]] is a corollary:

- [[Make Invalid States Unrepresentable]] — left-shift to slot 1
- [[Checked Constructors and Builders]] — left-shift to slot 2
- [[Domain Errors at Boundaries]] — left-shift to slot 3
- [[Three Levels of Tests]] — slots 4 and 5
- [[Clippy as Discipline]] — automated nagging that pulls slot 6 issues back to slot 4

### When this principle is uncomfortable

Pushing left often costs *more* code up front: a newtype instead of a `u32`, a builder instead of a struct literal, an enum instead of two booleans. That cost is real but local. The payoff is global — fewer surprises across every caller, every refactor, every future contributor.

The wrong reaction to "this newtype is verbose" is to delete the newtype. The right reaction is to ask whether the *use sites* could be cleaner.

### What this principle is **not**

It is not "ban runtime checks." Some invariants can only be checked at runtime (network input, file contents, untrusted external data). For those, push the check **once**, at the boundary, and produce a validated type — then never re-check downstream. See [[State Transition Types]].

*Source: [[Rust Practices/Foundational/Push Correctness Left]]*

---

## Related bundles

- [[Bundles/API-Design]] — what the new module's public surface should look like
- [[Bundles/Error-Handling]] — what errors the new module's boundary returns
- [[Bundles/Test-Design]] — how the new module gets its test coverage without heroics
- [[Bundles/Written-Analysis]] — the design doc that should accompany a non-trivial new module
