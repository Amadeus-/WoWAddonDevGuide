# WoW Addon Development - Quick Start Guide

## Table of Contents
1. [Welcome](#welcome)
2. [How to Use This Knowledge Base](#how-to-use-this-knowledge-base)
3. [Complete Documentation Structure](#complete-documentation-structure)
4. [Your First Addon - 5 Minute Tutorial](#your-first-addon---5-minute-tutorial)
5. [Common Tasks - Quick Reference](#common-tasks---quick-reference)
6. [Getting Help](#getting-help)
7. [File Paths Reference](#file-paths-reference)
8. [Next Steps](#next-steps)

---

## Welcome!
<!-- CLAUDE_SKIP_START -->

This knowledge base contains everything you need to create, debug, and maintain World of Warcraft addons.
<!-- CLAUDE_SKIP_END -->

## How to Use This Knowledge Base
<!-- CLAUDE_SKIP_START -->

### For First-Time Addon Developers

**Read in this order:**
1. **Start Here:** `00_MASTER_PROMPT.md` - Overview of addon development
2. **Basics:** `04_Addon_Structure.md` - TOC files, file organization
3. **Data:** `06_Data_Persistence.md` - Saving settings between sessions
4. **Patterns:** `05_Patterns_And_Best_Practices.md` - How to write good code
5. **Practice:** `07_Blizzard_UI_Examples.md` - Real working examples

### For UI Development

**Read in this order:**
1. `03_UI_Framework.md` - XML, frames, templates, mixins
2. `07_Blizzard_UI_Examples.md` - Practical examples (action buttons, tooltips, etc.)
3. `05_Patterns_And_Best_Practices.md` - Best practices section

### For Specific Features

| I want to... | Read this file | Section |
|--------------|----------------|---------|
| **Create action buttons** | `07_Blizzard_UI_Examples.md` | Action Buttons |
| **Show buffs/debuffs** | `07_Blizzard_UI_Examples.md` | Buff/Debuff Frames |
| **Create a scrolling list** | `03_UI_Framework.md`, `07_Blizzard_UI_Examples.md` | ScrollBox Pattern |
| **Save addon settings** | `06_Data_Persistence.md` | All sections |
| **Track quests** | `07_Blizzard_UI_Examples.md` | Quest Tracking |
| **Use slash commands** | `08_Community_Addon_Patterns.md` | Slash Commands |
| **Create options menu** | `08_Community_Addon_Patterns.md` | Configuration |
| **Add tooltips** | `07_Blizzard_UI_Examples.md` | Tooltips |
| **Understand events** | `02_Event_System.md` | All sections |
| **Find API functions** | `01_API_Reference.md` | API categories |
| **Use mixins** | `05_Patterns_And_Best_Practices.md` | Mixin Patterns |
| **Handle errors** | `05_Patterns_And_Best_Practices.md` | Error Handling |
| **Optimize performance** | `05_Patterns_And_Best_Practices.md` | Performance |

## Complete Documentation Structure

### Main Documentation Files

1. **00_MASTER_PROMPT.md** - Master overview and entry point
2. **01_API_Reference.md** - WoW API functions reference
3. **02_Event_System.md** - Event system and event reference
4. **03_UI_Framework.md** - XML, frames, widgets, templates
5. **04_Addon_Structure.md** - TOC files, file organization, load order
6. **05_Patterns_And_Best_Practices.md** - Coding patterns, best practices
7. **06_Data_Persistence.md** - Saved variables, database management
8. **07_Blizzard_UI_Examples.md** - Real-world code examples
9. **08_Community_Addon_Patterns.md** - Community patterns, Ace3, LibStub

### Supporting Files

- **file_lists/** - Categorized lists of Blizzard addons
- **api_extracted/** - API function lists and categories
- **events_extracted/** - Event lists and documentation

<!-- CLAUDE_SKIP_END -->
## Your First Addon - 5 Minute Tutorial

### Step 1: Create Addon Folder
```
D:\Games\World of Warcraft\_retail_\Interface\AddOns\MyFirstAddon\
```

### Step 2: Create TOC File
**MyFirstAddon.toc:**
```
## Interface: 110207
## Title: My First Addon
## Author: Your Name
## Version: 1.0.0
## SavedVariables: MyFirstAddonDB

MyFirstAddon.lua
```

### Step 3: Create Lua File
**MyFirstAddon.lua:**
```lua
-- Initialize saved variable
MyFirstAddonDB = MyFirstAddonDB or {};

-- Create event frame
local frame = CreateFrame("Frame");
frame:RegisterEvent("PLAYER_LOGIN");

-- Event handler
frame:SetScript("OnEvent", function(self, event)
    if event == "PLAYER_LOGIN" then
        print("My First Addon loaded!");

        -- Save player name
        MyFirstAddonDB.playerName = UnitName("player");
        print("Hello,", MyFirstAddonDB.playerName);
    end
end);

-- Slash command
SLASH_MYFIRSTADDON1 = "/mfa";
SlashCmdList["MYFIRSTADDON"] = function(msg)
    print("My First Addon - You typed:", msg);
end;
```

### Step 4: Test
<!-- CLAUDE_SKIP_START -->
1. Save both files
2. Launch WoW
3. Type `/reload` in game
4. You should see "My First Addon loaded!"
5. Type `/mfa hello` to test the slash command

### Step 5: Next Steps

Now that you have a working addon, learn more:
- **Read:** `04_Addon_Structure.md` to understand TOC files
- **Read:** `06_Data_Persistence.md` to properly save data
- **Read:** `05_Patterns_And_Best_Practices.md` for coding patterns
- **Try:** Adding a simple UI frame (see `03_UI_Framework.md`)
<!-- CLAUDE_SKIP_END -->

## Common Tasks - Quick Reference

### Debugging

```lua
-- Print to chat
print("Debug:", value);

-- Dump table
/dump MyTable

-- Enable error display
/console scriptErrors 1

-- Trace events
/eventtrace

-- Show frame stack
/fstack
```

### Saved Variables

```lua
-- Declare in TOC
## SavedVariables: MyAddonDB

-- Use in code
MyAddonDB = MyAddonDB or {};
MyAddonDB.setting = true;

-- Load event
frame:RegisterEvent("ADDON_LOADED");
```

### Events

```lua
-- Register event
frame:RegisterEvent("PLAYER_LOGIN");

-- Handle event
frame:SetScript("OnEvent", function(self, event, ...)
    if event == "PLAYER_LOGIN" then
        -- Your code
    end
end);
```

### API Calls

```lua
-- Unit info
local name = UnitName("player");
local health = UnitHealth("player");

-- Item info
local itemName = GetItemInfo(itemID);

-- Spell info
local spellName = GetSpellInfo(spellID);

-- Modern C_* API
local quests = C_QuestLog.GetAllCompletedQuestIDs();
```

### UI Frames

```lua
-- Create frame
local frame = CreateFrame("Frame", "MyFrame", UIParent);
frame:SetSize(200, 100);
frame:SetPoint("CENTER");

-- Add texture background
local bg = frame:CreateTexture(nil, "BACKGROUND");
bg:SetAllPoints();
bg:SetColorTexture(0, 0, 0, 0.8);

-- Add text
local text = frame:CreateFontString(nil, "OVERLAY", "GameFontNormal");
text:SetPoint("CENTER");
text:SetText("Hello, World!");

-- Show frame
frame:Show();
```

## Getting Help
<!-- CLAUDE_SKIP_START -->

### In-Game Commands
- `/help` - WoW help
- `/fstack` - Show frame stack (identify frames under mouse)
- `/eventtrace` - Monitor event firing
- `/framestack` - Show complete frame hierarchy
- `/dump variable` - Print variable value
- `/console scriptErrors 1` - Enable Lua error display

### Useful Addons for Development
- **BugSack** - Lua error capture
- **BugGrabber** - Error handler
- **DevTool** - Variable inspector
- **Blizzard_DebugTools** - Built-in debug tools (`/dump`, `/run`)

### External Resources
- **Wowpedia** - https://wowpedia.fandom.com/wiki/World_of_Warcraft_API
- **WoW AddOn Discord** - Community support
- **GitHub** - Search for addon examples
- **CurseForge/Wago** - Download and study popular addons
<!-- CLAUDE_SKIP_END -->

## File Paths Reference
<!-- CLAUDE_SKIP_START -->

### WoW UI Source
```
D:\Games\World of Warcraft\_retail_\Interface\+wow-ui-source+ (11.2.7)\
```

### Your Addons
```
D:\Games\World of Warcraft\_retail_\Interface\AddOns\
```

### Saved Variables
```
WTF\Account\[Account]\SavedVariables\
WTF\Account\[Account]\[Server]\[Character]\SavedVariables\
```

### Error Log
```
WTF\Errors\
```
<!-- CLAUDE_SKIP_END -->

## Next Steps
<!-- CLAUDE_SKIP_START -->

1. **Build something small** - Start with a simple addon
2. **Study examples** - Read Blizzard addon code in `07_Blizzard_UI_Examples.md`
3. **Learn patterns** - Study `05_Patterns_And_Best_Practices.md`
4. **Experiment** - Try different features
5. **Ask for help** - WoW addon community is helpful!

<!-- CLAUDE_SKIP_END -->
---

## Ready to Start?
<!-- CLAUDE_SKIP_START -->

1. ✅ Read this guide
2. ✅ Create your first addon (5-minute tutorial above)
3. ✅ Test it in-game
4. ✅ Read `00_MASTER_PROMPT.md` for full overview
5. ✅ Study specific topics as needed

**Happy coding!**

---

**Version:** 1.0 - Based on WoW 11.2.7 (The War Within)
**Last Updated:** 2025-10-19
<!-- CLAUDE_SKIP_END -->
