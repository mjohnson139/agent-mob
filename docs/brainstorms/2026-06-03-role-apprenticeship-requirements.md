# Role Apprenticeship System

**Date:** 2026-06-03
**Status:** Draft
**Area:** Contributor join flow · R-phase UX · Role expertise accumulation

---

## Product Thesis

Every contribution session has two outputs: a project artifact and an increment of role expertise.

When an iOS developer contributes their knowledge to a design task, they are not just writing `R/@id.md`. They are teaching the system what an iOS developer brings to a project. Over enough sessions, that accumulated expertise becomes an agent persona that can perform the role autonomously. The human is progressively training their own replacement.

Agent Mob is not a collaboration tool. It is a **role apprenticeship system** — a structured way for humans to teach AI agents how to do their jobs, through the natural motion of delivering real software.

---

## Problem

The current contributor join flow treats humans as executors, not experts:

- `/mob-join` presents numbered logistics menus ("Type 1, 2, or 3")
- `mob-researcher` generates options for what the human might contribute — the AI does the generative thinking, the human picks
- The human's domain expertise is never directly elicited; it is approximated by the agent
- Contribution sessions produce only a project artifact — the expertise embodied in the session is discarded
- There is no accumulation of role-specific knowledge across sessions

This was confirmed empirically in a live test: a contributor joining a project received a menu of four AI-generated "angles to contribute" (A/B/C/D) before they had said a single word about what they actually knew.

**Prior work:** `docs/brainstorms/2026-05-26-001-guided-research-mode.md` (Delivered) addressed keeping the human in-session for codebase research. The present initiative goes further: it reframes what the contribution session is *for* and adds role expertise as a first-class output.

---

## Actors

| Actor | Description |
|---|---|
| Lead | Creates and owns the project; writes Q/questions.md; drives phase advancement |
| Contributor | Human team member with a specific role (iOS, Rails, QA, etc.) |
| Role agent | AI agent persona seeded by accumulated role expertise; eventual autonomous contributor |
| Role | A named specialization (e.g., `ios-developer`, `rails-developer`, `qa`) that maps to both human contributors and future agent personas |

**Target team:** engineering manager, product manager, Android developer, iOS developer, 2× Rails developer, QA.

---

## Core User Flows

### 1. Human-First Join Flow

When a contributor runs `/mob-join`, they receive a **status overview** followed by an invitation to self-introduce — before any options are presented.

**Phase 1 — Status overview:**

```
Red Study 2  ·  Phase R  ·  active/red-study-2

Task: Describe the color red from three distinct angles.
      We want scientific, cultural, and psychological lenses
      that together build a rich multi-dimensional picture.

Questions asked (3):
  Q1: What is red physically?              → @agent-alice ✓
  Q2: How has red been used symbolically?  → @agent-bob ✓
  Q3: What psychological effects does red have?  → pending

Participants:
  @mjohnson139  ·  shared  ·  no artifact yet
  @agent-alice   ·  researcher  ·  complete
  @agent-bob     ·  writer      ·  complete
```

**Phase 2 — Human self-introduction:**

The agent asks a single open question before presenting any options:

> "Before we figure out where you fit — what do you bring to this? In a sentence or two, what's your angle on this topic?"

The human responds in their own words. Examples:

- "I'm an iOS engineer. I can speak to how Apple's color APIs work and why red is tricky for accessibility."
- "I've done a lot of UX work. Red is loaded with convention — I have strong opinions about when it backfires."
- "I don't know much about the physics but I've read a ton about color symbolism in East Asian design."

**Phase 3 — Routing from self-introduction:**

The agent uses the human's self-introduction to route them — it does not present a generic menu. It maps their stated expertise to the open questions and proposes a fit:

> "Q3 (psychological effects) is unclaimed and your UX background is directly relevant — you've seen red misfire in real products. Want to take that one?"

If no open questions match, the agent proposes a complementary angle grounded in what the human said they know.

---

### 2. Expert Interview Mode

Once a contributor is routed to a question, `mob-researcher` shifts into interview mode. The agent elicits the human's expertise through dialogue rather than generating content.

**Opening:**
> "Let's start with Q3: psychological effects of red. You mentioned red backfires — can you walk me through a specific case where you saw that happen?"

The agent asks follow-up questions to probe depth, surface edge cases, and draw out the reasoning behind the human's expertise:

- "What made you realize it was the red causing the problem, not something else?"
- "Does that pattern hold across platforms, or is it iOS-specific?"
- "What would you have used instead — and why?"

Throughout, the agent is taking structured notes. At the end of the interview, it proposes an artifact draft:

> "Here's what I have from our conversation. Does this capture your take, or is there something missing?"

The human reviews, corrects, and approves. **The artifact reflects the human's expertise, not the agent's synthesis.**

---

### 3. Dual Capture

Every contribution session produces two outputs:

**Output A — Project artifact:** `tasks/{task-id}/R/@{id}.md`
Standard format, unchanged from today.

**Output B — Role expertise record:** `roles/{role}/expertise/{date}-{task-slug}-{id}.md`

The expertise record captures:
- The questions the contributor answered
- The reasoning and domain knowledge they expressed (not just the conclusions)
- Key judgments, heuristics, and edge cases surfaced during the interview
- A tag for personal flavor (`contributor: {github-id}`)

The expertise record is written by the agent from the interview transcript at the end of the session, alongside the project artifact.

---

## Role Expertise System

### Directory structure

```
roles/
  ios-developer/
    expertise/
      2026-06-03-red-study-mjohnson139.md
      2026-05-15-auth-redesign-alice.md
    contributors/
      mjohnson139/
        flavor.md          ← personal communication style, domain emphases
  rails-developer/
    expertise/
      ...
  qa/
    expertise/
      ...
```

### Accumulation model

- **Per-role** is the primary accumulation target. Expertise records from all contributors with a given role feed the role's knowledge base.
- **Personal flavor** is a secondary layer. Each contributor's sessions accumulate into `contributors/{id}/flavor.md` — their characteristic emphases, communication style, and domain preferences within the role.
- A future `ios-agent` is seeded from `roles/ios-developer/expertise/` (what the role knows) plus optionally `contributors/mjohnson139/flavor.md` (how Matt specifically approaches it).

### Expertise record format

```markdown
# Role Expertise: {role}

**Date:** {YYYY-MM-DD}
**Task:** {task-id}
**Contributor:** {github-id}
**Questions covered:** {Q-ids}

---

## Domain knowledge expressed

{Reasoning, heuristics, and judgments from the interview — not just conclusions}

## Key decisions made

{Where the contributor exercised judgment and why}

## Edge cases surfaced

{Things the contributor flagged that a non-expert would have missed}

## Personal flavor tags

{Communication style notes, domain emphases, characteristic framings}
```

---

## Connection to Session Archival

The `/capture-session` skill (PR #16) feeds this pipeline. After a contribution session:

1. `/capture-session` produces a structured recipe of the session
2. The mob-researcher agent extracts the expertise record from the recipe and writes it to `roles/{role}/expertise/`
3. Personal flavor is updated in `contributors/{id}/flavor.md`

This means session archival is not just documentation — it is the mechanism by which human expertise enters the apprenticeship pipeline.

---

## Success Criteria

- A contributor joining a project says what they know **before** the agent presents any options
- The agent's first response to a self-introduction is a specific routing proposal, not a generic menu
- At the end of a contribution session, two artifacts are written: the project R file and the role expertise record
- After 5+ sessions, `roles/ios-developer/expertise/` contains enough signal to meaningfully seed an ios-agent persona
- A contributor can read their expertise records and say "yes, that sounds like me"

---

## Scope Boundaries

### In scope

- Redesigned `/mob-join` flow: status overview + human self-introduction before any options
- Expert interview mode in `mob-researcher`: dialogue-first, agent as scribe
- Role expertise record format and directory structure
- Dual-capture at session end (project artifact + expertise record)
- `/capture-session` → expertise pipeline wiring
- Personal flavor layer (`contributors/{id}/flavor.md`)

### Deferred

- Agent persona distillation — turning accumulated expertise into a runnable agent persona (next horizon)
- Automated expertise record generation without a live session (batch processing old sessions)
- Expertise search / retrieval interface
- Cross-project expertise synthesis

### Outside this product's identity

- LLM fine-tuning or external model training pipelines — expertise accumulation is prompt-context seeding, not weight updates
- General-purpose knowledge management — this system is scoped to role expertise for software delivery roles

---

## Open Questions

1. **Expertise record trigger:** does the expertise record get written automatically at session end, or does the contributor explicitly run a command to capture it?
2. **Role assignment:** is role assigned at `/mob-add-member` time (as today) or can contributors declare/update their role at join time?
3. **Flavor update cadence:** does `flavor.md` update after every session or periodically (e.g., every 5 sessions)?
