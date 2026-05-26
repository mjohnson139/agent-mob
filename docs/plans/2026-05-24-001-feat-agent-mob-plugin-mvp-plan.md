---
title: "feat: Agent Mob Plugin MVP"
date: 2026-05-24
status: delivered
origin: docs/brainstorms/claude-mob-requirements.md
---

# feat: Agent Mob Plugin MVP

## Problem Frame

The `agent-mob` repo exists with the system-layer files (`AGENTS.md`, `CLAUDE.md`, `team.yml`) but has no Claude Code plugin machinery — no agents, no templates, no installable manifest. A team cannot yet run `/mob-new-project` or hand off a QRSPI fork to a second person.

This plan builds the minimal plugin that proves the two core mechanics:

1. **Project lifecycle** — a lead can create a project branch, scaffold a task, and push Q-phase artifacts in one working session
2. **Parallel fork** — a second team member, working in a separate clone, can pull the Q artifacts, run their own research, push an R artifact, and see the phase advance

The proof is structured as a real project: the lead and an iOS engineer run QRSPI through Q → R → D on the feature "D3 visualization of the agent-mob system." The design document is the output — the visualization itself is not built here.

---

## Scope

**In scope:**
- Plugin manifest (`.claude-plugin/plugin.json`)
- Three agents: `mob-agent` (orchestrator), `mob-researcher` (R phase), `mob-designer` (D phase)
- Template files embedded in the plugin (`templates/AGENTS.md`, `templates/PROJECT.yml`, `templates/project-CLAUDE.md`)
- Two `/tmp` clone workspaces as the test harness
- Full QRSPI Q → R → D walkthrough on the D3 viz task

**Out of scope:**
- Linear integration (`mob-linear-agent`) — deferred per requirements Non-Goals
- `/mob-init` command (mob repo already bootstrapped)
- `/mob-pause`, `/mob-archive` lifecycle commands — deferred
- D3 visualization implementation — the design document is the artifact

---

## Requirements Traceability

| Requirement | Source |
|---|---|
| Plugin installs in one command; team operational in <5 min | Success Criteria 4 (origin) |
| Agent reading AGENTS.md + file tree derives phase with no STATUS.yml | Success Criteria 1 (origin) |
| Two members fork from Q with `git pull` + one command | Success Criteria 2 (origin) |
| `main` stays clean, no project artifacts | Success Criteria 3 (origin) |
| File-tree-as-state-machine (presence = completion) | File System as State Machine (origin) |
| Per-member artifacts named `@{id}.md` | Artifact Rules (origin) |
| Commit format `[mob] {action}: {description}` | Commit Format (origin) |
| mob-designer asks lead questions before writing | Phase D authorship decision (origin) |
| mob-researcher reads questions only, not task.md | Phase R rules (origin) |

---

## Output Structure

Files added to the existing `agent-mob` repo:

```
agent-mob/
├── .claude-plugin/
│   └── plugin.json           ← installable plugin manifest
├── agents/
│   ├── mob-agent.md          ← orchestrator (new-project, new-task, fork, status, push)
│   ├── mob-researcher.md     ← R phase specialist
│   └── mob-designer.md       ← D phase specialist
└── templates/
    ├── AGENTS.md             ← installed into mob repos by mob-agent
    ├── PROJECT.yml           ← project manifest template
    └── project-CLAUDE.md     ← project branch CLAUDE.md template
```

Existing files modified:
- `team.yml` — add iOS engineer test member
- `AGENTS.md` — update to reflect plugin authoring context
- `CLAUDE.md` — update quick reference for plugin development

Test artifacts (not committed to repo):
```
/tmp/agent-mob-leader/    ← leader clone
/tmp/agent-mob-ios/       ← iOS engineer clone
```

---

## High-Level Technical Design

*This illustrates the intended approach and is directional guidance for review, not implementation specification.*

### Agent context boundaries

Each agent is deliberately narrow — it loads only what it needs for its phase:

```
mob-agent
  loads: AGENTS.md, team.yml, PROJECT.yml, file-tree listing
  does:  project lifecycle, phase reporting, commits, push

mob-researcher
  loads: Q/task.md (identity only), Q/questions.md, participant's app codebase
  does:  structured codebase research → R/@{id}.md
  must NOT load: other R/@*.md files, D/design.md

mob-designer
  loads: Q/task.md, Q/questions.md, all R/@*.md
  does:  asks lead questions → D/design.md
  rule:  never writes design.md before asking at least one clarifying question
```

### Phase state machine (file-tree logic)

The mob-agent evaluates phase by checking file presence in this order:

```
task directory
  └─ Q/questions.md absent?         → Phase Q
  └─ Q/questions.md present
       └─ any R/@{id}.md missing    → Phase R (list who's pending)
       └─ all R/@{id}.md present
            └─ D/design.md absent   → Ready for D (all R complete)
            └─ D/design.md present
                 └─ any S/@{id}.md missing → Phase S
                 └─ all S/@{id}.md present
                      └─ any P/@{id}.md missing → Phase P
                      └─ all P/@{id}.md present  → Complete
```

"All participants" = keys under `participants:` in `PROJECT.yml`.

### Two-workspace test flow

```
/tmp/agent-mob-leader                    /tmp/agent-mob-ios
─────────────────────                    ──────────────────
git clone agent-mob                      git clone agent-mob
checkout active/document-agent-mob       checkout active/document-agent-mob

mob-agent: new-project                         ←── push ───→  pull
mob-agent: new-task (D3 viz)                   ←── push ───→  pull
write Q/task.md + Q/questions.md               ←── push ───→  pull

                                               mob-researcher: write R/@ios-engineer.md
                                               git push ──────────────────────────→

mob-researcher: write R/@mjohnson139.md
git push ──────────────────────────────→

                                               git pull → sees both R files
mob-agent: status → "Phase D, all R complete"
mob-designer: write D/design.md
git push ──────────────────────────────→

                                               git pull → sees D/design.md
```

---

## Implementation Units

### U1. Plugin manifest

**Goal:** Make `agent-mob` installable as a Claude Code plugin via git URL.

**Requirements:** Plugin Distribution decision (origin), Success Criteria 4

**Dependencies:** None

**Files:**
- `.claude-plugin/plugin.json`

**Approach:**
- `name`: `agent-mob`
- `version`: `0.1.0`
- `description`: concise one-liner describing agent-mob's purpose
- `author`, `repository`, `license` fields populated
- No `mcpServers` section — plugin has no MCP dependencies in MVP
- Claude Code auto-discovers `agents/` and `templates/` via directory convention; no explicit registration needed

**Test scenarios:**
- After plugin file is written, validate JSON is well-formed (`python3 -m json.tool`)
- `name` matches `agent-mob`, `version` is semver

**Verification:** JSON is valid; required fields present; no MCP stanzas

---

### U2. `mob-agent` — Orchestrator

**Goal:** Define the primary agent that handles project lifecycle commands in the mob repo.

**Requirements:** All agent capabilities (origin Plugin Agents section), file-tree state machine, commit format

**Dependencies:** U1

**Files:**
- `agents/mob-agent.md`

**Approach:**

The agent definition is a markdown file with YAML frontmatter:
```
name: mob-agent
description: [trigger phrases and examples for when Claude invokes this agent]
model: inherit
tools: Read, Write, Bash, Glob
```

The agent body (system prompt) covers:

**Identity:** Orchestrates agent-mob project lifecycle from within the mob repo. Only runs when cwd is a mob repo (contains `team.yml` and `AGENTS.md`).

**Commands the agent handles (natural language → action):**

`/mob-new-project "{name}"`:
- Derive slug from name (lowercase, hyphens, max 40 chars)
- Validate slug unique across all git branches
- Create and checkout `active/{slug}`
- Write `PROJECT.yml` with name, lead (from git config user), task: (empty), participants: (lead only, scope: shared)
- Write project `CLAUDE.md` from template
- Commit: `[mob] new-project: active/{slug}`

`/mob-new-task "{description}"`:
- Derive task-id: `{YYYYMMDD}-{slug}`
- Create `tasks/{task-id}/Q/` directory
- Write empty `tasks/{task-id}/Q/task.md` with description as first line
- Update `PROJECT.yml` `task:` field
- Commit: `[mob] new-task: {task-id}`

`/mob-status`:
- Read `PROJECT.yml` (participants list, current task)
- Walk task directory tree
- Apply phase determination rules (in order, as specified in AGENTS.md)
- Report: current phase, who's complete per phase, who's pending, what to do next

`/mob-fork`:
- Read `PROJECT.yml` current task, current participants
- Apply phase logic to determine what the calling user should do next
- Output exactly: (1) git pull command, (2) file to create, (3) which agent to invoke, (4) what inputs to load
- Does not make any git changes

`/mob-push`:
- Read staged files, determine action verb from file paths
- Commit with `[mob] {action}: {description}` format
- Push to origin
- After push, print: "Pushed. Other participants can now `git pull`."

`/mob-add-member {id}`:
- Validate: check `id` is not already in `team.yml`
- Append to `team.yml`
- Checkout main, commit, checkout back to project branch
- Commit on main: `[mob] add-member: {id}`

**Prohibitions the agent must enforce:** All prohibitions from `AGENTS.md` — stated verbatim in the agent definition so they travel with the agent at runtime.

**Test scenarios:**
- `/mob-new-project "D3 System Viz"` → creates branch `active/d3-system-viz`, writes PROJECT.yml with correct schema
- `/mob-new-project` twice with same name → agent detects slug collision, refuses
- `/mob-new-task "D3 visualization design"` → creates `tasks/20260524-d3-visualization-design/Q/` with `task.md`
- `/mob-status` with no Q/questions.md → reports "Phase Q, questions.md missing"
- `/mob-status` with Q/questions.md + 1 of 2 R files → reports "Phase R, @ios-engineer pending"
- `/mob-status` with all R files → reports "All R complete, ready for Design phase"
- `/mob-push` → commit message starts with `[mob]`, contains valid action verb
- `/mob-add-member alice-ios` → team.yml gains entry, committed to main
- Prohibition: agent refuses to create a file on `main` that isn't `AGENTS.md`, `CLAUDE.md`, or `team.yml`

**Verification:** All commands produce correct file-system state; phase logic is accurate; commit format matches spec

---

### U3. `mob-researcher` — R Phase Specialist

**Goal:** Guide a participant through structured codebase research and produce their `R/@{id}.md` artifact.

**Requirements:** Phase R rules (origin), contributor completion by file presence

**Dependencies:** U2

**Files:**
- `agents/mob-researcher.md`

**Approach:**

Frontmatter:
```
name: mob-researcher
description: [trigger: "start my research", "I'm the iOS engineer", "mob-fork told me to use this"]
model: inherit
tools: Read, Bash, Glob, Write
```

Agent body:

**Identity:** R-phase research specialist. Runs in the participant's application repo (not the mob repo). Treats the mob repo as a sibling directory available via relative path.

**Startup behavior:**
1. Ask: "What is your GitHub ID?" (needed for artifact filename)
2. Load `Q/questions.md` from the mob repo path — ask user for the path if unclear
3. Do NOT read `Q/task.md` — explicitly skip it to avoid premature design opinions
4. Do NOT read any existing `R/@*.md` files

**Research behavior:**
- For each question in `Q/questions.md`, explore the participant's codebase
- All findings must include `file:line` references
- No design proposals — only objective findings ("currently X is implemented as Y at file:line")
- Follow-up questions to the participant allowed; design opinions are not

**Output:**
- Writes `R/@{github-id}.md` into the mob repo's task R/ directory
- Standard header: participant ID, date, codebase scope, task-id
- Sections: one per question from Q/questions.md, plus a "Key Files" summary at the end

**Completion signal:**
- After writing the file: "Your research is committed. Run: `git add tasks/{task-id}/R/@{id}.md && git push`"
- Prints the mob repo path and exact file so the user can verify

**Test scenarios:**
- Agent loads Q/questions.md, confirms it did not read Q/task.md or any R files
- Research output contains `file:line` references
- Research output contains no design proposals (no "should", "could", "I recommend")
- Output file is named exactly `R/@{ios-engineer}.md` (matching the participant's id)
- If participant tries to read another's R file, agent refuses

**Verification:** R artifact written with correct filename, no design opinions, file:line refs present

---

### U4. `mob-designer` — D Phase Specialist

**Goal:** Synthesize all R artifacts into `D/design.md` under lead authorship.

**Requirements:** Phase D authorship decision — lead only, must ask questions first, never invents decisions (origin)

**Dependencies:** U2, U3

**Files:**
- `agents/mob-designer.md`

**Approach:**

Frontmatter:
```
name: mob-designer
description: [trigger: "start the design phase", "all research is done", "write design.md"]
model: inherit
tools: Read, Write, Bash
```

Agent body:

**Identity:** D-phase design synthesizer. Runs in the mob repo. Reads all R artifacts and produces a single lead-authored design document.

**Startup behavior:**
1. Confirm with user: "I'm about to read all R files and synthesize them into D/design.md. You are the lead and will make final decisions. I will ask questions before writing anything. Ready?"
2. Load all `R/@*.md` files from the current task
3. Also load `Q/task.md` and `Q/questions.md` for full context

**Before writing design.md — required question round:**
- Identify at least one design decision that the R files leave ambiguous or in conflict
- Ask the lead to resolve it
- Do not write design.md until at least one question has been answered
- Record the question + answer in the design doc under "Decisions Made"

**design.md structure (~200 lines):**
```
# Design: {task description}
## Current State
## Goal
## Decisions Made
  - [question asked] → [lead's answer]
## Cross-Platform Notes (from research)
## Open Questions Deferred
```

**Completion:**
- Writes `D/design.md`
- Prompts: "Review design.md, then commit and push with: `mob-agent /mob-push`"

**Test scenarios:**
- Agent refuses to write design.md without first asking at least one question
- design.md is ~200 lines (not a stub, not >300 lines)
- design.md has "Decisions Made" section with at least one entry
- "Current State" and "Goal" sections are present
- If only one R file exists (other participant hasn't pushed), agent warns and asks lead whether to proceed or wait

**Verification:** design.md written; question-first rule observed; standard sections present

---

### U5. Template files

**Goal:** Embed the templates that mob-agent uses when initializing project branches.

**Requirements:** AGENTS.md template sections (origin), PROJECT.yml schema (origin), project CLAUDE.md content (origin)

**Dependencies:** U2

**Files:**
- `templates/AGENTS.md`
- `templates/PROJECT.yml`
- `templates/project-CLAUDE.md`

**Approach:**

`templates/AGENTS.md` — the mob repo AGENTS.md. Content mirrors the current `AGENTS.md` in the repo (the authoritative version), extracted into a template file that mob-agent copies verbatim when setting up a new mob repo.

`templates/PROJECT.yml` — annotated schema showing all required and optional fields:
```yaml
name: ""            # Human-readable project name
lead: ""            # GitHub ID of project lead
task: ""            # Current active task-id (YYYYMMDD-slug)
participants:
  # github-id: scope  (scope: shared | ios | android | rails | all)
linear_issue: ""    # optional — set by /mob-linear-link
```

`templates/project-CLAUDE.md` — the project-branch CLAUDE.md template. Mob-agent fills in: project name, branch, lead, participant list, current task, phase, and pointers to AGENTS.md and PROJECT.yml.

**Test scenarios:**
- `templates/PROJECT.yml` is valid YAML
- `templates/AGENTS.md` contains all 8 required sections per origin
- `templates/project-CLAUDE.md` contains placeholder markers that mob-agent can string-replace

**Verification:** All three template files parse correctly; AGENTS.md template covers all required sections

---

### U6. Two-workspace test setup

**Goal:** Create two realistic independent clones of the mob repo simulating a lead and an iOS engineer working on different machines.

**Requirements:** Success Criteria 2 — two members fork with `git pull` + one command

**Dependencies:** U1–U5 (plugin files committed and pushed before cloning)

**Files:** No repo files — setup is `/tmp` only

**Approach:**

After all plugin files are committed and pushed to `origin/main`:

```
Clone 1 (leader):
  /tmp/agent-mob-leader  ← git clone git@github.com:mjohnson139/agent-mob.git
  git config user.name "Matthew Johnson"
  git config user.email "mjohnson139@github.local"

Clone 2 (iOS engineer):
  /tmp/agent-mob-ios     ← git clone git@github.com:mjohnson139/agent-mob.git
  git config user.name "iOS Engineer"
  git config user.email "ios-engineer@github.local"
```

Also: add iOS engineer to `team.yml` on `main` (in the primary working directory), commit, push — before cloning so both clones see the full team.

**Test scenarios:**
- Both clones show same `git log --oneline` after clone
- Both clones have `team.yml` with both members
- Each clone has independent git config (different user.name)
- `git remote -v` in each clone points to `github.com:mjohnson139/agent-mob`

**Verification:** Two directories exist under /tmp; each shows correct git state; both see same remote

---

### U7. QRSPI walkthrough — D3 visualization design task

**Goal:** Exercise the full Q → R → D flow end-to-end with two real git clones, proving the parallel-fork mechanic.

**Requirements:** Success Criteria 1, 2, 3; full QRSPI phase flow

**Dependencies:** U6 (clones ready), U2–U4 (agents written)

**Files written during test (committed to `active/document-agent-mob` branch):**
- `tasks/20260524-d3-visualization-design/Q/task.md`
- `tasks/20260524-d3-visualization-design/Q/questions.md`
- `tasks/20260524-d3-visualization-design/R/@mjohnson139.md`
- `tasks/20260524-d3-visualization-design/R/@ios-engineer.md`
- `tasks/20260524-d3-visualization-design/D/design.md`
- `PROJECT.yml` (updated task field)

**Approach — sequenced steps:**

**Step 1 — Leader creates project and task (in `/tmp/agent-mob-leader`):**
- Run `mob-agent` → `/mob-new-project "Document Agent Mob"`
  - Creates `active/document-agent-mob` branch
  - Writes `PROJECT.yml` with lead: mjohnson139
- Run `mob-agent` → `/mob-new-task "D3 visualization design"`
  - Creates task directory scaffold
  - Updates `PROJECT.yml` `task:` field
- Manually write `Q/task.md` — description of the D3 viz feature
- Manually write `Q/questions.md` — targeted questions for each participant:
  - For both: "What data entities does agent-mob track, and what are their relationships?"
  - For both: "What events or state transitions happen in a typical QRSPI session?"
- Run `mob-agent` → `/mob-push` → commits Q artifacts, pushes

**Step 2 — iOS engineer forks (in `/tmp/agent-mob-ios`):**
- `git pull` → sees Q artifacts on `active/document-agent-mob`
- Run `mob-agent` → `/mob-fork` → agent outputs: which file to create, which agent to run
- Run `mob-researcher` → produces `R/@ios-engineer.md`
  - Researches Q/questions.md (does not read Q/task.md)
  - Records findings about agent-mob's entities and state transitions from their perspective
- `git add R/@ios-engineer.md && git push`

**Step 3 — Leader does their own research (in `/tmp/agent-mob-leader`):**
- `git pull` → sees iOS R file (verifies fork worked)
- Run `mob-researcher` → produces `R/@mjohnson139.md`
- `git add R/@mjohnson139.md && mob-agent /mob-push`

**Step 4 — Phase advance verification:**
- Run `mob-agent` → `/mob-status` in either clone (after pull)
- Expected output: "Phase D — all R files present: @mjohnson139 ✓, @ios-engineer ✓"

**Step 5 — Design (leader only):**
- Run `mob-designer` → asks lead at least one design question → writes `D/design.md`
- `mob-agent /mob-push` → commits design.md

**Step 6 — iOS pull verification:**
- In `/tmp/agent-mob-ios`: `git pull` → sees `D/design.md`
- Run `mob-agent /mob-status` → phase shows Design complete

**Test scenarios:**
- After Step 1 push: iOS clone `git pull` shows Q artifacts (confirms push/pull works)
- After Step 2: `mob-agent /mob-status` in leader shows "@ios-engineer complete, @mjohnson139 pending" for Phase R
- After Step 3: `mob-agent /mob-status` shows "all R complete, ready for D"
- After Step 5: `D/design.md` exists, is ~200 lines, has Decisions Made section
- After Step 6: iOS clone sees design.md on `git pull`
- `git log --oneline` on the branch shows only `[mob]`-prefixed commits
- `main` branch has zero files from `active/document-agent-mob`

**Verification:** All 6 steps complete without manual git intervention beyond `git pull` and `git push`; phase logic advances correctly; design.md is a coherent ~200-line design doc for the D3 visualization

---

## Sequencing

```
U1 (manifest)
  └── U2 (mob-agent)
        └── U3 (mob-researcher)
        └── U4 (mob-designer)
        └── U5 (templates)
              └── U6 (two-workspace setup)
                    └── U7 (QRSPI walkthrough)
```

U2–U5 can be written in parallel. U6 requires all plugin files committed and pushed. U7 runs after U6.

---

## Scope Boundaries

### Deferred to follow-up work
- `mob-linear-agent` — Linear issue creation, PR comment posting
- `/mob-init` — first-run bootstrap for brand-new mob repos
- `/mob-pause`, `/mob-archive` — branch lifecycle rename commands
- `mob-designer` review mode (iOS engineer annotates D/design.md in a separate file)
- CI check enforcing "no project artifacts on `main`"
- Plugin registry publication

### Out of scope for this plan
- D3 visualization implementation — the design document from U7 is the artifact
- Session JSONL sharing — deferred per requirements Non-Goals
- MCP server infrastructure
- Agent participation as team members

---

## Risks

| Risk | Mitigation |
|---|---|
| Agent frontmatter trigger phrases too broad — mob-agent fires in non-mob repos | Startup check: agent confirms `team.yml` and `AGENTS.md` exist before proceeding |
| R artifact naming collision if GitHub ID contains special characters | Slug rule: IDs should be valid GitHub usernames (alphanumeric + hyphen only) — documented |
| `mob-designer` writes design.md without asking questions if lead adds `--force` | Prohibition stated verbatim in agent body, not just in AGENTS.md |
| Push/pull timing: both participants push R simultaneously, git conflict | R files are per-participant (`@{id}.md`), no two participants write the same file — no conflicts possible |
| `/tmp` clones lose state between sessions | Test walkthrough is designed to complete in one session; intermediate state documented in this plan |

---

## Key Decisions

| Decision | Rationale |
|---|---|
| Plugin co-located in agent-mob repo | Avoids a second repo for the MVP; the team's own mob repo doubles as the plugin distribution point |
| `/tmp` clones over git worktrees | Closer to real multi-user behavior; each clone has its own git config (different user identity) |
| mob-researcher explicitly skips Q/task.md | Preserves QRSPI's research integrity — participants answer questions without task description biasing their findings |
| mob-designer requires ≥1 question before writing | Enforces the "must ask first" QRSPI rule at the agent level, not just as a guideline |
| D3 viz is task topic, not implementation | Lets the test exercise a real design conversation without scope creep into front-end code |
