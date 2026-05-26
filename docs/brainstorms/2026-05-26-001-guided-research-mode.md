# Guided Research Mode

**Date:** 2026-05-26
**Status:** Delivered
**Area:** `/mob join` — R-phase participant flow

---

## Problem

When a human participant runs `/mob join` and starts their R-phase research, the current flow immediately dispatches `mob-researcher` as a subagent. The human picks their questions, then watches from the outside while an autonomous agent does the work. There is no way to stay in the session and direct the investigation.

---

## What We're Building

A third option in the `/mob join` orientation menu that keeps research in the main Claude session. The human chooses questions to focus on, then directs the investigation conversationally — where to dig, what to prioritize, what to look at next. Claude explores the codebase and reports findings. At the end of the session, the standard R artifact is written.

---

## Scope

### In scope

- Add **"Guide me"** as option 3 in the "How would you like to start?" menu shown to first-time participants
- In guided mode: no subagent dispatch — research happens inline in the main session
- Conversational back-and-forth: human steers direction, Claude reports objective findings with file:line references
- All existing research rules apply (objective findings only, no design opinions, file:line required)
- Output: standard `R/@{github-id}.md` in the same path and format as automated research
- Claim writing (`Q/claims.yml`) happens in the join flow before guided research starts — same as automated path

### Out of scope

- New commands or entry points — this is entirely within the existing `/mob join` flow
- Alternative output formats (plan doc, brainstorm doc) — output is always the R artifact
- Relaxing research rules in guided mode
- Changes to "Brief me" (option 1) or "Let's go" (option 2) — both remain unchanged as subagent-dispatch paths

---

## User-Facing Behavior

### Updated orientation prompt

```
How would you like to start?
  1. Brief me — I'll summarize the task and orient you before you begin
  2. Let's go — start writing your research artifact now
  3. Guide me — we'll explore the codebase together, you direct the investigation

Type 1, 2, or 3.
```

### Guided mode session flow

1. **Claims are written** for the participant's questions before research begins (same as today, inline in SKILL.md)

2. **Claude opens the session** by confirming the questions claimed and asking where the human wants to start:
   > "I've claimed Q1 and Q3 for you. Where do you want to start — want me to begin with Q1, or is there a specific part of the codebase you'd like to look at first?"

3. **Human directs** — the human can:
   - Say which question to tackle first
   - Point to a specific file, module, or area to investigate
   - Ask Claude to search for something specific
   - Redirect mid-investigation ("skip that, let's look at X instead")
   - Ask follow-up questions about findings

4. **Claude reports findings** using the same objective phrasing as automated research:
   - "X is implemented at `path/to/file.ts:42` as Y"
   - "No implementation of X was found in the scanned directories"
   - Does NOT use: "I recommend", "should", "could be improved"

5. **Human decides when each question is answered** — Claude does not auto-advance to the next question; it surfaces findings and waits for direction

6. **Session ends** when the human says they're done (or all claimed questions are covered). Claude writes `R/@{github-id}.md` using the standard format.

7. **Completion message** is the same as automated:
   > "Your research artifact is ready. Run /mob contribute to commit and push it."

---

## Research Rules (unchanged)

These apply in guided mode exactly as in automated mode:

- Report what exists, not what should exist
- Every finding must include a `file:line` reference
- Forbidden phrasing: "should", "could", "I recommend", "would be better", "this could be improved"
- Follow-up questions to the participant about their codebase are allowed
- If no evidence found: "No evidence found for X in the codebase"

---

## Implementation Notes

**SKILL.md changes:**
- Add option 3 to the orientation menu text
- Add a handler branch: `if user picks 3 → guided mode (no Agent dispatch)`
- In guided mode: confirm claims inline, then open the conversational research session
- Claims writing already happens inline — no change needed there

**mob-researcher.md changes:**
- None required for the initial implementation — guided mode runs inline using the same research rules documented there as reference
- Optional future: extract research rules into a shared reference so SKILL.md can cite them without duplicating

**Version bump — required on every `update-system` commit:**
Every `update-system` commit that adds a changelog entry must also bump the patch version in `plugins/agent-mob/.claude-plugin/plugin.json`. This change establishes that convention. For this commit: `0.1.0` → `0.1.1`.

The rule: changelog entry + version bump are atomic — they ship in the same commit. An `update-system` commit without a version bump is incomplete.

This rule should also be added to `AGENTS.md` under the "Commit Conventions" section so future contributors see it.

---

## Success Criteria

- A human participant can run `/mob join`, pick option 3, and complete their R artifact through conversation without a subagent being dispatched
- The resulting `R/@{github-id}.md` is indistinguishable in format from one produced by the automated path
- The human can redirect the investigation mid-session (change question order, point to specific files, skip areas)
- Claims are written to `Q/claims.yml` before research begins, same as the automated path
