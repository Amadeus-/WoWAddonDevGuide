# Advanced WoW Addon Techniques

## Table of Contents
1. [Overview](#overview)
2. [Secret Values System (12.0.0)](#secret-values-system-1200)
3. [Secure Code & Restricted Actions](#secure-code--restricted-actions)
4. [Cross-Client Compatibility](#cross-client-compatibility)
5. [Event Bucketing & Batching](#event-bucketing--batching)
6. [Advanced Profile Systems](#advanced-profile-systems)
7. [Performance Profiling](#performance-profiling)
8. [Addon Profiling API (11.0.7+)](#addon-profiling-api-1107)
9. [Screen-Aware UI Positioning](#screen-aware-ui-positioning)
10. [Data Migration & Schema Evolution](#data-migration--schema-evolution)
11. [API Versioning & Deprecation](#api-versioning--deprecation)
12. [Parser Sandboxing](#parser-sandboxing)
13. [Multi-Addon Architecture](#multi-addon-architecture)
14. [Advanced Caching Strategies](#advanced-caching-strategies)
15. [Encounter Integration (12.0.0)](#encounter-integration-1200)
16. [Damage Meter Integration (12.0.0)](#damage-meter-integration-1200)
17. [Performance Optimizations (11.1.7+)](#performance-optimizations-1117)

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

## Secret Values System (12.0.0)

> **Complete Reference:** See `12a_Secret_Safe_APIs.md` for the comprehensive secret values API reference, including all detection functions, manipulation functions, SecureTypes containers, and real-world examples from Platynator and Blizzard UI source.

### Overview

Starting with WoW 12.0.0 (Midnight expansion), Blizzard introduced a **secret values system** to protect sensitive combat information from automation addons. This is a critical change that affects how addons interact with combat-related APIs.

### Understanding Secrets

When secret restrictions are active (typically during encounters, M+ dungeons, or PvP), certain combat APIs return "secret" values that tainted code cannot operate on:

- **Addons cannot read** secret values (attempting to do so returns `nil` or errors)
- **Addons cannot compare** secrets (no `==`, `~=`, `<`, `>` comparisons)
- **Addons cannot do arithmetic** on secrets (no `+`, `-`, `*`, `/`)
- **Addons CAN store** secrets in variables
- **Addons CAN pass** secrets to functions
- **Addons CAN use** secrets with specific widget APIs (Cooldown, StatusBar)

### Built-In Secret Functions

```lua
-- Check if a value is a secret
issecretvalue(value)        -- Returns true if value is secret

-- Check if caller can access secret values
canaccessvalue(value)       -- Check specific value accessibility
canaccesstable(table)       -- Check if table contains accessible values
canaccesssecrets()          -- General secret access check

-- Drop access to secrets (useful for library code)
dropsecretaccess()          -- Caller loses ability to access secrets

-- Remove secrets from a table (useful for serialization)
scrubsecretvalues(table)    -- Replaces secrets with nil

-- Create a secret value (for testing)
secretwrap(value)           -- Wrap a value as secret

-- Safe string concatenation with secrets
string.concat(...)          -- Concatenate strings that may contain secrets
```

### Secret Predicates

Check which categories of information are currently protected:

```lua
-- Check if secret restrictions are active
if C_Secrets.HasSecretRestrictions() then
    -- We're in a protected context (encounter, M+, PvP)
end

-- Check specific categories
C_Secrets.ShouldAurasBeSecret()           -- Buff/debuff info protected?
C_Secrets.ShouldCooldownsBeSecret()       -- Cooldown info protected?
C_Secrets.ShouldUnitIdentityBeSecret()    -- Unit names/GUIDs protected?
C_Secrets.ShouldUnitComparisonBeSecret()  -- UnitIsUnit() comparisons protected?
```

### Working with Secret Cooldowns

Cooldowns and durations often return secret values during encounters. Use the new widget APIs:

```lua
-- Traditional approach (may fail with secrets)
local start, duration = GetSpellCooldown(spellID)
-- If secrets are active, start and duration may be secret values!

-- Secret-safe approach using Duration objects
local duration = C_DurationUtil.CreateDuration()
-- Duration object can hold secret time values

-- Set cooldown using duration object
cooldown:SetCooldownFromDurationObject(duration)

-- Set timer on status bar
statusBar:SetTimerDuration(duration)
```

### Curve Objects for Secret-Safe UI

For displaying values that may be secret (like health percentages), use Curve objects:

```lua
-- Create a value curve (maps input values to output)
local valueCurve = C_CurveUtil.CreateCurve()
valueCurve:AddControlPoint(0, 0)      -- At 0 input, output 0
valueCurve:AddControlPoint(100, 1)    -- At 100 input, output 1

-- Create a color curve (maps values to colors)
local colorCurve = C_CurveUtil.CreateColorCurve()
colorCurve:AddColorStop(0, 1, 0, 0, 1)    -- Red at 0%
colorCurve:AddColorStop(50, 1, 1, 0, 1)   -- Yellow at 50%
colorCurve:AddColorStop(100, 0, 1, 0, 1)  -- Green at 100%

-- Apply to StatusBar (works with secret values)
statusBar:SetStatusBarCurve(valueCurve)
statusBar:SetStatusBarColorCurve(colorCurve)
```

### Pattern: Handling Secret Values Gracefully

```lua
local function UpdateCooldownDisplay(cooldownFrame, spellID)
    local start, duration, enabled = GetSpellCooldown(spellID)

    -- Check if values are secret
    if issecretvalue(start) or issecretvalue(duration) then
        -- Use Duration object for secret-safe display
        local durationObj = C_DurationUtil.CreateDuration()
        cooldownFrame:SetCooldownFromDurationObject(durationObj)

        -- Optionally show "protected" indicator
        cooldownFrame.protectedIcon:Show()
    else
        -- Normal path when secrets not active
        if enabled and duration > 0 then
            cooldownFrame:SetCooldown(start, duration)
        else
            cooldownFrame:Clear()
        end
        cooldownFrame.protectedIcon:Hide()
    end
end
```

### Pattern: Safe Serialization

```lua
-- Before saving data that might contain secrets
function MyAddon:SaveCombatData(data)
    -- Scrub any secret values before saving
    local cleanData = CopyTable(data)
    scrubsecretvalues(cleanData)

    -- Now safe to serialize
    MyAddonDB.combatData = cleanData
end
```

### Testing Secret Restrictions

Use CVars to force secret restrictions for testing:

```lua
-- Force secret restrictions (for testing outside encounters)
/run SetCVar("secretCombatRestrictionsForced", 1)

-- Force encounter-specific restrictions
/run SetCVar("secretEncounterRestrictionsForced", 1)

-- Force M+ restrictions
/run SetCVar("secretChallengeModeRestrictionsForced", 1)

-- Force PvP restrictions
/run SetCVar("secretPvPMatchRestrictionsForced", 1)

-- Clear all forced restrictions
/run SetCVar("secretCombatRestrictionsForced", 0)
```

### Responding to Restriction Changes

```lua
local frame = CreateFrame("Frame")
frame:RegisterEvent("ADDON_RESTRICTION_STATE_CHANGED")
frame:SetScript("OnEvent", function(self, event)
    if event == "ADDON_RESTRICTION_STATE_CHANGED" then
        local hasRestrictions = C_Secrets.HasSecretRestrictions()
        if hasRestrictions then
            -- Switch to secret-safe display mode
            MyAddon:EnableSecretSafeMode()
        else
            -- Switch back to full information display
            MyAddon:DisableSecretSafeMode()
        end
    end
end)
```

---

## Secure Code & Restricted Actions

### Understanding Taint

WoW uses a **taint system** to protect sensitive actions. Code executed from addon files is "tainted" while Blizzard UI code is "secure."

```lua
-- Tainted code CANNOT:
-- - Toggle action bars in combat
-- - Cast spells programmatically
-- - Change target in combat
-- - Modify secure frame attributes in combat

-- Check if in combat lockdown
if InCombatLockdown() then
    -- Cannot modify secure frames
    print("Cannot change settings during combat")
    return
end
```

### C_RestrictedActions Namespace

The `C_RestrictedActions` namespace provides information about what actions are currently restricted:

```lua
-- Check if a specific action is restricted
if C_RestrictedActions.IsRestricted() then
    -- General restriction check
end

-- Check specific restriction types
if C_RestrictedActions.IsCombatRestricted() then
    -- Combat-specific restrictions active
end

-- Safe pattern for restricted actions
function MyAddon:PerformAction()
    if InCombatLockdown() then
        -- Queue action for after combat
        self:RegisterEvent("PLAYER_REGEN_ENABLED")
        self.pendingAction = true
        return false
    end

    -- Safe to perform action
    self:DoTheActualAction()
    return true
end

function MyAddon:PLAYER_REGEN_ENABLED()
    self:UnregisterEvent("PLAYER_REGEN_ENABLED")
    if self.pendingAction then
        self.pendingAction = false
        self:PerformAction()
    end
end
```

### Secure Template Usage

For buttons that need to perform protected actions:

```lua
-- Create secure action button
local button = CreateFrame("Button", "MySecureButton", UIParent, "SecureActionButtonTemplate")

-- Set attributes BEFORE combat
button:SetAttribute("type", "spell")
button:SetAttribute("spell", "Fireball")

-- Cannot change these during combat!
-- This will silently fail in combat:
-- button:SetAttribute("spell", "Frostbolt")  -- NO!
```

### ADDON_RESTRICTION_STATE_CHANGED Event

```lua
-- Register for restriction changes
local frame = CreateFrame("Frame")
frame:RegisterEvent("ADDON_RESTRICTION_STATE_CHANGED")

frame:SetScript("OnEvent", function(self, event)
    -- Restrictions have changed
    -- Could be entering/leaving encounter, M+, or PvP

    local restricted = C_Secrets.HasSecretRestrictions()
    print("Addon restrictions active:", restricted)

    -- Update addon behavior accordingly
    MyAddon:OnRestrictionsChanged(restricted)
end)
```

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
            [12] = { TOC = { MIN = 120000, MAX = 129999 } }, -- Midnight
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
ZGV.Expansion_WarWithin = (tocversion >= 110000);
ZGV.Expansion_Midnight = (tocversion >= 120000);
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

#### Option 1: Comma-Separated Interface Versions (10.1.0+, Recommended)

Since Patch 10.1.0, you can declare multiple Interface versions in a single TOC file:

```
MyAddon/
├── MyAddon.toc
└── Core.lua
```

**MyAddon.toc:**
```
## Interface: 120000, 110207, 50503, 40402, 11508
## Title: My Addon
## Version: 1.0.0

Core.lua
```

**Use comma-separated versions when:**
- Your addon code is identical across all versions
- API differences are handled in Lua with runtime detection
- Simpler maintenance is preferred

#### Option 2: Multiple TOC Files (When Needed)

**Create multiple TOC files when you need different files per version:**
```
MyAddon/
├── MyAddon_Mainline.toc    ## Interface: 120000
├── MyAddon_Vanilla.toc     ## Interface: 11508
├── MyAddon_Cata.toc        ## Interface: 40402
├── Core.lua                # Shared
├── RetailOnly.lua          # Only in Mainline TOC
└── ClassicOnly.lua         # Only in Classic TOCs
```

**In Mainline TOC:**
```
## Interface: 120000
## Title: My Addon

Core.lua
RetailOnly.lua
```

**In Classic TOC:**
```
## Interface: 11508
## Title: My Addon

Core.lua
ClassicOnly.lua
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

## Addon Profiling API (11.0.7+)

### Overview

Starting with 11.0.7, WoW provides a dedicated `C_AddOnProfiler` API for measuring addon performance. This replaced older methods and provides more accurate metrics.

### Getting Addon Metrics

```lua
-- Get specific addon metrics
local avgTime = C_AddOnProfiler.GetAddOnMetric(
    "MyAddon",
    Enum.AddOnProfilerMetric.SessionAverageTime
)

-- Available metrics (Enum.AddOnProfilerMetric)
Enum.AddOnProfilerMetric.SessionAverageTime     -- Average over entire session
Enum.AddOnProfilerMetric.RecentAverageTime      -- Average over recent samples
Enum.AddOnProfilerMetric.EncounterAverageTime   -- Average during encounters
Enum.AddOnProfilerMetric.LastTime               -- Most recent frame time
Enum.AddOnProfilerMetric.PeakTime               -- Peak time recorded
Enum.AddOnProfilerMetric.CountTimeOver1Ms       -- Frames taking >1ms
Enum.AddOnProfilerMetric.CountTimeOver5Ms       -- Frames taking >5ms
Enum.AddOnProfilerMetric.CountTimeOver10Ms      -- Frames taking >10ms
Enum.AddOnProfilerMetric.CountTimeOver50Ms      -- Frames taking >50ms
```

### Profiling Specific Functions (11.1.7+)

```lua
-- Measure a specific function call
local result = C_AddOnProfiler.MeasureCall(myExpensiveFunction, arg1, arg2)

-- This wraps the function call with timing and returns:
-- result.returnValues  - Table of return values from the function
-- result.elapsedMs     - Time taken in milliseconds
```

### Pattern: Self-Profiling Addon

```lua
local MyAddon = {}
MyAddon.profiling = {
    enabled = false,
    metrics = {},
}

function MyAddon:EnableProfiling()
    self.profiling.enabled = true
end

function MyAddon:ProfileFunction(name, func, ...)
    if not self.profiling.enabled then
        return func(...)
    end

    if C_AddOnProfiler and C_AddOnProfiler.MeasureCall then
        -- Use native profiler (11.1.7+)
        local result = C_AddOnProfiler.MeasureCall(func, ...)

        -- Track our own metrics
        local metrics = self.profiling.metrics[name] or { calls = 0, totalTime = 0, peakTime = 0 }
        metrics.calls = metrics.calls + 1
        metrics.totalTime = metrics.totalTime + result.elapsedMs
        metrics.peakTime = math.max(metrics.peakTime, result.elapsedMs)
        self.profiling.metrics[name] = metrics

        return unpack(result.returnValues)
    else
        -- Fallback to debugprofilestop
        local start = debugprofilestop()
        local results = {func(...)}
        local elapsed = debugprofilestop() - start

        local metrics = self.profiling.metrics[name] or { calls = 0, totalTime = 0, peakTime = 0 }
        metrics.calls = metrics.calls + 1
        metrics.totalTime = metrics.totalTime + elapsed
        metrics.peakTime = math.max(metrics.peakTime, elapsed)
        self.profiling.metrics[name] = metrics

        return unpack(results)
    end
end

function MyAddon:PrintProfilingReport()
    print("=== MyAddon Profiling Report ===")
    for name, metrics in pairs(self.profiling.metrics) do
        local avgTime = metrics.totalTime / metrics.calls
        print(string.format(
            "%s: %d calls, %.3fms avg, %.3fms peak",
            name, metrics.calls, avgTime, metrics.peakTime
        ))
    end
end
```

### Pattern: Comparing Addon Performance

```lua
-- Print performance comparison of all loaded addons
function PrintAddonPerformance()
    local addons = {}

    for i = 1, C_AddOns.GetNumAddOns() do
        local name = C_AddOns.GetAddOnInfo(i)
        if C_AddOns.IsAddOnLoaded(i) then
            local avgTime = C_AddOnProfiler.GetAddOnMetric(
                name,
                Enum.AddOnProfilerMetric.SessionAverageTime
            )
            if avgTime and avgTime > 0 then
                table.insert(addons, { name = name, time = avgTime })
            end
        end
    end

    -- Sort by time (descending)
    table.sort(addons, function(a, b) return a.time > b.time end)

    print("=== Addon Performance (Session Average) ===")
    for i, addon in ipairs(addons) do
        print(string.format("%d. %s: %.3fms", i, addon.name, addon.time))
        if i >= 10 then break end  -- Top 10
    end
end
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
## Interface: 120000
## Title: ElvUI
## SavedVariables: ElvDB, ElvPrivateDB

Core\init.lua
Core\Modules\Load_Modules.xml
```

**Options TOC:**
```
## Interface: 120000
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

## Encounter Integration (12.0.0)

### Overview

WoW 12.0.0 introduces official APIs for encounter timeline and warning systems, providing addon developers with structured access to boss encounter data.

### Encounter Timeline API

```lua
-- Check if encounter timeline is available
if C_EncounterTimeline then
    -- Get list of upcoming events in current encounter
    local events = C_EncounterTimeline.GetEventList()

    for _, eventID in ipairs(events) do
        -- Get detailed info about each event
        local eventInfo = C_EncounterTimeline.GetEventInfo(eventID)
        -- eventInfo contains:
        -- .name        - Event name (e.g., "Decimating Breath")
        -- .spellID     - Associated spell ID
        -- .icon        - Texture path
        -- .duration    - Duration of the effect
        -- .isImportant - Whether this is a critical mechanic

        -- Get time until event fires
        local timeRemaining = C_EncounterTimeline.GetEventTimeRemaining(eventID)
        if timeRemaining then
            print(string.format("%s in %.1fs", eventInfo.name, timeRemaining))
        end
    end
end
```

### Adding Script Events

```lua
-- Add custom events to the timeline (for addon-provided mechanics)
local eventID = C_EncounterTimeline.AddScriptEvent({
    name = "Custom Mechanic",
    time = GetTime() + 30,  -- 30 seconds from now
    duration = 5,
    spellID = 12345,        -- Optional: associated spell
    icon = "Interface\\Icons\\Spell_Fire_Fire",
    priority = 1,           -- Higher = more important
})

-- Remove a custom event
C_EncounterTimeline.RemoveScriptEvent(eventID)
```

### Encounter Warnings API

```lua
-- Check if encounter warnings are enabled
if C_EncounterWarnings.IsFeatureEnabled() then
    -- Play warning sound based on severity
    C_EncounterWarnings.PlaySound(Enum.EncounterWarningSeverity.Low)
    C_EncounterWarnings.PlaySound(Enum.EncounterWarningSeverity.Medium)
    C_EncounterWarnings.PlaySound(Enum.EncounterWarningSeverity.High)
    C_EncounterWarnings.PlaySound(Enum.EncounterWarningSeverity.Critical)
end
```

### Encounter State API

```lua
-- Check if currently in an encounter
if C_InstanceEncounter.IsEncounterInProgress() then
    -- We're fighting a boss
end

-- Pattern: Encounter-aware addon
local frame = CreateFrame("Frame")
frame:RegisterEvent("ENCOUNTER_START")
frame:RegisterEvent("ENCOUNTER_END")

frame:SetScript("OnEvent", function(self, event, encounterID, encounterName, difficultyID, groupSize)
    if event == "ENCOUNTER_START" then
        MyAddon:OnEncounterStart(encounterID, encounterName, difficultyID)
    elseif event == "ENCOUNTER_END" then
        local _, _, _, _, success = ...
        MyAddon:OnEncounterEnd(encounterID, success == 1)
    end
end)

function MyAddon:OnEncounterStart(encounterID, encounterName, difficultyID)
    -- Enable encounter-specific features
    self:StartEncounterTracking()

    -- Check for secret restrictions
    if C_Secrets.HasSecretRestrictions() then
        self:EnableSecretSafeMode()
    end
end

function MyAddon:OnEncounterEnd(encounterID, success)
    self:StopEncounterTracking()
    self:DisableSecretSafeMode()

    if success then
        self:RecordKill(encounterID)
    end
end
```

---

## Damage Meter Integration (12.0.0)

### Overview

WoW 12.0.0 introduces an official damage meter API (`C_DamageMeter`) that provides combat statistics. This allows addons to access official damage/healing data without parsing combat logs.

### Checking Availability

```lua
-- Check if damage meter API is available
if C_DamageMeter and C_DamageMeter.IsDamageMeterAvailable() then
    -- Feature is enabled for this content
end
```

### Getting Combat Sessions

```lua
-- Get all available combat sessions
local sessions = C_DamageMeter.GetAvailableCombatSessions()

for _, sessionInfo in ipairs(sessions) do
    print(string.format("Session: %s, Duration: %.1fs",
        sessionInfo.name,
        sessionInfo.duration
    ))
end

-- Get session by type
local currentSession = C_DamageMeter.GetCombatSessionFromType(
    Enum.DamageMeterSessionType.CurrentFight
)

local lastBossSession = C_DamageMeter.GetCombatSessionFromType(
    Enum.DamageMeterSessionType.LastBoss
)

local dungeonSession = C_DamageMeter.GetCombatSessionFromType(
    Enum.DamageMeterSessionType.Dungeon
)
```

### Pattern: Damage Meter Display

```lua
local DamageMeterAddon = {}

function DamageMeterAddon:UpdateDisplay()
    if not C_DamageMeter or not C_DamageMeter.IsDamageMeterAvailable() then
        self:ShowUnavailableMessage()
        return
    end

    local session = C_DamageMeter.GetCombatSessionFromType(
        Enum.DamageMeterSessionType.CurrentFight
    )

    if not session then
        self:ClearDisplay()
        return
    end

    -- Get damage data for all participants
    local damageData = C_DamageMeter.GetSessionDamageData(session.sessionID)

    -- Sort by damage done
    table.sort(damageData, function(a, b)
        return a.totalDamage > b.totalDamage
    end)

    -- Update UI
    for i, data in ipairs(damageData) do
        if i > self.maxRows then break end

        local dps = data.totalDamage / session.duration
        self:SetRowData(i, {
            name = data.playerName,
            damage = data.totalDamage,
            dps = dps,
            percent = (data.totalDamage / damageData[1].totalDamage) * 100,
        })
    end
end
```

### Listening for Updates

```lua
local frame = CreateFrame("Frame")
frame:RegisterEvent("DAMAGE_METER_DATA_UPDATED")
frame:RegisterEvent("DAMAGE_METER_SESSION_STARTED")
frame:RegisterEvent("DAMAGE_METER_SESSION_ENDED")

frame:SetScript("OnEvent", function(self, event, ...)
    if event == "DAMAGE_METER_DATA_UPDATED" then
        local sessionID = ...
        MyDamageMeter:OnDataUpdated(sessionID)

    elseif event == "DAMAGE_METER_SESSION_STARTED" then
        local sessionID, sessionType = ...
        MyDamageMeter:OnSessionStarted(sessionID, sessionType)

    elseif event == "DAMAGE_METER_SESSION_ENDED" then
        local sessionID, sessionType = ...
        MyDamageMeter:OnSessionEnded(sessionID, sessionType)
    end
end)
```

---

## Performance Optimizations (11.1.7+)

### Preallocated Tables (11.1.7+)

```lua
-- Create a table with preallocated space
-- More efficient when you know the approximate size

-- Old way (causes multiple reallocations)
local items = {}
for i = 1, 1000 do
    items[i] = GetItemInfo(i)
end

-- New way (single allocation)
local items = table.create(1000)  -- Preallocate 1000 slots
for i = 1, 1000 do
    items[i] = GetItemInfo(i)
end

-- Also works for hash tables with expected key count
local lookup = table.create(0, 500)  -- 0 array, 500 hash slots
```

### Table Counting (11.2.5+)

```lua
-- Count table entries efficiently

-- Old way (manual iteration)
local function countTable(t)
    local count = 0
    for _ in pairs(t) do
        count = count + 1
    end
    return count
end

-- New way (native function, much faster)
local count = table.count(myTable)

-- Works for both array and hash portions
local totalEntries = table.count(mixedTable)
```

### Native Compression (C_EncodingUtil)

For data serialization and compression, prefer the native `C_EncodingUtil` over Lua-based libraries like LibSerialize:

```lua
-- Compress data for storage/transmission
local originalData = { ... }  -- Your data
local serialized = -- serialize your data to string first

-- Compress using native API (faster than LibCompress)
local compressed = C_EncodingUtil.CompressData(serialized)

-- Decompress
local decompressed = C_EncodingUtil.DecompressData(compressed)

-- Base64 encoding (for text-safe storage)
local encoded = C_EncodingUtil.EncodeToBase64(compressed)
local decoded = C_EncodingUtil.DecodeFromBase64(encoded)
```

### Pattern: Optimized Data Processing

```lua
local MyAddon = {}

function MyAddon:ProcessLargeDataset(data)
    local numItems = #data

    -- Preallocate result table
    local results = table.create(numItems)

    -- Process in batches to avoid hitching
    local BATCH_SIZE = 100
    local processed = 0

    local function ProcessBatch()
        local batchEnd = math.min(processed + BATCH_SIZE, numItems)

        for i = processed + 1, batchEnd do
            results[i] = self:ProcessItem(data[i])
        end

        processed = batchEnd

        if processed < numItems then
            -- Schedule next batch
            C_Timer.After(0, ProcessBatch)
        else
            -- Done processing
            self:OnProcessingComplete(results)
        end
    end

    ProcessBatch()
end

-- Use table.count for statistics
function MyAddon:GetStatistics()
    return {
        totalItems = table.count(self.items),
        cachedQueries = table.count(self.cache),
        activeTimers = table.count(self.timers),
    }
end
```

### String Concatenation with Secrets

When building strings that might include secret values:

```lua
-- Traditional concatenation fails with secrets
local msg = "Player " .. name .. " dealt " .. damage .. " damage"
-- If 'damage' is a secret, this errors!

-- Safe approach using string.concat
local msg = string.concat("Player ", name, " dealt ", damage, " damage")
-- Works even if damage is a secret value
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
local MIN_TOC = 120000;  -- Midnight expansion

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

1. ✅ **Secret values system** - Handle protected combat data (12.0.0)
2. ✅ **Event bucketing** - Batch rapid events for performance
3. ✅ **Triple-tier profiles** - Private/Profile/Global separation
4. ✅ **Cross-client compatibility** - Support multiple WoW versions
5. ✅ **Data migration** - Version-based upgrades without data loss
6. ✅ **API versioning** - Deprecated function forwarding
7. ✅ **Screen-aware positioning** - Dynamic anchor calculation
8. ✅ **Performance profiling** - Built-in CPU/memory tracking (C_AddOnProfiler)
9. ✅ **Parser sandboxing** - Safe user expression evaluation
10. ✅ **Multi-addon architecture** - Core + LoadOnDemand options
11. ✅ **Smart caching** - Multi-layer with invalidation
12. ✅ **Encounter integration** - Official timeline/warning APIs (12.0.0)
13. ✅ **Damage meter API** - Official combat statistics (12.0.0)

### When to Use:

- **Combat addons** - Use secret values system and restriction handlers
- **Large datasets** - Use caching, bucketing, and table.create()
- **Multi-version support** - Use TOC checks and wrappers
- **Complex settings** - Use triple-tier profiles
- **Public API** - Use versioning and deprecation
- **User expressions** - Use sandboxing
- **Large UI** - Use multi-addon split
- **Performance issues** - Use C_AddOnProfiler and optimization
- **Boss timers** - Use C_EncounterTimeline APIs
- **Damage meters** - Use C_DamageMeter APIs

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

**Version:** 2.0 - Based on WoW 12.0.0 (Midnight)
**Last Updated:** 2026-01-20
**Source Analysis:** ArkInventory, ElvUI, ZygorGuidesViewer, Blizzard API Documentation

<!-- CLAUDE_SKIP_END -->
