---
name: mob
description: Use when the user invokes /mob or any mob subcommand (new-project, new-task, status, join, contribute, add-member, sync), or asks about project phases, creating a mob project, checking task status, syncing remote changes, or collaborating via QRSPI workflow
---

# Agent Mob — Inline Fast Path

Execute all mob commands **inline** using direct Bash/Read/Write tool calls. Do NOT dispatch to mob-agent via the Agent tool (except as noted under /mob join).

**Output formatting:** When displaying slash commands to the user (e.g. /mob contribute, /mob status), write them as plain text — do NOT wrap them in backticks.

## Workspace verification

Before taking any action (except `/mob init`), verify this is a mob workspace:

```bash
ls AGENTS.md
```

If `AGENTS.md` is missing, do not hard-fail — see "Uninitialized workspace" below.

---

## Uninitialized workspace

If `AGENTS.md` is not present in the current directory, say:

```
This doesn't look like a mob workspace yet — no AGENTS.md found.

Would you like to initialize this repo as a mob workspace?
This will copy the mob rulebook (AGENTS.md) into the current directory and commit it.

Reply "yes" to initialize now.
```

If the user replies "yes":
1. Run the `/mob init` steps (see below).
2. If a command other than `/mob init` was originally invoked, continue with that command after init completes.

---

## `/mob` — no arguments

1. Run `git branch -a` to find all branches matching `active/*`, `paused/*`, or `archived/*`
2. For each branch, get the date of its most recent commit: `git log -1 --format="%ci" {branch}`
3. Sort branches by that date, most recent first
4. Print:

```
Agent Mob — projects (most recently active first)

  active/{slug}    {YYYY-MM-DD}   [current branch indicator if applicable]
  paused/{slug}    {YYYY-MM-DD}
  archived/{slug}  {YYYY-MM-DD}

Commands:
  /mob new-project "Name"      Create a new project branch
  /mob new-task "description"  Create a task scaffold on the current branch
  /mob status                  Show current phase and who is pending
  /mob join                    Join a project and get your next-step instructions
  /mob contribute              Commit staged files and push to origin
  /mob add-member {github-id}  Add a participant to the current project
```

If no project branches exist yet, omit the projects section and just show the commands.

---

## `/mob new-project "{name}"`

Execute inline. Steps:

1. **Derive slug:** lowercase, hyphens as separators, no underscores/spaces/special chars, max 40 chars.
   - Example: "iOS Auth Redesign" → `ios-auth-redesign`

2. **Check uniqueness:**
   ```bash
   git branch -a | grep "{slug}"
   ```
   If any match (in any status), stop: "A project with slug '{slug}' already exists. Choose a different name."

3. **Create branch:**
   ```bash
   git checkout -b active/{slug}
   ```

4. **Get lead GitHub ID:**
   ```bash
   git config user.name
   ```
   Fall back to asking the user if the result is unclear or empty.

5. **Ask for specialist roles:** "What specialist roles does this project need? (e.g., 'iOS engineer, backend engineer') — press Enter to skip."
   - If roles provided: derive role slugs (lowercase, hyphens, e.g., "iOS engineer" → `ios-engineer`).
   - If skipped: roles = `[]`

6. **Write `PROJECT.yml`:**
   ```yaml
   name: {Human-readable project name}
   lead: {github-id}
   task: ""
   roles:
     - {role-slug}   # one per line; omit if roles: []
   participants:
     {github-id}: shared
   ```
   If no roles were provided, write `roles: []` on a single line.

7. **Write `CLAUDE.md`** using the template at `templates/project-CLAUDE.md`, substituting:
   - `{{project_name}}` → project name
   - `{{branch}}` → `active/{slug}`
   - `{{lead}}` → lead GitHub ID
   - `{{participants}}` → `{lead-github-id}: shared`
   - `{{task}}` → (empty string)

   Read the template first:
   ```bash
   cat templates/project-CLAUDE.md
   ```
   Then write the substituted result to `CLAUDE.md`.

8. **Commit:**
   ```bash
   git add PROJECT.yml CLAUDE.md && git commit -m "[mob] new-project: active/{slug}"
   ```

**Output:** "Project '{name}' created on branch active/{slug}. Add participants with /mob add-member, then create a task with /mob new-task."

---

## `/mob new-task "{description}"`

Execute inline. Steps:

1. **Verify active branch:**
   ```bash
   git branch --show-current
   ```
   If not on an `active/` branch, stop: "Run /mob new-task from an active project branch, not main or a paused/archived branch."

2. **Derive task-id:** `{YYYYMMDD}-{description-slug}`
   - Today's date in YYYYMMDD format
   - Description: lowercase, hyphens, max 40 chars for the slug part

3. **Create directories:**
   ```bash
   mkdir -p tasks/{task-id}/Q
   ```

4. **Write `tasks/{task-id}/Q/task.md`** with the description as the first line.

5. **Update `PROJECT.yml`:** Read the file, set `task: {task-id}`, write it back.

6. **Commit:**
   ```bash
   git add tasks/{task-id}/ PROJECT.yml && git commit -m "[mob] new-task: {task-id}"
   ```

**Output:** "Task '{task-id}' created. Write Q/questions.md next, then run /mob contribute."

---

## `/mob sync`

Fetch remote changes, merge them into the current branch, and produce a human-readable summary of what changed in the mob workspace.

Execute inline. Steps:

1. **Capture pre-merge state** — record the current HEAD so the diff is clean:
   ```bash
   git rev-parse HEAD
   ```
   Store as `{before_sha}`.

2. **Fetch and merge:**
   ```bash
   git fetch origin
   git merge origin/{current-branch} --no-edit
   ```
   If merge fails with conflicts, stop and report:
   > "Merge conflict — resolve manually, then run /mob sync again to get the change summary."

3. **Diff mob-tracked files** since `{before_sha}`:
   ```bash
   git diff {before_sha} HEAD -- tasks/ PROJECT.yml AGENTS.md
   ```
   If the diff is empty (already up to date), report: "Already up to date — no changes from remote." and stop.

4. **Interpret the diff** into a human-readable update. Map file-level changes to plain-English events using this table:

   | Changed file/pattern | Plain-English meaning |
   |---|---|
   | `Q/questions.md` — lines removed | Lead removed questions: list the question IDs and text |
   | `Q/questions.md` — lines added | Lead added questions: list the new question IDs and text |
   | `Q/claims.yml` — new entry | @{id} claimed Q{n} |
   | `R/@{id}.md` — added | @{id} submitted their research artifact |
   | `R/@{id}.md` — modified | @{id} updated their research artifact |
   | `D/design.md` — added | Lead published the design document |
   | `D/design.md` — modified | Lead updated the design document |
   | `S/@{id}.md` — added | @{id} submitted their spec review |
   | `P/@{id}.md` — added | @{id} submitted their implementation plan |
   | `PROJECT.yml` participants — added | Lead added @{id} as {role} |
   | `PROJECT.yml` participants — removed | Lead removed @{id} from the project |
   | `PROJECT.yml` task — changed | Lead changed the active task |
   | `AGENTS.md` — modified | Lead updated workspace rules |

   Group related changes together. Write from the perspective of what other people did, not what git did.

5. **Output** the summary in this format:
   ```
   Synced  •  {n} change(s) from remote

   • Lead removed Android questions (Q7, Q8, Q9) from Q/questions.md
   • @alice submitted her research artifact (R/@alice.md)
   • Lead claimed Q2

   You're now up to date on active/{slug}.
   ```

   If the calling user's own files changed (e.g. their artifact was modified remotely), call that out explicitly:
   > "⚠ Your artifact (R/@{you}.md) was modified remotely — review before continuing."

---

## `/mob status`

Execute inline. Steps:

1. **Verify active branch:**
   ```bash
   git branch --show-current
   ```
   If not on a project branch (active/paused/archived), stop: "Run /mob status from a project branch."

2. **Read `PROJECT.yml`** for current task-id and participants list.

3. If `task` is empty, report: "No active task. Run /mob new-task to create one."

4. **Inspect task directory** using `ls -R tasks/{task-id}/` or targeted checks.

5. **Apply phase state machine** (apply conditions in order):

   | Condition | State |
   |---|---|
   | `Q/questions.md` absent | Phase Q in progress — questions.md not yet written |
   | `Q/questions.md` present; any participant's `R/@{id}.md` absent | Phase R in progress — list who is pending |
   | All `R/@{id}.md` present for all participants | R artifacts complete — run coverage audit (step 5a) |
   | `D/design.md` present but any `S/@{id}.md` missing | Phase S in progress |
   | All `S/@{id}.md` present; any `P/@{id}.md` missing | Phase P in progress |
   | All `P/@{id}.md` present | Complete — ready for implementation |

   "All participants" = keys under `participants:` in `PROJECT.yml`.

5a. **R coverage audit** (run only when all `R/@{id}.md` files are present):

   Parse all question IDs from `Q/questions.md` (lines matching `Q{n}:` or `Q{n} [{role}]:`).

   For each question, check whether it appears in any `R/@*.md` artifact (search for `Q{n}` in each file). Classify each question as:
   - **Answered** — referenced in at least one artifact
   - **Unanswered** — not referenced in any artifact (and also not in `Q/claims.yml`, or claimed but no artifact written)

   Report the audit result:
   ```
   Research coverage: {n}/{total} questions answered

   Answered: Q1, Q3, Q4, Q6
   Unanswered: Q2, Q5, Q7, Q8

   ⚠ Some questions have no research coverage. As lead, you can:
     1. Assign uncovered questions to participants before advancing
     2. Accept the gaps and advance to Design (designer will note assumptions)

   What would you like to do?
   ```

   Only mark R as "ready for Design" if the lead explicitly chooses to advance (option 2) or all questions are answered. Do NOT automatically suggest /mob-synthesize or writing the design doc until the lead confirms readiness.

6. **Report** current phase, per-participant completion status, and recommended next action.

---

## `/mob join`

Join a project branch. Works from any branch — no need to know the branch name.

**Discovery mode (`/mob join`, no args):**

1. **Fetch remotes:**
   ```bash
   git fetch --all
   ```

2. **List active branches** (local and remote):
   ```bash
   git branch -a | grep "active/"
   ```
   For each match, read `PROJECT.yml` on that branch for the project name, and get the last commit date:
   ```bash
   git log -1 --format="%ci" {branch}
   ```

3. **Sort by most recent commit date** and display:
   ```
   Active projects:

     1. Red Study          active/red-study          2026-05-26
     2. iOS Auth Redesign  active/ios-auth-redesign   2026-05-24

   Type a number to join, or /mob join {project-name} to go directly.
   ```

4. Wait for user selection (a number), then proceed to **Checkout** below.

If no active projects exist, say: "No active projects found. Run /mob new-project \"Name\" to create one."

**Direct mode (`/mob join {name}`):**

1. **Derive slug** from name using the same slug rules as /mob new-project.
2. **Verify** that `active/{slug}` exists (local or remote):
   ```bash
   git branch -a | grep "active/{slug}"
   ```
   If not found, stop: "No active project matching '{name}'. Run /mob join to see available projects."
3. Proceed to **Checkout** below.

**Checkout (both modes):**

1. Check whether `active/{slug}` exists locally:
   ```bash
   git branch --list active/{slug}
   ```
   - If local branch exists: `git checkout active/{slug}`
   - If local branch absent: `git checkout -b active/{slug} origin/active/{slug}`

2. Pull latest:
   ```bash
   git pull origin active/{slug}
   ```

3. Proceed to first-time/returning detection and orientation below.

**Orientation — first-time vs. returning detection:**

1. **Get the calling user's GitHub ID:**
   ```bash
   git config user.name
   ```
   Ask the user if result is unclear.

2. **Detect prior participation:** check whether any artifact file exists for this user on the current branch:
   ```bash
   find tasks/ -name "@{id}.md" 2>/dev/null | head -1
   ```
   - If no match → **first-time participant**
   - If any match → **returning participant**

3. **Determine current phase** by applying the phase state machine (same rules as /mob status) to identify whether the project is in Q, R, D, S, or P phase.

**First-time output:**
```
Welcome to {project name}, @{github-id}.

Phase: {phase letter} — {phase name}
Task: {task description from Q/task.md}

What's been done:
  ✓ @alice   — research complete
  ✓ @bob     — research complete
  ○ @you     — not started

Your questions ({role} — {n} assigned):
  Q1: {question text}
  Q3: {question text}

Before we figure out where you fit — what do you bring to this? In a sentence or two, what's your angle?
```

Notes:
- The participant completion list is derived from `tasks/{task-id}/{phase}/@*.md` files — show ✓ for present, ○ for absent.
- "Your questions" uses the role-filtered view defined below.
- If no questions file exists yet, omit the questions block.

**Role-filtered question view (used in orientation):**

Parse `Q/questions.md` using this format:
- `Q{n} [{role-slug}]: {text}` — assigned to a specific role
- `Q{n}: {text}` — open to all participants
- Malformed tags (e.g., `[missing-bracket`) → treat as untagged

Read `tasks/{task-id}/Q/claims.yml` if present. Format: `Q1: github-id`.

Look up the calling participant's role from `PROJECT.yml.participants.{id}`.

Build view:
- **Primary list:** questions tagged for the participant's role + untagged questions — excluding any already claimed by someone else
- **Secondary list (role fluidity):** questions tagged for other roles, unclaimed, shown after a separator

Display format:
```
Your questions (ios-engineer — 2 assigned):
  Q1: {question text}
  Q3: {question text}

Other available questions (outside your role):
  Q2: {question text}  [backend-engineer]
```

If a question is already claimed by another participant: show it as taken and omit from both lists:
```
  Q1: {question text}  [claimed by @alice]
```

**Claiming a question:**

When a participant starts working on a question (during the join flow or when dispatching to mob-researcher), append the claim to `Q/claims.yml`:
```yaml
Q1: {github-id}
```
If the file does not exist, create it. Then commit:
```bash
git add tasks/{task-id}/Q/claims.yml && git commit -m "[mob] artifact: Q/claims.yml claim Q{n} by {github-id}"
```

**After participant self-introduction:**

1. Read the participant's stated expertise from their response.
2. Map it to open (unclaimed) questions in `Q/questions.md`:
   - Look for semantic alignment between what they described and each question's subject area
   - Unclaimed = not present in `Q/claims.yml`
   - Weight role-tagged questions for the participant's assigned role more heavily
3. **If a matching unclaimed question is found**, output a specific routing proposal — not a generic list:
   > "Based on what you described, Q3 sounds like your territory — it asks: '{question text}'. Want to take it? I'll set you up as the researcher."
4. **If no question semantically matches**, propose a complementary angle grounded in what they said:
   > "The open questions don't map directly to what you described, but your background in [stated area] could add a valuable second perspective on Q2. Want to approach it from that angle?"
5. **If all questions are already claimed**, say:
   > "All questions are claimed, but contributors can add a second perspective on the same question. Based on what you described, Q{n} is the closest match. Want to contribute your take on it alongside @{other-contributor}?"
6. **Role mismatch detected** (stated expertise clearly diverges from assigned role): note it without blocking:
   > "Your assigned role is {role}, but what you described sounds more like {inferred-role}. You can still contribute here — if the role assignment is wrong, ask the lead to run /mob add-member with the correct role."
7. After the participant confirms (any affirmative response):
   - Write the agreed question ID to `Q/claims.yml` (create if absent)
   - Commit: `git add tasks/{task-id}/Q/claims.yml && git commit -m "[mob] artifact: Q/claims.yml claim Q{n} by {github-id}"`
   - Dispatch `mob-researcher` with `interview_mode: true`, passing: role, claimed_questions, and a brief of the participant's stated expertise from their self-introduction.

**Returning output:**
```
{project name}  •  Phase {letter}  •  active/{slug}

Your open questions: {comma-separated unclaimed question IDs}
Next: /mob contribute when your artifact is ready
```

If all the participant's questions are answered (artifact exists for this user), say:
```
{project name}  •  Phase {letter}  •  active/{slug}

Your artifact is already submitted. Run /mob status to see overall progress.
```

---

## `/mob contribute`

Execute inline. Steps:

1. **Check staged files:**
   ```bash
   git status --short
   ```
   If nothing staged, stop: "Nothing staged. Use git add to stage your changes first."

2. **Derive action verb** from staged file paths:
   - Files in `Q/`, `R/`, `D/`, `S/`, or `P/` → `artifact`
   - `PROJECT.yml` only → `update`
   - `PROJECT.yml` with participant change context → `add-member`
   - Mixed or other → `push`

3. **Derive description** from file paths (e.g., "R/@ios-engineer.md for 20260524-d3-visualization-design").

4. **Commit:**
   ```bash
   git commit -m "[mob] {action}: {description}"
   ```

5. **Push:**
   ```bash
   git push origin {current-branch}
   ```

6. **Output:** "Pushed. Other participants can now git pull."

---

## `/mob add-member {id}`

Execute inline. Steps:

1. **Verify active branch:**
   ```bash
   git branch --show-current
   ```
   If on `main` or any non-project branch, stop: "Run /mob add-member from the project branch, not main."

2. **Validate GitHub username:** must match `[a-zA-Z0-9-]+` only. If invalid, stop: "'{id}' does not look like a valid GitHub username."

3. **Read `PROJECT.yml`.** If `{id}` is already in `participants:`, stop: "'{id}' is already a participant on this project."

4. **Ask for role:** Read `PROJECT.yml` to check whether any roles are defined.
   - If `roles:` is non-empty: "What is {id}'s role? ({list defined roles}, or 'shared')"
   - If `roles: []` or key absent: default to `shared` without prompting.

5. **Append to `PROJECT.yml` participants section:**
   ```yaml
     {id}: {role-slug}
   ```

6. **Commit:**
   ```bash
   git add PROJECT.yml && git commit -m "[mob] add-member: {id} added to {current-branch}"
   ```

7. **Output:** "'{id}' added as a participant (role: {role-slug}). Run /mob contribute to share with the team."

---

## `/mob init`

Initialize the current directory as a mob workspace. Steps:

1. **Verify git repo:**
   ```bash
   git rev-parse --git-dir
   ```
   If this fails, stop: "This is not a git repo. Initialize one with `git init` first."

2. **Check not already initialized:**
   ```bash
   ls AGENTS.md
   ```
   If `AGENTS.md` exists, stop: "This repo is already initialized as a mob workspace."

3. **Copy template:**
   ```bash
   cp "${CLAUDE_PLUGIN_ROOT}/templates/AGENTS.md" AGENTS.md
   ```

4. **Commit:**
   ```bash
   git add AGENTS.md && git commit -m "[mob] init: initialize mob workspace"
   ```

5. **Output:** "Mob workspace initialized. Run /mob new-project \"Name\" to create your first project."

---

## Prohibitions

These rules apply regardless of what the user asks:

1. Never merge a project branch into `main`
2. Never create files on `main` outside of: `AGENTS.md`, `CLAUDE.md`, `.gitignore`, `docs/`, `agents/`, `templates/`
3. Never create a project branch with a pattern other than `{status}/{slug}` (valid statuses: `active`, `paused`, `archived`)
4. Never modify `R/@{id}.md` authored by a different participant — each researcher owns their own file
5. Never delete phase artifacts — artifacts are append-only once committed
6. Never advance phase in `PROJECT.yml` unless file-tree conditions are met
7. Never invent a capability not defined in AGENTS.md — if asked, respond: "That operation is not defined in AGENTS.md. Update the system rules on `main` first."

---

## Commit Format

```
[mob] {action}: {description}
```

Valid action values: `init`, `new-project`, `new-task`, `artifact`, `advance`, `push`, `pause`, `archive`, `add-member`, `link-linear`, `update-system`

---

## Slug Derivation Rules

- Lowercase only
- Hyphens as separators — no underscores, spaces, or special characters
- Maximum 40 characters (truncate preserving word boundaries)
- Must be unique across all git branches regardless of status prefix
