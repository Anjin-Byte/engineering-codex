<!--
Read CONTRIBUTING.md §10 (GitHub workflow) before filling this in.
Each PR fits one lane (Add / Refine / Audit / Meta). Mark it below.
-->

## Lane

<!-- Pick one. Delete the others. -->

- [ ] **Add** — new content (note, bundle, meta-file). Governed by CONTRIBUTING.md §1–§9.
- [ ] **Refine** — resolves a tracked issue in `Issues/`. Governed by ISSUES.md §8.
- [ ] **Audit** — applies findings from a vault audit. Governed by AUDIT.md §4 repair policy.
- [ ] **Meta** — touches CONTRIBUTING / any AGENTS file (root router or per-scope) / README / AUDIT / ISSUES / CHANGELOG.

## What changed

<!-- File-level granularity. -->

-

## Why

<!--
The lesson, observation, or finding driving the change.
For Refine PRs, link the resolved issue file: `Resolves Issues/YYYY-MM-DD-slug.md`.
-->

## Index updates

<!-- Confirm each consequence applicable to this PR. CONTRIBUTING.md §7. -->

- [ ] Scope MOC updated (Engineering Philosophy / Languages/Rust/Practices / Languages/Rust/Workspace / future per-language MOC) — N/A if no notes added/moved.
- [ ] Relevant scope's AGENTS.md "When to consult what" trigger groups — updated or N/A. (The root [[AGENTS]] router rarely needs touching; per-scope AGENTS files do.)
- [ ] Relevant scope's AGENTS.md "Full note index" — updated or N/A. Each note appears in exactly one scope's index.
- [ ] Bundles refreshed if a constituent note's TL;DR or body changed — N/A if not applicable. Bundles are scoped per language (`Bundles/Universal/`, `Bundles/Rust/`).

## Audit checks

<!--
For non-trivial PRs, state which subset of AUDIT.md §1 mechanical checks was run.
Single-note body edits don't need the full audit.
-->

- [ ] Frontmatter completeness (§1.1).
- [ ] TL;DR line present (§1.2).
- [ ] Wikilinks resolve (§1.3).
- [ ] No orphans introduced (§1.4).
- [ ] No project nouns (§1.5).
- [ ] No Obsidian-only syntax (§1.6).
- [ ] All AGENTS files structurally intact (§1.7) — root router and per-scope.
- [ ] Bundle freshness verified, if bundles were touched (§1.8).

## Changelog

<!-- REQUIRED. PR will not merge without a CHANGELOG entry. -->

- [ ] Entry added to `## [Unreleased]` in CHANGELOG.md under the appropriate category (Added / Changed / Deprecated / Removed / Fixed / Issues Resolved).

## Breaking changes

<!--
List any: note rename/move/delete, trigger-group rename, frontmatter schema change,
material rule reversal. Pre-1.0, breaking changes in a MINOR release must be called out here.
-->

- None.
