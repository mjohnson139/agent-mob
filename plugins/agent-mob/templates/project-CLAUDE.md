# {{project_name}} — Project Context

**Branch:** `{{branch}}`
**Lead:** @{{lead}}
**Active Task:** {{task}}

---

## Participants

{{participants}}

---

## Quick Reference

### Current Phase
Run /mob-status to see the current phase and what's pending.

### Your Next Action
Run /mob-join to get personalized instructions for what to do next.

### Push Your Work
After creating your artifact, run /mob-contribute to commit and push.

---

## Project Rules

This branch follows the Agent Mob QRSPI workflow defined in `AGENTS.md`.

- **Phase artifacts are append-only** — never delete or overwrite another participant's file
- **R phase:** Read only `Q/questions.md` — do not read `Q/task.md` or other R files
- **D phase:** Lead only. The designer must ask clarifying questions before writing `design.md`
- **Commit format:** `[mob] {action}: {description}`
- **main branch:** Never merge project work to main

See `AGENTS.md` in the root of the mob repo for the full rulebook.

---

## File Structure

```
{{branch}}/
├── PROJECT.yml         ← project manifest (lead, participants, active task)
├── CLAUDE.md           ← this file
└── tasks/
    └── {task-id}/
        ├── Q/task.md           ← task description (lead writes)
        ├── Q/questions.md      ← research questions (lead writes)
        ├── R/@{id}.md          ← one per participant (researcher writes)
        ├── D/design.md         ← design doc (lead writes via mob-designer)
        ├── S/@{id}.md          ← spec slice (each participant writes)
        └── P/@{id}.md          ← plan file (each participant writes)
```
