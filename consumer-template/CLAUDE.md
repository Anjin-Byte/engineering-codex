---
title: Consumer-Template CLAUDE Snippet
tags: [meta, integration, snippet]
summary: Drop-in CLAUDE.md snippet that consuming projects paste into their own root CLAUDE.md to point agents at this vault.
---

*This file is a snippet, not a vault note. The body below the horizontal rule is meant to be copied verbatim into a consuming project's own CLAUDE.md.*

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

1. **Read `<PATH-TO-VAULT>/AGENTS.md` first** — it is a router. It detects scope and points at the relevant per-scope AGENTS files. For any engineering task: always read `<PATH-TO-VAULT>/Engineering Philosophy/AGENTS.md`. If this project uses Rust, also read `<PATH-TO-VAULT>/Languages/Rust/AGENTS.md`. If the project crosses Rust↔WASM↔TS boundaries, also read `<PATH-TO-VAULT>/Integration/Rust-WASM-TS/AGENTS.md`. Each per-scope AGENTS file's "When to consult what" section maps task triggers to the relevant notes.
2. **For high-frequency tasks, prefer the bundle** at `<PATH-TO-VAULT>/Bundles/<scope>/<Task>.md` (e.g. `Bundles/Universal/Written-Analysis.md`, `Bundles/Rust/Code-Review.md`) — these inline all relevant notes into one file and save round-trips. Bundles are scoped per language so loading one never costs cross-scope context.
3. **Treat the vault's rules as authoritative** by default. Deviations are allowed, but justify them explicitly when reviewing or writing code.
4. **Do not edit notes in the vault.** If you observe a problem with an existing note during this project's work, file an issue per `<PATH-TO-VAULT>/ISSUES.md` rather than silently working around it or autonomously editing.
5. **Do not add new notes during project work.** New general-knowledge notes go through the contribution process at `<PATH-TO-VAULT>/CONTRIBUTING.md`, separate from project tasks.

### Project-specific notes

[Keep your project-specific guidance below this line. The vault is general; this section is where project domain, conventions, and constraints live.]
