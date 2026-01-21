# World of Warcraft Addon Development - Master Knowledge Base Prompt

## Table of Contents
1. [Purpose](#purpose)
2. [How to Use This Knowledge Base](#how-to-use-this-knowledge-base)
3. [WoW Addon Development Overview](#wow-addon-development-overview)
4. [Key Components](#key-components)
5. [Common Development Tasks](#common-development-tasks)
6. [File Organization Best Practices](#file-organization-best-practices)
7. [Required Reading for Context](#required-reading-for-context)
8. [Source Code Locations](#source-code-locations)
9. [Example Usage Prompt](#example-usage-prompt)

---

## Purpose
This directory contains comprehensive documentation and source code references for World of Warcraft addon development. Use this prompt when you need Claude to assist with creating, debugging, analyzing, or enhancing WoW addons.

<!-- CLAUDE_SKIP_START -->
## How to Use This Knowledge Base

### Quick Start
**New to WoW addon development?** Start with `QUICK_START_GUIDE.md`

### For Addon Development Assistance

When requesting addon development assistance, provide Claude with:

1. **This master prompt** - Start here for context
2. **Relevant specialized prompts** based on your task:
   - `01_API_Reference.md` - For WoW API function usage
   - `02_Event_System.md` - For event handling
   - `03_UI_Framework.md` - For frames, widgets, and XML
   - `04_Addon_Structure.md` - For TOC files, file organization
   - `05_Patterns_And_Best_Practices.md` - For common patterns, mixins, namespaces
   - `06_Data_Persistence.md` - For saved variables
   - `07_Blizzard_UI_Examples.md` - For official UI code examples
   - `08_Community_Addon_Patterns.md` - For community patterns, Ace3, LibStub
   - `09_Addon_Libraries_Guide.md` - For libraries (LibStub, Ace3, LibDataBroker, etc.)
   - `10_Advanced_Techniques.md` - For production-level patterns (cross-client, performance, multi-addon)
   - `11_Housing_System_Guide.md` - For player housing APIs and patterns (12.0+)
   - `12_API_Migration_Guide.md` - For version upgrades, API compatibility, migration patterns
   - `QUICK_START_GUIDE.md` - For getting started quickly

3. **Source file references** from the Blizzard UI source code
<!-- CLAUDE_SKIP_END -->

## WoW Addon Development Overview

### Current Version
- **Retail (Mainline)**: 12.0.0 (Midnight)
- **API Documentation Files**: 513+ comprehensive Lua files
- **Blizzard UI AddOns**: 281+ official addons with source code
- **Total Source Files Analyzed**: 3,417+ files

### CRITICAL: 12.0.0 "Addon Apocalypse" Changes

The Midnight expansion (12.0.0) introduced massive security changes that broke most combat-related addons. Key impacts:

**Secret Values System:**
- Many APIs now return "secret values" during combat that cannot be read by addons
- Damage meters, combat logs, and threat meters fundamentally changed
- New official APIs replace removed functionality: `C_DamageMeter`, `C_EncounterTimeline`, `C_EncounterWarnings`
- Addons must adapt to callback-based data retrieval for combat information

**Removed/Changed APIs:**
- Global action bar functions REMOVED - use `C_ActionBar` namespace
- Combat log parsing functions REMOVED - use `C_CombatLog` namespace
- Many `UnitX()` functions return secret values in combat
- Transmog system completely redesigned - old APIs deprecated

**New Official Features:**
- Built-in damage meter integration via `C_DamageMeter`
- Official encounter warnings system via `C_EncounterWarnings`
- Timeline recording via `C_EncounterTimeline`
- Player Housing system via `C_Housing` (see `11_Housing_System_Guide.md`)

### Core Technologies
- **Language**: Lua 5.1 (modified)
- **UI Definition**: XML (WoW-specific schema)
- **Configuration**: TOC (Table of Contents) files
- **API**: C_* namespaces (preferred) + legacy global functions
- **Event System**: Frame-based event registration + callback-based registration (12.0+)

### Development Patterns
1. **Mixin Pattern**: `CreateFromMixins()` - Composition over inheritance
2. **Namespace Pattern**: `C_*` for C API exposure, custom namespaces for organization
3. **Event-Driven**: Register events on frames, handle via `OnEvent`
4. **Saved Variables**: Declared in TOC, persisted globally or per-character
5. **Frame Templating**: XML templates with Lua mixin initialization
6. **Secret Value Handling** (12.0+): Check for secret values before using combat data

### Major API Namespaces

**Core Namespaces (All Versions):**
- `C_Timer` - Timer management
- `C_Map` - Map and zone information
- `C_ChatInfo` - Chat channel information
- `C_Item` - Item information
- `C_Spell` - Spell information
- `C_Container` - Bag/container management
- `C_QuestLog` - Quest tracking

**New/Changed in 12.0 (Midnight):**
- `C_ActionBar` - Action bar management (replaces global functions)
- `C_CombatLog` - Combat log access (replaces CLEU parsing)
- `C_DamageMeter` - Official damage meter data
- `C_EncounterTimeline` - Encounter recording/playback
- `C_EncounterWarnings` - Boss ability warnings
- `C_Housing` - Player housing system
- `C_Transmog` - Redesigned transmog system

**Deprecated/Removed in 12.0:**
- Global `GetActionInfo()`, `PickupAction()`, etc. - Use `C_ActionBar`
- Direct CLEU parsing for damage - Use `C_DamageMeter` or `C_CombatLog`
- `GetTransmogSlotInfo()` and related - Use `C_Transmog`

## Key Components

### 1. TOC (Table of Contents) File
Every addon requires a `.toc` file specifying metadata and file load order.

**Essential Fields:**
```
## Interface: 120000
## Title: Your Addon Name
## Author: Your Name
## Version: 1.0.0
## SavedVariables: GlobalSavedVar
## SavedVariablesPerCharacter: CharacterSavedVar
## Dependencies: Addon1, Addon2
## OptionalDeps: OptionalAddon
```

**File Load Order:** Files are loaded in the order listed (Lua then XML pairs)

### 2. Lua API Structure

**Namespace APIs** (Modern, required for 12.0+):
```lua
C_ChatInfo.GetChannelInfo(channelID)
C_Map.GetPlayerMapPosition()
C_Timer.After(seconds, callback)
-- New 12.0 namespaces:
C_DamageMeter.GetDamageSummary()
C_ActionBar.GetActionInfo(slot)
C_CombatLog.GetCurrentEventInfo()
C_Housing.GetPlotInfo()
```

**Global APIs** (Legacy, many deprecated/removed in 12.0):
```lua
UnitName("player")  -- Returns secret value in combat for enemy units
GetItemInfo(itemID)
CreateFrame("Frame", "name", parent, "template")
-- NOTE: Always check API existence before use for cross-version compatibility
```

**API Existence Checking (Required for 12.0 Compatibility):**
```lua
-- Always check before using potentially removed APIs
if C_ActionBar and C_ActionBar.GetActionInfo then
    local info = C_ActionBar.GetActionInfo(slot)
end
```

### 3. Event System

**Event Registration (Traditional):**
```lua
local frame = CreateFrame("Frame")
frame:RegisterEvent("PLAYER_LOGIN")
frame:RegisterEvent("COMBAT_LOG_EVENT_UNFILTERED")

frame:SetScript("OnEvent", function(self, event, ...)
    if event == "PLAYER_LOGIN" then
        local arg1, arg2 = ...  -- Event-specific payload
    end
end)
```

**Callback-Based Registration (12.0+ Modern Pattern):**
```lua
-- New callback system for certain event types
EventRegistry:RegisterCallback("EditMode.Enter", function()
    -- Handle edit mode
end, addonName)

-- Unregister when done
EventRegistry:UnregisterCallback("EditMode.Enter", addonName)
```

### 4. Frame and Widget Types

**Common Frame Types:**
- `Frame` - Base container
- `Button` - Clickable button
- `ScrollFrame` - Scrollable container
- `EditBox` - Text input
- `StatusBar` - Progress bar
- `Texture` - Image display
- `FontString` - Text display
- `Model` - 3D model display

### 5. XML UI Definition

**Basic Frame Template:**
```xml
<Ui xmlns="http://www.blizzard.com/wow/ui/"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://www.blizzard.com/wow/ui/
                        https://raw.githubusercontent.com/Gethe/wow-ui-source/live/Interface/AddOns/Blizzard_SharedXML/UI.xsd">
    <Frame name="MyAddonFrame" parent="UIParent" inherits="BasicFrameTemplate">
        <Size x="400" y="300"/>
        <Anchors>
            <Anchor point="CENTER"/>
        </Anchors>
    </Frame>
</Ui>
```

## Common Development Tasks

### Creating a New Addon
1. Create addon folder in `Interface/AddOns/AddonName/`
2. Create `AddonName.toc` file
3. Create main Lua file (e.g., `Core.lua`)
4. Optional: Create XML file for UI elements
5. Reload UI (`/reload`) to test

### Debugging
- `/dump variable` - Print variable value
- `/console scriptErrors 1` - Enable Lua error display
- `/eventtrace` - Monitor event firing
- `/fstack` - Identify frames under mouse
- `/framestack` - Show frame hierarchy
- Use `DevTool` or `Blizzard_DebugTools` addons

### Performance Optimization
- Minimize OnUpdate usage (use C_Timer instead)
- Cache API results when possible
- Unregister unused events
- Use local variables for frequently accessed globals
- Profile with `/run UpdateAddOnMemoryUsage()` and `GetAddOnMemoryUsage()`

### 12.0 Compatibility Best Practices

**Secret Values Handling:**
```lua
-- Check if value is a secret before using
local function SafeGetUnitName(unit)
    local name = UnitName(unit)
    if name and not C_SecretValue.IsSecretValue(name) then
        return name
    end
    return "Unknown"
end
```

**API Existence Checking:**
```lua
-- Always verify APIs exist for cross-version compatibility
local function GetActionBarInfo(slot)
    if C_ActionBar and C_ActionBar.GetActionInfo then
        return C_ActionBar.GetActionInfo(slot)
    elseif GetActionInfo then  -- Legacy fallback
        return GetActionInfo(slot)
    end
    return nil
end
```

**Combat Data Access (12.0+):**
```lua
-- Use official damage meter API instead of CLEU parsing
if C_DamageMeter then
    local summary = C_DamageMeter.GetDamageSummary()
    -- Process official data
end
```

## File Organization Best Practices

```
AddonName/
├── AddonName.toc          # Main TOC file
├── Core.lua               # Initialization and main logic
├── Config.lua             # Configuration and constants
├── Modules/               # Feature modules
│   ├── ModuleA.lua
│   └── ModuleB.lua
├── UI/                    # UI components
│   ├── MainFrame.xml
│   ├── MainFrame.lua
│   └── Templates.xml
├── Libs/                  # Embedded libraries (Ace3, LibStub, etc.)
│   └── LibStub.lua
└── Localization/          # Translations
    ├── enUS.lua
    └── deDE.lua
```

## Required Reading for Context

When asking Claude for addon development help, provide:

1. **For API usage**: Reference `01_API_Reference.md` or specific API documentation files
2. **For events**: Reference `02_Event_System.md` and the event list
3. **For UI work**: Reference `03_UI_Framework.md` and Blizzard examples
4. **For architecture**: Reference `05_Patterns_And_Best_Practices.md`
5. **For examples**: Reference Blizzard source code or community addon patterns in `08_Community_Addon_Patterns.md`
6. **For housing addons**: Reference `11_Housing_System_Guide.md` for C_Housing APIs
7. **For 12.0 migration**: Reference `12_API_Migration_Guide.md` for breaking changes

## Source Code Locations

- **WoW UI Source**: `D:\Games\World of Warcraft\_retail_\Interface\+wow-ui-source+\` (version-specific subdirectory)
- **Blizzard AddOns**: `.../+wow-ui-source+/Interface/AddOns/`
- **Community AddOns**: `D:\Games\World of Warcraft\_retail_\Interface\AddOns\`
- **API Documentation**: `.../Blizzard_APIDocumentationGenerated/*.lua` (513+ files)

<!-- CLAUDE_SKIP_START -->
## Example Usage Prompt

```
I need help creating/debugging/analyzing a World of Warcraft addon.

Context:
- Read 00_MASTER_PROMPT.md for overview
- Read 01_API_Reference.md for API functions
- Read 02_Event_System.md for event handling
- Read 05_Patterns_And_Best_Practices.md for coding patterns

Task: [Describe your specific addon task here]

Relevant addon source (if debugging): [Paste or reference your code]

Expected behavior: [What should happen]
Actual behavior: [What is happening]
```

## Next Steps

After reading this master prompt, Claude should:

1. Ask clarifying questions about the specific addon task
2. Request relevant specialized documentation files
3. Reference Blizzard source code examples when appropriate
4. Provide complete, working code with explanations
5. Follow WoW addon best practices and patterns
6. Consider performance, compatibility, and maintainability

---

**Version**: 2.0 - Based on WoW 12.0.0 (Midnight)
**Last Updated**: 2026-01-20
**Source Files Analyzed**: 3,417+ Blizzard UI files + 21,514+ community addon files
**Major Changes**: 12.0 "Addon Apocalypse" security updates, Secret Values system, new official APIs
<!-- CLAUDE_SKIP_END -->
