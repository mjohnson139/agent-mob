# Requirements: Lightweight Topics Track

**Date:** 2026-05-28
**Status:** Delivered — PR #15

---

## Problem

QRSPI is purpose-built for design work — it has structured Q&A, a design synthesis phase, and spec/plan phases. Teams also need a lighter way to share and accumulate context collaboratively: session learnings from a codebase, notes on a platform decision, observations on a technical area. These don't need the full QRSPI ceremony, but they do need the same git-backed, concurrent, collision-safe contribution model.

---

## Core Outcome

Participants can create a topic inside a project, contribute their observations concurrently, and optionally synthesize contributions into a single document — all without touching the QRSPI workflow.

---

## Primary Actors

- **Lead** — creates the topic, optionally triggers synthesis
- **Participants** — contribute their observations independently and concurrently

---

## New Command: `/mob-new-topic "{description}"`

Creates a topic scaffold inside the current project branch.

**Behavior:**
- Must be on an `active/` branch (same guard as `/mob-new-task`)
- Derives a topic-id: `{YYYYMMDD}-{description-slug}` (same slug rules as task-id)
- Creates `topics/{topic-id}/description.md` with the description text
- Does **not** update `PROJECT.yml.task` — topics are independent of tasks

**Output:** "Topic '{topic-id}' created. Participants can now contribute with `/mob-join`."

---

## Directory Structure

```
active/{slug}/
├── PROJECT.yml
├── CLAUDE.md
├── tasks/          ← QRSPI (unchanged)
└── topics/
    └── {topic-id}/
        ├── description.md     ← lead writes; defines the topic
        ├── @{id}.md           ← one per contributing participant
        └── synthesis.md       ← optional; lead-triggered
```

---

## State Machine

Topics have a simple, non-blocking state:

| Condition | State |
|---|---|
| `description.md` absent | Topic not initialized |
| `description.md` present; no `@{id}.md` files | Open — awaiting contributions |
| One or more `@{id}.md` present | In progress |
| All participants' `@{id}.md` present | Contributions complete — synthesis optional |
| `synthesis.md` present | Synthesized |

"All participants" = keys under `participants:` in `PROJECT.yml`, same as QRSPI.

Topics do **not** gate project advancement. A project can have active tasks and open topics simultaneously.

---

## Contribution Format (`@{id}.md`)

Unstructured — participants write what they observed, learned, or want the team to know about the topic. No required format. File is owned by the contributor; others must not modify it.

---

## Synthesis (`synthesis.md`)

- Lead-triggered only
- Optional — a topic with no `synthesis.md` is still valid and complete
- Produced by an agent (new `mob-synthesizer` agent or extension of `mob-designer`)
- Reads all `@{id}.md` contributions and produces a single coherent document
- Append-only once committed

---

## `/mob-status` Changes

`/mob-status` should include a topics summary alongside the task status:

```
Topics: 2 open, 1 synthesized
  - 20260528-auth-session-learnings: 2/3 contributions, no synthesis
  - 20260527-ios-performance-notes: synthesized
```

---

## `/mob-join` Changes

When a participant runs `/mob-join`, open topics with missing contributions surface as additional action items alongside the current task phase.

---

## AGENTS.md Changes

The `AGENTS.md` template needs:
- `topics/` added to the project branch structure diagram
- Topic state machine added to the Phase State section
- New prohibition: **Never modify `@{id}.md` authored by a different participant** (already exists for R files; extend to topic contributions)
- New artifact rule: `synthesis.md` is lead-only and append-only once committed

---

## Out of Scope

- Topics are not linked to Linear issues
- Topics have no S or P phases
- No cross-project topic sharing
- No topic archival command (topics persist as long as the project branch does)

---

## Success Criteria

1. A participant can run `/mob-new-topic`, push, and other participants can `git pull` and add their `@{id}.md` without conflicts
2. `/mob-status` shows topic progress alongside task progress
3. Synthesis is triggerable by the lead and produces a readable single document
4. QRSPI workflow is unaffected — existing projects and tasks work identically
