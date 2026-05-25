# Agent Mob

**Read `AGENTS.md` before taking any action in this repository.**

Agent Mob is a git-backed collaboration system that coordinates QRSPI-structured work across multi-repo engineering teams. This `main` branch is the system layer — team roster, rules, and documentation only.

---

## Quick Reference

### You are on `main`
This branch is system-only. To work on a project, switch to a project branch:
```bash
git checkout active/{slug}
```

### Start a new project
```
/mob-new-project "Project Name"
```
Creates `active/{slug}` branch with `PROJECT.yml` and project `CLAUDE.md`.

### Check current team
See `team.yml` in this directory.

### Add a team member
```
/mob-add-member {github-id}
```
Adds the member to `team.yml` and commits to `main`.

### Initialize a new mob repo from scratch
```
/mob-init
```

### Install this plugin in another Claude Code project
```bash
claude plugin install git@github.com:mjohnson139/agent-mob.git
```

### Check current phase of active task
```
/mob-status
```

### Fork and start R-phase research
```
/mob-fork
```

---

## Plugin Structure

The Claude Code plugin is co-located in this repo:

```
.claude-plugin/plugin.json    ← plugin manifest
agents/
  mob-agent.md               ← orchestrator (lifecycle commands)
  mob-researcher.md          ← R-phase research specialist
  mob-designer.md            ← D-phase design synthesizer
templates/
  AGENTS.md                  ← system rules template (for new mob repos)
  PROJECT.yml                ← project manifest schema
  project-CLAUDE.md          ← project branch CLAUDE.md template
```

---

## Current Team

See `team.yml` — GitHub IDs are the canonical identifiers.

---

## Branch Layout

```
main/              ← you are here (system only)
active/{slug}/     ← live projects
paused/{slug}/     ← paused projects
archived/{slug}/   ← completed projects
```

Projects are never merged to `main`. Branch lifecycle is: `active/` → `paused/` → `archived/`.

---

## Commit Format

All commits use:
```
[mob] {action}: {description}
```

---

## QRSPI Phase Flow

```
Q (lead)  →  R (all participants, parallel)  →  D (lead)  →  S (each)  →  P (each)  →  implement
```

Phase state is derived from which files exist in the task directory — there is no separate status file. Read `AGENTS.md` for the full phase determination rules.
