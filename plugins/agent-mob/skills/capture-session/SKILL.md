---
name: capture-session
description: Use when the user invokes /capture-session or says "capture this session", "archive this session", "save this session as a recipe", "make this session reusable", "create a recipe from this session"
---

# Session Archiver

Analyze a Claude Code session and produce a structured, portable recipe artifact in markdown. No mob workspace required — works in any directory.

The artifact is storage-agnostic: write it to a file, paste it into Notion, push it to Google Drive, or hand it to any indexing system. The format is the contract.

---

## Invocation

```
/capture-session
/capture-session {path}
```

- **No args:** analyze the current session context.
- **`{path}`:** read a historical session file from `.claude/projects/` instead.

Do not prompt the user for a name. Derive everything from the session content.

---

## Mode 1 — Current Session (default)

The session is already in context. Analyze the conversation without reading any files.

**Steps:**

1. **Extract the session arc** from the conversation context:
   - **Title:** derive a concise, descriptive name from the session content (e.g., "iOS Screen with ViewModel and Service"). Do not ask the user.
   - **Starting prompt:** the first substantive user message that set the direction (verbatim or close paraphrase — do not editorialize)
   - **Tools used:** enumerate the distinct tools invoked (Read, Edit, Bash, Write, Agent, etc.) and the rough pattern of use (e.g., "Read → Edit loop ×4, then Bash to verify")
   - **Key turns:** the 3-7 decision points or pivots that shaped the session — include the user's direction AND what the agent did in response
   - **Outcome:** what was produced and whether it succeeded
   - **Tags:** infer 3-6 tags from the domain, stack, and task type
   - **Stack:** list the technologies or frameworks that appeared

2. **Write the artifact** (see format below).

3. **Derive output filename** from the title: lowercase, hyphens, `.recipe.md` suffix. Example: `ios-screen-with-viewmodel-service.recipe.md`

4. **Write to the current directory.**

5. **Output:**
   ```
   Recipe saved: {filename}

   To index it:
     - Paste into Notion and add to your recipe database
     - Run: open-brain capture < {filename}   (if using open-brain CLI)
     - Or move it wherever your team stores reusable patterns
   ```

---

## Mode 2 — Historical Session (`--session {path}`)

Read a session file from `.claude/projects/` and analyze it instead of the current context.

**Steps:**

1. **Verify the path exists:**
   ```bash
   ls "{path}"
   ```
   If not found, stop: "Session file not found at '{path}'. Check `ls ~/.claude/projects/` to find available sessions."

2. **Read the session file.** Claude Code stores sessions as JSONL — each line is a conversation turn. Read the file and parse the turn sequence.

3. **Extract the same arc fields as Mode 1** from the parsed turns.

4. Continue from Mode 1 step 3.

---

## Artifact Format

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

## Prohibitions

- **Never fabricate session content** — every field must trace to something that actually happened in the session or session file
- **Never omit Reuse Guidance** — this section is the reason the artifact exists; if you can't write it, say so and explain why
- **Never run in a mob workspace guard** — this skill works in any directory; do not check for AGENTS.md
- **Never modify the session file** when using Mode 2 — read-only
