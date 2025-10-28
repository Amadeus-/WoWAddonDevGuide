<!-- CLAUDE_SKIP_START -->
# How to Use This Guide with Claude Code

This document explains how to reference the WoW Addon Development Guide when working with Claude Code for World of Warcraft addon development. This document is NOT intended to be read by Claude Code for any sort of reference -- it's intended for human users only.

---

## Table of Contents

1. [Quick Reference (Simple Method)](#quick-reference-simple-method)
2. [Custom Command Setup (Advanced Method)](#custom-command-setup-advanced-method)
3. [Example Usage](#example-usage)
4. [Tips for Best Results](#tips-for-best-results)

---

## Quick Reference (Simple Method)

### Option 1: Direct Path Reference

Simply tell Claude Code to read the guide from wherever you've stored it:

```
I need help with WoW addon development. Please read the documentation at:
<path-to-guide>/WoW Addon Development Guide (AI Generated)

Then help me [describe your task].
```

**Example (replace the path with your actual location):**
```
I need help with WoW addon development. Please read the documentation at:
D:\Games\World of Warcraft\_retail_\Interface\+++WoW Addon Development Guide (AI Generated)+++

Then help me create an addon that tracks my gold across all characters.
```

**Common Locations:**
- `D:\Games\World of Warcraft\_retail_\Interface\+++WoW Addon Development Guide (AI Generated)+++`
- `C:\Program Files\World of Warcraft\_retail_\Interface\+++WoW Addon Development Guide (AI Generated)+++`
- `C:\Users\YourName\Documents\WoW Addon Development Guide`

### Option 2: Reference Specific Guides

If you know which guide you need, reference it directly:

```
Please reference the WoW Addon Guide at <path-to-guide>/WoW Addon Development Guide (AI Generated)/[FILENAME].md
to help me with [your task].
```

**Common Files:**
- `00_MASTER_PROMPT.md` - Master overview and entry point
- `QUICK_START_GUIDE.md` - 5-minute tutorial to get started
- `01_API_Reference.md` - WoW API functions
- `03_UI_Framework.md` - XML, frames, templates, mixins
- `05_Patterns_And_Best_Practices.md` - Coding patterns and best practices
- `07_Blizzard_UI_Examples.md` - Real working code examples
- `09_Addon_Libraries_Guide.md` - LibStub, Ace3, and other libraries

**Example (replace with your path):**
```
Please reference D:\Games\World of Warcraft\_retail_\Interface\+++WoW Addon Development Guide (AI Generated)+++\07_Blizzard_UI_Examples.md
to help me create action buttons for my addon.
```

### Option 3: Start from the Master Guide

For comprehensive help:

```
Please read the master guide at:
<path-to-guide>/WoW Addon Development Guide (AI Generated)/00_MASTER_PROMPT.md

This contains links to all other documentation. Then help me [your task].
```

---

## Custom Command Setup (Advanced Method)

### What is a Custom Command?

A custom Claude Code command allows you to type `/wow` instead of providing the full path each time. This is faster and more convenient for frequent WoW addon development.

### Setup Instructions

**Step 1: Locate Your Working Directory**

First, determine where you want to create the custom command. This should be a directory where you frequently work with WoW addons. For example:
- `D:\Games\World of Warcraft\_retail_\Interface\AddOns\` (if you work from the AddOns folder)
- `C:\Dev\WoW Addons\` (if you have a dedicated development folder)
- `D:\MyProjects\WoWDev\` (any custom location)

**Step 2: Create the Commands Directory**

In your chosen working directory, create the `.claude` subdirectory structure:

```
<your-working-directory>\
└── .claude\
    └── commands\
        └── wow.md
```

**Examples:**
```
D:\Games\World of Warcraft\_retail_\Interface\AddOns\.claude\commands\wow.md
C:\Dev\WoW\.claude\commands\wow.md
```

**Step 3: Create the Command File**

Create the `wow.md` file in the commands directory with this content (**update the paths to match your system**):

```markdown
You are an expert World of Warcraft addon developer with deep knowledge of Lua, WoW API, XML UI framework, and modern addon development patterns.

## Knowledge Base

**PRIMARY REFERENCE - Read these files as needed:**
- **Master Overview:** `<YOUR_PATH>\WoW Addon Development Guide (AI Generated)\00_MASTER_PROMPT.md`
- **Quick Start:** `<YOUR_PATH>\WoW Addon Development Guide (AI Generated)\QUICK_START_GUIDE.md`
- **API Reference:** `<YOUR_PATH>\WoW Addon Development Guide (AI Generated)\01_API_Reference.md`
- **Event System:** `<YOUR_PATH>\WoW Addon Development Guide (AI Generated)\02_Event_System.md`
- **UI Framework:** `<YOUR_PATH>\WoW Addon Development Guide (AI Generated)\03_UI_Framework.md`
- **Addon Structure:** `<YOUR_PATH>\WoW Addon Development Guide (AI Generated)\04_Addon_Structure.md`
- **Best Practices:** `<YOUR_PATH>\WoW Addon Development Guide (AI Generated)\05_Patterns_And_Best_Practices.md`
- **Data Persistence:** `<YOUR_PATH>\WoW Addon Development Guide (AI Generated)\06_Data_Persistence.md`
- **Working Examples:** `<YOUR_PATH>\WoW Addon Development Guide (AI Generated)\07_Blizzard_UI_Examples.md`
- **Community Patterns:** `<YOUR_PATH>\WoW Addon Development Guide (AI Generated)\08_Community_Addon_Patterns.md`
- **Libraries Guide:** `<YOUR_PATH>\WoW Addon Development Guide (AI Generated)\09_Addon_Libraries_Guide.md`
- **Advanced Techniques:** `<YOUR_PATH>\WoW Addon Development Guide (AI Generated)\10_Advanced_Techniques.md`
- **API Migration:** `<YOUR_PATH>\WoW Addon Development Guide (AI Generated)\11_API_Migration_Guide.md`

**IMPORTANT: When reading these files, IGNORE all content between `<!-- CLAUDE_SKIP_START -->` and `<!-- CLAUDE_SKIP_END -->` markers. These sections contain human-oriented content (version histories, navigation tables, motivational introductions, practice exercises, and learning path recommendations) that are not needed for AI code assistance. Only process the technical content outside these markers.**

**USER'S DIRECTORIES:**
- Production Addons: `D:\Games\World of Warcraft\_retail_\Interface\AddOns\`
- Addon Guide: `<YOUR_PATH>\WoW Addon Development Guide (AI Generated)\`

## Core Responsibilities

### 1. Addon Creation
- Write complete, working WoW addons in Lua
- Create proper TOC files with correct metadata
- Build XML UI using frames, widgets, and templates
- Follow Blizzard coding conventions and modern patterns
- Use Ace3, LibStub, and community libraries appropriately

### 2. Debugging
- Identify common WoW addon errors (nil references, API changes, event issues)
- Check for proper event registration and handlers
- Verify secure execution for combat-protected functions
- Validate XML syntax and frame inheritance
- Debug saved variables and data persistence

### 3. Code Quality
- Use mixins for code reuse
- Implement proper event bucketing for performance
- Follow namespace conventions (AddonName_FunctionName)
- Use local variables and upvalues for optimization
- Apply lazy loading and on-demand initialization

### 4. API Usage
- Use modern C_* namespaced APIs (not deprecated globals)
- Handle API changes across WoW versions
- Implement compatibility wrappers when needed
- Use proper event payloads and registration
- Apply secure templates for action buttons

## Critical Rules

**ALWAYS:**
- Check if APIs/events exist before using (`C_SomeAPI.SomeFunction ~= nil`)
- Use `ADDON_LOADED` event to initialize saved variables
- Localize global lookups for performance (`local GetTime = GetTime`)
- Use proper frame secure attributes for combat-protected actions
- Reference the comprehensive guide when uncertain
- Use `:` for method calls, `.` for property access
- Check `InCombatLockdown()` before restricted operations

**NEVER:**
- Use deprecated global API functions (use C_* equivalents)
- Access saved variables before `ADDON_LOADED` fires
- Modify protected frames during combat
- Create frame names that conflict with Blizzard UI
- Poll in `OnUpdate` when events can be used
- Assume API availability without version checks

## Workflow

1. **Understand requirements** - Clarify addon purpose and features
2. **Reference guide** - Read relevant sections for the task
3. **Check existing addons** - Look at user's production addons for patterns
4. **Apply modern patterns** - Use mixins, C_* APIs, event bucketing
5. **Test secure execution** - Ensure combat-protected code is proper
6. **Verify compatibility** - Check for API version requirements

## Code Style

Follow Blizzard conventions:
- PascalCase for addon names and mixins
- camelCase for local functions
- SCREAMING_CASE for constants
- Indentation: tabs (Blizzard standard)
- Comments for complex logic
- Localize globals at file top

Your goal is to help users create robust, performant, maintainable WoW addons using modern APIs and proven patterns from both Blizzard and community sources.

---

**Now help the user with their World of Warcraft addon development task.**
```

**Step 4: Verify the Command**

In Claude Code, type `/` and you should see `wow` in the list of available commands.

### Using the Custom Command

Once set up, simply type:

```
/wow
```

Then describe what you need help with. Claude will automatically load the documentation and assist you.

**Example:**
```
/wow

I need to create an addon that:
1. Tracks my gold across all characters
2. Shows a summary in a custom UI window
3. Adds a minimap icon to toggle the window
```

---

## Example Usage

### Example 1: Creating a New Addon

**Without Custom Command:**
```
I need help with WoW addon development. Please read:
<path-to-guide>\WoW Addon Development Guide (AI Generated)

Then help me create an addon that tracks profession cooldowns.
```
(Replace `<path-to-guide>` with your actual path, e.g., `D:\Games\World of Warcraft\_retail_\Interface`)

**With Custom Command:**
```
/wow

Help me create an addon that tracks profession cooldowns.
```

### Example 2: Debugging Existing Code

**Without Custom Command:**
```
Please reference <path-to-guide>\WoW Addon Development Guide (AI Generated)\05_Patterns_And_Best_Practices.md

My addon's saved variables aren't loading. Here's my code:
[paste code]
```
(Replace `<path-to-guide>` with your actual path)

**With Custom Command:**
```
/wow

My addon's saved variables aren't loading. Here's my code:
[paste code]
```

### Example 3: Learning a Specific Topic

**Without Custom Command:**
```
Please read <path-to-guide>\WoW Addon Development Guide (AI Generated)\09_Addon_Libraries_Guide.md

Teach me how to use Ace3 framework to create an options panel.
```
(Replace `<path-to-guide>` with your actual path)

**With Custom Command:**
```
/wow

Teach me how to use Ace3 framework to create an options panel.
```

### Example 4: Getting Started as a Beginner

**Without Custom Command:**
```
I'm new to WoW addon development. Please read:
<path-to-guide>\WoW Addon Development Guide (AI Generated)\QUICK_START_GUIDE.md
and
<path-to-guide>\WoW Addon Development Guide (AI Generated)\04_Addon_Structure.md

Then guide me through creating my first addon.
```
(Replace `<path-to-guide>` with your actual path)

**With Custom Command:**
```
/wow

I'm a complete beginner. Guide me through creating my first WoW addon.
```

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
/wow

Help with UI.
```

### Mention Your Experience Level

This helps Claude choose the right documentation:

```
/wow

I'm familiar with Lua but new to WoW addon development. I need help
understanding how to register and handle events.
```

### Reference Existing Code When Debugging

```
/wow

Here's my addon code:
[paste code]

It's throwing an error when I try to access saved variables. Can you help debug it?
```

### Ask for Specific Documentation Sections

```
/wow

I'm trying to understand the difference between CreateFrame and using
XML templates. Please reference the UI Framework guide and explain with examples.
```

### Request Modern Patterns

```
/wow

Show me the modern way to create action buttons. I want to use mixins
and templates like Blizzard does.
```

### Ask for Complete Examples

```
/wow

Give me a complete working example of an addon that uses saved variables
to store settings across sessions, with proper error handling.
```

---

## Common Tasks and Recommended Guides

| Task | Recommended Guide(s) |
|------|---------------------|
| **First time creating addon** | QUICK_START_GUIDE.md, 04_Addon_Structure.md |
| **API reference lookup** | 01_API_Reference.md, 00_MASTER_PROMPT.md |
| **Creating UI frames** | 03_UI_Framework.md, 07_Blizzard_UI_Examples.md |
| **Saving addon settings** | 06_Data_Persistence.md, 08_Community_Addon_Patterns.md |
| **Event handling** | 02_Event_System.md, 05_Patterns_And_Best_Practices.md |
| **Using Ace3 or LibStub** | 09_Addon_Libraries_Guide.md, 08_Community_Addon_Patterns.md |
| **Action buttons/tooltips** | 07_Blizzard_UI_Examples.md, 03_UI_Framework.md |
| **Understanding mixins** | 05_Patterns_And_Best_Practices.md, 03_UI_Framework.md |
| **Advanced patterns** | 10_Advanced_Techniques.md, 05_Patterns_And_Best_Practices.md |
| **Updating for new patch** | 11_API_Migration_Guide.md |
| **Scroll frames/lists** | 03_UI_Framework.md, 07_Blizzard_UI_Examples.md |
| **Cross-client compatibility** | 10_Advanced_Techniques.md |

---

## Troubleshooting

### Command Not Appearing

If `/wow` doesn't appear in the command list:

1. Verify the file exists at: `<your-working-directory>\.claude\commands\wow.md`
2. Restart Claude Code
3. Make sure the file has the proper frontmatter (the markdown content)
4. Check that the `.claude` directory is in your project root

### Claude Doesn't Load Documentation

If Claude doesn't seem to be using the guide:

1. Be explicit in your request: "Please read the WoW Addon Development Guide first"
2. Reference specific files when possible
3. Use the `/wow` command if you've set it up
4. Verify the path in your command file is correct (check the path you specified in the wow.md file)

### Getting Better Responses

- Provide context about what you're trying to accomplish
- Share relevant code snippets (Lua, XML, TOC files)
- Mention if you're getting specific error messages
- Indicate your experience level
- Ask for explanations of concepts you don't understand
- Specify which WoW version you're targeting (Retail, Classic, etc.)

---

## Quick Start Template

Copy and paste this template to get started:

### Without Custom Command:
```
I need help with WoW addon development. Please read the documentation at:
<path-to-guide>\WoW Addon Development Guide (AI Generated)

My experience level: [beginner/intermediate/advanced]

Task: [Describe what you want to do]

Current code (if any): [Paste code or write "starting from scratch"]

Specific questions: [List any specific questions]
```
(Replace `<path-to-guide>` with your actual path, e.g., `D:\Games\World of Warcraft\_retail_\Interface`)

### With Custom Command:
```
/wow

My experience level: [beginner/intermediate/advanced]

Task: [Describe what you want to do]

Current code (if any): [Paste code or write "starting from scratch"]

Specific questions: [List any specific questions]
```

---

## Additional Resources

### Within the Guide

- **README.md** - Complete overview with navigation by task
- **00_MASTER_PROMPT.md** - Master overview and entry point
- **QUICK_START_GUIDE.md** - 5-minute tutorial to get started

### File Locations

These are examples - your paths will vary based on where you installed WoW and where you stored the guide:

- **Guide Location:** `<path-to-guide>\WoW Addon Development Guide (AI Generated)\`
  - Example: `D:\Games\World of Warcraft\_retail_\Interface\+++WoW Addon Development Guide (AI Generated)+++\`
  - Example: `C:\Program Files\World of Warcraft\_retail_\Interface\+++WoW Addon Development Guide (AI Generated)+++\`
- **Command File (if using):** `<your-working-directory>\.claude\commands\wow.md`
  - Example: `D:\Games\World of Warcraft\_retail_\Interface\AddOns\.claude\commands\wow.md`
  - Example: `C:\Dev\WoW\.claude\commands\wow.md`
- **Your Addons:** Typically found in your WoW installation
  - Example: `D:\Games\World of Warcraft\_retail_\Interface\AddOns\`
  - Example: `C:\Program Files\World of Warcraft\_retail_\Interface\AddOns\`

---

**Last Updated:** 2025-10-27

---

*This guide is designed to help you make the most of the WoW Addon Development Guide when working with Claude Code. Whether you use the simple path reference method or set up the custom command, you'll have access to comprehensive documentation covering all aspects of WoW addon development.*
<!-- CLAUDE_SKIP_END -->
