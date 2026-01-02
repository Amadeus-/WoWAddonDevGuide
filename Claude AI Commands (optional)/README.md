# WoWAddon-Expert - Quick Start Guide

## Architecture

The WoW addon system uses a **coordinator/worker architecture**:

| Component | Role |
|-----------|------|
| `/wow` command (coordinator) | Handles user interaction, asks clarifying questions, delegates heavy work to the agent |
| `WoWAddon-Expert` agent (worker) | Does the actual work: reads documentation, analyzes addons, creates/edits files |

This conserves context in the main conversation while maintaining thoroughness.

---

## Installation

1. Copy `WoWAddon-Expert.md` to `~/.claude/agents/`
2. Copy `wow.md` to `~/.claude/commands/`
3. Edit `WoWAddon-Expert.md` and update the **Local Paths** section at the top:
   - `ADDONS_DIR`: Your WoW AddOns directory
   - `GUIDE_DIR`: Your WoW Addon Development Guide directory

> **Note:** The coordinator (`wow.md`) has no paths - only the agent needs updating.

---

## How to Use

Simply invoke `/wow` and describe what you need:

```
/wow "I need an addon that tracks my cooldowns"
```

The coordinator will:
1. Ask clarifying questions if needed
2. Delegate heavy work to the WoWAddon-Expert agent
3. Summarize results

You can also spawn the agent directly via Task tool:

```
Task(subagent_type="WoWAddon-Expert", prompt="Create a cooldown tracker...")
```

---

## What the Agent Does

- Reads your comprehensive guide (14 documentation files)
- Knows your directories (AddOns, Guide location)
- Follows best practices (Blizzard patterns, event handling, secure execution)
- Handles all tasks (creating, editing, debugging, refactoring addons)
- Applies correct API usage (C_* APIs, events, frames, XML UI)
- Ensures code quality (mixins, event bucketing, localized globals)
- Can spawn sub-subagents for complex research tasks

---

## Context Efficiency

The architecture is designed to conserve context:

- Coordinator stays lightweight (no large file reads)
- Agent reads documentation in isolated context
- Large addon analysis happens in agent context
- Only results/summaries return to main conversation

WoW has 1 large file (Events Index ~3,900 lines) that the agent will delegate to sub-subagents when needed.
