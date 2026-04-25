# WoW Addon Master Guide
**Complete API Reference and Navigation Hub**

**Language:** Lua 5.1 (modified)
**UI:** XML + FrameXML widgets
**Target Game:** World of Warcraft — Retail (Midnight / 12.0.5)

---

## Purpose of This Guide

This master guide is the **reference hub** for WoW addon development — version facts, key components (TOC, Lua API, event system, frames, XML), the major API namespaces, and documentation navigation across the rest of the knowledge base. It also includes a 5-minute "your first addon" tutorial so new developers have a single entry point.

**For the navigation index and learning paths:** see [README.md](README.md).
**For quick lookup and core concepts:** use this guide.

---

## Table of Contents

1. [Your First Addon - 5 Minute Tutorial](#your-first-addon---5-minute-tutorial)
2. [WoW Addon Development Overview](#wow-addon-development-overview)
3. [Critical 12.0.x Changes](#critical-120x-changes)
4. [12.0.5 Refinements (Current Patch)](#1205-refinements-current-patch)
5. [Core Technologies and Patterns](#core-technologies-and-patterns)
6. [Major API Namespaces](#major-api-namespaces)
7. [Key Components](#key-components)
8. [Common Development Tasks](#common-development-tasks)
9. [Quick Reference Snippets](#quick-reference-snippets)
10. [File Organization Best Practices](#file-organization-best-practices)
11. [Documentation Navigation](#documentation-navigation)
12. [File Paths Reference](#file-paths-reference)
13. [Getting Help](#getting-help)

---

## Your First Addon - 5 Minute Tutorial

### Step 1: Create Addon Folder

```
<WoW Install>\Interface\AddOns\MyFirstAddon\
```

### Step 2: Create TOC File

**MyFirstAddon.toc:**

```
## Interface: 120005
## Title: My First Addon
## Author: Your Name
## Version: 1.0.0
## Category: Miscellaneous
## SavedVariables: MyFirstAddonDB

MyFirstAddon.lua
```

Since Patch 10.1.0 you can support multiple WoW versions with comma-separated Interface values (if your code is compatible across them):

```
## Interface: 120005, 110207, 40402, 11508
```

See [04_Addon_Structure.md](04_Addon_Structure.md) for the full TOC directive reference.

### Step 3: Create Lua File

**MyFirstAddon.lua:**

```lua
-- Initialize saved variable
MyFirstAddonDB = MyFirstAddonDB or {}

-- Create event frame
local frame = CreateFrame("Frame")
frame:RegisterEvent("PLAYER_LOGIN")

-- Event handler
frame:SetScript("OnEvent", function(self, event)
    if event == "PLAYER_LOGIN" then
        print("My First Addon loaded!")

        -- Save player name
        MyFirstAddonDB.playerName = UnitName("player")
        print("Hello,", MyFirstAddonDB.playerName)
    end
end)

-- Slash command
SLASH_MYFIRSTADDON1 = "/mfa"
SlashCmdList["MYFIRSTADDON"] = function(msg)
    print("My First Addon - You typed:", msg)
end
```

### Step 4: Test

1. Save both files
2. Launch WoW
3. Type `/reload` in game
4. You should see "My First Addon loaded!"
5. Type `/mfa hello` to test the slash command

### Step 5: Next Steps

- Read [04_Addon_Structure.md](04_Addon_Structure.md) for TOC directives and file organization
- Read [06_Data_Persistence.md](06_Data_Persistence.md) for proper saved-variable handling
- Read [05_Patterns_And_Best_Practices.md](05_Patterns_And_Best_Practices.md) for coding patterns
- Add a simple UI frame — see [03_UI_Framework.md](03_UI_Framework.md)

---

## WoW Addon Development Overview

### Current Version

- **Retail (Mainline):** 12.0.5 (Midnight, current) — patch line: 12.0.0 → 12.0.1 → 12.0.5 → 12.0.7 (planned); no 12.0.2/3/4/6
- **Interface Version:** 120005
- **API Documentation Files:** 500+ comprehensive Lua files in `Blizzard_APIDocumentationGenerated/` (12.0.5 adds `AbbreviatedNumberFormatter`, `NumericRuleFormatter`, `SecondsFormatter`, `FrameAPINamePlate`, `EncounterEvents`, and others)
- **Blizzard UI Addons:** 281+ official addons with source code
- **Total Source Files Analyzed:** 3,417+

### Core Technologies

- **Language:** Lua 5.1 (modified, with WoW extensions like `table.freeze` (12.0.5), `table.create`, `table.count`)
- **UI Definition:** XML (WoW-specific schema)
- **Configuration:** TOC (Table of Contents) files
- **API:** `C_*` namespaces (preferred) + legacy globals (many deprecated/removed in 12.0)
- **Event System:** Frame-based event registration + callback-based registration (12.0+)

---

## Critical 12.0.x Changes

The Midnight expansion (12.0.0) introduced the largest API overhaul in WoW history. These are the changes that most addons need to address.

### "Addon Apocalypse" — Secret Values System

Combat-sensitive data is hidden from addons via "secret values":

- `UnitHealth()`, `UnitHealthMax()`, `UnitPower()`, `UnitPowerMax()` return SECRET values during combat
- `UnitGetTotalAbsorbs()`, `UnitGetIncomingHeals()`, `C_ActionBar.GetActionInfo()` (id field) also secret during combat
- Secret values CANNOT be used for Lua arithmetic, comparison, or string concatenation — they throw "attempt to compare (a secret value)" errors
- **Native `StatusBar` frames accept secret values directly** (handled at C++ level) — `bar:SetValue(UnitHealth(unit))` works
- **For custom bars / math:** use `UnitHealthPercent(unit, false, CurveConstants.ScaleTo100)` and `UnitPowerPercent(...)` — they return NON-SECRET 0-100 values
- **To check:** `issecretvalue(value)` returns true if the value is secret
- **For formatted display of any secret number:** `AbbreviateNumbers(value, { breakpointData = {...} })` (12.0.0, `AllowedWhenTainted`) returns a non-secret formatted string — more general than `UnitHealthPercent`/`UnitPowerPercent`. `C_StringUtil.TruncateWhenZero(n)` replaces `n > 0 and tostring(n) or ""` on secret numbers. See [12a_Secret_Safe_APIs.md](12a_Secret_Safe_APIs.md#abbreviatenumbers-general-purpose-secret-safe-numeric-formatter-1200).
- Traditional combat log parsing is blocked — `COMBAT_LOG_EVENT_UNFILTERED` registration fails with `ADDON_ACTION_FORBIDDEN`
- `C_DamageMeter` API data is SECRET-protected during combat — but workarounds exist: `pcall(string.format, "%.0f", secret)` extracts secret numbers as text; `StatusBar:SetValue(secret)` accepts secrets natively; array index from `combatSources` preserves sort order; after `PLAYER_REGEN_ENABLED` all values become readable
- **Unit tooltips:** `GameTooltip:GetUnit()` (a wrapper around `TooltipUtil.GetDisplayedUnit`) has no internal taint guards and can return secret `name` / `unit` / `guid`. The fix is to inline-reconstruct the helper's body and `canaccessvalue()`-guard each intermediate (`tooltipData` → `guid` → `unit` → `name`) — the obvious "guard the returns" approach fails because the comparison `unit ~= "none"` taints before any guard could run. This generalises to a broader technique: when a Blizzard convenience helper bundles secret-prone intermediates with no internal guards, replace the helper call with its source body and add per-step guards. See [12a_Secret_Safe_APIs.md](12a_Secret_Safe_APIs.md#unit-tooltip-identity-gametooltipgetunit-returns-secrets-and-the-helper-has-no-internal-guards).

**See [12a_Secret_Safe_APIs.md](12a_Secret_Safe_APIs.md) for the complete secret values API reference and workaround patterns.**

### API Migrations

Many globals moved to `C_*` namespaces in 12.0.0. The globals typically survive as deprecation shims gated by the `loadDeprecationFallbacks` CVar (disabled by default), so prefer the namespaced form:

| Old (Deprecated) | New (C_* namespace) |
|------------------|---------------------|
| `GetActionTexture()` | `C_ActionBar.GetActionTexture()` |
| `HasAction()` | `C_ActionBar.HasAction()` |
| `GetActionCount()` | `C_ActionBar.GetActionUseCount()` |
| `ActionHasRange()` | `C_ActionBar.HasRangeRequirements()` |
| `CombatLogGetCurrentEventInfo()` | `C_CombatLog.GetCurrentEventInfo()` |
| `GetSpellCooldown()` | `C_Spell.GetSpellCooldown()` |
| `GetSpellInfo(id)` (number only) | `C_Spell.GetSpellInfo(id)` (names no longer accepted) |
| `UnitAura()` | `C_UnitAuras.*` family |
| `GetTransmogSlotInfo()` | `C_Transmog.*` family |

**Note:** `PickupAction()` and `PlaceAction()` remain LIVE globals in 12.0.0 — they were NOT moved to `C_ActionBar`. See [12_API_Migration_Guide.md](12_API_Migration_Guide.md) for the complete migration reference.

### New TOC Directives (12.0.0+)

```
## Category: Combat                  -- Addon category for organization
## Group: MyAddonSuite               -- Group related addons
## LoadSavedVariablesFirst: 1        -- Load saved vars before code
## AllowAddOnTableAccess: 1          -- Allow other addons to read namespace table
```

### New Namespaces (12.0.0)

- `C_ActionBar` — Action bar management (replaces globals)
- `C_CombatLog` — Combat log access (limited by secret values)
- `C_DamageMeter` — Official damage meter data (secret-protected during combat; see workarounds above)
- `C_EncounterTimeline` — Encounter recording / playback
- `C_EncounterWarnings` — Boss ability warnings
- `C_Housing` — Player housing system (see [11_Housing_System_Guide.md](11_Housing_System_Guide.md))
- `C_CooldownViewer` — Cooldown Manager system (see [13_Cooldown_Viewer_Guide.md](13_Cooldown_Viewer_Guide.md))
- `C_EncodingUtil` — Native compression / serialization (11.1.5+)
- `C_RestrictedActions` — Addon restriction state queries (`CheckAllowProtectedFunctions`, `GetAddOnRestrictionState`, `IsAddOnRestrictionActive`)

---

## 12.0.5 Refinements (Current Patch)

Changes added in 12.0.5 on top of the 12.0.0/12.0.1 baseline. See [12_API_Migration_Guide.md](12_API_Migration_Guide.md) for the full delta.

- **Numeric formatters for secret values:** `AbbreviatedNumberFormatter`, `NumericRuleFormatter`, `SecondsFormatter` plus `Cooldown:SetCountdownFormatter(formatter)` and `Cooldown:SetCountdownMillisecondsThreshold(seconds)` — reduces reliance on `pcall(string.format)` for secret-number display.
- **Per-plate nameplate hit rect:** `nameplate:SetHitTestPoints(...)`, `:SetAllHitTestPoints(region)`, `:CanChangeHitTestPoints()`, `:ClearAllHitTestPoints()`, `:GetHitTestPoints()` — replaces the global `C_NamePlate.SetNamePlateSize` workaround.
- **Aura classification fields no longer secret:** `isHelpful`, `isHarmful`, `isRaid`, `isNameplateOnly`, `isFromPlayerOrPlayerPet` can be read during combat. Identifier fields (`name`, `spellId`, `icon`, durations) remain secret.
- **`auraInstanceID` re-randomizes** at encounter / M+ / PvP boundaries — invalidate any instance-id-keyed cache on `ENCOUNTER_START`, `CHALLENGE_MODE_START`, and arena/battleground boundaries.
- **`UnitIsUnit` restrictions:** comparisons involving `targettarget`, `focustarget`, or nameplate tokens may return `nil` or secret. See migration guide for the comparison allow-list.
- **`table.freeze(t)` / `table.isfrozen(t)`** for read-only tables.
- **New `"outfit"` action type** for `SecureActionButtonTemplate` — applies transmog outfits.
- **Misc fixes:** `GetRaidRosterInfo` returns `"Unknown"` instead of `nil` for uncached names; `UNIT_CONNECTION` reliability fixed; `IsInInstance` returns `true` inside delves; throttles relaxed during `PLAYER_LOGOUT` / `ADDONS_UNLOADING`.
- **New `Predicates` table** in generated API docs describes per-API secret/taint restrictions.

---

## Core Technologies and Patterns

### Development Patterns

1. **Mixin Pattern** — `CreateFromMixins()` for composition over inheritance. See [05_Patterns_And_Best_Practices.md](05_Patterns_And_Best_Practices.md) (Mixin Patterns).
2. **Namespace Pattern** — `C_*` for C API exposure; custom `MyAddon = {}` namespaces for organization.
3. **Event-Driven** — Register events on frames, handle via `OnEvent`. See [02_Event_System.md](02_Event_System.md).
4. **Saved Variables** — Declared in TOC, persisted per-account or per-character. See [06_Data_Persistence.md](06_Data_Persistence.md).
5. **Frame Templating** — XML templates with Lua mixin initialization. See [03_UI_Framework.md](03_UI_Framework.md).
6. **Secret Value Handling** (12.0+) — `issecretvalue()` guards, percentage APIs for math, native `StatusBar` for display.

---

## Major API Namespaces

### Core Namespaces (all versions)

| Namespace | Purpose |
|-----------|---------|
| `C_Timer` | Delayed execution, `After`, `NewTimer`, `NewTicker` |
| `C_Map` | Map/zone info, player position |
| `C_ChatInfo` | Chat channels, addon message prefixes |
| `C_Item` | Item info (replaces many `GetItemInfo*` globals) |
| `C_Spell` | Spell info (replaces `GetSpellInfo`, `GetSpellCooldown` globals) |
| `C_Container` | Bag/container access |
| `C_QuestLog` | Quest tracking and queries |
| `C_UnitAuras` | Aura queries (replaces `UnitAura`) |

### New/Changed in 12.0 (Midnight)

| Namespace | Purpose |
|-----------|---------|
| `C_ActionBar` | Action bar management (replaces globals) |
| `C_CombatLog` | Combat log access (secret-protected during combat) |
| `C_DamageMeter` | Official damage meter data |
| `C_EncounterTimeline` | Encounter recording and playback |
| `C_EncounterWarnings` | Boss ability warnings |
| `C_Housing` | Player housing system |
| `C_CooldownViewer` | Cooldown Manager system |
| `C_Transmog` | Redesigned transmog system |
| `C_EncodingUtil` | Native compression and serialization |
| `C_RestrictedActions` | Addon restriction state queries |

Full per-function details live in [01_API_Reference.md](01_API_Reference.md).

---

## Key Components

### 1. TOC (Table of Contents) File

Every addon requires a `.toc` file specifying metadata and file load order.

**Essential fields:**

```
## Interface: 120005
## Title: Your Addon Name
## Author: Your Name
## Version: 1.0.0
## SavedVariables: GlobalSavedVar
## SavedVariablesPerCharacter: CharacterSavedVar
## Dependencies: Addon1, Addon2
## OptionalDeps: OptionalAddon
## Category: Combat
```

**File load order:** Files are loaded in the order listed (Lua then XML pairs). See [04_Addon_Structure.md](04_Addon_Structure.md) for full directive reference and cross-client TOC patterns.

### 2. Lua API Structure

**Namespace APIs (preferred in 12.0+):**

```lua
C_ChatInfo.GetChannelInfo(channelID)
C_Map.GetPlayerMapPosition(mapID, unit)
C_Timer.After(seconds, callback)
C_ActionBar.GetActionTexture(slot)
C_Spell.GetSpellCooldown(spellID)
C_Housing.GetPlotInfo()
```

**Global APIs (legacy; many deprecated or removed in 12.0):**

```lua
UnitName("player")       -- OK; secret for some units during combat
GetItemInfo(itemID)      -- Still live; prefer C_Item
CreateFrame("Frame", "name", parent, "template")
```

**API existence checking (required for cross-version compatibility):**

```lua
if C_ActionBar and C_ActionBar.GetActionTexture then
    local texture = C_ActionBar.GetActionTexture(slot)
end
```

### 3. Event System

**Traditional frame-based registration:**

```lua
local frame = CreateFrame("Frame")
frame:RegisterEvent("PLAYER_LOGIN")
frame:RegisterEvent("PLAYER_ENTERING_WORLD")

frame:SetScript("OnEvent", function(self, event, ...)
    if event == "PLAYER_LOGIN" then
        -- Event payload in ...
    end
end)
```

**Callback-based registration (12.0+ modern pattern):**

```lua
EventRegistry:RegisterCallback("EditMode.Enter", function()
    -- Handle edit mode
end, "MyAddonContext")

EventRegistry:UnregisterCallback("EditMode.Enter", "MyAddonContext")
```

See [02_Event_System.md](02_Event_System.md) for the complete event reference and `C_EventUtils.IsEventValid` version-safe registration pattern.

### 4. Frame and Widget Types

| Type | Purpose |
|------|---------|
| `Frame` | Base container |
| `Button` | Clickable button |
| `ScrollFrame` / `ScrollBox` | Scrollable container (prefer ScrollBox in 11.0+) |
| `EditBox` | Text input |
| `StatusBar` | Progress bar (accepts secret values natively) |
| `Texture` | Image display |
| `FontString` | Text display |
| `Cooldown` | Cooldown swipe overlay (12.0.5: `SetCountdownFormatter` accepts secret numbers) |
| `Model` / `ModelScene` | 3D model display |

Full widget method reference in [03_UI_Framework.md](03_UI_Framework.md).

### 5. XML UI Definition

**Minimal frame template:**

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

---

## Common Development Tasks

### Creating a New Addon

1. Create `Interface/AddOns/AddonName/`
2. Create `AddonName.toc` with interface version, title, and saved-variable declarations
3. Create main Lua file (e.g., `Core.lua`)
4. Optional: create XML file for UI elements
5. `/reload` to test

### Debugging

- `/dump variable` — Print variable value
- `/console scriptErrors 1` — Enable Lua error display
- `/eventtrace` — Monitor event firing
- `/fstack` — Identify frame under mouse cursor
- `/framestack` — Show frame hierarchy
- Addons: **BugSack / BugGrabber** (error capture), **DevTool** (variable inspector), `Blizzard_DebugTools` (`/dump`, `/run`)

### Performance Optimization

- Minimize `OnUpdate` usage — prefer `C_Timer.NewTicker` or events
- Cache API results when possible; localize frequently-used globals
- Unregister unused events
- Profile with `UpdateAddOnMemoryUsage()` + `GetAddOnMemoryUsage(addonName)`
- For high-frequency events (e.g., `UNIT_AURA`), bucket with `AceBucket-3.0` or dirty-flag coalescing — see [10_Advanced_Techniques.md](10_Advanced_Techniques.md)

### 12.0 Compatibility Patterns

**Safe unit-name read (secret during combat for some units):**

```lua
local function SafeGetUnitName(unit)
    local name = UnitName(unit)
    if name and not issecretvalue(name) then
        return name
    end
    return "Unknown"
end
```

**Custom health bar using NON-SECRET percentage:**

```lua
local function UpdateCustomHealthBar(unit, barTexture, maxWidth)
    local percent = UnitHealthPercent(unit, false, CurveConstants.ScaleTo100) or 0
    barTexture:SetWidth(math.max(1, (percent / 100) * maxWidth))
end
```

**Native StatusBar accepts secrets directly:**

```lua
local function UpdateNativeStatusBar(unit, statusBar)
    statusBar:SetMinMaxValues(0, UnitHealthMax(unit))   -- Secret OK
    statusBar:SetValue(UnitHealth(unit))                -- Secret OK
end
```

**API existence check with legacy fallback:**

```lua
local function GetActionBarTexture(slot)
    if C_ActionBar and C_ActionBar.GetActionTexture then
        return C_ActionBar.GetActionTexture(slot)
    elseif GetActionTexture then
        return GetActionTexture(slot)
    end
    return nil
end
```

---

## Quick Reference Snippets

### Saved Variables

```lua
-- In TOC:
## SavedVariables: MyAddonDB

-- In code:
MyAddonDB = MyAddonDB or {}
MyAddonDB.setting = true

-- Defer initialization to ADDON_LOADED:
frame:RegisterEvent("ADDON_LOADED")
-- Handler: if event == "ADDON_LOADED" and arg1 == "MyAddon" then ... end
```

### Events

```lua
local frame = CreateFrame("Frame")
frame:RegisterEvent("PLAYER_LOGIN")

frame:SetScript("OnEvent", function(self, event, ...)
    if event == "PLAYER_LOGIN" then
        -- Your code
    end
end)
```

### API Calls

```lua
-- Unit info
local name = UnitName("player")
local health = UnitHealth("player")  -- SECRET during combat for enemies (12.0.0+)

-- Item info (C_Item in 12.0.0+)
local itemInfo = C_Item.GetItemInfo(itemID)

-- Spell info (C_Spell in 12.0.0+; names no longer accepted)
local spellInfo = C_Spell.GetSpellInfo(spellID)

-- Quest info
local quests = C_QuestLog.GetAllCompletedQuestIDs()

-- Action bar (C_ActionBar in 12.0.0+)
local texture = C_ActionBar.GetActionTexture(slot)
local hasAction = C_ActionBar.HasAction(slot)
```

### UI Frames

```lua
local frame = CreateFrame("Frame", "MyFrame", UIParent)
frame:SetSize(200, 100)
frame:SetPoint("CENTER")

local bg = frame:CreateTexture(nil, "BACKGROUND")
bg:SetAllPoints()
bg:SetColorTexture(0, 0, 0, 0.8)

local text = frame:CreateFontString(nil, "OVERLAY", "GameFontNormal")
text:SetPoint("CENTER")
text:SetText("Hello, World!")

frame:Show()
```

---

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

Full conventions in [04_Addon_Structure.md](04_Addon_Structure.md).

---

## Documentation Navigation

### Core Reference

| File | Purpose |
|------|---------|
| [01_API_Reference.md](01_API_Reference.md) | WoW API functions organized by category |
| [02_Event_System.md](02_Event_System.md) | Complete event list, payloads, registration patterns |
| [03_UI_Framework.md](03_UI_Framework.md) | XML, frames, widgets, templates, mixins, nameplates |
| [04_Addon_Structure.md](04_Addon_Structure.md) | TOC files, file organization, load order, dependencies |

### Patterns and Architecture

| File | Purpose |
|------|---------|
| [05_Patterns_And_Best_Practices.md](05_Patterns_And_Best_Practices.md) | Mixins, event patterns, performance, state management |
| [06_Data_Persistence.md](06_Data_Persistence.md) | Saved variables, profiles, migrations, database structure |
| [07_Blizzard_UI_Examples.md](07_Blizzard_UI_Examples.md) | Working examples from Blizzard source (action buttons, tooltips, scrolling, quest tracking) |
| [10_Advanced_Techniques.md](10_Advanced_Techniques.md) | Cross-client compatibility, event bucketing, profiling, multi-addon architecture |

### Libraries and Frameworks

| File | Purpose |
|------|---------|
| [08_Community_Addon_Patterns.md](08_Community_Addon_Patterns.md) | Community patterns, localization, slash commands, addon communication |
| [09_Addon_Libraries_Guide.md](09_Addon_Libraries_Guide.md) | Non-Ace3 libraries (LibStub, LibDataBroker, LibDBIcon, LibSharedMedia) |
| [09a_Ace3_Library_Guide.md](09a_Ace3_Library_Guide.md) | Comprehensive Ace3 framework reference (all 14+ libraries) |

### Specialized Systems (12.0+)

| File | Purpose |
|------|---------|
| [11_Housing_System_Guide.md](11_Housing_System_Guide.md) | Player housing APIs, furniture, decoration (`C_Housing`) |
| [12_API_Migration_Guide.md](12_API_Migration_Guide.md) | Version upgrades, API changes, 12.0.0 → 12.0.1 → 12.0.5 delta |
| [12a_Secret_Safe_APIs.md](12a_Secret_Safe_APIs.md) | Complete 12.0+ secret values reference, detection/manipulation functions, SecureTypes containers |
| [13_Cooldown_Viewer_Guide.md](13_Cooldown_Viewer_Guide.md) | Cooldown Manager: `C_CooldownViewer` API, alerts, layout serialization, `CooldownFrame` widget |

### Claude Code Integration (optional)

| File | Purpose |
|------|---------|
| [+How To Use This Guide with Claude Code+.md](+How%20To%20Use%20This%20Guide%20with%20Claude%20Code+.md) | Setup walkthrough for AI-assisted addon development |
| [Claude AI Commands (optional)/README.md](Claude%20AI%20Commands%20%28optional%29/README.md) | Agent and command file reference |

---

## File Paths Reference

### WoW UI Source (reference)

```
<WoW Install>\Interface\+wow-ui-source+ (<version>)\
```

Or online: [Gethe/wow-ui-source on GitHub](https://github.com/Gethe/wow-ui-source) (`live` branch for current retail).

### Your Addons

```
<WoW Install>\Interface\AddOns\
```

### Saved Variables

```
WTF\Account\[Account]\SavedVariables\                         (SavedVariables)
WTF\Account\[Account]\[Server]\[Character]\SavedVariables\    (SavedVariablesPerCharacter)
WTF\SavedVariables\                                           (SavedVariablesMachine — per installation)
```

### Error Log

```
WTF\Errors\
```

---

## Getting Help

### In-Game Commands

- `/reload` — Reload UI
- `/fstack` — Show frame under mouse
- `/framestack` — Show complete frame hierarchy
- `/eventtrace` — Monitor event firing
- `/dump <variable>` — Print variable
- `/console scriptErrors 1` — Enable Lua error display

### External Resources

See [README.md](README.md) (External Links section) for the full list. Quick references:

- [warcraft.wiki.gg](https://warcraft.wiki.gg/) — current community wiki (post-2023 Fandom migration)
- [Gethe/wow-ui-source](https://github.com/Gethe/wow-ui-source) — online mirror of Blizzard UI source
- [WoWUIDev Discord](https://discord.com/invite/txUg39Vhc6) — addon development community
- [CurseForge](https://www.curseforge.com/wow/addons) / [Wago.io](https://addons.wago.io/) — addon hosting and study material
