---
title: Wiring the Codex into a Project
tags: [meta, integration, consumer]
summary: Three integration paths (vendored, submodule, symlink) for pointing a project's AI tools at the Engineering Codex.
---

*Pick an integration path, drop the snippet into your project's CLAUDE.md, and your project's agents will route engineering questions through the vault.*

# Wiring the Codex into a Project

This folder contains a drop-in template for projects that want their AI coding tools (Claude Code, OpenAI Codex, etc.) to consult the Engineering Codex on engineering tasks.

The vault is general — it makes no project-specific assumptions. Each consuming project decides how the vault is co-located and how its own agents reach it.

## What you're wiring

Two things:
1. The vault location must be reachable from the project (filesystem path).
2. The project's own `CLAUDE.md` (or equivalent) must point agents at the vault's `AGENTS.md`.

## Integration paths

Pick one based on how tightly the project wants to couple to a specific vault version.

### Option A — Vendored copy

```bash
cp -R /path/to/Engineering_Codex ./engineering-codex
git add engineering-codex && git commit -m "vendor engineering codex"
```

**When to use:** small projects, single contributor, or projects that want a frozen snapshot. No coordination with vault updates.
**Tradeoff:** drift from upstream is silent. You'll need to re-vendor periodically. Issues observed during project work cannot easily flow back to the canonical vault unless you maintain it separately.

### Option B — Git submodule

```bash
git submodule add <vault-repo-url> engineering-codex
git submodule update --init --recursive
```

**When to use:** the vault is in version control and the project wants a pinned but updatable reference.
**Tradeoff:** every clone needs `--recursive` or a follow-up `submodule update`. Some teams find submodules awkward.

### Option C — Symlink

```bash
ln -s ~/Documents/dev_hub/Brynhild/Engineering_Codex ./engineering-codex
echo engineering-codex >> .gitignore
```

**When to use:** local dev only, single machine, fast iteration on the vault while using it from a project.
**Tradeoff:** the symlink is per-machine, not portable. Don't commit it. Other contributors must replicate the layout.

## Wiring the project's `CLAUDE.md`

Copy or append the snippet from [`CLAUDE.md`](./CLAUDE.md) into your project's `CLAUDE.md` at the repo root. Replace `<PATH-TO-VAULT>` with the actual path:

- Vendored copy / submodule: `./engineering-codex`
- Symlink: `./engineering-codex`
- External absolute path: `~/Documents/dev_hub/Brynhild/Engineering_Codex` (note: not portable across machines)

If your project already has a `CLAUDE.md`, append the "Engineering Codex" section. The snippet is structured so the project's own conventions go *after* the vault wiring — the vault is general; the project section below it is specific.

## Path-resolution gotcha

Wikilinks inside the vault (`[[Folder/Note]]`) resolve relative to the vault root. An agent reading a note from your project must understand that wikilinks are vault-internal, not project-relative. The vault's `AGENTS.md` already explains this; the consumer-side snippet just makes the path explicit so the agent knows where to start.

If you symlink, agents may resolve through the symlink without issue on most platforms. If you vendor or submodule, the path is just a subdirectory.

## What goes in your project, not the vault

Project domain knowledge, library choices, deploy conventions, business logic — none of this belongs in the vault. The vault is for lessons that travel. Keep project-specific knowledge in your project's own `CLAUDE.md`, README, or `docs/` folder.

If during project work an agent observes that a vault note is wrong, vague, or missing a case, the path is to file an issue per `<PATH-TO-VAULT>/ISSUES.md` — not to patch the vault inline as part of the project task.
