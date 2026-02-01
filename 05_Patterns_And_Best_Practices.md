# WoW Addon Development - Patterns and Best Practices

## Table of Contents
1. [Namespace and Code Organization](#namespace-and-code-organization)
2. [Mixin Patterns](#mixin-patterns)
3. [Event-Driven Architecture](#event-driven-architecture)
4. [State Management Patterns](#state-management-patterns)
5. [Table Manipulation Patterns](#table-manipulation-patterns)
6. [Iterator and Enumeration Patterns](#iterator-and-enumeration-patterns)
7. [Performance Optimization](#performance-optimization)
8. [Error Handling and Validation](#error-handling-and-validation)
9. [API Wrapper Patterns](#api-wrapper-patterns)
10. [API Migration Patterns](#api-migration-patterns)
11. [Secret Values Handling (12.0.0+)](#secret-values-handling-1200)
12. [Encoding and Serialization (11.1.5+)](#encoding-and-serialization-1115)
13. [Inter-Addon Communication (11.1.7+)](#inter-addon-communication-1117)
14. [Housing Addon Patterns](#housing-addon-patterns)
15. [Color Management (11.1.5+)](#color-management-1115)
16. [Common Lua Idioms](#common-lua-idioms)

---

## Namespace and Code Organization

### Global Namespace Management

**Pattern: Single Global Table**
```lua
-- Create addon namespace
MyAddon = {};
MyAddon.Config = {};
MyAddon.DB = {};
MyAddon.Modules = {};

-- All addon code lives in this namespace
function MyAddon:Initialize()
    self.initialized = true;
end
```

**Pattern: C_* Style Namespace (Modern)**
```lua
-- Blizzard-style namespace for API functions
MyAddon_Functions = {};

function MyAddon_Functions.GetPlayerData()
    -- Implementation
end

function MyAddon_Functions.SetOption(key, value)
    -- Implementation
end
```

### Module Organization

**Pattern: Module Registration**
```lua
-- Core.lua
MyAddon.Modules = {};

function MyAddon:RegisterModule(name, module)
    self.Modules[name] = module;
end

-- Module file
local InventoryModule = {};

function InventoryModule:OnLoad()
    -- Initialize module
end

function InventoryModule:OnEnable()
    -- Enable module functionality
end

MyAddon:RegisterModule("Inventory", InventoryModule);
```

### Local vs Global Variables

**Best Practice:**
```lua
-- GOOD: Use locals for frequently accessed globals
local pairs, ipairs = pairs, ipairs;
local tinsert, tremove = table.insert, table.remove;
local UnitName, UnitClass = UnitName, UnitClass;

function MyAddon:ProcessPlayers()
    for i = 1, 40 do
        -- Uses local reference (faster)
        local name = UnitName("raid" .. i);
    end
end

-- AVOID: Direct global access in loops
function MyAddon:ProcessPlayersS

low()
    for i = 1, 40 do
        -- Slower: global table lookup every iteration
        local name = _G.UnitName("raid" .. i);
    end
end
```

---

## Mixin Patterns

### Basic Mixin Creation

**Source:** Common pattern throughout Blizzard addons

```lua
-- Define mixin table
MyAddonFrameMixin = {};

function MyAddonFrameMixin:OnLoad()
    self:RegisterEvent("PLAYER_LOGIN");
    self.data = {};
end

function MyAddonFrameMixin:OnEvent(event, ...)
    if event == "PLAYER_LOGIN" then
        self:Initialize();
    end
end

function MyAddonFrameMixin:Initialize()
    print("Initialized");
end
```

**XML Binding:**
```xml
<Frame name="MyAddonFrame" mixin="MyAddonFrameMixin">
    <Scripts>
        <OnLoad method="OnLoad"/>
        <OnEvent method="OnEvent"/>
    </Scripts>
</Frame>
```

### Composition with CreateFromMixins

**Source:** `Blizzard_ActionBar\Shared\StatusTrackingManager.lua`

```lua
-- Combine multiple mixins
StatusTrackingManagerMixin = CreateFromMixins(CallbackRegistryMixin);

StatusTrackingManagerMixin:GenerateCallbackEvents({
    "OnBarSelected",
    "OnBarVisibilityChanged",
});

function StatusTrackingManagerMixin:Init()
    -- Call parent mixin init
    CallbackRegistryMixin.OnLoad(self);

    -- Own initialization
    self.bars = {};
end
```

**Benefits:**
- Methods from all mixins are merged into one table
- Later mixins override earlier ones
- Enables true composition over inheritance

### Factory Pattern with Mixins

```lua
function CreateMyWidget(parent)
    local widget = CreateFromMixins(MyWidgetMixin, CallbackRegistryMixin);
    widget:Init(parent);
    return widget;
end
```

### Mixin Naming Conventions

**Standard Conventions:**
- `MixinName` - The mixin table (e.g., `MyAddonFrameMixin`)
- `MixinName:OnLoad()` - Initialization method
- `MixinName:OnShow()` - Show handler
- `MixinName_Intrinsic` - Framework internals (rare, Blizzard only)
- Private methods use lowercase: `mixinName:privateMethod()`

---

## Event-Driven Architecture

### Event Utility Functions

**Source:** `Blizzard_SharedXML\EventUtil.lua`

**Pattern: Continue After All Events**
```lua
-- Wait for multiple events before executing callback
EventUtil.ContinueAfterAllEvents(function()
    print("Both events received!");
end, "PLAYER_LOGIN", "VARIABLES_LOADED");
```

**Pattern: Register Once**
```lua
-- Execute callback once when event fires, then unregister
EventUtil.RegisterOnceFrameEventAndCallback("PLAYER_ENTERING_WORLD", function()
    print("Player entered world");
end);
```

**Pattern: Continue On Addon Loaded**
```lua
-- Wait for specific addon to load
EventUtil.ContinueOnAddOnLoaded("Blizzard_Communities", function()
    print("Communities addon loaded");
end);
```

### Callback Handle Container

**Source:** `Blizzard_SharedXML\EventUtil.lua`

```lua
-- Container to manage multiple event registrations
local handles = EventUtil.CreateCallbackHandleContainer();

-- Register multiple callbacks
handles:RegisterCallback(EventRegistry, "SomeEvent", function()
    print("Event 1");
end);

handles:RegisterCallback(EventRegistry, "AnotherEvent", function()
    print("Event 2");
end);

-- Unregister all at once
handles:Unregister();
```

### CallbackRegistry Pattern

**Common Pattern:**
```lua
MyAddonMixin = CreateFromMixins(CallbackRegistryMixin);

-- Generate events this mixin can trigger
MyAddonMixin:GenerateCallbackEvents({
    "OnDataLoaded",
    "OnSettingsChanged",
    "OnError",
});

function MyAddonMixin:LoadData()
    -- Load data...

    -- Trigger callback
    self:TriggerEvent("OnDataLoaded", data);
end

-- Usage
myAddon:RegisterCallback("OnDataLoaded", function(owner, data)
    print("Data loaded:", data);
end, self);
```

### Callback-Based Event Registration (12.0.0+)

**New Pattern:** Standalone callback registration without frames

```lua
-- Modern callback pattern (12.0.0+)
-- Register callback for any event
RegisterEventCallback("PLAYER_LOGIN", function(...)
    print("Player logged in!");
end);

-- With identifier for later unregistration
local callbackID = RegisterEventCallback("PLAYER_ENTERING_WORLD", function(...)
    print("Entered world");
end);

-- Unregister when done
UnregisterEventCallback("PLAYER_ENTERING_WORLD", callbackID);

-- Unit-specific callbacks (more efficient for unit events)
RegisterUnitEventCallback("player", "UNIT_HEALTH", function(unit, ...)
    local health = UnitHealth(unit);
    local maxHealth = UnitHealthMax(unit);
    print(format("Player health: %d/%d", health, maxHealth));
end);

-- Multiple units
RegisterUnitEventCallback("target", "UNIT_AURA", function(unit, updateInfo)
    -- Handle target aura changes
end);
```

**Benefits of Callback Pattern:**
- No frame creation required
- Cleaner syntax for simple event handling
- Automatic cleanup options
- Better for modular code organization

---

## State Management Patterns

### Dirty Tracking Pattern

**Source:** `Blizzard_SharedXML\LayoutFrame.lua`

```lua
BaseLayoutMixin = {};

function BaseLayoutMixin:MarkDirty()
    if self.dirty then
        return;  -- Already marked
    end

    self.dirty = true;

    -- Only set OnUpdate while dirty (performance optimization)
    self:SetScript("OnUpdate", self.OnUpdate);

    -- Propagate to parent
    local parent = self:GetParent();
    while parent do
        if parent.MarkDirty then
            parent:MarkDirty();
            break;
        end
        parent = parent:GetParent();
    end
end

function BaseLayoutMixin:OnUpdate()
    if not self:IsDirty() then
        return;
    end

    self:Layout();
end

function BaseLayoutMixin:Layout()
    -- Perform layout...

    -- Clear dirty flag and remove OnUpdate
    self.dirty = false;
    self:SetScript("OnUpdate", nil);
end
```

**Key Benefits:**
- OnUpdate only active when needed
- Batches multiple changes into one update
- Propagates up hierarchy

### Caching Pattern

```lua
MyAddonMixin = {};

function MyAddonMixin:GetPlayerInfo()
    -- Check cache first
    if self.cachedPlayerInfo then
        return self.cachedPlayerInfo;
    end

    -- Expensive operation
    local info = {
        name = UnitName("player"),
        class = UnitClass("player"),
        level = UnitLevel("player"),
    };

    -- Cache result
    self.cachedPlayerInfo = info;
    return info;
end

function MyAddonMixin:InvalidateCache()
    self.cachedPlayerInfo = nil;
end
```

### State Machine Pattern

```lua
local States = {
    IDLE = 1,
    LOADING = 2,
    READY = 3,
    ERROR = 4,
};

MyAddonMixin = {};

function MyAddonMixin:SetState(newState)
    if self.state == newState then
        return;
    end

    local oldState = self.state;
    self.state = newState;

    -- State transition handlers
    if newState == States.LOADING then
        self:OnEnterLoading();
    elseif newState == States.READY then
        self:OnEnterReady();
    elseif newState == States.ERROR then
        self:OnEnterError();
    end

    self:TriggerEvent("OnStateChanged", oldState, newState);
end

function MyAddonMixin:IsReady()
    return self.state == States.READY;
end
```

---

## Table Manipulation Patterns

### Table Builder Pattern

**Source:** `Blizzard_SharedXML\TableBuilder.lua`

```lua
-- Separation of concerns: data vs UI
TableBuilderElementMixin = {};

function TableBuilderElementMixin:Init(...)
    -- Initialize element
end

function TableBuilderElementMixin:Populate(rowData, dataProviderKey)
    -- Populate from data
end

-- Cell extends element
TableBuilderCellMixin = CreateFromMixins(TableBuilderElementMixin);

function TableBuilderCellMixin:OnLineEnter()
    -- Handle mouse enter
end

-- Row extends element
TableBuilderRowMixin = CreateFromMixins(TableBuilderElementMixin);

function TableBuilderRowMixin:OnEnter()
    self:OnLineEnter();
    for i, cell in ipairs(self.cells) do
        cell:OnLineEnter();
    end
end
```

### Safe Table Access

```lua
-- Safe pack that handles nil values
local function SafePack(...)
    return {n = select("#", ...), ...};
end

-- Safe unpack
local function SafeUnpack(tbl)
    return unpack(tbl, 1, tbl.n);
end

-- Usage
local args = SafePack(nil, "hello", nil, "world");
print(SafeUnpack(args));  -- Correctly handles nils
```

### Table Copying

```lua
-- Shallow copy
local function CopyTable(source)
    local copy = {};
    for k, v in pairs(source) do
        copy[k] = v;
    end
    return copy;
end

-- Deep copy
local function DeepCopyTable(source)
    if type(source) ~= "table" then
        return source;
    end

    local copy = {};
    for k, v in pairs(source) do
        copy[k] = DeepCopyTable(v);
    end
    return copy;
end
```

### Table Find and Filter

```lua
-- Find first matching element
local function FindInTable(tbl, predicate)
    for i, value in ipairs(tbl) do
        if predicate(value) then
            return value, i;
        end
    end
    return nil;
end

-- Filter table
local function FilterTable(tbl, predicate)
    local result = {};
    for i, value in ipairs(tbl) do
        if predicate(value) then
            table.insert(result, value);
        end
    end
    return result;
end

-- Usage
local players = {
    {name = "Alice", level = 80},
    {name = "Bob", level = 75},
    {name = "Charlie", level = 80},
};

local maxLevelPlayers = FilterTable(players, function(player)
    return player.level == 80;  -- Max level in Midnight (12.0.0)
end);
```

---

## Iterator and Enumeration Patterns

### Standard Iterators

```lua
-- Array iteration
for index, value in ipairs(myArray) do
    print(index, value);
end

-- Table iteration (unordered)
for key, value in pairs(myTable) do
    print(key, value);
end

-- Reverse iteration
for i = #myArray, 1, -1 do
    local value = myArray[i];
    print(i, value);
end
```

### Custom Iterators

```lua
-- Enumerate with range
local function CreateTableEnumerator(tbl, indexBegin, indexEnd)
    indexBegin = indexBegin or 1;
    indexEnd = indexEnd or #tbl;

    local index = indexBegin - 1;
    return function()
        index = index + 1;
        if index <= indexEnd then
            return index, tbl[index];
        end
    end;
end

-- Usage
for index, value in CreateTableEnumerator(myTable, 5, 10) do
    print(index, value);
end
```

### Filtered Iterator

```lua
local function FilteredPairs(tbl, predicate)
    local key, value;
    return function()
        repeat
            key, value = next(tbl, key);
        until key == nil or predicate(key, value);
        return key, value;
    end;
end

-- Usage: Only iterate over numeric values
for key, value in FilteredPairs(myTable, function(k, v)
    return type(v) == "number";
end) do
    print(key, value);
end
```

---

## Performance Optimization

### Localize Frequently Used Globals

```lua
-- At file scope
local pairs, ipairs, type = pairs, ipairs, type;
local tinsert, tremove, wipe = table.insert, table.remove, wipe;
local floor, ceil, max, min = math.floor, math.ceil, math.max, math.min;
local format, gsub, match = string.format, string.gsub, string.match;

-- WoW API
local UnitName, UnitGUID = UnitName, UnitGUID;
local GetTime, GetTimePreciseSec = GetTime, GetTimePreciseSec;
```

### Table Preallocation (11.1.7+)

```lua
-- GOOD: Preallocate table size for known quantities
local myTable = table.create(100);  -- Preallocate array for 100 elements
for i = 1, 100 do
    myTable[i] = i * 2;
end

-- Use for known-size arrays to avoid reallocation
local function ProcessRaidMembers()
    local members = table.create(40);  -- Max raid size
    for i = 1, GetNumGroupMembers() do
        members[i] = UnitName("raid" .. i);
    end
    return members;
end

-- Note: table.create() only helps with array parts, not hash parts
```

### Addon Profiling (11.1.7+)

```lua
-- Use C_AddOnProfiler for accurate performance measurement
local function ProfileExpensiveOperation()
    local startTime = C_AddOnProfiler.GetCurrentTime();

    -- Your code here
    DoExpensiveWork();

    local endTime = C_AddOnProfiler.GetCurrentTime();
    local elapsed = endTime - startTime;
    print(format("Operation took %.3f ms", elapsed * 1000));
end

-- Measure specific function calls
local result, duration = C_AddOnProfiler.MeasureCall(function()
    return DoSomethingExpensive();
end);
print(format("Function returned %s in %.3f ms", tostring(result), duration * 1000));

-- Get addon memory usage
local memoryKB = C_AddOnProfiler.GetAddOnMemoryUsage("MyAddon");
print(format("Memory usage: %.2f KB", memoryKB));

-- Profile info available in the addon profiler UI (/console enableAddOnCPUProfiling 1)
```

### Avoid OnUpdate When Possible

```lua
-- BAD: Constant OnUpdate
frame:SetScript("OnUpdate", function(self, elapsed)
    self.timer = (self.timer or 0) + elapsed;
    if self.timer >= 1 then
        self:Update();
        self.timer = 0;
    end
end);

-- GOOD: Use C_Timer instead
C_Timer.NewTicker(1, function()
    frame:Update();
end);

-- BETTER: Event-driven
frame:RegisterEvent("PLAYER_REGEN_ENABLED");
frame:SetScript("OnEvent", function(self, event)
    if event == "PLAYER_REGEN_ENABLED" then
        self:Update();
    end
end);
```

### Table Recycling

```lua
-- Reuse tables instead of creating new ones
local tableCache = {};

local function AcquireTable()
    return tremove(tableCache) or {};
end

local function ReleaseTable(tbl)
    wipe(tbl);
    tinsert(tableCache, tbl);
end

-- Usage
local myTable = AcquireTable();
myTable.foo = "bar";
-- ... use table ...
ReleaseTable(myTable);  -- Return to pool
```

### String Concatenation

```lua
-- BAD: Creates many intermediate strings
local str = "";
for i = 1, 1000 do
    str = str .. tostring(i);  -- Very slow!
end

-- GOOD: Use table concatenation
local parts = {};
for i = 1, 1000 do
    tinsert(parts, tostring(i));
end
local str = table.concat(parts);

-- BETTER: Use string.format for known patterns
local str = format("%d items, %s quality", count, quality);
```

### Frame Pool Pattern

```lua
-- Create pool
local pool = CreateFramePool("Button", parent, "MyButtonTemplate");

-- Acquire from pool
local button = pool:Acquire();
button:SetText("Hello");
button:SetPoint("CENTER");
button:Show();

-- Release when done
pool:Release(button);

-- Release all
pool:ReleaseAll();
```

---

## Error Handling and Validation

### Assert for Validation

```lua
function MyAddon:SetData(data)
    assert(type(data) == "table", "Data must be a table");
    assert(data.name, "Data must have a name field");
    assert(data.level and data.level > 0, "Invalid level");

    self.data = data;
end
```

### Protected Calls

```lua
-- pcall for error handling
local success, result = pcall(function()
    -- Code that might error
    return SomeRiskyOperation();
end);

if success then
    print("Result:", result);
else
    print("Error:", result);  -- result is error message
end

-- xpcall with error handler
local function errorHandler(err)
    print("Error occurred:", err);
    print(debugstack());
    return err;
end

local success, result = xpcall(function()
    return SomeRiskyOperation();
end, errorHandler);
```

### Validation Pattern

```lua
local function ValidateConfig(config)
    local errors = {};

    if not config.name or config.name == "" then
        tinsert(errors, "Name is required");
    end

    -- Max level is 80 in Midnight (12.0.0)
    local MAX_LEVEL = GetMaxLevelForPlayerExpansion and GetMaxLevelForPlayerExpansion() or 80;
    if not config.level or config.level < 1 or config.level > MAX_LEVEL then
        tinsert(errors, "Level must be between 1 and " .. MAX_LEVEL);
    end

    if #errors > 0 then
        return false, errors;
    end

    return true;
end

-- Usage
local valid, errors = ValidateConfig(userConfig);
if not valid then
    for _, error in ipairs(errors) do
        print("Validation error:", error);
    end
    return;
end
```

---

## API Wrapper Patterns

### Nil-Safe Wrappers

```lua
-- Wrap API calls that may return nil
function MyAddon:GetItemInfo(itemID)
    if not itemID then
        return nil;
    end

    -- GetItemInfo may return nil if not cached
    local name, link, quality, level = GetItemInfo(itemID);

    if not name then
        -- Item not in cache, queue load
        C_Item.RequestLoadItemDataByID(itemID);
        return nil;
    end

    return {
        name = name,
        link = link,
        quality = quality,
        level = level,
    };
end
```

### Cached API Calls

```lua
MyAddon.Cache = {};

function MyAddon:GetCachedItemInfo(itemID)
    -- Check cache
    if self.Cache[itemID] then
        return self.Cache[itemID];
    end

    -- Call API
    local info = self:GetItemInfo(itemID);

    if info then
        -- Cache result
        self.Cache[itemID] = info;
    end

    return info;
end
```

### Event-Driven API Pattern

```lua
-- Some APIs require waiting for events
MyAddon.PendingQueries = {};

function MyAddon:QueryItemInfo(itemID, callback)
    -- Check cache first
    local cached = self:GetCachedItemInfo(itemID);
    if cached then
        callback(cached);
        return;
    end

    -- Queue callback
    if not self.PendingQueries[itemID] then
        self.PendingQueries[itemID] = {};
    end
    tinsert(self.PendingQueries[itemID], callback);

    -- Request load
    C_Item.RequestLoadItemDataByID(itemID);
end

-- Event handler
function MyAddon:OnItemDataLoaded(itemID)
    local callbacks = self.PendingQueries[itemID];
    if not callbacks then
        return;
    end

    -- Get item info
    local info = self:GetItemInfo(itemID);

    -- Execute all pending callbacks
    for _, callback in ipairs(callbacks) do
        callback(info);
    end

    -- Clear pending
    self.PendingQueries[itemID] = nil;
end
```

---

## API Migration Patterns

### Function Existence Checking

```lua
-- Always check if APIs exist before using them
local function SafeCallAPI(apiNamespace, funcName, ...)
    if apiNamespace and apiNamespace[funcName] then
        return apiNamespace[funcName](...);
    end
    return nil;
end

-- Usage
local result = SafeCallAPI(C_Item, "GetItemInfo", itemID);

-- Direct existence check pattern
if C_ActionBar and C_ActionBar.GetActionBarState then
    local state = C_ActionBar.GetActionBarState();
end
```

### Version-Safe Wrappers

```lua
-- Create wrappers that work across WoW versions
local function GetActionBarStateCompat()
    -- 12.0.0+ uses C_ActionBar
    if C_ActionBar and C_ActionBar.GetActionBarState then
        return C_ActionBar.GetActionBarState();
    end
    -- Fallback for older versions (if needed)
    return nil;
end

-- Pattern for deprecated functions
local function GetCombatLogEventCompat(...)
    -- Modern API (11.0.0+)
    if C_CombatLog and C_CombatLog.GetCurrentEventInfo then
        return C_CombatLog.GetCurrentEventInfo();
    end
    -- Legacy global (deprecated but may still exist)
    if CombatLogGetCurrentEventInfo then
        return CombatLogGetCurrentEventInfo();
    end
    return nil;
end
```

### Compatibility Shim Pattern

```lua
-- Create namespace if it doesn't exist (for cross-version addons)
MyAddon.Compat = MyAddon.Compat or {};

-- Shim for C_TransmogOutfitInfo (replaces old global functions)
if not C_TransmogOutfitInfo then
    MyAddon.Compat.GetOutfitInfo = function(outfitID)
        -- Fallback implementation using old API
        if GetOutfitInfo then
            return GetOutfitInfo(outfitID);
        end
        return nil;
    end;
else
    MyAddon.Compat.GetOutfitInfo = function(outfitID)
        return C_TransmogOutfitInfo.GetOutfitInfo(outfitID);
    end;
end

-- Shim for C_ItemSocketInfo (replaces old socket globals)
if not C_ItemSocketInfo then
    MyAddon.Compat.GetSocketInfo = function(socketIndex)
        if GetSocketItemInfo then
            return GetSocketItemInfo(socketIndex);
        end
        return nil;
    end;
else
    MyAddon.Compat.GetSocketInfo = function(socketIndex)
        return C_ItemSocketInfo.GetSocketItemInfo(socketIndex);
    end;
end
```

### Interface Version Detection

```lua
-- Check interface version for feature availability
local INTERFACE_VERSION = select(4, GetBuildInfo());

local IS_RETAIL = INTERFACE_VERSION >= 110000;
local IS_MIDNIGHT = INTERFACE_VERSION >= 120000;
local IS_TWW = INTERFACE_VERSION >= 110000 and INTERFACE_VERSION < 120000;

-- Use for conditional feature loading
if IS_MIDNIGHT then
    -- Use 12.0.0+ specific features
    MyAddon:EnableHousingFeatures();
end

-- Feature flags based on API availability
MyAddon.Features = {
    HasSecretValues = C_RestrictedActions ~= nil and C_RestrictedActions.IsAddOnRestrictionActive ~= nil,
    HasEncodingUtil = C_EncodingUtil ~= nil,
    HasAddOnLocalTable = C_AddOns ~= nil and C_AddOns.GetAddOnLocalTable ~= nil,
    HasHousing = C_Housing ~= nil,
};
```

---

## Secret Values Handling (12.0.0+)

### Understanding Secret Values

In 12.0.0 (Midnight), Blizzard introduced "secret values" to prevent addons from automating gameplay during combat. Certain API return values become "secret" and cannot be directly used for decision-making.

### Checking Restriction State

```lua
-- Check if restrictions are currently active
local function AreRestrictionsActive()
    if C_RestrictedActions and C_RestrictedActions.IsAddOnRestrictionActive then
        return C_RestrictedActions.IsAddOnRestrictionActive();
    end
    return false;
end

-- Typical usage
local frame = CreateFrame("Frame");
frame:RegisterEvent("PLAYER_REGEN_DISABLED");
frame:RegisterEvent("PLAYER_REGEN_ENABLED");
frame:SetScript("OnEvent", function(self, event)
    if event == "PLAYER_REGEN_DISABLED" then
        if AreRestrictionsActive() then
            print("Combat restrictions active - some features limited");
        end
    end
end);
```

### Detecting Secret Values

```lua
-- Check if a value is secret
local function HandlePotentiallySecretValue(value)
    if issecretvalue and issecretvalue(value) then
        -- Value is secret, cannot use for automation
        return nil, true;  -- Return nil and flag as secret
    end
    return value, false;
end

-- Usage with unit health
local function GetSafeUnitHealth(unit)
    local health = UnitHealth(unit);
    local actualHealth, isSecret = HandlePotentiallySecretValue(health);

    if isSecret then
        -- Fall back to percentage or visual indicator
        return nil;
    end

    return actualHealth;
end
```

### Scrubbing Secret Values from Tables

```lua
-- Clean tables before processing
local function ProcessDataSafely(dataTable)
    if scrubsecretvalues then
        -- Remove secret values from table (modifies in place)
        scrubsecretvalues(dataTable);
    end

    -- Now safe to process
    for key, value in pairs(dataTable) do
        -- Process cleaned data
    end
end

-- Example: Processing unit info
local function GetSafeUnitInfo(unit)
    local info = {
        health = UnitHealth(unit),
        power = UnitPower(unit),
        name = UnitName(unit),
    };

    if scrubsecretvalues then
        scrubsecretvalues(info);
    end

    return info;
end
```

### Preventing Secret Values on Frames

```lua
-- Mark frames that shouldn't receive secret values
local myFrame = CreateFrame("Frame", "MyAddonFrame", UIParent);

-- Prevent secret values from being passed to this frame's scripts
if myFrame.SetPreventSecretValues then
    myFrame:SetPreventSecretValues(true);
end

-- This is useful for frames that:
-- - Display data to users (information only)
-- - Don't make automated decisions
-- - Want cleaner data handling
```

### Using Curve Objects for UI Display

```lua
-- For health bars and similar UI elements, use Curve objects
local function CreateHealthBar(parent)
    local healthBar = CreateFrame("StatusBar", nil, parent);

    -- Health curve provides smooth animation without exposing exact values
    local healthCurve = CreateCurve();

    healthBar:SetScript("OnUpdate", function(self, elapsed)
        -- Curve automatically handles secret value restrictions
        local displayValue = healthCurve:GetCurrentValue();
        self:SetValue(displayValue);
    end);

    return healthBar, healthCurve;
end

-- ColorCurve for smooth color transitions
local colorCurve = CreateColorCurve();
colorCurve:SetStartColor(1, 0, 0, 1);  -- Red
colorCurve:SetEndColor(0, 1, 0, 1);    -- Green
colorCurve:SetDuration(1.0);

-- Duration objects for timing
local duration = CreateDuration(5.0);  -- 5 second duration
```

### StatusBar Floating-Point Precision

**WARNING:** Never pass large epoch-millisecond timestamps directly to `StatusBar:SetMinMaxValues()` and `SetValue()`. The StatusBar widget internally computes `(value - min) / (max - min)` using floating-point arithmetic. With values like `1,738,412,345,000` (13 significant digits) but only ~7 digits of float precision, the fill fraction gets quantized, causing the bar to jump in discrete increments instead of filling smoothly.

**Always normalize to small ranges:**
```lua
-- WRONG: Raw epoch milliseconds (choppy animation)
castBar:SetMinMaxValues(startTimeMs, endTimeMs)  -- e.g., 1738412345000, 1738412347500
castBar:SetValue(GetTime() * 1000)

-- CORRECT: Normalized to small range (smooth animation)
local durationSec = (endTimeMs - startTimeMs) / 1000
castBar:SetMinMaxValues(0, durationSec)  -- e.g., 0, 2.5
castBar:SetValue(GetTime() - startTimeMs / 1000)

-- BEST (12.0.0+): Use SetTimerDuration (C++ handles everything)
castBar:SetTimerDuration(UnitCastingDuration(unitid))
```

This applies to any StatusBar where the min/max range involves large absolute values with small relative differences. Health bars are typically fine (small values), but cast bars using epoch timestamps are particularly vulnerable.

### Testing Secret Value Handling

```lua
-- CVars for testing (development only)
-- /console secretCombatRestrictionsForced 1  -- Force restrictions on
-- /console secretCombatRestrictionsForced 0  -- Normal behavior

-- Test helper
local function TestSecretValueHandling()
    print("Testing secret value handling...");

    local testValue = UnitHealth("player");

    if issecretvalue and issecretvalue(testValue) then
        print("Health is SECRET - restrictions active");
    else
        print("Health is NORMAL:", testValue);
    end

    -- Test table scrubbing
    local testTable = {health = UnitHealth("player"), name = "Test"};
    print("Before scrub:", testTable.health);

    if scrubsecretvalues then
        scrubsecretvalues(testTable);
    end
    print("After scrub:", testTable.health);
end
```

### Best Practices for Secret Values

1. **Check before using**: Always check `issecretvalue()` before using values for automation
2. **Scrub tables**: Use `scrubsecretvalues()` when processing data from multiple sources
3. **UI vs Logic**: Separate display code from decision-making code
4. **Graceful degradation**: Have fallbacks when values are secret
5. **Test thoroughly**: Use the debug CVars to test both restricted and unrestricted states

---

## Encoding and Serialization (11.1.5+)

### Compression and Encoding

```lua
-- Compress string data
local function CompressData(data)
    if not C_EncodingUtil then
        return data;  -- Fallback: return uncompressed
    end

    local compressed = C_EncodingUtil.CompressString(data);
    return compressed;
end

-- Decompress string data
local function DecompressData(compressed)
    if not C_EncodingUtil then
        return compressed;
    end

    local decompressed = C_EncodingUtil.DecompressString(compressed);
    return decompressed;
end

-- Base64 encoding for transmission
local function EncodeForTransmission(data)
    local compressed = C_EncodingUtil.CompressString(data);
    local base64 = C_EncodingUtil.EncodeBase64(compressed);
    return base64;
end

local function DecodeFromTransmission(encoded)
    local compressed = C_EncodingUtil.DecodeBase64(encoded);
    local data = C_EncodingUtil.DecompressString(compressed);
    return data;
end
```

### JSON Serialization

```lua
-- Serialize table to JSON
local function TableToJSON(tbl)
    if C_EncodingUtil and C_EncodingUtil.SerializeJSON then
        return C_EncodingUtil.SerializeJSON(tbl);
    end
    -- Fallback: manual serialization
    return nil;
end

-- Deserialize JSON to table
local function JSONToTable(jsonString)
    if C_EncodingUtil and C_EncodingUtil.DeserializeJSON then
        return C_EncodingUtil.DeserializeJSON(jsonString);
    end
    return nil;
end

-- Example usage
local playerData = {
    name = UnitName("player"),
    class = select(2, UnitClass("player")),
    level = UnitLevel("player"),
    settings = {
        showMinimap = true,
        scale = 1.0,
    },
};

local json = TableToJSON(playerData);
print("JSON:", json);

local restored = JSONToTable(json);
print("Name:", restored.name);
```

### CBOR Serialization (Binary)

```lua
-- CBOR is more compact than JSON for binary data
local function TableToCBOR(tbl)
    if C_EncodingUtil and C_EncodingUtil.SerializeCBOR then
        return C_EncodingUtil.SerializeCBOR(tbl);
    end
    return nil;
end

local function CBORToTable(cborData)
    if C_EncodingUtil and C_EncodingUtil.DeserializeCBOR then
        return C_EncodingUtil.DeserializeCBOR(cborData);
    end
    return nil;
end

-- Prefer CBOR for:
-- - Addon communication (smaller payloads)
-- - SavedVariables with binary data
-- - Performance-critical serialization

-- Prefer JSON for:
-- - Human-readable exports
-- - External tool compatibility
-- - Debugging
```

### Addon Communication Pattern

```lua
-- Pattern for sending compressed addon messages
local PREFIX = "MyAddon";

local function SendCompressedMessage(data, channel, target)
    local json = C_EncodingUtil.SerializeJSON(data);
    local compressed = C_EncodingUtil.CompressString(json);
    local encoded = C_EncodingUtil.EncodeBase64(compressed);

    C_ChatInfo.SendAddonMessage(PREFIX, encoded, channel, target);
end

local function ReceiveCompressedMessage(prefix, message, channel, sender)
    if prefix ~= PREFIX then return end

    local compressed = C_EncodingUtil.DecodeBase64(message);
    local json = C_EncodingUtil.DecompressString(compressed);
    local data = C_EncodingUtil.DeserializeJSON(json);

    -- Process received data
    MyAddon:HandleReceivedData(data, sender);
end

C_ChatInfo.RegisterAddonMessagePrefix(PREFIX);
-- Register event handler for CHAT_MSG_ADDON
```

---

## Inter-Addon Communication (11.1.7+)

### Exposing Your Addon's API

```lua
-- In your TOC file, add:
-- ## AllowAddOnTableAccess: 1

-- In your main file:
local addonName, ns = ...;

-- Create public API in namespace
ns.API = {};

function ns.API.GetVersion()
    return ns.version;
end

function ns.API.GetPlayerData()
    return ns.playerData;
end

function ns.API.RegisterCallback(event, callback)
    if ns.callbacks[event] then
        tinsert(ns.callbacks[event], callback);
        return true;
    end
    return false;
end

-- Mark what's private (convention)
ns.Internal = {};  -- Other addons shouldn't access this
```

### Accessing Other Addon's API

```lua
-- Access another addon's namespace
local function TryAccessOtherAddon(addonName)
    if not C_AddOns or not C_AddOns.GetAddOnLocalTable then
        return nil;
    end

    local otherNS = C_AddOns.GetAddOnLocalTable(addonName);

    if not otherNS then
        print(addonName .. " not loaded or doesn't allow access");
        return nil;
    end

    if not otherNS.API then
        print(addonName .. " doesn't expose a public API");
        return nil;
    end

    return otherNS.API;
end

-- Example: Integrating with another addon
local function IntegrateWithDetailsAddon()
    local detailsAPI = TryAccessOtherAddon("Details");

    if detailsAPI and detailsAPI.GetCurrentCombat then
        local combat = detailsAPI.GetCurrentCombat();
        -- Use combat data
    end
end
```

### Safe Integration Pattern

```lua
-- Wait for addon to load before accessing
EventUtil.ContinueOnAddOnLoaded("OtherAddon", function()
    local api = C_AddOns.GetAddOnLocalTable("OtherAddon");

    if api and api.API then
        -- Safe to use the API now
        MyAddon.OtherAddonAPI = api.API;
        MyAddon:OnOtherAddonReady();
    end
end);

-- Version checking
local function CheckAPIVersion(api, requiredVersion)
    if not api or not api.GetVersion then
        return false;
    end

    local version = api.GetVersion();
    return version >= requiredVersion;
end
```

---

## Housing Addon Patterns

### Detecting Housing Context

```lua
-- Check if player is in their house
local function IsInHouse()
    if C_Housing and C_Housing.IsInsideHouse then
        return C_Housing.IsInsideHouse();
    end
    return false;
end

-- Check if in edit mode
local function IsInHousingEditMode()
    if C_Housing and C_Housing.IsInEditMode then
        return C_Housing.IsInEditMode();
    end
    return false;
end

-- Get current house info
local function GetCurrentHouseInfo()
    if not C_Housing then return nil end

    return {
        isInside = C_Housing.IsInsideHouse(),
        isOwner = C_Housing.IsOwnHouse and C_Housing.IsOwnHouse(),
        isEditing = C_Housing.IsInEditMode and C_Housing.IsInEditMode(),
    };
end
```

### Housing Event Handling

```lua
local HousingMixin = {};

function HousingMixin:OnLoad()
    self:RegisterEvent("HOUSING_ENTERED");
    self:RegisterEvent("HOUSING_EXITED");
    self:RegisterEvent("HOUSING_EDIT_MODE_CHANGED");
    self:RegisterEvent("HOUSING_FURNITURE_PLACED");
    self:RegisterEvent("HOUSING_FURNITURE_REMOVED");
end

function HousingMixin:OnEvent(event, ...)
    if self[event] then
        self[event](self, ...);
    end
end

function HousingMixin:HOUSING_ENTERED()
    print("Entered housing area");
    self:RefreshHousingUI();
end

function HousingMixin:HOUSING_EXITED()
    print("Left housing area");
    self:HideHousingUI();
end

function HousingMixin:HOUSING_EDIT_MODE_CHANGED(isEditing)
    if isEditing then
        self:EnableEditModeFeatures();
    else
        self:DisableEditModeFeatures();
    end
end

function HousingMixin:HOUSING_FURNITURE_PLACED(furnitureID, x, y, z)
    -- Track placed furniture
    self:OnFurniturePlaced(furnitureID, x, y, z);
end

function HousingMixin:HOUSING_FURNITURE_REMOVED(furnitureID)
    -- Update tracking
    self:OnFurnitureRemoved(furnitureID);
end
```

### Housing-Aware Addon Pattern

```lua
-- Addon that changes behavior in housing
MyAddon = {};

function MyAddon:Initialize()
    -- Check initial state
    self:UpdateHousingState();

    -- Register for housing events
    local frame = CreateFrame("Frame");
    frame:RegisterEvent("HOUSING_ENTERED");
    frame:RegisterEvent("HOUSING_EXITED");
    frame:SetScript("OnEvent", function(_, event)
        self:UpdateHousingState();
    end);
end

function MyAddon:UpdateHousingState()
    self.isInHousing = C_Housing and C_Housing.IsInsideHouse and C_Housing.IsInsideHouse();

    if self.isInHousing then
        self:EnableHousingFeatures();
    else
        self:EnableNormalFeatures();
    end
end

function MyAddon:EnableHousingFeatures()
    -- Show housing-specific UI
    -- Enable furniture placement helpers
    -- Show housing inventory
end

function MyAddon:EnableNormalFeatures()
    -- Standard addon functionality
end
```

---

## Color Management (11.1.5+)

### Using ColorManager for Item Quality

```lua
-- PREFER: Use ColorManager for item quality colors
local function GetItemQualityColor(quality)
    if ColorManager and ColorManager.GetColorDataForItemQuality then
        local colorData = ColorManager.GetColorDataForItemQuality(quality);
        if colorData then
            return colorData:GetRGB();  -- Returns r, g, b
        end
    end

    -- Fallback to constant table
    local color = ITEM_QUALITY_COLORS[quality];
    if color then
        return color.r, color.g, color.b;
    end

    return 1, 1, 1;  -- White default
end

-- AVOID: Hardcoded colors
-- local EPIC_COLOR = {r = 0.639, g = 0.208, b = 0.933};  -- Don't do this
```

### Handling Color Override Events

```lua
-- Colors can be overridden by accessibility settings
local ColorAwareMixin = {};

function ColorAwareMixin:OnLoad()
    self:RegisterEvent("COLOR_OVERRIDE_UPDATED");
    self:UpdateColors();
end

function ColorAwareMixin:OnEvent(event)
    if event == "COLOR_OVERRIDE_UPDATED" then
        self:UpdateColors();
    end
end

function ColorAwareMixin:UpdateColors()
    -- Refresh all color-dependent UI elements
    for _, element in pairs(self.coloredElements) do
        local quality = element.itemQuality;
        local r, g, b = GetItemQualityColor(quality);
        element:SetTextColor(r, g, b);
    end
end
```

### Class Color Handling

```lua
-- Get class color with override support
local function GetSafeClassColor(classToken)
    -- ColorManager respects user overrides
    if ColorManager and ColorManager.GetClassColor then
        local color = ColorManager.GetClassColor(classToken);
        if color then
            return color:GetRGB();
        end
    end

    -- Fallback
    local color = RAID_CLASS_COLORS[classToken];
    if color then
        return color.r, color.g, color.b;
    end

    return 1, 1, 1;
end
```

---

## Common Lua Idioms

### Ternary Operator Alternative

```lua
-- Lua doesn't have ternary, use 'and'/'or'
local value = condition and trueValue or falseValue;

-- But beware: if trueValue is false/nil, this breaks!
local value = someBoolean and false or "default";  -- WRONG: always returns "default"

-- Safe version for all cases:
local value = condition and trueValue or (not condition and falseValue);

-- Or just use if/else
local value;
if condition then
    value = trueValue;
else
    value = falseValue;
end
```

### Default Values

```lua
-- Use 'or' for defaults
local name = userName or "Unknown";
local count = itemCount or 0;

-- Function parameters
function MyAddon:SetOption(key, value, updateUI)
    updateUI = updateUI ~= false;  -- Default to true
    self.options[key] = value;
    if updateUI then
        self:UpdateUI();
    end
end
```

### Memoization

```lua
-- Cache expensive function results
local memoizedFibonacci = (function()
    local cache = {};
    return function(n)
        if cache[n] then
            return cache[n];
        end

        local result;
        if n <= 1 then
            result = n;
        else
            result = memoizedFibonacci(n - 1) + memoizedFibonacci(n - 2);
        end

        cache[n] = result;
        return result;
    end;
end)();
```

### Method Call Syntax

```lua
-- Colon syntax automatically passes self
MyAddonMixin = {};

function MyAddonMixin:DoSomething(arg)
    print(self, arg);
end

-- These are equivalent:
myAddon:DoSomething("hello");
MyAddonMixin.DoSomething(myAddon, "hello");
```

### Varargs Handling

```lua
-- Get argument count (includes nils)
local function CountArgs(...)
    return select("#", ...);
end

-- Get nth argument
local function GetArg(n, ...)
    return select(n, ...);
end

-- Get all args starting from n
local function GetArgsFrom(n, ...)
    return select(n, ...);
end

-- Usage
CountArgs(1, nil, 3);  -- Returns 3
GetArg(2, "a", "b", "c");  -- Returns "b"
GetArgsFrom(2, "a", "b", "c");  -- Returns "b", "c"
```

---

## Sort Utilities

**Source:** `Blizzard_SharedXML\SortUtil.lua`

```lua
-- Numeric comparison
function SortUtil.CompareNumeric(lhs, rhs)
    return Sign(lhs - rhs);
end

-- UTF-8 case-insensitive string comparison
function SortUtil.CompareUtf8i(lhs, rhs)
    return Sign(strcmputf8i(lhs, rhs));
end

-- Create sort manager
local sortManager = SortUtil.CreateSortManager();

-- Add comparators
sortManager:InsertComparator("name", function(a, b)
    return SortUtil.CompareUtf8i(a.name, b.name);
end);

sortManager:InsertComparator("level", function(a, b)
    return SortUtil.CompareNumeric(a.level, b.level);
end);

-- Set default comparator (required)
sortManager:SetDefaultComparator(function(a, b)
    return a.id < b.id;
end);

-- Set sort order function
sortManager:SetSortOrderFunc(function()
    return currentSortOrder;
end);

-- Get comparator for table.sort
local comparator = sortManager:CreateComparator();
table.sort(myTable, comparator);

-- Toggle sort direction
sortManager:ToggleSortAscending("name");
```

---

<!-- CLAUDE_SKIP_START -->
## Key Takeaways

### Do's:
1. ✅ Use mixins for composition
2. ✅ Localize frequently used globals
3. ✅ Prefer events over OnUpdate
4. ✅ Use frame pools for repeated UI elements
5. ✅ Implement dirty tracking for UI updates
6. ✅ Cache expensive API calls
7. ✅ Validate inputs with assert
8. ✅ Use callback registries for decoupling
9. ✅ Organize code into namespaces
10. ✅ Reuse tables instead of creating new ones
11. ✅ Use table.create() for preallocated arrays (11.1.7+)
12. ✅ Use C_EncodingUtil for serialization (11.1.5+)
13. ✅ Check issecretvalue() before automation (12.0.0+)
14. ✅ Use ColorManager for dynamic colors (11.1.5+)

### Don'ts:
1. ❌ Don't pollute global namespace
2. ❌ Don't use OnUpdate for timers
3. ❌ Don't concatenate strings in loops
4. ❌ Don't forget to unregister events
5. ❌ Don't hardcode values (use constants)
6. ❌ Don't ignore nil returns from APIs
7. ❌ Don't create frames every time (use pools)
8. ❌ Don't update UI on every data change (batch)
9. ❌ Don't use global functions (use mixins/namespaces)
10. ❌ Don't forget to release pooled resources
11. ❌ Don't hardcode item quality colors (use ColorManager)
12. ❌ Don't use deprecated global APIs (use C_* namespaces)
13. ❌ Don't automate based on secret values in combat (12.0.0+)

---

## Reference Files

| Pattern | File Path |
|---------|-----------|
| Event Utilities | `Blizzard_SharedXML\EventUtil.lua` |
| Table Builder | `Blizzard_SharedXML\TableBuilder.lua` |
| Sort Utilities | `Blizzard_SharedXML\SortUtil.lua` |
| Data Provider | `Blizzard_SharedXML\DataProvider.lua` |
| Layout Frame | `Blizzard_SharedXML\LayoutFrame.lua` |
| Callback Registry | Referenced throughout mixins |
| Mixin Examples | `Blizzard_ActionBar\Shared\*.lua` |
| Secret Values | `Blizzard_SharedXML\RestrictedActions.lua` |
| Encoding Util | `Blizzard_SharedXML\EncodingUtil.lua` |
| Housing API | `Blizzard_Housing\*.lua` |

All paths relative to: `D:\Games\World of Warcraft\_retail_\Interface\+wow-ui-source+\Interface\AddOns\`

---

**Version:** 2.0 - Updated for WoW 12.0.0 (Midnight)
**Last Updated:** 2026-01-20

<!-- CLAUDE_SKIP_END -->
