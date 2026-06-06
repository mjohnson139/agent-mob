# Agent Mob — Agent Operational Rules

This is the authoritative rulebook for any AI agent operating in this repository. Read this file before taking any action. Follow these rules strictly. Do not infer behavior not defined here.

---

## System Identity

Agent Mob is a git-backed collaboration system for small engineering teams building multi-repo products with AI assistance. This repository is the coordination layer. QRSPI phase artifacts (markdown files) are the unit of shared context. The file tree is the state machine — phase and contributor progress are derived from which files exist.

**This is a `main` branch session.** The `main` branch contains only system files. All project work lives on project branches (`active/{slug}`, `paused/{slug}`, `archived/{slug}`).

---

## Branch Rules

| Branch | Purpose | Created by | Merges to `main`? |
|---|---|---|---|
| `main` | System only | Initial setup | N/A |
| `active/{slug}` | Live project | Agent (`mob-agent`) | **Never** |
| `paused/{slug}` | Paused project | Agent (rename) | **Never** |
| `archived/{slug}` | Completed project | Agent (rename) | **Never** |

**`main` contains only:** `AGENTS.md`, `CLAUDE.md`, `.gitignore`, `docs/`, `agents/`, `templates/`. No instance data, no project artifacts, no task directories, no QRSPI phase files ever exist on `main`.

---

## Branch Naming

The agent derives slugs from human-provided project names. Rules:
- Lowercase only
- Hyphens as separators — no underscores, spaces, or special characters
- Maximum 40 characters (truncate preserving meaning)
- Must be unique across all statuses — a slug used in `archived/` cannot be reused in `active/`

Valid examples: `active/ios-auth-redesign`, `paused/analytics-dashboard`, `archived/legacy-payment-flow`

---

## Team Roster

Each project branch is self-contained. The `participants:` map in `PROJECT.yml` defines who is working on that project. Per-member phase artifacts are named `@{id}.md`.

A participant is added to a project with `/mob-add-member {id}`, which appends them to `PROJECT.yml.participants` on the project branch.

---

## Phase State (File Tree as State Machine)

Phase is derived from what files exist in a task directory. Apply these rules in order:

| Condition | State |
|---|---|
| `Q/questions.md` absent | Phase Q in progress |
| `Q/questions.md` present; any participant's `R/@{id}.md` absent | Phase R in progress |
| All `R/@{id}.md` present for all participants | R complete → D |
| `D/design.md` present | D complete → S |
| All `S/@{id}.md` present for all participants | S complete → P |
| All `P/@{id}.md` present for all participants | P complete → implementation |

"All participants" = the keys listed under `participants` in `PROJECT.yml`.

## Topic State

Topics have a simple, non-blocking state derived from `topics/{topic-id}/`:

| Condition | State |
|---|---|
| `description.md` absent | Not initialized |
| `description.md` present; no `@{id}.md` files | Open — awaiting contributions |
| One or more `@{id}.md` present | In progress |
| All participants' `@{id}.md` present | Contributions complete — synthesis optional |
| `synthesis.md` present | Synthesized |

Topics do **not** gate project advancement. A project can have active tasks and open topics simultaneously.

---

## Artifact Rules

### Project branch structure
```
active/{slug}/
├── PROJECT.yml
├── CLAUDE.md
├── tasks/
│   └── {task-id}/
│       ├── Q/task.md
│       ├── Q/questions.md
│       ├── R/@{id}.md       ← one per participant
│       ├── D/design.md
│       ├── S/@{id}.md       ← one per participant
│       └── P/@{id}.md       ← one per participant
├── topics/
│   └── {topic-id}/
│       ├── description.md   ← lead writes; defines the topic
│       ├── @{id}.md         ← one per contributing participant
│       └── synthesis.md     ← optional; lead-triggered
├── roles/
│   └── {role-slug}/
│       └── expertise/
│           └── {YYYY-MM-DD}-{task-slug}-{github-id}.md   ← one per session; append-only
├── contributors/
│   └── {github-id}/
│       └── flavor.md        ← synthesized flavor summary; overwritten after each session
└── archives/
    └── {YYYYMMDD}-{slug}.recipe.md   ← one per captured session; any participant
```

### Task ID format
```
{YYYYMMDD}-{description-slug}
Example: 20260523-user-auth-flow
```

### Topic ID format
```
{YYYYMMDD}-{description-slug}
Example: 20260528-auth-session-learnings
```

### Archive ID format
```
{YYYYMMDD}-{description-slug}.recipe.md
Example: 20260528-ios-screen-viewmodel-service.recipe.md
```

Archives are session recipe artifacts produced by `/capture-session`. Any participant can add an archive at any time. Archives do not gate task or topic progression.

### Expertise record format

Expertise records capture domain knowledge expressed during a contribution session. Written by `mob-researcher` at session end.

```markdown
---
date: {YYYY-MM-DD}
task: {task-id}
contributor: {github-id}
role: {role-slug}
questions: [{Q1, Q3}]
---

# Expertise: @{github-id} — {role-slug}

## Domain knowledge expressed

{What the participant knows: mental models, heuristics, patterns they reach for}

## Key decisions made

{Judgments the participant articulated during the session, with their reasoning}

## Edge cases surfaced

{Boundary conditions, failure modes, or non-obvious constraints the participant named}

## Personal flavor tags

{3-6 short descriptors of the participant's style or perspective: e.g., "systems-first", "test-driven", "skeptical-of-abstraction"}
```

### Workspace-root promotion procedure

Expertise records on a project branch are project-scoped by default. To share expertise across projects with the whole team:

1. From the project branch, copy the relevant `roles/{role}/expertise/` directory (or individual files) to a temporary location.
2. Check out `main`.
3. Copy the files into `roles/{role}/expertise/` at the workspace root.
4. Commit to `main` with action `update-system`.

Teams can reference workspace-root expertise records when seeding future agents or onboarding new participants. Copy the full role directory to preserve the contributor sub-structure.

### PROJECT.yml schema
```yaml
name: Human-readable project name
lead: github-id
task: 20260523-current-active-task
roles:                  # optional specialist roles; [] if no specialist model
  - ios-engineer
  - backend-engineer
participants:
  github-id: shared     # role slug matching one in roles:, or "shared"
  github-id: ios-engineer
linear_issue: ENG-123   # optional
```

### Question format (Q/questions.md)

The lead writes questions using this format. Role tags are optional — untagged questions are open to all participants:

```
Q1 [ios-engineer]: How does the current token refresh flow work on iOS?
Q2 [backend-engineer]: What are the current session expiry rules on the API?
Q3: What are the user-facing symptoms of an expired session?
```

- `Q{n} [{role-slug}]: {text}` — question assigned to a specific role
- `Q{n}: {text}` — open question, visible to all participants
- Role slug must match one of the roles defined in `PROJECT.yml`
- Malformed tags (e.g., `[missing-bracket`) are treated as untagged

### Question claim state (Q/claims.yml)

Participants claim questions before working on them. The claim file lives alongside questions.md:

```yaml
Q1: alice
Q2: bob
```

- Absent key = unclaimed
- Append-only — do not remove or overwrite existing entries
- Created on first claim, appended on subsequent claims
- If two participants claim the same question simultaneously, last-writer-wins (merge conflicts are resolved manually)

### Phase rules
- **Q:** Lead only. `task.md` = original description. `questions.md` = targeted codebase questions.
- **R:** One file per participant (`@{id}.md`). Objective findings with `file:line` refs. No design opinions. Participant must not read `Q/task.md` — only `Q/questions.md`.
- **D:** Lead only. `design.md` ~200 lines: current state, desired outcome, decisions. Agent must ask lead clarifying questions before writing — never invent decisions.
- **S:** One file per participant. Vertical slices, function signatures, verification checkpoints.
- **P:** One file per participant. Tactical steps with checkboxes.
- **W/I/PR:** Executed in participant's application repo. PR URLs posted as comments on the Linear issue.
- **Topics:** Lead creates `description.md`. Each participant writes `@{id}.md` with unstructured observations. Lead may optionally trigger mob-synthesizer to produce `synthesis.md`. Topics are non-blocking.

---

## Commit Format

Every commit in this repository uses this format:

```
[mob] {action}: {description}
```

Valid action values: `init`, `new-project`, `new-task`, `artifact`, `advance`, `push`, `pause`, `archive`, `add-member`, `link-linear`, `update-system`, `sync`

Examples:
```
[mob] new-project: active/ios-auth-redesign
[mob] artifact: R/@alice-ios.md for 20260523-user-auth-flow
[mob] advance: phase R → D for 20260523-user-auth-flow
[mob] add-member: bob-android added to active/ios-auth-redesign
```

---

## Fast-Track Rule

The lead may start the next task (update `PROJECT.yml.task`) before all participants have merged their PRs from the current task. The agent warns if prior task has unmerged PRs but does not block. The prior task directory remains intact.

---

## Linear Integration

Linear is a reference layer, not a mirror.
- One Linear issue per mob task (all repos, all participants)
- Issue created on explicit request only (`/mob-linear-link`)
- PR URLs are posted as comments on the single shared issue
- Issue marked Done on `/mob-archive`
- No automatic phase-label syncing

---

## Prohibitions

The agent must never:

1. Merge a project branch into `main`
2. Create files on `main` outside of `AGENTS.md`, `CLAUDE.md`, `.gitignore`, `docs/`, `agents/`, `templates/`
3. Create a project branch with a pattern other than `{status}/{slug}`
4. Modify `R/@{id}.md` authored by a different participant
5. Delete phase artifacts — artifacts are append-only once committed
6. Advance phase in `PROJECT.yml` unless file-tree conditions are met
7. Invent a capability not defined in this document
8. Modify `@{id}.md` files in `topics/` authored by a different participant — each contributor owns their own file
9. Write `synthesis.md` directly without using mob-synthesizer
10. Modify an `archives/*.recipe.md` file after it has been committed — archives are append-only
11. Overwrite or edit a committed expertise record in `roles/{role}/expertise/` — each session produces a new file; existing records are immutable

If asked to do something undefined, respond: *"That operation is not defined in AGENTS.md. Update the system rules on `main` first."*

---

## How to Update This File

Changes to `AGENTS.md` are committed to `main` only. Every change must include an entry in `docs/SYSTEM_CHANGELOG.md` with date, author, and description of the change. The agent must not modify `AGENTS.md` without explicit human instruction.
