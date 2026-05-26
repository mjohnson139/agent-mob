---
title: "feat: Participant UX Overhaul + Specialist Model"
date: 2026-05-25
status: active
type: requirements
---

# Participant UX Overhaul + Specialist Model

## Problem

The Agent Mob participant experience feels mechanical — closer to running git commands than joining a team. Specific friction observed in live testing:

- `/mob fork` is a git term that doesn't communicate what you're doing (joining a project and starting your phase work)
- `/mob push` doesn't communicate that you're contributing something to the team
- Suggested commands are wrapped in backticks, requiring the user to delete them before running
- Participants must manually checkout the right branch before running any command
- Everyone does every question — there's no way to match work to expertise
- There's no agent-assisted onboarding for new team members joining mid-project

## Goals

1. Make the participant entry point branch-agnostic and self-guiding
2. Replace git-centric command names with collaboration vocabulary
3. Introduce a specialist model so work is matched to expertise
4. Offer AI assistance at the moment of contribution — briefing or mechanics support
5. Fix backtick formatting in suggested command output

---

## Command Renames (Participant-Facing)

| Old | New | Reason |
|---|---|---|
| `/mob fork` | `/mob join` | "Join" communicates the act; covers all phases, not just R |
| `/mob push` | `/mob contribute` | Reflects contributing to the team, not a git operation |

Commands not renamed: `new-project`, `new-task`, `add-member`, `status`, `init` — lead commands run infrequently and are out of scope for this change. `/mob status` is already descriptive.

---

## The Join Flow

### Entry point

`/mob join` works from **any branch**, including `main`. The plugin owns all git operations — the participant never needs to know a branch name.

**Discovery mode** (`/mob join`, no args):
- Lists all active projects sorted by most recent commit date
- Shows each project's name, current phase, and how many open questions remain
- User picks by number

**Direct mode** (`/mob join {project-name}`):
- Skips discovery
- Checks out and pulls the correct branch immediately

After landing on the branch, the plugin runs the full status check and presents the participant's personal view.

---

### First-time vs. returning detection

Detection heuristic: does the calling user have any artifacts on this project branch?

- **First time** (no artifacts): guided experience
- **Returning** (artifacts present): terse experience

---

### First-time experience (guided)

1. Greet the participant by GitHub ID
2. Show project name, current phase, and a brief summary of what's been done so far (who has contributed, which questions are answered)
3. Show the participant's role-assigned questions (see Specialist Model below)
4. Offer two paths:
   - **"Brief me"** — agent summarizes the task context, explains what each question is asking for, and orients the participant before they start
   - **"Let's go"** — agent moves directly to the contribution flow, handling mechanics and formatting while the participant provides substance

---

### Returning experience (terse)

Single compact output:
- Current phase
- Participant's open/unclaimed questions (if any)
- One recommended next action

No greetings, no summaries, no ceremony.

---

## Backtick Fix

Suggested slash commands in skill output must **not** be wrapped in backticks. Plain text so they are runnable as-is in Claude Code.

**Before:** "Run `/mob join` to get started."
**After:** "Run /mob join to get started."

This applies everywhere in `plugins/agent-mob/skills/mob/SKILL.md` and agent response instructions.

---

## Specialist Model

### Overview

The current model assumes all participants answer all questions. The specialist model matches questions to expertise. The lead declares what specialist roles a project needs; questions are tagged with required roles; participants self-assign based on their specialty with optional role fluidity.

---

### Role definition (lead, at project creation)

When the lead runs `/mob new-project`, they declare the specialist roles needed:

```
/mob new-project "iOS Auth Redesign"
→ "What specialist roles does this project need?"
→ Lead: "iOS engineer, backend engineer, security researcher"
```

Roles are stored in `PROJECT.yml`:

```yaml
roles:
  - ios-engineer
  - backend-engineer
  - security-researcher
```

Participants are associated with a role when added or when they join:

```yaml
participants:
  alice: ios-engineer
  bob: backend-engineer
  mjohnson139: security-researcher
```

---

### Question tagging (lead, when writing Q questions)

When the lead writes `Q/questions.md`, each question is tagged with the role best suited to answer it. Format TBD by planning, but the intent is:

```
Q1 [ios-engineer]: How does the current token refresh flow work on iOS?
Q2 [backend-engineer]: What are the current session expiry rules on the API?
Q3 [security-researcher]: What attack surfaces exist in the current auth handshake?
```

Questions without a role tag are available to all participants.

---

### Question claiming (participant, on join)

After join:
- System shows the participant their role-assigned questions
- Questions already claimed by another participant are marked as taken
- **Role fluidity**: participant can view and claim questions outside their role if they choose — no hard locks

Claim state is tracked per question in the task directory (format TBD by planning).

---

### Specialist agent execution

One generic `mob-researcher` handles all specialist roles. When dispatched, it receives:
- The participant's declared role
- Their claimed questions
- Task context and prior R artifacts (for briefing)

No separate agent files are needed per specialty. The design is intentionally extensible — future specialist agents (e.g., a dedicated `mob-ios-researcher`) can be added to `plugins/agent-mob/agents/` and dispatched by role name without changing the core join flow.

---

### Contribution flow (after join + orientation)

After the participant is oriented and has claimed their questions, the agent offers:

- **Brief mode**: "Want me to brief you on the task and your questions before you start?" — agent reads prior artifacts and task context, explains what's needed, then hands off
- **Mechanic mode**: agent handles structure and formatting while the participant provides substance — useful for participants who know their domain but want help with artifact format

The human directs the substance. The agent assists with mechanics.

`/mob contribute` (was `/mob push`) commits and pushes the participant's artifact with the standard `[mob] artifact:` commit format.

---

## Scope Boundaries

### In scope
- `plugins/agent-mob/skills/mob/SKILL.md`: rewrite join flow, rename fork→join and push→contribute, fix backtick formatting throughout
- `plugins/agent-mob/agents/mob-researcher.md`: update to accept role + claimed questions as context, update dispatch references
- `plugins/agent-mob/agents/mob-agent.md`: update command references
- `plugins/agent-mob/templates/AGENTS.md`: update command vocabulary in participant-facing sections
- `PROJECT.yml` schema: add `roles:` block, update `participants:` to map to roles

### Out of scope (deferred)
- Typed contributions (attaching repos, documents, Linear issues to artifacts)
- Pre-built specialist agent files per domain (ios, backend, etc.)
- Lead command renames (`new-project`, `new-task`, `add-member`)
- Q/questions.md format specification (planning concern)
- Question claim state storage format (planning concern)

---

## Success Criteria

- A participant can clone a mob workspace, run `/mob join`, and land on the right branch without knowing its name
- A first-time participant receives a guided orientation; a returning participant gets a single-screen terse update
- Role-assigned questions are surfaced automatically on join; out-of-role questions are accessible but not default
- The specialist researcher agent produces a correctly formatted R artifact with role context injected
- No suggested slash commands appear in backtick code formatting in skill output
- All references to `fork` and `push` in participant-facing files are replaced with `join` and `contribute`
