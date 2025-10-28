# WoW Addon Data Persistence - Saved Variables Guide

## Table of Contents
1. [Saved Variables Overview](#saved-variables-overview)
2. [TOC File Configuration](#toc-file-configuration)
3. [Variable Types and Scopes](#variable-types-and-scopes)
4. [Best Practices for Data Structure](#best-practices-for-data-structure)
5. [Initialization Patterns](#initialization-patterns)
6. [Defaults and Migrations](#defaults-and-migrations)
7. [Performance Considerations](#performance-considerations)
8. [Security and Validation](#security-and-validation)
9. [Debugging Saved Variables](#debugging-saved-variables)

---

## Saved Variables Overview

WoW addons persist data between sessions using **saved variables**. These are Lua variables declared in the TOC file that WoW automatically saves/loads from disk.

**Key Concepts:**
- Variables are saved when logging out or `/reload`
- Loaded before `ADDON_LOADED` event fires
- Stored in `WTF/Account/[Account]/SavedVariables/` or character-specific folders
- Must be global variables (not local)
- Can be any Lua-serializable type (tables, numbers, strings, booleans)

---

## TOC File Configuration

### SavedVariables (Account-Wide)

Shared across all characters on the account:

```
## Interface: 110207
## Title: My Addon
## SavedVariables: MyAddonDB, MyAddonConfig
```

**File Location:**
```
WTF/Account/[AccountName]/SavedVariables/MyAddon.lua
```

### SavedVariablesPerCharacter (Character-Specific)

Unique to each character:

```
## Interface: 110207
## Title: My Addon
## SavedVariablesPerCharacter: MyAddonCharDB, MyAddonProfiles
```

**File Location:**
```
WTF/Account/[AccountName]/[Server]/[Character]/SavedVariables/MyAddon.lua
```

### SavedVariablesMachine (Per-Computer)

Stored per computer, not synced:

**Source:** `Blizzard_AddOnList/Blizzard_AddOnList.toc`
```
## SavedVariablesMachine: g_addonCategoriesCollapsed
```

**File Location:**
```
WTF/Config-cache.wtf
```

**Use Cases:**
- UI layout preferences
- Window positions/sizes
- Local cache data

### Multiple Variables

You can declare multiple saved variables (comma-separated):

```
## SavedVariables: MyAddonDB, MyAddonSettings, MyAddonCache
## SavedVariablesPerCharacter: MyAddonCharData, MyAddonProfiles
```

---

## Variable Types and Scopes

### Global Scope Requirement

**IMPORTANT:** Saved variables MUST be global (not local).

```lua
-- CORRECT: Global variable
MyAddonDB = MyAddonDB or {};

-- WRONG: Local variable (will not be saved!)
local MyAddonDB = MyAddonDB or {};
```

### Supported Data Types

```lua
-- All of these can be saved:
MyAddonDB = {
    -- Primitives
    stringValue = "hello",
    numberValue = 123.45,
    booleanValue = true,
    nilValue = nil,  -- Saved as nil

    -- Tables (nested)
    nested = {
        deep = {
            value = "works"
        }
    },

    -- Arrays
    items = {"item1", "item2", "item3"},

    -- Mixed tables
    mixed = {
        [1] = "array element",
        ["key"] = "hash element",
    },
};

-- NOT serializable:
-- Functions, metatables, userdata, coroutines
-- These will cause errors or be ignored
```

---

## Best Practices for Data Structure

### Database Pattern

**Namespace your data with version control:**

```lua
MyAddonDB = {
    -- Version for migration
    version = 1,

    -- Global settings
    global = {
        debugMode = false,
        enableSound = true,
    },

    -- Per-character data
    characters = {
        ["RealmName-CharacterName"] = {
            level = 70,
            lastSeen = time(),
            gold = 1000000,
        },
    },

    -- Per-profile data
    profiles = {
        ["Default"] = {
            uiScale = 1.0,
            showMinimap = true,
        },
    },
};
```

### Character Key Pattern

**Source:** Common pattern in Blizzard addons

```lua
-- Generate unique character key
local function GetCharacterKey()
    local name = UnitName("player");
    local realm = GetRealmName();
    return format("%s-%s", realm, name);
end

-- Initialize character data
local function InitializeCharacterData()
    local key = GetCharacterKey();

    if not MyAddonDB.characters[key] then
        MyAddonDB.characters[key] = {
            created = time(),
            gold = 0,
            achievements = {},
        };
    end

    return MyAddonDB.characters[key];
end
```

### Ace3-Style Database

**Popular community pattern:**

```lua
MyAddonDB = {
    profileKeys = {
        ["Realm-Character1"] = "Default",
        ["Realm-Character2"] = "Custom",
    },
    profiles = {
        ["Default"] = {
            -- Profile settings
        },
        ["Custom"] = {
            -- Custom profile settings
        },
    },
    global = {
        -- Cross-character data
    },
};
```

---

## Initialization Patterns

### Safe Initialization

**Always check if variable exists before assigning:**

```lua
-- Initialize main DB
MyAddonDB = MyAddonDB or {};

-- Initialize with defaults
MyAddonDB.version = MyAddonDB.version or 1;
MyAddonDB.settings = MyAddonDB.settings or {};

-- Don't overwrite existing data!
if not MyAddonDB.characters then
    MyAddonDB.characters = {};
end
```

### Deep Merge Pattern

```lua
local function DeepMerge(target, source)
    for key, value in pairs(source) do
        if type(value) == "table" then
            if type(target[key]) ~= "table" then
                target[key] = {};
            end
            DeepMerge(target[key], value);
        else
            if target[key] == nil then
                target[key] = value;
            end
        end
    end
    return target;
end

-- Default structure
local defaults = {
    version = 1,
    settings = {
        debugMode = false,
        showTooltips = true,
    },
    characters = {},
};

-- Merge defaults without overwriting existing data
DeepMerge(MyAddonDB, defaults);
```

### ADDON_LOADED Event Pattern

```lua
local frame = CreateFrame("Frame");
frame:RegisterEvent("ADDON_LOADED");
frame:SetScript("OnEvent", function(self, event, addonName)
    if addonName == "MyAddon" then
        -- Saved variables are now loaded
        MyAddonDB = MyAddonDB or {};

        -- Initialize defaults
        if not MyAddonDB.version then
            -- First time load
            MyAddonDB.version = 1;
            MyAddonDB.settings = {
                enabled = true,
            };
        end

        -- Perform any migrations
        if MyAddonDB.version < 2 then
            MyAddon:MigrateToVersion2();
        end

        -- Unregister - only need this once
        self:UnregisterEvent("ADDON_LOADED");
    end
end);
```

---

## Defaults and Migrations

### Default Values Pattern

```lua
local DEFAULTS = {
    version = 2,
    global = {
        debugMode = false,
        trackingEnabled = true,
    },
    characters = {},
    settings = {
        uiScale = 1.0,
        showMinimap = true,
        anchorPoint = "TOPRIGHT",
    },
};

local function InitializeDatabase()
    MyAddonDB = MyAddonDB or {};

    -- Apply defaults for missing keys
    for key, value in pairs(DEFAULTS) do
        if MyAddonDB[key] == nil then
            if type(value) == "table" then
                MyAddonDB[key] = CopyTable(value);
            else
                MyAddonDB[key] = value;
            end
        end
    end
end
```

### Version Migration Pattern

```lua
local CURRENT_VERSION = 3;

local function MigrateDatabase()
    local db = MyAddonDB;

    -- Version 1 -> 2
    if db.version < 2 then
        -- Rename old key
        if db.oldSettings then
            db.settings = db.oldSettings;
            db.oldSettings = nil;
        end
        db.version = 2;
    end

    -- Version 2 -> 3
    if db.version < 3 then
        -- Convert array to hash
        if db.itemList then
            local items = {};
            for _, itemID in ipairs(db.itemList) do
                items[itemID] = true;
            end
            db.items = items;
            db.itemList = nil;
        end
        db.version = 3;
    end

    db.version = CURRENT_VERSION;
end
```

### Reset to Defaults

```lua
local function ResetToDefaults()
    -- Confirm with user
    StaticPopupDialogs["MYADDON_RESET_CONFIRM"] = {
        text = "Reset all settings to defaults?",
        button1 = "Yes",
        button2 = "No",
        OnAccept = function()
            -- Wipe existing data
            wipe(MyAddonDB);

            -- Reinitialize with defaults
            for key, value in pairs(DEFAULTS) do
                if type(value) == "table" then
                    MyAddonDB[key] = CopyTable(value);
                else
                    MyAddonDB[key] = value;
                end
            end

            -- Reload UI
            ReloadUI();
        end,
        timeout = 0,
        whileDead = true,
        hideOnEscape = true,
    };

    StaticPopup_Show("MYADDON_RESET_CONFIRM");
end
```

---

## Performance Considerations

### Limit Data Size

**SavedVariables files have practical size limits (~10MB recommended)**

```lua
-- BAD: Storing huge amounts of data
for i = 1, 1000000 do
    MyAddonDB.hugeArray[i] = {
        lots = "of",
        data = "here",
    };
end

-- GOOD: Store only essential data
MyAddonDB.statistics = {
    totalKills = 1000000,
    averageDamage = 50000,
    -- Aggregate, don't store every event
};
```

### Data Pruning

```lua
-- Remove old character data
local function PruneOldCharacters()
    local cutoff = time() - (30 * 24 * 60 * 60);  -- 30 days

    for key, data in pairs(MyAddonDB.characters) do
        if data.lastSeen and data.lastSeen < cutoff then
            MyAddonDB.characters[key] = nil;
        end
    end
end
```

### Lazy Loading

```lua
-- Don't load all data at once
MyAddonDB.cache = MyAddonDB.cache or {};

local function GetCachedData(key)
    if MyAddonDB.cache[key] then
        return MyAddonDB.cache[key];
    end

    -- Load/compute data
    local data = ExpensiveOperation(key);

    -- Cache it
    MyAddonDB.cache[key] = data;

    return data;
end

-- Clear cache on logout
frame:RegisterEvent("PLAYER_LOGOUT");
frame:SetScript("OnEvent", function(self, event)
    if event == "PLAYER_LOGOUT" then
        -- Don't save cache
        MyAddonDB.cache = {};
    end
end);
```

---

## Security and Validation

### Validate Loaded Data

**NEVER trust saved variables!** Users can edit them manually.

```lua
local function ValidateDatabase()
    local db = MyAddonDB;

    -- Check version is a number
    if type(db.version) ~= "number" then
        print("MyAddon: Invalid database version, resetting...");
        MyAddonDB = {};
        return false;
    end

    -- Check settings is a table
    if type(db.settings) ~= "table" then
        print("MyAddon: Invalid settings, resetting...");
        db.settings = {};
    end

    -- Validate setting values
    if type(db.settings.uiScale) ~= "number" or
       db.settings.uiScale < 0.5 or
       db.settings.uiScale > 2.0 then
        db.settings.uiScale = 1.0;
    end

    return true;
end
```

### Sanitize User Input

```lua
local function SetOption(key, value)
    -- Whitelist allowed keys
    local allowedKeys = {
        debugMode = "boolean",
        uiScale = "number",
        playerName = "string",
    };

    if not allowedKeys[key] then
        error("Invalid option key: " .. tostring(key));
        return;
    end

    -- Type check
    if type(value) ~= allowedKeys[key] then
        error(format("Option '%s' must be %s, got %s",
            key, allowedKeys[key], type(value)));
        return;
    end

    -- Additional validation
    if key == "uiScale" then
        value = Clamp(value, 0.5, 2.0);
    end

    MyAddonDB.settings[key] = value;
end
```

---

## Debugging Saved Variables

### Viewing Saved Variables

```lua
-- In-game command
/dump MyAddonDB

-- Pretty print
/run DevTools_Dump(MyAddonDB)

-- Count keys
/run print("Keys:", count(MyAddonDB))

-- Check size
/run UpdateAddOnMemoryUsage(); print("Size:", GetAddOnMemoryUsage("MyAddon") .. " KB")
```

### Manual File Editing

**Location:**
```
WTF/Account/[Account]/SavedVariables/MyAddon.lua
```

**Format:**
```lua
MyAddonDB = {
    ["version"] = 1,
    ["settings"] = {
        ["debugMode"] = true,
        ["uiScale"] = 1.2,
    },
}
```

**Best Practices:**
- Always backup before editing
- Use proper Lua syntax
- `/reload` to load changes
- Watch for syntax errors

### Debug Logging

```lua
local function DebugPrint(...)
    if MyAddonDB.settings.debugMode then
        print("|cff00ff00[MyAddon]|r", ...);
    end
end

local function DumpDatabase()
    if not MyAddonDB.settings.debugMode then
        return;
    end

    print("=== MyAddon Database ===");
    print("Version:", MyAddonDB.version);
    print("Characters:", count(MyAddonDB.characters));

    for key, data in pairs(MyAddonDB.characters) do
        print(format("  %s: Level %d, Gold %d",
            key, data.level or 0, data.gold or 0));
    end
end
```

---

## Real-World Examples

### Blizzard Addon Examples

**Source:** Blizzard TOC files

**Auction House:**
```
## SavedVariables: g_auctionHouseSortsBySearchContext
## SavedVariablesPerCharacter: g_activeBidAuctionIDs
```

**Combat Log:**
```
## SavedVariables: Blizzard_CombatLog_Filters, Blizzard_CombatLog_Filter_Version
```

**Communities:**
```
## SavedVariablesPerCharacter: g_clubIdToSeenApplicants
```

### Complete Example

**MyAddon.toc:**
```
## Interface: 110207
## Title: My Tracking Addon
## SavedVariables: MyTrackerDB
## SavedVariablesPerCharacter: MyTrackerCharDB

Core.lua
```

**Core.lua:**
```lua
-- Default database structure
local DEFAULTS = {
    version = 1,
    global = {
        syncEnabled = true,
        trackAll = false,
    },
    characters = {},
};

-- Initialize
local frame = CreateFrame("Frame");
frame:RegisterEvent("ADDON_LOADED");
frame:RegisterEvent("PLAYER_LOGOUT");

frame:SetScript("OnEvent", function(self, event, ...)
    if event == "ADDON_LOADED" then
        local addonName = ...;
        if addonName == "MyAddon" then
            -- Initialize saved variables
            MyTrackerDB = MyTrackerDB or {};
            MyTrackerCharDB = MyTrackerCharDB or {};

            -- Apply defaults
            for key, value in pairs(DEFAULTS) do
                if MyTrackerDB[key] == nil then
                    if type(value) == "table" then
                        MyTrackerDB[key] = CopyTable(value);
                    else
                        MyTrackerDB[key] = value;
                    end
                end
            end

            -- Initialize character data
            local charKey = format("%s-%s", GetRealmName(), UnitName("player"));
            if not MyTrackerDB.characters[charKey] then
                MyTrackerDB.characters[charKey] = {
                    created = time(),
                };
            end

            -- Also init per-char DB
            MyTrackerCharDB.items = MyTrackerCharDB.items or {};

            print("MyAddon loaded with version", MyTrackerDB.version);
        end

    elseif event == "PLAYER_LOGOUT" then
        -- Clean up before save
        local charKey = format("%s-%s", GetRealmName(), UnitName("player"));
        if MyTrackerDB.characters[charKey] then
            MyTrackerDB.characters[charKey].lastSeen = time();
        end
    end
end);
```

---

<!-- CLAUDE_SKIP_START -->
## Summary

### Key Points:
1. ✅ Declare variables in TOC file
2. ✅ Use global scope (not local)
3. ✅ Initialize with defaults
4. ✅ Validate loaded data
5. ✅ Version your database
6. ✅ Implement migrations
7. ✅ Keep data size reasonable
8. ✅ Prune old data
9. ✅ Use character keys for per-character data
10. ✅ Don't save functions or metatables

### Common Mistakes:
1. ❌ Using local variables
2. ❌ Not checking if variable exists
3. ❌ Overwriting existing data on init
4. ❌ Storing too much data
5. ❌ Not validating loaded data
6. ❌ Missing version/migration system
7. ❌ Trusting user-edited data
8. ❌ Saving cache/temp data
9. ❌ Not cleaning up old data
10. ❌ Complex nested structures (hard to migrate)

---

**Version:** 1.0 - Based on WoW 11.2.7 (The War Within)
**Last Updated:** 2025-10-19
**Source:** Blizzard TOC files and common community patterns

<!-- CLAUDE_SKIP_END -->
