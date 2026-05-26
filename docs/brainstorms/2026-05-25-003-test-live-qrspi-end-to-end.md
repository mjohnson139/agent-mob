---
title: "test: Live QRSPI End-to-End Smoke Test"
date: 2026-05-25
status: active
type: requirements
---

# Live QRSPI End-to-End Smoke Test

## Problem

Agent Mob has never been exercised as a real multi-participant workflow against a live GitHub repo. Before building on the system further, we need confidence that the full Q→R loop works: branch protection, plugin initialization, participant registration, subagent research, Codespace-based human participation, and completion detection all interlock correctly.

## Goal

Run a single collaborative test that touches every part of the Q→R workflow with a real human participant (on a Codespace) and simulated subagent participants (in the orchestrating session). When the human signals completion, the system confirms D-phase readiness.

---

## Actors

| Role | Identity | Where they run |
|---|---|---|
| Lead / orchestrator | This Claude Code session | Local machine |
| Simulated participant A | `agent-alice` | Subagent spawned from this session |
| Simulated participant B | `agent-bob` | Subagent spawned from this session |
| Human participant | `mjohnson139` | GitHub Codespace (mob plugin installed) |

---

## Test Repo

- **Name:** `mjohnson139/mob-test-workspace`
- **Visibility:** Public
- **Main branch protection:** No direct pushes, even by admins (`--require-review`, `--no-allow-admin-bypass`, or equivalent `gh` API calls)
- **Purpose:** Ephemeral test workspace — not merged back anywhere, kept for post-test inspection

---

## Test Task

**Purely mechanical** — known-answer placeholder, content irrelevant to the test's validity.

> "Describe the color red from three distinct angles."

This task is simple enough that subagents can produce plausible R artifacts without real research, keeping test runtime short. The Q questions will be similarly mechanical (e.g., "What is red physically?", "What is red emotionally?", "What is red culturally?").

---

## Participant Pre-Registration

`mjohnson139` must be listed in `PROJECT.yml` before subagents run their R-phase work, so the R-completion check waits for the human participant's artifact.

```yaml
participants:
  agent-alice: shared
  agent-bob: shared
  mjohnson139: shared
```

---

## Workflow Sequence

### Lead actions (this session)

1. Create `mjohnson139/mob-test-workspace` on GitHub via `gh repo create`
2. Clone locally, run `/mob init` to install `AGENTS.md`
3. Run `/mob new-project "Red Study"` → creates `active/red-study` branch
4. Add all three participants to `PROJECT.yml`
5. Run `/mob new-task "describe the color red from three angles"` → creates task scaffold
6. Write `Q/task.md` and `Q/questions.md` (three mechanical questions)
7. Commit and push `active/red-study`
8. Set branch protection on `main` via `gh api`
9. Print onboarding notification to chat (see below)
10. Spawn `agent-alice` and `agent-bob` as parallel subagents → each writes and pushes `R/@{id}.md`

### Human participant actions (Codespace)

1. Clone `mob-test-workspace`
2. `git pull` to see current state (subagent R artifacts may already be present)
3. Checkout `active/red-study`
4. Run `/mob fork` → get personal next-step instructions
5. Write `R/@mjohnson139.md`
6. Run `/mob push`
7. Say **"pumpkin"** in the Codespace Claude session

### Lead response to "pumpkin"

1. Pull latest from `active/red-study`
2. Check that `R/@agent-alice.md`, `R/@agent-bob.md`, and `R/@mjohnson139.md` all exist
3. Announce: "All three R artifacts present. Ready to begin D-phase on your signal."
4. Wait — D-phase does not auto-trigger

---

## Onboarding Notification (printed to chat)

The lead generates this text after step 9 above. Format: copy-paste ready block.

**Required content:**
- Repo URL: `https://github.com/mjohnson139/mob-test-workspace`
- Clone command
- Branch to check out: `active/red-study`
- How to add yourself as a member (pre-done by lead — just pull and check `PROJECT.yml`)
- Command to get personal instructions: `/mob fork`
- Completion signal: say "pumpkin" when R artifact is pushed

---

## Pass Criteria

The test passes when ALL of the following are true:

| Check | Expected state |
|---|---|
| `active/red-study` branch exists | ✓ |
| `Q/questions.md` present | ✓ |
| `R/@agent-alice.md` present | ✓ |
| `R/@agent-bob.md` present | ✓ |
| `R/@mjohnson139.md` present | ✓ |
| `main` branch has no project commits | ✓ |
| Lead announces D-phase readiness | ✓ |

---

## Out of Scope

- S-phase, P-phase, implementation
- Teardown or archival of the test repo (kept for inspection)
- Testing with more than one human participant
- Testing the D-phase itself (that fires only if the user gives the green light after "pumpkin")

---

## Risks / Assumptions

| Item | Note |
|---|---|
| `gh` CLI permissions | Creating a repo and setting branch protection requires the orchestrating session to be authenticated as `mjohnson139`. Verify `gh auth status` before starting. |
| Subagent push access | Subagents push via the same git credentials as the parent session. No separate auth needed. |
| Codespace plugin install | User installs the mob plugin in the Codespace before the test: `claude plugin install git@github.com:mjohnson139/agent-mob.git` |
| Subagent timing | Subagents may finish before or after the user clones. `git pull` before `/mob fork` is required and included in the onboarding notification. |
| "Pumpkin" detection | The Codespace Claude session does not have a direct signal channel to this session. The human communicates "pumpkin" in their session; the lead polls by pulling and checking file state. |
