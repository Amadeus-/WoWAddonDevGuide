# WoW Event System Reference

## Table of Contents
1. [Overview](#overview)
2. [Event System Basics](#event-system-basics)
3. [Event Registration Patterns](#event-registration-patterns)
4. [Common Events by Category](#common-events-by-category)
5. [Event Best Practices](#event-best-practices)
6. [Event Reference](#event-reference)

---

## Overview
The WoW event system is the backbone of addon development. Events notify addons when something happens in the game world.

**Statistics**:
- **Total Events**: 1,645 events
- **Event Systems**: 192 different systems
- **Event Index**: `C:\Dev\WoW_Addon_Dev_Knowledge_Base\events_extracted\00_EVENTS_INDEX.md`

## Event System Basics

### How Events Work

1. Something happens in game (player logs in, item acquired, combat starts, etc.)
2. WoW fires an event with that event's name
3. All frames registered for that event have their OnEvent handler called
4. The handler receives the event name and any payload data

### Event Registration

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

### Combat Events

- `PLAYER_REGEN_DISABLED` - Entered combat
- `PLAYER_REGEN_ENABLED` - Left combat
- `COMBAT_LOG_EVENT_UNFILTERED` - Combat log event (use `CombatLogGetCurrentEventInfo()`)
- `PLAYER_DAMAGE_DONE_MODS` - Damage modifiers changed
  - Payload: `unit`

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
- `PLAYERREAGENTBANKSLOTS_CHANGED` - Reagent bank changed
  - Payload: `slotID`

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
-- After you're done with an event
frame:UnregisterEvent("SOME_EVENT")

-- Unregister all events
frame:UnregisterAllEvents()
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

## Event Testing and Debugging

### Monitor Events
```lua
-- Print all events
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

## Complete Event Reference

**Location**: `C:\Dev\WoW_Addon_Dev_Knowledge_Base\events_extracted\00_EVENTS_INDEX.md`

This file contains:
- All 1,645 events alphabetically
- Events organized by 192 systems
- Event source files

<!-- CLAUDE_SKIP_START -->
## Next Steps

For API functions to use with events, see `01_API_Reference.md`
For UI frame setup, see `03_UI_Framework.md`
For complete event list with payloads, read the API documentation files

---

**Version**: 1.0
**Based on**: WoW 11.2.7 (The War Within)
**Total Events**: 1,645 events across 192 systems

<!-- CLAUDE_SKIP_END -->
