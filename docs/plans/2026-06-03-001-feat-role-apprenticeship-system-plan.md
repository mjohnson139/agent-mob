---
title: "feat: Role Apprenticeship System"
type: feat
status: completed
date: 2026-06-03
origin: docs/brainstorms/2026-06-03-role-apprenticeship-requirements.md
---

# feat: Role Apprenticeship System

Invert the contributor join flow to elicit self-introduction before presenting options, add expert interview mode to `mob-researcher`, and capture dual outputs — project artifact and role expertise record — at every contribution session end.

---

## Problem Frame

The current `/mob join` flow treats contributors as executors: it presents a numbered menu (1/2/3) after a status overview, before asking anything about what the contributor knows. The AI does the generative thinking; the human picks from a list. Sessions produce a project artifact only — the domain knowledge expressed in the session is discarded.

This was confirmed empirically: a live test produced an A/B/C/D angle menu before the contributor had said a single word. Agent Mob's deeper purpose is to accumulate role expertise toward eventual autonomous role agents. That accumulation requires the human to speak first.

---

## Requirements

### Join Flow

- R1. When a first-time participant runs `/mob join`, the status overview is shown before any routing options appear.
- R2. After the status overview, the agent asks a single open question eliciting the participant's domain expertise ("What do you bring to this?") before presenting any options.
- R3. The agent's first response to a self-introduction is a specific routing proposal mapped to open questions — not a generic menu.
- R4. If no open questions match the participant's stated expertise, the agent proposes a complementary angle grounded in what the human said.

### Expert Interview Mode

- R5. When a participant is routed to a question via the new flow, `mob-researcher` enters dialogue mode: it asks probing follow-up questions, takes structured notes, and drafts an artifact from the interview transcript.
- R6. The artifact reflects the human's expertise, not the agent's synthesis — the participant reviews and approves the draft before it is written.
- R7. Research rules (objective phrasing, `file:line` references) do not apply in expert interview mode — the content source is the human, not the codebase.

### Dual Capture

- R8. At the end of every contribution session, `mob-researcher` writes both the project R artifact and a role expertise record.
- R9. The expertise record is strongly prompted but skippable — the participant must give a genuine reason to skip; the agent acknowledges the skip and notes the absence.
- R10. The expertise record captures: questions answered, reasoning and domain knowledge expressed, key judgments, edge cases surfaced, and personal flavor tags.
- R11. Expertise records are written to `roles/{role}/expertise/{date}-{task-slug}-{id}.md` on the current project branch.
- R12. The contributor's personal flavor summary (`contributors/{id}/flavor.md`) is updated (overwritten with a synthesized view) after each session.
- R13. Expertise records on a project branch can be promoted to workspace-root `roles/` for team-wide sharing via a documented procedure in `AGENTS.md`.

### Directory Structure and Conventions

- R14. The `roles/` directory structure is defined and documented in the workspace `AGENTS.md` template.
- R15. Committed expertise records are never overwritten — each session produces a new file (append-only convention).

### Pipeline

- R16. `/capture-session` running inside a mob workspace offers to extract a role expertise record after writing the session recipe.

---

## Key Technical Decisions

- **Expert interview mode as a dispatch flag on mob-researcher, not a new agent** (R5): a new `interview_mode: true` flag triggers a new startup branch in `mob-researcher.md`, parallel to the existing `briefing: true` and role-context injection paths. A separate `mob-interviewer.md` agent was considered but rejected — it would duplicate startup, role-context injection, and artifact write paths that `mob-researcher` already owns. The guide-me inline pattern in SKILL.md was also considered but rejected — subagent dispatch gives better session isolation for a structured multi-turn interview.

- **Automatic expertise record with skippable strong prompt** (R9): at session end, `mob-researcher` strongly prompts ("Your role expertise from this session is valuable to the team — I'm writing your expertise record now (takes about 30 seconds). If you'd prefer to skip, tell me why and I'll note the absence instead.") and proceeds unless the participant declines with a genuine reason. Explicit command (e.g., `/capture-expertise`) was considered; rejected because the brainstorm prose is clear that the record is "written by the agent from the interview transcript at the end of the session." Resolves brainstorm open question 1 (automatic vs. explicit trigger).

- **`roles/` lives on the project branch; workspace-root promotion is explicit** (R11, R13): expertise records are project-scoped by default (same branch as `tasks/`, `archives/`). Cross-project sharing is opt-in via a documented promotion procedure. Workspace-root-by-default was considered; rejected because it creates a shared mutable directory that all project branches would need to coordinate around, which breaks the isolated branch model.

- **`flavor.md` is overwritten, not appended, after each session** (R12): the flavor summary is a derived view synthesized from all available expertise records, not a raw log. Overwrite produces better signal for future agent seeding and avoids append-and-merge complexity. The expertise records themselves are the source of truth (append-only).

- **Role assignment stays at add-member time** (R2, R3): the self-introduction confirms or surfaces a mismatch with the assigned role but does not change the assignment. If a mismatch is detected, the agent notes it and suggests `/mob add-member` with the correct role.

- **Slash commands in agent output are plain text, not backtick-wrapped**: established convention — backtick wrapping forces manual removal before the command runs in Claude Code.

---

## High-Level Technical Design

### New /mob join flow

```mermaid
flowchart TB
  A[/mob join] --> B[Discovery & checkout]
  B --> C{First-time?}
  C -->|returning| R[Returning output:\nopen questions + next step]
  C -->|first-time| D[Status overview:\nphase · task · participant completion · your questions]
  D --> E[Single open question:\n'What do you bring to this?']
  E --> F[Participant self-introduction]
  F --> G[Agent maps stated expertise\nto open questions]
  G --> H{Match found?}
  H -->|yes| I[Specific routing proposal:\n'Q3 matches your background — want to take it?']
  H -->|no| J[Complementary angle proposal\nbased on stated expertise]
  I --> K[Participant confirms]
  J --> K
  K --> L[Claim question in Q/claims.yml]
  L --> M[Dispatch mob-researcher\nwith interview_mode: true]
```

### Dual-capture sequence

```mermaid
sequenceDiagram
  participant P as Participant
  participant MR as mob-researcher
  participant FS as File system

  P->>MR: dispatched (interview_mode: true, role, claimed_questions)
  MR->>P: Opening question based on stated expertise
  loop Interview loop
    P->>MR: Domain knowledge, reasoning, judgments
    MR->>P: Probing follow-up question
  end
  MR->>P: Draft artifact for review
  P->>MR: Corrections / approval
  MR->>FS: Write R/@{id}.md (project artifact)
  MR->>P: Strong prompt for expertise record
  alt Participant agrees (or no objection)
    MR->>FS: Write roles/{role}/expertise/{date}-{slug}-{id}.md
    MR->>FS: Overwrite contributors/{id}/flavor.md
  else Participant declines with reason
    MR->>P: Acknowledge skip; note absence
  end
  MR->>P: Run /mob contribute to commit and push
```

---

## Scope Boundaries

### Deferred for later
- Agent persona distillation — turning accumulated expertise into a runnable agent persona
- Automated batch processing of old sessions into expertise records
- Expertise search / retrieval interface
- Cross-project expertise synthesis

### Deferred to Follow-Up Work
- A `/mob promote-expertise` command to formalize the branch-to-workspace-root promotion as a first-class operation (the plan documents the procedure in AGENTS.md; the command is a later polish item)

### Outside this product's identity
- LLM fine-tuning or external model training pipelines — expertise accumulation is prompt-context seeding, not weight updates
- General-purpose knowledge management — scoped to role expertise for software delivery roles

---

## Open Questions

1. **Format of workspace-root promotion procedure** — should teams copy individual expertise records, or copy the full role directory? Lean toward copying the full directory to preserve the contributor sub-structure, but this is a documentation decision for U4 implementation.

---

## Implementation Units

### U1. Redesign /mob-join first-time orientation

**Goal:** Replace the 1/2/3 menu with a human-first flow: status overview → single open question → routing from self-introduction.

**Requirements:** R1, R2, R3, R4

**Dependencies:** none

**Files:**
- `plugins/agent-mob/skills/mob/SKILL.md` (modify: `/mob join` orientation section)
- `plugins/agent-mob/agents/mob-agent.md` (modify: mirror join flow changes)

**Approach:**
- In the first-time output block, remove the "Type 1, 2, or 3" menu.
- After printing the status overview, add a single open question: "Before we figure out where you fit — what do you bring to this? In a sentence or two, what's your angle?"
- After the participant responds, the agent reads their stated expertise and maps it to open questions using `Q/claims.yml` (already claimed) and `Q/questions.md` (question role tags). It then outputs a specific routing proposal, not a menu.
- If no open question maps to the stated expertise, propose a complementary angle grounded in what they said.
- Claiming logic is unchanged — write the chosen question to `Q/claims.yml` before dispatching.
- Dispatch `mob-researcher` with `interview_mode: true` (instead of the old `briefing: true` / direct dispatch split).
- Keep the returning-participant path unchanged.
- Mirror all changes in `mob-agent.md`.

**Patterns to follow:** existing claiming and dispatch logic in SKILL.md; guide-me option 3 for the conversational response shape; existing status overview output format.

**Test scenarios:**
- First-time participant self-introduces with expertise matching an open question → agent proposes that specific question, not a generic list.
- First-time participant self-introduces with expertise matching no open question → agent proposes a complementary angle naming what they said.
- First-time participant self-introduces; Q/claims.yml shows all questions are claimed → agent routes them to a complementary angle or second-perspective contribution.
- Returning participant runs `/mob join` → existing returning output unchanged (phase · open questions · next step).
- Participant is `shared` role (no specific role assignment) → routing still works using untagged questions.
- Q/questions.md not yet written → questions block omitted, self-introduction still fires.

**Verification:** Running `/mob join` on a project in R phase as a first-time participant shows status overview then open question; no menu appears until after the participant speaks.

---

### U2. Add expert interview mode to mob-researcher

**Goal:** Give `mob-researcher` a new startup branch that runs a structured dialogue to elicit the human's domain expertise, rather than searching the codebase.

**Requirements:** R5, R6, R7

**Dependencies:** U1 (provides the dispatch context; U2 must handle the new dispatch flag)

**Files:**
- `plugins/agent-mob/agents/mob-researcher.md` (modify: add `interview_mode: true` startup branch and interview loop)

**Approach:**
- Add a new top-level startup branch: `If dispatched with interview_mode: true`.
- Opening: reference the participant's stated expertise from the dispatch context (passed from the join flow). Open with a specific question tied to what they said, not a generic opener.
- Interview loop:
  - Ask one follow-up question at a time.
  - Take structured notes (internal, not shown) on: reasoning expressed, judgments made, edge cases surfaced, heuristics named.
  - Do not generate conclusions — surface and deepen what the human offers.
  - Approved probing patterns: "Can you walk me through a specific case?", "What made you choose X over Y?", "Does that hold in all contexts or just Z?", "What would you have done differently?"
- Session end: synthesize notes into a draft artifact in the standard R file format. Show the draft to the participant: "Here's what I captured. Does this reflect your take, or is something off?"
- After participant approves, write `R/@{github-id}.md`.
- Research rules (objective phrasing, `file:line`) explicitly do not apply — state this in the Prohibitions and in the startup confirmation message.
- The existing non-interview startup branch (codebase research mode) is unchanged.

**Patterns to follow:** `briefing: true` dispatch branch as the structural template for a new conditional startup; guide-me conversational loop for pacing discipline (do not auto-advance).

**Technical design (directional):**

```
Startup confirmation (interview_mode):
  "I'll be interviewing you as {role} to capture your expertise on {claimed_questions}.
   I'm the scribe — your knowledge is the source. No codebase search in this mode.
   Let's start: you mentioned [stated_expertise]. Can you walk me through a specific case?"
```

**Test scenarios:**
- Dispatched with `interview_mode: true` → opens with specific question tied to stated expertise, not a generic opener.
- Dispatched without `interview_mode` → existing codebase research startup unchanged.
- Interview produces draft; participant says "that's missing X" → agent incorporates correction and re-presents the updated draft before writing.
- Participant approves draft → agent writes `R/@{id}.md` and proceeds to dual-capture prompt (U3).
- Participant is `shared` role → interview still works; expertise record written to `roles/shared/expertise/` (or omitted from role accumulation if `shared` is not a meaningful accumulation target — defer this edge case to U3 implementation).

**Verification:** A full interview session (3-5 turns) produces an `R/@{id}.md` that reads as the participant's own expertise, not an AI synthesis.

---

### U3. Add dual-capture at session end

**Goal:** After writing the project artifact, strongly prompt for the role expertise record and write both outputs: `roles/{role}/expertise/*.md` and an updated `contributors/{id}/flavor.md`.

**Requirements:** R8, R9, R10, R11, R12, R15

**Dependencies:** U2 (interview mode produces the content; dual-capture fires from the same completion path), U4 (roles/ directory structure must be defined before the first expertise record write)

**Files:**
- `plugins/agent-mob/agents/mob-researcher.md` (modify: extend Completion section)

**Approach:**
- After writing `R/@{github-id}.md`, fire the strong prompt:
  > "Your role expertise from this session is valuable to the team — I'm writing your expertise record now (takes about 30 seconds). If you'd prefer to skip, tell me why and I'll note the absence instead."
- If no objection or a weak response (silence, "sure", "go ahead"): proceed to write.
- If a genuine reason is given: acknowledge it, note that the session produced no expertise record, and stop.
- If an ambiguous reason is given: ask once for clarification; if still unclear, default to writing.
- Expertise record format (see `roles/` directory structure defined in U4):
  - Frontmatter: date, task, contributor, questions covered
  - Sections: Domain knowledge expressed, Key decisions made, Edge cases surfaced, Personal flavor tags
  - Write from the structured interview notes (U2), not from the final artifact prose
- `flavor.md` update: synthesize a flavor summary from all existing expertise records in `contributors/{id}/` plus the current session notes. Overwrite the file.
- Path for expertise record: `roles/{role}/expertise/{YYYY-MM-DD}-{task-slug}-{github-id}.md`
- Path for flavor: `contributors/{id}/flavor.md`
- Both paths relative to the mob workspace root (same branch as the R artifact).
- Slash commands in completion output are plain text: `/mob contribute` not `` `/mob contribute` ``.

**Patterns to follow:** existing Completion section in `mob-researcher.md` for output ordering and phrasing; expertise record format from the brainstorm (see `origin:`).

**Test scenarios:**
- Session ends; participant does not respond to prompt → agent writes expertise record and flavor.
- Session ends; participant says "I'd prefer not to share this" → agent asks why; participant explains privacy concern → agent acknowledges, skips, notes the absence in chat.
- Session ends; participant says "skip" with no reason → agent says "I need a good reason — what's the concern?" and waits.
- `contributors/{id}/flavor.md` does not yet exist → agent creates it.
- `contributors/{id}/flavor.md` already exists → agent overwrites with synthesized update.
- Participant role is `shared` → expertise record is still written, to `roles/shared/expertise/`; flavor is still updated (no special case).
- Existing expertise records in `roles/{role}/expertise/` → new file appended alongside them; no existing file touched.

**Verification:** After a session, both `R/@{id}.md` and `roles/{role}/expertise/{date}-{slug}-{id}.md` exist on the branch. Running `ls roles/` and `ls contributors/` confirms the directory structure was created.

---

### U4. Define roles/ directory structure; document in AGENTS.md template

**Goal:** Establish the `roles/` and `contributors/` directory conventions in the workspace rulebook so teams and future agents understand the structure and the workspace-root promotion path.

**Requirements:** R13, R14, R15

**Dependencies:** none (can land before or alongside U1–U3)

**Files:**
- `plugins/agent-mob/templates/AGENTS.md` (modify: add directory structure, expertise record format, promotion procedure, and prohibitions)

**Approach:**
- Add a `roles/` section to the workspace layout in AGENTS.md showing:
  ```
  roles/
    {role-slug}/
      expertise/
        {YYYY-MM-DD}-{task-slug}-{github-id}.md
      contributors/
        {github-id}/
          flavor.md
  ```
- Document the expertise record format (matching the brainstorm format exactly).
- Document the workspace-root promotion procedure: to share expertise across projects, copy the relevant `roles/{role}/expertise/` directory from a project branch to the workspace root `roles/` directory and commit to the main branch. Teams can then reference it when seeding future agents.
- Add prohibitions: "Do not overwrite or edit committed expertise records — each session produces a new file."
- Note that `roles/` on the project branch is project-scoped; `roles/` at workspace root is team-scoped.

**Patterns to follow:** existing workspace layout section in `templates/AGENTS.md`; existing prohibition style.

**Test scenarios:**
- Test expectation: none — this is documentation-only; correctness is verified by inspection.

**Verification:** `plugins/agent-mob/templates/AGENTS.md` contains the `roles/` directory tree, expertise record format, promotion procedure, and the no-overwrite prohibition.

---

### U5. Wire /capture-session expertise extraction in mob workspace mode

**Goal:** After writing the session recipe, `/capture-session` offers to extract a role expertise record from the session in mob workspace mode.

**Requirements:** R16

**Dependencies:** U3 (establishes the expertise record format and path conventions)

**Files:**
- `plugins/agent-mob/skills/capture-session/SKILL.md` (modify: add expertise extraction offer after workspace-mode recipe write)

**Approach:**
- In the mob workspace path (after writing `archives/{date}-{slug}.recipe.md` and prompting `/mob contribute`), add:
  > "Would you like me to extract a role expertise record from this session? I'll analyze the session arc and surface domain knowledge, judgments, and edge cases to `roles/{role}/expertise/`. (Your GitHub ID and role are needed if not detectable from git config and PROJECT.yml.)"
- If the user says yes: read `PROJECT.yml` to get the participant's role and task-id; synthesize an expertise record from the session recipe; write to `roles/{role}/expertise/{date}-{slug}-{id}.md`.
- If the user says no or does not respond affirmatively: skip without prompting further.
- This is a softer prompt than U3's strong prompt — capture-session is a general utility, not an interview session where expertise was actively elicited. The extraction is additive, not required.
- The expertise record written here uses the same format as U3.

**Patterns to follow:** existing mob workspace mode routing in capture-session SKILL.md; expertise record format from U3/U4.

**Test scenarios:**
- `/capture-session` in a mob workspace; user says yes to extraction → expertise record written to correct path.
- `/capture-session` in a mob workspace; user says no → session recipe written, no expertise record, no further prompting.
- `/capture-session` outside a mob workspace → expertise extraction offer does not appear (stdout mode only).
- PROJECT.yml is readable and participant role is deterministic → extraction uses that role without asking.
- Participant role is ambiguous or PROJECT.yml absent → agent asks before writing.

**Verification:** Running `/capture-session` in a mob workspace shows the extraction offer after the recipe is written. Saying yes produces a valid expertise record in `roles/{role}/expertise/`.

---

### U6. Version bump and system changelog

**Goal:** Keep plugin.json and SYSTEM_CHANGELOG.md in sync with the feature addition as required by the update-system commit convention.

**Requirements:** R1–R16 (all, by convention)

**Dependencies:** U1, U2, U3, U4, U5

**Files:**
- `plugins/agent-mob/.claude-plugin/plugin.json` (modify: bump version from 0.1.5 to 0.1.6)
- `docs/SYSTEM_CHANGELOG.md` (modify: add changelog entry)

**Approach:**
- Bump patch version in `plugin.json` (0.1.5 → 0.1.6). If this feature ships in multiple commits, bump incrementally per commit.
- Add a changelog entry with the appropriate U-number sequence (continuing from U11) and a concise description of each change.

**Patterns to follow:** existing changelog entry format in `docs/SYSTEM_CHANGELOG.md`; existing version field in `plugin.json`.

**Test scenarios:**
- Test expectation: none — version bump and changelog are convention compliance, verified by inspection.

**Verification:** `plugins/agent-mob/.claude-plugin/plugin.json` shows a version greater than `0.1.5`. `docs/SYSTEM_CHANGELOG.md` has entries for each file changed in this feature.

---

## Risks & Dependencies

- **Dual-file sync discipline (SKILL.md ↔ mob-agent.md):** The join flow lives in both files; a change to one that isn't mirrored in the other produces divergent behavior depending on how the user invokes the skill. U1 must update both files in the same commit.
- **`shared` role edge case in expertise accumulation (U3):** Participants with role `shared` have no meaningful role-specific accumulation target. U3 implementation should decide whether to write to `roles/shared/expertise/` (broadest reach, least signal) or to skip expertise accumulation for shared participants. Defer to implementation; note in the expertise record if skipped.
- **Interview mode produces only the human's knowledge:** If a participant's self-introduction was vague or their expertise is thin, the interview artifacts will reflect that. No fallback to codebase research in interview mode — if the participant has little to contribute, they should decline at the routing proposal stage (U1) rather than produce a weak expertise record.

---

## Sources & Research

- `plugins/agent-mob/skills/mob/SKILL.md:195–363` — current `/mob join` orientation and 1/2/3 menu implementation
- `plugins/agent-mob/agents/mob-researcher.md` — current startup sequence, research rules, and completion section
- `plugins/agent-mob/skills/capture-session/SKILL.md` — workspace routing and recipe format pattern
- `docs/plans/2026-05-26-002-feat-participant-ux-specialist-model-plan.md` — specialist model delivery: role injection, claims.yml, role-filtered question view
- `docs/brainstorms/2026-05-26-001-guided-research-mode.md` — guide-me inline pattern precedent (agent as conversational partner, no auto-advance)
- `docs/brainstorms/2026-06-03-role-apprenticeship-requirements.md` — origin document; expertise record format and directory structure defined there
