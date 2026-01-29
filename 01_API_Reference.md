# WoW API Reference Guide

## Table of Contents
1. [Overview](#overview)
2. [API Structure](#api-structure)
3. [API Categories](#api-categories)
4. [New 12.0.0 APIs](#new-1200-apis)
5. [Secret Values System](#secret-values-system)
6. [Lua Extensions](#lua-extensions)
7. [Common API Patterns](#common-api-patterns)
8. [API Usage Examples](#api-usage-examples)
9. [API Migration Guide](#api-migration-guide)
10. [Finding API Documentation](#finding-api-documentation)

---

## Overview
This document provides comprehensive information about the World of Warcraft API structure and usage patterns.

**Statistics**:
- **Total API Documentation Files**: 513+
- **Total Events**: 1,700+
- **API Systems**: 200+
- **Current Version**: 12.0.0 (Midnight)

## API Structure

### Namespace APIs (C_* APIs)

Modern WoW APIs use the C_* namespace pattern. These are table-based APIs that group related functions.

**Example**:
```lua
C_ChatInfo.GetChannelInfo(channelID)
C_Map.GetBestMapForUnit("player")
C_Timer.After(3, function() print("3 seconds passed") end)
```

**Benefits**:
- Clear organization by functionality
- Less global namespace pollution
- Better IntelliSense support in IDEs
- Official and maintained by Blizzard

### Global APIs (Legacy)

Older APIs exist in the global namespace. Many are still actively used, though Blizzard continues migrating them to C_* namespaces.

**Examples**:
```lua
UnitName("player")
GetItemInfo(itemID)
CreateFrame("Frame", "MyFrame", UIParent)
```

**Note**: As of 12.0.0, many previously global functions have been moved to C_* namespaces. See the [API Migration Guide](#api-migration-guide) section for details.

## Common API Categories

### 1. Unit APIs
Functions for querying information about units (players, NPCs, pets, etc.)

**Key Functions**:
```lua
UnitName("unit")                    -- Get unit's name
UnitHealth("unit")                  -- Get current health (SECRET in combat - 12.0.0)
UnitHealthMax("unit")               -- Get maximum health (SECRET in combat - 12.0.0)
UnitHealthMissing("unit")           -- Get missing health (12.0.0)
UnitHealthPercent("unit", usePredicted, curveConstant) -- Get health % (NOT SECRET - 12.0.0)
UnitPower("unit", powerType)        -- Get current power (SECRET in combat - 12.0.0)
UnitPowerMax("unit", powerType)     -- Get maximum power (SECRET in combat - 12.0.0)
UnitPowerMissing("unit", powerType) -- Get missing power (12.0.0)
UnitPowerPercent("unit", powerType, curveConstant) -- Get power % (NOT SECRET - 12.0.0)
UnitClass("unit")                   -- Get class name and class file
UnitRace("unit")                    -- Get race name and race file
UnitLevel("unit")                   -- Get level
UnitExists("unit")                  -- Check if unit exists
UnitIsPlayer("unit")                -- Check if unit is a player
UnitIsHumanPlayer("unit")           -- Check if unit is a human player (12.0.0)
UnitIsDead("unit")                  -- Check if unit is dead
UnitAffectingCombat("unit")         -- Check if in combat
UnitIsLieutenant("unit")            -- Check if unit is a lieutenant (12.0.0)
UnitIsMinion("unit")                -- Check if unit is a minion (12.0.0)
UnitCreatureID("unit")              -- Get creature ID directly (12.0.0)
UnitBuff("unit", index or "name")   -- Get buff information
UnitDebuff("unit", index or "name") -- Get debuff information
UnitGetTotalAbsorbs("unit")         -- Get absorb shield total (SECRET in combat - 12.0.0)
UnitGetTotalHealAbsorbs("unit")     -- Get heal absorb total (12.0.0)
UnitGetIncomingHeals("unit", healer)-- Get incoming heals (SECRET in combat - 12.0.0)
```

**Secret Values in Unit APIs (12.0.0 - CRITICAL):**

In WoW 12.0.0, the following Unit APIs return **secret values** during combat:
- `UnitHealth()`, `UnitHealthMax()`
- `UnitPower()`, `UnitPowerMax()`
- `UnitGetTotalAbsorbs()`, `UnitGetIncomingHeals()`

Secret values **CANNOT** be used for arithmetic, comparisons, or string concatenation:
```lua
-- THESE FAIL with secret values:
local percent = UnitHealth("target") / UnitHealthMax("target")  -- ERROR!
if UnitHealth("target") > 0 then  -- ERROR!
print("Health: " .. UnitHealth("target"))  -- ERROR!
```

**Use UnitHealthPercent/UnitPowerPercent for custom UI calculations:**
```lua
-- Returns NON-SECRET percentage (0-100)
local healthPct = UnitHealthPercent("target", false, CurveConstants.ScaleTo100)
local width = (healthPct / 100) * BAR_WIDTH  -- Safe arithmetic!
```

**Native StatusBar frames accept secret values directly:**
```lua
local bar = CreateFrame("StatusBar", nil, parent)
bar:SetMinMaxValues(0, UnitHealthMax("target"))  -- Works with secrets!
bar:SetValue(UnitHealth("target"))               -- Works with secrets!
```

See `12a_Secret_Safe_APIs.md` for the complete secret values reference, and `12_API_Migration_Guide.md` for migration patterns.

**New Event Callback System (12.0.0)**:
```lua
-- Register for unit-specific events without frame overhead
RegisterEventCallback("UNIT_HEALTH", function(unit)
    print(unit .. " health changed")
end)

UnregisterEventCallback("UNIT_HEALTH", callbackID)

-- Unit-specific event registration
RegisterUnitEventCallback("UNIT_HEALTH", "player", function(unit)
    print("Player health changed to: " .. UnitHealth("player"))
end)

UnregisterUnitEventCallback("UNIT_HEALTH", "player", callbackID)
```

**Unit Tokens**:
- `"player"` - The player
- `"target"` - Current target
- `"pet"` - Player's pet
- `"party1"` through `"party4"` - Party members
- `"raid1"` through `"raid40"` - Raid members
- `"boss1"` through `"boss5"` - Boss frames
- `"arena1"` through `"arena5"` - Arena opponents
- `"mouseover"` - Unit under mouse cursor
- `"focus"` - Focus target
- `"vehicleNN"` - Vehicle passengers

### 2. Item and Inventory APIs

**C_Item Namespace**:
```lua
C_Item.GetItemInfo(itemID)
C_Item.GetItemCount(itemID, includeBank, includeCharges)
C_Item.IsItemInRange(itemID, "unit")
C_Item.GetItemQuality(itemLocation)
```

**C_ItemSocketInfo Namespace (11.2.5+)**:
```lua
-- Replaces global Socket APIs
C_ItemSocketInfo.GetSocketInfo(socketIndex)
C_ItemSocketInfo.GetNumSockets()
C_ItemSocketInfo.GetExistingSocketLink(socketIndex)
C_ItemSocketInfo.SocketContainerItem(bagID, slotIndex)
C_ItemSocketInfo.CompleteSocketing()
C_ItemSocketInfo.CloseSocketInfo()
```

**Global Functions**:
```lua
GetItemInfo(itemID or "itemLink" or "itemString")
    -- Returns: name, link, quality, iLevel, reqLevel, class, subclass,
    --          maxStack, equipSloc, texture, sellPrice, classID, subclassID,
    --          bindType, expacID, setID, isCraftingReagent

GetContainerItemInfo(bagID, slotIndex)
GetContainerNumSlots(bagID)
UseContainerItem(bagID, slotIndex)
PickupContainerItem(bagID, slotIndex)
```

**Bag IDs**:
- `0` - Backpack
- `1-4` - Bag slots
- `5` - Reagent bag (if equipped)
- `BANK_CONTAINER` or `-1` - Bank
- `6-12` - Bank bag slots

### 3. Spell and Ability APIs

**C_Spell Namespace**:
```lua
C_Spell.GetSpellInfo(spellID)
C_Spell.IsSpellInRange(spellID, "unit")
C_Spell.GetSpellCooldown(spellID)
C_Spell.DoesSpellExist(spellID)

-- New in 12.0.0
C_Spell.GetSpellChargeDuration(spellID)      -- Duration per charge
C_Spell.GetSpellCooldownDuration(spellID)    -- Base cooldown duration
C_Spell.IsExternalDefensive(spellID)         -- External defensive cooldown
C_Spell.IsSpellCrowdControl(spellID)         -- CC ability check
C_Spell.IsSpellImportant(spellID)            -- Important spell marker
C_Spell.IsPriorityAura(spellID)              -- Priority aura check
C_Spell.IsSelfBuff(spellID)                  -- Self-buff check
```

**C_SpellBook Namespace (11.2.0+)**:
```lua
-- Replaces IsSpellKnown()
C_SpellBook.IsSpellKnown(spellID)
C_SpellBook.GetSpellBookItemInfo(slotIndex, spellBookType)
C_SpellBook.GetNumSpellBookSkillLines()
```

**C_SpellDiminish Namespace (12.0.0)**:
```lua
-- Diminishing returns tracking
C_SpellDiminish.GetDiminishingReturnsByUnit("unit")
C_SpellDiminish.GetDiminishingReturnsForSpell(spellID)
C_SpellDiminish.GetDiminishingCategory(spellID)
```

**Global Functions**:
```lua
GetSpellInfo(spellID or "spellName" or spellIndex, "bookType")
    -- Returns: name, rank, icon, castTime, minRange, maxRange, spellID, originalIcon

GetSpellCooldown(spellID or "spellName")
    -- Returns: start, duration, enabled, modRate

-- DEPRECATED in 11.2.0: Use C_SpellBook.IsSpellKnown() instead
IsSpellKnown(spellID)
IsPlayerSpell(spellID)
CastSpellByID(spellID)
CastSpellByName("spellName")
```

### 4. Chat and Communication APIs

**C_ChatInfo Namespace**:
```lua
C_ChatInfo.GetChannelInfo(channelID)
C_ChatInfo.GetChannelInfoFromIdentifier(channelIdentifier)
C_ChatInfo.GetChannelRosterInfo(channelIndex, rosterIndex)
C_ChatInfo.CanPlayerSpeakLanguage(languageID)

-- 11.2.0+: SendChatMessage moved here
C_ChatInfo.SendChatMessage("message", "chatType", languageID, "channelOrTarget")

-- 12.0.0+: Emote functions moved from global
C_ChatInfo.DoEmote("emote", "target")
C_ChatInfo.CancelEmote()
```

**C_BattleNet Namespace (12.0.0)**:
```lua
-- Replaces global BN* functions
C_BattleNet.SendWhisper(presenceID, "message")
C_BattleNet.GetFriendInfo(friendIndex)
C_BattleNet.GetNumFriends()
C_BattleNet.GetFOFInfo(accountID)
```

**Global Functions (Legacy)**:
```lua
-- DEPRECATED in 11.2.0: Use C_ChatInfo.SendChatMessage() instead
SendChatMessage("message", "chatType", languageID, "channelOrTarget")
```

**Chat Types**:
- `"SAY"` - Say
- `"YELL"` - Yell
- `"PARTY"` - Party chat
- `"RAID"` - Raid chat
- `"GUILD"` - Guild chat
- `"OFFICER"` - Officer chat
- `"WHISPER"` - Whisper (requires target name)
- `"CHANNEL"` - Chat channel (requires channel ID)
- `"EMOTE"` - Emote
- `"AFK"` - AFK message
- `"DND"` - DND message

### 5. Map and Position APIs

**C_Map Namespace**:
```lua
C_Map.GetBestMapForUnit("unit")
C_Map.GetPlayerMapPosition(uiMapID, "unit")
    -- Returns: positionObject with :GetXY() method

C_Map.GetMapInfo(uiMapID)
    -- Returns: info table with name, mapID, parentMapID, mapType, etc.

C_Map.GetWorldPosFromMapPos(uiMapID, mapPosition)
```

### 6. Timer and Frame APIs

**C_Timer Namespace**:
```lua
C_Timer.After(seconds, callback)
    -- Execute callback after X seconds (once)

C_Timer.NewTimer(seconds, callback)
    -- Returns timer object with :Cancel() method

C_Timer.NewTicker(interval, callback, iterations)
    -- Repeating timer, optional iteration limit
```

**Alternative - Frame OnUpdate**:
```lua
local frame = CreateFrame("Frame")
local elapsed = 0
frame:SetScript("OnUpdate", function(self, deltaTime)
    elapsed = elapsed + deltaTime
    if elapsed >= 1.0 then  -- Every 1 second
        -- Do something
        elapsed = 0
    end
end)
```

### 7. Quest APIs

**C_QuestLog Namespace**:
```lua
C_QuestLog.GetInfo(questLogIndex)
C_QuestLog.GetNumQuestLogEntries()
C_QuestLog.GetQuestObjectives(questID)
C_QuestLog.GetSelectedQuest()
C_QuestLog.SetSelectedQuest(questID)
C_QuestLog.IsQuestFlaggedCompleted(questID)
```

### 8. Achievement APIs

**C_Achievement Namespace**:
```lua
C_AchievementInfo.GetAchievementInfo(achievementID)
C_AchievementInfo.GetNumAchievements()
C_AchievementInfo.GetCriteriaInfo(achievementID, criteriaIndex)
```

### 9. Auction House APIs

**C_AuctionHouse Namespace**:
```lua
C_AuctionHouse.GetBrowseResults()
C_AuctionHouse.GetItemKeyInfo(itemKey)
C_AuctionHouse.SearchForItemKeys(itemKeys, sorts)
C_AuctionHouse.GetNumBids()
C_AuctionHouse.PlaceBid(auctionID, bidAmount)
```

### 10. Profession APIs

**C_TradeSkillUI Namespace**:
```lua
C_TradeSkillUI.GetRecipeInfo(recipeID)
C_TradeSkillUI.GetRecipeNumReagents(recipeID)
C_TradeSkillUI.GetRecipeReagentInfo(recipeID, reagentIndex)
C_TradeSkillUI.CraftRecipe(recipeID, numCasts)
```

---

## New 12.0.0 APIs

### C_ActionBar - Action Bar Management
Replaces global action bar functions with a structured namespace.

```lua
-- Get action bar slot information
C_ActionBar.GetActionInfo(slot)
C_ActionBar.GetActionTexture(slot)
C_ActionBar.GetActionText(slot)
C_ActionBar.GetActionCooldown(slot)

-- Action bar operations
C_ActionBar.HasAction(slot)
C_ActionBar.IsCurrentAction(slot)
C_ActionBar.IsAutoRepeatAction(slot)
C_ActionBar.IsUsableAction(slot)
C_ActionBar.IsConsumableAction(slot)
C_ActionBar.IsStackableAction(slot)
C_ActionBar.IsAttackAction(slot)

-- Pickup and placement
C_ActionBar.PickupAction(slot)
C_ActionBar.PlaceAction(slot)
C_ActionBar.PickupSpellBookItem(slotIndex, bookType)

-- Action counts
C_ActionBar.GetActionCount(slot)
C_ActionBar.GetActionCharges(slot)
```

### C_CombatLog - Combat Log Management
Structured access to combat log data.

```lua
-- Combat log event access
C_CombatLog.GetCurrentEventInfo()
C_CombatLog.GetNumCombatLogFilters()
C_CombatLog.GetCombatLogFilterInfo(filterIndex)

-- Combat log management
C_CombatLog.ClearCombatLog()
C_CombatLog.SetCombatLogFilterEnabled(filterIndex, enabled)
```

### C_DamageMeter - Official Damage Meter API
Blizzard's official damage meter support for addons.

```lua
-- Damage/healing statistics
C_DamageMeter.GetCombatData()
C_DamageMeter.GetPlayerDamage("unit")
C_DamageMeter.GetPlayerHealing("unit")
C_DamageMeter.GetEncounterInfo()

-- Combat segments
C_DamageMeter.GetCurrentSegment()
C_DamageMeter.GetSegmentInfo(segmentIndex)
C_DamageMeter.GetNumSegments()
```

### C_EncounterTimeline - Boss Ability Timeline
Access to encounter ability timers and phases.

```lua
-- Timeline data
C_EncounterTimeline.GetEncounterTimeline(encounterID)
C_EncounterTimeline.GetCurrentPhase()
C_EncounterTimeline.GetPhaseInfo(phaseIndex)
C_EncounterTimeline.GetAbilityTimers()

-- Ability predictions
C_EncounterTimeline.GetNextAbility()
C_EncounterTimeline.GetAbilityCooldown(abilityID)
```

### C_EncounterWarnings - Encounter Warnings
Built-in encounter warning system.

```lua
-- Warning management
C_EncounterWarnings.GetActiveWarnings()
C_EncounterWarnings.GetWarningInfo(warningID)
C_EncounterWarnings.AcknowledgeWarning(warningID)

-- Warning configuration
C_EncounterWarnings.IsWarningEnabled(warningType)
C_EncounterWarnings.SetWarningEnabled(warningType, enabled)
```

### C_InstanceEncounter - Encounter State
Replaces `IsEncounterInProgress()` with expanded functionality.

```lua
-- Encounter state
C_InstanceEncounter.IsEncounterInProgress()  -- Replaces global IsEncounterInProgress()
C_InstanceEncounter.GetEncounterID()
C_InstanceEncounter.GetEncounterName()
C_InstanceEncounter.GetEncounterDifficulty()

-- Instance information
C_InstanceEncounter.GetInstanceInfo()
C_InstanceEncounter.IsInInstance()
C_InstanceEncounter.GetBossInfo(bossIndex)
```

### C_RestrictedActions - Addon Restriction Management
Manage addon restrictions and secure execution context.

```lua
-- Check restrictions
C_RestrictedActions.IsActionRestricted(actionType)
C_RestrictedActions.GetRestrictionReason(actionType)
C_RestrictedActions.CanPerformAction(actionType)

-- Context checking
C_RestrictedActions.IsInCombatLockdown()
C_RestrictedActions.IsSecureContext()
```

### C_CombatAudioAlert - TTS Accessibility
Text-to-speech accessibility features for combat.

```lua
-- Audio alerts
C_CombatAudioAlert.PlayAlert(alertType)
C_CombatAudioAlert.SpeakText("text")
C_CombatAudioAlert.SetAlertEnabled(alertType, enabled)

-- Configuration
C_CombatAudioAlert.GetVoiceSettings()
C_CombatAudioAlert.SetVoiceSettings(settings)
```

### C_TransmogOutfitInfo - New Transmog System
Expanded transmog outfit management (replaces older transmog APIs).

```lua
-- Outfit management
C_TransmogOutfitInfo.GetOutfits()
C_TransmogOutfitInfo.GetOutfitInfo(outfitID)
C_TransmogOutfitInfo.CreateOutfit("name", sources)
C_TransmogOutfitInfo.DeleteOutfit(outfitID)
C_TransmogOutfitInfo.ModifyOutfit(outfitID, sources)

-- Application
C_TransmogOutfitInfo.ApplyOutfit(outfitID)
C_TransmogOutfitInfo.CanApplyOutfit(outfitID)
C_TransmogOutfitInfo.GetOutfitCost(outfitID)
```

### C_DeathRecap - Death Recap Information
Access death recap data programmatically.

```lua
-- Death information
C_DeathRecap.GetDeathRecapInfo()
C_DeathRecap.GetDeathInfo(deathIndex)
C_DeathRecap.GetNumDeaths()

-- Damage sources
C_DeathRecap.GetDamageSources()
C_DeathRecap.GetDeathDetails()
```

### C_Housing - Player Housing System
New player housing feature in Midnight. See dedicated Housing guide for full details.

```lua
-- Housing basics
C_Housing.GetPlayerHouse()
C_Housing.IsInHouse()
C_Housing.GetHouseInfo(houseID)

-- Furniture and decoration
C_Housing.GetPlacedFurniture()
C_Housing.PlaceFurniture(furnitureID, position, rotation)
C_Housing.RemoveFurniture(placementID)

-- Visitors
C_Housing.GetHouseVisitors()
C_Housing.InviteToHouse("playerName")
C_Housing.SetHousePermissions(permissionType, value)
```

### Utility Namespaces

**C_CurveUtil** - Curve/Animation Utilities:
```lua
C_CurveUtil.GetCurveValue(curveID, progress)
C_CurveUtil.GetCurvePointCount(curveID)
```

**C_DurationUtil** - Duration Formatting:
```lua
C_DurationUtil.FormatDuration(seconds)
C_DurationUtil.FormatShortDuration(seconds)
C_DurationUtil.ParseDuration("durationString")
```

**C_StringUtil** - String Utilities:
```lua
C_StringUtil.SplitString(delimiter, "string")
C_StringUtil.TrimString("string")
C_StringUtil.FormatLargeNumber(number)
```

**C_ColorUtil** - Color Utilities:
```lua
C_ColorUtil.CreateColor(r, g, b, a)
C_ColorUtil.GetColorFromHexString("hexColor")
C_ColorUtil.ConvertToHexString(r, g, b)
```

---

## Secret Values System

New in 12.0.0, WoW introduces a "secret values" system that protects sensitive combat data from unauthorized addon access. This is part of Blizzard's ongoing effort to prevent automation and botting.

### What Are Secret Values?

Secret values are wrapped data that cannot be directly read or manipulated by addon code. They're used for sensitive combat-related information that could be exploited for automation.

### Built-in Functions

```lua
-- Check if a value is a secret
issecretvalue(value)
    -- Returns: true if value is a secret, false otherwise

-- Check if current context can access a secret value
canaccessvalue(secretValue)
    -- Returns: true if the value can be accessed, false otherwise

-- Wrap a value as a secret (internal use)
secretwrap(value)
    -- Returns: wrapped secret value

-- Remove secret values from a table (for logging/display)
scrubsecretvalues(table)
    -- Returns: table with secrets replaced with placeholder strings
```

### How Secrets Affect Addons

```lua
-- Example: Accessing unit health in different contexts
local function OnCombatLogEvent()
    local health = UnitHealth("target")

    -- In some contexts, health may be a secret value
    if issecretvalue(health) then
        -- Cannot directly use this value for automation
        -- Can only pass it to secure functions
        print("Health is protected")
    else
        -- Normal usage
        print("Target health: " .. health)
    end
end
```

### Restrictions for Tainted Code

When code is "tainted" (execution started from insecure addon code), certain values become secrets:

1. **Combat-related values**: Target health, power, position in certain contexts
2. **Action state**: Whether actions can be used, cooldown states
3. **Unit targeting information**: Precise unit positions, facing angles

### Working with Secrets

```lua
-- String concatenation with secrets (12.0.0)
-- Use string.concat() which handles secrets properly
local message = string.concat("Health: ", secretHealthValue)

-- Scrub secrets before logging
local debugInfo = {
    targetHealth = UnitHealth("target"),
    targetPower = UnitPower("target")
}
local safeInfo = scrubsecretvalues(debugInfo)
print("Debug: " .. safeInfo.targetHealth)  -- Will show "[SECRET]" if protected
```

### Best Practices

1. **Check before arithmetic**: Always check `issecretvalue()` before doing math with potentially secret values
2. **Use string.concat()**: For string building with potentially secret values
3. **Scrub for display**: Use `scrubsecretvalues()` before displaying debug information
4. **Don't fight the system**: If a value is secret, it's intentionally protected

---

## Lua Extensions

WoW includes custom Lua extensions beyond standard Lua 5.1.

### Table Functions

```lua
-- 11.1.7+: Pre-allocate table size for performance
table.create(arraySizeHint, nodeSizeHint)
    -- Creates a table with pre-allocated space
    -- arraySizeHint: expected number of array elements
    -- nodeSizeHint: expected number of hash table entries

-- Example usage
local myTable = table.create(100, 10)  -- 100 array slots, 10 hash entries
for i = 1, 100 do
    myTable[i] = i * 2
end

-- 11.2.5+: Count table entries (including non-array)
table.count(tbl)
    -- Returns: number of entries in the table
    -- More accurate than #tbl for mixed tables

-- Example
local mixed = {1, 2, 3, foo = "bar", baz = "qux"}
print(#mixed)          -- Returns 3 (only array part)
print(table.count(mixed))  -- Returns 5 (all entries)

-- Standard WoW table functions
tinsert(table, [pos,] value)  -- Insert into table
tremove(table [, pos])         -- Remove from table
wipe(table)                    -- Clear all table contents
CopyTable(table)               -- Deep copy a table
tContains(table, value)        -- Check if table contains value
tIndexOf(table, value)         -- Find index of value
```

### String Functions

```lua
-- 12.0.0+: Safe string concatenation (handles secrets)
string.concat(...)
    -- Concatenates strings, properly handling secret values
    -- Returns: concatenated string

-- Example
local msg = string.concat("Player: ", playerName, " has ", healthValue, " HP")

-- Standard WoW string functions
strsplit(delimiter, string)    -- Split string by delimiter
strjoin(delimiter, ...)        -- Join strings with delimiter
strtrim(string)                -- Remove leading/trailing whitespace
strmatch(string, pattern)      -- Pattern matching
format(formatString, ...)      -- String formatting (same as string.format)
```

### Math Functions

```lua
-- Blizzard extensions
Clamp(value, min, max)         -- Clamp value between min and max
Lerp(startValue, endValue, amount)  -- Linear interpolation
Round(value)                   -- Round to nearest integer
Saturate(value)                -- Clamp between 0 and 1
```

### Utility Functions

```lua
-- Type checking
type(value)                    -- Standard Lua type
issecurevariable(table, key)   -- Check if variable is secure
issecure()                     -- Check if current execution is secure

-- Time
GetTime()                      -- Game time in seconds (high precision)
time()                         -- Real-world Unix timestamp
date(format, time)             -- Format time as string

-- Debugging
debugstack(start, top, bottom) -- Get stack trace
debuglocals(level)             -- Get local variables at stack level
```

---

## Common API Patterns

### 1. Checking Return Values
```lua
local name, realm = UnitName("player")
if not name then
    -- Unit doesn't exist
    return
end
```

### 2. Iterating Collections
```lua
for i = 1, C_QuestLog.GetNumQuestLogEntries() do
    local info = C_QuestLog.GetInfo(i)
    if info and not info.isHeader then
        print(info.title)
    end
end
```

### 3. Using Callbacks
```lua
C_Timer.After(5, function()
    print("Delayed message")
end)
```

### 4. Checking Existence Before Use
```lua
if C_Spell.DoesSpellExist(spellID) then
    local info = C_Spell.GetSpellInfo(spellID)
end
```

### 5. API Version Checking
```lua
-- Check if new API exists before using
if C_ActionBar and C_ActionBar.GetActionInfo then
    -- Use new 12.0.0 API
    local info = C_ActionBar.GetActionInfo(slot)
else
    -- Fall back to legacy
    local actionType, id, subType = GetActionInfo(slot)
end
```

### 6. Handling Secret Values
```lua
local function SafeHealthDisplay(unit)
    local health = UnitHealth(unit)
    if issecretvalue(health) then
        return "Protected"
    end
    return tostring(health)
end
```

## API Type Reference

**Common Types**:
- `number` - Lua number
- `string` - Lua string
- `cstring` - C string (null-terminated)
- `bool` - Boolean (true/false)
- `table` - Lua table
- `function` - Lua function
- `uiUnit` - Unit token string
- `WOWGUID` - WoW GUID string
- `luaIndex` - 1-based index
- `FileDataID` - File data ID number
- `itemID` - Item ID number
- `spellID` - Spell ID number
- `secretvalue` - Protected secret value (12.0.0)

---

## API Migration Guide

### 12.0.0 Migrations

| Old API (Deprecated) | New API (12.0.0+) |
|---------------------|-------------------|
| `GetActionInfo(slot)` | `C_ActionBar.GetActionInfo(slot)` |
| `GetActionTexture(slot)` | `C_ActionBar.GetActionTexture(slot)` |
| `HasAction(slot)` | `C_ActionBar.HasAction(slot)` |
| `IsCurrentAction(slot)` | `C_ActionBar.IsCurrentAction(slot)` |
| `IsUsableAction(slot)` | `C_ActionBar.IsUsableAction(slot)` |
| `PickupAction(slot)` | `C_ActionBar.PickupAction(slot)` |
| `DoEmote("emote")` | `C_ChatInfo.DoEmote("emote")` |
| `CancelEmote()` | `C_ChatInfo.CancelEmote()` |
| `BNSendWhisper(...)` | `C_BattleNet.SendWhisper(...)` |
| `BNGetFriendInfo(...)` | `C_BattleNet.GetFriendInfo(...)` |
| `IsEncounterInProgress()` | `C_InstanceEncounter.IsEncounterInProgress()` |
| Old Transmog APIs | `C_TransmogOutfitInfo.*` |

### 11.2.5 Migrations

| Old API (Deprecated) | New API (11.2.5+) |
|---------------------|-------------------|
| `GetSocketInfo(index)` | `C_ItemSocketInfo.GetSocketInfo(index)` |
| `GetNumSockets()` | `C_ItemSocketInfo.GetNumSockets()` |
| `GetExistingSocketLink(index)` | `C_ItemSocketInfo.GetExistingSocketLink(index)` |
| `SocketContainerItem(bag, slot)` | `C_ItemSocketInfo.SocketContainerItem(bag, slot)` |

### 11.2.0 Migrations

| Old API (Deprecated) | New API (11.2.0+) |
|---------------------|-------------------|
| `SendChatMessage(...)` | `C_ChatInfo.SendChatMessage(...)` |
| `IsSpellKnown(spellID)` | `C_SpellBook.IsSpellKnown(spellID)` |

### Compatibility Wrapper Example

```lua
-- Create compatibility layer for different WoW versions
local function GetActionInfoCompat(slot)
    if C_ActionBar and C_ActionBar.GetActionInfo then
        return C_ActionBar.GetActionInfo(slot)
    else
        return GetActionInfo(slot)
    end
end

local function SendChatMessageCompat(msg, chatType, language, channel)
    if C_ChatInfo and C_ChatInfo.SendChatMessage then
        C_ChatInfo.SendChatMessage(msg, chatType, language, channel)
    else
        SendChatMessage(msg, chatType, language, channel)
    end
end

local function IsSpellKnownCompat(spellID)
    if C_SpellBook and C_SpellBook.IsSpellKnown then
        return C_SpellBook.IsSpellKnown(spellID)
    else
        return IsSpellKnown(spellID)
    end
end
```

---

## API Documentation Files Location

All API documentation files are located at:
```
D:\Games\World of Warcraft\_retail_\Interface\+wow-ui-source+ (12.0.0)\Interface\AddOns\Blizzard_APIDocumentationGenerated\
```

## Using API Documentation Files

Each API documentation file follows this structure:

```lua
local SystemName = {
    Name = "SystemName",
    Type = "System",
    Namespace = "C_SystemName",  -- or "" for global

    Functions = {
        {
            Name = "FunctionName",
            Type = "Function",
            Arguments = {
                { Name = "argName", Type = "number", Nilable = false },
            },
            Returns = {
                { Name = "result", Type = "bool", Nilable = false },
            },
        },
    },

    Events = {
        {
            Name = "EVENT_NAME",
            Type = "Event",
            LiteralName = "EVENT_NAME",
            Payload = {
                { Name = "param1", Type = "number", Nilable = false },
            },
        },
    },

    Tables = {
        {
            Name = "StructureName",
            Type = "Structure",
            Fields = {
                { Name = "fieldName", Type = "string", Nilable = false },
            },
        },
    },
}
```

## Reference When to Read Specific API Files

When working on:
- **Chat systems**: Read `ChatInfoDocumentation.lua`, `ChatConstantsDocumentation.lua`
- **Items/Inventory**: Read `BagConstantsDocumentation.lua`, `ItemDocumentation.lua`, `ItemSocketInfoDocumentation.lua`
- **Combat/Spells**: Read `ActionBarDocumentation.lua`, `SpellDocumentation.lua`, `CombatLogDocumentation.lua`
- **UI Frames**: Read `UIObjectDocumentation.lua`, `UIWidgetDocumentation.lua`
- **Maps/Coords**: Read `MapCanvasDocumentation.lua`, `AreaPoiInfoDocumentation.lua`
- **Quests**: Read `QuestLogDocumentation.lua`, `QuestInfoDocumentation.lua`
- **Encounters**: Read `EncounterTimelineDocumentation.lua`, `InstanceEncounterDocumentation.lua`
- **Housing**: Read `HousingDocumentation.lua`

<!-- CLAUDE_SKIP_START -->
## Next Steps

For event handling, see `02_Event_System.md`
For UI framework, see `03_UI_Framework.md`
For housing system details, see `11_Housing_System_Guide.md`

---

**Version**: 2.0
**Based on**: WoW 12.0.0 (Midnight)
**API Files**: 513+ documentation files
**Events**: 1,700+ total events

<!-- CLAUDE_SKIP_END -->
