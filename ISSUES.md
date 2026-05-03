---
title: Raising Issues Against Existing Content
tags: [meta, issues, process]
summary: How humans and agents file, track, and resolve issues against existing notes when using the vault surfaces a problem.
---

*Use-time observation → structured issue file → triage → resolution recorded in the target note. Refinements accumulate visibly, not silently.*

# Raising Issues Against Existing Content

A note in the vault is wrong, vague, contradictory, stale, or missing a case you actually hit. This file describes how to file that observation as a tracked issue, how it gets triaged, and how the resolution closes the loop back into the affected note.

This is **not** a contribution process for new content — see [[CONTRIBUTING]] for that. It is **not** a sweep — see [[AUDIT]] for that. It is the channel for *use-time* refinements: someone consulted a note, the note failed them in some way, and the vault should improve as a result.

---

## 1. Scope

File an issue against an existing note when, in the course of real use, you observe one of:

- **Inaccuracy** — the note's claim is factually wrong, or its example doesn't compile / doesn't behave as described.
- **Contradiction** — two notes give incompatible advice on the same situation.
- **Vagueness** — the rule is too soft to act on; it doesn't tell you what to do at the decision point you actually faced.
- **Missed case** — the rule held in general but failed for a situation you encountered; the note needs an exception, qualification, or worked counter-example.
- **Generalization slip** — a project-specific term, library, or assumption has leaked into a note that's supposed to be general.
- **Staleness** — the note references a tool, version, or convention that has moved on.
- **Wrong placement** — the note's dominant claim no longer fits its current folder per [[CONTRIBUTING]] §4.
- **Broken structure** — missing TL;DR, missing Related, broken wikilink, malformed frontmatter, etc. These are usually caught by [[AUDIT]] but may also be filed here if observed in passing.

Do **not** file an issue for:

- "I want to add a new note" → that's [[CONTRIBUTING]].
- "Run a full validation sweep" → that's [[AUDIT]].
- "I disagree with the philosophy" — open a discussion, not an issue. Issues are concrete and actionable; philosophical disagreement needs a different forum.
- Project-specific complaints. The vault is general by design; a note that is general but unhelpful for your specific project is not broken.

---

## 2. Where issues live

All issues live in the `Issues/` folder at the vault root. Each issue is a single markdown file. Issues stay in `Issues/` for their entire lifecycle — status is tracked in frontmatter, not by folder.

### Filename

```
Issues/YYYY-MM-DD-short-kebab-slug.md
```

- `YYYY-MM-DD` is the date filed.
- `short-kebab-slug` is 3–6 words describing the issue, lowercase, hyphen-separated.
- Example: `Issues/2026-04-26-sharp-oracles-missing-property-test-case.md`

If two issues collide on the same date and slug, append `-2`, `-3`, etc.

---

## 3. Issue file shape

Required frontmatter:

```yaml
---
id: <YYYY-MM-DD-slug>           # matches the filename without .md
status: open                     # open | triaged | in-progress | resolved | rejected | duplicate
target: [Folder/Note Name]       # one or more affected notes (wikilink-style, no brackets, comma-separated)
type: inaccuracy                 # one of the categories from §1, lowercased and hyphenated
severity: low                    # low | medium | high — see §3.1
raised_by: human | agent         # who filed it
raised_at: 2026-04-26
resolved_at:                     # filled in at close; empty otherwise
resolution_commit:               # short SHA or change reference; empty until resolved
---
```

Required body sections, in order:

```markdown
*One italic line summarizing the issue.*

# <Title>

## Observation
What you saw. Be specific. Quote the exact passage from the note if relevant.

## Where surfaced
What you were doing when this came up. The real-use context — task, decision, code under review. This is what makes the issue valuable; it explains why the abstract rule failed in practice.

## Proposed change
What should the note say or do instead? If you don't know, say "open — needs triage." Concrete proposals are easier to triage than complaints.

## Counter-considerations
Reasons the note might be right as-is, or reasons your proposed change has tradeoffs. Steelman the existing content. If you can't think of any, write "none considered."

## Resolution
[Filled in at close. Empty until then.]
```

### 3.1 Severity rubric

- **High** — the note actively misleads someone following it; following the rule produces a worse outcome than not consulting the vault. Fix urgent.
- **Medium** — the note is correct but unhelpful at the decision point that surfaced the issue; readers would benefit from a refinement.
- **Low** — cosmetic, structural, or generalization slips with no functional impact on advice quality.

Severity is the filer's *initial* judgment. Triage may revise it.

---

## 4. Lifecycle

```
open → triaged → in-progress → resolved
                              ↘ rejected
                              ↘ duplicate (link to canonical issue)
```

- **open** — filed; not yet read by a triager.
- **triaged** — read, severity confirmed or revised, accepted as actionable. May still be unscheduled.
- **in-progress** — someone is editing the target note.
- **resolved** — target note edited; `Resolution` section filled in; commit/change referenced.
- **rejected** — declined with reasoning recorded in `Resolution`. Common reasons: out of scope, philosophical disagreement, "not broken." A rejection is still informative and stays in the folder.
- **duplicate** — the canonical issue is referenced in `Resolution`. Both stay on disk; only the canonical one progresses.

Status transitions are recorded as a one-line note appended to a `## History` section *only if useful* — most issues will move directly from `open` to `resolved` and don't need an audit trail beyond the timestamps.

---

## 5. Filing flow — humans

1. Confirm the issue is in scope (§1).
2. Search `Issues/` for an existing open issue covering the same observation. If found, add a comment to that issue rather than filing a duplicate. (See §7.)
3. Copy the template from §9 into a new file under `Issues/` with the conventional filename.
4. Fill in frontmatter and required sections. Be specific in `Observation` and `Where surfaced` — these are what make the issue triageable.
5. Save. The issue is now `open`.

You do not need to fix the note when filing the issue. Filing and fixing can be the same person or different people.

---

## 6. Filing flow — agents

When an agent (during normal use of the vault, not during an audit run) observes one of the conditions in §1:

1. **Do not silently work around the issue.** A vague rule the agent privately reinterprets is drift compounded.
2. **Do not edit the target note autonomously.** Edits to existing notes go through triage.
3. **File an issue file** following exactly the same structure as the human flow. Set `raised_by: agent`.
4. In `Where surfaced`, include the user task and prompt context that surfaced the issue. This is the agent's most useful contribution — humans don't always know in which conversations a given note failed.
5. **Continue the user's task** using the agent's best judgment and a brief note in the user-visible response that an issue has been filed (with the issue path).

If the issue is `severity: high` and following the note as written would actively harm the user's work, the agent should both file the issue *and* warn the user inline before proceeding with a workaround.

Agents must not file issues during audit runs — those findings belong in the audit report, not as new files in `Issues/`. Audits may, however, recommend converting specific findings into tracked issues.

---

## 7. Triage

Triage happens when a human reviews open issues. Cadence is at the user's discretion (weekly, on-demand, before adding a related note). The triager:

1. Reads the issue.
2. Confirms or revises `severity`.
3. Sets `status: triaged` if the issue is accepted as actionable, `rejected` (with reasoning) if not, `duplicate` (linking the canonical) if redundant.
4. Optionally adds a `## Triage notes` section if the path forward is non-obvious.

Multiple comments / discussion on a single issue go into a `## Discussion` section, dated and attributed:

```markdown
## Discussion

**2026-04-28 — agent:** Noted that the proposed change conflicts with [[Other Note]] on point X. Suggest narrowing to case Y.

**2026-04-28 — human:** Agreed. Resolution should add a stated exception, not change the headline rule.
```

---

## 8. Resolution

When the target note is edited to address the issue:

1. Make the edit per [[CONTRIBUTING]].
2. Run the relevant subset of [[AUDIT]] §1 mechanical checks on the edited note (link resolution, frontmatter, MOC linkage if scope changed).
3. Update the issue file:
   - `status: resolved`
   - `resolved_at: <date>`
   - `resolution_commit: <short-sha-or-change-ref>`
   - Fill in `## Resolution` with: what changed, in which note(s), and how the change addresses the observation. One short paragraph is plenty.
4. If the resolution materially changed the target note's thesis, also confirm the three consequences from [[CONTRIBUTING]] §7 (the scope's MOC entry, the scope's AGENTS.md trigger group fit, the scope's AGENTS.md full index).
5. Add an entry to the `Issues Resolved` category in the `## [Unreleased]` section of [[CHANGELOG]]. Include the issue file path as a wikilink and a one-line description of what changed in which note. This step is non-optional — see [[CONTRIBUTING]] §11.5.

Resolved issues stay in `Issues/`. They are the change log of the vault's refinement under real use, and reading old resolutions is a useful way to understand why a note is shaped the way it is. The `CHANGELOG` is the public-facing summary; `Issues/` is the long-form record.

---

## 9. Template

Copy this skeleton into `Issues/YYYY-MM-DD-short-slug.md`.

```markdown
---
id: YYYY-MM-DD-short-slug
status: open
target: Folder/Note Name
type: inaccuracy
severity: medium
raised_by: human
raised_at: YYYY-MM-DD
resolved_at:
resolution_commit:
---

*One italic line summarizing the issue.*

# Short descriptive title

## Observation
What you saw. Quote the relevant passage if applicable.

## Where surfaced
The task, decision, or context in which the note failed. Why this matters.

## Proposed change
Concrete suggestion, or "open — needs triage" if you don't know yet.

## Counter-considerations
Reasons the note may be right as-is, or tradeoffs of the proposed change. "None considered" is acceptable.

## Resolution
[Empty until closed.]
```

---

## Related

- [[CONTRIBUTING]]
- [[AUDIT]]
- [[AGENTS]]
