# WoW Event System Reference

## Table of Contents
1. [Overview](#overview)
2. [Event System Basics](#event-system-basics)
3. [Event Registration Patterns](#event-registration-patterns)
4. [Callback-Based Event Registration (12.0.0+)](#callback-based-event-registration-1200)
5. [Common Events by Category](#common-events-by-category)
6. [New Events (11.x - 12.0.0)](#new-events-11x---1200)
7. [Removed and Changed Events](#removed-and-changed-events)
8. [Event Best Practices](#event-best-practices)
9. [Event Reference](#event-reference)

---

## Overview
The WoW event system is the backbone of addon development. Events notify addons when something happens in the game world.

**Statistics**:
- **Total Events**: 1,800+ events (12.0.0)
- **Event Systems**: 200+ different systems

**Version Note**: This document covers WoW 12.0.0 (Midnight expansion). Events may differ in Classic versions.

## Event System Basics

### How Events Work

1. Something happens in game (player logs in, item acquired, combat starts, etc.)
2. WoW fires an event with that event's name
3. All frames registered for that event have their OnEvent handler called
4. The handler receives the event name and any payload data

### Event Registration (Traditional Frame-Based)

**Basic Pattern**:
```lua
local frame = CreateFrame("Frame")

-- Register for events
frame:RegisterEvent("PLAYER_LOGIN")
frame:RegisterEvent("PLAYER_ENTERING_WORLD")
frame:RegisterEvent("COMBAT_LOG_EVENT_UNFILTERED")

-- Set event handler
frame:SetScript("OnEvent", function(self, event, ...)
    if event == "PLAYER_LOGIN" then
        print("Player logged in!")
    elseif event == "PLAYER_ENTERING_WORLD" then
        local isInitialLogin, isReloadingUi = ...
        if isInitialLogin then
            print("First login on this character")
        elseif isReloadingUi then
            print("UI was reloaded")
        end
    elseif event == "COMBAT_LOG_EVENT_UNFILTERED" then
        local timestamp, subevent, _, sourceGUID, sourceName = CombatLogGetCurrentEventInfo()
        -- Process combat log event
    end
end)
```

### Event Handler Patterns

#### Pattern 1: Single Function Dispatcher
```lua
local MyAddon = {}

function MyAddon:OnEvent(event, ...)
    if self[event] then
        self[event](self, ...)
    end
end

function MyAddon:PLAYER_LOGIN()
    print("Logged in!")
end

function MyAddon:BAG_UPDATE(bagID)
    print("Bag " .. bagID .. " updated")
end

local frame = CreateFrame("Frame")
frame:RegisterEvent("PLAYER_LOGIN")
frame:RegisterEvent("BAG_UPDATE")
frame:SetScript("OnEvent", function(self, event, ...)
    MyAddon:OnEvent(event, ...)
end)
```

#### Pattern 2: Event Registry Pattern
```lua
local EventRegistry = {}

function EventRegistry:RegisterEvent(event, callback)
    if not self.events then
        self.events = {}
    end
    if not self.events[event] then
        self.events[event] = {}
        self.frame:RegisterEvent(event)
    end
    table.insert(self.events[event], callback)
end

function EventRegistry:OnEvent(event, ...)
    if self.events and self.events[event] then
        for _, callback in ipairs(self.events[event]) do
            callback(...)
        end
    end
end

EventRegistry.frame = CreateFrame("Frame")
EventRegistry.frame:SetScript("OnEvent", function(self, event, ...)
    EventRegistry:OnEvent(event, ...)
end)

-- Usage
EventRegistry:RegisterEvent("PLAYER_LOGIN", function()
    print("Login callback 1")
end)

EventRegistry:RegisterEvent("PLAYER_LOGIN", function()
    print("Login callback 2")
end)
```

#### Pattern 3: Object-Oriented Event Handling
```lua
local MyFrame = CreateFrame("Frame")
MyFrame:RegisterEvent("PLAYER_LOGIN")
MyFrame:RegisterEvent("PLAYER_LOGOUT")

MyFrame.events = {}

function MyFrame.events:PLAYER_LOGIN()
    print("Logged in!")
end

function MyFrame.events:PLAYER_LOGOUT()
    print("Logging out!")
end

MyFrame:SetScript("OnEvent", function(self, event, ...)
    if self.events[event] then
        self.events[event](self, ...)
    end
end)
```

## Callback-Based Event Registration (12.0.0+)

**New in 12.0.0**: WoW now provides a native callback-based event registration system that does not require creating frames. This is the recommended approach for new addons.

### Basic Callback Registration

```lua
-- Register a callback for an event
local function OnPlayerLogin(...)
    print("Player logged in!")
end

RegisterEventCallback("PLAYER_LOGIN", OnPlayerLogin)

-- Unregister when no longer needed
UnregisterEventCallback("PLAYER_LOGIN", OnPlayerLogin)
```

### Unit Event Callbacks

For unit-specific events, use the unit event callbacks to filter by unit:

```lua
-- Register for a specific unit's events
local function OnPlayerHealthChanged(unit, ...)
    local health = UnitHealth(unit)
    local maxHealth = UnitHealthMax(unit)
    print(format("%s health: %d/%d", unit, health, maxHealth))
end

-- Only fires for "player" unit
RegisterUnitEventCallback("player", "UNIT_HEALTH", OnPlayerHealthChanged)

-- Unregister when done
UnregisterUnitEventCallback("player", "UNIT_HEALTH", OnPlayerHealthChanged)
```

### Multiple Callbacks

You can register multiple callbacks for the same event:

```lua
local function Callback1(...)
    print("Callback 1 fired")
end

local function Callback2(...)
    print("Callback 2 fired")
end

RegisterEventCallback("PLAYER_ENTERING_WORLD", Callback1)
RegisterEventCallback("PLAYER_ENTERING_WORLD", Callback2)
-- Both callbacks will fire when the event occurs
```

### Callback vs Frame Pattern Comparison

| Feature | Frame-Based | Callback-Based (12.0.0+) |
|---------|-------------|--------------------------|
| Requires frame creation | Yes | No |
| Memory overhead | Higher | Lower |
| Unregistration | Frame method | Function reference |
| Multiple handlers | Manual dispatch | Native support |
| Unit filtering | RegisterUnitEvent | RegisterUnitEventCallback |
| Backwards compatible | All versions | 12.0.0+ only |

### Hybrid Approach for Compatibility

If your addon needs to support both pre-12.0 and 12.0+ clients:

```lua
local MyAddon = {}

-- Check if callback system exists
if RegisterEventCallback then
    -- Use new callback system
    local function OnLogin(...)
        MyAddon:OnPlayerLogin(...)
    end
    RegisterEventCallback("PLAYER_LOGIN", OnLogin)
else
    -- Fall back to frame-based
    local frame = CreateFrame("Frame")
    frame:RegisterEvent("PLAYER_LOGIN")
    frame:SetScript("OnEvent", function(self, event, ...)
        if event == "PLAYER_LOGIN" then
            MyAddon:OnPlayerLogin(...)
        end
    end)
end

function MyAddon:OnPlayerLogin(...)
    print("Logged in!")
end
```

## Common Events by Category

### Player Events

**Login/Logout**:
- `PLAYER_LOGIN` - Fired when player logs in (once per session)
- `PLAYER_ENTERING_WORLD` - Fired when entering world (login, zoning, reload)
  - Payload: `isInitialLogin, isReloadingUi`
- `PLAYER_LEAVING_WORLD` - Fired when leaving world
- `PLAYER_LOGOUT` - Fired when logging out

**Player State**:
- `PLAYER_ALIVE` - Player is alive
- `PLAYER_DEAD` - Player died
- `PLAYER_UNGHOST` - Player released from death
- `PLAYER_LEVEL_UP` - Player leveled up
  - Payload: `newLevel, healthGained, powerGained, numNewTalents, numNewPvpTalentSlots, strengthGained, agilityGained, staminaGained, intellectGained`
- `PLAYER_MONEY` - Player's money changed
- `PLAYER_XP_UPDATE` - XP changed
- `PLAYER_REGEN_DISABLED` - Entered combat
- `PLAYER_REGEN_ENABLED` - Left combat
- `PLAYER_FLAGS_CHANGED` - Player flags changed (AFK, DND, etc.)
  - Payload: `unit`

### Unit Events

- `UNIT_HEALTH` - Unit health changed
  - Payload: `unitTarget`
- `UNIT_POWER_UPDATE` - Unit power changed (mana, energy, etc.)
  - Payload: `unitTarget, powerType`
- `UNIT_AURA` - Unit buffs/debuffs changed
  - Payload: `unitTarget, updateInfo`
- `UNIT_TARGET` - Unit's target changed
  - Payload: `unitTarget`
- `UNIT_SPELLCAST_START` - Unit started casting
  - Payload: `unitTarget, castGUID, spellID`
- `UNIT_SPELLCAST_SUCCEEDED` - Unit finished casting
  - Payload: `unitTarget, castGUID, spellID`
- `UNIT_SPELLCAST_FAILED` - Cast failed
  - Payload: `unitTarget, castGUID, spellID`
- `UNIT_SPELLCAST_INTERRUPTED` - Cast interrupted
  - Payload: `unitTarget, castGUID, spellID`
- `UNIT_DIED` - Unit died (12.0.0+)
  - Payload: `unitTarget`
- `UNIT_LOOT` - Unit looted (12.0.0+)
  - Payload: `unitTarget`
- `UNIT_SPELL_DIMINISH_CATEGORY_STATE_UPDATED` - Diminishing returns state changed (12.0.0+)
  - Payload: `unitTarget, category, state`

### Combat Events

- `PLAYER_REGEN_DISABLED` - Entered combat
- `PLAYER_REGEN_ENABLED` - Left combat
- `COMBAT_LOG_EVENT_UNFILTERED` - Combat log event (use `CombatLogGetCurrentEventInfo()`)
- `PLAYER_DAMAGE_DONE_MODS` - Damage modifiers changed
  - Payload: `unit`
- `PARTY_KILL` - Party killed a unit (12.0.0+)
  - Payload: `unitTarget`
- `PLAYER_TARGET_DIED` - Player's target died (12.0.0+)
- `CHAT_MSG_ENCOUNTER_EVENT` - Encounter event message (12.0.0+)
  - Payload: `text, ...`

**Combat Log Processing**:
```lua
frame:RegisterEvent("COMBAT_LOG_EVENT_UNFILTERED")
frame:SetScript("OnEvent", function(self, event)
    if event == "COMBAT_LOG_EVENT_UNFILTERED" then
        local timestamp, subevent, _, sourceGUID, sourceName, sourceFlags, sourceRaidFlags,
              destGUID, destName, destFlags, destRaidFlags = CombatLogGetCurrentEventInfo()

        if subevent == "SPELL_DAMAGE" then
            local spellId, spellName, spellSchool, amount, overkill, school, resisted,
                  blocked, absorbed, critical, glancing, crushing = select(12, CombatLogGetCurrentEventInfo())
            -- Process damage
        elseif subevent == "SPELL_HEAL" then
            local spellId, spellName, spellSchool, amount, overhealing, absorbed,
                  critical = select(12, CombatLogGetCurrentEventInfo())
            -- Process healing
        end
    end
end)
```

### Inventory/Bag Events

- `BAG_UPDATE` - Bag contents changed
  - Payload: `bagID`
- `BAG_UPDATE_DELAYED` - Bag update finished (throttled version of BAG_UPDATE)
- `ITEM_LOCKED` - Item locked (being moved)
  - Payload: `bagID, slotID`
- `ITEM_UNLOCKED` - Item unlocked
  - Payload: `bagID, slotID`
- `PLAYERBANKSLOTS_CHANGED` - Bank slot changed
  - Payload: `slotID`

**Note**: `PLAYERREAGENTBANKSLOTS_CHANGED` was removed in 11.2.0 when the Reagent Bank was consolidated into the main bank.

### Quest Events

- `QUEST_ACCEPTED` - Quest accepted
  - Payload: `questID`
- `QUEST_REMOVED` - Quest removed
  - Payload: `questID`
- `QUEST_TURNED_IN` - Quest turned in
  - Payload: `questID, xpReward, moneyReward`
- `QUEST_LOG_UPDATE` - Quest log changed
- `QUEST_WATCH_UPDATE` - Tracked quest changed
  - Payload: `questID`
- `QUEST_POI_UPDATE` - Quest POI updated
- `QUEST_AUTOCOMPLETE` - Quest auto-completed
  - Payload: `questID`

### Chat Events

- `CHAT_MSG_SAY` - Say message
  - Payload: `text, playerName, languageName, channelName, playerName2, specialFlags, zoneChannelID, channelIndex, channelBaseName, languageID, lineID, guid, bnSenderID, isMobile, isSubtitle, hideSenderInLetterbox, supressRaidIcons`
- `CHAT_MSG_YELL` - Yell message (same payload as SAY)
- `CHAT_MSG_WHISPER` - Whisper received (same payload)
- `CHAT_MSG_PARTY` - Party message (same payload)
- `CHAT_MSG_RAID` - Raid message (same payload)
- `CHAT_MSG_GUILD` - Guild message (same payload)
- `CHAT_MSG_OFFICER` - Officer message (same payload)
- `CHAT_MSG_CHANNEL` - Custom channel message (same payload)
- `CHAT_MSG_SYSTEM` - System message
  - Payload: `text`
- `CHAT_MSG_EMOTE` - Emote message
- `CHAT_MSG_ENCOUNTER_EVENT` - Encounter/boss event message (12.0.0+)
  - Payload: `text, ...`

### UI Events

- `ADDON_LOADED` - Addon finished loading
  - Payload: `addOnName`
- `VARIABLES_LOADED` - Saved variables loaded
- `PLAYER_LOGIN` - Player logged in (UI is ready)
- `UPDATE_MOUSEOVER_UNIT` - Mouse is over new unit
- `UI_ERROR_MESSAGE` - UI error occurred
  - Payload: `messageType, message`
- `UI_INFO_MESSAGE` - UI info message
  - Payload: `messageType, message`
- `SETTINGS_LOADED` - Settings system loaded (11.0.0+)
- `SETTINGS_PANEL_OPEN` - Settings panel opened (12.0.0+)
- `ADDON_RESTRICTION_STATE_CHANGED` - Addon restriction state changed (12.0.0+)
  - Payload: `addonName, state`

### Loot Events

- `LOOT_READY` - Loot window ready
  - Payload: `autoloot`
- `LOOT_OPENED` - Loot window opened
- `LOOT_CLOSED` - Loot window closed
- `LOOT_SLOT_CLEARED` - Loot slot cleared
  - Payload: `slot`
- `LOOT_SLOT_CHANGED` - Loot slot changed
  - Payload: `slot`

### Group/Raid Events

- `GROUP_FORMED` - Group formed
- `GROUP_JOINED` - Joined group
- `GROUP_LEFT` - Left group
- `GROUP_ROSTER_UPDATE` - Group roster changed
- `RAID_ROSTER_UPDATE` - Raid roster changed
- `PARTY_LEADER_CHANGED` - Party leader changed
- `PARTY_MEMBER_ENABLE` - Party member came online
- `PARTY_MEMBER_DISABLE` - Party member went offline

### Auction House Events

- `AUCTION_HOUSE_SHOW` - AH opened
- `AUCTION_HOUSE_CLOSED` - AH closed
- `AUCTION_OWNED_LIST_UPDATE` - Owned auctions updated
- `AUCTION_BIDDER_LIST_UPDATE` - Bid auctions updated
- `AUCTION_MULTISELL_START` - Multi-sell started
- `AUCTION_MULTISELL_UPDATE` - Multi-sell progress
  - Payload: `numPosted, numTotal`
- `AUCTION_MULTISELL_FAILURE` - Multi-sell failed

### Profession Events

- `TRADE_SKILL_SHOW` - Profession window opened
- `TRADE_SKILL_CLOSE` - Profession window closed
- `TRADE_SKILL_LIST_UPDATE` - Recipe list updated
- `TRADE_SKILL_DATA_SOURCE_CHANGED` - Data source changed
- `CRAFT_SHOW` - Crafting UI shown
- `CRAFT_CLOSE` - Crafting UI closed

### Map/Zone Events

- `ZONE_CHANGED` - Zone changed
- `ZONE_CHANGED_INDOORS` - Indoor/outdoor changed
- `ZONE_CHANGED_NEW_AREA` - New area entered
- `PLAYER_STARTED_MOVING` - Player started moving
- `PLAYER_STOPPED_MOVING` - Player stopped moving
- `MIRROR_TIMER_START` - Breath/fatigue timer started
  - Payload: `timerName, value, maxValue, scale, paused, label`

## New Events (11.x - 12.0.0)

### Encounter/Dungeon Events (12.0.0)

The new Encounter Timeline system provides detailed boss fight information:

- `ENCOUNTER_TIMELINE_EVENT_ADDED` - Timeline event added
  - Payload: `encounterID, eventID, eventData`
- `ENCOUNTER_TIMELINE_EVENT_REMOVED` - Timeline event removed
  - Payload: `encounterID, eventID`
- `ENCOUNTER_TIMELINE_EVENT_STATE_CHANGED` - Event state changed
  - Payload: `encounterID, eventID, newState`
- `ENCOUNTER_TIMELINE_LAYOUT_UPDATED` - Timeline layout refreshed
  - Payload: `encounterID`
- `ENCOUNTER_WARNING` - Boss ability warning
  - Payload: `text, warningType`
- `ENCOUNTER_STATE_CHANGED` - Encounter state changed
  - Payload: `encounterID, state`

### Damage Meter Events (12.0.0)

Built-in damage meter support:

- `DAMAGE_METER_COMBAT_SESSION_UPDATED` - Combat session data updated
  - Payload: `sessionID`
- `DAMAGE_METER_CURRENT_SESSION_UPDATED` - Current session data updated
- `DAMAGE_METER_RESET` - Damage meter reset

### Combat Log Events (Refactored 12.0.0)

Combat log system has been refactored with new events:

- `COMBAT_LOG_APPLY_FILTER_SETTINGS` - Filter settings applied
- `COMBAT_LOG_ENTRIES_CLEARED` - Log entries cleared
- `COMBAT_LOG_MESSAGE_LIMIT_CHANGED` - Message limit changed
  - Payload: `newLimit`
- `COMBAT_LOG_REFILTER_ENTRIES` - Entries being refiltered
- `COMBAT_LOG_MESSAGE` - New combat log message (alternative to CLEU)
  - Payload: `messageInfo`

### Housing Events (12.0.0)

Over 100 new events for the Housing system. Key events include:

- `HOUSE_INFO_UPDATED` - House information updated
- `HOUSING_DECOR_PLACE_SUCCESS` - Decor item placed successfully
  - Payload: `decorID, position`
- `HOUSING_DECOR_REMOVE_SUCCESS` - Decor item removed
  - Payload: `decorID`
- `HOUSING_DECOR_INVENTORY_UPDATED` - Decor inventory changed
- `HOUSING_MODE_ENTERED` - Entered housing edit mode
- `HOUSING_MODE_EXITED` - Exited housing edit mode
- `NEIGHBORHOOD_INFO_UPDATED` - Neighborhood info changed
- `HOUSING_VISITOR_ENTERED` - Visitor entered your house
  - Payload: `visitorName, visitorGUID`
- `HOUSING_VISITOR_LEFT` - Visitor left your house
  - Payload: `visitorName, visitorGUID`

See `12_Housing_System_Guide.md` for the complete Housing event reference.

### Transmog Events (11.x - 12.0.0)

Enhanced transmogrification events:

- `TRANSMOG_CUSTOM_SETS_CHANGED` - Custom transmog sets changed
- `TRANSMOG_DISPLAYED_OUTFIT_CHANGED` - Displayed outfit changed
  - Payload: `outfitID`
- `TRANSMOG_OUTFITS_CHANGED` - Outfits list changed
- `VIEWED_TRANSMOG_OUTFIT_STARTED` - Started viewing transmog outfit
  - Payload: `outfitID`
- `VIEWED_TRANSMOG_OUTFIT_ENDED` - Stopped viewing transmog outfit
- `VIEWED_TRANSMOG_SOURCE_STARTED` - Started viewing transmog source
  - Payload: `sourceID`
- `VIEWED_TRANSMOG_SOURCE_ENDED` - Stopped viewing transmog source

### Event Scheduler (11.1.0+)

The `C_EventScheduler` API provides scheduled event support:

- `EVENT_SCHEDULER_UPDATE` - Scheduled event tick
  - Payload: `schedulerID, elapsed`

```lua
-- Using the event scheduler
local schedulerID = C_EventScheduler.CreateScheduler(interval, callback)
C_EventScheduler.DestroyScheduler(schedulerID)

-- Listen for updates
RegisterEventCallback("EVENT_SCHEDULER_UPDATE", function(schedulerID, elapsed)
    -- Handle scheduled tick
end)
```

### Settings Events (11.0.0+)

- `SETTINGS_LOADED` - Settings system initialized
- `SETTINGS_PANEL_OPEN` - Settings panel opened (12.0.0+)
- `SETTINGS_PANEL_CLOSE` - Settings panel closed (12.0.0+)

### Addon Management Events (12.0.0)

- `ADDON_RESTRICTION_STATE_CHANGED` - Addon restriction state changed
  - Payload: `addonName, restrictionState`
  - States: `NONE`, `RESTRICTED`, `BLOCKED`

### Spell Diminishing Returns (12.0.0)

- `UNIT_SPELL_DIMINISH_CATEGORY_STATE_UPDATED` - DR state updated
  - Payload: `unitTarget, category, state, timeRemaining`

## Removed and Changed Events

### Removed in 12.0.0

- `LEARNED_SPELL_IN_TAB` - Removed; use `LEARNED_SPELL` instead
- `RETURNING_PLAYER_PROMPT` - Behavior changed, event signature modified

### Removed in 11.2.0

With the consolidation of Void Storage and Reagent Bank into the unified Bank system:

**Void Storage Events (Removed)**:
- `VOID_STORAGE_OPEN`
- `VOID_STORAGE_CLOSE`
- `VOID_STORAGE_UPDATE`
- `VOID_STORAGE_CONTENTS_UPDATE`
- `VOID_STORAGE_DEPOSIT_UPDATE`
- `VOID_TRANSFER_DONE`

**Reagent Bank Events (Removed)**:
- `PLAYERREAGENTBANKSLOTS_CHANGED`
- `REAGENTBANK_PURCHASED`
- `REAGENTBANK_UPDATE`

### Changed Event Payloads

**LUA_WARNING** (12.0.0):
```lua
-- Old (pre-12.0.0):
-- Payload: warnType, message

-- New (12.0.0+):
-- Payload: message
-- warnType parameter removed

-- Compatibility wrapper:
frame:SetScript("OnEvent", function(self, event, arg1, arg2)
    local message
    if arg2 then
        -- Pre-12.0.0 format
        message = arg2
    else
        -- 12.0.0+ format
        message = arg1
    end
end)
```

**VOICE_CHAT_TTS_* Events** (12.0.0):
Several voice chat text-to-speech events have had parameters removed:
- `VOICE_CHAT_TTS_PLAYBACK_STARTED` - Removed `voiceID` parameter
- `VOICE_CHAT_TTS_PLAYBACK_FINISHED` - Removed `voiceID` parameter

## Event Timing and Order

**Login Sequence**:
1. `ADDON_LOADED` - Fires once for each addon as it loads
2. `VARIABLES_LOADED` - SavedVariables are available
3. `PLAYER_LOGIN` - Player is fully loaded, UI is ready
4. `PLAYER_ENTERING_WORLD` - Player in world (also fires on zone/reload)

**Important Notes**:
- Saved variables are NOT available in `ADDON_LOADED`
- Saved variables ARE available in `VARIABLES_LOADED` and later
- `PLAYER_LOGIN` fires exactly once per session
- `PLAYER_ENTERING_WORLD` fires on login, zone, and `/reload`

## Event Performance Considerations

### 1. Unregister Unused Events
```lua
-- After you're done with an event (frame-based)
frame:UnregisterEvent("SOME_EVENT")

-- Unregister all events
frame:UnregisterAllEvents()

-- Callback-based (12.0.0+)
UnregisterEventCallback("SOME_EVENT", myCallback)
```

### 2. Throttle High-Frequency Events
```lua
local throttle = 0
frame:SetScript("OnEvent", function(self, event, ...)
    if event == "UNIT_HEALTH" then
        -- This fires VERY frequently, throttle it
        local now = GetTime()
        if now - throttle < 0.1 then  -- Max 10 times per second
            return
        end
        throttle = now
        -- Process event
    end
end)
```

### 3. Use More Specific Events
```lua
-- Bad: Fires too often
frame:RegisterEvent("BAG_UPDATE")

-- Better: Only fires when done updating
frame:RegisterEvent("BAG_UPDATE_DELAYED")
```

### 4. Use Unit Event Callbacks for Filtering (12.0.0+)
```lua
-- Bad: Receives all unit health changes
RegisterEventCallback("UNIT_HEALTH", function(unit, ...)
    if unit == "player" then
        -- Process
    end
end)

-- Good: Only receives player health changes
RegisterUnitEventCallback("player", "UNIT_HEALTH", function(unit, ...)
    -- Process (always "player")
end)
```

## Event Testing and Debugging

### Monitor Events
```lua
-- Print all events (use sparingly - very spammy!)
local f = CreateFrame("Frame")
f:RegisterAllEvents()
f:SetScript("OnEvent", function(self, event, ...)
    print("Event:", event, ...)
end)
```

### Use /eventtrace
```
/eventtrace
```
Opens a window showing all fired events in real-time.

### Use /fstack
```
/fstack
```
Shows frame names under mouse cursor (useful for finding what handles events).

### Event Filtering in Debug
```lua
-- Monitor specific events only
local watchEvents = {
    ["PLAYER_LOGIN"] = true,
    ["PLAYER_ENTERING_WORLD"] = true,
    ["ADDON_LOADED"] = true,
}

local f = CreateFrame("Frame")
for event in pairs(watchEvents) do
    f:RegisterEvent(event)
end
f:SetScript("OnEvent", function(self, event, ...)
    print(format("[%s] %s: %s", date("%H:%M:%S"), event, strjoin(", ", tostringall(...))))
end)
```

## Advanced Event Patterns

### Conditional Event Registration
```lua
local MyAddon = {}

function MyAddon:EnableCombatTracking()
    if not self.combatTrackingEnabled then
        self.frame:RegisterEvent("PLAYER_REGEN_DISABLED")
        self.frame:RegisterEvent("PLAYER_REGEN_ENABLED")
        self.combatTrackingEnabled = true
    end
end

function MyAddon:DisableCombatTracking()
    if self.combatTrackingEnabled then
        self.frame:UnregisterEvent("PLAYER_REGEN_DISABLED")
        self.frame:UnregisterEvent("PLAYER_REGEN_ENABLED")
        self.combatTrackingEnabled = false
    end
end
```

### Event Queuing (for Load Order)
```lua
local eventQueue = {}
local isReady = false

local frame = CreateFrame("Frame")
frame:RegisterEvent("PLAYER_LOGIN")
frame:RegisterEvent("SOME_OTHER_EVENT")

frame:SetScript("OnEvent", function(self, event, ...)
    if not isReady and event ~= "PLAYER_LOGIN" then
        -- Queue events until ready
        table.insert(eventQueue, {event = event, args = {...}})
        return
    end

    if event == "PLAYER_LOGIN" then
        isReady = true
        -- Process queued events
        for _, queued in ipairs(eventQueue) do
            -- Process queued.event with queued.args
        end
        eventQueue = nil
    end
end)
```

### Version-Safe Event Registration
```lua
-- Handle events that may not exist in all versions
local function SafeRegisterEvent(frame, event)
    -- Try to register; will silently fail if event doesn't exist
    pcall(function()
        frame:RegisterEvent(event)
    end)
end

-- Or check API availability
if C_EventUtils and C_EventUtils.IsEventValid then
    if C_EventUtils.IsEventValid("NEW_EVENT_NAME") then
        frame:RegisterEvent("NEW_EVENT_NAME")
    end
end
```

## Complete Event Reference

For a complete event reference, consult the Blizzard API documentation files located in:
```
Interface\AddOns\Blizzard_APIDocumentationGenerated\
```

These files contain all 1,800+ events organized by system, including event payloads and documentation.

<!-- CLAUDE_SKIP_START -->
## Next Steps

For API functions to use with events, see `01_API_Reference.md`
For UI frame setup, see `03_UI_Framework.md`
For Housing system events, see `12_Housing_System_Guide.md`
For complete event list with payloads, read the API documentation files

---

**Version**: 2.0
**Based on**: WoW 12.0.0 (Midnight)
**Total Events**: 1,800+ events across 200+ systems

<!-- CLAUDE_SKIP_END -->
