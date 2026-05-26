---
title: "feat: Participant UX Overhaul + Specialist Model"
date: 2026-05-26
status: delivered
origin: docs/brainstorms/2026-05-25-004-ux-participant-flow-and-specialist-model.md
---

# feat: Participant UX Overhaul + Specialist Model

## Problem Frame

The Agent Mob participant experience is mechanical — participants must know branch names, run git commands manually, and answer every research question regardless of expertise. This plan delivers two interrelated improvements: a friendlier participant entry point (`/mob join`) that owns all git operations and adapts its guidance to first-time vs. returning participants, and a specialist model that lets the lead define roles, tag questions by required expertise, and let participants self-assign with optional role fluidity.

(see origin: `docs/brainstorms/2026-05-25-004-ux-participant-flow-and-specialist-model.md`)

---

## Scope Boundaries

### In scope
- Rename `/mob fork` → `/mob join` and `/mob push` → `/mob contribute` across all plugin files
- Rewrite the join flow in SKILL.md: branch-agnostic entry, project discovery, git checkout/pull, first-time/returning detection, guided vs. terse output
- Introduce specialist model: `roles:` in PROJECT.yml schema, roles prompt in new-project, role-assignment in add-member, role-tagged questions format, claim state tracking, role-filtered view on join
- Update mob-researcher to accept role + claimed questions as injected context
- Fix backtick formatting on all suggested slash commands in skill and agent output

### Out of scope (deferred)
- Typed contributions (attaching repos, documents, Linear issues to artifacts)
- Pre-built specialist agent files per domain
- Lead command renames (`new-project`, `new-task`, `add-member`)
- D/S/P phase join behavior beyond what the existing status logic already covers

---

## Key Technical Decisions

**Q/questions.md role-tag format:** Inline annotation `Q1 [role-slug]: Question text`. Keeps the file human-readable, requires no new file type, and degrades gracefully — untagged questions remain available to all participants. Planning default confirmed by user.

**Question claim state storage:** `Q/claims.yml` in the task directory. Format: `Q1: github-id`, `Q2: github-id`. Append-only (unclaimed questions absent from the file). Chosen over partial-artifact detection because it separates intent from progress and allows pre-claiming before writing.

**First-time detection heuristic:** Check for the presence of any artifact by the calling user (`R/@{id}.md`, `S/@{id}.md`, `P/@{id}.md`) on the current project branch. Absent = first time; present = returning.

**Specialist model scope for new-project and add-member:** The brainstorm scoped lead command *renames* out of this plan, but the specialist model requires minimal functional additions: a roles prompt in new-project and a role-assignment step in add-member. These are additive — existing behavior is preserved, new fields are optional.

**Researcher dispatch:** `mob-researcher` remains the single agent for all roles. Role description and claimed question list are injected as context at dispatch time, not encoded in separate agent files.

---

## System-Wide Impact

| File | Nature of change |
|---|---|
| `plugins/agent-mob/skills/mob/SKILL.md` | Major rewrite of join flow; rename fork/push; backtick fix throughout |
| `plugins/agent-mob/agents/mob-agent.md` | Frontmatter description update; output references updated |
| `plugins/agent-mob/agents/mob-researcher.md` | Startup sequence updated; role/questions context added; completion references `/mob contribute` |
| `plugins/agent-mob/templates/AGENTS.md` | Participant-facing command vocabulary updated |
| `plugins/agent-mob/templates/PROJECT.yml` | Add `roles:` block; update `participants:` schema comment |

---

## Implementation Units

### U1. Command renames and backtick fix

**Goal:** Replace all `fork` → `join` and `push` → `contribute` references across plugin files, and remove backtick wrapping from all suggested slash commands in output instructions.

**Requirements:** Goals 2 and 5 from origin; Success criteria: no backtick-wrapped commands, all fork/push references replaced.

**Dependencies:** None — safe to do first, establishes the vocabulary all other units build on.

**Files:**
- `plugins/agent-mob/skills/mob/SKILL.md`
- `plugins/agent-mob/agents/mob-agent.md`
- `plugins/agent-mob/agents/mob-researcher.md`
- `plugins/agent-mob/templates/AGENTS.md`

**Approach:**
- In SKILL.md: rename the `/mob fork` section to `/mob join`, rename the `/mob push` section to `/mob contribute`, update the command list in `/mob` (no-args output), update all output strings that reference these commands
- In mob-agent.md: update frontmatter `description` field (currently lists `/mob-fork`, `/mob-push`), update all output strings in command sections
- In mob-researcher.md: update the Completion section — replace the manual `git add` / `git push` instructions with "Run /mob contribute to push your artifact"
- In templates/AGENTS.md: search for any participant-facing command references and update vocabulary
- Backtick rule: scan every output block and prose instruction; any inline slash command wrapped in backticks (e.g., `` `/mob join` ``) becomes plain text (`/mob join`)

**Patterns to follow:** Existing SKILL.md output string format for consistency.

**Test scenarios:**
- All occurrences of `fork` in participant-facing output strings are replaced with `join`
- All occurrences of `push` in participant-facing output strings are replaced with `contribute`
- No slash command in any output instruction appears wrapped in backticks
- The `/mob` no-args command list shows `join` and `contribute`, not `fork` and `push`
- mob-researcher Completion section references `/mob contribute`, not `git push`

**Verification:** `grep -r "mob fork\|mob push\|mob-fork\|mob-push" plugins/` returns only internal technical references (none in output strings). `grep -r "\`/mob" plugins/` returns no matches.

---

### U2. Join flow — branch-agnostic entry, discovery, and direct modes

**Goal:** Rewrite `/mob join` in SKILL.md so it works from any branch, lists active projects in discovery mode, and handles direct-mode checkout — all without requiring the participant to know a branch name.

**Requirements:** Goal 1; Success criteria: participant can clone, run `/mob join`, and land on the right branch without knowing its name.

**Dependencies:** U1 (command rename must be in place).

**Files:**
- `plugins/agent-mob/skills/mob/SKILL.md`

**Approach:**

Discovery mode (`/mob join`, no args):
1. Run `git fetch --all` to ensure remote branches are visible
2. List branches matching `active/*` from both local and remote
3. For each, read `PROJECT.yml` (on that branch) for project name; get last commit date
4. Sort by most recent commit date
5. Display:
   ```
   Active projects:

     1. Red Study          active/red-study          2026-05-26
     2. iOS Auth Redesign  active/ios-auth-redesign   2026-05-24

   Type a number to join, or /mob join {project-name} to go directly.
   ```
6. Wait for user selection
7. Proceed to checkout (step below)

Direct mode (`/mob join {name}`):
1. Derive slug from name (same slug rules as new-project)
2. Verify `active/{slug}` exists (local or remote); if not, stop: "No active project matching '{name}'. Run /mob join to see available projects."

Checkout (both modes):
1. `git checkout active/{slug}` (or `git checkout -b active/{slug} origin/active/{slug}` if local branch absent)
2. `git pull origin active/{slug}`
3. Proceed to U3 (first-time/returning detection and orientation)

**Patterns to follow:** Existing `/mob new-project` branch uniqueness check pattern in SKILL.md.

**Test scenarios:**
- From `main`, `/mob join` with no args lists all `active/*` branches sorted by recent commit date
- Selecting a number checks out and pulls the correct branch
- `/mob join red-study` skips discovery and goes directly to checkout
- `/mob join nonexistent` stops with a clear message naming available projects
- Running from a branch that is already `active/red-study` still works (pull + re-orient)
- Remote-only branches (not yet checked out locally) appear in discovery and can be joined

**Verification:** After running `/mob join`, `git branch --show-current` returns `active/{slug}` and the branch is up to date with remote.

---

### U3. First-time vs. returning detection and adaptive orientation

**Goal:** After landing on the project branch, detect whether this is the participant's first visit and render either a guided orientation or a terse next-step summary.

**Requirements:** Goal 1 and 4; Success criteria: first-time = guided, returning = terse.

**Dependencies:** U2 (join flow must land on branch before orientation runs), U5 (role-assigned questions are part of orientation — U5 can land before or after, with a graceful fallback if roles not yet defined).

**Files:**
- `plugins/agent-mob/skills/mob/SKILL.md`

**Approach:**

Detection:
1. Get calling user's GitHub ID (`git config user.name`; ask if unclear)
2. Check for any artifact file matching `tasks/{task-id}/*/@{id}.md` on the current branch
3. If none found → first-time; if any found → returning

First-time output:
```
Welcome to {project name}, @{github-id}.

Phase: R — Research
Task: {task description from Q/task.md}

What's been done:
  ✓ @alice   — research complete
  ✓ @bob     — research complete
  ○ @you     — not started

Your questions ({role} — 2 assigned):
  Q1: {question text}
  Q3: {question text}

How would you like to start?
  1. Brief me — I'll summarize the task and orient you before you begin
  2. Let's go — start writing your research artifact now

Type 1 or 2.
```

Returning output:
```
Red Study  •  Phase R  •  active/red-study

Your open questions: Q1, Q3
Next: /mob contribute when your artifact is ready
```

Brief path (user picks 1): dispatch `mob-researcher` with role context and claimed questions (see U6 for researcher changes). The researcher opens with a 2-3 sentence task summary before diving into questions.

Let's-go path (user picks 2): dispatch `mob-researcher` directly to research mode, skipping the briefing preamble.

**Patterns to follow:** Existing `/mob status` phase state machine and participant completion check pattern in SKILL.md.

**Test scenarios:**
- New participant with no artifacts sees the guided first-time output
- Returning participant with an existing R artifact sees the terse output
- First-time output correctly shows which other participants have completed artifacts
- Both "Brief me" and "Let's go" paths dispatch to mob-researcher
- If no roles are defined yet (pre-U4), orientation shows all questions without role filtering
- Phase display reflects the actual current phase (R, D, S, or P), not hardcoded R

**Verification:** A participant with no prior artifacts on the branch sees the guided output. A participant with an existing `R/@{id}.md` sees the terse output.

---

### U4. Specialist model — PROJECT.yml schema, new-project roles, add-member role assignment

**Goal:** Extend PROJECT.yml to hold a `roles:` list, prompt the lead for roles at project creation, and assign participants to roles (replacing the scope field).

**Requirements:** Specialist model — role definition and participant association from origin.

**Dependencies:** None — schema changes are independent of join flow changes.

**Files:**
- `plugins/agent-mob/templates/PROJECT.yml`
- `plugins/agent-mob/skills/mob/SKILL.md` (new-project and add-member sections)
- `plugins/agent-mob/agents/mob-agent.md` (new-project and add-member sections)

**Approach:**

Updated PROJECT.yml schema:
```yaml
name: ""
lead: ""
task: ""
roles:
  - ios-engineer        # specialist roles this project needs
  - backend-engineer
participants:
  github-id: ios-engineer   # role slug, or "shared" if no specialist role
```

new-project changes (SKILL.md and mob-agent.md):
- After writing PROJECT.yml with the lead as first participant, ask: "What specialist roles does this project need? (e.g., 'iOS engineer, backend engineer') — press Enter to skip."
- If roles provided: derive slugs (lowercase, hyphens), write `roles:` block to PROJECT.yml
- If skipped: write `roles: []` — specialist model is opt-in

add-member changes:
- Replace "What is {id}'s scope?" prompt with "What is {id}'s role? ({list any defined roles, or 'shared'})"
- If the project has no defined roles, default to `shared`
- Store as `{id}: {role-slug}` in participants

**Patterns to follow:** Existing slug derivation rules in SKILL.md; add-member scope prompt pattern.

**Test scenarios:**
- `/mob new-project` with roles provided writes a `roles:` block to PROJECT.yml
- `/mob new-project` with roles skipped writes `roles: []`
- `/mob add-member` with a roles-enabled project prompts with defined role options
- `/mob add-member` on a roles-free project defaults to `shared`
- Existing projects without a `roles:` key are treated as `roles: []` (backward compatible)

**Verification:** After `/mob new-project "iOS Auth" → "iOS engineer, backend engineer"`, PROJECT.yml contains `roles: [ios-engineer, backend-engineer]` and the lead is listed as `shared` (not role-assigned by default).

---

### U5. Question tagging, role-filtered view, and claim state

**Goal:** Define the inline role-tag format for questions.md, surface role-matched questions to participants on join, and track which questions are claimed in Q/claims.yml.

**Requirements:** Specialist model — question tagging, claiming with role fluidity from origin.

**Dependencies:** U3 (join orientation reads role-filtered questions), U4 (roles must exist in PROJECT.yml before filtering makes sense).

**Files:**
- `plugins/agent-mob/skills/mob/SKILL.md` (join orientation, new-task section)
- `plugins/agent-mob/templates/AGENTS.md` (document Q question format for leads)

**Approach:**

Question format (leads write this in questions.md):
```
Q1 [ios-engineer]: How does the current token refresh flow work on iOS?
Q2 [backend-engineer]: What are the current session expiry rules on the API?
Q3: What are the user-facing symptoms of an expired session? (no tag = all roles)
```

Parsing logic in SKILL.md:
- Parse `Q{n} [{role}]: {text}` — extract number, optional role, text
- Questions without a role tag are "open" — visible to all
- Questions with a role tag that matches the participant's role are "assigned"
- Questions with a role tag that does not match are "available but not assigned" (role fluidity)

Claim state — `tasks/{task-id}/Q/claims.yml`:
```yaml
Q1: alice
Q2: bob
```
- Absent key = unclaimed
- On join, show participant their assigned + unclaimed questions; show claimed-by-others as taken
- Claiming: when a participant starts working on a question (picks it in the join flow), append their ID to claims.yml and commit it

Role-filtered join view:
- Primary list: questions tagged for the participant's role + untagged questions (unclaimed)
- Secondary list (role fluidity): questions tagged for other roles, shown after a separator: "Other available questions (outside your role):"

**Patterns to follow:** Existing phase state machine file-existence checks in SKILL.md.

**Test scenarios:**
- Questions with no role tag appear in every participant's primary list
- Questions tagged `[ios-engineer]` appear in primary list for ios-engineer participants, secondary for others
- A claimed question shows the claimer's ID and is visually marked taken
- A participant can claim a question outside their role from the secondary list
- Q/claims.yml is created on first claim and appended on subsequent claims
- Projects with no role-tagged questions (pre-specialist-model content) show all questions to all participants (backward compatible)
- Malformed tags (e.g., `[missing-bracket`) fall back to treating the question as untagged

**Verification:** On join, a participant with role `ios-engineer` sees ios-engineer questions in their primary list and backend questions in the secondary list. After claiming Q1, Q/claims.yml contains `Q1: {their-id}`.

---

### U6. mob-researcher — role context injection and contribute references

**Goal:** Update mob-researcher to receive and use injected role context and claimed questions, and update its completion section to reference `/mob contribute`.

**Requirements:** Goal 3 and 4; specialist agent execution from origin.

**Dependencies:** U1 (contribute reference), U3 (dispatch point passes role context), U5 (claimed questions available at dispatch).

**Files:**
- `plugins/agent-mob/agents/mob-researcher.md`

**Approach:**

Startup sequence additions:
- Before asking for GitHub ID, check if role context was injected: if `role` and `claimed_questions` are provided in the dispatch context, skip asking and use them directly
- If dispatched without role context (legacy path), proceed with existing startup sequence unchanged (backward compatible)
- Add to the "Confirm what you will NOT read" step: "My assigned questions for this session: Q{n}, Q{n}..."

Research process update:
- When role context is present, only answer the claimed questions — do not attempt all questions
- Output format: include only the claimed Q sections; do not leave empty sections for unclaimed questions

Completion section update:
- Replace the manual `git add` / `git push` block with: "Your research artifact is ready. Run /mob contribute to commit and push it."
- Remove the `cd {mob-repo-path}` instruction (participant is already in the workspace)

**Patterns to follow:** Existing mob-researcher startup sequence and output format.

**Test scenarios:**
- When dispatched with role and claimed questions, researcher confirms which questions it will answer at startup
- When dispatched without role context, researcher falls back to the existing full-questions flow
- Output artifact only contains sections for claimed questions — no empty Q sections
- Completion message says "Run /mob contribute" not `git push`
- Researcher still refuses to read Q/task.md or other R artifacts regardless of role context

**Verification:** Dispatch mob-researcher with `{role: "ios-engineer", claimed_questions: ["Q1", "Q3"]}` — startup message names Q1 and Q3; output artifact contains only Q1 and Q3 sections; completion references `/mob contribute`.

---

## Deferred Notes

- **Q/claims.yml merge conflicts:** If two participants claim the same question simultaneously, a merge conflict will occur on claims.yml. Resolution: last-writer-wins is acceptable for now; a proper locking mechanism is deferred.
- **Phase state machine and specialist model:** The current R-completion check (`all R/@{id}.md present for all participants`) does not account for question-level completion. A participant who only answers 2 of 3 questions still produces one `R/@{id}.md`. Phase completion logic is unchanged in this plan.
- **Specialist agents per domain:** Custom agent files (e.g., `mob-ios-researcher.md`) can be added to `plugins/agent-mob/agents/` in a future plan and dispatched by role name from the join flow without changing this plan's architecture.
- **Lead command renames:** `new-project`, `new-task`, `add-member` receive functional additions only. Renaming these to the friendlier vocabulary (start, assign, invite) is deferred.
