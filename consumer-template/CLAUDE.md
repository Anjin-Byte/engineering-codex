# CLAUDE.md — Project Snippet

> Copy or append the section below into your project's own `CLAUDE.md`. Replace the path placeholder with the actual location of the Engineering Codex on your machine or in your repo.

---

## Engineering Codex

This project follows the engineering standards in the Engineering Codex at:

```
<PATH-TO-VAULT>/Engineering_Codex/
```

Common values:
- `~/Documents/dev_hub/Brynhild/Engineering_Codex/` (local absolute path)
- `./engineering-codex/` (vendored copy or git submodule, relative to repo root)
- `./.engineering-codex/` (hidden vendored copy)

### How agents should use it

1. **Read `<PATH-TO-VAULT>/AGENTS.md` first** for any task that overlaps engineering practice — code review, test design, module/crate design, error handling, API design, refactoring, written analysis, CI/tooling. The file's "When to consult what" section maps task triggers to the relevant notes.
2. **For high-frequency tasks, prefer the bundle** at `<PATH-TO-VAULT>/Bundles/<Task>.md` — these inline all relevant notes into one file and save round-trips.
3. **Treat the vault's rules as authoritative** by default. Deviations are allowed, but justify them explicitly when reviewing or writing code.
4. **Do not edit notes in the vault.** If you observe a problem with an existing note during this project's work, file an issue per `<PATH-TO-VAULT>/ISSUES.md` rather than silently working around it or autonomously editing.
5. **Do not add new notes during project work.** New general-knowledge notes go through the contribution process at `<PATH-TO-VAULT>/CONTRIBUTING.md`, separate from project tasks.

### Project-specific notes

[Keep your project-specific guidance below this line. The vault is general; this section is where project domain, conventions, and constraints live.]
