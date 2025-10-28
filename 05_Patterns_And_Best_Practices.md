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
10. [Common Lua Idioms](#common-lua-idioms)

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
    {name = "Alice", level = 70},
    {name = "Bob", level = 65},
    {name = "Charlie", level = 70},
};

local maxLevelPlayers = FilterTable(players, function(player)
    return player.level == 70;
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

    if not config.level or config.level < 1 or config.level > 70 then
        tinsert(errors, "Level must be between 1 and 70");
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

All paths relative to: `D:\Games\World of Warcraft\_retail_\Interface\+wow-ui-source+ (11.2.7)\Interface\AddOns\`

---

**Version:** 1.0 - Based on WoW 11.2.7 (The War Within)
**Last Updated:** 2025-10-19

<!-- CLAUDE_SKIP_END -->
