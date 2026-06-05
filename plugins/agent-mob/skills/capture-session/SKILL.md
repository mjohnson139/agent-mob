---
name: capture-session
description: Use when the user invokes /capture-session or says "capture this session", "archive this session", "save this session as a recipe", "make this session reusable", "create a recipe from this session"
---

# Session Archiver

Analyze a Claude Code session and produce a structured, portable recipe artifact in markdown. No mob workspace required — works in any directory.

**Output routing:**
- **In a mob workspace** (`AGENTS.md` present) → write to `archives/{date}-{slug}.recipe.md`, then prompt to `/mob contribute`
- **Anywhere else** → print the artifact to the chat (stdout); the user decides where it goes

---

## Invocation

```
/capture-session
/capture-session {path}
```

- **No args:** analyze the current session context.
- **`{path}`:** read a historical session file from `.claude/projects/` instead.

Do not prompt the user for a name or destination. Derive everything from the session content and the workspace context.

---

## Step 1 — Detect workspace context

```bash
ls AGENTS.md 2>/dev/null
```

- **Found** → mob workspace mode (write to `archives/`)
- **Not found** → stdout mode (print to chat)

---

## Step 2 — Extract the session arc

**For current session (no path arg):** analyze the conversation context without reading any files.

**For historical session (`{path}` provided):**
1. Verify the path exists: `ls "{path}"`. If not found, stop: "Session file not found at '{path}'. Check `ls ~/.claude/projects/` to find available sessions."
2. Read the session file (Claude Code stores sessions as JSONL — each line is a conversation turn).
3. Extract the arc from the parsed turns.

**Fields to extract:**
- **Title:** derive a concise, descriptive name from the session content. Do not ask the user.
- **Starting prompt:** the first substantive user message that set the direction (verbatim or close paraphrase — do not editorialize)
- **Tools used:** enumerate distinct tools invoked (Read, Edit, Bash, Write, Agent, etc.) and the rough pattern (e.g., "Read → Edit loop ×4, then Bash to verify")
- **Key turns:** 3-7 decision points or pivots — include the user's direction AND the agent's response
- **Outcome:** what was produced and whether it succeeded
- **Tags:** infer 3-6 tags from the domain, stack, and task type
- **Stack:** technologies or frameworks that appeared

---

## Step 3 — Write the artifact

Use this format:

```markdown
---
title: "{Human-readable recipe name}"
date: {YYYY-MM-DD}
tags: [{comma-separated tags}]
tools-used: [{comma-separated tools}]
stack: [{comma-separated technologies}]
outcome: {working-pattern | partial | exploratory}
session-type: {implementation | debugging | architecture | refactor | research}
---

# Session Recipe: {Human-readable recipe name}

## Starting Prompt

{The exact or close paraphrase of the first user message that set the session's direction.
Preserve the user's phrasing — this is the retrieval hook for future agents.}

## Arc

{3-5 sentences: what the session set out to do, how it evolved, what pivoted or surprised,
and what the final state was. Written for a future agent that needs to decide if this recipe
is relevant to their task.}

## Key Turns

{3-7 bullet points. Each describes a decision point:
what the user directed, what the agent did, and why it mattered.
Format: "**Turn N:** {user asked X} → {agent did Y} — {why this mattered}"}

## Tools Used

{Which tools were invoked, and in what pattern.
Example: "Read → Edit loop ×4 to build the screen incrementally, then Bash to run the simulator."}

## Outcome

{What was produced. Be specific — name the files, patterns, or results delivered.
Note whether the outcome was fully working, partial, or left open.}

## Reuse Guidance

{The most important section. Written directly to a future agent:
- What starting prompt to use (or adapt)
- What to watch out for
- What made this session work
- Any constraints or assumptions that were baked in
Use imperative voice: "Start by reading X", "Avoid Y because Z", "Check for W before editing V."}
```

**`outcome` values:** `working-pattern` | `partial` | `exploratory`
**`session-type` values:** `implementation` | `debugging` | `architecture` | `refactor` | `research`

---

## Step 4 — Output

**Derive filename** from title: lowercase, hyphens, `.recipe.md` suffix.
Example: `ios-screen-with-viewmodel-service.recipe.md`

---

### Mob workspace mode

1. Create `archives/` directory if it doesn't exist: `mkdir -p archives`
2. Write artifact to `archives/{YYYYMMDD}-{slug}.recipe.md`
3. Output:
   ```
   Recipe saved: archives/{filename}

   Run /mob contribute to commit and share it with the team.

   To index it in open-brain:
     mcp__claude_ai_open-brain__capture_thought with the Arc and Reuse Guidance as the thought body
   ```
4. **Offer expertise extraction:**
   > "Would you like me to extract a role expertise record from this session? I'll analyze the session arc and surface domain knowledge, judgments, and edge cases to `roles/{role}/expertise/`. (Your GitHub ID and role are needed if not detectable from git config and PROJECT.yml.)"

   - **If user says yes (or any affirmative):**
     1. Read `PROJECT.yml` to determine the participant's role and task-id. If `PROJECT.yml` is absent or the participant's role is ambiguous, ask before writing.
     2. Get GitHub ID from `git config user.name` — ask if unclear.
     3. Derive `{task-slug}` from the task-id in PROJECT.yml (strip the date prefix).
     4. Synthesize an expertise record from the session recipe (Arc, Key Turns, Outcome, and Reuse Guidance sections) — extract domain knowledge, key decisions, edge cases, and participant style signals.
     5. Write to `roles/{role}/expertise/{YYYY-MM-DD}-{task-slug}-{github-id}.md` using the expertise record format defined in AGENTS.md.
     6. Output: "Expertise record written: roles/{role}/expertise/{filename}. Run /mob contribute to commit it."
   - **If user says no or does not respond affirmatively:** skip without prompting further. The session recipe is already saved.

   This offer is softer than the prompt in `mob-researcher` — capture-session is a general utility, not an active interview session. Extraction is additive and optional.

---

### Stdout mode

Print the full artifact markdown to the chat, then output:

```
--- recipe end ---

To save and index this recipe:
  - Copy into Notion and add to your recipe database
  - Run: /mob contribute  (if you initialize a mob workspace here)
  - Capture to open-brain with the Arc + Reuse Guidance as the thought body
```

---

## Prohibitions

- **Never fabricate session content** — every field must trace to something that actually happened in the session or session file
- **Never omit Reuse Guidance** — this section is the reason the artifact exists; if you can't write it, say so and explain why
- **Never check for AGENTS.md as a guard that blocks execution** — its presence only determines output destination, not whether the skill runs
- **Never modify the session file** when using the historical path — read-only
- **Never write to `archives/` on `main`** — archives are project branch artifacts; if on `main`, treat as stdout mode
