<!-- CLAUDE_SKIP_START -->
# How to Use This Guide with Claude Code

This document explains how to set up and use the WoW Addon Development Guide with Claude Code for World of Warcraft addon development. This document is NOT intended to be read by Claude Code for any sort of reference -- it's intended for human users only.

---

## Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [Installation](#installation)
3. [Using the /wow Command](#using-the-wow-command)
4. [How It Works](#how-it-works)
5. [Example Usage](#example-usage)
6. [Tips for Best Results](#tips-for-best-results)
7. [Alternative: Direct Path Reference](#alternative-direct-path-reference)
8. [Troubleshooting](#troubleshooting)

---

## Architecture Overview

The WoW Claude Code integration uses a **coordinator/worker architecture** designed to conserve context while maintaining thoroughness:

```
┌─────────────────────────────────────────────────────────────────┐
│                     Your Conversation                           │
│                                                                 │
│  You ──► /wow ──► Coordinator ──► WoWAddon-Expert Agent        │
│                       │                    │                    │
│                  (lightweight)        (isolated context)        │
│                       │                    │                    │
│                  Asks questions       Reads docs                │
│                  Plans approach       Analyzes code             │
│                  Summarizes           Creates/edits files       │
│                       │                    │                    │
│                       ◄────── Results ─────┘                    │
└─────────────────────────────────────────────────────────────────┘
```

**Benefits:**
- **Context Conservation**: Large documentation files are read in isolated agent context, not your main conversation
- **Thoroughness**: The agent has full access to all 14 documentation files
- **Efficiency**: Coordinator stays lightweight; only results return to main conversation
- **Sub-subagent Support**: For very large research tasks, the agent can spawn additional subagents

---

## Installation

The command and agent files are located in `Claude AI Commands (optional)/` within this guide directory.

### Step 1: Copy the Agent File

Copy `Claude AI Commands (optional)/WoWAddon-Expert.md` to your Claude agents directory:

```
~/.claude/agents/WoWAddon-Expert.md
```

On Windows, this is typically:
```
C:\Users\<YourUsername>\.claude\agents\WoWAddon-Expert.md
```

### Step 2: Copy the Command File

Copy `Claude AI Commands (optional)/wow.md` to your Claude commands directory:

```
~/.claude/commands/wow.md
```

On Windows, this is typically:
```
C:\Users\<YourUsername>\.claude\commands\wow.md
```

### Step 3: Update the Agent's Local Paths

Edit `WoWAddon-Expert.md` and update the **Local Paths** section at the very top of the file:

```
## Local Paths (UPDATE THESE FOR YOUR SYSTEM)

ADDONS_DIR: D:\Games\World of Warcraft\_retail_\Interface\AddOns\
GUIDE_DIR:  D:\Games\World of Warcraft\_retail_\Interface\+++WoW Addon Development Guide (AI Generated)+++\
```

Change these paths to match where you have:
- **ADDONS_DIR**: Your WoW AddOns directory
- **GUIDE_DIR**: This WoW Addon Development Guide directory

**Note:** The coordinator file (`wow.md`) has no paths to update - only the agent file needs configuration.

### Step 4: Verify Installation

1. Start a new Claude Code session
2. Type `/` and look for `wow` in the command list
3. If it appears, you're ready to go!

---

## Using the /wow Command

Once installed, simply invoke the command and describe what you need:

```
/wow I need an addon that tracks my cooldowns and shows alerts.
```

The coordinator will:
1. **Ask clarifying questions** if your request is ambiguous
2. **Delegate the heavy work** to the WoWAddon-Expert agent
3. **Summarize the results** and ask if you need anything else

---

## How It Works

### The Coordinator (`/wow`)

The coordinator is a lightweight prompt that:
- Understands user requests
- Asks clarifying questions when needed
- Delegates actual work to the WoWAddon-Expert agent
- Synthesizes and summarizes results

It does NOT read documentation files directly, keeping your main conversation context clean.

### The Worker Agent (WoWAddon-Expert)

The agent runs in an isolated context and:
- Reads documentation files (all 14 guides)
- Analyzes existing addons
- Creates and edits addon files
- Debugs issues
- Has full edit authority

**Large File Handling:** The agent knows which documentation file is large (3,000+ lines) and can spawn sub-subagents to read it if needed:
- `events_extracted/00_EVENTS_INDEX.md` (~3,900 lines)

---

## Example Usage

### Example 1: Creating a New Addon

```
/wow

I need to create an addon that:
1. Tracks my gold across all characters
2. Shows a summary in a custom UI window
3. Adds a minimap icon to toggle the window
```

The coordinator will ask clarifying questions like:
- "Should it track gold per realm or account-wide?"
- "Do you want to use Ace3 or vanilla Blizzard UI?"
- "Should it include LibDataBroker support?"

Then it delegates to the agent to create the addon.

### Example 2: Debugging Existing Code

```
/wow

My addon's saved variables aren't loading. Here's my code:
[paste your code]

What's wrong?
```

The agent will analyze your code, reference the documentation, and identify the issue (likely accessing SavedVariables before ADDON_LOADED fires).

### Example 3: Learning a Specific Topic

```
/wow I want to learn how to use mixins and templates like Blizzard does. Show me examples.
```

The agent will read the relevant documentation and provide examples with explanations.

### Example 4: UI Development

```
/wow Help me create action buttons with cooldown spinners for my addon.
```

The agent will reference the UI Framework guide and create the code.

---

## Tips for Best Results

### Be Specific About Your Needs

**Good:**
```
/wow

I need an addon that monitors my target's health and displays it
in a custom status bar with configurable colors. I want to use
modern Blizzard UI patterns.
```

**Less Effective:**
```
/wow Help with UI.
```

### Mention Your Experience Level

```
/wow I'm familiar with Lua but new to WoW addon development. I need help
understanding how to register and handle events.
```

### Reference Existing Code When Debugging

```
/wow

Here's my addon code:
[paste code]

It's throwing an error when I try to access saved variables.
```

### Request Modern Patterns

```
/wow Show me the modern way to create action buttons. I want to use mixins
and templates like Blizzard does.
```

---

## Alternative: Direct Path Reference

If you don't want to set up the command/agent system, you can still reference the guide directly:

```
I need help with WoW addon development. Please read the documentation at:
D:\Path\To\WoW Addon Development Guide

Then help me [describe your task].
```

**However, this approach:**
- Consumes more context in your main conversation
- Requires specifying the path each time
- Doesn't benefit from the coordinator/worker architecture

The custom command setup is recommended for regular WoW addon development.

---

## Troubleshooting

### Command Not Appearing

If `/wow` doesn't appear in the command list:

1. Verify the file exists at: `~/.claude/commands/wow.md`
2. Restart Claude Code
3. Check that the file has proper markdown formatting

### Agent Not Working

If the agent doesn't seem to work properly:

1. Verify the agent file exists at: `~/.claude/agents/WoWAddon-Expert.md`
2. Check that the Local Paths section has valid paths
3. Ensure the GUIDE_DIR path points to this guide directory

### Documentation Not Found

If the agent reports it can't find documentation:

1. Verify GUIDE_DIR in the agent file points to the correct location
2. Check that all documentation files exist in that directory
3. Make sure paths don't have trailing spaces

---

## File Reference

### Files in `Claude AI Commands (optional)/`

| File | Purpose |
|------|---------|
| `wow.md` | Coordinator command - copy to `~/.claude/commands/` |
| `WoWAddon-Expert.md` | Worker agent - copy to `~/.claude/agents/` |
| `README.md` | Quick reference for installation |

### Documentation Files (14 total)

| File | Lines | Description |
|------|-------|-------------|
| `00_MASTER_PROMPT.md` | ~250 | Master overview |
| `QUICK_START_GUIDE.md` | ~310 | Quick start |
| `01_API_Reference.md` | ~400 | API reference |
| `02_Event_System.md` | ~470 | Event system |
| `03_UI_Framework.md` | ~1,220 | UI framework |
| `04_Addon_Structure.md` | ~1,630 | Addon structure |
| `05_Patterns_And_Best_Practices.md` | ~1,050 | Best practices |
| `06_Data_Persistence.md` | ~760 | Data persistence |
| `07_Blizzard_UI_Examples.md` | ~710 | Working examples |
| `08_Community_Addon_Patterns.md` | ~710 | Community patterns |
| `09_Addon_Libraries_Guide.md` | ~930 | Libraries guide |
| `10_Advanced_Techniques.md` | ~1,070 | Advanced techniques |
| `11_API_Migration_Guide.md` | ~630 | API migration |
| `events_extracted/00_EVENTS_INDEX.md` | ~3,900 | Events index (large) |

---

*This guide helps you make the most of the WoW Addon Development Guide when working with Claude Code. The coordinator/worker architecture ensures efficient context usage while providing comprehensive documentation access.*
<!-- CLAUDE_SKIP_END -->
