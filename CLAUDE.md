# Agent Mob

**Read `AGENTS.md` before taking any action in this repository.**

Agent Mob is a Claude Code plugin that brings QRSPI-structured collaboration to any git repo. This repo is the plugin source — it contains agents, skills, and templates. It is not itself a mob workspace.

---

## Quick Reference

### Install this plugin in another Claude Code project

```bash
claude plugin install git@github.com:mjohnson139/agent-mob.git
```

### Initialize a new mob workspace

Run this in any git repo after installing the plugin:

```
/mob init
```

### Start a new project

```
/mob new-project "Project Name"
```

### Add a participant to a project

```
/mob add-member {github-id}
```

### Check current phase of active task

```
/mob status
```

### Fork and start R-phase research

```
/mob fork
```

---

## Plugin Structure

```
plugins/agent-mob/
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
  plugin.json             ← plugin manifest
```

---

## Commit Format

All commits use:

```
[mob] {action}: {description}
```

Valid action values: `init`, `new-project`, `new-task`, `artifact`, `advance`, `push`, `pause`, `archive`, `add-member`, `link-linear`, `update-system`

---

## Making Changes

See `AGENTS.md` for full plugin development guidelines — which files to edit, what to avoid, and the changelog requirement for `update-system` commits.
