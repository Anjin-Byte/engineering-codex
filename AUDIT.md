---
title: Vault Drift Audit Procedure
tags: [meta, audit, agent-instructions]
summary: Step-by-step procedure an LLM follows when prompted to audit this vault for drift against CONTRIBUTING.md.
---

*A repeatable runbook: load the spec, run mechanical checks, run semantic checks, emit a structured report with severities and proposed fixes.*

# Vault Drift Audit Procedure

This file is a runbook for an LLM agent. When the user asks "audit the vault," "check for drift," "validate the vault," or similar, follow this procedure end-to-end before reporting.

The audit is a comparison: vault state **vs** [[CONTRIBUTING]]. Anything in the vault that violates a rule in CONTRIBUTING is drift. Anything ambiguous is a finding for human judgment, not for autonomous repair.

---

## 0. Setup

1. Read [[CONTRIBUTING]] in full. It is the spec.
2. Read every AGENTS file in full — they are audited artifacts (full indices + trigger groups). The set is: [[AGENTS]] (root router), [[Engineering Philosophy/AGENTS]], [[Languages/Rust/AGENTS]], [[Integration/Rust-WASM-TS/AGENTS]], and any future per-language or per-integration AGENTS files. Call this set **A**.
3. Read [[README]] in full. Short — it is also audited.
4. Enumerate every `.md` file under the vault root, excluding `.obsidian/` and `reference_reports_not_apart_of_vault/`. Call this set **N**.

Do not skip step 1. Without re-reading CONTRIBUTING, the audit drifts itself.

---

## 1. Mechanical checks

These are deterministic — no judgment, just structural verification. Run all of them. Each emits findings into the final report.

### 1.1 Frontmatter completeness

For every note in **N**:
- Has a YAML frontmatter block (`---` ... `---`) at the top.
- Block contains required keys: `title`, `tags`, `summary`.
- `summary` is one sentence, ≤ 25 words, present, non-empty.
- `title` matches the filename (without `.md`).

**Finding form:** `[FRONTMATTER] <path> — missing key: summary` / `title mismatch: file is X, frontmatter says Y`.

### 1.2 TL;DR line

For every note in **N**:
- Immediately after the closing `---`, there is a blank line, then a single italicized line (matching `^\*[^*].*\*$` in spirit), then a blank line, then the first heading.

Index-style files (MOCs, indices, AGENTS, README, CONTRIBUTING, AUDIT) may have their TL;DR describe the index rather than assert a thesis — acceptable.

**Finding form:** `[TLDR] <path> — missing italic TL;DR line` / `TL;DR present but malformed`.

### 1.3 Wikilink resolution

Extract every wikilink `[[...]]` from every note in **N** plus every file in **A** (all AGENTS files), README.md, and CONTRIBUTING.md.
- Each link must resolve to an existing file in the vault.
- Links inside `## Related` sections **must** use full paths (`[[Folder/Note Name]]`), not bare names. The same rule applies to all body links per [[CONTRIBUTING]] §5.
- Cross-scope wikilinks (e.g. a Rust note linking into `Engineering Philosophy/`) follow the same path-qualified form.

**Finding form:** `[LINK] <source> → [[target]] — does not resolve` / `bare link in Related section, should be [[Folder/Note]]`.

### 1.4 Orphan check

For every note in **N**, count incoming wikilinks from:
- The note's scope MOC. The set of scope MOCs is: [[Engineering Philosophy/Engineering Philosophy MOC]], [[Languages/Rust/Practices/Rust Practices MOC]], [[Languages/Rust/Workspace/Workspace Architecture MOC]], plus any future per-language MOCs.
- Any `## Related` section in any other note.
- The scope's AGENTS.md full index. Each scope's AGENTS.md indexes its own scope; every note appears in exactly one scope's index.

Every note must have **≥ 1 scope-MOC link** AND **≥ 1 Related-section link** AND **exactly 1 scope-AGENTS index entry**. Notes at the vault root (README, AGENTS, CONTRIBUTING, AUDIT, ISSUES, CHANGELOG) and the per-scope AGENTS files themselves are exempt from the MOC requirement but should appear in the root [[AGENTS]] router or be linked from it.

**Finding form:** `[ORPHAN] <path> — no scope-MOC link` / `not in any Related section` / `missing from scope AGENTS.md index` / `appears N times across AGENTS.md indices (expected 1)`.

### 1.5 Project-noun scrub

Grep across the entire vault (case-insensitive) for project-specific tokens that should never appear:

```
SDF, lattice, wgpu, winit, Quilez, Woodward, Inayat, isosurface,
SIBR, lattice-gen, xtask, pressure drop, AM platform, mesh decimation
```

Some words may legitimately appear in unrelated code examples (e.g. a function literally named `mesh()` in a non-mesh context). Use judgment but default to scrub.

**Finding form:** `[PROJECT-TOKEN] <path>:<line> — "<token>" — likely leak from source vault`.

### 1.6 Forbidden Obsidian-only syntax

Grep for syntax that doesn't render outside Obsidian:
- Embeds: `![[...]]`
- Callouts: `> [!note]`, `> [!warning]`, etc.
- Dataview blocks: ` ```dataview `

**Finding form:** `[NON-PORTABLE] <path>:<line> — embed/callout/dataview should be plain markdown`.

### 1.7 AGENTS files structural integrity

For every file in **A** (root AGENTS, every per-scope AGENTS):

- Every note that belongs to that scope appears exactly once in the AGENTS file's "Full note index" section, under its correct folder group.
- Every wikilink in any AGENTS file uses a full path.
- "When to consult what" trigger groups link only to existing notes.
- The root [[AGENTS]] router contains scope-detection routing pointing to every per-scope AGENTS file that currently exists.
- A note must appear in exactly one scope's full index, never in two — cross-scope visibility is achieved through wikilinks within trigger groups, not duplicate indexing.

**Finding form:** `[AGENTS] note <path> missing from <scope> AGENTS.md index` / `note appears in wrong folder group within <scope> AGENTS.md` / `trigger group "X" links to non-existent note` / `note <path> indexed in N AGENTS files (expected 1)` / `root AGENTS.md routing missing pointer to <scope> AGENTS.md`.

### 1.8 Bundle freshness

Each file under `Bundles/<scope>/` is a pre-materialized inlining of several source notes. Bundles can drift from their source notes when the sources are edited. Verify:

- Every bundle has `source_trigger:` and `bundles:` fields in frontmatter.
- The `source_trigger:` value matches an existing trigger group in the bundle's scope AGENTS.md "When to consult what" (e.g. a `Bundles/Rust/` bundle's source_trigger matches a trigger in [[Languages/Rust/AGENTS]]; a `Bundles/Universal/` bundle's source_trigger matches a trigger in [[Engineering Philosophy/AGENTS]]).
- The `bundles:` list matches the notes the trigger group currently references — no missing inlines, no extras.
- Each inlined note's `## <Note Title>` section opens with an italic TL;DR identical to the current TL;DR in the source note.
- Each inlined note section ends with a `Source: [[Folder/Note]]` link that resolves.
- The bundle's `## Related bundles` section links only to existing bundle files (path-qualified to the correct scope under `Bundles/Rust/`, `Bundles/Universal/`, or another scope folder).
- Bundles are scope-pure: a `Bundles/Rust/` bundle should not inline notes from outside Engineering Philosophy or Languages/Rust. Cross-language bundles, if any, live under `Bundles/Integration/` (none exist yet).

If a source note's TL;DR or body has changed since the bundle was generated, the bundle is stale and must be regenerated. Stale bundles silently mislead any agent that reads them.

**Finding form:** `[BUNDLE] <bundle> — source_trigger does not match any current trigger group in scope AGENTS.md` / `bundle inlines stale TL;DR for [[Folder/Note]]` / `bundles: list missing [[Folder/Note]] which the trigger group now references` / `Source: link does not resolve` / `bundle inlines a note outside its declared scope`.

### 1.9 Changelog and resolved-issue coupling

Verify the `CHANGELOG.md` is consistent with the `Issues/` folder and the git tag history:

- For every issue file in `Issues/` with `status: resolved`, an `Issues Resolved` entry exists in [[CHANGELOG]] (either in `## [Unreleased]` or in a versioned section). The entry must link the issue file path. ([[CONTRIBUTING]] §11.5; [[ISSUES]] §8 step 5.)
- Every released version section in CHANGELOG (`## [0.X.0] — YYYY-MM-DD`) corresponds to an existing git tag (`vO.X.0`). Use `git tag --list` to verify.
- The CHANGELOG's `## [Unreleased]` section is not empty across an extended period without a release; if Unreleased has accumulated more than ~30 entries since the last release, surface this as a warning (signal the project is overdue for a release cut per [[CONTRIBUTING]] §11.3).
- No `## [Unreleased]` entries reference files or issues that have since been deleted.

**Finding form:** `[CHANGELOG] resolved issue <path> has no Issues-Resolved entry in CHANGELOG` / `version section [0.X.0] has no matching git tag vO.X.0` / `Unreleased section has N entries; consider cutting a release` / `Unreleased entry references missing file <path>`.

---

## 2. Semantic checks

These require judgment. Read the full body of each note involved.

### 2.1 Summary–content fit

For each note, compare the `summary:` field against the note body.
- Does the summary still describe what the note now claims?
- Has the body acquired a second thesis the summary doesn't capture?
- Is the summary technically true but uselessly vague?

**Finding form:** `[SUMMARY-DRIFT] <path> — summary says X but note now defends Y` / `summary too vague; body asserts a sharper rule`.

### 2.2 TL;DR–thesis fit

For each note, the italic TL;DR should be the body's thesis in compressed form. Check that body and TL;DR have not diverged.

**Finding form:** `[TLDR-DRIFT] <path> — TL;DR claims X; body actually argues Y`.

### 2.3 Folder fit

For each note, compare its location against the folder taxonomy in [[CONTRIBUTING]] §4. The note belongs in the folder matching its **dominant claim**, not the topic it touches in passing. Apply both axes: scope (Engineering Philosophy / Languages/`<lang>` / Integration) and within-scope folder. A note that genuinely applies to ≥ 2 languages should sit in `Engineering Philosophy/` rather than be duplicated per language scope.

**Finding form:** `[FOLDER-FIT] <path> — currently in <folder>; dominant claim is about <topic>; consider <other-folder>` / `[SCOPE-FIT] <path> — currently in <scope>; lesson appears to apply to N languages; consider lifting to Engineering Philosophy/`.

### 2.4 Single-thesis check

For each note, identify the single claim it defends. If the note defends two distinct theses neither of which is subordinate to the other, it should be split.

**Finding form:** `[SCOPE] <path> — defends two distinct theses (A: ..., B: ...); consider split or absorb one into a sibling note`.

### 2.5 Bidirectional Related links

For each `## Related` link from note A to note B, check that B's `## Related` (or AGENTS.md trigger group) refers back to A. Asymmetric Related links are a smell, not always wrong — judge whether the relationship is genuinely one-way.

**Finding form:** `[RELATED-ASYMMETRY] <A> → <B> but <B> does not link back to <A>; verify intent`.

### 2.6 Trigger-group coverage

For every "When to consult what" section across all AGENTS files in **A**:
- Does each trigger group still contain the *most relevant* notes for that work type, given current vault contents?
- Are there new notes that should be added to existing trigger groups?
- Are there work types the vault now covers but no trigger group represents?
- Is each trigger group in the *correct scope's* AGENTS.md? Universal triggers belong in [[Engineering Philosophy/AGENTS]]; language-conditional triggers belong in the relevant language's AGENTS file. A Rust-only trigger appearing in Engineering Philosophy/AGENTS.md (or vice versa) is a finding.

**Finding form:** `[TRIGGER] group "<name>" in <scope> AGENTS.md — note <X> seems relevant but is not listed` / `vault now covers <work-type> but no trigger group exists in any AGENTS.md` / `trigger "<name>" is in <scope> AGENTS.md but its notes are all from <other-scope>; relocate trigger`.

### 2.7 Style audit (sample-based)

For a random sample of 5 notes, check against [[CONTRIBUTING]] §6:
- Does the note lead with the rule?
- Any throat-clearing preambles ("This note explains…")?
- Code examples earning their space?
- Imperative voice for rules?

**Finding form:** `[STYLE] <path> — opens with throat-clearing preamble` / `code example does not illustrate prose claim`.

---

## 3. Report

Emit one report. Structure:

```
# Vault Audit Report — <date>

## Summary
- Notes audited: N
- Mechanical findings: X (Y critical)
- Semantic findings: Z

## Critical
[Findings that violate a hard rule in CONTRIBUTING — must be fixed.]
[Use the finding-form strings above. One bullet per finding.]

## Warnings
[Findings that suggest drift but require human judgment.]

## Suggestions
[Style, scope, and structure observations that are not violations.]

## Proposed fixes (mechanical only)
[A list of edits the agent could apply autonomously to resolve mechanical findings — frontmatter additions, missing TL;DR lines, broken full-path links. Each fix should reference the finding it addresses.]
```

### Severity rules

- **Critical**: missing required frontmatter; broken wikilink; orphan note; project-noun leak; non-portable syntax; missing/duplicated entry across AGENTS files; root [[AGENTS]] router missing a pointer to a per-scope AGENTS file that exists.
- **Warning**: summary/TL;DR drift; folder-fit or scope-fit concern; scope (multiple theses); trigger-group gap; trigger in wrong scope's AGENTS.md; non-bidirectional Related link.
- **Suggestion**: style observations; sample-based findings; "this could be tighter."

---

## 4. Repair policy

The agent **may apply** mechanical fixes autonomously when the user has asked for "audit and fix" or equivalent:
- Add missing frontmatter keys (with `summary: TODO — needs human review` if the agent cannot infer a sentence).
- Add missing italic TL;DR (with `*TODO — needs human review*` if the agent cannot infer the thesis).
- Convert bare wikilinks in Related sections to full-path form.
- Fix duplicate or missing AGENTS.md index entries.

The agent **must not** apply autonomously:
- Anything in the Warning or Suggestion categories.
- Folder moves.
- Note splits.
- Rewriting summaries or TL;DRs that already exist.
- Removing Related links.
- Edits to CONTRIBUTING.md, AGENTS.md trigger groups, or the body of any note.

If the user has asked for "audit only," apply nothing — emit the report and stop.

---

## 5. Performance notes

For a vault of ≤ 100 notes, read every file in full. Above that, the structural checks (§1) still scan every file, but the semantic checks (§2) may sample 20% of the corpus per category, with priority to notes modified since the last audit (use `git log` or file mtime).

The audit should complete in one pass — do not re-open files multiple times. Cache parsed frontmatter and link graphs in working memory.

---

## Related

- [[CONTRIBUTING]]
- [[AGENTS]]
- [[README]]
