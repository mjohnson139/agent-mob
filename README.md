# Agent Mob

A Claude Code plugin that brings **QRSPI-structured collaboration** to any git repo. Small engineering teams use it to coordinate research, design, and implementation across multiple contributors — with AI assistance baked in at every phase.

---

## What is QRSPI?

QRSPI is a structured workflow for collaborative software work, adapted from Dexter Horthy's [seven-step pipeline](https://www.youtube.com/watch?v=YwZR6tc7qYg). Agent Mob distills it into five participant-facing phases with an internal lead-only Design step between R and S:

| Phase | Name | What happens |
|---|---|---|
| **Q** | Questions | The lead lists unknowns and gaps, writes open questions |
| **R** | Research | Participants independently document what exists (no suggestions) |
| *(D)* | *Design* | *Lead only — synthesizes research into a ~200-line design doc* |
| **S** | Structure | Participants outline phases and validation steps |
| **P** | Plan | The lead synthesizes into actionable, task-based instructions |
| **I** | Implement | Work gets built in small, manageable chunks |

Each phase produces a markdown artifact committed to a project branch. The file tree is the state machine.

---

## Installation

Run this inside any Claude Code project:

```bash
claude plugin install git@github.com:mjohnson139/agent-mob.git
```

Then initialize a mob workspace in that repo:

```
/mob init
```

This copies the workspace rulebook (`AGENTS.md`) into your repo and sets up the branch structure.

---

## Quick Start

```
# Start a new project
/mob new-project "Project Name"

# Add a participant
/mob add-member {github-id}

# Check current phase
/mob status

# Join as a participant and start R-phase research
/mob join

# Contribute completed work
/mob contribute
```

---

## How It Works

- **Projects live on branches** — `active/{slug}`, `paused/{slug}`, `archived/{slug}`
- **`main` is system-only** — no project artifacts ever land on `main`
- **Phases are file-driven** — the presence of `Q.md`, `R-{user}.md`, `design.md`, etc. determines what phase a project is in
- **AI agents handle orchestration** — mob-agent drives the lifecycle; mob-researcher guides R-phase work; mob-designer synthesizes research into design

---

## This Repo

This repo is the **plugin source**. It is not itself a mob workspace. If you want to run a QRSPI project, install the plugin into another repo and run `/mob init` there.

Plugin source lives under `plugins/agent-mob/`. The workspace rulebook (installed into teams' repos) is at `plugins/agent-mob/templates/AGENTS.md`.

---

## Plugin Structure

```
plugins/agent-mob/
  .claude-plugin/
    plugin.json             ← plugin manifest and version
  agents/
    mob-agent.md            ← orchestrator (lifecycle commands)
    mob-researcher.md       ← R-phase research specialist
    mob-designer.md         ← D-phase design synthesizer
  skills/
    mob/
      SKILL.md              ← inline fast-path skill
  templates/
    AGENTS.md               ← workspace rulebook (installed by /mob init)
    PROJECT.yml             ← project manifest schema
    project-CLAUDE.md       ← project branch CLAUDE.md template
docs/
  brainstorms/              ← design explorations and requirements docs
  plans/                    ← implementation plans
  SYSTEM_CHANGELOG.md       ← log of plugin-level changes
AGENTS.md                   ← plugin development guidelines
CLAUDE.md                   ← quick reference for contributors
```

---

## Development

See `AGENTS.md` for full plugin development guidelines — which files to edit, commit conventions, and the changelog + version bump requirement for every `update-system` commit.

All commits use:

```
[mob] {action}: {description}
```

Valid actions: `init`, `new-project`, `new-task`, `artifact`, `advance`, `push`, `pause`, `archive`, `add-member`, `link-linear`, `update-system`

---

## License

MIT
