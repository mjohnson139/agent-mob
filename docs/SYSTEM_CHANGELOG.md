# System Changelog

Changes to system-level files (`AGENTS.md`, `CLAUDE.md`, `team.yml`) are logged here.

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
