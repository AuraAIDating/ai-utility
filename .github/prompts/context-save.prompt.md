---
mode: agent
description: "Save working session context for your services. Captures git state, decisions made, remaining work, and open questions so any future Claude Code session can resume without losing a beat. Stores context files in .my-context/ within the repo. Pair with /context-restore."
tools: ["codebase", "githubRepo"]
---

Use the context-save skill from .claude/skills/context-save/SKILL.md to help with the following request:

$ARGUMENTS
