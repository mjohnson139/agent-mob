# System Changelog

Changes to system-level files (`AGENTS.md`, `CLAUDE.md`, `team.yml`) are logged here.

---

## 2026-05-26 (U8: README and plugin.json path drift fix)

**Author:** mjohnson139
**Action:** `update-system`
Added README.md at the repo root — dual-audience (GitHub cold-reader and maintainer reference). Covers what Agent Mob is, QRSPI phases, installation, quick-start commands, how the workspace model works, plugin structure, and development conventions. Corrected stale `plugin.json` path in three places: CLAUDE.md plugin structure diagram, AGENTS.md repo structure diagram, and AGENTS.md What-NOT-to-Do list — all now reference the correct `plugins/agent-mob/.claude-plugin/plugin.json` location. Bumped plugin version 0.1.1 → 0.1.2.

---

## 2026-05-26 (U7: guided research mode — option 3 in /mob join orientation)

**Author:** mjohnson139
**Action:** `update-system`
Added "Guide me" as option 3 in the first-time participant orientation menu in SKILL.md. When selected, research runs inline in the main Claude session — no subagent dispatch. Menu text updated to include option 3 and prompt changed from "Type 1 or 2" to "Type 1, 2, or 3". Added "User picks 3 (Guide me)" handler: opens with a confirmed-claims greeting asking where to start, runs a conversational research loop with the same objective-findings-only rules as mob-researcher.md (no auto-advance, all forbidden phrasing applies), and writes the standard R/@{github-id}.md artifact at session end with answered questions only. Options 1 and 2 unchanged. Also bumped plugin version 0.1.0 → 0.1.1 and added version-bump-on-update-system rule to AGENTS.md Commit Conventions.

---

## 2026-05-26 (U6: mob-researcher — role context injection and contribute references)

**Author:** mjohnson139
**Action:** `update-system`
Updated mob-researcher.md startup sequence to support injected role context. When dispatched with `role` and `claimed_questions`, the researcher confirms the role and claimed question IDs at startup and skips the GitHub ID + workspace path prompts. When dispatched without role context (legacy path), the existing startup sequence is preserved unchanged. Added `briefing: true` dispatch flag: when present, researcher provides a 2-3 sentence task summary (from questions context, not Q/task.md) before beginning research. Research process updated: with role context, only claimed questions are answered and no empty sections are written for unclaimed questions; without role context, all questions are answered. The confirmation message in step 5 now names the assigned questions (or "all questions" for the legacy path).

---

## 2026-05-26 (U5: question tagging, role-filtered view, and claim state)

**Author:** mjohnson139
**Action:** `update-system`
Defined the inline role-tag format for questions.md in both SKILL.md and templates/AGENTS.md. Format: `Q{n} [{role-slug}]: {text}` for role-assigned questions, `Q{n}: {text}` for open questions visible to all. Malformed tags fall back to untagged. Added Q/claims.yml spec to AGENTS.md: append-only YAML `Q{n}: github-id`, absent key means unclaimed. Added role-filtered question view logic to SKILL.md join orientation: primary list (participant's role + untagged, excluding claimed-by-others); secondary list (other-role questions, shown after separator with role label); claimed questions shown as taken. Added claim write logic: on question selection, append to Q/claims.yml and commit. Backward compatible — projects with no role-tagged questions show all questions to all participants.

---

## 2026-05-26 (U4: specialist model — PROJECT.yml roles schema, new-project prompt, add-member role assignment)

**Author:** mjohnson139
**Action:** `update-system`
Extended the specialist model across three files. templates/PROJECT.yml: added `roles: []` field with comments explaining role slugs and the participant role-slug mapping (replacing the old scope comment). templates/AGENTS.md: updated PROJECT.yml schema block to show `roles:` field. SKILL.md new-project: added step 5 to ask "What specialist roles does this project need?" — derives role slugs from free-form input, writes `roles:` block if provided or `roles: []` if skipped; PROJECT.yml template now includes roles field; step numbers renumbered. SKILL.md add-member: replaced "scope" prompt with role-aware prompt — if project has defined roles, asks which role; if no roles defined, defaults to `shared` without prompting; output says "role" not "scope". mob-agent.md: same new-project and add-member changes mirrored. The specialist model is fully opt-in — existing projects without a `roles:` key behave as `roles: []`.

---

## 2026-05-26 (U3: first-time vs. returning detection and adaptive orientation)

**Author:** mjohnson139
**Action:** `update-system`
Added first-time/returning participant detection to the /mob join flow in SKILL.md. After checkout, the skill gets the calling user's GitHub ID, checks for any existing artifact (`find tasks/ -name "@{id}.md"`) to determine first-time vs. returning status, and applies the phase state machine to determine current phase. First-time participants see a guided orientation: project name, current phase, task description, per-participant completion list (✓/○), their assigned questions, and a choice between "Brief me" (dispatches mob-researcher with briefing flag) or "Let's go" (dispatches mob-researcher directly). Returning participants see a terse summary: project + phase header, open question IDs, and /mob contribute reminder. If the returning participant's artifact already exists, they see a completion message instead.

---

## 2026-05-26 (U2: join flow — branch-agnostic entry, discovery, and direct modes)

**Author:** mjohnson139
**Action:** `update-system`
Rewrote `/mob join` in SKILL.md to support branch-agnostic entry. Discovery mode (`/mob join`, no args): runs `git fetch --all`, lists all `active/*` branches (local and remote), reads PROJECT.yml on each branch for project name and last commit date, displays a numbered list sorted by most recent commit, and waits for user selection. Direct mode (`/mob join {name}`): derives slug, verifies `active/{slug}` exists locally or remotely, stops with a clear message if not found. Checkout path (both modes): handles local-branch-present vs. remote-only cases, then pulls latest. Participants no longer need to know branch names.

---

## 2026-05-26 (U1: command renames and backtick fix)

**Author:** mjohnson139
**Action:** `update-system`
Renamed participant commands throughout the plugin: `/mob fork` → `/mob join` and `/mob push` → `/mob contribute` in SKILL.md (section headers, command list, output strings), mob-agent.md (frontmatter description, section headers, output strings), mob-researcher.md (description trigger phrase, completion section), mob-designer.md (completion section), and templates/project-CLAUDE.md (Quick Reference section). Removed backtick wrapping from all slash commands in agent output instructions and participant-facing prose — output strings now use plain text slash commands (e.g., /mob contribute instead of `/mob contribute`). Also removed the `cd {mob-repo-path}` + manual git commands from mob-researcher completion, replacing with a single /mob contribute instruction.

---

## 2026-05-25 (plugin/workspace separation)

**Author:** mjohnson139
**Action:** `update-system`
Refactored plugin to separate workspace detection from plugin installation. Removed `.claude-plugin/plugin.json` from workspace checks in all three agents (`mob-agent.md`, `mob-researcher.md`, `mob-designer.md`) and the mob skill (`SKILL.md`) — workspace is now identified by `AGENTS.md` presence alone. Added `/mob init` subcommand to the skill: verifies git repo, checks for existing init, copies `${CLAUDE_PLUGIN_ROOT}/templates/AGENTS.md` into the workspace root, and commits. Added "Uninitialized workspace" soft-fail path in the skill: when `AGENTS.md` is absent, offer to run init rather than hard-failing. Removed `.claude-plugin/` from the allowed-on-main list in `plugins/agent-mob/templates/AGENTS.md` (both Branch Rules and Prohibitions sections) and from the mob skill Prohibitions. Rewrote root `AGENTS.md` as plugin development guidelines (not a mob workspace rulebook). Rewrote root `CLAUDE.md` with correct plugin structure under `plugins/agent-mob/`, correct `/mob <subcommand>` command syntax, and removed workspace-oriented sections (branch layout, QRSPI phase flow).

---

## 2026-05-24 (plugin MVP)

**Author:** mjohnson139
**Action:** `update-system`
Added Claude Code plugin MVP. New files: `.claude-plugin/plugin.json`, `agents/mob-agent.md`, `agents/mob-researcher.md`, `agents/mob-designer.md`, `templates/AGENTS.md`, `templates/PROJECT.yml`, `templates/project-CLAUDE.md`. Updated `AGENTS.md` to add `.claude-plugin/`, `agents/`, `templates/` to the allowed-on-main list. Updated `CLAUDE.md` with plugin structure and new quick-reference commands. Added `ios-engineer` to `team.yml`.

---

## 2026-05-24 (init)

**Author:** mjohnson139
**Action:** `init`
Initial repository setup. Created `AGENTS.md`, `CLAUDE.md`, `team.yml`, `.gitignore`. Established branch model, QRSPI phase rules, and agent prohibitions.
