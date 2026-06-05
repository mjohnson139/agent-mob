---
title: "Designing and Building the Session Archiver Skill for Agent Mob"
date: 2026-05-28
tags: [agent-mob, claude-code, plugin-development, session-archiving, knowledge-management, architecture]
tools-used: [Read, Bash, Edit, Write]
stack: [Claude Code plugins, Markdown, YAML, GitHub]
outcome: working-pattern
session-type: architecture
---

# Session Recipe: Designing and Building the Session Archiver Skill for Agent Mob

## Starting Prompt

"I have this idea. An agent that will read a session's turns and describe the arc of the session, paying particular attention to 1) the starting prompt, 2) the tools used, 3) the interaction with the AI, 4) the outcome achieved. And then this session summary can be published in the workspace where others can use it as a basis for what they are doing."

## Arc

The session opened with an idea for a session arc analyzer and immediately required a fit assessment against the existing Agent Mob system (QRSPI phases, topics track). The first major pivot was storage: git was initially assumed, then correctly rejected in favor of an external knowledge store. The second pivot was the specific storage stack — Notion as document hub, open-brain as semantic search — which was then generalized after the user identified open-brain's immutability as a fragility. The final design landed on a storage-agnostic artifact format: the skill produces a well-structured markdown file and stops; users compose their own storage and search layer. Implementation followed quickly once the design was stable, with a late simplification removing the required name argument so the agent derives everything from the session itself.

## Key Turns

- **Turn 1:** User described the idea → assessed fit with existing system; identified topics track as closest analog but noted this was a different artifact type (recipe vs. observation)
- **Turn 2:** Proposed git-backed `recipes/` directory → user challenged it ("isn't a git repo, I made a bad choice") — pivoted to external storage
- **Turn 3:** Proposed Notion + open-brain architecture → user confirmed access to both MCPs but surfaced open-brain immutability as a deletion/update risk
- **Turn 4:** Resolved immutability concern: open-brain stores pointer (title + summary + Notion URL) not document; Notion is source of truth; stale pointers are detectable and acceptable for append-mostly recipes
- **Turn 5:** User asked to "cut out the open brain and notion pieces, make this generic" → pivoted to storage-agnostic design; artifact format becomes the contract, storage is user-composed
- **Turn 6:** Format agreed (6-section markdown with YAML frontmatter) → implemented skill file, version bump, changelog
- **Turn 7:** User simplified invocation — remove required name argument; AI derives title, filename, tags, stack from session content

## Tools Used

Read (×5 to understand existing agents, skill format, plugin.json, changelog, templates) → Bash (git log, find, mkdir) → Write (new SKILL.md) → Edit (×3 for invocation simplification, plugin.json version bump, changelog entry) → Bash (git branch/commit/push, gh pr create)

## Outcome

New `/capture-session` skill shipped as PR #16 (`feat/capture-session-skill` branch). Skill file at `plugins/agent-mob/skills/capture-session/SKILL.md`. Plugin version bumped 0.1.3 → 0.1.4. Changelog entry U10 added. Skill is storage-agnostic, workspace-free, zero-argument invocation.

## Reuse Guidance

When designing a new Agent Mob skill, start by reading `AGENTS.md`, `plugins/agent-mob/agents/mob-agent.md`, and the existing skill at `plugins/agent-mob/skills/mob/SKILL.md` to understand conventions before writing anything. The pattern for a new skill: create `plugins/agent-mob/skills/{name}/SKILL.md` with a YAML frontmatter `description` block that includes trigger phrases, then bump `plugins/agent-mob/.claude-plugin/plugin.json` version and add a changelog entry to `docs/SYSTEM_CHANGELOG.md` in the same commit.

Watch out for the workspace guard: most mob skills check for `AGENTS.md` before doing anything. If the new skill should work outside a mob workspace (like this one), explicitly state "do not check for AGENTS.md" in the Prohibitions section — the default assumption is workspace-required.

For skills that interact with external MCPs (Notion, open-brain, Google Drive): keep the skill itself MCP-agnostic at design time. Design the artifact format first, confirm it's portable, then let users wire their own storage. Hardcoding a specific MCP creates a dependency that fragments the user base.

When the storage design debate happens: git is right for concurrent collaborative artifacts (code, research files, phase artifacts). Git is wrong for searchable knowledge bases. The signal that you've crossed the line: if the primary access pattern is "search for something relevant" rather than "commit and pull," it belongs outside git.
