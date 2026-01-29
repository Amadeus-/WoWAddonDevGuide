# WoW API Migration & Version Compatibility Guide

## Table of Contents
1. [Overview](#overview)
2. [Where to Find API Changes](#where-to-find-api-changes)
3. [Major API Changes by Expansion](#major-api-changes-by-expansion)
   - [Midnight (12.0.0+)](#midnight-1200---the-addon-apocalypse)
   - [The War Within Patches (11.0.2 - 11.2.7)](#the-war-within-patches-1102---1127)
   - [The War Within (11.0.0+)](#the-war-within-1100)
   - [Dragonflight (10.0.0 - 10.2.7)](#dragonflight-1000---1027)
   - [Shadowlands (9.0.0+)](#shadowlands-900)
4. [Common Migration Patterns](#common-migration-patterns)
5. [Version Detection & Compatibility](#version-detection--compatibility)
6. [Automated Migration Checklist](#automated-migration-checklist)
7. [Testing Across Versions](#testing-across-versions)
8. [Real-World Migration Examples](#real-world-migration-examples)
9. [Migration Quick Reference Table](#migration-quick-reference-table)

---

## Overview

When Blizzard releases new WoW expansions or major patches, addon APIs frequently change. Functions get deprecated, removed, or replaced with new alternatives. This guide helps you:

- Find official API change documentation
- Understand common migration patterns
- Create version-compatible code
- Migrate existing addons to new API versions

**Key Principle**: Always check API changes when a new expansion or major patch releases!

---

## Where to Find API Changes

### Official Resources

#### 1. Warcraft Wiki (PRIMARY SOURCE)
The most comprehensive and up-to-date resource for API changes.

**Main API Change Pages:**
```
https://warcraft.wiki.gg/wiki/Patch_[VERSION]/API_changes
```

**Examples:**
- Patch 12.0.0: https://warcraft.wiki.gg/wiki/Patch_12.0.0/API_changes
- Patch 11.0.0: https://warcraft.wiki.gg/wiki/Patch_11.0.0/API_changes
- Patch 11.0.2: https://warcraft.wiki.gg/wiki/Patch_11.0.2/API_changes
- Patch 10.2.5: https://warcraft.wiki.gg/wiki/Patch_10.2.5/API_changes

**What You'll Find:**
- New APIs added
- Deprecated APIs (still work, but scheduled for removal)
- Removed APIs (no longer work)
- Changed API signatures
- New namespaces (C_* APIs)

#### 2. WoW Forums - UI & Macro Forum
```
https://us.forums.blizzard.com/en/wow/c/ui-and-macro/
```

Search for threads like:
- "API changes in [patch]"
- "Deprecated API post [patch]"
- "Addon broke after [patch]"

#### 3. GitHub - WoW UI Source
```
https://github.com/Gethe/wow-ui-source
```

Browse commits to see changes between versions.

#### 4. CurseForge/Wago.io Addon Comments
Check popular addons' update notes for insights on what changed.

### Version History Quick Reference

| Expansion | Interface Version | Major API Changes |
|-----------|------------------|-------------------|
| **Midnight** | 12.0.0+ | Secret Values system, C_ActionBar, C_CombatLog, C_DamageMeter, 432 new APIs |
| **The War Within** | 11.0.0 - 11.2.7 | UIDropDownMenu deprecated, GetSpellInfo removed, C_Spell changes, Housing system |
| **Dragonflight** | 10.0.0 - 10.2.7 | UnitAura/UnitBuff/UnitDebuff deprecated, TextStatusBar changes |
| **Shadowlands** | 9.0.0 - 9.2.7 | C_Container namespace, many containerAPIs moved |
| **Battle for Azeroth** | 8.0.0 - 8.3.x | Major UI scale changes, backdrop changes |
| **Legion** | 7.0.0 - 7.3.x | Protected functions expanded, macro changes |

### Current Interface Version Numbers

Use these values in your `## Interface:` directive. Since **Patch 10.1.0**, you can use comma-separated values to support multiple versions in a single TOC file:

```
## Interface: 120000, 110207, 50503, 40402, 11508
```

| Game Client | Version | Interface Number |
|-------------|---------|-----------------|
| **Retail (Midnight)** | 12.0.0 | 120000 |
| **Retail (Midnight)** | 12.0.1 | 120001 |
| **Retail (TWW)** | 11.2.7 | 110207 |
| **Retail (TWW)** | 11.1.0 | 110100 |
| **Classic Cata** | 4.4.2 | 40402 |
| **Classic Cata** | 4.4.1 | 40401 |
| **Classic Cata** | 5.0.5 | 50503 |
| **Classic SoD/Era** | 1.15.8 | 11508 |
| **Wrath Classic** | 3.4.3 | 30403 |

**Comma-Separated Interface (10.1.0+):**
- One TOC file can declare compatibility with multiple WoW versions
- Use when your addon code is identical across versions
- Eliminates need for separate `_Mainline.toc`, `_Classic.toc` files
- Example: `## Interface: 120000, 40402, 11508` supports Retail + Cata Classic + Era

**When to use multiple TOC files instead:**
- Different Lua/XML files need to load per version
- Version-specific libraries required
- Major code path differences

---

## Major API Changes by Expansion

### Midnight (12.0.0+) - "The Addon Apocalypse"

Patch 12.0.0 is the largest API change since the original addon system was introduced. With 432 new APIs added and 140+ removed, this expansion fundamentally changes how addons interact with combat data.

#### SECRET VALUES SYSTEM (CRITICAL!)

The most significant change in 12.0.0 is the introduction of the **Secret Values** system. This system restricts addon automation during combat by making certain API returns unusable by tainted (addon) code.

> **For comprehensive documentation on secret values, see [12a_Secret_Safe_APIs.md](12a_Secret_Safe_APIs.md).**

**Quick Overview:**
- Secret values LOOK normal but CANNOT be used in arithmetic, comparisons, or concatenations during combat
- Use `issecretvalue(value)` to check if a value is secret
- Native StatusBar frames accept secret values at C++ level
- Use `UnitHealthPercent()` / `UnitPowerPercent()` for non-secret percentages

**Migration Strategy for Combat Addons:**
```lua
-- OLD approach (may fail with secret values):
local function OnUpdate()
    local spell = GetActionInfo(1)
    if spell == myTrackedSpell then  -- May fail!
        DoSomething()
    end
end

-- NEW approach (secret-value aware):
local function OnUpdate()
    local spell = GetActionInfo(1)
    if not issecretvalue(spell) and spell == myTrackedSpell then
        DoSomething()
    elseif issecretvalue(spell) then
        -- Value is secret, use alternative approach or defer
        ScheduleForOutOfCombat()
    end
end
```

> **See [12a_Secret_Safe_APIs.md](12a_Secret_Safe_APIs.md) for:**
> - Complete list of secret-safe functions (`issecretvalue`, `canaccessvalue`, `scrubsecretvalues`, etc.)
> - All APIs that return secret values (health, power, action bar, tooltip data)
> - All APIs that accept secret values (StatusBar, SetTimerDuration, CreateUnitHealPredictionCalculator)
> - Real-world patterns from Platynator nameplate addon
> - Testing CVars and debugging techniques

---

#### Nameplate Click Targeting Changes (12.0.0)

WoW 12.0.0 fundamentally changed how nameplate click targeting works. This is a **critical change** for any addon that customizes nameplates (TidyPlates, NeatPlates, Plater, KuiNameplates, etc.).

**The Problem:**

In 12.0.0, Blizzard moved click detection from Lua-level frame handling to C++ level using a `HitTestFrame` child of the nameplate's UnitFrame. This has major implications:

```lua
-- BEFORE 12.0.0: This was safe
local blizzardPlate = nameplate.UnitFrame
blizzardPlate:Hide()  -- Hide Blizzard's plate, show custom plate
-- Click targeting still worked through custom frames

-- AFTER 12.0.0: This BREAKS click targeting!
local blizzardPlate = nameplate.UnitFrame
blizzardPlate:Hide()  -- PROBLEM: Also hides the HitTestFrame child!
-- Result: Players cannot click nameplates to target units!
```

**Why It Breaks:**

The `HitTestFrame` is a child of `UnitFrame`. When you call `Hide()` on UnitFrame, it also hides all children, including the HitTestFrame that Blizzard's C++ code uses for click detection. Your custom nameplate frames do NOT receive click events for targeting - only the HitTestFrame does.

**The Solution:**

Use `SetAlpha(0)` instead of `Hide()` to make the Blizzard nameplate invisible while keeping the HitTestFrame active:

```lua
-- CORRECT approach for 12.0.0+
local function HideBlizzardPlate(nameplate)
    local unitFrame = nameplate.UnitFrame
    if unitFrame then
        -- Make invisible but keep HitTestFrame active for click targeting
        unitFrame:SetAlpha(0)

        -- Optionally disable mouseover highlights on Blizzard elements
        -- (the HitTestFrame still handles the actual click detection)
    end
end

local function ShowBlizzardPlate(nameplate)
    local unitFrame = nameplate.UnitFrame
    if unitFrame then
        -- Restore visibility
        unitFrame:SetAlpha(1)
    end
end

-- Example usage in nameplate addon:
local function OnNamePlateAdded(unit)
    local nameplate = C_NamePlate.GetNamePlateForUnit(unit)
    if nameplate then
        -- Hide Blizzard plate visually (but keep click targeting!)
        HideBlizzardPlate(nameplate)

        -- Create/show your custom plate
        ShowCustomPlate(nameplate, unit)
    end
end

local function OnNamePlateRemoved(unit)
    local nameplate = C_NamePlate.GetNamePlateForUnit(unit)
    if nameplate then
        -- Restore Blizzard plate
        ShowBlizzardPlate(nameplate)

        -- Hide your custom plate
        HideCustomPlate(nameplate)
    end
end
```

**New API (May Be Restricted):**

WoW 12.0.0 added a new API for registering custom hit test frames:

```lua
C_NamePlateManager.SetNamePlateHitTestFrame(frame)
```

However, this API may be restricted to Blizzard code only. Testing indicates it may not work for third-party addons. The `SetAlpha(0)` approach is the reliable solution.

**Migration Checklist for Nameplate Addons:**

1. Search your code for any `UnitFrame:Hide()` or `plate.UnitFrame:Hide()` calls
2. Replace with `UnitFrame:SetAlpha(0)`
3. Update show logic to use `UnitFrame:SetAlpha(1)` instead of `Show()`
4. Test click targeting on both friendly and hostile nameplates
5. Test in combat (where this is most critical)
6. Verify tab-targeting still works (separate system, but worth checking)

**Debug Tip:**

If click targeting stops working after your addon loads, check if the nameplate's UnitFrame or its children are hidden:

```lua
local function DebugNameplateHitTest(unit)
    local nameplate = C_NamePlate.GetNamePlateForUnit(unit)
    if nameplate and nameplate.UnitFrame then
        local uf = nameplate.UnitFrame
        print("UnitFrame shown:", uf:IsShown())
        print("UnitFrame alpha:", uf:GetAlpha())

        -- Check for HitTestFrame child
        for _, child in pairs({uf:GetChildren()}) do
            local name = child:GetName() or child:GetDebugName() or "unnamed"
            if name:find("HitTest") then
                print("HitTestFrame found:", name, "shown:", child:IsShown())
            end
        end
    end
end
```

---

#### C_ActionBar Namespace (Replaces Global Action Bar Functions)

> **Note:** C_ActionBar functions may return **secret values during combat**. See [12a_Secret_Safe_APIs.md](12a_Secret_Safe_APIs.md) for handling patterns.

```lua
-- OLD globals (REMOVED):
GetActionInfo(slot)
GetActionTexture(slot)
GetActionCooldown(slot)
GetActionCount(slot)
IsUsableAction(slot)
IsCurrentAction(slot)
IsAutoRepeatAction(slot)
IsAttackAction(slot)
PickupAction(slot)
PlaceAction(slot)
HasAction(slot)
ActionHasRange(slot)

-- NEW C_ActionBar namespace:
C_ActionBar.GetActionInfo(slot)       -- id may be SECRET during combat
C_ActionBar.GetActionTexture(slot)    -- May be SECRET during combat
C_ActionBar.GetActionCooldown(slot)
C_ActionBar.GetActionCount(slot)
C_ActionBar.IsUsableAction(slot)      -- May be SECRET during combat
C_ActionBar.IsCurrentAction(slot)
C_ActionBar.IsAutoRepeatAction(slot)
C_ActionBar.IsAttackAction(slot)
C_ActionBar.PickupAction(slot)
C_ActionBar.PlaceAction(slot)
C_ActionBar.HasAction(slot)
C_ActionBar.ActionHasRange(slot)

-- NEW functions in C_ActionBar:
C_ActionBar.GetActionBarPage()
C_ActionBar.SetActionBarPage(page)
C_ActionBar.GetBonusBarOffset()
C_ActionBar.GetOverrideBarSkin()
C_ActionBar.IsActionInRange(slot, unit)
```

**Migration Example:**
```lua
-- Compatibility wrapper for action bar addons
local GetActionInfo = C_ActionBar and C_ActionBar.GetActionInfo or GetActionInfo
local HasAction = C_ActionBar and C_ActionBar.HasAction or HasAction

-- Use wrapped functions
local actionType, id, subType = GetActionInfo(slot)
```

#### C_CombatLog Namespace (Replaces CombatLog Globals)

```lua
-- OLD globals (REMOVED):
CombatLogGetCurrentEventInfo()
CombatLogSetCurrentEntry(index)
CombatLogGetNumEntries()
CombatLogResetFilter()
CombatLogAddFilter()

-- NEW C_CombatLog namespace:
C_CombatLog.GetCurrentEventInfo()
C_CombatLog.SetCurrentEntry(index)
C_CombatLog.GetNumEntries()
C_CombatLog.ResetFilter()
C_CombatLog.AddFilter(...)

-- Additional new functions:
C_CombatLog.GetEntry(index)
C_CombatLog.GetEventCategories()
C_CombatLog.IsEventFiltered(eventType)
```

**Migration for Combat Log Addons:**
```lua
-- OLD (frame-based combat log reading):
local frame = CreateFrame("Frame")
frame:RegisterEvent("COMBAT_LOG_EVENT_UNFILTERED")
frame:SetScript("OnEvent", function()
    local timestamp, event, ... = CombatLogGetCurrentEventInfo()
end)

-- NEW (12.0.0+):
local frame = CreateFrame("Frame")
frame:RegisterEvent("COMBAT_LOG_EVENT_UNFILTERED")
frame:SetScript("OnEvent", function()
    local timestamp, event, ... = C_CombatLog.GetCurrentEventInfo()
end)
```

#### ⚠️ CRITICAL: Combat Log Events BLOCKED in 12.0.0 ⚠️

**This is the most important change in 12.0.0 for damage meter and combat addon developers.**

In WoW 12.0.0 (Midnight), Blizzard has **completely blocked third-party addon access to combat log events** as part of their "addon disarmament" initiative. This is NOT a bug - it is intentional.

**What Blizzard Officially Stated:**
> "Combat Log Events are no longer available to addons, and messages in the Combat Log chat tab have been converted to KStrings to prevent addons from parsing the information inside them."

**What This Means:**
```lua
-- THIS CODE WILL THROW ADDON_ACTION_FORBIDDEN IN 12.0.0:
local frame = CreateFrame("Frame")
frame:RegisterEvent("COMBAT_LOG_EVENT_UNFILTERED")  -- BLOCKED!

-- Error message you will see:
-- [ADDON_ACTION_FORBIDDEN] AddOn 'YourAddon' tried to call the protected function 'Frame:RegisterEvent()'.

-- Alternative registration methods are ALSO blocked:
EventRegistry:RegisterFrameEventAndCallback("COMBAT_LOG_EVENT_UNFILTERED", ...)  -- BLOCKED!
RegisterEventCallback("COMBAT_LOG_EVENT_UNFILTERED", ...)  -- BLOCKED!
```

**The restrictions:**
- Apply **always** for event registration (not just during combat)
- Registration is blocked even at addon load time
- Using `pcall()` does NOT prevent the ADDON_ACTION_FORBIDDEN error from being raised
- Other addons loading your addon via `LoadAddOn()` can cause taint issues

**What Still Works:**
1. **Blizzard's built-in damage meter** - Now part of the base UI
2. **C_DamageMeter API** - Official API for aggregated damage/healing data (see below)
3. **Non-combat events** - Other events still work normally

**Migration Path for Damage Meters:**
Traditional damage meters that parse `COMBAT_LOG_EVENT_UNFILTERED` **cannot function in 12.0.0**. You have two options:

1. **Use C_DamageMeter API** (recommended) - Provides aggregated damage/healing data through official channels
2. **Discontinue combat log parsing** - Accept that raw combat log access is no longer available to addons

**Why Blizzard Made This Change:**
> "When addons could analyze combat information in real-time, they could process combat information to make decisions for players, meaning players no longer need to make these decisions themselves."

This is part of Blizzard's effort to prevent addons from "solving" encounter mechanics automatically.

**References:**
- [Combat Addon Restrictions Eased in Midnight - Icy Veins](https://www.icy-veins.com/wow/news/combat-addon-restrictions-eased-in-midnight/)
- [Patch 12.0.0/Planned API changes - Warcraft Wiki](https://warcraft.wiki.gg/wiki/Patch_12.0.0/Planned_API_changes)

#### C_DamageMeter Namespace - ⛔ SECRET-PROTECTED (UNUSABLE BY ADDONS)

> **⚠️ CRITICAL DISCOVERY (Verified January 2026):**
> While C_DamageMeter API exists and appears functional, **the actual data returned is protected as "secret values"** and cannot be used by third-party addons. This API appears to be designed ONLY for Blizzard's built-in damage meter UI.
>
> For details on secret values, see [12a_Secret_Safe_APIs.md](12a_Secret_Safe_APIs.md).

**What Happens When You Try to Use It:**
```lua
-- The API returns data, but key values are SECRET:
local sessionData = C_DamageMeter.GetCombatSessionFromType(sessionType, meterType)
for _, source in ipairs(sessionData.combatSources) do
    -- These are ACCESSIBLE (cosmetic only):
    print(source.classFilename)  -- Works: "DEATHKNIGHT"
    print(source.isLocalPlayer)  -- Works: true
    print(source.specIconID)     -- Works: 135770

    -- These are SECRET (unusable):
    print(source.name)           -- ERROR: "attempt to compare (a secret value)"
    print(source.totalAmount)    -- SECRET - no value accessible
    print(source.amountPerSecond)-- SECRET - no value accessible
    print(source.sourceGUID)     -- SECRET - no value accessible
end
```

**Error You'll See:**
```
attempt to compare local 'name' (a secret value)
```

**The Reality:**
| Field | Status | Notes |
|-------|--------|-------|
| `name` | 🔒 SECRET | Player names hidden |
| `totalAmount` | 🔒 SECRET | Damage/healing totals hidden |
| `amountPerSecond` | 🔒 SECRET | DPS/HPS hidden |
| `sourceGUID` | 🔒 SECRET | Player GUIDs hidden |
| `classFilename` | ✅ Accessible | Cosmetic only |
| `isLocalPlayer` | ✅ Accessible | Cosmetic only |
| `specIconID` | ✅ Accessible | Cosmetic only |

**Bottom Line:**
- **Third-party damage meters CANNOT function in WoW 12.0.0+**
- Combat log is blocked AND C_DamageMeter data is secret-protected
- Players must use Blizzard's built-in damage meter (Shift+P or Encounter Journal)
- This is intentional as part of Blizzard's "addon disarmament" initiative

**API Reference (for documentation purposes only):**
```lua
-- Check availability (this works, returns true, but data is still secret!)
C_DamageMeter.IsDamageMeterAvailable()
  -- Returns: isAvailable (bool), failureReason (string)

-- Get session data - DATA IS SECRET
C_DamageMeter.GetCombatSessionFromType(sessionType, meterType)
C_DamageMeter.GetCombatSessionFromID(sessionID, meterType)

-- Get spell breakdown - DATA IS SECRET
C_DamageMeter.GetCombatSessionSourceFromType(sessionType, meterType, sourceGUID)

-- Reset (this may work)
C_DamageMeter.ResetAllCombatSessions()

-- Enums exist but data they access is secret:
-- Enum.DamageMeterSessionType = { Overall, CurrentFight, LastFight }
-- Enum.DamageMeterType = { DamageDone, HealingDone, DamageTaken }

-- Events fire but contain no usable data:
-- DAMAGE_METER_COMBAT_SESSION_UPDATED
-- DAMAGE_METER_CURRENT_SESSION_UPDATED
-- DAMAGE_METER_RESET
```

**If You're Maintaining a Damage Meter Addon:**
1. Display a message explaining the limitation to users
2. Suggest using Blizzard's built-in meter
3. Consider discontinuing 12.0.0+ support
4. Monitor for potential future changes from Blizzard

#### C_EncounterTimeline and C_EncounterWarnings

```lua
-- Boss timeline system
C_EncounterTimeline.GetCurrentEncounterTimeline()
C_EncounterTimeline.GetPhaseInfo(phaseIndex)
C_EncounterTimeline.GetUpcomingEvents(timeWindow)
C_EncounterTimeline.GetEventInfo(eventID)

-- Encounter warnings (DBM/BigWigs-style built-in)
C_EncounterWarnings.GetActiveWarnings()
C_EncounterWarnings.GetWarningInfo(warningID)
C_EncounterWarnings.AcknowledgeWarning(warningID)
C_EncounterWarnings.SetWarningEnabled(warningType, enabled)
C_EncounterWarnings.GetWarningSettings()
```

#### C_CombatAudioAlert - TTS Accessibility

```lua
-- Text-to-speech combat alerts for accessibility
C_CombatAudioAlert.PlayAlert(alertType, message)
C_CombatAudioAlert.SetAlertEnabled(alertType, enabled)
C_CombatAudioAlert.GetAlertSettings()
C_CombatAudioAlert.SetVoice(voiceID)
C_CombatAudioAlert.SetVolume(volume)
C_CombatAudioAlert.GetAvailableVoices()
```

#### C_SpellDiminish - Diminishing Returns Tracking

```lua
-- Official DR tracking (huge for PvP addons!)
C_SpellDiminish.GetDiminishingCategory(spellID)
C_SpellDiminish.GetDiminishingState(targetGUID, category)
C_SpellDiminish.GetDiminishingDuration(targetGUID, category)
C_SpellDiminish.GetDiminishingCategories()
C_SpellDiminish.IsDiminishingSpell(spellID)
```

#### C_TransmogOutfitInfo - Complete Transmog Overhaul

```lua
-- OLD transmog APIs removed, use new namespace:
C_TransmogOutfitInfo.GetOutfits()
C_TransmogOutfitInfo.GetOutfitInfo(outfitID)
C_TransmogOutfitInfo.CreateOutfit(name, appearanceTable)
C_TransmogOutfitInfo.DeleteOutfit(outfitID)
C_TransmogOutfitInfo.ModifyOutfit(outfitID, appearanceTable)
C_TransmogOutfitInfo.ApplyOutfit(outfitID)
C_TransmogOutfitInfo.GetOutfitAppearances(outfitID)
```

#### Utility Namespaces

**C_CurveUtil and C_DurationUtil (for Secret Values):**
```lua
-- These help work with secret numeric values safely
C_CurveUtil.GetCurveValue(curveID, position)
C_CurveUtil.InterpolateCurve(curveID, startPos, endPos, t)

C_DurationUtil.GetRemainingTime(endTime)
C_DurationUtil.GetElapsedTime(startTime)
C_DurationUtil.FormatDuration(seconds)
```

**C_StringUtil:**
```lua
C_StringUtil.SanitizeString(str)
C_StringUtil.TruncateString(str, maxLength)
C_StringUtil.FormatLargeNumber(number)
C_StringUtil.ParseItemLink(link)
```

**C_ColorUtil:**
```lua
C_ColorUtil.CreateColor(r, g, b, a)
C_ColorUtil.GetClassColor(classFilename)
C_ColorUtil.GetQualityColor(quality)
C_ColorUtil.HexToRGB(hexString)
C_ColorUtil.RGBToHex(r, g, b)
```

**C_DeathRecap (Moved from Globals):**
```lua
-- OLD:
local events = GetDeathRecapEvents()

-- NEW:
local events = C_DeathRecap.GetDeathRecapEvents()
C_DeathRecap.GetDeathRecapInfo(recapID)
C_DeathRecap.GetNumDeathRecaps()
```

#### GetMouseFocus() Removed - Use GetMouseFoci()

The `GetMouseFocus()` function was removed in 12.0.0. Use `GetMouseFoci()` which returns a table of frames under the cursor.

**Compatibility Wrapper:**
```lua
-- Works on both pre-12.0.0 and 12.0.0+
local GetMouseFocus = GetMouseFocus or function()
    local mouseFoci = GetMouseFoci and GetMouseFoci()
    return mouseFoci and mouseFoci[1] or nil
end
```

**Why It Changed:**
The new `GetMouseFoci()` returns ALL frames under the cursor (useful for overlapping frames), while the old API only returned one. The wrapper above returns just the first frame for backward compatibility.

#### Menu System Migration (UIDropDownMenu to MenuUtil)

The legacy `UIDropDownMenu` system was deprecated in 11.0.0 and the new Menu system became standard. While basic usage is documented elsewhere, here are **critical lessons learned from real-world library migrations**:

##### 1. MenuUtil.CreateContextMenu() Positioning

`MenuUtil.CreateContextMenu()` **always anchors to the cursor position**. You cannot change this behavior - it's hardcoded to use `MenuConstants.VerticalLinearDirection` at the cursor.

**For custom positioning, use the lower-level Menu API directly:**
```lua
-- Create menu description
local menuDescription = MenuUtil.CreateContextMenuDescription()
menuDescription:CreateButton("Option 1", function() print("Selected 1") end)
menuDescription:CreateButton("Option 2", function() print("Selected 2") end)

-- Create custom anchor (TOPRIGHT of menu at BOTTOMRIGHT of parent)
local anchor = CreateAnchor("TOPRIGHT", parentFrame, "BOTTOMRIGHT", 0, 0)

-- Open menu with custom positioning
local menu = Menu.GetManager():OpenMenu(parentFrame, menuDescription, anchor)
```

**Anchor Parameters:**
- `CreateAnchor(menuPoint, relativeFrame, relativePoint, offsetX, offsetY)`
- `menuPoint` - Where on the menu to anchor (e.g., "TOPRIGHT", "TOPLEFT")
- `relativeFrame` - Frame to anchor to
- `relativePoint` - Point on that frame to anchor to
- `offsetX`, `offsetY` - Pixel offsets

##### 2. MenuUtil Buttons Are NOT Secure

**Critical: Menu buttons created via MenuUtil CANNOT cast spells, use items, or use toys directly.** They are NOT `SecureActionButtonTemplate` frames.

**The Problem:**
```lua
-- THIS DOES NOT WORK for casting:
menuDescription:CreateButton("Cast Hearthstone", function()
    -- SecureActionButton attributes have no effect here
    -- You cannot cast spells from this callback
end)
```

**The Solution - Overlay a SecureActionButton:**
```lua
-- Create a secure overlay button
local secureButton = CreateFrame("Button", nil, UIParent, "SecureActionButtonTemplate")
secureButton:SetAttribute("type", "spell")
secureButton:SetAttribute("spell", "Hearthstone")

-- Position it over the menu item using AddInitializer
menuDescription:CreateButton("Cast Hearthstone")
    :AddInitializer(function(button, elementDescription, menu)
        -- Position secure button over this menu button
        secureButton:SetAllPoints(button)
        secureButton:SetFrameStrata("DIALOG")
        secureButton:SetFrameLevel(button:GetFrameLevel() + 10)
        secureButton:Show()

        -- Make secure button visually transparent (menu button provides visuals)
        secureButton:SetAlpha(0)
        secureButton:EnableMouse(true)
    end)
```

**Attribute Types for Secure Buttons:**
```lua
-- For spells:
secureButton:SetAttribute("type", "spell")
secureButton:SetAttribute("spell", "Spell Name or ID")

-- For items:
secureButton:SetAttribute("type", "item")
secureButton:SetAttribute("item", "item:12345")  -- Item link format

-- For toys:
secureButton:SetAttribute("type", "toy")
secureButton:SetAttribute("toy", 12345)  -- Toy item ID

-- For macros:
secureButton:SetAttribute("type", "macro")
secureButton:SetAttribute("macrotext", "/cast Hearthstone")
```

##### 3. SetTooltip() Overwrites SetOnEnter()

When using both tooltips and custom OnEnter handlers on menu elements, **order matters**:

**Problem - OnEnter handler gets replaced:**
```lua
local button = menuDescription:CreateButton("My Button")
button:SetOnEnter(function(button)
    -- Custom enter behavior
    PlaySound(SOUNDKIT.IG_MAINMENU_OPTION_CHECKBOX_ON)
end)
button:SetTooltip(function(tooltip)
    tooltip:AddLine("Tooltip text")
end)
-- Result: OnEnter is GONE, only tooltip shows
```

**Solution - Call SetTooltip FIRST, then use HookOnEnter:**
```lua
local button = menuDescription:CreateButton("My Button")

-- 1. Set tooltip FIRST
button:SetTooltip(function(tooltip)
    tooltip:AddLine("Tooltip text")
end)

-- 2. THEN hook the enter handler (adds to existing, doesn't replace)
button:HookOnEnter(function(button)
    -- Your custom behavior runs IN ADDITION to tooltip
    PlaySound(SOUNDKIT.IG_MAINMENU_OPTION_CHECKBOX_ON)
end)
```

##### 4. Menu AddInitializer OnEnter/OnLeave Scripts

The Menu system **overwrites button scripts AFTER initializers run**. Setting scripts directly in AddInitializer does not work.

**Problem - Scripts get overwritten:**
```lua
menuDescription:CreateButton("My Button")
    :AddInitializer(function(button, elementDescription, menu)
        -- THIS DOES NOT WORK - gets overwritten after initializer returns
        button:SetScript("OnEnter", function(self)
            print("Enter!")  -- Never fires
        end)
    end)
```

**Solution - Use HookOnEnter/HookOnLeave on the element description:**
```lua
local buttonDesc = menuDescription:CreateButton("My Button")

-- Hook on the description object, not the button
buttonDesc:HookOnEnter(function(button)
    print("Enter!")  -- This WILL fire
end)

buttonDesc:HookOnLeave(function(button)
    print("Leave!")  -- This WILL fire
end)

-- AddInitializer is still useful for other setup
buttonDesc:AddInitializer(function(button, elementDescription, menu)
    -- Set up button appearance, text, icons, etc.
    button.customData = "some value"
end)
```

##### 5. Font Sizing in Menus

Menu button font strings are **compositor font strings** managed by the Menu system. Calling `SetFont()` directly on them is **disallowed** and will fail silently or throw an error.

**Problem - SetFont() fails:**
```lua
menuDescription:CreateButton("My Button")
    :AddInitializer(function(button, elementDescription, menu)
        -- ❌ THIS DOES NOT WORK - compositor font strings cannot use SetFont()
        button.fontString:SetFont("Fonts\\FRIZQT__.TTF", 14, "OUTLINE")
    end)
```

**Solution - Create a custom Font object and use SetFontObject():**
```lua
-- Create a Font object (do this ONCE, not per-button)
local MyMenuFont = CreateFont("MyAddon_MenuFont")
MyMenuFont:SetFont("Fonts\\FRIZQT__.TTF", 14, "OUTLINE")

-- Apply the font object to menu buttons
menuDescription:CreateButton("My Button")
    :AddInitializer(function(button, elementDescription, menu)
        -- ✅ THIS WORKS - SetFontObject is allowed
        button.fontString:SetFontObject(MyMenuFont)
    end)
```

##### 6. Row Height Scaling for Custom Fonts

When using a larger font size in menus, the default button height may be too small, causing **text overlap** between menu items.

**Problem - Text overlaps:**
```lua
-- Using a 16pt font with default ~20px button height = overlap
local LargeFont = CreateFont("MyAddon_LargeFont")
LargeFont:SetFont("Fonts\\FRIZQT__.TTF", 16, "OUTLINE")

menuDescription:CreateButton("Line 1"):AddInitializer(function(btn)
    btn.fontString:SetFontObject(LargeFont)
end)
menuDescription:CreateButton("Line 2"):AddInitializer(function(btn)
    btn.fontString:SetFontObject(LargeFont)  -- May overlap with Line 1!
end)
```

**Solution - Set button height to match font size:**
```lua
-- Rule of thumb: button height = fontSize + 6 (for padding)
local fontSize = 14
local buttonHeight = fontSize + 6  -- 20px

local MenuFont = CreateFont("MyAddon_MenuFont")
MenuFont:SetFont("Fonts\\FRIZQT__.TTF", fontSize, "OUTLINE")

menuDescription:CreateButton("My Button")
    :AddInitializer(function(button, elementDescription, menu)
        button.fontString:SetFontObject(MenuFont)
        button:SetHeight(buttonHeight)  -- Prevent text overlap
    end)
```

**Complete Font Customization Pattern:**
```lua
-- Configuration
local MENU_FONT_SIZE = 13
local MENU_BUTTON_HEIGHT = MENU_FONT_SIZE + 6

-- Create font object once at addon load
local MyMenuFont = CreateFont("MyAddon_MenuFont")
MyMenuFont:SetFont("Fonts\\FRIZQT__.TTF", MENU_FONT_SIZE, "OUTLINE")

-- Helper function for styled buttons
local function CreateStyledButton(menuDesc, text, onClick)
    local button = menuDesc:CreateButton(text, onClick)
    button:AddInitializer(function(btn)
        btn.fontString:SetFontObject(MyMenuFont)
        btn:SetHeight(MENU_BUTTON_HEIGHT)
    end)
    return button
end

-- Usage
CreateStyledButton(menuDescription, "Option 1", function() print("1") end)
CreateStyledButton(menuDescription, "Option 2", function() print("2") end)
```

##### 7. Secure Overlay Flickering Prevention

When overlaying a secure button on a menu item, mouse movement between the menu button and the overlay can cause flickering (hide/show loops).

**Problem - Flickering when mouse moves between frames:**
```lua
-- Menu button's OnLeave hides the overlay
-- But mouse is now over overlay, which triggers menu button's OnLeave again
-- Result: Rapid flickering
```

**Solution - Check GetMouseFocus() in OnLeave:**
```lua
-- GetMouseFocus compatibility wrapper (see GetMouseFocus section above)
local GetMouseFocus = GetMouseFocus or function()
    local mouseFoci = GetMouseFoci and GetMouseFoci()
    return mouseFoci and mouseFoci[1] or nil
end

local secureOverlay = CreateFrame("Button", nil, UIParent, "SecureActionButtonTemplate")

buttonDesc:HookOnLeave(function(button)
    -- Only hide overlay if mouse isn't over the overlay itself
    local focus = GetMouseFocus()
    if focus ~= secureOverlay then
        secureOverlay:Hide()
    end
end)

-- Also handle overlay's own OnLeave
secureOverlay:SetScript("OnLeave", function(self)
    local focus = GetMouseFocus()
    -- Check if mouse moved to the underlying menu button
    if focus ~= currentMenuButton then
        self:Hide()
    end
end)
```

**Complete Pattern for Secure Menu Overlays:**
```lua
local function CreateSecureMenuButton(menuDesc, text, actionType, actionValue)
    local secureBtn = CreateFrame("Button", nil, UIParent, "SecureActionButtonTemplate")
    secureBtn:SetAttribute("type", actionType)
    secureBtn:SetAttribute(actionType, actionValue)
    secureBtn:Hide()

    local currentButton = nil

    local buttonDesc = menuDesc:CreateButton(text)

    buttonDesc:AddInitializer(function(button, elementDesc, menu)
        currentButton = button
        secureBtn:SetParent(button)
        secureBtn:SetAllPoints(button)
        secureBtn:SetFrameStrata("DIALOG")
        secureBtn:SetFrameLevel(button:GetFrameLevel() + 10)
        secureBtn:SetAlpha(0)
        secureBtn:EnableMouse(true)
        secureBtn:Show()
    end)

    buttonDesc:HookOnLeave(function(button)
        local focus = GetMouseFocus()
        if focus ~= secureBtn then
            secureBtn:Hide()
        end
    end)

    secureBtn:SetScript("OnLeave", function(self)
        local focus = GetMouseFocus()
        if focus ~= currentButton then
            self:Hide()
        end
    end)

    return buttonDesc, secureBtn
end

-- Usage:
local spellBtn, secureSpell = CreateSecureMenuButton(
    menuDescription,
    "Cast Hearthstone",
    "spell",
    "Hearthstone"
)
```

#### Major Removals in 12.0.0

**Action Bar Globals (Use C_ActionBar):**
- All `GetAction*()` globals
- All `IsAction*()` globals
- `PickupAction()`, `PlaceAction()`

**Combat Log Globals (Use C_CombatLog):**
- `CombatLogGetCurrentEventInfo()`
- `CombatLog*()` functions

**Emote Functions (Use C_ChatInfo):**
```lua
-- OLD:
DoEmote("wave", "target")
CancelEmote()

-- NEW:
C_ChatInfo.DoEmote("wave", "target")
C_ChatInfo.CancelEmote()
```

**BattleNet Functions (Use C_BattleNet):**
```lua
-- OLD:
BNSendWhisper(presenceID, message)
BNSendGameData(presenceID, prefix, data)

-- NEW:
C_BattleNet.SendWhisper(presenceID, message)
C_BattleNet.SendGameData(presenceID, prefix, data)
```

**Encounter Functions (Use C_InstanceEncounter):**
```lua
-- OLD:
local inProgress = IsEncounterInProgress()

-- NEW:
local inProgress = C_InstanceEncounter.IsEncounterInProgress()
C_InstanceEncounter.GetCurrentEncounterInfo()
```

#### Patch 12.0.0 (TOC 120000)

Minor refinements to 12.0.0 systems:

**Removed APIs:**
```lua
-- Some cooldown percent APIs removed:
GetActionCooldownRemainingPercent()  -- REMOVED, use GetActionCooldown() math
```

**Restored APIs:**
```lua
-- Adventure Journal CVars restored:
GetCVar("showTutorials")  -- Works again
SetCVar("showTutorials", value)
```

**Secret Values Refinements:**
- Better error messages for secret value violations
- Additional testing CVars for addon developers
- Performance improvements for secret value checks
- See [12a_Secret_Safe_APIs.md](12a_Secret_Safe_APIs.md) for complete reference

**Closing the Blizzard Settings Panel (12.0.0+):**

When addon code needs to close the Blizzard Settings panel (e.g., before opening a custom config dialog), there are important considerations:

```lua
-- WRONG: Does NOT exist
Settings.CloseUI()  -- nil error - this function does not exist

-- WRONG: Triggers protected function error
SettingsPanel:Close()  -- Causes ADDON_ACTION_BLOCKED

-- CORRECT: Use HideUIPanel instead
if SettingsPanel and SettingsPanel:IsShown() then
    HideUIPanel(SettingsPanel)
end
```

**Why SettingsPanel:Close() fails:**
- The `Close()` method internally calls `Commit()` which triggers `SaveBindings()` - a protected function
- When tainted (addon) code calls this chain, WoW raises `ADDON_ACTION_BLOCKED`
- `HideUIPanel()` simply hides the panel without attempting to save/commit settings

**Common Scenario - AceConfig Standalone Dialogs:**
```lua
-- When opening an AceConfig standalone dialog, you may want to close
-- Blizzard's Settings panel if it's open

local function OpenMyAddonConfig()
    -- Close Blizzard Settings panel first
    if SettingsPanel and SettingsPanel:IsShown() then
        HideUIPanel(SettingsPanel)
    end

    -- Now open our standalone config dialog
    LibStub("AceConfigDialog-3.0"):Open("MyAddon")
end
```

**Integration with Interface Options:**
```lua
-- If registering with Interface Options, you can use this pattern
-- to redirect to a standalone dialog

local function OpenStandaloneOptions()
    if SettingsPanel and SettingsPanel:IsShown() then
        HideUIPanel(SettingsPanel)
    end
    LibStub("AceConfigDialog-3.0"):Open("MyAddon")
end

-- Register a basic category that redirects
local category = Settings.RegisterCanvasLayoutCategory(
    panel, "MyAddon"
)
category.ID = "MyAddon"
Settings.RegisterAddOnCategory(category)

-- On the panel, add a button to open standalone config
-- (Or just hook your slash command to call OpenStandaloneOptions)
```

---

### The War Within Patches (11.0.2 - 11.2.7)

#### Patch 11.0.2 (TOC 110002)

**Settings API Signature Change (BREAKING):**
```lua
-- OLD (11.0.0):
Settings.RegisterAddOnSetting(category, name, variable, variableType, defaultValue)

-- NEW (11.0.2+):
Settings.RegisterAddOnSetting(category, variableKey, variableTbl, variableType, name, defaultValue)
-- variableKey: string key in variableTbl
-- variableTbl: table containing the setting (usually SavedVariables table)
```

**Macro Changes:**
- Macro chaining has been disabled
- `macrotext` attribute limited to 255 characters
- Affects action button automation addons

**C_GameRules Namespace (Replaces C_GameModeManager):**
```lua
-- OLD:
C_GameModeManager.GetCurrentGameMode()

-- NEW:
C_GameRules.GetCurrentGameMode()
C_GameRules.IsGameRuleActive(ruleType)
```

**New Delves APIs:**
```lua
C_DelvesUI.GetCurrentDelvesSeasonNumber()
C_DelvesUI.GetDelvesAffixSpellsForSeason(seasonNumber)
-- Plus additional Delves-related APIs
```

#### Patch 11.0.5 (TOC 110005)

**Merchant API Migration:**
```lua
-- OLD:
local name, texture, price, quantity = GetMerchantItemInfo(index)

-- NEW:
local info = C_MerchantFrame.GetItemInfo(index)
-- info.name, info.texture, info.price, info.quantity, etc.
```

**Specialization API Migration:**
```lua
-- OLD:
local numSpecs = GetNumSpecializationsForClassID(classID)

-- NEW:
local numSpecs = C_SpecializationInfo.GetNumSpecializationsForClassID(classID)
```

**Glue Screen Detection:**
```lua
-- OLD:
if IsOnGlueScreen() then ... end

-- NEW:
if C_Glue.IsOnGlueScreen() then ... end
```

**Challenge Mode API Change:**
```lua
-- OLD:
local info = C_ChallengeMode.GetCompletionInfo()

-- NEW:
local info = C_ChallengeMode.GetChallengeCompletionInfo()
```

**New C_ChatInfo Logging APIs:**
```lua
C_ChatInfo.GetRegisteredAddonMessagePrefixes()
C_ChatInfo.IsAddonMessagePrefixRegistered(prefix)
```

#### Patch 11.0.7 (TOC 110007)

**New C_AddOnProfiler Namespace:**
```lua
-- Enable addon profiling (development tool)
C_AddOnProfiler.EnableProfiling(addonName)
C_AddOnProfiler.DisableProfiling(addonName)
C_AddOnProfiler.GetAddOnMetric(addonName, metric)
C_AddOnProfiler.ResetAddOnMetrics(addonName)
-- Metrics: "Time", "Calls", "Allocs", etc.
```

**New C_AccountStore Namespace:**
```lua
C_AccountStore.GetProductInfo(productID)
C_AccountStore.IsProductOwned(productID)
-- For account-wide store purchases
```

**New C_LobbyMatchmakerInfo Namespace:**
```lua
C_LobbyMatchmakerInfo.GetMatchmakingState()
C_LobbyMatchmakerInfo.GetQueueInfo()
-- For lobby/matchmaking information
```

**C_LFGList Search Result Change (BREAKING):**
```lua
-- OLD:
local info = C_LFGList.GetSearchResultInfo(resultID)
local activityID = info.activityID  -- number

-- NEW:
local info = C_LFGList.GetSearchResultInfo(resultID)
local activityIDs = info.activityIDs  -- TABLE of numbers (multiple activities)

-- Migration:
local primaryActivity = info.activityIDs and info.activityIDs[1] or info.activityID
```

**Removed Quest APIs:**
```lua
-- REMOVED:
C_QuestLog.IsLegendaryQuest(questID)  -- No replacement
C_QuestLog.IsQuestRepeatableType(questID)  -- Use C_QuestLog.IsRepeatableQuest(questID)
```

#### Patch 11.1.0 (TOC 110100)

**New TOC Directives for Addon Organization:**
```toc
## Interface: 110100
## Title: MyAddon
## Category: Combat       -- Groups addon in AddOns list by category
## Group: MyAddonSuite    -- Groups related addons together
```

Valid Categories: Combat, Chat, Interface, Utility, Miscellaneous, etc.

**Specialization API Migration:**
```lua
-- OLD:
SetSpecialization(specIndex)

-- NEW:
C_SpecializationInfo.SetSpecialization(specIndex)
```

**Console Message API:**
```lua
-- OLD:
ConsoleAddMessage("message")

-- NEW:
ConsoleEcho("message")
```

**Gender/Sex Parameter Changes:**
```lua
-- OLD: Using numeric gender values
local gender = UnitSex("player")  -- 1=unknown, 2=male, 3=female
local name = GetClassInfo(classID, gender)

-- NEW: APIs now expect Enum.UnitSex values directly
local unitSex = UnitSex("player")  -- Returns Enum.UnitSex value
-- Many APIs updated to use UnitSex enum instead of raw numbers
```

**New C_EventScheduler Namespace (10 functions):**
```lua
-- Schedule delayed function calls (replaces C_Timer in some cases)
C_EventScheduler.ScheduleEvent(eventName, delay)
C_EventScheduler.CancelEvent(eventHandle)
C_EventScheduler.GetScheduledEvents()
C_EventScheduler.IsEventScheduled(eventHandle)
-- Better for game-state-aware scheduling
```

**New C_WarbandScene Namespace (7 functions):**
```lua
C_WarbandScene.GetWarbandSceneInfo()
C_WarbandScene.SetWarbandSceneState(state)
-- For warband character select scene management
```

#### Patch 11.1.5 (TOC 110105)

**New C_EncodingUtil Namespace (MAJOR ADDITION):**
```lua
-- Compression
local compressed = C_EncodingUtil.CompressData(data)
local decompressed = C_EncodingUtil.DecompressData(compressed)

-- Base64 encoding
local encoded = C_EncodingUtil.EncodeBase64(data)
local decoded = C_EncodingUtil.DecodeBase64(encoded)

-- Hex encoding
local hexString = C_EncodingUtil.EncodeHex(data)
local rawData = C_EncodingUtil.DecodeHex(hexString)

-- JSON serialization (HUGE for addon communication!)
local jsonString = C_EncodingUtil.EncodeJSON(luaTable)
local luaTable = C_EncodingUtil.DecodeJSON(jsonString)

-- CBOR binary serialization (compact)
local cborData = C_EncodingUtil.EncodeCBOR(luaTable)
local luaTable = C_EncodingUtil.DecodeCBOR(cborData)
```

**Color Override System:**
```lua
-- Override item quality colors
C_ColorOverride.SetQualityColorOverride(quality, colorTable)
C_ColorOverride.GetQualityColorOverride(quality)
C_ColorOverride.ResetQualityColorOverride(quality)
-- Allows customizing item rarity colors globally
```

**New TOC Directive - LoadSavedVariablesFirst:**
```toc
## Interface: 110105
## Title: MyAddon
## SavedVariables: MyAddonDB
## LoadSavedVariablesFirst: 1   -- Loads saved vars BEFORE any scripts run
```

This is extremely useful for addons that need saved data during file execution.

**AllowLoadGameType Now Usable:**
```toc
## AllowLoadGameType: glue   -- Previously restricted, now available
## AllowLoadGameType: mainline
## AllowLoadGameType: classic
```

**TOC Inline Variables:**
```toc
## Title: MyAddon [Family]     -- Expands to addon family name
## Title: MyAddon [Game]       -- Expands to game type (Retail, Classic, etc.)
## Title: MyAddon [TextLocale] -- Expands to current locale (enUS, deDE, etc.)
```

**Trade Money APIs Moved:**
```lua
-- OLD:
AddTradeMoney(copper)
SetTradeMoney(copper)
GetPlayerTradeMoney()

-- NEW:
C_TradeInfo.AddTradeMoney(copper)
C_TradeInfo.SetTradeMoney(copper)
C_TradeInfo.GetPlayerTradeMoney()
```

**UnitCreatureFamily/UnitCreatureType Enhancement:**
```lua
-- OLD: Returns localized string only
local familyName = UnitCreatureFamily("pet")  -- "Cat" (localized)

-- NEW: Second return is locale-independent ID
local familyName, familyID = UnitCreatureFamily("pet")  -- "Cat", 1
local typeName, typeID = UnitCreatureType("target")     -- "Humanoid", 7
-- Use ID for reliable comparisons across locales!
```

**New table.count Function:**
```lua
-- Count total entries in a table (including hash part)
local t = {a = 1, b = 2, [1] = "x", [2] = "y"}
local count = table.count(t)  -- Returns 4
-- More efficient than manual iteration
```

#### Patch 11.1.7 (TOC 110107)

**C_AddOnProfiler Enhancement:**
```lua
-- NEW: Profile individual function calls
local elapsed, returnVal1, returnVal2 = C_AddOnProfiler.MeasureCall(myFunc, arg1, arg2)
-- elapsed is execution time in seconds
-- All return values from myFunc are passed through
```

**New C_AddOns.GetAddOnLocalTable:**
```lua
-- Access another addon's local table (requires permission)
local otherAddonTable = C_AddOns.GetAddOnLocalTable("OtherAddon")

-- Requires TOC directive in the TARGET addon:
## AllowAddOnTableAccess: 1  -- Allows other addons to access this addon's table
```

**New table.create Function:**
```lua
-- Pre-allocate table memory for performance
local arrayTable = table.create(1000, 0)    -- 1000 array slots, 0 hash slots
local mixedTable = table.create(100, 50)    -- 100 array, 50 hash slots
local hashTable = table.create(0, 200)      -- 0 array, 200 hash slots

-- Use when you know approximate table size in advance
-- Reduces memory fragmentation and improves performance
```

**New C_AssistedCombat Namespace:**
```lua
-- Assisted Combat system for accessibility features
C_AssistedCombat.IsAssistedCombatEnabled()
C_AssistedCombat.SetAssistedCombatEnabled(enabled)
C_AssistedCombat.GetAssistedCombatSettings()
-- Helps players who need combat assistance
```

#### Patch 11.2.0 (TOC 110200)

**MAJOR: Reagent Bank and Void Storage REMOVED:**
```lua
-- These APIs no longer function:
-- Reagent Bank:
C_Container.GetContainerNumFreeSlots(Enum.BagIndex.ReagentBank)  -- REMOVED
-- Void Storage:
C_VoidStorage.*  -- REMOVED (entire namespace)

-- Bank system reworked - BagIndex enum values changed:
-- OLD Enum.BagIndex values may be different
-- Check Enum.BagIndex documentation for new values
```

**Specialization Globals Moved:**
```lua
-- OLD globals:
GetSpecialization()
GetSpecializationInfo(specIndex)
GetSpecializationRole(specIndex)

-- NEW C_SpecializationInfo namespace:
C_SpecializationInfo.GetSpecialization()
C_SpecializationInfo.GetSpecializationInfo(specIndex)
C_SpecializationInfo.GetSpecializationRole(specIndex)
```

**SendChatMessage Migration:**
```lua
-- OLD:
SendChatMessage(msg, chatType, language, channel)

-- NEW:
C_ChatInfo.SendChatMessage(msg, chatType, language, channel)
```

**Spell APIs Moved:**
```lua
-- OLD:
local known = IsSpellKnown(spellID)
local overlayed = IsSpellOverlayed(spellID)

-- NEW:
local known = C_SpellBook.IsSpellKnown(spellID)
local overlayed = C_SpellActivationOverlay.IsSpellOverlayed(spellID)
```

**StaticPopup Changes:**
```lua
-- OLD:
for i, frame in pairs(StaticPopup_DisplayedFrames) do
    -- iterate displayed popups
end

-- NEW:
StaticPopup_ForEachShownDialog(function(dialog, data)
    -- process each shown dialog
end)
```

**Font Scaling Support:**
```lua
-- New CVar for user font scaling
SetCVar("userFontScale", 1.2)  -- 120% font size
local scale = GetCVar("userFontScale")

-- Check if font scaling is active
local isScaled = (tonumber(GetCVar("userFontScale")) or 1) ~= 1
```

**Instance Abandon Vote System:**
```lua
-- New party vote APIs
C_PartyInfo.StartInstanceAbandonVote()
C_PartyInfo.GetInstanceAbandonVoteInfo()
C_PartyInfo.CanStartInstanceAbandonVote()
-- Allows voting to abandon dungeon instances
```

#### Patch 11.2.5 (TOC 110205)

**MAJOR: Item Socket APIs Moved to C_ItemSocketInfo:**
```lua
-- OLD globals:
AcceptSockets()
GetNumSockets()
GetExistingSocketInfo(socketIndex)
GetNewSocketInfo(socketIndex)
GetSocketTypes(socketIndex)
ClickSocketButton(socketIndex)
CloseSocketInfo()
GetSocketItemBoundTradeable()
GetSocketItemInfo()
GetSocketItemRefundable()
HasBoundGemProposed()
SocketContainerItem(bagID, slot)
SocketInventoryItem(slotID)

-- NEW C_ItemSocketInfo namespace:
C_ItemSocketInfo.AcceptSockets()
C_ItemSocketInfo.GetNumSockets()
C_ItemSocketInfo.GetExistingSocketInfo(socketIndex)
C_ItemSocketInfo.GetNewSocketInfo(socketIndex)
C_ItemSocketInfo.GetSocketTypes(socketIndex)
C_ItemSocketInfo.ClickSocketButton(socketIndex)
C_ItemSocketInfo.CloseSocketInfo()
C_ItemSocketInfo.GetSocketItemBoundTradeable()
C_ItemSocketInfo.GetSocketItemInfo()
C_ItemSocketInfo.GetSocketItemRefundable()
C_ItemSocketInfo.HasBoundGemProposed()
C_ItemSocketInfo.SocketContainerItem(bagID, slot)
C_ItemSocketInfo.SocketInventoryItem(slotID)
```

**Pet APIs Moved:**
```lua
-- OLD:
local tree = GetPetTalentTree()

-- NEW:
local tree = C_PetInfo.GetPetTalentTree()
```

**Twitter Integration REMOVED:**
```lua
-- These APIs no longer exist:
C_Social.TwitterConnect()
C_Social.TwitterDisconnect()
C_Social.TwitterPostMessage()
C_Social.GetTwitterStatus()
-- All Twitter-related functionality removed
```

**New C_CatalogShop Namespace (21 functions):**
```lua
-- In-game shop functionality
C_CatalogShop.GetCategoryInfo(categoryID)
C_CatalogShop.GetProductInfo(productID)
C_CatalogShop.PurchaseProduct(productID)
C_CatalogShop.GetOwnedProducts()
C_CatalogShop.IsProductOwned(productID)
C_CatalogShop.GetProductPrice(productID)
-- Full in-game catalog shopping API
```

**New C_RecentAllies Namespace (14 functions):**
```lua
-- Track recently grouped players
C_RecentAllies.GetRecentAllies()
C_RecentAllies.GetAllyInfo(playerGUID)
C_RecentAllies.AddRecentAlly(playerGUID)
C_RecentAllies.RemoveRecentAlly(playerGUID)
-- Useful for social/grouping addons
```

**New C_RemixArtifactUI Namespace:**
```lua
-- Legion Remix artifact support
C_RemixArtifactUI.GetArtifactInfo()
C_RemixArtifactUI.GetArtifactTraits()
-- For Legion Remix event
```

#### Patch 11.2.7 (TOC 110207)

**MASSIVE: Housing System Added (287+ new APIs!):**

The Housing system introduces 10+ new namespaces:

```lua
-- Core Housing APIs
C_Housing.GetHousingInfo()
C_Housing.GetOwnedHouses()
C_Housing.EnterHouse(houseID)
C_Housing.LeaveHouse()
C_Housing.GetCurrentHouseInfo()
C_Housing.InviteToHouse(playerName)

-- House Editor
C_HouseEditor.EnterEditMode()
C_HouseEditor.ExitEditMode()
C_HouseEditor.SaveChanges()
C_HouseEditor.RevertChanges()
C_HouseEditor.GetPlacedItems()

-- Exterior Customization
C_HouseExterior.GetExteriorOptions()
C_HouseExterior.SetExteriorStyle(styleID)
C_HouseExterior.GetCurrentExteriorStyle()

-- Basic/Simple Editing Mode
C_HousingBasicMode.PlaceItem(itemID, x, y, z)
C_HousingBasicMode.RemoveItem(placementID)
C_HousingBasicMode.MoveItem(placementID, x, y, z)

-- Item Catalog
C_HousingCatalog.GetAvailableItems()
C_HousingCatalog.GetItemInfo(itemID)
C_HousingCatalog.GetUnlockedItems()

-- Advanced Customization
C_HousingCustomizeMode.GetCustomizationOptions()
C_HousingCustomizeMode.SetCustomization(optionID, value)

-- Decor/Furniture
C_HousingDecor.GetDecorCategories()
C_HousingDecor.GetDecorInCategory(categoryID)
C_HousingDecor.GetDecorInfo(decorID)

-- Expert Editing Mode
C_HousingExpertMode.EnableGridSnap(enabled)
C_HousingExpertMode.SetGridSize(size)
C_HousingExpertMode.GetPrecisionPlacementInfo()

-- Layouts/Presets
C_HousingLayout.GetSavedLayouts()
C_HousingLayout.SaveLayout(name)
C_HousingLayout.LoadLayout(layoutID)
C_HousingLayout.DeleteLayout(layoutID)

-- Neighborhood System
C_HousingNeighborhood.GetNeighborhoods()
C_HousingNeighborhood.GetNeighborhoodInfo(neighborhoodID)
C_HousingNeighborhood.VisitNeighbor(playerGUID)
```

**New C_DyeColor Namespace (8 functions):**
```lua
-- Armor/item dye coloring system
C_DyeColor.GetAvailableDyeColors()
C_DyeColor.GetDyeColorInfo(dyeColorID)
C_DyeColor.ApplyDyeColor(itemGUID, dyeColorID)
C_DyeColor.RemoveDyeColor(itemGUID)
C_DyeColor.GetItemDyeColor(itemGUID)
C_DyeColor.IsDyeColorUnlocked(dyeColorID)
```

**New C_KeyBindings Namespace (9 functions):**
```lua
-- OLD:
local binding = GetBindingByKey("A")
local key = GetBindingKey("MOVEFORWARD")

-- NEW:
local binding = C_KeyBindings.GetBindingByKey("A")
local key = C_KeyBindings.GetBindingKey("MOVEFORWARD")

-- Additional new functions:
C_KeyBindings.GetNumBindings()
C_KeyBindings.GetBinding(index)
C_KeyBindings.SetBinding(key, command)
C_KeyBindings.GetCurrentBindingSet()
```

**PvP API Migration:**
```lua
-- OLD:
JoinBattlefield(index)

-- NEW:
C_PvP.JoinBattlefield(index)
```

**Expansion Upgrade Check:**
```lua
-- OLD:
local canUpgrade = CanUpgradeExpansion()

-- NEW:
local canUpgrade = CanUpgradeToCurrentExpansion()
```

**ChatFrame Functions Reorganized:**
```lua
-- Many ChatFrame global functions moved to ChatFrameUtil:
-- OLD:
ChatFrame_AddChannel(chatFrame, channel)
ChatFrame_RemoveChannel(chatFrame, channel)

-- NEW:
ChatFrameUtil.AddChannel(chatFrame, channel)
ChatFrameUtil.RemoveChannel(chatFrame, channel)
```

### The War Within (11.0.0+)

#### Removed APIs
```lua
-- REMOVED in 11.0.0
GetSpellInfo(spellID)  -- Use C_Spell.GetSpellInfo() or C_Spell.GetSpellName()
GetSpellCooldown(spellID)  -- Use C_Spell.GetSpellCooldown()
```

#### New/Changed APIs
```lua
-- NEW: C_Spell namespace (11.0.0)
C_Spell.GetSpellInfo(spellID)  -- Returns table
C_Spell.GetSpellName(spellID)  -- Returns name only
C_Spell.GetSpellCooldown(spellID)  -- Returns table

-- Example return values:
local spellInfo = C_Spell.GetSpellInfo(spellID)
-- spellInfo = {
--   name = "Spell Name",
--   iconID = 123456,
--   castTime = 1500,
--   minRange = 0,
--   maxRange = 40,
--   spellID = 12345
-- }

local cooldownInfo = C_Spell.GetSpellCooldown(spellID)
-- cooldownInfo = {
--   startTime = 123456.789,
--   duration = 30,
--   isEnabled = true,
--   modRate = 1.0
-- }
```

#### Deprecated (Still Works)
```lua
-- DEPRECATED in 11.0.0
UIDropDownMenu  -- Use new menu system or keep for compatibility
```

---

### Dragonflight (10.0.0 - 10.2.7)

#### Deprecated (Removed in 11.0)
```lua
-- DEPRECATED in 10.2.5
UnitAura(unit, index)  -- Use C_UnitAuras.GetAuraDataByIndex()
UnitBuff(unit, index)  -- Use C_UnitAuras.GetBuffDataByIndex()
UnitDebuff(unit, index)  -- Use C_UnitAuras.GetDebuffDataByIndex()

-- DEPRECATED in 10.2.7
TextStatusBar global functions  -- Use methods on StatusBar objects
```

### Shadowlands (9.0.0+)

#### New Namespaces
```lua
-- Container APIs moved to C_Container (9.0.0)
C_Container.GetContainerNumSlots(bagID)  -- Was GetContainerNumSlots()
C_Container.GetContainerItemID(bagID, slot)  -- Was GetContainerItemID()
C_Container.GetContainerItemInfo(bagID, slot)  -- Was GetContainerItemInfo()
```

---

## Common Migration Patterns

### Pattern 1: Single Function to Namespace

**Before (Old API):**
```lua
local name = GetSpellInfo(spellID)
```

**After (New API):**
```lua
local name = C_Spell.GetSpellName(spellID)
```

### Pattern 2: Multiple Return Values to Table

**Before (Old API):**
```lua
local start, duration, enabled = GetSpellCooldown(spellID)
```

**After (New API):**
```lua
local cooldownInfo = C_Spell.GetSpellCooldown(spellID)
local start = cooldownInfo.startTime
local duration = cooldownInfo.duration
local enabled = cooldownInfo.isEnabled
```

### Pattern 3: Compatibility Wrapper (Best Practice)

Create wrappers for backward compatibility with old code and libraries:

```lua
-- In your Constants.lua or init file (loaded first)

-- Compatibility wrapper for GetSpellInfo
if not _G.GetSpellInfo then
    _G.GetSpellInfo = function(spellID)
        if not spellID then return nil end
        local spellInfo = C_Spell.GetSpellInfo(spellID)
        if spellInfo then
            return spellInfo.name, nil, spellInfo.iconID,
                   spellInfo.castTime, spellInfo.minRange,
                   spellInfo.maxRange, spellInfo.spellID,
                   spellInfo.originalIconID
        end
        return nil
    end
end

-- Compatibility wrapper for GetSpellCooldown
if not _G.GetSpellCooldown then
    _G.GetSpellCooldown = function(spellID)
        local cooldownInfo = C_Spell.GetSpellCooldown(spellID)
        if cooldownInfo then
            return cooldownInfo.startTime, cooldownInfo.duration,
                   cooldownInfo.isEnabled, cooldownInfo.modRate
        end
        return 0, 0, false, 1
    end
end
```

**Benefits:**
- ✅ Old code continues to work
- ✅ Embedded libraries don't need updates
- ✅ Gradual migration possible
- ✅ One-file fix

### Pattern 4: Version-Specific Code Blocks

```lua
local tocVersion = select(4, GetBuildInfo())

if tocVersion >= 110000 then  -- 11.0.0+
    -- Use new API
    local name = C_Spell.GetSpellName(spellID)
else  -- 10.x or earlier
    -- Use old API
    local name = GetSpellInfo(spellID)
end
```

### Pattern 5: Safe API Wrapper with Fallback

```lua
-- Wrapper that works across all versions
local function GetSpellNameSafe(spellID)
    if C_Spell and C_Spell.GetSpellName then
        return C_Spell.GetSpellName(spellID)
    elseif GetSpellInfo then
        return GetSpellInfo(spellID)
    else
        return nil
    end
end
```

---

## Version Detection & Compatibility

### TOC Version Detection

```lua
-- Get current game version
local tocVersion = select(4, GetBuildInfo())
local majorVersion = math.floor(tocVersion / 10000)

-- Expansion detection
local isMidnight = (tocVersion >= 120000)
local isWarWithin = (tocVersion >= 110000 and tocVersion < 120000)
local isDragonflight = (tocVersion >= 100000 and tocVersion < 110000)
local isShadowlands = (tocVersion >= 90000 and tocVersion < 100000)
local isBFA = (tocVersion >= 80000 and tocVersion < 90000)

-- Use in code
if isMidnight then
    -- Midnight specific code (12.0+)
    -- Handle secret values, use C_ActionBar, C_CombatLog, etc.
elseif isWarWithin then
    -- War Within specific code (11.0 - 11.2.7)
elseif isDragonflight then
    -- Dragonflight specific code
end

-- Patch-level detection for War Within
local isWW_11_0_2 = (tocVersion >= 110002)  -- Settings API change
local isWW_11_1_0 = (tocVersion >= 110100)  -- Category/Group TOC
local isWW_11_1_5 = (tocVersion >= 110105)  -- C_EncodingUtil, table.count
local isWW_11_2_0 = (tocVersion >= 110200)  -- Reagent Bank removed
local isWW_11_2_5 = (tocVersion >= 110205)  -- C_ItemSocketInfo
local isWW_11_2_7 = (tocVersion >= 110207)  -- Housing system
```

### Client Type Detection

```lua
-- Detect Retail vs Classic
local isRetail = (WOW_PROJECT_ID == WOW_PROJECT_MAINLINE)
local isClassic = (WOW_PROJECT_ID == WOW_PROJECT_CLASSIC)
local isWrathClassic = (WOW_PROJECT_ID == WOW_PROJECT_WRATH_CLASSIC)
local isCataClassic = (WOW_PROJECT_ID == WOW_PROJECT_CATACLYSM_CLASSIC)

if isRetail then
    -- Use Retail-only APIs
end
```

### Function Existence Check (Safest)

```lua
-- Check if API exists before using
if C_Spell and C_Spell.GetSpellName then
    local name = C_Spell.GetSpellName(spellID)
elseif GetSpellInfo then
    local name = GetSpellInfo(spellID)
else
    error("No spell API available!")
end
```

---

## Automated Migration Checklist

Use this checklist when migrating an addon to a new WoW version:

### Pre-Migration

- [ ] Check Warcraft Wiki for API changes in new version
- [ ] Read patch notes for UI/API section
- [ ] Search WoW forums for common issues
- [ ] Check if addon uses embedded libraries (may need updates)
- [ ] Create backup of working addon

### TOC File Updates

- [ ] Update `## Interface:` version number
- [ ] Update `## Version:` string
- [ ] Check `## Dependencies:` are compatible
- [ ] Test without `## OptionalDeps:` first

### Code Audit

Run these searches in your addon folder:

#### Search for Deprecated Spell APIs (11.0+)
```bash
# Search for these patterns:
GetSpellInfo\(
GetSpellCooldown\(
```

#### Search for Deprecated Aura APIs (10.2.5+)
```bash
# Search for these patterns:
UnitAura\(
UnitBuff\(
UnitDebuff\(
```

#### Search for Old Container APIs (9.0+)
```bash
# Search for these patterns:
GetContainerNumSlots\(
GetContainerItemID\(
GetContainerItemInfo\(
GetContainerItemLink\(
```

#### Search for 11.x Deprecated APIs
```bash
# Search for these patterns (11.0.5+):
GetMerchantItemInfo\(
GetNumSpecializationsForClassID\(
IsOnGlueScreen\(

# Search for these patterns (11.1.0+):
SetSpecialization\(
ConsoleAddMessage\(

# Search for these patterns (11.2.0+):
SendChatMessage\(
IsSpellKnown\(
IsSpellOverlayed\(
StaticPopup_DisplayedFrames

# Search for these patterns (11.2.5+):
AcceptSockets\(
GetNumSockets\(
GetExistingSocketInfo\(
SocketContainerItem\(
GetPetTalentTree\(

# Search for these patterns (11.2.7+):
GetBindingByKey\(
JoinBattlefield\(
ChatFrame_AddChannel\(
```

#### Search for 12.0+ Removed APIs (Midnight)
```bash
# Search for these patterns (action bar):
GetActionInfo\(
GetActionTexture\(
GetActionCooldown\(
HasAction\(
IsUsableAction\(
IsCurrentAction\(
IsAutoRepeatAction\(
PickupAction\(
PlaceAction\(

# Search for these patterns (combat log):
CombatLogGetCurrentEventInfo\(
CombatLogSetCurrentEntry\(
CombatLogGetNumEntries\(

# Search for these patterns (emotes):
DoEmote\(
CancelEmote\(

# Search for these patterns (BattleNet):
BNSendWhisper\(
BNSendGameData\(

# Search for these patterns (encounters):
IsEncounterInProgress\(
```

### Testing

- [ ] Test addon loads without errors (`/reload`)
- [ ] Enable script errors: `/console scriptErrors 1`
- [ ] Test core functionality
- [ ] Check for deprecation warnings in chat
- [ ] Test with other popular addons
- [ ] Test in different zones/scenarios

### Documentation

- [ ] Update README with supported versions
- [ ] Document API changes in CHANGELOG
- [ ] Update CurseForge/Wago description
- [ ] Add migration notes for other developers

---

## Testing Across Versions

### In-Game Testing Commands

```lua
-- Check game version
/run print(select(4, GetBuildInfo()))

-- Check for function existence
/run print(C_Spell ~= nil)
/run print(C_Spell.GetSpellName ~= nil)

-- Enable error display
/console scriptErrors 1

-- Reload UI
/reload

-- Monitor events
/eventtrace

-- Check frame stack
/fstack
```

### 12.0.0+ Specific Testing Commands

```lua
-- Check for 12.0.0 namespaces
/run print("C_ActionBar:", C_ActionBar ~= nil)
/run print("C_CombatLog:", C_CombatLog ~= nil)
/run print("C_DamageMeter:", C_DamageMeter ~= nil)
/run print("C_Secrets:", C_Secrets ~= nil)

-- Test secret value functions
/run print("issecretvalue available:", issecretvalue ~= nil)
/run print("canaccessvalue available:", canaccessvalue ~= nil)

-- Force secret restrictions for testing (development)
/console secretCombatRestrictionsForced 1
/console secretEncounterRestrictionsForced 1
/console secretRestrictionsDebug 1

-- Disable secret restrictions (return to normal)
/console secretCombatRestrictionsForced 0
/console secretEncounterRestrictionsForced 0
/console secretRestrictionsDebug 0

-- Check if in restricted state
/run if C_RestrictedActions then print(C_RestrictedActions.IsInRestrictedState()) end

-- Test action bar API
/run if C_ActionBar then local t,i,s = C_ActionBar.GetActionInfo(1); print("Type:", t, "ID:", i) end

-- Test combat log API
/run if C_CombatLog then print("Entries:", C_CombatLog.GetNumEntries()) end
```

### 11.1.5+ Specific Testing (C_EncodingUtil)

```lua
-- Test JSON encoding/decoding
/run local t = {name="Test", value=42}; local json = C_EncodingUtil.EncodeJSON(t); print(json)
/run local data = C_EncodingUtil.DecodeJSON('{"test":true}'); print(data.test)

-- Test Base64
/run local encoded = C_EncodingUtil.EncodeBase64("Hello World"); print(encoded)
/run local decoded = C_EncodingUtil.DecodeBase64("SGVsbG8gV29ybGQ="); print(decoded)

-- Test table.count (11.1.5+)
/run local t = {a=1, b=2, [1]="x", [2]="y"}; print("Count:", table.count(t))

-- Test table.create (11.1.7+)
/run local t = table.create(100, 50); print("Created pre-allocated table")
```

### Testing Methodology

1. **Fresh Character Test**
   - Create new character
   - Test with no saved variables
   - Verify first-run experience

2. **Migrated Data Test**
   - Use character with old saved variables
   - Check data migration works
   - Verify no corruption

3. **Compatibility Test**
   - Test with other popular addons
   - Check for conflicts
   - Test Ace3 compatibility

---

## Real-World Migration Examples

### Example 1: Archaeology Addon (10.0 → 11.0)

**Issues Found:**
- `GetSpellInfo()` removed in 11.0
- `GetSpellCooldown()` changed signature

**Files Modified:**
```
Archy_Mainline.toc  - Update Interface version
Constants.lua       - Add compatibility wrappers
Archy.lua           - Update 3 locations
Race.lua            - Update 1 location
```

**Solution Applied:**
```lua
-- Added to Constants.lua (loaded first)
if not _G.GetSpellInfo then
    _G.GetSpellInfo = function(spellID)
        if not spellID then return nil end
        local spellInfo = C_Spell.GetSpellInfo(spellID)
        if spellInfo then
            return spellInfo.name, nil, spellInfo.iconID,
                   spellInfo.castTime, spellInfo.minRange,
                   spellInfo.maxRange, spellInfo.spellID,
                   spellInfo.originalIconID
        end
        return nil
    end
end

-- Direct API updates in addon code
local name = C_Spell.GetSpellName(SURVEY_SPELL_ID)

-- Updated cooldown checks
local cooldownInfo = C_Spell.GetSpellCooldown(spellID)
local duration = cooldownInfo and cooldownInfo.duration or 0
```

**Result:** ✅ Addon works, embedded libraries work via wrapper

### Example 2: Bag Addon (8.0 → 9.0)

**Issue:** Container APIs moved to `C_Container` namespace

**Before:**
```lua
local numSlots = GetContainerNumSlots(bagID)
local itemID = GetContainerItemID(bagID, slot)
```

**After:**
```lua
local numSlots = C_Container.GetContainerNumSlots(bagID)
local itemID = C_Container.GetContainerItemID(bagID, slot)
```

**Compatibility Wrapper:**
```lua
-- Support both old and new APIs
local GetNumSlots = C_Container and C_Container.GetContainerNumSlots
                    or GetContainerNumSlots
local GetItemID = C_Container and C_Container.GetContainerItemID
                  or GetContainerItemID

-- Use in code
local numSlots = GetNumSlots(bagID)
```

### Example 3: Aura Tracking (10.2 → 11.0)

**Issue:** `UnitAura()` deprecated in 10.2.5

**Before (Old):**
```lua
local name, icon, count, debuffType, duration, expirationTime =
    UnitAura("player", i, "HELPFUL")
```

**After (New):**
```lua
local auraData = C_UnitAuras.GetAuraDataByIndex("player", i, "HELPFUL")
if auraData then
    local name = auraData.name
    local icon = auraData.icon
    local count = auraData.applications
    local debuffType = auraData.dispelName
    local duration = auraData.duration
    local expirationTime = auraData.expirationTime
end
```

**Compatibility Solution:**
```lua
local function GetAuraInfo(unit, index, filter)
    -- Try new API first (10.2.5+)
    if C_UnitAuras and C_UnitAuras.GetAuraDataByIndex then
        local auraData = C_UnitAuras.GetAuraDataByIndex(unit, index, filter)
        if auraData then
            return auraData.name, auraData.icon, auraData.applications,
                   auraData.dispelName, auraData.duration,
                   auraData.expirationTime
        end
    -- Fall back to old API
    elseif UnitAura then
        return UnitAura(unit, index, filter)
    end
    return nil
end
```

### Example 4: Action Bar Addon (11.2.7 -> 12.0.0)

**Issues Found:**
- All global action bar functions removed
- Secret values returned during combat
- Combat log API moved to namespace

**Before (11.x):**
```lua
local frame = CreateFrame("Frame")
frame:RegisterEvent("ACTIONBAR_SLOT_CHANGED")
frame:SetScript("OnEvent", function(self, event, slot)
    local actionType, id, subType = GetActionInfo(slot)
    local texture = GetActionTexture(slot)
    local usable = IsUsableAction(slot)

    if HasAction(slot) then
        UpdateButton(slot, texture, usable)
    end
end)

-- Combat log processing
local combatFrame = CreateFrame("Frame")
combatFrame:RegisterEvent("COMBAT_LOG_EVENT_UNFILTERED")
combatFrame:SetScript("OnEvent", function()
    local timestamp, event, ... = CombatLogGetCurrentEventInfo()
    ProcessCombatEvent(event, ...)
end)
```

**After (12.0.0+):**
```lua
local frame = CreateFrame("Frame")
frame:RegisterEvent("ACTIONBAR_SLOT_CHANGED")
frame:SetScript("OnEvent", function(self, event, slot)
    -- Use new C_ActionBar namespace
    local actionType, id, subType = C_ActionBar.GetActionInfo(slot)
    local texture = C_ActionBar.GetActionTexture(slot)
    local usable = C_ActionBar.IsUsableAction(slot)

    -- Check for secret values during combat
    if issecretvalue(id) then
        -- Value is secret, defer processing or use cached data
        ScheduleOutOfCombat(function()
            UpdateButtonDeferred(slot)
        end)
        return
    end

    if C_ActionBar.HasAction(slot) then
        UpdateButton(slot, texture, usable)
    end
end)

-- Combat log processing with new namespace
local combatFrame = CreateFrame("Frame")
combatFrame:RegisterEvent("COMBAT_LOG_EVENT_UNFILTERED")
combatFrame:SetScript("OnEvent", function()
    local timestamp, event, ... = C_CombatLog.GetCurrentEventInfo()
    ProcessCombatEvent(event, ...)
end)
```

**Compatibility Wrapper for Multi-Version Support:**
```lua
-- Compatibility layer for action bar APIs
local ActionBarAPI = {}

if C_ActionBar then
    -- 12.0.0+
    ActionBarAPI.GetActionInfo = C_ActionBar.GetActionInfo
    ActionBarAPI.GetActionTexture = C_ActionBar.GetActionTexture
    ActionBarAPI.GetActionCooldown = C_ActionBar.GetActionCooldown
    ActionBarAPI.HasAction = C_ActionBar.HasAction
    ActionBarAPI.IsUsableAction = C_ActionBar.IsUsableAction
    ActionBarAPI.PickupAction = C_ActionBar.PickupAction
else
    -- Pre-12.0.0 (fallback to globals)
    ActionBarAPI.GetActionInfo = GetActionInfo
    ActionBarAPI.GetActionTexture = GetActionTexture
    ActionBarAPI.GetActionCooldown = GetActionCooldown
    ActionBarAPI.HasAction = HasAction
    ActionBarAPI.IsUsableAction = IsUsableAction
    ActionBarAPI.PickupAction = PickupAction
end

-- Combat log compatibility
local CombatLogAPI = {}
if C_CombatLog then
    CombatLogAPI.GetCurrentEventInfo = C_CombatLog.GetCurrentEventInfo
else
    CombatLogAPI.GetCurrentEventInfo = CombatLogGetCurrentEventInfo
end

-- Secret value safe wrapper
local function SafeGetActionInfo(slot)
    local actionType, id, subType = ActionBarAPI.GetActionInfo(slot)
    if issecretvalue and issecretvalue(id) then
        return nil, nil, nil, true  -- Last return indicates secret
    end
    return actionType, id, subType, false
end
```

**Result:** Addon works on both 11.x and 12.0+, handles secret values gracefully

### Example 5: Damage Meter Addon (11.x -> 12.0.0) - ⛔ NO LONGER POSSIBLE

> **⚠️ VERIFIED JANUARY 2026:** Third-party damage meters **CANNOT function in WoW 12.0.0+**. Both paths are blocked:
> 1. Combat log events throw `ADDON_ACTION_FORBIDDEN`
> 2. C_DamageMeter API data is protected as "secret values"

**Before (11.x - Combat Log Parsing) - BLOCKED:**
```lua
-- ❌ THIS CODE THROWS ADDON_ACTION_FORBIDDEN IN 12.0.0:
local damageData = {}
local frame = CreateFrame("Frame")
frame:RegisterEvent("COMBAT_LOG_EVENT_UNFILTERED")  -- ADDON_ACTION_FORBIDDEN!
frame:SetScript("OnEvent", function()
    -- This handler will NEVER fire - registration itself fails
end)
```

**Attempted Workaround (C_DamageMeter API) - ALSO BLOCKED:**
```lua
-- ❌ THIS CODE FAILS WITH "secret value" ERRORS:
local function TryToGetData()
    local available = C_DamageMeter.IsDamageMeterAvailable()  -- Returns true!

    if available then
        local sessionData = C_DamageMeter.GetCombatSessionFromType(
            Enum.DamageMeterSessionType.Overall,
            Enum.DamageMeterType.DamageDone
        )

        for _, source in ipairs(sessionData.combatSources) do
            -- These work (cosmetic only):
            print(source.classFilename)  -- "WARRIOR"
            print(source.isLocalPlayer)  -- true

            -- These FAIL with "attempt to compare (a secret value)":
            if source.name == "SomePlayer" then  -- ERROR!
                -- Cannot compare secret values
            end
            local damage = source.totalAmount  -- SECRET, unusable
            local dps = source.amountPerSecond -- SECRET, unusable
        end
    end
end
```

**The Only Valid "Migration" - Show Users a Message:**
```lua
-- ✅ The only thing damage meter addons can do in 12.0.0+:
local function OnAddonLoaded(self, event, addonName)
    if addonName == "YourDamageMeter" then
        -- Check WoW version
        local _, _, _, tocVersion = GetBuildInfo()
        if tocVersion >= 120000 then
            -- 12.0.0+ - damage meters cannot function
            print("|cFFFF6600[YourDamageMeter]|r WoW 12.0.0 has blocked damage meter functionality.")
            print("|cFFFF6600[YourDamageMeter]|r Please use Blizzard's built-in meter (Shift+P or Encounter Journal).")

            -- Optionally disable the addon's UI
            if YourDamageMeter.MainFrame then
                YourDamageMeter.MainFrame:Hide()
            end
        end
    end
end

local frame = CreateFrame("Frame")
frame:RegisterEvent("ADDON_LOADED")
frame:SetScript("OnEvent", OnAddonLoaded)
```

**What This Means for Existing Damage Meters:**
- **Recount, Skada, Details!** and similar addons **cannot be fixed** for 12.0.0+
- There is NO migration path - both combat log AND C_DamageMeter are blocked
- Players must use Blizzard's built-in damage meter
- Addon authors should display clear messages explaining this to users

**Why C_DamageMeter Doesn't Work (Verified):**

| Field | Status | What You Get |
|-------|--------|--------------|
| `name` | 🔒 SECRET | `<no value>` - comparison throws error |
| `totalAmount` | 🔒 SECRET | `<no value>` - unusable |
| `amountPerSecond` | 🔒 SECRET | `<no value>` - unusable |
| `sourceGUID` | 🔒 SECRET | `<no value>` - unusable |
| `classFilename` | ✅ Accessible | Works, but useless without names/amounts |
| `isLocalPlayer` | ✅ Accessible | Works, but useless without damage data |

**Blizzard's Intent:**
This is intentional. C_DamageMeter exists for Blizzard's OWN built-in UI, not for third-party addons. The "addon disarmament" in 12.0.0 was designed to prevent third-party combat analysis addons from functioning.

---

## Migration Quick Reference Table

### 11.x API Migrations

| Old API | New API | Version | Notes |
|---------|---------|---------|-------|
| `GetSpellInfo(id)` | `C_Spell.GetSpellName(id)` | 11.0.0 | Returns name only |
| `GetSpellCooldown(id)` | `C_Spell.GetSpellCooldown(id)` | 11.0.0 | Returns table |
| `UnitAura(unit, i)` | `C_UnitAuras.GetAuraDataByIndex(unit, i)` | 10.2.5 | Returns table |
| `GetContainerNumSlots(bag)` | `C_Container.GetContainerNumSlots(bag)` | 9.0.0 | Namespace move |
| `GetMerchantItemInfo(i)` | `C_MerchantFrame.GetItemInfo(i)` | 11.0.5 | Returns table |
| `GetNumSpecializationsForClassID(id)` | `C_SpecializationInfo.GetNumSpecializationsForClassID(id)` | 11.0.5 | Direct replacement |
| `IsOnGlueScreen()` | `C_Glue.IsOnGlueScreen()` | 11.0.5 | Namespace move |
| `C_ChallengeMode.GetCompletionInfo()` | `C_ChallengeMode.GetChallengeCompletionInfo()` | 11.0.5 | Renamed |
| `SetSpecialization(idx)` | `C_SpecializationInfo.SetSpecialization(idx)` | 11.1.0 | Namespace move |
| `ConsoleAddMessage(msg)` | `ConsoleEcho(msg)` | 11.1.0 | Renamed |
| `AddTradeMoney(copper)` | `C_TradeInfo.AddTradeMoney(copper)` | 11.1.5 | Namespace move |
| `GetSpecialization()` | `C_SpecializationInfo.GetSpecialization()` | 11.2.0 | Namespace move |
| `SendChatMessage(...)` | `C_ChatInfo.SendChatMessage(...)` | 11.2.0 | Namespace move |
| `IsSpellKnown(id)` | `C_SpellBook.IsSpellKnown(id)` | 11.2.0 | Namespace move |
| `IsSpellOverlayed(id)` | `C_SpellActivationOverlay.IsSpellOverlayed(id)` | 11.2.0 | Namespace move |
| `AcceptSockets()` | `C_ItemSocketInfo.AcceptSockets()` | 11.2.5 | Namespace move |
| `GetNumSockets()` | `C_ItemSocketInfo.GetNumSockets()` | 11.2.5 | Namespace move |
| `GetPetTalentTree()` | `C_PetInfo.GetPetTalentTree()` | 11.2.5 | Namespace move |
| `GetBindingByKey(key)` | `C_KeyBindings.GetBindingByKey(key)` | 11.2.7 | Namespace move |
| `JoinBattlefield(idx)` | `C_PvP.JoinBattlefield(idx)` | 11.2.7 | Namespace move |
| `CanUpgradeExpansion()` | `CanUpgradeToCurrentExpansion()` | 11.2.7 | Renamed |

### 12.x API Migrations (Midnight)

| Old API | New API | Version | Notes |
|---------|---------|---------|-------|
| `GetMouseFocus()` | `GetMouseFoci()[1]` | 12.0.0 | Returns table; use compat wrapper |
| `GetActionInfo(slot)` | `C_ActionBar.GetActionInfo(slot)` | 12.0.0 | Secret values in combat - see [12a](12a_Secret_Safe_APIs.md) |
| `GetActionTexture(slot)` | `C_ActionBar.GetActionTexture(slot)` | 12.0.0 | Namespace move |
| `GetActionCooldown(slot)` | `C_ActionBar.GetActionCooldown(slot)` | 12.0.0 | Namespace move |
| `HasAction(slot)` | `C_ActionBar.HasAction(slot)` | 12.0.0 | Namespace move |
| `IsUsableAction(slot)` | `C_ActionBar.IsUsableAction(slot)` | 12.0.0 | Namespace move |
| `PickupAction(slot)` | `C_ActionBar.PickupAction(slot)` | 12.0.0 | Namespace move |
| `CombatLogGetCurrentEventInfo()` | `C_CombatLog.GetCurrentEventInfo()` | 12.0.0 | Namespace move |
| `DoEmote(emote, target)` | `C_ChatInfo.DoEmote(emote, target)` | 12.0.0 | Namespace move |
| `CancelEmote()` | `C_ChatInfo.CancelEmote()` | 12.0.0 | Namespace move |
| `BNSendWhisper(id, msg)` | `C_BattleNet.SendWhisper(id, msg)` | 12.0.0 | Namespace move |
| `BNSendGameData(...)` | `C_BattleNet.SendGameData(...)` | 12.0.0 | Namespace move |
| `IsEncounterInProgress()` | `C_InstanceEncounter.IsEncounterInProgress()` | 12.0.0 | Namespace move |
| `GetDeathRecapEvents()` | `C_DeathRecap.GetDeathRecapEvents()` | 12.0.0 | Namespace move |

---

<!-- CLAUDE_SKIP_START -->
## Best Practices for Future-Proof Addons

### 1. Use Compatibility Wrappers
Create wrapper functions for commonly used APIs that might change.

### 2. Check Function Existence
Always check if new APIs exist before using them.

### 3. Version-Gate Features
Use TOC version checks for expansion-specific features.

### 4. Document Version Requirements
Clearly state minimum/maximum supported versions.

### 5. Monitor API Deprecation Warnings
Test with `/console scriptErrors 1` to catch warnings.

### 6. Subscribe to Update Notifications
- Follow Warcraft Wiki
- Join WoW Addon Discord
- Monitor CurseForge/GitHub issues

### 7. Test on PTR
Test your addon on Public Test Realm before patches go live.

### 8. Keep Libraries Updated
Regularly update embedded libraries (Ace3, LibStub, etc.).

---

## Resources & Links

### Essential Bookmarks

**Warcraft Wiki - API Changes**
```
https://warcraft.wiki.gg/wiki/Category:API_changes
```

**WoW API Documentation**
```
https://warcraft.wiki.gg/wiki/World_of_Warcraft_API
```

**WoW Forums - UI & Macro**
```
https://us.forums.blizzard.com/en/wow/c/ui-and-macro/
```

**GitHub - WoW UI Source**
```
https://github.com/Gethe/wow-ui-source
```

### Community Resources

- **WoW AddOn Discord**: Real-time help with API changes
- **CurseForge Forums**: Addon-specific discussions
- **Wago.io**: Addon hosting with version tracking

---

## Conclusion

API migrations are a regular part of WoW addon development. By:
- Checking Warcraft Wiki for API changes
- Using compatibility wrappers
- Testing thoroughly
- Following migration patterns

You can keep your addons working across expansions with minimal effort.

**Remember:** Check API changes BEFORE each major patch, not after your addon breaks!

---

**Related Guides:**
- [12a_Secret_Safe_APIs.md](12a_Secret_Safe_APIs.md) - **Complete Secret Values API reference**
- [10_Advanced_Techniques.md](10_Advanced_Techniques.md) - Cross-client compatibility
- [04_Addon_Structure.md](04_Addon_Structure.md) - TOC file versioning
- [05_Patterns_And_Best_Practices.md](05_Patterns_And_Best_Practices.md) - Code patterns

**Next Steps:**
1. Bookmark Warcraft Wiki API changes pages
2. Create compatibility wrapper template
3. Set up automated testing for new patches
4. Join WoW Addon Discord for early warnings

---

**Version:** 2.4 - Deduplicated secret values content; moved detailed API docs to 12a_Secret_Safe_APIs.md
**Last Updated:** 2026-01-28
**Coverage:** API changes from WoW 9.0 through 12.0.0

<!-- CLAUDE_SKIP_END -->
