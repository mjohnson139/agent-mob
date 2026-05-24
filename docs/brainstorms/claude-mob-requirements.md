# Claude Mob — Requirements Document

**Status:** Draft  
**Date:** 2026-05-23  
**Author:** mjohnson139

---

## Overview

Claude Mob is two things:

1. **A Claude Code plugin** — installable, provides `/mob-*` commands, embeds the AGENTS.md and CLAUDE.md templates that govern a mob repo
2. **A git repo** (one per team) — pure data: team roster, project branches, QRSPI phase artifacts

The plugin owns behavior. The repo owns state. Teams install the plugin once; thereafter the repo is self-governing.

The coordination primitive is the **QRSPI phase artifact** — markdown files committed by each participant at each phase. The file tree is the state machine: phase and contributor progress are derived by reading what files exist, not tracked in a separate status document.

---

## Goals

- Eliminate the "one driver, everyone watches" bottleneck
- Structured collaboration around QRSPI phase artifacts across multiple repos
- Self-governing: an agent reading AGENTS.md + the file tree can determine state and advance the workflow
- Minimal tracking overhead — no sync jobs, no status files to keep consistent
- Installable: any team gets the full system by installing the plugin and running `/mob-init`

## Non-Goals (Deferred)

- Session JSONL sharing
- MCP server infrastructure
- Agent participation as team members
- Real-time sync (git push/pull is the sync primitive)
- Automatic Linear phase-label syncing

---

## System Architecture

```
claude-mob (plugin)              team-mob (repo, one per team)
────────────────────             ─────────────────────────────
/mob-init                        main/
/mob-new-project                   AGENTS.md       ← installed by plugin
/mob-new-task                      CLAUDE.md       ← installed by plugin
/mob-fork                          team.yml        ← team roster
/mob-status                      active/{slug}/
/mob-advance                       PROJECT.yml     ← project manifest
/mob-push                          tasks/
/mob-linear-link                     {task-id}/
                                       Q/
AGENTS.md template                     R/
CLAUDE.md template                     D/
                                       S/
                                       P/
```

The plugin is installed in Claude Code. The mob repo is a GitHub repo the team creates (one repo per team, not per project — projects are branches).

---

## Team Membership

Every member is identified by their GitHub username. No other identifiers exist in the system.

The roster lives on `main` in `team.yml`. Linear user IDs are optional and addable later.

```yaml
# team.yml
team:
  - id: mjohnson139
  - id: alice-ios
  - id: bob-android
    linear: bob-linear-id   # optional; enables Linear assignment
```

**Rules:**
- A member must appear in `team.yml` before they can be added to a project
- `linear` field is populated by the agent via `list_users` when explicitly requested
- Per-member phase artifacts are named `@{id}.md` throughout

---

## Branch Model

| Pattern | Purpose | Created by | Merges to `main`? |
|---|---|---|---|
| `main` | System only — plugin files, team roster | `/mob-init` | N/A |
| `active/{slug}` | Live project | `/mob-new-project` | **Never** |
| `paused/{slug}` | Paused project | `/mob-pause` (rename) | **Never** |
| `archived/{slug}` | Completed project | `/mob-archive` (rename) | **Never** |

**`main` contains only:** `AGENTS.md`, `CLAUDE.md`, `team.yml`. No project artifacts ever.

### Slug rules

Derived by the plugin from the human-provided project name:
- Lowercase, hyphens only, max 40 characters
- Unique across all statuses — a slug in `archived/` cannot be reused

---

## Data Schemas

### `team.yml` (on `main`)

```yaml
team:
  - id: mjohnson139
  - id: alice-ios
  - id: bob-android
```

### `PROJECT.yml` (root of each project branch)

```yaml
name: Shared Notification System
lead: mjohnson139
task: 20260523-push-notifications
participants:
  mjohnson139: shared
  alice-ios: ios
  bob-android: android
linear_issue: ENG-123        # optional; set by /mob-linear-link
```

That's it. No nested objects, no status tracking, no linked repo lists. The branch name carries status (`active/`). The task field points to the current active task directory.

---

## File System as State Machine

Phase and contributor progress are **derived from the file tree** — there is no separate status document.

### Phase determination

The plugin reads the task directory and applies these rules in order:

| Condition | Current phase |
|---|---|
| `Q/questions.md` absent | Q (in progress) |
| `Q/questions.md` present, any `R/@*.md` absent for a participant | R (in progress) |
| All `R/@{id}.md` present for all participants | R complete → D |
| `D/design.md` present | D complete → S |
| All `S/@{id}.md` present | S complete → P |
| All `P/@{id}.md` present | P complete → implementation |

"All participants" = the keys listed under `participants` in `PROJECT.yml`.

### Contributor completion

For phases with per-participant artifacts (R, S, P), completion is determined by file presence:
- `R/@alice-ios.md` exists → Alice is done with R
- `R/@alice-ios.md` absent → Alice is pending

The plugin never needs to track this separately.

---

## Project Branch Structure

```
active/{slug}/
├── PROJECT.yml
├── CLAUDE.md                    # auto-generated on /mob-new-project
└── tasks/
    └── {task-id}/
        ├── Q/
        │   ├── task.md          # original feature description
        │   └── questions.md     # targeted codebase questions (lead only)
        ├── R/
        │   └── @{id}.md         # one per participant — their codebase research
        ├── D/
        │   └── design.md        # shared design (~200 lines, lead-authored)
        ├── S/
        │   └── @{id}.md         # per-participant structure outline
        └── P/
            └── @{id}.md         # per-participant implementation plan
```

### Task ID convention

```
{YYYYMMDD}-{description-slug}

20260523-push-notifications
20260601-offline-sync
```

Only one task is tracked as `active` at a time in `PROJECT.yml.task`. Completed task directories stay in the branch as a record.

**Fast-track:** The lead can explicitly start the next task before all participants have merged their PRs from the current task. This is signaled by updating `PROJECT.yml.task` to the new task ID. The previous task directory remains intact. The `mob-agent` warns if the prior task has unmerged PRs, but does not block.

---

## QRSPI Phase Rules

### Q — Questions (lead only)
Artifacts: `Q/task.md`, `Q/questions.md`  
Done when: `questions.md` committed and pushed  
Fork signal: participants pull and begin R in their own codebases

### R — Research (per participant)
Artifacts: `R/@{id}.md` — objective findings with `file:line` refs, no design opinions  
Done when: all participants' files exist  
Rule: a participant must not modify another's R file

### D — Design (lead only, using all R outputs)
Artifacts: `D/design.md` — ~200 lines, current state + decisions  
Done when: `design.md` committed  
Authorship: lead only — single voice, clear decisions, no committee drafting. The `mob-designer` agent must ask the lead clarifying questions before writing; it never invents decisions.

### S — Structure (per participant)
Artifacts: `S/@{id}.md` — vertical slices, function signatures, verification checkpoints

### P — Plan (per participant)
Artifacts: `P/@{id}.md` — tactical steps with checkboxes

### W, I, PR
Executed in each participant's own application repo, not in the mob repo.  
On completion, participant posts their PR URL as a comment on the Linear issue (if linked).

---

## Plugin Agents

The plugin defines specialized agents rather than generic slash commands. Each agent has a narrow context and responsibility — consistent with QRSPI's principle that each phase should run in its own context window. Humans invoke agents; agents manage the repo.

### Agent Roster

#### `mob-agent` — Orchestrator
The primary agent. Handles repo lifecycle and project state. Run in the mob repo's working directory.

Responsibilities:
- `/mob-init` — installs `AGENTS.md`, `CLAUDE.md`, creates `team.yml` on `main`
- `/mob-new-project "{name}"` — creates `active/{slug}` branch, writes `PROJECT.yml`
- `/mob-new-task "{description}"` — scaffolds `tasks/{task-id}/Q/`, updates `PROJECT.yml.task`
- `/mob-status` — reads file tree + `PROJECT.yml`, reports phase, who's done, who's pending
- `/mob-advance` — validates phase completion via file tree, updates project `CLAUDE.md`
- `/mob-push` — commits staged files with correct `[mob]` format, pushes
- `/mob-pause` / `/mob-archive` — renames branch, updates PROJECT.yml
- `/mob-add-member {id}` — adds entry to `team.yml` on `main`

Context loaded: `AGENTS.md`, `team.yml`, `PROJECT.yml`, file tree listing (not file contents)

---

#### `mob-researcher` — R Phase Specialist
Run by a participant **in their application repo** (iOS, Android, Rails). Loads the Q-phase questions from the mob repo and guides structured codebase research, then writes the `R/@{id}.md` artifact back to the mob repo.

Responsibilities:
- Reads `Q/questions.md` from the mob repo (pulled locally)
- Explores the participant's codebase in response to each question
- Produces `R/@{github-id}.md` with `file:line` references, no design opinions
- Commits and pushes the artifact to the mob repo branch

Context loaded: `Q/task.md`, `Q/questions.md`, participant's codebase (scoped to their area), `PROJECT.yml` (for participant identity)

Key constraint: must not read other participants' R files or the task description (`task.md`) during research — only questions.

---

#### `mob-designer` — D Phase Specialist
Run by the lead after all R artifacts are present. Synthesizes across all research files into `D/design.md`.

Responsibilities:
- Reads all `R/@*.md` files
- Identifies cross-platform patterns, conflicts, and decisions
- Produces `D/design.md` (~200 lines): current state, desired outcome, decisions made, open questions resolved
- Must ask the lead clarifying questions before committing to decisions (the "must ask first" rule from QRSPI)

Context loaded: `Q/task.md`, `Q/questions.md`, all `R/@*.md` files, `PROJECT.yml`

---

#### `mob-linear-agent` — Linear Integration
Handles all Linear operations. Can be invoked by `mob-agent` or directly.

Responsibilities:
- `/mob-linear-link` — creates or links a Linear issue, stores ID in `PROJECT.yml`
- `/mob-add-member` (assist) — resolves `linear` ID via `list_users`
- On PR phase: posts PR URLs as comments on the Linear issue
- On `/mob-archive`: marks Linear issue `Done`

Context loaded: `PROJECT.yml`, `team.yml` (for `linear` ID mapping)
Tools: `mcp__claude_ai_Linear__save_issue`, `mcp__claude_ai_Linear__save_comment`, `mcp__claude_ai_Linear__list_users`

---

### Agent Interaction Model

```
Human (in mob repo)          Human (in app repo)
        │                            │
   mob-agent                  mob-researcher
  (orchestrates)             (researches codebase)
        │                            │
        └──── mob-designer ──────────┘
              (synthesizes R → D)
                    │
             mob-linear-agent
             (creates/updates issues)
```

Agents communicate via the mob repo's file tree — not via direct message passing. Each agent reads what it needs, writes its artifact, and pushes. The next agent reads from the file tree.

### `/mob-fork` — The Handoff Command

`/mob-fork` is not a full agent — it's a single-turn command the `mob-agent` answers when a participant asks "how do I pick this up?". The agent reads `PROJECT.yml` and the current file tree and responds with exactly:

1. `git checkout active/{slug} && git pull`
2. Which file to create (`R/@{your-id}.md`, `S/@{your-id}.md`, etc.)
3. Which agent to invoke (`mob-researcher` or `mob-designer`)
4. What input files to load (e.g., "load `Q/questions.md` before starting")

---

## Linear Integration (Lightweight)

Linear is a reference layer, not a mirror. The plugin does not attempt to keep Linear phase labels in sync automatically.

**Granularity:** One Linear issue per mob task. All participants' PRs (iOS, Android, Rails) are posted as comments on the same issue. This matches how a feature is conceptually one thing, regardless of how many repos implement it.

**What the plugin does with Linear:**

1. `/mob-linear-link` — creates a Linear issue (or links an existing one), stores the ID in `PROJECT.yml.linear_issue`
2. When a participant posts their PR URL at the PR phase, `mob-linear-agent` posts it as a comment on the single shared issue via `save_comment`
3. `/mob-archive` marks the Linear issue `Done`

**What the plugin does NOT do:**
- Auto-update Linear status on every phase advance
- Create or manage Linear phase labels
- Create separate Linear issues per participant

**Tooling:**
- `gh` CLI — GitHub operations (PR creation, branch management)
- `mcp__claude_ai_Linear__save_issue` — create/update issues
- `mcp__claude_ai_Linear__save_comment` — post PR links to issue thread
- `mcp__claude_ai_Linear__list_users` — resolve `linear` IDs for team.yml

---

## AGENTS.md Template (embedded in plugin)

The plugin installs this on `/mob-init`. Required sections:

1. **System identity** — what this repo is
2. **Branch rules** — `main` is system-only, never merge projects to `main`, branch naming
3. **Team roster** — "read `team.yml`; validate all participants against it"
4. **Phase state** — "derive phase from file tree per the rules in this section"
5. **Artifact rules** — what goes in each phase folder, `@{id}.md` naming
6. **Commit format** — `[mob] {action}: {description}`
7. **Prohibitions** — explicit list (see below)
8. **How to update this file** — commit to `main` only

**Prohibitions (stated verbatim):**
- Never merge a project branch into `main`
- Never create files on `main` outside of `AGENTS.md`, `CLAUDE.md`, `team.yml`
- Never modify `R/@{id}.md` authored by a different participant
- Never delete phase artifacts
- Never invent a capability not defined here; respond: *"That operation is not defined in AGENTS.md."*

### Commit format

```
[mob] {action}: {description}

action values: init | new-project | new-task | artifact | advance |
               push | pause | archive | add-member | link-linear
```

---

## `.gitignore`

```gitignore
# OS
.DS_Store
Thumbs.db

# Claude Code local state
.claude/daemon*
.claude/session-env/
.claude/file-history/
.claude/paste-cache/
.claude/backups/
.claude/settings.json
.claude/__store.db
.claude/sessions/

# Session files — share docs, not sessions
**/*.jsonl

# Secrets
.env
.env.local
secrets/

# Editor
.idea/
*.swp
*.swo
```

---

## Success Criteria

1. An agent reading only `AGENTS.md` + `PROJECT.yml` + the file tree can determine current phase, who's pending, and what to do next — with no STATUS.yml
2. Two team members on different machines can fork from the same Q-phase context with a single `git pull` + `/mob-fork`
3. `main` stays clean — zero project artifacts; enforceable with a simple CI check
4. Plugin installs in one command; a new team is operational in under 5 minutes

---

## Plugin Distribution

**Initial:** Install via git URL — `claude plugin install github:mjohnson139/claude-mob`. No registry required; allows fast private iteration while proving the system with the team.

**Later:** Publish to the Claude Code plugin registry once the system is stable and the interface is settled. No changes to the plugin's internal design are needed to make this transition.

---

## Decisions Log

| Question | Decision |
|---|---|
| Task concurrency | Sequential by default; lead can fast-track (start next task before prior PRs merge) |
| Phase D authorship | Lead only — single voice, `mob-designer` asks clarifying questions, never invents decisions |
| Linear issue granularity | One issue per mob task; all participant PRs posted as comments |
| Plugin distribution | Git URL now → registry when stable |
