# Agent Mob — Plugin Development Guidelines

This is the `claude-mob` plugin repository. It is **not** a mob workspace. The mob workspace rulebook lives at `plugins/agent-mob/templates/AGENTS.md` — that file is installed into teams' repos by `/mob init`.

Read this file before making any changes to the plugin source. Follow these guidelines strictly.

---

## Purpose of This Repo

This repo contains the installable Claude Code plugin for Agent Mob. It is a plugin source repo, not a place to run QRSPI projects. There are no `active/`, `paused/`, or `archived/` branches here, and no `PROJECT.yml` or task artifacts should ever be created here.

---

## Repo Structure

```
plugins/
  agent-mob/
    agents/
      mob-agent.md          ← orchestrator (lifecycle commands)
      mob-researcher.md     ← R-phase research specialist
      mob-designer.md       ← D-phase design synthesizer
    skills/
      mob/
        SKILL.md            ← inline fast-path skill
    templates/
      AGENTS.md             ← workspace rulebook (installed by /mob init)
      PROJECT.yml           ← project manifest schema
      project-CLAUDE.md     ← project branch CLAUDE.md template
    .claude-plugin/
      plugin.json           ← plugin manifest
docs/
  brainstorms/              ← design explorations and requirements docs
  plans/                    ← implementation plans
  SYSTEM_CHANGELOG.md       ← log of plugin-level changes
AGENTS.md                   ← this file (plugin dev guidelines)
CLAUDE.md                   ← quick reference for contributors
```

---

## Modifying Agents

Agent files are in `plugins/agent-mob/agents/`. Each agent has a YAML frontmatter block and a markdown body.

- **frontmatter `description`** — the phrase-match trigger used to dispatch this agent. Keep it accurate.
- **Identity section** — defines the agent's role and the startup verification check. Workspace detection uses `AGENTS.md` presence only — do not add `.claude-plugin/plugin.json` or any other file to this check.
- **Prohibitions** — rules the agent enforces. The allowed-on-main list must not include `.claude-plugin/` — that path is not part of a mob workspace.

---

## Modifying the Skill

The inline fast-path skill is at `plugins/agent-mob/skills/mob/SKILL.md`.

- Workspace verification uses `AGENTS.md` only — no `plugin.json` check.
- `/mob init` is the bootstrap subcommand. It copies `${CLAUDE_PLUGIN_ROOT}/templates/AGENTS.md` into the target repo. Keep this path — `CLAUDE_PLUGIN_ROOT` resolves to the plugin root at runtime.
- The "Uninitialized workspace" section defines the soft-fail path when `AGENTS.md` is absent. Do not change this to a hard-fail.

---

## Modifying Templates

Templates are in `plugins/agent-mob/templates/`. These files are copied into users' mob workspaces.

- `AGENTS.md` — the workspace rulebook. This is what teams read. Keep it accurate and self-contained. The allowed-on-main list here must not include `.claude-plugin/`.
- `PROJECT.yml` — schema only; no real data.
- `project-CLAUDE.md` — template with `{{substitution}}` placeholders.

---

## Commit Conventions

All commits in this repo use:

```
[mob] {action}: {description}
```

Valid action values: `init`, `new-project`, `new-task`, `artifact`, `advance`, `push`, `pause`, `archive`, `add-member`, `link-linear`, `update-system`

Use `update-system` for changes to plugin source files (agents, skills, templates, manifest).

Every `update-system` commit must have a corresponding entry in `docs/SYSTEM_CHANGELOG.md`.

Every `update-system` commit must also bump the patch version in `plugins/agent-mob/.claude-plugin/plugin.json`. The changelog entry and version bump are atomic — they ship in the same commit.

---

## What NOT to Do Here

- Do not create `active/`, `paused/`, or `archived/` branches — this is not a mob workspace
- Do not create `PROJECT.yml` or task artifacts on any branch
- Do not run `/mob new-project` or `/mob new-task` in this repo
- Do not modify `plugins/agent-mob/.claude-plugin/plugin.json` without updating the manifest version
- Do not add `.claude-plugin/plugin.json` back to workspace detection checks — that was a previous design that mixed plugin and workspace concerns
