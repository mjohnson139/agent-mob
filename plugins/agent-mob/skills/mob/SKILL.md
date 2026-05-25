---
name: mob
description: Use when the user invokes /mob or any mob subcommand (new-project, new-task, status, fork, push, add-member), or asks about project phases, creating a mob project, checking task status, or collaborating via QRSPI workflow
---

# Agent Mob

Use the `mob-agent` (via the Agent tool) to handle all mob operations.

## Available Commands

| Command | What to invoke |
|---|---|
| `/mob new-project "Name"` | mob-agent → `/mob-new-project "Name"` |
| `/mob new-task "description"` | mob-agent → `/mob-new-task "description"` |
| `/mob status` | mob-agent → `/mob-status` |
| `/mob fork` | mob-agent → `/mob-fork` |
| `/mob push` | mob-agent → `/mob-push` |
| `/mob add-member {github-id}` | mob-agent → `/mob-add-member {id}` |

## If invoked with no arguments

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
  /mob fork                    Get your personal next-step instructions
  /mob push                    Commit staged files and push to origin
  /mob add-member {github-id}  Add a participant to the current project
```

If no project branches exist yet, omit the projects section and just show the commands.

## If invoked with a subcommand

Parse the first word, map it to the mob-agent command above, and invoke mob-agent with the Agent tool, passing the full original arguments.
