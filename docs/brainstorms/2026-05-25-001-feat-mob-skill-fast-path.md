---
title: "Mob Skill Fast Path — Inline Command Execution"
date: 2026-05-25
status: draft
type: feature
---

# Mob Skill Fast Path

## Problem

`/mob new-project` takes ~95 seconds and ~15,000 tokens. The operation is fully deterministic (derive slug → check uniqueness → create branch → write 2 files → commit), but the current architecture routes through a sub-agent spawn:

```
/mob {command}  →  mob/SKILL.md  →  Agent(mob-agent)  →  Bash/Read/Write
```

Sub-agent cold-start dominates: the agent loads context, verifies the repo, reads templates, reasons through each step. All of that overhead is wasted on operations with no branching logic.

## Goal

All mob lifecycle commands feel instant — comparable to `git init`. Token cost for deterministic operations drops to near-zero reasoning overhead.

## Success Criteria

- `/mob new-project` completes in under 5 seconds
- No sub-agent dispatch for lifecycle commands (new-project, new-task, status, push, add-member)
- `/mob fork` may still dispatch to mob-researcher or mob-designer as needed
- All existing user-facing behavior is preserved (same commands, same outputs, same file conventions)

## Approach

Rewrite `plugins/agent-mob/skills/mob/SKILL.md` to contain complete per-command logic. Each command section provides exact steps — bash commands, file paths, template substitutions — executed inline in the main conversation context using direct Bash/Read/Write tool calls.

Remove the `Use the mob-agent via the Agent tool` routing pattern for lifecycle commands.

### Command routing after this change

| Command | Execution |
|---|---|
| `new-project` | Inline (bash + write) |
| `new-task` | Inline (bash + write) |
| `status` | Inline (read + bash) |
| `push` | Inline (bash) |
| `add-member` | Inline (read + write + bash) |
| `fork` | Inline determination → dispatches to mob-researcher or mob-designer if needed |

### mob-agent.md fate

Retained as an optional direct-invocation agent. No longer in the hot path for any command. Users can still invoke it directly; it is not deprecated.

## Scope Boundaries

**In scope:**
- Rewrite `plugins/agent-mob/skills/mob/SKILL.md`
- Preserve all command behavior, file conventions, commit format, and phase rules

**Out of scope:**
- Changes to mob-researcher.md or mob-designer.md (they do open-ended reasoning, agent model is appropriate)
- Changes to PROJECT.yml schema, branch model, or QRSPI phase logic
- New commands or capabilities

## Files Changed

- `plugins/agent-mob/skills/mob/SKILL.md` — full rewrite
- `plugins/agent-mob/agents/mob-agent.md` — add note that it is no longer the primary execution path (optional; low priority)
