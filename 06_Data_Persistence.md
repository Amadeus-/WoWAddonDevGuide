# WoW Addon Data Persistence - Saved Variables Guide

## Table of Contents
1. [Saved Variables Overview](#saved-variables-overview)
2. [TOC File Configuration](#toc-file-configuration)
3. [Variable Types and Scopes](#variable-types-and-scopes)
4. [Best Practices for Data Structure](#best-practices-for-data-structure)
5. [Initialization Patterns](#initialization-patterns)
6. [Defaults and Migrations](#defaults-and-migrations)
7. [Data Serialization (C_EncodingUtil)](#data-serialization-c_encodingutil)
8. [Sharing Data Between Addons](#sharing-data-between-addons)
9. [Performance Considerations](#performance-considerations)
10. [Security and Validation](#security-and-validation)
11. [Debugging Saved Variables](#debugging-saved-variables)
12. [Warband and Account-Wide Data](#warband-and-account-wide-data)

---

## Saved Variables Overview

WoW addons persist data between sessions using **saved variables**. These are Lua variables declared in the TOC file that WoW automatically saves/loads from disk.

**Key Concepts:**
- Variables are saved when logging out or `/reload`
- By default, loaded before `ADDON_LOADED` event fires (but after Lua files execute)
- With `LoadSavedVariablesFirst: 1` (11.1.5+), loaded BEFORE any addon Lua files execute
- Stored in `WTF/Account/[Account]/SavedVariables/` or character-specific folders
- Must be global variables (not local)
- Can be any Lua-serializable type (tables, numbers, strings, booleans)

---

## TOC File Configuration

### SavedVariables (Account-Wide)

Shared across all characters on the account:

```
## Interface: 120000
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
## Interface: 120000
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

### LoadSavedVariablesFirst (11.1.5+)

**New in 11.1.5:** This directive changes when saved variables are loaded relative to addon Lua files.

```
## Interface: 120000
## Title: My Addon
## SavedVariables: MyAddonDB
## LoadSavedVariablesFirst: 1
```

**Default Behavior (without directive):**
1. Addon Lua files execute
2. Saved variables are loaded
3. `ADDON_LOADED` event fires

**With `LoadSavedVariablesFirst: 1`:**
1. Saved variables are loaded
2. Addon Lua files execute (saved variables already available!)
3. `ADDON_LOADED` event fires

**Use Cases:**
- Addons that need configuration during file initialization
- Avoiding the `ADDON_LOADED` dance for simple addons
- Settings that affect how files are parsed

**Example with LoadSavedVariablesFirst:**
```lua
-- Core.lua - SavedVariables already loaded when this runs!
-- No need to wait for ADDON_LOADED

-- Directly access saved settings at file scope
local debugMode = MyAddonDB and MyAddonDB.debugMode or false;

if debugMode then
    print("MyAddon: Debug mode enabled at load time!");
end

-- Initialize defaults if needed (still recommended)
MyAddonDB = MyAddonDB or {};
MyAddonDB.debugMode = MyAddonDB.debugMode or false;
```

**Comparison - Traditional vs LoadSavedVariablesFirst:**
```lua
-- TRADITIONAL PATTERN (without LoadSavedVariablesFirst)
local isDebug = false;  -- Must use default, variable not loaded yet

local frame = CreateFrame("Frame");
frame:RegisterEvent("ADDON_LOADED");
frame:SetScript("OnEvent", function(self, event, addonName)
    if addonName == "MyAddon" then
        MyAddonDB = MyAddonDB or {};
        isDebug = MyAddonDB.debugMode;  -- NOW we can read it
        self:UnregisterEvent("ADDON_LOADED");
    end
end);

-- WITH LoadSavedVariablesFirst: 1
-- SavedVariables already loaded, so this works immediately:
MyAddonDB = MyAddonDB or {};
local isDebug = MyAddonDB.debugMode or false;  -- Works at file scope!
```

**Important Notes:**
- Still recommended to use `ADDON_LOADED` for complex initialization
- The directive only changes load ORDER, not the fact that variables may be nil on first run
- Combine with defensive coding (`MyAddonDB = MyAddonDB or {}`)

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

## Data Serialization (C_EncodingUtil)

**New in 11.1.5:** WoW now provides native serialization APIs through `C_EncodingUtil`. This is ideal for sharing data between players (import/export strings), compressing large data, and cross-addon communication.

### JSON Serialization (Human Readable)

```lua
-- Serialize a table to JSON string
local myData = {
    settings = {
        enabled = true,
        scale = 1.5,
    },
    items = {12345, 67890, 11111},
};

local jsonStr = C_EncodingUtil.SerializeJSON(myData);
-- Result: {"settings":{"enabled":true,"scale":1.5},"items":[12345,67890,11111]}

-- Deserialize JSON back to table
local restored = C_EncodingUtil.DeserializeJSON(jsonStr);
print(restored.settings.enabled);  -- true
```

### CBOR Serialization (Binary, Compact)

CBOR (Concise Binary Object Representation) is more compact than JSON and preserves Lua types better:

```lua
-- Serialize to CBOR (binary format)
local cborStr = C_EncodingUtil.SerializeCBOR(myData);

-- Deserialize CBOR back to table
local restored = C_EncodingUtil.DeserializeCBOR(cborStr);
```

**When to use CBOR vs JSON:**
- **JSON:** Human-readable, easier debugging, widely compatible
- **CBOR:** Smaller output, faster serialization, better type preservation

### String Compression

Compress large strings before storage or transmission:

```lua
-- Compress a string
local largeData = string.rep("Hello World! ", 1000);
local compressed = C_EncodingUtil.CompressString(largeData);

-- Decompress back
local original = C_EncodingUtil.DecompressString(compressed);

-- Check compression ratio
print(format("Original: %d bytes, Compressed: %d bytes",
    #largeData, #compressed));
```

### Base64 Encoding (For Sharing)

Convert binary data to safe shareable strings:

```lua
-- Encode for sharing (safe for chat/clipboard)
local base64 = C_EncodingUtil.EncodeBase64(compressed);

-- Decode received string
local decoded = C_EncodingUtil.DecodeBase64(base64);
```

### Complete Import/Export Pattern

```lua
local MyAddon = {};

-- Export settings as shareable string
function MyAddon:ExportSettings()
    local exportData = {
        version = 1,
        settings = MyAddonDB.settings,
        timestamp = time(),
    };

    -- Serialize to JSON
    local json = C_EncodingUtil.SerializeJSON(exportData);

    -- Compress
    local compressed = C_EncodingUtil.CompressString(json);

    -- Base64 encode for safe sharing
    local exportString = C_EncodingUtil.EncodeBase64(compressed);

    return exportString;
end

-- Import settings from shared string
function MyAddon:ImportSettings(importString)
    -- Validate input
    if type(importString) ~= "string" or #importString == 0 then
        return false, "Invalid import string";
    end

    -- Decode Base64
    local decoded = C_EncodingUtil.DecodeBase64(importString);
    if not decoded then
        return false, "Failed to decode Base64";
    end

    -- Decompress
    local decompressed = C_EncodingUtil.DecompressString(decoded);
    if not decompressed then
        return false, "Failed to decompress data";
    end

    -- Parse JSON
    local importData = C_EncodingUtil.DeserializeJSON(decompressed);
    if not importData then
        return false, "Failed to parse JSON";
    end

    -- Validate version
    if not importData.version or importData.version > 1 then
        return false, "Incompatible version";
    end

    -- Apply settings
    if importData.settings then
        for key, value in pairs(importData.settings) do
            MyAddonDB.settings[key] = value;
        end
    end

    return true, "Settings imported successfully!";
end

-- Slash command handlers
SLASH_MYADDON_EXPORT1 = "/myaddonexport";
SlashCmdList["MYADDON_EXPORT"] = function()
    local exportStr = MyAddon:ExportSettings();
    -- Copy to editbox for user to copy
    MyAddon:ShowExportDialog(exportStr);
end
```

### Optimizing SavedVariables with Compression

For addons with large datasets, compress before saving:

```lua
-- Store compressed data in SavedVariables
function MyAddon:SaveLargeData(data)
    local json = C_EncodingUtil.SerializeJSON(data);
    MyAddonDB.compressedData = C_EncodingUtil.CompressString(json);
    MyAddonDB.dataVersion = 1;
end

-- Load and decompress
function MyAddon:LoadLargeData()
    if not MyAddonDB.compressedData then
        return nil;
    end

    local json = C_EncodingUtil.DecompressString(MyAddonDB.compressedData);
    return C_EncodingUtil.DeserializeJSON(json);
end
```

---

## Sharing Data Between Addons

**New in 11.1.7:** Addons can now expose their namespace table for other addons to access via `C_AddOns.GetAddOnLocalTable()`.

### Exposing Your Addon's API

**Step 1: Enable in TOC file**
```
## Interface: 120000
## Title: MyDataAddon
## SavedVariables: MyDataAddonDB
## AllowAddOnTableAccess: 1
```

**Step 2: Create public API in your namespace**
```lua
-- MyDataAddon/Core.lua
local addonName, ns = ...;

-- Private data (not directly accessible)
ns.privateData = {
    internalCache = {},
};

-- Public API (accessible by other addons)
ns.API = {
    -- Get addon version
    GetVersion = function()
        return "1.0.0";
    end,

    -- Get player's tracked data
    GetPlayerData = function(playerName)
        if not MyDataAddonDB.players then
            return nil;
        end
        return CopyTable(MyDataAddonDB.players[playerName] or {});
    end,

    -- Check if tracking is enabled
    IsTrackingEnabled = function()
        return MyDataAddonDB.settings and MyDataAddonDB.settings.trackingEnabled;
    end,

    -- Register callback for data updates
    RegisterCallback = function(callback)
        ns.callbacks = ns.callbacks or {};
        table.insert(ns.callbacks, callback);
    end,
};

-- Internal function to notify callbacks
function ns:NotifyCallbacks(event, data)
    if self.callbacks then
        for _, callback in ipairs(self.callbacks) do
            pcall(callback, event, data);
        end
    end
end
```

### Accessing Another Addon's API

```lua
-- OtherAddon/Core.lua
local addonName, ns = ...;

local function OnAddonLoaded()
    -- Try to get MyDataAddon's namespace
    local dataAddonNS = C_AddOns.GetAddOnLocalTable("MyDataAddon");

    if dataAddonNS and dataAddonNS.API then
        -- Access the API
        local version = dataAddonNS.API.GetVersion();
        print("MyDataAddon version:", version);

        -- Get player data
        local playerData = dataAddonNS.API.GetPlayerData(UnitName("player"));
        if playerData then
            print("Found player data!");
        end

        -- Register for updates
        dataAddonNS.API.RegisterCallback(function(event, data)
            print("Data update:", event);
        end);
    else
        print("MyDataAddon not available or API not exposed");
    end
end

-- Wait for both addons to load
local frame = CreateFrame("Frame");
frame:RegisterEvent("ADDON_LOADED");
frame:SetScript("OnEvent", function(self, event, loadedAddon)
    if loadedAddon == addonName then
        -- Use a slight delay to ensure other addons are ready
        C_Timer.After(0, OnAddonLoaded);
        self:UnregisterEvent("ADDON_LOADED");
    end
end);
```

### Best Practices for Cross-Addon Communication

```lua
-- GOOD: Return copies of data, not references
ns.API = {
    GetSettings = function()
        return CopyTable(ns.db.settings);  -- Safe copy
    end,
};

-- BAD: Returning direct references (allows modification)
ns.API = {
    GetSettings = function()
        return ns.db.settings;  -- Direct reference - dangerous!
    end,
};

-- GOOD: Validate inputs
ns.API = {
    SetOption = function(key, value)
        local validKeys = {debugMode = true, showTooltips = true};
        if not validKeys[key] then
            return false, "Invalid option key";
        end
        ns.db.settings[key] = value;
        return true;
    end,
};

-- GOOD: Version your API
ns.API = {
    API_VERSION = 1,

    IsCompatible = function(requiredVersion)
        return ns.API.API_VERSION >= requiredVersion;
    end,
};
```

### Checking Addon Availability

```lua
-- Check if addon is loaded and has API access enabled
local function IsAddonAPIAvailable(addonName)
    -- Check if addon is loaded
    if not C_AddOns.IsAddOnLoaded(addonName) then
        return false, "Addon not loaded";
    end

    -- Try to get the table
    local addonNS = C_AddOns.GetAddOnLocalTable(addonName);
    if not addonNS then
        return false, "API access not enabled (AllowAddOnTableAccess)";
    end

    return true, addonNS;
end

-- Usage
local available, result = IsAddonAPIAvailable("MyDataAddon");
if available then
    local ns = result;
    -- Use ns.API...
else
    print("Cannot access addon:", result);
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
## Interface: 120000
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

## Warband and Account-Wide Data

**New in The War Within (11.0) and Midnight (12.0):** The Warband system introduced account-wide character data sharing. This section covers addon considerations for Warband data.

### Warband Character Keys

When tracking data across characters, consider using Warband-aware identifiers:

```lua
-- Traditional character key
local function GetCharacterKey()
    local name = UnitName("player");
    local realm = GetRealmName();
    return format("%s-%s", realm, name);
end

-- Warband-aware character GUID (more stable)
local function GetWarbandCharacterID()
    -- Player GUID is stable across sessions
    local guid = UnitGUID("player");
    return guid;
end

-- Get all Warband characters for current account
local function GetWarbandCharacters()
    local characters = {};

    -- Use C_AccountInfo for warband character data
    if C_AccountInfo and C_AccountInfo.GetIDFromBattleNetAccountGUID then
        -- Warband character iteration (when available)
        -- Note: API availability may vary by patch
    end

    return characters;
end
```

### Account-Wide vs Character-Specific Data

Choose the appropriate storage based on data type:

```lua
-- TOC Configuration
-- ## SavedVariables: MyAddonDB           -- Shared account-wide
-- ## SavedVariablesPerCharacter: MyCharDB -- Per-character only

MyAddonDB = {
    -- Account-wide settings (shared across all characters)
    accountSettings = {
        soundEnabled = true,
        language = "enUS",
    },

    -- Warband tracking data (aggregated from all characters)
    warbandData = {
        totalGold = 0,           -- Sum across all characters
        achievements = {},       -- Shared achievements
        collectibles = {},       -- Mounts, pets, etc.
    },

    -- Character registry (keyed by GUID for stability)
    characters = {
        ["Player-GUID-1234"] = {
            name = "CharacterName",
            realm = "RealmName",
            class = "WARRIOR",
            lastSeen = time(),
        },
    },
};

-- Per-character data that should NOT be shared
MyCharDB = {
    -- UI positions specific to this character
    framePositions = {},

    -- Character-specific tracked data
    currentQuests = {},

    -- Session-specific cache
    sessionCache = {},
};
```

### Bank Storage Changes (11.2.0+)

**Important:** Void Storage was removed in 11.2.0. If your addon tracked Void Storage items, update accordingly:

```lua
-- REMOVED: Void Storage APIs (as of 11.2.0)
-- These functions no longer exist:
-- GetVoidItemInfo()
-- GetNumVoidTransferDeposit()
-- GetNumVoidTransferWithdrawal()
-- GetVoidTransferCost()
-- CanUseVoidStorage()
-- IsVoidStorageReady()

-- Bank storage now uses unified Warband bank (11.0+)
-- Use C_Bank API for modern bank operations

-- Check if bank is available
if C_Bank then
    -- Get bank bag slots
    local bagSlots = C_Container.GetNumBankSlots();

    -- Warband bank tab information
    if C_Bank.FetchPurchasedBankTabData then
        C_Bank.FetchPurchasedBankTabData(Enum.BankType.Account);
    end
end
```

### Migration from Void Storage Tracking

If your addon previously tracked Void Storage:

```lua
local function MigrateFromVoidStorage()
    local db = MyAddonDB;

    -- Version check for migration
    if db.version and db.version < 12 then
        -- Remove obsolete void storage data
        if db.voidStorage then
            -- Optionally migrate to regular bank tracking
            -- or just remove the obsolete data
            db.voidStorageLegacy = db.voidStorage;  -- Keep for reference
            db.voidStorage = nil;

            print("MyAddon: Void Storage tracking removed (feature no longer exists)");
        end

        db.version = 12;
    end
end
```

### Warband Bank Tracking

```lua
-- Track items across Warband bank
local function GetWarbandBankItems()
    local items = {};

    -- Warband bank uses account-wide bank type
    if C_Bank and Enum.BankType.Account then
        -- Iterate warband bank tabs
        local numTabs = C_Bank.FetchNumPurchasedBankTabs(Enum.BankType.Account) or 0;

        for tabIndex = 1, numTabs do
            local tabInfo = C_Bank.FetchPurchasedBankTabData(Enum.BankType.Account);
            -- Process tab data...
        end
    end

    return items;
end
```

### Best Practices for Warband Data

```lua
-- 1. Use GUIDs instead of name-realm for character identification
local charGUID = UnitGUID("player");
MyAddonDB.characters[charGUID] = {
    name = UnitName("player"),
    realm = GetRealmName(),
    -- ... other data
};

-- 2. Aggregate data at logout for cross-character visibility
frame:RegisterEvent("PLAYER_LOGOUT");
frame:SetScript("OnEvent", function(self, event)
    if event == "PLAYER_LOGOUT" then
        local charGUID = UnitGUID("player");
        local charData = MyAddonDB.characters[charGUID] or {};

        -- Update character's contribution to warband totals
        charData.gold = GetMoney();
        charData.lastSeen = time();

        MyAddonDB.characters[charGUID] = charData;

        -- Recalculate warband totals
        local totalGold = 0;
        for _, char in pairs(MyAddonDB.characters) do
            totalGold = totalGold + (char.gold or 0);
        end
        MyAddonDB.warbandData.totalGold = totalGold;
    end
end);

-- 3. Handle character deletion gracefully
local function PruneDeletedCharacters()
    -- Characters not seen in 90+ days might be deleted
    local cutoff = time() - (90 * 24 * 60 * 60);

    for guid, data in pairs(MyAddonDB.characters) do
        if data.lastSeen and data.lastSeen < cutoff then
            -- Mark as possibly deleted, don't remove immediately
            data.possiblyDeleted = true;
        end
    end
end
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

**Version:** 2.0 - Updated for WoW 12.0.0 (Midnight)
**Last Updated:** 2026-01-20
**Source:** Blizzard TOC files and common community patterns

### Changes in 2.0:
- Added `LoadSavedVariablesFirst` TOC directive (11.1.5+)
- Added `C_EncodingUtil` serialization APIs (11.1.5+)
- Added `AllowAddOnTableAccess` and `C_AddOns.GetAddOnLocalTable()` (11.1.7+)
- Added Warband and account-wide data section
- Documented Void Storage removal (11.2.0)
- Updated interface version to 120000

<!-- CLAUDE_SKIP_END -->
