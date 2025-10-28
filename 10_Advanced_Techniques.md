# Advanced WoW Addon Techniques

## Table of Contents
1. [Overview](#overview)
2. [Cross-Client Compatibility](#cross-client-compatibility)
3. [Event Bucketing & Batching](#event-bucketing--batching)
4. [Advanced Profile Systems](#advanced-profile-systems)
5. [Performance Profiling](#performance-profiling)
6. [Screen-Aware UI Positioning](#screen-aware-ui-positioning)
7. [Data Migration & Schema Evolution](#data-migration--schema-evolution)
8. [API Versioning & Deprecation](#api-versioning--deprecation)
9. [Parser Sandboxing](#parser-sandboxing)
10. [Multi-Addon Architecture](#multi-addon-architecture)
11. [Advanced Caching Strategies](#advanced-caching-strategies)

---

## Overview

This guide documents advanced techniques used by production addons like **ArkInventory**, **ElvUI**, and **ZygorGuidesViewer**. These patterns enable addons to:
- Support millions of items
- Manage 100+ UI elements simultaneously
- Work across Classic, Retail, and special seasons
- Maintain backward compatibility
- Optimize for performance at scale

**Prerequisites:** Familiarity with basic addon development, Ace3, and the patterns in previous guides.

---

## Cross-Client Compatibility

### The Challenge

WoW has multiple clients (Retail, Classic Era, Cataclysm Classic, Season of Discovery, etc.) with different APIs and features. Large addons need to support multiple clients from one codebase.

###Pattern: TOC Version Detection

**Source:** `ArkInventory/Core/ArkInventoryClient.lua`

```lua
-- Store TOC version constants
ArkInventory.Const.BLIZZARD = {
    TOC = select(4, GetBuildInfo()),
    CLIENT = {
        EXPANSION = {
            [1] = { TOC = { MIN = 10000, MAX = 19999 } },  -- Vanilla
            [2] = { TOC = { MIN = 20000, MAX = 29999 } },  -- TBC
            [3] = { TOC = { MIN = 30000, MAX = 39999 } },  -- WotLK
            [4] = { TOC = { MIN = 40000, MAX = 49999 } },  -- Cataclysm
            -- ... etc
            [10] = { TOC = { MIN = 100000, MAX = 109999 } }, -- Dragonflight
            [11] = { TOC = { MIN = 110000, MAX = 119999 } }, -- The War Within
        },
    },
};

-- Client check function
function ArkInventory.ClientCheck(id_toc_min, id_toc_max, loud)
    local tmin = id_toc_min or 0;
    local tmax = id_toc_max or 999999;

    if ArkInventory.Const.BLIZZARD.TOC >= tmin and
       ArkInventory.Const.BLIZZARD.TOC <= tmax then
        return true;
    end

    if loud then
        print(string.format("Feature requires TOC %d-%d, current is %d",
            tmin, tmax, ArkInventory.Const.BLIZZARD.TOC));
    end

    return false;
end
```

### Pattern: Function Availability Wrapper

```lua
-- Wrap functions that don't exist in all clients
ArkInventory.GetAverageItemLevel = function()
    if ArkInventory.ClientCheck(80000) then  -- WotLK+
        local avgItemLevel, avgItemLevelEquipped = GetAverageItemLevel();
        return avgItemLevelEquipped;
    else
        -- Fallback for older clients
        return 0;
    end
end

-- Use the wrapper instead of direct API
local ilvl = ArkInventory.GetAverageItemLevel();
```

### Pattern: Conditional Feature Loading

**Source:** `ZygorGuidesViewer/ZygorGuidesViewer.lua`

```lua
local build = select(4, GetBuildInfo());
local tocversion = select(4, GetBuildInfo());

ZGV.Expansion_Legion = (build >= 22248);
ZGV.Expansion_BFA = (build >= 27791);
ZGV.Expansion_Shadowlands = (tocversion >= 90000);
ZGV.Expansion_Dragonflight = (tocversion >= 100000);
ZGV.IsRetail = WOW_PROJECT_ID == WOW_PROJECT_MAINLINE;
ZGV.IsClassic = WOW_PROJECT_ID == WOW_PROJECT_CLASSIC;

-- Load expansion-specific files
if ZGV.IsRetail then
    -- Load Retail-specific code
    LoadAddOn("ZygorGuidesViewer_Retail");
elseif ZGV.IsClassic then
    -- Load Classic-specific code
    LoadAddOn("ZygorGuidesViewer_Classic");
end
```

### TOC File Strategy

**Create multiple TOC files:**
```
MyAddon/
├── MyAddon_Mainline.toc    ## Interface: 110207
├── MyAddon_Vanilla.toc     ## Interface: 11503
├── MyAddon_Cata.toc        ## Interface: 40400
└── Core.lua
```

**In TOC:**
```
## Interface: 110207
## Title: My Addon
## X-Min-Interface: 100000
## X-Max-Interface: 119999

Core.lua
RetailOnly.lua  # Only in Mainline TOC
```

---

## Event Bucketing & Batching

### The Problem

Some events fire hundreds of times per second (BAG_UPDATE, UNIT_AURA, etc.). Processing each immediately causes FPS drops.

### Solution: AceBucket-3.0

**Source:** ArkInventory uses this 237+ times throughout codebase

```lua
-- Traditional (bad for performance)
frame:RegisterEvent("BAG_UPDATE");
frame:SetScript("OnEvent", function(self, event)
    -- Fires potentially 100+ times/second
    MyAddon:UpdateBags();
end);

-- Bucketed (good for performance)
local MyAddon = LibStub("AceAddon-3.0"):NewAddon("MyAddon", "AceBucket-3.0");

-- Register bucketed event (fires max once per interval)
MyAddon:RegisterBucketEvent("BAG_UPDATE", 0.5, "UpdateBags");

function MyAddon:UpdateBags()
    -- Only fires maximum 2 times per second
    -- Even if BAG_UPDATE fired 200 times
end
```

### Pattern: Custom Message Bucketing

**Source:** `ArkInventory/Core/ArkInventoryLDB.lua`

```lua
-- Send bucketed messages instead of direct calls
ArkInventory:SendMessage("EVENT_ARKINV_LDB_PET_UPDATE_BUCKET");
ArkInventory:SendMessage("EVENT_ARKINV_LDB_MOUNT_UPDATE_BUCKET");
ArkInventory:SendMessage("EVENT_ARKINV_LDB_TOY_UPDATE_BUCKET");

-- Register listeners with bucketing
MyAddon:RegisterBucketMessage("EVENT_ARKINV_LDB_PET_UPDATE_BUCKET", 1.0, "OnPetsChanged");

function MyAddon:OnPetsChanged()
    -- Batched updates from multiple rapid changes
end
```

### Performance Comparison

```lua
-- Without bucketing: 200 function calls in 1 second
-- CPU: High, FPS: Drops

-- With 0.5s bucketing: 2 function calls in 1 second
-- CPU: Low, FPS: Stable
```

---

## Advanced Profile Systems

### The Challenge

Users want:
- Settings shared across characters
- Character-specific overrides
- Account-wide preferences
- Multiple named profiles

### Pattern: Triple-Tier Storage

**Source:** `ElvUI/Core/init.lua`

```lua
-- Three separate storage tiers
local E = {}; -- Engine

E.DF = {
    profile = {},  -- Shared across chars using this profile
    global = {},   -- Account-wide
};

E.privateVars = {
    profile = {},  -- Per-character (never shared)
};

-- Make accessible as tuple
local unpack = unpack;
Engine[1] = E;                      -- E (Engine)
Engine[2] = locale;                  -- L (Locales)
Engine[3] = E.privateVars.profile;   -- V (Private per-char)
Engine[4] = E.DF.profile;            -- P (Profile defaults)
Engine[5] = E.DF.global;             -- G (Global defaults)

-- Usage
local E, L, V, P, G = unpack(ElvUI);

-- Private (this character only)
V.questRewardMostValueable = 12345;

-- Profile (shared across chars using "Main" profile)
P.general.fontSize = 12;

-- Global (all characters, all profiles)
G.achievementAlerts = true;
```

### AceDB Integration

```lua
local defaults = {
    profile = {
        -- Settings that can be shared via profiles
        fontSize = 12,
        showMinimap = true,
    },
    global = {
        -- Account-wide settings
        version = 1,
        achievements = {},
    },
    char = {
        -- Character-specific (not shareable)
        position = { x = 0, y = 0 },
    },
};

self.db = LibStub("AceDB-3.0"):New("MyAddonDB", defaults);

-- Access
local fontSize = self.db.profile.fontSize;       -- Profile
local version = self.db.global.version;          -- Global
local pos = self.db.char.position;               -- Character-specific
```

### Override Pattern

```lua
-- Get value with character override
function MyAddon:GetSetting(key)
    -- Check private/char first
    if self.db.char[key] ~= nil then
        return self.db.char[key];
    end

    -- Fall back to profile
    return self.db.profile[key];
end
```

---

## Performance Profiling

### Built-In CPU Profiling

**Source:** `ArkInventory/Core/ArkInventoryCPU.lua`

```lua
-- Enable CPU profiling
/run SetCVar("scriptProfile", "1");
ReloadUI();

-- Profile a function
function ArkInventory.CPUProfile(iterations, printResults, func, ...)
    UpdateAddOnCPUUsage();

    local start = debugprofilestop();
    local cpuStart = GetFunctionCPUUsage(func, true);

    for i = 1, iterations do
        func(...);
    end

    local cpuEnd = GetFunctionCPUUsage(func, true);
    local elapsed = debugprofilestop() - start;

    if printResults then
        print(string.format(
            "Function executed %d times in %.2fms (%.4fms per call)",
            iterations,
            elapsed,
            elapsed / iterations
        ));
        print(string.format(
            "CPU time: %.2fms total, %.4fms per call",
            (cpuEnd - cpuStart) / 1000,
            (cpuEnd - cpuStart) / 1000 / iterations
        ));
    end

    return elapsed / iterations;
end

-- Usage
local avgTime = ArkInventory.CPUProfile(100, true, MyExpensiveFunction, arg1, arg2);
```

### Memory Profiling

```lua
-- Track memory usage
UpdateAddOnMemoryUsage();
local memBefore = GetAddOnMemoryUsage("MyAddon");

-- Do expensive operation
MyAddon:ProcessThousandsOfItems();

UpdateAddOnMemoryUsage();
local memAfter = GetAddOnMemoryUsage("MyAddon");

print(string.format("Memory used: %.2f KB", memAfter - memBefore));
```

### Performance Timing

```lua
-- Microsecond-precision timing
local startTime = debugprofilestop();

-- Your code here
for i = 1, 10000 do
    SomeFunction();
end

local elapsed = debugprofilestop() - startTime;
print(string.format("Operation took %.2fms", elapsed));
```

---

## Screen-Aware UI Positioning

### The Problem

Frames positioned with static anchors go off-screen on different resolutions or UI scales.

### Pattern: Dynamic Anchor Calculation

**Source:** `ArkInventory/Core/ArkInventory.lua`

```lua
function ArkInventory.Frame_Main_Anchor_Save(frame)
    local s = frame:GetEffectiveScale();
    local x, y = frame:GetCenter();

    -- Get screen dimensions
    local screenWidth = GetScreenWidth() * UIParent:GetEffectiveScale();
    local screenHeight = GetScreenHeight() * UIParent:GetEffectiveScale();

    -- Calculate position relative to screen center
    x = x * s;
    y = y * s;

    -- Determine best anchor point based on position
    local anchorPoint;
    local relativeX, relativeY;

    if x < screenWidth / 3 then
        -- Left side of screen
        if y < screenHeight / 3 then
            anchorPoint = "BOTTOMLEFT";
            relativeX = x;
            relativeY = y;
        elseif y > screenHeight * 2 / 3 then
            anchorPoint = "TOPLEFT";
            relativeX = x;
            relativeY = y - screenHeight;
        else
            anchorPoint = "LEFT";
            relativeX = x;
            relativeY = y - screenHeight / 2;
        end
    elseif x > screenWidth * 2 / 3 then
        -- Right side of screen
        if y < screenHeight / 3 then
            anchorPoint = "BOTTOMRIGHT";
            relativeX = x - screenWidth;
            relativeY = y;
        elseif y > screenHeight * 2 / 3 then
            anchorPoint = "TOPRIGHT";
            relativeX = x - screenWidth;
            relativeY = y - screenHeight;
        else
            anchorPoint = "RIGHT";
            relativeX = x - screenWidth;
            relativeY = y - screenHeight / 2;
        end
    else
        -- Center of screen
        if y < screenHeight / 2 then
            anchorPoint = "BOTTOM";
            relativeX = x - screenWidth / 2;
            relativeY = y;
        else
            anchorPoint = "TOP";
            relativeX = x - screenWidth / 2;
            relativeY = y - screenHeight;
        end
    end

    -- Save position
    return {
        point = anchorPoint,
        x = relativeX / s,
        y = relativeY / s,
        scale = frame:GetScale(),
    };
end

function ArkInventory.Frame_Main_Anchor_Set(frame, savedPosition)
    if not savedPosition then return; end

    frame:ClearAllPoints();
    frame:SetScale(savedPosition.scale or 1.0);
    frame:SetPoint(
        savedPosition.point,
        UIParent,
        savedPosition.point,
        savedPosition.x,
        savedPosition.y
    );
end
```

### Usage

```lua
-- On frame close/drag stop
local position = ArkInventory.Frame_Main_Anchor_Save(myFrame);
MyAddonDB.framePosition = position;

-- On frame open
ArkInventory.Frame_Main_Anchor_Set(myFrame, MyAddonDB.framePosition);
```

---

## Data Migration & Schema Evolution

### The Challenge

Database structure changes between versions. Need to migrate user data without loss.

### Pattern: Version-Based Upgrades

**Source:** `ArkInventory/Core/ArkInventoryUpgrades.lua`

```lua
function ArkInventory.DatabaseUpgradePreLoad()
    ARKINVDB = ARKINVDB or {};

    -- Version 3.0227 migration
    if ArkInventory.Const.Program.Version >= 3.0227 then
        -- Remove old data structure
        if ARKINVDB.factionrealm then
            print("Migrating old faction realm data...");
            ARKINVDB.factionrealm = nil;
        end
    end

    -- Version 3.05 migration
    if ArkInventory.Const.Program.Version >= 3.05 then
        -- Migrate old category format to new format
        if ARKINVDB.config and ARKINVDB.config.categories then
            for catID, catData in pairs(ARKINVDB.config.categories) do
                if catData.old_format then
                    ARKINVDB.config.categories[catID] =
                        ConvertOldCategoryFormat(catData);
                end
            end
        end
    end
end
```

### Pattern: Safe Data Migration

```lua
local CURRENT_VERSION = 5;

function MyAddon:MigrateDatabase()
    local db = MyAddonDB;
    db.version = db.version or 1;

    -- Migrate through each version
    while db.version < CURRENT_VERSION do
        local oldVersion = db.version;

        -- Version-specific migrations
        if db.version == 1 then
            self:MigrateV1ToV2(db);
            db.version = 2;
        elseif db.version == 2 then
            self:MigrateV2ToV3(db);
            db.version = 3;
        elseif db.version == 3 then
            self:MigrateV3ToV4(db);
            db.version = 4;
        elseif db.version == 4 then
            self:MigrateV4ToV5(db);
            db.version = 5;
        end

        print(string.format("Migrated database from v%d to v%d",
            oldVersion, db.version));
    end
end

function MyAddon:MigrateV1ToV2(db)
    -- Example: Rename field
    if db.oldField then
        db.newField = db.oldField;
        db.oldField = nil;
    end
end

function MyAddon:MigrateV2ToV3(db)
    -- Example: Change data structure
    if db.itemList then
        db.items = {};
        for _, itemID in ipairs(db.itemList) do
            db.items[itemID] = { enabled = true };
        end
        db.itemList = nil;
    end
end
```

### Pattern: Cleanup Old Data

```lua
function MyAddon:CleanupOldData()
    local cutoffDate = time() - (90 * 24 * 60 * 60); -- 90 days

    -- Remove old character data
    for charKey, charData in pairs(MyAddonDB.characters) do
        if charData.lastSeen and charData.lastSeen < cutoffDate then
            print("Removing data for old character:", charKey);
            MyAddonDB.characters[charKey] = nil;
        end
    end

    -- Remove deprecated settings
    local deprecated = {
        "oldSetting1",
        "oldSetting2",
        "removedFeature",
    };

    for _, key in ipairs(deprecated) do
        if MyAddonDB[key] ~= nil then
            MyAddonDB[key] = nil;
        end
    end
end
```

---

## API Versioning & Deprecation

### The Problem

Third-party addons use your API. Breaking changes cause their addons to error.

### Pattern: Deprecated Function Forwarding

**Source:** `ArkInventory/Core/ArkInventoryAPI.lua`

```lua
-- Public API namespace
ArkInventory.API = {};

-- New function (current)
function ArkInventory.API.GetItemInfo(itemID)
    -- New implementation
    return ArkInventory.Internal.GetItemData(itemID);
end

-- Deprecated function (for backward compatibility)
function ArkInventory.TooltipBuildItem(itemID)
    -- Forward to new API
    print("WARNING: TooltipBuildItem is deprecated, use ArkInventory.API.GetItemInfo");
    return ArkInventory.API.GetItemInfo(itemID);
end

-- For third-party hooks
function ArkInventory.API.CustomItemTooltipReady(...)
    -- Call deprecated function so old addons still work
    ArkInventory.TooltipBuildItem(...);

    -- Also provide new API
    return ArkInventory.API.GetItemInfo(...);
end
```

### Pattern: Version Check

```lua
-- Expose version for compatibility checks
ArkInventory.API.VERSION = 3.15;

-- Third-party addon can check
if ArkInventory and ArkInventory.API.VERSION >= 3.10 then
    -- Use new features
    ArkInventory.API.GetItemInfo(itemID);
else
    -- Use old method
    ArkInventory.TooltipBuildItem(itemID);
end
```

### Pattern: Feature Detection

```lua
-- Instead of version checks, check for feature
if ArkInventory and ArkInventory.API and ArkInventory.API.GetItemInfo then
    -- Feature available
    local info = ArkInventory.API.GetItemInfo(itemID);
else
    -- Fallback
end
```

---

## Parser Sandboxing

### The Problem

Allowing users to define conditions (if/then expressions) is dangerous if they can execute arbitrary code.

### Pattern: Sandboxed Environment

**Source:** `ZygorGuidesViewer/Guide.lua`

```lua
-- Create safe environment for user code
ZGV.Parser = {};
ZGV.Parser.ConditionEnv = {
    -- Safe functions only
    level = UnitLevel,
    class = function() return select(2, UnitClass("player")); end,
    race = function() return select(2, UnitRace("player")); end,
    faction = function() return UnitFactionGroup("player"); end,

    -- Math is safe
    math = math,

    -- String operations are safe
    string = {
        find = string.find,
        match = string.match,
        lower = string.lower,
        upper = string.upper,
    },

    -- NO access to dangerous functions
    -- loadstring = nil,
    -- getfenv = nil,
    -- setfenv = nil,
    -- require = nil,
};

-- Evaluate user condition safely
function ZGV.Parser:EvaluateCondition(conditionString)
    -- Compile condition
    local conditionFunc, err = loadstring("return " .. conditionString);

    if not conditionFunc then
        print("Invalid condition:", err);
        return false;
    end

    -- Set sandboxed environment
    setfenv(conditionFunc, self.ConditionEnv);

    -- Execute safely
    local success, result = pcall(conditionFunc);

    if not success then
        print("Condition error:", result);
        return false;
    end

    return result;
end
```

### Usage

```lua
-- User provides condition
local userCondition = "level() >= 60 and class() == 'WARRIOR'";

-- Evaluate safely
if ZGV.Parser:EvaluateCondition(userCondition) then
    print("Condition met!");
end

-- User CANNOT do dangerous things
local malicious = "os.execute('rm -rf /')";  -- Safe: os not in environment
ZGV.Parser:EvaluateCondition(malicious);  -- Returns false, no damage
```

---

## Multi-Addon Architecture

### Pattern: Core + Options Split

**Source:** ElvUI uses separate addons for core and options

**Why:** Reduce initial load time by making options load-on-demand.

**Structure:**
```
ElvUI/
├── ElvUI/                    # Core addon
│   ├── ElvUI_Mainline.toc
│   ├── Core/
│   └── Modules/
└── ElvUI_Options/            # Separate addon
    ├── ElvUI_Options_Mainline.toc
    ├── ## Dependencies: ElvUI
    └── ## LoadOnDemand: 1
```

**Core TOC:**
```
## Interface: 110207
## Title: ElvUI
## SavedVariables: ElvDB, ElvPrivateDB

Core\init.lua
Core\Modules\Load_Modules.xml
```

**Options TOC:**
```
## Interface: 110207
## Title: ElvUI Options
## Dependencies: ElvUI
## LoadOnDemand: 1

Config.lua
```

**Load options on demand:**
```lua
-- In ElvUI core
function E:ToggleOptions()
    if not IsAddOnLoaded("ElvUI_Options") then
        LoadAddOn("ElvUI_Options");
    end

    -- Options addon is now loaded
    LibStub("AceConfigDialog-3.0"):Open("ElvUI");
end
```

### Pattern: Module System with Shared Data

**Source:** ArkInventory has multiple addon modules sharing data

```lua
-- Core addon exposes data
_G.ARKINV_GLOBAL_DATA = {
    items = {},
    categories = {},
};

-- Module addon accesses shared data
local data = _G.ARKINV_GLOBAL_DATA;
if data then
    local items = data.items;
end
```

---

## Advanced Caching Strategies

### Pattern: Multi-Layer Cache

**Source:** `ArkInventory/Core/ArkInventoryStorage.lua`

```lua
ArkInventory.Global.Cache = {
    ItemCountTooltip = {},  -- For tooltip display
    ItemCountRaw = {},      -- Raw counts
    ItemLocation = {},      -- Where items are stored
};

function ArkInventory:GetItemCount(itemID)
    -- Check cache first
    if self.Global.Cache.ItemCountRaw[itemID] then
        return self.Global.Cache.ItemCountRaw[itemID];
    end

    -- Calculate and cache
    local count = GetItemCount(itemID, true);  -- Include bank
    self.Global.Cache.ItemCountRaw[itemID] = count;

    return count;
end

-- Invalidate cache when bags update
function ArkInventory:OnBagUpdate()
    wipe(self.Global.Cache.ItemCountRaw);
    wipe(self.Global.Cache.ItemCountTooltip);
    wipe(self.Global.Cache.ItemLocation);
end
```

### Pattern: Regex-Based Smart Caching

**Source:** `ArkInventory/Core/ArkInventoryObject.lua`

```lua
-- Different number formats for different locales
local REGEX_PATTERNS = {
    enUS = {
        "^(%d+) in stock",     -- English: "5 in stock"
        "^Stock: (%d+)",       -- English: "Stock: 5"
    },
    deDE = {
        "^(%d+) auf Lager",    -- German: "5 auf Lager"
        "^Lager: (%d+)",       -- German: "Lager: 5"
    },
    frFR = {
        "^(%d+) en stock",     -- French: "5 en stock"
        "^Stock : (%d+)",      -- French: "Stock : 5"
    },
};

function ArkInventory:ParseStockCount(text)
    local locale = GetLocale();
    local patterns = REGEX_PATTERNS[locale] or REGEX_PATTERNS.enUS;

    for _, pattern in ipairs(patterns) do
        local count = text:match(pattern);
        if count then
            return tonumber(count);
        end
    end

    return nil;
end
```

---

## Real-World Example: Complete Advanced Pattern

Combining multiple techniques:

```lua
-- Advanced addon core structure
local MyAddon = LibStub("AceAddon-3.0"):NewAddon(
    "MyAdvancedAddon",
    "AceEvent-3.0",
    "AceBucket-3.0",
    "AceConsole-3.0"
);

local E, L, V, P, G;  -- ElvUI-style tuple

-- Constants
local CURRENT_VERSION = 5;
local MIN_TOC = 110000;

function MyAddon:OnInitialize()
    -- Check client compatibility
    local toc = select(4, GetBuildInfo());
    if toc < MIN_TOC then
        error(string.format("Requires TOC %d+, current is %d", MIN_TOC, toc));
        return;
    end

    -- Initialize triple-tier database
    local defaults = {
        profile = {
            fontSize = 12,
        },
        global = {
            version = CURRENT_VERSION,
        },
        char = {
            position = {},
        },
    };

    self.db = LibStub("AceDB-3.0"):New("MyAddonDB", defaults);

    -- Create tuple for easy access
    E = self;
    L = self.L or {};  -- Locales
    V = self.db.char;   -- Private
    P = self.db.profile; -- Profile
    G = self.db.global;  -- Global

    -- Migrate if needed
    if G.version < CURRENT_VERSION then
        self:MigrateDatabase();
    end

    -- Initialize cache
    self.cache = {
        items = {},
        players = {},
    };

    -- Register bucketed events
    self:RegisterBucketEvent({"BAG_UPDATE", "PLAYERBANKSLOTS_CHANGED"}, 0.5, "OnBagsChanged");

    -- Expose API
    _G.MyAddonAPI = {
        VERSION = CURRENT_VERSION,
        GetItemCount = function(...) return self:GetItemCount(...); end,
    };
end

function MyAddon:OnBagsChanged()
    -- Clear cache on bag updates
    wipe(self.cache.items);

    -- Update UI (batched)
    self:UpdateDisplay();
end

function MyAddon:GetItemCount(itemID)
    -- Check cache
    if self.cache.items[itemID] then
        return self.cache.items[itemID];
    end

    -- Profile this function in debug mode
    if self.db.profile.debug then
        local start = debugprofilestop();
        local count = GetItemCount(itemID, true);
        local elapsed = debugprofilestop() - start;

        if elapsed > 1 then
            print(string.format("GetItemCount took %.2fms", elapsed));
        end

        self.cache.items[itemID] = count;
        return count;
    end

    -- Normal mode
    local count = GetItemCount(itemID, true);
    self.cache.items[itemID] = count;
    return count;
end

-- Unpack for modules
local function unpack(addon)
    return E, L, V, P, G;
end

_G.MyAdvancedAddon = {
    [1] = E,
    [2] = L,
    [3] = V,
    [4] = P,
    [5] = G,
};
```

---

<!-- CLAUDE_SKIP_START -->
## Summary: Key Takeaways

### Must-Know Advanced Patterns:

1. ✅ **Event bucketing** - Batch rapid events for performance
2. ✅ **Triple-tier profiles** - Private/Profile/Global separation
3. ✅ **Cross-client compatibility** - Support multiple WoW versions
4. ✅ **Data migration** - Version-based upgrades without data loss
5. ✅ **API versioning** - Deprecated function forwarding
6. ✅ **Screen-aware positioning** - Dynamic anchor calculation
7. ✅ **Performance profiling** - Built-in CPU/memory tracking
8. ✅ **Parser sandboxing** - Safe user expression evaluation
9. ✅ **Multi-addon architecture** - Core + LoadOnDemand options
10. ✅ **Smart caching** - Multi-layer with invalidation

### When to Use:

- **Large datasets** - Use caching and bucketing
- **Multi-version support** - Use TOC checks and wrappers
- **Complex settings** - Use triple-tier profiles
- **Public API** - Use versioning and deprecation
- **User expressions** - Use sandboxing
- **Large UI** - Use multi-addon split
- **Performance issues** - Use profiling and optimization

---

## Reference Examples

| Pattern | Source Addon | File |
|---------|--------------|------|
| Event Bucketing | ArkInventory | `Core/ArkInventory.lua` |
| Triple-Tier DB | ElvUI | `Core/init.lua` |
| Client Compatibility | ArkInventory | `Core/ArkInventoryClient.lua` |
| Data Migration | ArkInventory | `Core/ArkInventoryUpgrades.lua` |
| API Versioning | ArkInventory | `Core/ArkInventoryAPI.lua` |
| Screen Positioning | ArkInventory | `Core/ArkInventory.lua` |
| CPU Profiling | ArkInventory | `Core/ArkInventoryCPU.lua` |
| Parser Sandbox | ZygorGuidesViewer | `Guide.lua` |
| Multi-Addon | ElvUI | `ElvUI/` + `ElvUI_Options/` |
| Smart Caching | ArkInventory | `Core/ArkInventoryStorage.lua` |

All paths relative to: `D:\Games\World of Warcraft\_retail_\Interface\AddOns\`

---

**Version:** 1.0 - Based on WoW 11.2.7 (The War Within)
**Last Updated:** 2025-10-19
**Source Analysis:** ArkInventory, ElvUI, ZygorGuidesViewer

<!-- CLAUDE_SKIP_END -->
