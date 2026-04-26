# Changelog

All notable changes to the Engineering Codex are recorded in this file.

The format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/) with one extension: an **Issues Resolved** category that links to resolved issue files in `Issues/`. Versioning follows the policy in [CONTRIBUTING.md §11](./CONTRIBUTING.md#11-versioning-and-the-changelog).

> **Pre-1.0:** the vault is currently in the `v0.x` series. Until `v1.0`, MINOR releases may include breaking changes (renamed/moved notes, trigger-group renames) — projects pinning to a specific version are recommended during this period.

---

## [Unreleased]

### Added
-

### Changed
- [CONTRIBUTING.md](./CONTRIBUTING.md) §6 makes the no-contributor-disclosure rule explicit; §10.2 bans `Co-Authored-By:` trailers and similar commit metadata; §11.3 clarifies that changelog entries describe *what changed*, never *who changed it*.

### Removed
-

### Fixed
-

### Issues Resolved
-

---

## [0.1.0] — 2026-04-26

Initial release. The vault contains 46 content notes across three sections (Engineering Philosophy, Rust Practices, Architecture), six pre-bundled task contexts, and a complete set of meta-files governing addition, refinement, and audit.

### Added
- 8 Engineering Philosophy principles (Sharp Oracles, Tests Should Make Programs Fail, Reason About Correctness, Reasoning Beside Code, Normal Usage Is Not Testing, Regression Discipline, Explicit Not Magical, Human Understanding First) plus the Output Format note and section MOC.
- 23 Rust Practices notes across Foundational, Type-Driven Design, Effect Isolation, Error Handling, Ownership and Mutation, Functions and Data, Testing, and Tooling and Quality, plus the Practices Checklist and section MOC.
- 11 Architecture notes covering Workspace Layout, Cargo Workspace Configuration, Core Principles, six cross-cutting Patterns (Headless First, Feature Gating, Binary Strategy, Shared Dependencies, Workspace Lints and Profiles, CPU Reference Path), the Patterns Index, and section MOC.
- 6 task bundles in `Bundles/` (Code-Review, Test-Design, Module-Design, Error-Handling, API-Design, Written-Analysis) inlining each trigger group's notes for single-read agent consumption.
- Frontmatter standards on every note: `title`, `tags`, `summary`, `keywords` (4–8 retrieval terms per note), italic TL;DR line preceding the body.
- Vault-root meta-files governing process: [README.md](./README.md), [AGENTS.md](./AGENTS.md) (agent runbook with 16 trigger groups + full note index), [CLAUDE.md](./CLAUDE.md) (Claude Code entry pointer), [CONTRIBUTING.md](./CONTRIBUTING.md) (note authoring standards + GitHub workflow + versioning policy), [ISSUES.md](./ISSUES.md) (use-time refinement process for existing notes), [AUDIT.md](./AUDIT.md) (drift-check runbook).
- `consumer-template/` with drop-in CLAUDE.md snippet and README covering three integration paths (vendored, submodule, symlink) for consuming projects.
- `Issues/` folder structure (lifecycle: open → triaged → in-progress → resolved/rejected/duplicate; status tracked in frontmatter).

### Notes

This initial release was assembled by generalizing lessons drawn from prior project work. All domain-specific nouns were stripped during the port; the resulting principles and Rust idioms travel to any project.

---

[Unreleased]: ./
[0.1.0]: ./
