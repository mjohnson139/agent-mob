---
title: "refactor: Separate Plugin from Workspace"
date: 2026-05-25
status: delivered
type: requirements
---

# refactor: Separate Plugin from Workspace

## Problem

The `claude-mob` repo currently serves two roles simultaneously:

1. **Plugin source** — `plugins/agent-mob/` contains the installable Claude Code plugin (agents, skills, templates, manifest)
2. **Mob workspace** — the repo root contains `AGENTS.md`, `CLAUDE.md`, and project branches where QRSPI work lives

This mixing causes two concrete problems:

- **Wrong workspace detection** — the mob skill verifies it is in a mob workspace by checking for `AGENTS.md` + `.claude-plugin/plugin.json`. The plugin manifest lives inside the plugin installation, not in a user's mob workspace. Any team using the plugin in their own repo would fail this check.
- **No init path** — there is no command to bootstrap a new mob workspace, so the only way to get started is to manually copy files.

## Goal

Make the workspace (the git repo where a team runs QRSPI projects) a distinct thing from the plugin. A mob workspace is any git repo initialized with `/mob init`. The plugin directory structure (`plugins/agent-mob/`) stays as-is — it is working and leaves room for future additions.

---

## Success Criteria

1. A team can initialize any git repo as a mob workspace with one command: `/mob init`.
2. The mob skill detects an uninitialized workspace (no `AGENTS.md`) and offers to run init rather than hard-failing.
3. Workspace detection uses only `AGENTS.md` presence — not `.claude-plugin/plugin.json`.
4. The root `AGENTS.md` in `claude-mob` is updated to reflect its role as plugin development guidelines, not a mob workspace file.

---

## Decisions Made

| Decision | Rationale |
|---|---|
| Keep `plugins/agent-mob/` subdirectory as-is | It works; collapsing to root would be YAGNI — the subdirectory leaves room for future plugins |
| `/mob init` copies `templates/AGENTS.md` into workspace | Templates already exist — init is just delivery; no new content to author |
| Workspace identified by `AGENTS.md` alone | Simple and unambiguous — it's exactly what `/mob init` writes |
| Detect uninitialized workspace and offer to init | Better UX than a hard failure; user doesn't need to know the init command exists |

---

## Scope

### In scope
- Add `/mob init` subcommand to the mob skill
- Fix workspace detection in the mob skill and all three agents — remove `.claude-plugin/plugin.json` check
- Add "not initialized" detection with offer to run init inline
- Update root `AGENTS.md` to reflect plugin dev guidelines (not mob workspace rules)
- Update root `CLAUDE.md` to reflect the actual plugin structure and correct command names

### Out of scope
- Restructuring `plugins/agent-mob/` or moving files to the repo root
- Changing marketplace.json or plugin.json
- Changing QRSPI phase logic
- The U6/U7 QRSPI walkthrough from the MVP plan
- `/mob pause`, `/mob archive` lifecycle commands

---

## Requirements

### R1 — `/mob init` subcommand

Add `init` as a subcommand handled inline by the mob skill (alongside `new-project`, `new-task`, etc.).

**Behavior:**
1. Verify the current directory is a git repo (`git rev-parse --git-dir`). If not, stop: "This is not a git repo. Initialize one with `git init` first."
2. Check if `AGENTS.md` already exists — if so, say: "This repo is already initialized as a mob workspace." and stop.
3. Copy `templates/AGENTS.md` (from `${CLAUDE_PLUGIN_ROOT}/plugins/agent-mob/templates/AGENTS.md`) into the workspace root as `AGENTS.md`.
4. Stage and commit: `[mob] init: initialize mob workspace`
5. Output: "Mob workspace initialized. Run `/mob new-project \"Name\"` to create your first project."

### R2 — Workspace detection fix

Update the mob skill and all three agents (`mob-agent`, `mob-researcher`, `mob-designer`) to remove the `.claude-plugin/plugin.json` check.

**Before (in mob skill and agents):**
```bash
ls AGENTS.md .claude-plugin/plugin.json
```

**After:**
```bash
ls AGENTS.md
```

`AGENTS.md` presence is the single signal that a repo is a mob workspace.

### R3 — Uninitialized workspace detection with init offer

When the mob skill or mob-agent is invoked in a directory without `AGENTS.md`, instead of a hard failure, offer to initialize inline:

```
This doesn't look like a mob workspace yet — no AGENTS.md found.

Would you like to initialize this repo as a mob workspace?
This will copy the mob rulebook (AGENTS.md) into the current directory and commit it.

Reply "yes" to initialize now.
```

If the user confirms, run the R1 init steps inline and continue with the original command if applicable (e.g., if they ran `/mob new-project` and confirmed init, proceed to create the project after init completes).

### R4 — Update root AGENTS.md

The `AGENTS.md` at the `claude-mob` repo root currently reads as a mob workspace rulebook. Update it to reflect its actual role: guidelines for developing and maintaining the agent-mob plugin itself. The mob workspace rulebook lives at `plugins/agent-mob/templates/AGENTS.md` — that is the canonical source installed into workspaces.

---

## Non-Goals

- `/mob init` does not create `PROJECT.yml` — that is created by `/mob new-project`
- `/mob init` does not push to remote — the team decides when to push
- The plugin repo itself does not need to be a mob workspace

---

### R5 — Update root `CLAUDE.md`

The root `CLAUDE.md` currently shows an incorrect plugin structure (paths at repo root instead of `plugins/agent-mob/`) and uses stale command names (`/mob-new-project`, `/mob-status`, `/mob-fork` instead of `/mob new-project`, `/mob status`, `/mob fork`).

Update to reflect:
- Correct plugin structure under `plugins/agent-mob/`
- Correct command syntax (`/mob <subcommand>`)
- Remove workspace-oriented sections that don't apply to the plugin dev repo (branch layout for `active/`, QRSPI phase flow)
- Keep: plugin install command, plugin structure diagram, commit format, quick reference for plugin development commands

---

## Open Questions

- Should `/mob init` auto-commit, or leave `AGENTS.md` unstaged for the user to review first? (Current spec: auto-commit, consistent with how `new-project` and `new-task` work.)
