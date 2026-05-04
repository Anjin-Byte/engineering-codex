---
title: Contributing to this Vault
tags: [meta, authoring, guide]
summary: Standards for adding, organizing, and linking notes — including the generalization test that gates project lessons into general knowledge.
---

*Project pain motivates new notes; only generalized lessons enter the vault. Strip the project nouns first; if the rule still stands, it belongs.*

# Contributing to this Vault

This vault collects engineering standards that apply across projects. New notes will accrete over time as real work surfaces lessons. The standards below keep the vault dense, navigable, and free of project trivia.

If you (human or agent) are about to add or change a note, read this file first.

If instead you've observed a problem with an *existing* note during real use — wrong claim, contradiction, vagueness, missed case, generalization slip — that is not a contribution; file a tracked issue per [[ISSUES]]. Edits to existing notes flow through the issue process, not through this guide.

---

## 1. The generalization test

**The gate.** Before writing or extending a note, run this test:

> Remove every project-specific noun, framework name, library name, and domain term. Does the lesson still hold? Does it still teach?

- **Yes** → it belongs. Write it in the generalized form.
- **No** → it's project trivia. Keep it in the project's repo, not here.

### Worked example

A team learns: *"Our `OrderService` blew up because the `Postgres` connection pool was shared between the request handler and the background worker; we fixed it by giving the worker its own pool."*

Strip the nouns: *"A shared mutable resource between two consumers with different lifetimes will eventually surface as a hang or corruption; give each lifetime its own instance."*

That generalizes. It belongs (likely as an addition to [[Languages/Rust/Practices/Ownership and Mutation/Minimize Shared Mutability]]).

Counter-example: *"Always pin `tokio` to 1.35 because 1.36 broke our retry logic."* — useful for the project, no general lesson, doesn't belong.

### What "general" means here

A note is general if it would still be true and useful for a project the author has never seen, written in a language they may not use. The Engineering Philosophy scope is the gold standard — its principles travel to any language. Per-language scopes (`Languages/<lang>/`) are general within their language: they assume the language but make no project-specific assumptions inside that scope.

### Language-scope dimension

The vault now spans multiple language disciplines. When the generalization test passes, run a second test:

> If the lesson applies to ≥ 2 languages we cover, it belongs under `Engineering Philosophy/`, not under any single language scope. If it applies to one language only, it belongs under `Languages/<lang>/`.

A lesson that *seems* universal but uses language-specific mechanisms (e.g. "use Cargo features for capability slicing") splits into two notes: the universal principle in `Engineering Philosophy/`, the language-specific application in `Languages/<lang>/`. Pick the level the dominant claim sits at.

---

## 2. New note or extend existing?

**Default to extending.** A new note is justified only when a distinct thesis emerges that an existing note cannot absorb without losing focus.

Decision flow:
1. Search the vault for the topic. Read the closest existing notes.
2. If your lesson **sharpens, qualifies, or adds a worked example** to an existing note → extend it.
3. If your lesson **conflicts** with an existing note → resolve the conflict in the existing note (one of you is wrong, or the rule needs a stated exception). Do not create a parallel note.
4. If your lesson is **a new thesis** that would dilute an existing note's focus → new note.

A note exists to defend exactly one claim. If you find yourself writing "and another thing…" twice, you have two notes.

---

## 3. Note shape

Every note follows this skeleton. A copy-paste template is at the bottom of this file.

### Required frontmatter

```yaml
---
title: Plain Title Case Title
tags: [category, subcategory]
summary: One sentence, ≤ 25 words, self-contained — works when the note is pulled out of context.
---
```

- `title` matches the filename (without `.md`).
- `tags` use the conventions already in the vault — check sibling notes; do not invent new tag taxonomies casually.
- `summary` is a *description* of the note, suitable for an index or RAG retrieval.

### Required body structure

```
*One italic line: the thesis in compressed form.*

# Title

> Optional pull-quote: the rule in its sharpest form.

## The principle  (or a similar opening section)
...

## Related
- [[Folder/Other Note]]
- [[Folder/Another Note]]
```

- The italic TL;DR is *content*, not description. Different from `summary:`.
- Lead with the rule. No "This note explains…" preamble.
- Code examples only when they sharpen the point. Strip them to the minimum that makes the case.
- A `## Related` section at the end is required if the note has any siblings — and it almost always does.

---

## 4. Where it goes

| Folder | Use when… |
|---|---|
| `Engineering Philosophy/Principles/` | The lesson is language-agnostic — about reasoning, testing posture, architectural principle, or how to attack programs. Travels to any project, any language. |
| `Engineering Philosophy/` (root) | Format and process notes that frame how output is produced (e.g. [[Engineering Philosophy/Output Format]]). |
| `Languages/Rust/Practices/Foundational/` | The lesson is a load-bearing Rust principle that touches multiple subcategories (push correctness left, predictable APIs, type system as design tool). |
| `Languages/Rust/Practices/Type-Driven Design/` | About using the type system to constrain or model — newtypes, builders, state types, making invalid states unrepresentable. |
| `Languages/Rust/Practices/Effect Isolation/` | About separating pure logic from effects — pure cores, traits as seams, testability without heroics. |
| `Languages/Rust/Practices/Error Handling/` | About `Result`, panics, error types, error boundaries. |
| `Languages/Rust/Practices/Ownership and Mutation/` | About ownership, borrowing, shared mutability, `unsafe`. |
| `Languages/Rust/Practices/Functions and Data/` | About function shape, contracts, data layout. |
| `Languages/Rust/Practices/Testing/` | About what to test, at what level, with what oracles. |
| `Languages/Rust/Practices/Tooling and Quality/` | About clippy, CI, docs, formatting — anything enforced by Rust tooling. |
| `Languages/Rust/Workspace/` (root) | Cargo workspace-scale rules: layout, configuration, the load-bearing principles of how crates relate. |
| `Languages/Rust/Workspace/Patterns/` | Cargo-specific recurring patterns: shared dependencies, workspace lints/profiles, Cargo feature mechanics. |
| `Languages/<other-lang>/...` | (When future language scopes land — TypeScript, etc. — they mirror this shape: `Languages/<lang>/Practices/...` and `Languages/<lang>/Workspace/...` or equivalent project-shape folder.) |
| `Integration/Rust-WASM-TS/` | Cross-language integration patterns: e.g. Rust→WASM→TS interop, generated TS surfaces, boundary-crossing performance. Cross-references both Rust and TS scopes. |

If a note plausibly fits two folders, pick the one that matches the **dominant claim** of the note — not the topic it touches in passing. If the note is genuinely cross-language, prefer `Engineering Philosophy/` (universal principle) plus a thin language-specific application note over duplicating the principle in each language scope. If neither folder fits, that is a signal: re-read section 2 (you may be writing the wrong note) and section 1 (it may be project-specific).

---

## 5. Linking conventions

- **Use full-path wikilinks** in the body and in MOCs: `[[Languages/Rust/Workspace/Patterns/Shared Dependencies]]` — never bare `[[Shared Dependencies]]`. Resolves unambiguously when the vault is read as raw markdown by an agent or by another tool. Cross-scope wikilinks (e.g. a per-language note linking into `Engineering Philosophy/`) follow the same rule.
- **Meta-file exception.** A few filenames appear at multiple paths by design — `AGENTS.md` (root router plus one per scope), `README.md` (root plus consumer-template), `CLAUDE.md` (root plus consumer-template). Convention: bare `[[AGENTS]]` / `[[README]]` / `[[CLAUDE]]` always refers to the **vault-root** copy (the entry point); any non-root copy must be path-qualified (`[[consumer-template/README]]`, `[[Languages/Rust/AGENTS]]`, etc.). The root [[AGENTS]] router is an exception to the full-path-by-default rule because the bare form is the canonical entry point.
- **Bundle basenames.** Task bundles share basenames across scopes — `Code-Review.md` exists under `Bundles/Rust/`, `Bundles/TypeScript/`, and any future per-scope `Bundles/` folder. There is no canonical scope, so **bundle wikilinks must always be path-qualified** (`[[Bundles/Rust/Code-Review]]`, `[[Bundles/TypeScript/Code-Review]]`, `[[Bundles/Universal/Written-Analysis]]`). Bare `[[Code-Review]]` is ambiguous and prohibited.
- **Every note must be linked from at least one MOC** in its scope *and* from at least one other note's `## Related` section. Orphan notes do not exist in this vault.
- **`## Related` is curated, not exhaustive.** List 2–5 strongest neighbors. If you find yourself listing eight, the note's scope is too broad — see section 2.
- **Links are bidirectional in spirit.** If note A links to note B as related, note B should link back. Check both directions when adding a Related entry.
- **Never link to non-existent notes.** No `[[TODO: write this]]` placeholders. If the linked idea doesn't have a note yet, write the note first or use prose.

---

## 6. Style

The vault is direct. Knuth-direct. Match the existing tone:

- **Lead with the rule.** The first non-frontmatter line should state the thesis. The first heading should defend it.
- **No throat-clearing.** Drop "This note explains…", "In this section we will…", "It is important to note that…".
- **Imperative voice for rules.** "Push the check to the type system" beats "It is generally a good idea to push the check to the type system when possible."
- **Be specific.** "Tests should fail when the bug is present" not "tests should be good." Cite mechanisms.
- **Worked examples earn their space.** A code block must illustrate something the prose alone cannot. If the prose already conveys it, drop the code.
- **No emoji, no decorative formatting.** Tables and pull-quotes when they sharpen, not when they decorate.
- **Footnotes and citations sparingly.** If a source is load-bearing (Knuth, an RFC, a paper), name it inline.
- **No contributor disclosure, ever.** Vault content must not name any individual or agent contributor — no author bylines, acknowledgements, "written by" / "drafted with" notices, AI-disclosure language, or `author:` frontmatter fields. The vault is authorless engineering standards; its authority is the content, not the source. This rule extends to commit messages (§10.2), changelog entries (§11.4), and the CHANGELOG release-cutting checklist (§11.3).

---

## 7. Updating the index

Adding or substantially editing a note has three downstream consequences. **All three are required**, not optional:

1. **The scope's MOC** — for `Engineering Philosophy/`: [[Engineering Philosophy/Engineering Philosophy MOC]]; for `Languages/Rust/Practices/`: [[Languages/Rust/Practices/Rust Practices MOC]]; for `Languages/Rust/Workspace/`: [[Languages/Rust/Workspace/Workspace Architecture MOC]]. Future language scopes have their own MOCs. List the note with a one-line summary in the appropriate group.
2. **The scope's AGENTS.md "When to consult what"** — every scope has its own AGENTS.md ([[Engineering Philosophy/AGENTS]], [[Languages/Rust/AGENTS]], [[Integration/Rust-WASM-TS/AGENTS]]). If the note changes *when* an agent should consult the vault (a new trigger, a new work type), add or update a trigger group in the relevant scope's AGENTS.md. If the trigger genuinely spans scopes, the trigger lives in the scope where the dominant action is taken.
3. **The scope's AGENTS.md "Full note index"** — every scope's AGENTS.md indexes its own scope. List the note with its `summary:` text in the appropriate folder group. Every note appears in exactly one scope's index.

If the note replaces an existing one or significantly changes its thesis, also audit `## Related` sections in neighbor notes for staleness.

The index update is the single most-skipped step. It is the step that, when skipped, decays the vault fastest. Treat it as part of the note, not as cleanup.

To detect when this step has been skipped, prompt an agent to run the audit procedure in [[AUDIT]]. It cross-checks every note against MOC presence, Related-section linkage, and the AGENTS.md full index — exactly the three consequences above.

---

## 8. Examples

### Good note (existing)

[[Engineering Philosophy/Principles/Sharp Oracles]] — one thesis (an assertion is only as strong as its oracle), defended directly, with concrete techniques (closed-form references, differential testing, invariants), linked Related section, generalizes to any language.

### Anti-patterns to reject

- *"Notes on our payment retry bug."* — project-specific, no generalized thesis. Doesn't pass the gate.
- *"Various tips for writing better code."* — multiple theses, no focus. Should be split or absorbed into existing notes.
- A note with no `## Related` section, no incoming MOC link, and no outgoing wikilinks — orphan, fails section 5.
- A note whose body restates its `summary:` in three different ways — padding. Cut to the rule.
- A note titled `Async Patterns` that turns out to be entirely about one project's `tokio` runtime configuration — failed the generalization test.

---

## 9. Template

Copy this skeleton into a new file under the appropriate folder.

```markdown
---
title: Title in Plain Title Case
tags: [category, subcategory]
summary: One sentence, ≤ 25 words, self-contained.
---

*One italic line stating the thesis in compressed form.*

# Title in Plain Title Case

> Optional pull-quote sharpening the rule.

## The principle

State the rule. Defend it.

## Why

The reasoning. What goes wrong without it.

## How to apply

Concrete techniques, in declining order of generality.

## Related

- [[Folder/Sibling Note]]
- [[Folder/Another Sibling Note]]
```

After saving the new file, complete the three index updates from section 7.

The template should also include `keywords:` after `summary:` — see existing notes for the convention.

---

## 10. GitHub workflow

The vault lives in a GitHub repository. Contributions land through pull requests against `main`. This section covers the repo-level mechanics; sections 1–9 still govern the *content* of any PR.

### 10.1 Branches

- `main` is the integration branch. It is always releasable. Direct pushes are not allowed; everything goes through a PR.
- Feature branches use the prefix scheme: `add/<note-slug>` (new note), `fix/<issue-id>` (resolves a tracked issue), `audit/<scope>` (audit cleanup), `docs/<short-desc>` (meta-file or README edit), `chore/<short-desc>` (tooling, repo hygiene).
- Branch from current `main`. Keep branches short-lived; rebase on `main` if it advances during your work.

### 10.2 Commit messages

- One commit, one purpose. A PR may have several commits, but each one should be reviewable on its own.
- Imperative mood, ≤ 72-char subject line: `add: Sharp Oracles note` / `fix: broken wikilink in Test-Design bundle` / `docs: clarify versioning policy in CONTRIBUTING`.
- Body (when needed) explains *why*, not what. The diff shows what.
- If a commit closes a tracked vault issue, reference the issue file path: `Resolves Issues/2026-04-26-sharp-oracles-missing-property-test-case.md`.
- **No contributor disclosure in commit metadata** (per §6). Do not add `Co-Authored-By:` trailers, `Signed-off-by:` lines that reveal personal identity beyond what git already records, or "with help from"-style mentions in commit bodies. The git author field will exist as a mechanical record; nothing in the message should add to it.

### 10.3 Pull request mapping

A PR fits into one of four lanes. Each lane has different requirements; mark the lane in the PR description.

| Lane | What it does | Required by |
|---|---|---|
| **Add** | Creates new content (note, bundle, meta-file). | §1–§9 of this guide. |
| **Refine** | Resolves a tracked issue in `Issues/`. | [[ISSUES]] §8 (resolution flow). |
| **Audit** | Applies findings from a vault audit. | [[AUDIT]] §4 repair policy. |
| **Meta** | Touches CONTRIBUTING/AGENTS/README/AUDIT/ISSUES/CHANGELOG. | This guide; usually multi-lane in effect. |

A single PR should stay in one lane when possible. Mixing lanes is allowed when the work is genuinely entangled (e.g. an audit finding triggers a refinement that also requires a CONTRIBUTING update), but state the entanglement in the PR description.

### 10.4 Pull request requirements

Every PR description (use the template at `.github/pull_request_template.md`) must include:

1. **Lane** — one of Add / Refine / Audit / Meta.
2. **What changed** — bullet list, file-level granularity.
3. **Why** — the lesson, observation, or finding driving the change. For a Refine PR, link the resolved issue file.
4. **Index updates done** — checklist confirming the three consequences from §7 (the scope's MOC, the scope's AGENTS.md trigger group fit, the scope's AGENTS.md full index) when applicable.
5. **Audit checks run** — for non-trivial PRs, state which subset of [[AUDIT]] §1 was run and that it passed (or list new findings the PR addresses). PRs that only touch a single note's body need not run the full audit.
6. **Changelog entry** — confirm the `## [Unreleased]` section of [[CHANGELOG]] has been updated with a one-line entry under the appropriate category. PR will not merge without this.

### 10.5 Reviews

- One approving review required for Add and Refine PRs. Reviewer checks: generalization test passed (no project nouns; lesson travels), single-thesis discipline, all required index updates, correct folder placement, link integrity.
- Audit PRs may self-merge if every finding is in [[AUDIT]] §4's auto-fixable category and a one-line summary of fixes is in the PR description.
- Meta PRs that touch CONTRIBUTING, AGENTS, AUDIT, or ISSUES require a second pair of eyes regardless of size — these files are load-bearing.

### 10.6 Vault `Issues/` vs GitHub Issues

The vault has two issue surfaces; they cover different things and should not be conflated.

- **Vault `Issues/` folder** ([[ISSUES]]) — tracked refinements to a specific note. Lives in the repo, version-controlled, has a defined lifecycle. Use this when there is a target note.
- **GitHub Issues** — repo-level concerns that don't have a target note: tooling, infrastructure, broad cross-cutting discussions ("should we add a Glossary?"), CI failures, integration friction with consuming projects. Use these when the issue spans the vault as a whole or is meta to its content.

A vault issue *should not* be mirrored as a GitHub Issue. PRs that resolve a vault issue reference the vault issue file path; that is the canonical thread. GitHub Issues that ask for a new note should be redirected to the [[CONTRIBUTING]] flow (write the note as a PR, don't accumulate a wishlist of unwritten notes in the issue tracker).

### 10.7 Forking and external contributors

If you are contributing from a fork:
1. Fork the repo, branch as in §10.1, push your branch to your fork.
2. Open a PR into `main` of the upstream repo. Use the same template.
3. Be prepared to rebase on `main` after review.
4. The maintainer may push small fixes (typos, link corrections) directly to your branch before merge if you allow maintainer edits.

---

## 11. Versioning and the changelog

The vault is versioned. Consuming projects can pin to a specific version; future versions can introduce breaking changes (note renames, trigger-group renames) safely under a tag.

### 11.1 Version format

`vMAJOR.MINOR.PATCH` — semver-ish, with a knowledge-vault interpretation:

- **PATCH** (`v0.3.0` → `v0.3.1`) — typos, broken links, frontmatter fixes, audit cleanup, single-note clarifications that do not change the rule. Safe to update without inspection.
- **MINOR** (`v0.3.0` → `v0.4.0`) — new notes, new bundles, new trigger groups, summary/TL;DR rewrites, expansion or qualification of an existing rule, new meta-files. Additive; does not invalidate consumer references.
- **MAJOR** (`v0.X.Y` → `v1.0.0`) — breaking changes. A change is breaking if it would silently invalidate a path or reference that a consumer might have hardcoded:
  - Note renamed, moved between folders, or deleted (breaks wikilinks and consumer-template paths).
  - Trigger group in AGENTS.md renamed or removed (breaks consumer-template snippets that reference triggers).
  - Frontmatter schema change that tooling depends on (e.g. removing `summary:` or renaming `keywords:`).
  - Material reversal of a load-bearing rule that consumers may be following (e.g. a Foundational note's thesis flips).

### 11.2 Pre-1.0 caveat

Until `v1.0.0`, MINOR releases may include breaking changes. Consumers should pin to a specific tag during this period, and the maintainer should call out breaking changes in the CHANGELOG even when the version bump alone wouldn't signal them. `v1.0.0` is reached when the structure is stable enough that semver guarantees can be honored.

### 11.3 Cutting a release

1. Open a release PR titled `release: v0.X.0`.
2. In the PR, move the `## [Unreleased]` block in [[CHANGELOG]] under a new `## [0.X.0] — YYYY-MM-DD` heading. Reset the Unreleased section to empty category headings.
3. Confirm every entry in the new release block is accurate and links to the file or issue it concerns. Entries describe *what changed*, never *who changed it* (per §6).
4. Merge the PR.
5. Tag `main` at the merge commit: `git tag -a v0.X.0 -m "v0.X.0"` and push the tag.
6. Optionally create a GitHub Release pointing at the tag, with the changelog block as the body.

Releases should be cut on a cadence the project can sustain — monthly or after each cluster of resolved issues is reasonable. Avoid daily releases (low signal) and avoid letting the Unreleased section grow past a few dozen entries (loses navigability).

### 11.4 What goes in the changelog

The Unreleased section of [[CHANGELOG]] is updated continuously — every PR adds a line under the right category before merge. Categories follow Keep a Changelog plus one extension:

- **Added** — new content (notes, bundles, meta-files, frontmatter fields).
- **Changed** — modifications to existing content. Note renames or moves go here AND must trigger a MAJOR bump.
- **Deprecated** — content kept but marked for removal in a future version.
- **Removed** — content deleted.
- **Fixed** — bug-style corrections (broken links, frontmatter errors, audit cleanup that fixes drift).
- **Issues Resolved** — closing a tracked issue in `Issues/` (extension; references the issue file path so the changelog doubles as the resolution log).

Entries are short, declarative, and link to the affected file or issue. Examples:

```markdown
### Added
- New note: [[Languages/Rust/Practices/Effect Isolation/Side Effect Inventory]] — catalogs effect categories that should sit at the edge.

### Changed
- [[Engineering Philosophy/Principles/Reference Implementation as Oracle]] — generalized title and body to cover SIMD/FFI cases. **Breaking:** wikilink target moved.

### Issues Resolved
- [Issues/2026-05-12-sharp-oracles-property-test-case.md](./Issues/2026-05-12-sharp-oracles-property-test-case.md) — added a worked property-test example to [[Engineering Philosophy/Principles/Sharp Oracles]].
```

### 11.5 Resolved issues feed the changelog

When an issue in `Issues/` is closed (status: resolved), the resolver:

1. Completes the resolution per [[ISSUES]] §8.
2. Adds an `Issues Resolved` entry to the Unreleased section of [[CHANGELOG]] with a link to the issue file and a one-line description of what was changed in which note.

This is non-optional. The changelog is the public-facing record of the vault's refinement history, and resolved issues are a major source of refinement.
