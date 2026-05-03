---
title: Written Analysis Bundle
tags: [bundle, writing, design-doc]
summary: "Bundle for producing a written analysis or design doc: the four-section format and the principles that make reasoning legible, explicit, and human-first."
source_trigger: "Producing a written analysis or design doc"
bundles: [Output Format, Reason About Correctness, Reasoning Beside Code, Explicit Not Magical, Human Understanding First]
---

*The through-line: a written analysis externalizes the correctness argument so a future reader can audit it, in a structure that puts human understanding before machine convenience.*

# Written Analysis Bundle

Read this bundle when producing a written analysis or design doc. It bundles the four-section deliverable template (correctness model, invariants, design, test strategy) together with the four philosophical principles that justify why writing is part of engineering: correctness must be reasoned about, reasoning belongs beside the code, explicit beats magical, and human understanding comes first.

---

## Output Format

*Structure non-trivial design or implementation work in four sections so each layer of failure has a dedicated place to surface.*

For non-trivial design or implementation work, structure the deliverable in **four sections**, in this order. The format exists because each section catches a class of failure the others cannot.

### 1. Correctness model

State, in plain language, **what the system must do for it to be considered correct**. Not what it must compute — what property must hold. Examples:

- "For any valid input, the solver produces an output graph that is connected."
- "For any accelerated output, there exists a reference computation that produces the same answer within a documented tolerance."
- "Calling `flush()` on an open writer makes all prior `write()` calls observable to subsequent readers, even after process death."

The correctness model is the single contract the rest of the work is justified against. If you can't write it in one paragraph, the design is not yet understood.

### 2. Invariants and assumptions

List the **load-bearing claims** the design relies on. Two flavors:

- **Invariants the system maintains** — properties true at every observation point (e.g., "the index map is always a bijection", "no `Port` value is ever zero").
- **Assumptions about the environment** — properties relied on but not enforced by the code (e.g., "the runtime supports the required API", "the input file is no larger than 4 GiB").

Each invariant should name **where it is established** (constructor, validation step, type definition) and **where it is relied on** (callers, downstream functions).

This section is doing the work of [[Reason About Correctness]] — externalizing the argument so a reader can audit it.

### 3. Design and implementation

The actual code, plus the prose that lets a reader follow it. Conventions:

- **Why before how.** Each major component opens with one to three sentences of intent.
- **Narrate the data movement.** A reader should be able to trace a value from input to output without running the code.
- **Surface the type-system arguments.** When a type makes an invalid state impossible, say so. The compiler enforcing it is not the same as a reader knowing it.
- **Keep cleverness on a leash.** Cleverness is fine; undocumented cleverness is debt.

This section embodies [[Human Understanding First]] and [[Explicit Not Magical]].

### 4. Test strategy

Split into four labeled subsections. Each test class targets a different failure mode; omitting any of them leaves a class uncovered.

#### a. Ordinary behavioral tests
- Confirm the design works on representative valid inputs.
- Catch outright wrong implementations.
- These are necessary but not sufficient.

#### b. Edge and boundary tests
- Empty, full, single-element, max, max+1, zero, negative, exact equality at thresholds.
- Catch off-by-one, fence-post errors, mishandled empty cases.
- See [[Edge Cases and Properties]].

#### c. Adversarial / torture tests
- Combine edges unpleasantly (empty + max, malformed + valid mixed).
- Repeated operations, idempotence violations, unusual orderings.
- Contradictory options. Invalid state transitions attempted from outside.
- Tests that **would embarrass a weak design**.
- Catch the bugs ordinary usage will never reveal — the entire point of [[Normal Usage Is Not Testing]].

#### d. Regression tests
- Each historical bug becomes a permanent test.
- Even if the bug seems unlikely to recur — *especially* if it does.
- Tests preserve hard-won knowledge that is otherwise only in commit messages and people's heads.
- See [[Regression Discipline]].

For **each class**, briefly state **what failure mode it is intended to expose**. A test list without that annotation is a list of activities, not a defense.

### When to skip the format

For trivial requests — a one-line edit, a question, a renamed variable — don't ceremoniously apply the four sections. But still:

- Favor [[Sharp Oracles|sharp oracles]] in any tests written.
- Surface invariants when they're load-bearing.
- Don't smuggle in cleverness without a comment.

The format scales with the stakes. The discipline doesn't.

*Source: [[Engineering Philosophy/Output Format]]*

---

## Reason About Correctness

*Correctness is an argument you write down, not an outcome you wait for; state invariants before code and check that the design actually preserves them.*

> Treat correctness as something to be **reasoned about**, not merely **hoped for**.

### What this means

- **Use types, interfaces, constructors, and module boundaries to make invalid states difficult or impossible.** Reasoning is easier when the set of representable states is closer to the set of legal states.
- **State the key invariants explicitly.** A property that holds because "the code happens to do that" is not maintained — it's lucky.
- **For each major component, explain *why* it should work before discussing *how* it is implemented.** The "why" is the correctness argument; the "how" is the artifact that satisfies it.
- **Where formal proof is impractical, provide an informal but disciplined argument.** "I tested it" is not an argument. "Here is the invariant and here is why every operation preserves it" is.

### The two halves of correctness work

#### Halve 1 — make wrong states unrepresentable

This is the constructive half. See [[Make Invalid States Unrepresentable]] in [[Rust Practices MOC]] for the Rust-specific techniques: newtypes, validated constructors, type-state, exhaustive enums.

When the type system rejects an illegal state, you have not "added a check." You have **eliminated the failure mode from the design**. There is nothing left to reason about, because the broken case cannot exist.

#### Halve 2 — argue for the cases the type system can't capture

Some invariants are not type-shaped:

- "After `flush()`, the on-disk state matches the in-memory state."
- "The graph constructed from this input is connected."
- "Accelerated output equals reference output within tolerance ε."

For these, write the argument. Even informally, even in a comment near the function, even in the design doc — but write it. The shape of a good argument:

1. **State the invariant.** Be specific.
2. **Identify the operations that could violate it.** Which functions touch the relevant state?
3. **Show that each one preserves the invariant.** Either by construction (the operation cannot do otherwise) or by maintenance (the operation actively re-establishes it).
4. **Note any assumptions** the argument depends on (clock monotonicity, single-writer, validation enabled, etc.).

This is what [[Output Format]]'s "Invariants and Assumptions" section is for.

### Why "reasoning" beats "testing alone"

Tests are samples. Reasoning is universal.

A test that passes shows the program worked **for those inputs**. An invariant argument shows the program works **for every input the invariant covers**. The two are complementary — see [[Tests Should Make Programs Fail]] — but reasoning is what gets you general statements; tests probe the boundary.

The strongest defense uses both: invariants narrow the space of possible bugs; adversarial tests probe what slips through.

### Anti-patterns

- **"It works in the tests we have"** as a stand-in for an argument. The tests bound a region of input space; they don't cover the rest.
- **"The compiler accepts it"** as a stand-in for an argument. The compiler enforces type rules, not domain invariants.
- **Implicit invariants.** A property only one person knows is one resignation away from being lost. Externalize it.

*Source: [[Engineering Philosophy/Principles/Reason About Correctness]]*

---

## Reasoning Beside Code

*Code without its reasoning rots into folklore; capture the why next to the what so future maintainers can keep both alive.*

> When producing code, also produce the **surrounding reasoning**.

Code without reasoning is a fact without an argument. The fact may be correct; the absence of the argument means no one can verify that, defend it under refactoring, or extend it with confidence.

### What "surrounding reasoning" means

For each non-trivial piece of work, deliver:

- **The architecture.** What the components are, how they fit together, why this decomposition. Not in exhaustive detail — enough that the next reader does not have to reverse-engineer the layout.
- **The invariants.** What properties the system maintains. Where each is established. Where each is relied on.
- **The likely failure modes.** What could go wrong. What would it look like. What would the system do.
- **The testing strategy.** Especially the **adversarial** part — see [[Tests Should Make Programs Fail]] and [[Output Format]]. Which failure classes the tests target, and which they intentionally don't.

This is the work of [[Output Format]]'s four-section template, applied at any scale: a one-line edit doesn't need a treatise; a new module deserves a paragraph or three; a new subsystem deserves the full structure.

### Why this matters

#### 1. It externalizes the engineer's mental model

The hardest part of joining a project late is rebuilding the original engineer's mental model. Recorded reasoning short-circuits that. Without it, every new contributor pays the same archaeology tax.

#### 2. It makes the design auditable

A claim like "this can never deadlock" is unauditable on its own. The same claim with a brief argument — "no two locks are ever acquired in different orders, because all multi-lock acquisitions go through `acquire_pair()`, which orders them by address" — can be checked. Future changes that violate the argument become visible as violations.

#### 3. It catches design flaws in the writing

The act of writing the reasoning often reveals the holes. A correctness argument you cannot complete is a correctness argument that **does not exist yet**, even if the code happens to work. Better to find that out at design time than at incident time.

#### 4. It protects against future "small changes"

Most catastrophic regressions come from "small changes" made without understanding the load-bearing assumptions. Recorded reasoning makes assumptions visible. The change either respects them, updates them deliberately, or visibly violates them.

### Where the reasoning lives

Different scales of reasoning belong in different places:

- **System-wide invariants** → vault notes (this codex).
- **Subsystem / module-level reasoning** → module-level doc comments and design docs.
- **Function-level invariants** → `///` doc comments. See [[Documentation as Truth]].
- **Single-line subtleties** → a brief code comment, only when the *why* would surprise a reader.

Match the scale of the reasoning to the scale of the decision. Treatises about trivial helpers are noise; a one-line comment on a load-bearing invariant is gold.

### What "reasoning" is NOT

- It is not "explain what the code does." Well-named code does that. The reasoning is *why* the code is shaped this way.
- It is not "narrate every function call." Most calls speak for themselves.
- It is not retroactive marketing. Documentation written after the fact to make a design *sound* coherent without actually verifying it is worse than no documentation at all — it lies.

### A small example

#### Without reasoning

```rust
pub fn submit(&mut self, job: Job) -> Result<JobId, SubmitError> {
    let id = self.next_id;
    self.next_id += 1;
    self.queue.push_back((id, job));
    Ok(JobId(id))
}
```

#### With reasoning

```rust
/// Submit a job to the queue, returning a unique [`JobId`] for tracking.
///
/// # Invariant
/// `JobId`s are strictly monotonically increasing within a single
/// `Scheduler` instance. Callers may rely on `id_a < id_b` implying
/// `job_a` was submitted before `job_b`.
///
/// # Errors
/// Returns [`SubmitError::QueueFull`] when the queue has reached
/// `MAX_PENDING_JOBS`; the caller may retry after `poll`.
///
/// # Note
/// Wraparound of the underlying `u64` counter is not handled — at
/// the documented submission rate this is centuries away. If the
/// rate increases, revisit.
pub fn submit(&mut self, job: Job) -> Result<JobId, SubmitError> {
    if self.queue.len() >= MAX_PENDING_JOBS {
        return Err(SubmitError::QueueFull);
    }
    let id = self.next_id;
    self.next_id += 1;
    self.queue.push_back((id, job));
    Ok(JobId(id))
}
```

Same code, much more legible argument. Future maintainers know what they are allowed to rely on, what would break the invariant, and which assumption (`u64` wraparound) is parked deliberately.

*Source: [[Engineering Philosophy/Principles/Reasoning Beside Code]]*

---

## Explicit, Not Magical

*Cleverness saves keystrokes and costs comprehension; choose explicit code so debugging and inspection stay straightforward.*

> Prefer **clear control flow**, **stable interfaces**, and **obvious data movement**. Avoid cleverness that obscures reasoning. Make debugging and inspection straightforward.

### What "explicit" means

- **Control flow you can follow with your eyes.** A reader should be able to trace a value from input to output by reading top-to-bottom, not by chasing through a web of macros, traits, and reflection.
- **Stable interfaces.** A function's signature should imply how it's called, what it requires, and what it returns. Surprising side effects, hidden global lookups, and "do something different based on context" behavior are anti-features.
- **Obvious data movement.** Where data is constructed, where it's transformed, where it's consumed. If a struct's contents change, it should be clear *which function* changed them.
- **Inspection-friendly.** When something goes wrong, a `Debug` print, a logged span, or a stepped debugger should reveal what's happening. State that requires a custom tool to inspect is state that hides bugs.

### Why "magical" is a problem

Magical code — clever metaprogramming, deep generic chains, action-at-a-distance via globals or interior mutability, behavior that depends on attributes the compiler interprets — fails on three axes:

1. **Reading.** The reasoning is hidden. The reader cannot see what the program does without already understanding the magic.
2. **Debugging.** When the magic misbehaves, you cannot step through it; you can only stare at the output and guess.
3. **Refactoring.** Touching anything risks invalidating an invariant nobody documented because the magic "obviously" maintained it.

"Magic" is fine to *use* (libraries are full of it). It is **not** fine to *introduce* in your own code unless the alternative is meaningfully worse.

### Concrete heuristics

- **Prefer concrete types over generics** when only one type is ever used. See [[Traits as Seams]].
- **Prefer functions over macros** when the macro doesn't earn its keep. Macros that exist to save typing are debt.
- **Prefer explicit collaborators over global lookups.** A function that takes a `&Clock` is reasonable; one that consults a `static CLOCK` is a hidden coupling.
- **Prefer one obvious way over three clever ones.** A codebase with three different "configuration" patterns has none.
- **Prefer Debug-printable state.** If a type is opaque, derive `Debug` thoughtfully or implement a useful manual one. A field whose value cannot be inspected is one you can't debug.
- **Prefer `match` over deep `if let`/`Option` chains** when the intent is "handle each case." The match makes the case set visible.

### When "explicit" loses

A few legitimate cases where the magical option wins:

- **Derives.** `#[derive(Debug, Clone, PartialEq)]` is magic, but it is **shared, well-known magic** that every Rust reader understands. Don't reimplement these by hand for purity points.
- **Well-established framework conventions.** Serde, clap, thiserror — these are magical *and* universal. Reinventing them by hand is worse for readers, not better.
- **Genuine performance-critical hotspots** where the explicit version measurably loses. Measure first.

The principle is "explicit, not magical" — not "no abstraction at all." Use abstractions whose magic is well-documented and widely understood.

### Composes with [[Human Understanding First]]

This principle is the *defensive* sibling of "human understanding first." That principle says: structure code so a reader can follow it. This one says: do not undermine that with cleverness in the interest of saving lines.

The two together: **build code a reader can read, and don't sneak in things the reader can't see**.

*Source: [[Engineering Philosophy/Principles/Explicit Not Magical]]*

---

## Human Understanding First

*Optimize for the human who will read this next; if the design is not narratable, it is not really understood.*

> Structure code so that a reader can understand the **reasoning** behind it. Make the design **narratable**.

### What this means in practice

- **Expose invariants, assumptions, preconditions, postconditions, state transitions.** A reader should not have to reverse-engineer them.
- **Prefer organization that supports explanation and correctness over organization that is merely convenient for the machine.** A clever flat layout that's hard to walk through is a worse choice than a more verbose one a reader can follow.
- **Each major component should have a paragraph explaining its job before its code.** Not what it does (the code shows that) — *why it exists*, *what contract it satisfies*, *what invariants it maintains*.
- **Names earn their length.** A six-character abbreviation that needs a comment to decode is worse than a twenty-character name that doesn't.

### Why this is the first principle

If a reader cannot understand the reasoning, every other principle fails:

- They cannot **audit the correctness argument** ([[Reason About Correctness]]).
- They cannot **identify the neglected paths** to attack with adversarial tests ([[Normal Usage Is Not Testing]]).
- They cannot **debug a failure** when one occurs ([[Explicit Not Magical]]).
- They cannot **strengthen the suite** when a bug is found ([[Regression Discipline]]).

Understanding is the substrate. Without it the rest is decorative.

### Practical tests

Read the code as if you were a teammate joining the project six months from now:

- Can you state, after reading the file, what **invariants** the type or function maintains?
- Can you point to **where** each invariant is established?
- Can you say **why** the design chose this shape rather than an alternative?
- Can you find the **state transitions** without grepping for `mut`?

If any answer is "no," the structure is hiding the reasoning, not expressing it.

### What this is NOT

- **It is not "more comments."** Most comments restate code or rot into lies. Comments are useful for the *why* the code can't show — invariants, non-obvious constraints, references to bugs. See [[Documentation as Truth]] for the discipline.
- **It is not novel formatting or decorative diagrams.** A clear struct definition with one paragraph above it usually beats a UML diagram in a separate file.
- **It is not exhaustive prose.** A trivial helper does not need a treatise; surface the reasoning in proportion to the stakes.

### How this composes with [[Rust Practices MOC]]

- [[Newtypes and Domain Types]] makes invariants legible at the type level, not just in prose.
- [[State Transition Types]] makes phases visible at the type level — the reader sees the lifecycle in the signatures.
- [[Boring Data Layouts]] is this principle expressed at the data-modeling level.

*Source: [[Engineering Philosophy/Principles/Human Understanding First]]*

---

## Related bundles

- [[Bundles/Rust/Module-Design]] — when the analysis is for a new module, design and writing happen together
- [[Bundles/Rust/Test-Design]] — the test-strategy section of any design doc draws from this bundle
- [[Bundles/Rust/Code-Review]] — design docs and review checklists share the same correctness vocabulary
