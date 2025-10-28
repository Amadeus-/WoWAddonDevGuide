# WoW API Reference Guide

## Table of Contents
1. [Overview](#overview)
2. [API Structure](#api-structure)
3. [API Categories](#api-categories)
4. [Common API Patterns](#common-api-patterns)
5. [API Usage Examples](#api-usage-examples)
6. [Finding API Documentation](#finding-api-documentation)

---

## Overview
This document provides comprehensive information about the World of Warcraft API structure and usage patterns.

**Statistics**:
- **Total API Documentation Files**: 513
- **Total Events**: 1,645
- **API Systems**: 192

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

Older APIs exist in the global namespace. Many are still actively used.

**Examples**:
```lua
UnitName("player")
GetItemInfo(itemID)
CreateFrame("Frame", "MyFrame", UIParent)
```

## Common API Categories

### 1. Unit APIs
Functions for querying information about units (players, NPCs, pets, etc.)

**Key Functions**:
```lua
UnitName("unit")                    -- Get unit's name
UnitHealth("unit")                  -- Get current health
UnitHealthMax("unit")               -- Get maximum health
UnitPower("unit", powerType)        -- Get current power (mana, energy, etc.)
UnitClass("unit")                   -- Get class name and class file
UnitRace("unit")                    -- Get race name and race file
UnitLevel("unit")                   -- Get level
UnitExists("unit")                  -- Check if unit exists
UnitIsPlayer("unit")                -- Check if unit is a player
UnitIsDead("unit")                  -- Check if unit is dead
UnitAffectingCombat("unit")         -- Check if in combat
UnitBuff("unit", index or "name")   -- Get buff information
UnitDebuff("unit", index or "name") -- Get debuff information
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
```

**Global Functions**:
```lua
GetSpellInfo(spellID or "spellName" or spellIndex, "bookType")
    -- Returns: name, rank, icon, castTime, minRange, maxRange, spellID, originalIcon

GetSpellCooldown(spellID or "spellName")
    -- Returns: start, duration, enabled, modRate

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
```

**Global Functions**:
```lua
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

## API Documentation Files Location

All 513 API documentation files are located at:
```
D:\Games\World of Warcraft\_retail_\Interface\+wow-ui-source+ (11.2.7)\Interface\AddOns\Blizzard_APIDocumentationGenerated\
```

**Index Files Created**:
- `C:\Dev\WoW_Addon_Dev_Knowledge_Base\api_extracted\00_API_INDEX.md` - Complete list of all 513 API files
- `C:\Dev\WoW_Addon_Dev_Knowledge_Base\api_extracted\00_API_BY_CATEGORY.md` - APIs organized by category
- `C:\Dev\WoW_Addon_Dev_Knowledge_Base\api_extracted\00_API_STATISTICS.md` - Detailed statistics

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

## Deprecation Notes

Many old global APIs are deprecated but still functional. Prefer C_* namespace equivalents when available.

**Example Migration**:
```lua
-- Old (deprecated but works)
GetRealmName()

-- New (preferred)
C_RealmInfo.GetRealmName()
```

## Reference When to Read Specific API Files

When working on:
- **Chat systems**: Read `ChatInfoDocumentation.lua`, `ChatConstantsDocumentation.lua`
- **Items/Inventory**: Read `BagConstantsDocumentation.lua`, `ItemDocumentation.lua`
- **Combat/Spells**: Read `ActionDocumentation.lua`, `SpellDocumentation.lua`
- **UI Frames**: Read `UIObjectDocumentation.lua`, `UIWidgetDocumentation.lua`
- **Maps/Coords**: Read `MapCanvasDocumentation.lua`, `AreaPoiInfoDocumentation.lua`
- **Quests**: Read `QuestLogDocumentation.lua`, `QuestInfoDocumentation.lua`

<!-- CLAUDE_SKIP_START -->
## Next Steps

For event handling, see `02_Event_System.md`
For UI framework, see `03_UI_Framework.md`
For complete event list, see `events_extracted/00_EVENTS_INDEX.md`

---

**Version**: 1.0
**Based on**: WoW 11.2.7 (The War Within)
**API Files**: 513 documentation files
**Events**: 1,645 total events

<!-- CLAUDE_SKIP_END -->
