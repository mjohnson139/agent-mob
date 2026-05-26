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

---

## Artifact Rules

### Project branch structure
```
active/{slug}/
├── PROJECT.yml
├── CLAUDE.md
└── tasks/
    └── {task-id}/
        ├── Q/task.md
        ├── Q/questions.md
        ├── R/@{id}.md       ← one per participant
        ├── D/design.md
        ├── S/@{id}.md       ← one per participant
        └── P/@{id}.md       ← one per participant
```

### Task ID format
```
{YYYYMMDD}-{description-slug}
Example: 20260523-user-auth-flow
```

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

### Phase rules
- **Q:** Lead only. `task.md` = original description. `questions.md` = targeted codebase questions.
- **R:** One file per participant (`@{id}.md`). Objective findings with `file:line` refs. No design opinions. Participant must not read `Q/task.md` — only `Q/questions.md`.
- **D:** Lead only. `design.md` ~200 lines: current state, desired outcome, decisions. Agent must ask lead clarifying questions before writing — never invent decisions.
- **S:** One file per participant. Vertical slices, function signatures, verification checkpoints.
- **P:** One file per participant. Tactical steps with checkboxes.
- **W/I/PR:** Executed in participant's application repo. PR URLs posted as comments on the Linear issue.

---

## Commit Format

Every commit in this repository uses this format:

```
[mob] {action}: {description}
```

Valid action values: `init`, `new-project`, `new-task`, `artifact`, `advance`, `push`, `pause`, `archive`, `add-member`, `link-linear`, `update-system`

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

If asked to do something undefined, respond: *"That operation is not defined in AGENTS.md. Update the system rules on `main` first."*

---

## How to Update This File

Changes to `AGENTS.md` are committed to `main` only. Every change must include an entry in `docs/SYSTEM_CHANGELOG.md` with date, author, and description of the change. The agent must not modify `AGENTS.md` without explicit human instruction.
