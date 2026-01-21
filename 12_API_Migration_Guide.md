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

---

## Major API Changes by Expansion

### Midnight (12.0.0+) - "The Addon Apocalypse"

Patch 12.0.0 is the largest API change since the original addon system was introduced. With 432 new APIs added and 140+ removed, this expansion fundamentally changes how addons interact with combat data.

#### SECRET VALUES SYSTEM (CRITICAL!)

The most significant change in 12.0.0 is the introduction of the **Secret Values** system. This system restricts addon automation during combat by making certain API returns unusable by tainted (addon) code.

**What Are Secret Values?**
```lua
-- During combat, some APIs return "secret values"
-- These values LOOK normal but CANNOT be used in tainted code operations

-- Example: Getting action bar info during combat
local actionType, spellID = GetActionInfo(slot)  -- In combat, spellID may be "secret"

-- Attempting to use a secret value in addon code:
if spellID == 12345 then  -- ERROR: Cannot compare secret values!
    -- This code may not execute as expected
end
```

**Built-in Functions for Secret Values:**
```lua
-- Check if a value is secret
if issecretvalue(someValue) then
    print("Value is secret, cannot use directly")
end

-- Check if you can access a value
if canaccessvalue(someValue) then
    -- Safe to use
else
    -- Value is secret or restricted
end

-- Wrap a value to make it secret (for secure code)
local secretVal = secretwrap(regularValue)

-- Remove secret values from a table (returns clean copy)
local cleanTable = scrubsecretvalues(mixedTable)

-- Get underlying value (only works in secure code)
local realValue = secretunwrap(secretValue)  -- Returns nil for tainted code
```

**New Namespaces for Secret Value Handling:**
```lua
-- C_Secrets namespace
C_Secrets.IsSecretValue(value)
C_Secrets.CanAccessValue(value)
C_Secrets.GetSecretValueType(value)

-- C_RestrictedActions namespace
C_RestrictedActions.IsActionRestricted(actionType)
C_RestrictedActions.GetRestrictionReason(actionType)
C_RestrictedActions.CanPerformAction(actionType)
```

**Testing CVars for Development:**
```lua
-- Force secret restrictions for testing (development only)
SetCVar("secretCombatRestrictionsForced", 1)
SetCVar("secretEncounterRestrictionsForced", 1)
SetCVar("secretRestrictionsDebug", 1)

-- Check current restriction state
local inRestriction = C_RestrictedActions.IsInRestrictedState()
```

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

#### C_ActionBar Namespace (Replaces Global Action Bar Functions)

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
C_ActionBar.GetActionInfo(slot)
C_ActionBar.GetActionTexture(slot)
C_ActionBar.GetActionCooldown(slot)
C_ActionBar.GetActionCount(slot)
C_ActionBar.IsUsableAction(slot)
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

#### C_DamageMeter Namespace - OFFICIAL DAMAGE METER API!

This is a game-changer. Blizzard now provides official APIs for damage meter functionality:

```lua
-- Get encounter damage/healing data
C_DamageMeter.GetEncounterInfo()
C_DamageMeter.GetCombatantInfo(combatantGUID)
C_DamageMeter.GetCombatantDamage(combatantGUID)
C_DamageMeter.GetCombatantHealing(combatantGUID)
C_DamageMeter.GetCombatantDamageTaken(combatantGUID)

-- Get breakdown by ability
C_DamageMeter.GetAbilityDamage(combatantGUID, abilityID)
C_DamageMeter.GetAbilityHealing(combatantGUID, abilityID)

-- Get rankings
C_DamageMeter.GetDamageRankings()
C_DamageMeter.GetHealingRankings()
C_DamageMeter.GetDamageTakenRankings()

-- Encounter timeline
C_DamageMeter.GetEncounterTimeline()
C_DamageMeter.GetTimelineEvent(index)

-- Control
C_DamageMeter.StartTracking()
C_DamageMeter.StopTracking()
C_DamageMeter.ResetData()
C_DamageMeter.IsTracking()
```

**Why This Matters:**
- Official API means consistent data across all meters
- No more combat log parsing for basic meters
- Works correctly with secret values system
- May become required as combat log access becomes more restricted

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

#### Patch 12.0.1 (TOC 120001)

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

### Example 5: Damage Meter Addon (11.x -> 12.0.0)

**Major Change:** Consider using the new official C_DamageMeter API instead of parsing combat log.

**Before (11.x - Combat Log Parsing):**
```lua
local damageData = {}
local frame = CreateFrame("Frame")
frame:RegisterEvent("COMBAT_LOG_EVENT_UNFILTERED")
frame:SetScript("OnEvent", function()
    local _, event, _, sourceGUID, sourceName, _, _, destGUID, destName,
          _, _, spellID, spellName, _, amount = CombatLogGetCurrentEventInfo()

    if event == "SPELL_DAMAGE" or event == "SWING_DAMAGE" then
        damageData[sourceGUID] = (damageData[sourceGUID] or 0) + (amount or 0)
    end
end)
```

**After (12.0.0+ - Official API):**
```lua
-- Option 1: Use official C_DamageMeter API (recommended for basic meters)
local function UpdateDamageDisplay()
    local rankings = C_DamageMeter.GetDamageRankings()
    for i, entry in ipairs(rankings) do
        local damage = C_DamageMeter.GetCombatantDamage(entry.combatantGUID)
        UpdateMeterBar(i, entry.name, damage)
    end
end

-- Start tracking
C_DamageMeter.StartTracking()

-- Register for encounter events
local frame = CreateFrame("Frame")
frame:RegisterEvent("ENCOUNTER_START")
frame:RegisterEvent("ENCOUNTER_END")
frame:SetScript("OnEvent", function(self, event)
    if event == "ENCOUNTER_START" then
        C_DamageMeter.ResetData()
    elseif event == "ENCOUNTER_END" then
        UpdateDamageDisplay()
    end
end)

-- Option 2: Still use combat log (for advanced meters needing full data)
local frame = CreateFrame("Frame")
frame:RegisterEvent("COMBAT_LOG_EVENT_UNFILTERED")
frame:SetScript("OnEvent", function()
    -- Use new namespace
    local _, event, _, sourceGUID, sourceName, _, _, destGUID, destName,
          _, _, spellID, spellName, _, amount = C_CombatLog.GetCurrentEventInfo()
    -- Process as before
end)
```

**Benefits of Official API:**
- No combat log parsing overhead
- Consistent data across all meters
- Works correctly with secret values
- May be required as Blizzard restricts combat log access further

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
| `GetActionInfo(slot)` | `C_ActionBar.GetActionInfo(slot)` | 12.0.0 | Secret values in combat |
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
- [10_Advanced_Techniques.md](10_Advanced_Techniques.md) - Cross-client compatibility
- [04_Addon_Structure.md](04_Addon_Structure.md) - TOC file versioning
- [05_Patterns_And_Best_Practices.md](05_Patterns_And_Best_Practices.md) - Code patterns

**Next Steps:**
1. Bookmark Warcraft Wiki API changes pages
2. Create compatibility wrapper template
3. Set up automated testing for new patches
4. Join WoW Addon Discord for early warnings

---

**Version:** 2.0 - Updated for WoW 12.0.0 (Midnight)
**Last Updated:** 2026-01-20
**Coverage:** API changes from WoW 9.0 through 12.0.0

<!-- CLAUDE_SKIP_END -->
