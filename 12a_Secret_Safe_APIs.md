# WoW 12.0+ Secret Values: Complete API Reference

## Table of Contents
1. [Overview](#overview)
2. [Why Secret Values Exist](#why-secret-values-exist)
3. [Core Secret Value Functions](#core-secret-value-functions)
4. [Secret Value Detection Functions](#secret-value-detection-functions)
5. [Secret Value Manipulation Functions](#secret-value-manipulation-functions)
6. [Secure Execution Functions](#secure-execution-functions)
7. [Table Security System](#table-security-system)
8. [SecureTypes Containers](#securetypes-containers)
9. [APIs That Return Secret Values](#apis-that-return-secret-values)
10. [APIs That Accept Secret Values](#apis-that-accept-secret-values)
11. [Common Patterns and Solutions](#common-patterns-and-solutions)
12. [Testing and Debugging](#testing-and-debugging)
13. [Real-World Examples](#real-world-examples)
14. [Migration Checklist](#migration-checklist)
15. [Quick Reference Table](#quick-reference-table)

---

## Overview

WoW 12.0.0 (Midnight) introduced the **Secret Values** system as part of Blizzard's "addon disarmament" initiative. This system fundamentally changes how addons interact with combat-sensitive data by making certain API return values unusable by tainted (addon) code during combat or restricted states.

**Key Concept:** Secret values are special Lua values that LOOK normal but CANNOT be used in arithmetic, comparisons, concatenations, or most other operations when called from tainted (addon) code.

**Scope:** Secret values are active during **ALL combat contexts**, including open-world combat, not just instanced content (dungeons, raids, Mythic+, PvP). Any time a player is in combat anywhere in the game, affected APIs return secret values.

```lua
-- Example: UnitHealth() during combat
local health = UnitHealth("target")  -- Returns a secret value

-- FAILS during combat:
if health > 0 then           -- ERROR: attempt to compare (a secret value)
local percent = health / 100 -- ERROR: attempt to perform arithmetic on a secret value
print("HP: " .. health)      -- ERROR: attempt to concatenate a secret value

-- WORKS (special handling):
healthBar:SetValue(health)   -- Native StatusBar accepts secrets at C++ level
```

### Core Principle: Pass Secrets Directly to Engine APIs

When a WoW API returns a secret value, the preferred approach is to pass it **DIRECTLY** to engine-level C++ APIs rather than trying to unwrap, convert, or manipulate it in Lua.

Engine APIs that accept secret values include:
- `FontString:SetText()` -- display secret strings directly
- `StatusBar:SetValue()` / `SetMinMaxValues()` -- display secret numbers
- `StatusBar:SetTimerDuration()` -- animate with secret duration objects
- `CooldownFrame:SetCooldownFromDurationObject()` -- cooldown display
- `Frame:SetAlphaFromBoolean()` -- conditional visibility

The **WRONG** approach is to use `issecretvalue()` to check and then provide a fallback value for display. Only use `issecretvalue()` as a guard when you **MUST** perform Lua operations (arithmetic, comparison, string ops, table indexing) on the value. If the value is just being displayed by an engine widget, pass it through untouched.

```lua
-- CORRECT: Pass secret directly to engine API for display
local health = UnitHealth("target")
healthBar:SetValue(health)              -- Engine handles it
healthText:SetText(someSecretString)    -- Engine handles it

-- WRONG: Unnecessary fallback for display-only use
local health = UnitHealth("target")
if issecretvalue(health) then
    healthBar:SetValue(0)  -- Don't do this if the bar accepts secrets!
else
    healthBar:SetValue(health)
end

-- CORRECT use of issecretvalue: when you need Lua operations
local health = UnitHealth("target")
if issecretvalue(health) then
    -- Can't do arithmetic, use percentage API instead
    local pct = UnitHealthPercent("target", false, CurveConstants.ScaleTo100) or 0
    customTexture:SetWidth((pct / 100) * BAR_WIDTH)
else
    customTexture:SetWidth((health / maxHealth) * BAR_WIDTH)
end
```

---

## Why Secret Values Exist

Blizzard's stated rationale:

> "When addons could analyze combat information in real-time, they could process combat information to make decisions for players, meaning players no longer need to make these decisions themselves."

**Goals of the system:**
1. Prevent automation addons from making combat decisions for players
2. Stop damage meters from parsing combat data (see [12_API_Migration_Guide.md](12_API_Migration_Guide.md#c_damagemeter-namespace---secret-protected-unusable-by-addons))
3. Maintain visual display capabilities (health bars, unit frames still work)
4. Allow Blizzard's own UI to function normally via secure code paths

---

## Core Secret Value Functions

These are the fundamental functions for working with secret values. All are global functions available in the `_G` namespace.

### issecretvalue(value)

**The most important function for addon developers.** Checks if a value is a secret value.

```lua
-- Signature
local isSecret = issecretvalue(value)

-- Parameters:
--   value: Any Lua value to check

-- Returns:
--   boolean: true if the value is secret, false otherwise

-- Example usage:
local health = UnitHealth("target")
if issecretvalue(health) then
    -- Value is secret - use alternative approach
    local healthPercent = UnitHealthPercent("target", false, CurveConstants.ScaleTo100)
    UpdateHealthBarWithPercent(healthPercent)
else
    -- Value is normal - can use directly
    UpdateHealthBarWithValue(health)
end
```

**Best Practice:** Always check `issecretvalue` before performing operations on potentially secret values during combat.

### canaccessvalue(value)

Checks if the calling function has permission to access and operate on a specific value.

```lua
-- Signature
local canAccess = canaccessvalue(value)

-- Parameters:
--   value: Any Lua value to check

-- Returns:
--   boolean: true if the caller can access the value, false otherwise

-- Example:
local spellID = C_ActionBar.GetActionInfo(slot)
if canaccessvalue(spellID) then
    -- Safe to use
    ProcessSpellID(spellID)
end
```

**Difference from issecretvalue:** `canaccessvalue` considers the calling context - secure Blizzard code may be able to access values that addon code cannot.

### canaccessallvalues(...)

Checks if the caller can access ALL supplied values.

```lua
-- Signature
local canAccessAll = canaccessallvalues(value1, value2, value3, ...)

-- Parameters:
--   ...: Variable number of values to check

-- Returns:
--   boolean: true if ALL values are accessible, false if ANY is secret/inaccessible

-- Example:
local actionType, spellID, subType = C_ActionBar.GetActionInfo(slot)
if canaccessallvalues(actionType, spellID, subType) then
    -- All values are accessible
    ProcessActionInfo(actionType, spellID, subType)
end
```

### hasanysecretvalues(...)

Checks if ANY of the supplied values is a secret value.

```lua
-- Signature
local hasSecrets = hasanysecretvalues(value1, value2, value3, ...)

-- Parameters:
--   ...: Variable number of values to check

-- Returns:
--   boolean: true if ANY value is secret, false if none are secret

-- Example:
local data1, data2, data3 = SomeAPI()
if hasanysecretvalues(data1, data2, data3) then
    -- At least one value is secret - defer processing
    ScheduleForOutOfCombat(ProcessData, data1, data2, data3)
else
    ProcessData(data1, data2, data3)
end
```

---

## Secret Value Detection Functions

### canaccesssecrets()

Checks if the immediate calling function has permission to access or operate on secret values. This is primarily useful for library code that needs to know its execution context.

```lua
-- Signature
local canAccess = canaccesssecrets()

-- Returns:
--   boolean: true if the caller has secret access permissions

-- Example:
local function SecureAwareWrapper(value)
    if canaccesssecrets() then
        -- Running in secure context - full access
        return ProcessValueFully(value)
    else
        -- Running in tainted context - limited access
        return ProcessValueSafely(value)
    end
end
```

### canaccesstable(table)

Checks if the caller can index a potentially secret table.

```lua
-- Signature
local canAccess = canaccesstable(tbl)

-- Parameters:
--   tbl: A table to check

-- Returns:
--   boolean: true if the table is accessible, false if:
--     - The caller cannot access the table value itself
--     - Access to table contents is disallowed by taint

-- Example:
local someTable = GetSomeAPITable()
if canaccesstable(someTable) then
    for k, v in pairs(someTable) do
        -- Safe to iterate
    end
end
```

### issecrettable(table)

Checks if a table is secret or has contents that are secret.

```lua
-- Signature
local isSecretTable = issecrettable(tbl)

-- Parameters:
--   tbl: A table to check

-- Returns:
--   boolean: true if:
--     - The table value itself is secret, OR
--     - Flags on the table make accesses produce secrets

-- Example:
local data = C_SomeAPI.GetData()
if issecrettable(data) then
    -- Cannot safely iterate this table
    HandleSecretTableData(data)
end
```

---

## Secret Value Manipulation Functions

### scrubsecretvalues(...)

Transforms values by replacing any secrets with nil.

```lua
-- Signature
local clean1, clean2, ... = scrubsecretvalues(value1, value2, ...)

-- Parameters:
--   ...: Variable number of values to process

-- Returns:
--   ...: Same values, but secrets replaced with nil

-- Example:
local val1, val2, val3 = SomeAPI()
local safe1, safe2, safe3 = scrubsecretvalues(val1, val2, val3)
-- safe1, safe2, safe3 are either the original values or nil (if secret)
```

### scrub(...)

More aggressive than `scrubsecretvalues`. Replaces values that are:
- Secret values
- Not string, number, or boolean type

```lua
-- Signature
local clean1, clean2, ... = scrub(value1, value2, ...)

-- Parameters:
--   ...: Variable number of values to process

-- Returns:
--   ...: Same values, but secrets AND non-primitive types replaced with nil

-- Example:
local val1, val2, val3 = SomeAPI()
local safe1, safe2, safe3 = scrub(val1, val2, val3)
-- Only string, number, boolean values that aren't secret remain
```

### secretwrap(...)

Converts regular values into secret values. **Restricted function** - primarily for Blizzard code.

```lua
-- Signature
local secret1, secret2, ... = secretwrap(value1, value2, ...)

-- Parameters:
--   ...: Values to convert to secrets

-- Returns:
--   ...: Secret-wrapped versions of the values

-- Note: This is HasRestrictions = true - may not work from addon code
```

### secretunwrap(...)

Unwraps secret values back to regular values. **Restricted function** - only works in secure code.

```lua
-- Signature
local unwrapped1, unwrapped2, ... = secretunwrap(secret1, secret2, ...)

-- Parameters:
--   ...: Secret values to unwrap

-- Returns:
--   ...: Unwrapped values, or nil if called from tainted code

-- Note: HasRestrictions = true - returns nil for tainted callers
```

### mapvalues(func, ...)

Applies a function over all supplied values individually.

```lua
-- Signature
local result1, result2, ... = mapvalues(func, value1, value2, ...)

-- Parameters:
--   func: Function to apply to each value
--   ...: Values to transform

-- Returns:
--   ...: Transformed values

-- Example:
local function SafeToString(val)
    if issecretvalue(val) then
        return "<secret>"
    end
    return tostring(val)
end

local str1, str2, str3 = mapvalues(SafeToString, val1, val2, val3)
```

### dropsecretaccess()

Removes the ability for the immediate calling function to access secret values. Used by Blizzard code to voluntarily drop privileges.

```lua
-- Signature
dropsecretaccess()

-- No parameters, no return value
-- Effect: The calling function loses secret access permissions

-- Example (Blizzard pattern):
local function ProcessUntrustedInput(data)
    dropsecretaccess()  -- Drop privileges before processing
    -- Now this function cannot access secrets even if it could before
end
```

---

## Secure Execution Functions

These functions allow secure (untainted) code to be called in a way that prevents taint propagation.

### securecallfunction(func, ...)

Calls a function securely, preventing taint propagation.

```lua
-- Signature
local result1, result2, ... = securecallfunction(func, arg1, arg2, ...)

-- Parameters:
--   func: Function to call securely
--   ...: Arguments to pass to the function

-- Returns:
--   ...: Return values from the function

-- Example (from Blizzard code):
local function GetValueSecure(self)
    return self.value
end

function SomeObject:GetValue()
    return securecallfunction(GetValueSecure, self)
end
```

**Key Use Case:** Blizzard uses this extensively in SecureTypes to safely access values that might be tainted.

### secureexecuterange(table, func, ...)

Iterates over a table and calls a function for each key-value pair, securely.

```lua
-- Signature
secureexecuterange(tbl, func, arg1, arg2, ...)

-- Parameters:
--   tbl: Table to iterate
--   func: Function(index, key, value, ...) to call for each entry
--   ...: Additional arguments to pass to func

-- Example (from Blizzard Pools.lua):
local function Accumulate(poolKey, pool, total)
    total:Add(pool:GetNumActive())
end

function SecurePoolCollectionMixin:GetNumActive()
    local total = CreateSecureNumber()
    self.pools:ExecuteRange(Accumulate, total)
    return total:GetValue()
end
```

### forceinsecure()

Forces the execution context to be insecure/tainted.

```lua
-- Signature
forceinsecure()

-- No parameters, no return value
-- Effect: Makes the current execution tainted

-- Example (from Blizzard Dump.lua):
function DevTools_DumpCommand(msg, editBox)
    forceinsecure()  -- Force taint for safety
    -- ... dump logic
end
```

---

## Table Security System

WoW 12.0.0 introduces a table security option system for controlling how tables interact with the secret/taint system.

### SetTableSecurityOption(table, option)

Sets security options on a table. **Restricted function**.

```lua
-- Signature
SetTableSecurityOption(tbl, option)

-- Parameters:
--   tbl: The table to configure
--   option: TableSecurityOption enum value

-- TableSecurityOption enum:
-- TableSecurityOption.DisallowTaintedAccess (0)
--   - Prevents tainted code from accessing the table
--
-- TableSecurityOption.DisallowSecretKeys (1)
--   - Prevents secret values from being used as table keys
--
-- TableSecurityOption.SecretWrapContents (2)
--   - Automatically wraps all table contents as secrets
```

**Note:** This is `HasRestrictions = true` - addon code likely cannot use it.

---

## SecureTypes Containers

Blizzard provides a set of secure container types in `SecureTypes` namespace for managing data that may have mixed secure/insecure sources.

### SecureTypes.CreateSecureMap(mixin)

Creates a map (dictionary) that safely handles tainted access.

```lua
-- Signature
local map = SecureTypes.CreateSecureMap(mixin)

-- Parameters:
--   mixin: (Optional) Additional methods to add to the map

-- Methods:
--   map:GetValue(key)      - Safely get a value
--   map:SetValue(key, val) - Set a value (asserts no secrets)
--   map:ClearValue(key)    - Remove a key
--   map:HasKey(key)        - Check if key exists
--   map:GetNext(key)       - Iterate (next() equivalent)
--   map:GetSize()          - Count entries
--   map:IsEmpty()          - Check if empty
--   map:Wipe()             - Clear all entries
--   map:Enumerate()        - Iterator for for-loops
--   map:ExecuteRange(func, ...) - Secure iteration

-- Example (from Blizzard Pools.lua):
function SecureObjectPoolMixin:Init(proxy, createFunc, resetFunc, capacity)
    self.activeObjects = CreateSecureMap()
    -- ...
end
```

**Important:** SecureMap asserts that you cannot store secret keys or values:
```lua
function SecureMap:SetValue(key, value)
    assert(not issecretvalue(key), "attempted to store a secret key in a SecureMap")
    assert(not issecretvalue(value), "attempted to store a secret value in a SecureMap")
    self.tbl[key] = value
end
```

### SecureTypes.CreateSecureArray()

Creates an array that safely handles tainted access.

```lua
-- Signature
local array = SecureTypes.CreateSecureArray()

-- Methods:
--   array:GetValue(index)     - Safely get value at index
--   array:Insert(value, idx)  - Insert value (asserts no secrets)
--   array:UniqueInsert(value) - Insert only if not present
--   array:Remove(index)       - Remove and return value at index
--   array:RemoveValue(value)  - Remove specific value
--   array:FindInTableIf(pred) - Find matching element
--   array:ContainsIf(pred)    - Check if any match predicate
--   array:Contains(value)     - Check if value exists
--   array:GetSize()           - Get length
--   array:IsEmpty()           - Check if empty
--   array:Wipe()              - Clear array
--   array:HasValues()         - Alias for not IsEmpty
--   array:Enumerate()         - Forward iterator
--   array:EnumerateReverse()  - Reverse iterator
--   array:ExecuteRange(func)  - Secure iteration
```

### SecureTypes.CreateSecureStack()

Creates a stack (LIFO) that safely handles tainted access.

```lua
-- Signature
local stack = SecureTypes.CreateSecureStack()

-- Methods:
--   stack:Push(value)     - Push value (asserts no secrets)
--   stack:Pop()           - Pop and return top value
--   stack:Contains(value) - Check if value in stack
```

### SecureTypes.CreateSecureValue(value)

Creates a wrapper for a single value with secure access.

```lua
-- Signature
local secureVal = SecureTypes.CreateSecureValue(initialValue)

-- Methods:
--   secureVal:GetValue() - Safely retrieve the value
--   secureVal:SetValue(v) - Set new value (asserts no secrets)
```

### SecureTypes.CreateSecureNumber(value)

Creates a wrapper for a number with secure access and arithmetic operations.

```lua
-- Signature
local secureNum = SecureTypes.CreateSecureNumber(initialValue) -- default 0

-- Methods:
--   secureNum:GetValue()    - Get the number
--   secureNum:SetValue(v)   - Set new value (asserts no secrets)
--   secureNum:Add(v)        - Add to current value
--   secureNum:Subtract(v)   - Subtract from current value
--   secureNum:Increment()   - Add 1
--   secureNum:Decrement()   - Subtract 1

-- Example (from Blizzard Pools.lua):
function SecureObjectPoolMixin:Init(...)
    self.activeObjectCount = CreateSecureNumber()
end

function SecureObjectPoolMixin:AddObject(object)
    self.activeObjectCount:Increment()
end
```

### SecureTypes.CreateSecureBoolean(value)

Creates a wrapper for a boolean with secure access.

```lua
-- Signature
local secureBool = SecureTypes.CreateSecureBoolean(initialValue)

-- Methods:
--   secureBool:GetValue()    - Get the boolean
--   secureBool:SetValue(v)   - Set new value (asserts no secrets)
--   secureBool:ToggleValue() - Flip true/false
--   secureBool:IsTrue()      - Shorthand for GetValue() == true
```

### SecureTypes.CreateSecureFunction()

Creates a wrapper for a function with secure calling.

```lua
-- Signature
local secureFunc = SecureTypes.CreateSecureFunction()

-- Methods:
--   secureFunc:IsSet()           - Check if function is set
--   secureFunc:SetFunction(fn)   - Set the function (asserts no secrets)
--   secureFunc:CallFunction(...) - Call the function securely
--   secureFunc:CallFunctionIfSet(...) - Call only if set
```

---

## APIs That Return Secret Values

The following APIs return secret values **during combat** (or when `secretCombatRestrictionsForced` CVar is enabled):

### Unit Health/Power APIs

| API | Returns | Secret During Combat |
|-----|---------|---------------------|
| `UnitHealth(unit)` | Current health | **YES** |
| `UnitHealthMax(unit)` | Maximum health | **YES** |
| `UnitPower(unit, powerType)` | Current power | **YES** |
| `UnitPowerMax(unit, powerType)` | Maximum power | **YES** |
| `UnitGetTotalAbsorbs(unit)` | Absorb shields | **YES** |
| `UnitGetTotalHealAbsorbs(unit)` | Heal absorbs | **YES** |
| `UnitGetIncomingHeals(unit, healer)` | Incoming heals | **YES** |
| `UnitName(unit)` | Unit name | **Sometimes** (in restricted scenarios) |

### Non-Secret Alternatives

| API | Returns | Notes |
|-----|---------|-------|
| `UnitHealthPercent(unit, usePredicted, curveConstant)` | 0-100 percentage | **NOT SECRET** - Use for arithmetic! |
| `UnitPowerPercent(unit, powerType, curveConstant)` | 0-100 percentage | **NOT SECRET** - Use for arithmetic! |

### Unit Comparison and Identity APIs

Several unit APIs return secret values during combat that cause errors if used directly in boolean tests, comparisons, or as table keys.

| API | Returns | Secret During Combat | Notes |
|-----|---------|---------------------|-------|
| `UnitIsUnit(unit1, unit2)` | boolean | **YES** | Cannot use in `if` directly |
| `UnitIsDead(unit)` | boolean | **Sometimes** | May be secret in restricted contexts |
| `UnitName(unit)` | string | **Sometimes** | Cannot use as table key or in string ops |
| `UnitClass(unit)` | string, string | **Sometimes** | Class token may be secret |
| `UnitGUID(unit)` | string | **Sometimes** | Cannot use as table key |

**`UnitIsUnit()` is particularly dangerous** because its return value is commonly used in `if` conditions. Attempting a boolean test on a secret value causes: `"attempt to perform boolean test on a secret value"`.

```lua
-- WRONG: Direct boolean test on potentially secret value
local isTarget = UnitIsUnit("target", unitid)
if isTarget then  -- ERROR if isTarget is secret!
    -- highlight as target
end

-- CORRECT: Sanitize immediately after the API call
local isTarget = UnitIsUnit("target", unitid)
if issecretvalue and issecretvalue(isTarget) then
    isTarget = false  -- Safe fallback: assume not the target
end
if isTarget then
    -- highlight as target
end

-- ALSO WRONG: Storing secret in table then using later
local unitFlags = {}
unitFlags[unitid] = UnitIsUnit("target", unitid)  -- May store secret
-- ... later ...
if unitFlags[unitid] then  -- ERROR if secret was stored!

-- CORRECT: Sanitize before storing
local isTarget = UnitIsUnit("target", unitid)
if issecretvalue and issecretvalue(isTarget) then
    isTarget = false
end
unitFlags[unitid] = isTarget  -- Now safe to use later

-- UnitName/UnitGUID: Cannot use as table keys when secret
local name = UnitName("target")
if issecretvalue and issecretvalue(name) then
    -- Can't index tables with this name or do string operations
    -- But CAN pass to FontString:SetText()
    nameText:SetText(name)
end
```

### Action Bar APIs

All C_ActionBar functions may return secret values during combat:

| API | Returns | Notes |
|-----|---------|-------|
| `C_ActionBar.GetActionInfo(slot)` | actionType, id, subType | id may be secret |
| `C_ActionBar.GetActionTexture(slot)` | texture | May be secret |
| `C_ActionBar.IsUsableAction(slot)` | usable | May be secret |

### C_DamageMeter APIs

ALL meaningful data from C_DamageMeter is SECRET:

| Field | Status |
|-------|--------|
| `name` | SECRET |
| `totalAmount` | SECRET |
| `amountPerSecond` | SECRET |
| `sourceGUID` | SECRET |
| `classFilename` | Accessible (useless without above) |
| `isLocalPlayer` | Accessible (useless without above) |

### Tooltip Data

`C_TooltipInfo` may return secret data for certain tooltip lines:

```lua
local tooltipData = C_TooltipInfo.GetUnit(unit)
local line = tooltipData.lines[2]
-- line.leftText may be secret in combat
if issecretvalue(line.leftText) then
    -- Cannot read tooltip text
end
```

---

## APIs That Accept Secret Values

These UI widgets have C++ level support for accepting and displaying secret values. At the engine level, these methods have `SecretArguments = "AllowedWhenTainted"`, meaning the C++ code natively handles secret values without any Lua-side unwrapping.

### FontString:SetText()

`FontString:SetText()` has `SecretArguments = "AllowedWhenTainted"` at the engine level. This means it natively accepts and correctly renders secret values. You can pass secret return values directly from APIs like `GetAuraApplicationDisplayCount()` to `SetText()` without any conversion or unwrapping.

```lua
-- CORRECT: Pass secret value directly to SetText
local stackStr = C_UnitAuras.GetAuraApplicationDisplayCount(unitid, auraInstanceID, 2, 1000)
fontString:SetText(stackStr)  -- Works even if stackStr is a secret value

-- CORRECT: Any secret string can be displayed this way
local unitName = UnitName("target")  -- May be secret during combat
nameText:SetText(unitName)           -- Engine renders it correctly

-- WRONG: Don't try to unwrap/convert secret values for display
local stackStr = C_UnitAuras.GetAuraApplicationDisplayCount(unitid, auraInstanceID, 2, 1000)
local num = tonumber(stackStr)  -- ERROR: can't convert secret to number
if stackStr ~= "" then          -- ERROR: can't compare secret with string
    fontString:SetText(stackStr)
end

-- WRONG: Don't use SafeValue() or tostring() on secrets for SetText
-- SetText already handles them natively at the C++ level
```

### StatusBar Frames

```lua
local healthBar = CreateFrame("StatusBar", nil, parent)
healthBar:SetMinMaxValues(0, UnitHealthMax(unit))  -- Works with secrets!
healthBar:SetValue(UnitHealth(unit))               -- Works with secrets!

-- NEW in 12.0.0: SetTimerDuration for secret-safe duration objects
healthBar:SetTimerDuration(durationObject)         -- Works with secret durations!
```

### StatusBar:SetTimerDuration() (NEW in 12.0.0)

For cast bars and cooldown displays, use `SetTimerDuration()` with duration objects. This method provides **smooth frame-by-frame animation at the C++ level** -- no Lua `OnUpdate` script is needed for the bar fill. The engine interpolates the bar position every render frame, producing perfectly smooth animation that is impossible to match with Lua-driven `OnUpdate` + `SetValue()`.

This is the **preferred method for cast bars on 12.0.0+** because:
1. The engine handles all timing interpolation internally
2. No floating-point precision issues (see StatusBar precision warning below)
3. Works with secret duration objects from combat APIs
4. Zero Lua overhead for the animation itself

```lua
-- UnitCastingDuration/UnitChannelDuration return secret-safe duration objects
local duration = UnitCastingDuration(unit)
castBar:SetTimerDuration(duration)
-- That's it! The bar animates smoothly with no OnUpdate needed.

-- Duration objects have methods that work with secret values:
local total = duration:GetTotalDuration()      -- May be secret
local remaining = duration:GetRemainingDuration()  -- May be secret
local isZero = duration:IsZero()               -- Boolean, safe to use

-- Duration direction matters:
-- UnitCastingDuration() returns a forward-filling duration (ElapsedTime)
-- UnitChannelDuration() returns a reverse-filling duration (RemainingTime)
-- Both start Immediately when passed to SetTimerDuration()

-- Compare with manual Lua OnUpdate approach (more complex, less smooth):
-- castBar:SetMinMaxValues(0, durationSec)
-- castBar:SetScript("OnUpdate", function()
--     castBar:SetValue(GetTime() - startTime)  -- Requires careful float handling
-- end)
```

### Frame:SetAlphaFromBoolean() (NEW in 12.0.0)

Sets frame alpha based on a boolean and two values, handling secrets at C++ level:
```lua
-- SetAlphaFromBoolean(condition, valueIfTrue, valueIfFalse)
frame:SetAlphaFromBoolean(duration:IsZero(), 0, 1)
```

### C_CurveUtil.EvaluateColorValueFromBoolean() (NEW in 12.0.0)

Evaluates a boolean to produce a color/alpha value, secret-safe:
```lua
local alpha = C_CurveUtil.EvaluateColorValueFromBoolean(notInterruptible, 0, 1)
frame:SetAlphaFromBoolean(condition, 0, alpha)
```

### FontString:SetFormattedText() with Secret Values

`FontString:SetFormattedText(formatString, ...)` definitively accepts secret values as format arguments from tainted addon code. The format string itself must be a regular Lua string (not secret), but `%s` arguments can be secret values. The C++ implementation handles the substitution internally.

This is extremely useful for wrapping secret strings in formatting without Lua concatenation:

```lua
-- Wrap a secret player name in parentheses
fontString:SetFormattedText("(%s)", secretName)  -- Displays: (PlayerName)

-- Prefix with static text
fontString:SetFormattedText("Interrupted by %s", secretName)  -- Displays: Interrupted by PlayerName
```

**Note:** This only works when the entire FontString can be one color. For mixed-color display (e.g., white "Interrupted" + class-colored name), use the Two-FontString Inline Pattern (see Common Patterns below).

### SetAlpha on Frames

Platynator uses this pattern:
```lua
if issecretvalue(raw) then
    self.text:SetAlpha(raw)  -- StatusBar-like handling
else
    self.text:SetAlpha(raw > 0 and 1 or 0)
end
```

### CreateUnitHealPredictionCalculator() (NEW in 12.0.0)

Creates a calculator object that works with secret health/absorb values:
```lua
local calculator = CreateUnitHealPredictionCalculator()

-- Configure the calculator
if calculator.SetMaximumHealthMode then
    calculator:SetMaximumHealthMode(Enum.UnitMaximumHealthMode.WithAbsorbs)
    calculator:SetDamageAbsorbClampMode(Enum.UnitDamageAbsorbClampMode.MaximumHealth)
end

-- Feed it data (works with secrets internally)
UnitGetDetailedHealPrediction(unit, nil, calculator)

-- Get results (may return secrets, but StatusBar accepts them)
local maxHealth = calculator:GetMaximumHealth()
local absorbs = calculator:GetDamageAbsorbs()

statusBar:SetMinMaxValues(0, maxHealth)
statusBar:SetValue(UnitHealth(unit))
```

### Aura Data Secret Values (12.0.0 - CRITICAL)

**Secret values affect ALL combat contexts, including open world.** This is NOT limited to instanced content (dungeons, raids, M+, PvP). Any time a player is in combat anywhere in the game, aura data is secret.

#### What Is Secret vs. Non-Secret

During combat, virtually ALL aura data fields are secret. The ONLY non-secret field is `auraInstanceID`.

| Aura Data Field | Secret During Combat? | Notes |
|-----------------|----------------------|-------|
| `auraInstanceID` | **NO** | The ONLY non-secret field |
| `name` | **YES** | Cannot filter/compare by name |
| `spellId` | **YES** | Cannot filter/compare by spell ID |
| `icon` | **YES** | Cannot use for custom icon logic |
| `duration` | **YES** | Cannot do arithmetic |
| `expirationTime` | **YES** | Cannot calculate remaining time |
| `sourceUnit` | **YES** | Cannot identify caster |
| `dispelName` | **YES** | Cannot filter by dispel type |
| `isHarmful` | **YES** | Cannot distinguish buff/debuff |
| `isHelpful` | **YES** | Cannot distinguish buff/debuff |
| `isFromPlayerOrPlayerPet` | **YES** | Cannot filter "my" auras via data |
| `applications` (stacks) | **YES** | Cannot read stack count |
| `timeMod` | **YES** | Cannot read time modification |
| `points` | **YES** | Cannot read aura values |

#### All Aura APIs Return Secret Data During Combat

Every method of querying aura data is affected:

```lua
-- ALL of these return secret data during combat (except auraInstanceID):

-- Method 1: GetUnitAuras (bulk query)
local auras = C_UnitAuras.GetUnitAuras(unit, filter)
for _, aura in ipairs(auras) do
    -- aura.auraInstanceID  -- NON-SECRET (usable)
    -- aura.name            -- SECRET (unusable for comparisons)
    -- aura.spellId         -- SECRET (unusable for comparisons)
    -- aura.icon            -- SECRET
    -- aura.duration        -- SECRET
    -- All other fields     -- SECRET
end

-- Method 2: GetAuraDataByAuraInstanceID (single aura lookup)
local aura = C_UnitAuras.GetAuraDataByAuraInstanceID(unit, instanceID)
-- Same result: all fields secret except auraInstanceID

-- Method 3: GetUnitAuraBySpellID (spell ID lookup)
local aura = C_UnitAuras.GetUnitAuraBySpellID(unit, spellID)
-- Returns NIL during combat (not secret data, just nil!)

-- Method 4: GetAuraDataBySpellName (name lookup)
local aura = C_UnitAuras.GetAuraDataBySpellName(unit, "Moonfire")
-- Returns NIL during combat

-- Method 5: AuraUtil.FindAuraByName (helper)
local aura = AuraUtil.FindAuraByName("Moonfire", unit)
-- Returns NIL during combat

-- Method 6: UNIT_AURA event payload
-- The addedAuras table in UNIT_AURA events also has secret fields
-- spellId and name are secret even at the moment of FIRST APPLICATION
```

#### auraInstanceIDs Are Per-Instance, Not Per-Spell

Each individual aura application gets a unique `auraInstanceID`. The same spell on different targets has different IDs:

```lua
-- Blood Plague on Mob A: auraInstanceID = 12345
-- Blood Plague on Mob B: auraInstanceID = 67890
-- These are DIFFERENT IDs - cannot be used as spell identifiers
```

#### Out of Combat: All Data Is Readable

When the player is NOT in combat, all aura data is fully readable and non-secret:

```lua
-- Out of combat: everything works normally
local aura = C_UnitAuras.GetAuraDataByAuraInstanceID(unit, instanceID)
if aura then
    print(aura.name)     -- Works: "Moonfire"
    print(aura.spellId)  -- Works: 8921
    print(aura.duration) -- Works: 18
end
```

#### Why Pre-Combat Caching Does NOT Work

A natural instinct is to cache aura data before entering combat. This fails because:

1. **New auras applied during combat get new auraInstanceIDs** that were never seen out of combat
2. **There is no "window" where data is non-secret during combat** -- data is secret from the moment of application (the `addedAuras` payload in `UNIT_AURA` events contains secret spellId and name fields)
3. **Spell name to spell ID resolution does not work** -- `C_Spell.GetSpellInfo("Moonfire")` returns nil in 12.0 (see below)

```lua
-- THIS CACHING APPROACH DOES NOT WORK:
local auraCache = {}  -- spellId -> data

-- Cache before combat
local function CacheAuras(unit)
    local auras = C_UnitAuras.GetUnitAuras(unit, "HELPFUL")
    for _, aura in ipairs(auras) do
        auraCache[aura.spellId] = { name = aura.name, icon = aura.icon }
    end
end

-- Problem: During combat, new auras appear with new auraInstanceIDs.
-- Their spellId is SECRET, so we can't look them up in the cache.
-- Even the UNIT_AURA addedAuras payload has secret spellId values.
```

#### What Still Works During Combat

Despite the restrictions, some approaches still function:

```lua
-- 1. API-level filter strings still work
-- These are processed at the C++ level before data reaches Lua
local auras = C_UnitAuras.GetUnitAuras(unit, "HELPFUL|PLAYER")
-- Returns only the player's helpful auras (filtering done in C++)

local auras = C_UnitAuras.GetUnitAuras(unit, "INCLUDE_NAME_PLATE_ONLY|HARMFUL")
-- Returns only nameplate-relevant harmful auras

-- 2. Secret-safe display APIs work (see sections below)
local duration = C_UnitAuras.GetAuraDuration(unit, auraInstanceID)
local displayCount = C_UnitAuras.GetAuraApplicationDisplayCount(unit, auraInstanceID, 2, 1000)

-- 3. Blizzard's own UI code works because it runs in a privileged secure context
-- Third-party addons do NOT have this privilege

-- 4. issecretvalue() can detect secret values to avoid errors
if issecretvalue and issecretvalue(aura.spellId) then
    -- Handle gracefully
end
```

#### Practical Implications for Addon Developers

| Use Case | Status in 12.0 Combat | Alternative |
|----------|----------------------|-------------|
| Filter auras by spell name/ID | **IMPOSSIBLE** | Use API-level filter strings |
| Show aura icons on nameplates | **Works** via C++ filter + secret-safe display | `INCLUDE_NAME_PLATE_ONLY` filter |
| Custom aura priority lists | **IMPOSSIBLE** during combat | Pre-filter via API filter strings |
| Track specific player debuffs | **IMPOSSIBLE** to identify by name/ID | Use `HARMFUL\|PLAYER` filter, display all |
| Show aura duration countdown | **Works** via `C_UnitAuras.GetAuraDuration()` | Duration object + `SetCooldownFromDurationObject()` |
| Show aura stack count | **Works** via `C_UnitAuras.GetAuraApplicationDisplayCount()` | Pass directly to `SetText()` |
| Display aura tooltips | **Works** -- `GameTooltip:SetUnitAura()` handles secrets | Engine-level tooltip rendering |

### C_UnitAuras Secret-Safe Functions (NEW in 12.0.0)

```lua
-- GetAuraDuration returns a secret-safe duration object
local duration = C_UnitAuras.GetAuraDuration(unit, auraInstanceID)

-- GetAuraApplicationDisplayCount returns a SECRET STRING during combat
-- Parameters: unit, auraInstanceID, minDisplayCount, maxDisplayCount
local displayCount = C_UnitAuras.GetAuraApplicationDisplayCount(unit, auraInstanceID, 2, 1000)
-- Returns "" for stacks < minDisplayCount, or the stack count as a string
-- IMPORTANT: During combat this is a SECRET string, not a plain Lua string!
-- "Secret-safe" means it's safe to pass to SetText(), NOT safe for Lua string operations.

-- CORRECT: Pass directly to SetText (engine handles secrets)
fontString:SetText(displayCount)

-- WRONG: Do not use Lua operations on the return value
if displayCount ~= "" then ... end   -- ERROR if secret
local n = tonumber(displayCount)      -- ERROR if secret
string.find(displayCount, "%d")       -- ERROR if secret

-- The API already handles the "> 1 stack" check via minDisplayCount parameter.
-- If stacks < minDisplayCount, it returns "" (empty string, non-secret).
-- If stacks >= minDisplayCount, it returns the count string (may be secret).
-- So you never need to do a numeric comparison like `if stacks > 1`.
```

### C_Spell.GetSpellInfo() No Longer Accepts Spell Names (12.0.0)

In WoW 12.0.0, `C_Spell.GetSpellInfo()` and `C_Spell.GetSpellName()` **only accept numeric spell IDs**. Passing a spell name string returns nil:

```lua
-- WORKS in 12.0.0:
C_Spell.GetSpellInfo(8921)          -- Returns spell info table for Moonfire
C_Spell.GetSpellName(8921)         -- Returns "Moonfire"

-- DOES NOT WORK in 12.0.0 (returns nil):
C_Spell.GetSpellInfo("Moonfire")   -- Returns nil
C_Spell.GetSpellName("Moonfire")   -- Returns nil
```

**Why this matters for aura tracking:** One workaround developers might attempt for the aura secret values problem is to look up spell IDs from cached spell names using `C_Spell.GetSpellInfo("SpellName")`. This approach fails because the API no longer accepts string inputs. Addon developers must use hardcoded numeric spell IDs for any spell-specific logic.

### C_Spell.GetSpellCooldownDuration() (NEW in 12.0.0)

Returns a secret-safe duration object for spell cooldowns:
```lua
local duration = C_Spell.GetSpellCooldownDuration(spellID)

-- Duration object methods:
duration:GetTotalDuration()      -- Total cooldown length
duration:GetRemainingDuration()  -- Time remaining
duration:IsZero()                -- Boolean: is cooldown ready?
```

### SetCooldownFromDurationObject and SetUseAuraDisplayTime

**IMPORTANT:** Do NOT call `SetUseAuraDisplayTime(true)` when using `SetCooldownFromDurationObject()`. `SetCooldownFromDurationObject` already handles aura-style display timing internally. Combining both causes the countdown text to display **1 second LESS** than expected (e.g., showing "4" when there are 5 seconds remaining).

| Method | SetUseAuraDisplayTime? | Why |
|--------|----------------------|-----|
| `SetCooldownFromDurationObject(durationObj)` | **NO** -- Do NOT set it | Timing handled internally by the duration object |
| `SetCooldown(start, duration)` for **auras** | **YES** -- Set to `true` | Adjusts floor-to-ceil rounding so "5s" shows as "5" not "4" |
| `SetCooldown(start, duration)` for **cooldowns** | **NO** -- Do NOT set it | Floor rounding is correct for ability cooldowns |

```lua
-- CORRECT: Aura cooldown with duration object (Platynator pattern)
local duration = C_UnitAuras.GetAuraDuration(unit, auraInstanceID)
cooldown:SetCooldownFromDurationObject(duration)
-- Do NOT call cooldown:SetUseAuraDisplayTime(true) here!

-- CORRECT: Aura cooldown with SetCooldown (Blizzard nameplate pattern)
local expirationTime = auraData.expirationTime
local auraDuration = auraData.duration
cooldown:SetUseAuraDisplayTime(true)
cooldown:SetCooldown(expirationTime - auraDuration, auraDuration)

-- WRONG: Combining both (countdown shows 1 second less than expected)
cooldown:SetUseAuraDisplayTime(true)  -- Don't do this with duration objects!
cooldown:SetCooldownFromDurationObject(duration)
```

---

## Common Patterns and Solutions

### Pattern 1: Check Before Use

```lua
local function UpdateHealthDisplay(unit)
    local health = UnitHealth(unit)
    local maxHealth = UnitHealthMax(unit)

    if issecretvalue(health) or issecretvalue(maxHealth) then
        -- Use percentage alternative
        local percent = UnitHealthPercent(unit, false, CurveConstants.ScaleTo100) or 0
        healthBar:SetWidth((percent / 100) * BAR_WIDTH)
        healthText:SetFormattedText("%.0f%%", percent)
    else
        -- Use actual values
        healthBar:SetValue(health)
        healthText:SetFormattedText("%d / %d", health, maxHealth)
    end
end
```

### Pattern 2: Defer to Out-of-Combat

```lua
local pendingUpdates = {}

local function UpdateActionButton(slot)
    local actionType, spellID = C_ActionBar.GetActionInfo(slot)

    if issecretvalue(spellID) then
        -- Queue for later
        pendingUpdates[slot] = true
        return
    end

    ProcessActionButton(slot, actionType, spellID)
end

local frame = CreateFrame("Frame")
frame:RegisterEvent("PLAYER_REGEN_ENABLED")
frame:SetScript("OnEvent", function()
    for slot in pairs(pendingUpdates) do
        UpdateActionButton(slot)
    end
    wipe(pendingUpdates)
end)
```

### Pattern 3: Native StatusBar for Display

```lua
-- Use native StatusBar which handles secrets internally
local function CreateSecretSafeHealthBar(parent)
    local bar = CreateFrame("StatusBar", nil, parent)
    bar:SetStatusBarTexture("Interface\\Buttons\\WHITE8X8")

    function bar:UpdateHealth(unit)
        -- These calls work with secrets - handled at C++ level
        self:SetMinMaxValues(0, UnitHealthMax(unit))
        self:SetValue(UnitHealth(unit))
    end

    return bar
end
```

### Pattern 4: Percentage-Based Custom Bars

```lua
-- For texture-based bars, use percentage APIs
local function UpdateTextureHealthBar(unit)
    -- UnitHealthPercent returns NON-SECRET percentage!
    local percent = UnitHealthPercent(unit, false, CurveConstants.ScaleTo100) or 0

    -- Safe to do arithmetic
    local width = math.max(1, (percent / 100) * BAR_WIDTH)
    healthTexture:SetWidth(width)

    -- Safe to compare
    if percent < 25 then
        healthTexture:SetColorTexture(1, 0, 0, 1)  -- Red
    elseif percent < 50 then
        healthTexture:SetColorTexture(1, 1, 0, 1)  -- Yellow
    else
        healthTexture:SetColorTexture(0, 1, 0, 1)  -- Green
    end
end
```

### Pattern 5: Platynator's issecretvalue Guard (Real-World Example)

From Platynator's AbsorbText.lua:
```lua
function addonTable.Display.AbsorbTextMixin:UpdateText()
    if UnitIsDeadOrGhost(self.unit) then
        self.text:SetText("+0")
        self.text:SetAlpha(0)
    else
        local raw = UnitGetTotalAbsorbs(self.unit)
        local absolute = (AbbreviateNumbersAlt or AbbreviateNumbers)(raw)
        self.text:SetText("+" .. absolute)
        if issecretvalue and issecretvalue(raw) then
            -- Use SetAlpha with the secret value directly
            -- StatusBar-like behavior handles it at C++ level
            self.text:SetAlpha(raw)
        else
            self.text:SetAlpha(raw > 0 and 1 or 0)
        end
    end
end
```

### Pattern 6: Compatibility Check for issecretvalue

```lua
-- issecretvalue may not exist in older clients
local function SafeIsSecretValue(value)
    if issecretvalue then
        return issecretvalue(value)
    end
    return false  -- Pre-12.0, no secrets
end
```

### Pattern 7: Two-FontString Inline Display (Mixed-Color Secret Text)

When you need to display text with mixed colors and part of the text is a secret value, you cannot use color escape codes (which require Lua concatenation). Instead, use two adjacent FontStrings anchored inline:

```lua
-- Create two FontStrings on the same parent
local staticText = parent:CreateFontString(nil, "OVERLAY")
local secretText = parent:CreateFontString(nil, "OVERLAY")

-- Position: staticText at desired location, secretText immediately to its right
staticText:SetPoint("LEFT", parent, "LEFT", 2, 0)
secretText:SetPoint("LEFT", staticText, "RIGHT", 0, 0)

-- CRITICAL: Set staticText width to 0 so it auto-sizes to text content.
-- Otherwise its RIGHT anchor may be at a theme-set width boundary, not the text edge.
staticText:SetWidth(0)

-- Set text and colors independently
staticText:SetText("Interrupted ")
staticText:SetTextColor(1, 1, 1)  -- White

secretText:SetFormattedText("(%s)", secretName)  -- Wraps secret in parens at C++ level
secretText:SetTextColor(C_ClassColor.GetClassColor(engClass):GetRGB())  -- Class color
```

This creates the visual appearance of a single line: "Interrupted (PlayerName)" with the name in class color.

**Key requirements:**
- `SetWidth(0)` on the first FontString so its RIGHT edge matches the text edge (see `03_UI_Framework.md` - FontString Width and Anchoring)
- Use `SetFormattedText("(%s)", secret)` to add formatting around secret values without Lua concatenation
- Match fonts between FontStrings with `secretText:SetFont(staticText:GetFont())`

### UNIT_SPELLCAST_INTERRUPTED vs UNIT_SPELLCAST_FAILED: Different Payloads

These two events are commonly handled together since both indicate a cast ending prematurely. However, their 4th argument differs:

| Event | Args 1-3 | 4th Arg |
|-------|----------|---------|
| `UNIT_SPELLCAST_INTERRUPTED` | `unit, castGUID, spellID` | `interruptedByGUID` (string, possibly secret) |
| `UNIT_SPELLCAST_FAILED` | `unit, castGUID, spellID` | *None* (nil) |

If your handler passes the 4th arg to `UnitNameFromGUID()` or `GetPlayerInfoByGUID()`, it will harmlessly return nil for FAILED events, but the logic is semantically wrong. Best practice: handle these as separate event branches, or check the event name before treating the 4th arg as a GUID.

```lua
-- Good: Separate handlers
if event == "UNIT_SPELLCAST_INTERRUPTED" then
    local interruptedBy = select(4, ...)
    HandleInterrupt(unit, interruptedBy)
elseif event == "UNIT_SPELLCAST_FAILED" then
    HandleInterrupt(unit, nil)  -- No GUID available
end
```

### interruptedBy GUID Availability Scope

The `interruptedBy` GUID from `UNIT_SPELLCAST_INTERRUPTED` is **only populated for interrupts by the player, group members, or raid members**. If an NPC or enemy player outside the group/raid interrupts a cast, the 4th argument is `nil`.

This is a WoW API restriction, not a secret value issue. The GUID is restricted by `SecretWhenUnitSpellCastRestricted`, meaning it becomes secret during combat for group/raid members, and is entirely absent for outsiders.

Addons displaying interrupter information must have a fallback:

```lua
if interruptName then
    -- Display "Interrupted (PlayerName)" with class color
else
    -- Display just "Interrupted" -- no name available
end
```

---

## Testing and Debugging

### CVars for Testing

```lua
-- Force secret restrictions outside combat (DEVELOPMENT ONLY)
/console secretCombatRestrictionsForced 1

-- Enable encounter-level restrictions
/console secretEncounterRestrictionsForced 1

-- Enable debug output for secret value operations
/console secretRestrictionsDebug 1

-- Disable all forced restrictions
/console secretCombatRestrictionsForced 0
/console secretEncounterRestrictionsForced 0
/console secretRestrictionsDebug 0
```

### Testing Commands

```lua
-- Check if secret functions exist
/run print("issecretvalue:", issecretvalue ~= nil)
/run print("canaccessvalue:", canaccessvalue ~= nil)
/run print("scrubsecretvalues:", scrubsecretvalues ~= nil)

-- Test if a value is secret (with forced restrictions)
/run local h = UnitHealth("player"); print("Health secret:", issecretvalue(h))

-- Check current restriction state
/run if C_RestrictedActions then print(C_RestrictedActions.IsInRestrictedState()) end
```

### Test Checklist

1. [ ] Enable `secretCombatRestrictionsForced` CVar
2. [ ] Target a friendly player or NPC
3. [ ] Verify health bars display correctly
4. [ ] Verify no Lua errors in error log
5. [ ] Test health percentage calculations
6. [ ] Test health text formatting
7. [ ] Disable the CVar and verify normal operation
8. [ ] Test actual combat scenarios

---

## Real-World Examples

### Platynator Addon Patterns

Platynator (a nameplate addon) demonstrates comprehensive secret-safe patterns for WoW 12.0.0+.

#### Version Detection Pattern
```lua
-- Constants.lua - Detect Midnight for conditional code paths
addonTable.Constants = {
    IsMidnight = select(4, GetBuildInfo()) >= 120000,
    -- ...
}

-- Usage throughout codebase
if addonTable.Constants.IsMidnight then
    -- Use 12.0.0+ APIs and patterns
else
    -- Use legacy patterns
end
```

#### AbsorbText.lua - Handling Secret Absorb Values
```lua
local raw = UnitGetTotalAbsorbs(self.unit)
if issecretvalue and issecretvalue(raw) then
    self.text:SetAlpha(raw)  -- Uses C++ level handling for secret values
else
    self.text:SetAlpha(raw > 0 and 1 or 0)
end
```

#### CreatureTextMSP.lua - Guarding Against Secret Unit Names
```lua
local originalName, realm = UnitName(self.unit)
if UnitIsPlayer(self.unit) and (not issecretvalue or not issecretvalue(originalName)) then
    -- Safe to use name for RP addon integration
end
```

#### GuildText.lua - Protecting Tooltip Data Access
```lua
local tooltipData = C_TooltipInfo.GetUnit(self.unit)
local line = tooltipData.lines[isColorBlindMode and 3 or 2]
if not issecretvalue and line or (issecretvalue and not issecretvalue(line) and line and not issecretvalue(line.leftText)) then
    text = line.leftText
end
```

#### HealthBar.lua - CreateUnitHealPredictionCalculator Pattern
```lua
function addonTable.Display.HealthBarMixin:PostInit()
    if addonTable.Constants.IsMidnight then
        -- Create heal prediction calculator for secret-safe health/absorb display
        self.calculator = CreateUnitHealPredictionCalculator()
        if self.calculator.SetMaximumHealthMode then
            self.calculator:SetMaximumHealthMode(Enum.UnitMaximumHealthMode.WithAbsorbs)
            self.calculator:SetDamageAbsorbClampMode(Enum.UnitDamageAbsorbClampMode.MaximumHealth)
        end
    end
end

function addonTable.Display.HealthBarMixin:UpdateHealth()
    if self.calculator then
        if self.calculator.GetMaximumHealth then
            -- Use calculator for secret-safe values
            UnitGetDetailedHealPrediction(self.unit, nil, self.calculator)
            self.statusBar:SetMinMaxValues(0, self.calculator:GetMaximumHealth())
            self.statusBarAbsorb:SetMinMaxValues(self.statusBar:GetMinMaxValues())
            local absorbs = self.calculator:GetDamageAbsorbs()
            self.statusBarAbsorb:SetValue(absorbs)
        else
            -- Fallback for StatusBar native handling
            self.statusBar:SetMinMaxValues(0, UnitHealthMax(self.unit))
            self.statusBarAbsorb:SetMinMaxValues(self.statusBar:GetMinMaxValues())
            self.statusBarAbsorb:SetValue(UnitGetTotalAbsorbs(self.unit))
        end
        self.statusBar:SetValue(UnitHealth(self.unit))
    else
        -- Pre-Midnight fallback
        local absorbs = UnitGetTotalAbsorbs(self.unit)
        self.statusBar:SetMinMaxValues(0, UnitHealthMax(self.unit) + absorbs)
        self.statusBarAbsorb:SetMinMaxValues(self.statusBar:GetMinMaxValues())
        self.statusBar:SetValue(UnitHealth(self.unit, true))
        self.statusBarAbsorb:SetValue(absorbs)
    end
end
```

#### HealthText.lua - UnitHealthPercent for Text Display
```lua
function addonTable.Display.HealthTextMixin:UpdateText()
    if UnitIsDeadOrGhost(self.unit) then
        self.text:SetText("0")
    else
        local values = {
            percentage = "",
            absolute = (AbbreviateNumbersAlt or AbbreviateNumbers)(UnitHealth(self.unit)),
        }
        if UnitHealthPercent then -- Midnight APIs - returns non-secret percentage
            local value = UnitHealthPercent(self.unit, true, CurveConstants.ScaleTo100)
            values.percentage = string.format("%d%%", value)
        else
            -- Pre-Midnight: must calculate percentage manually
            local value = UnitHealth(self.unit, true)/UnitHealthMax(self.unit)*100
            values.percentage = string.format("%d%%", value)
        end
        -- Display based on user preference
        self.text:SetFormattedText("%s", values[types[1]])
    end
end
```

#### CastBar.lua - Secret-Safe Duration APIs
```lua
function addonTable.Display.CastBarMixin:ApplyCasting()
    local name, text, texture, startTime, endTime, _, _, notInterruptible, spellID = UnitCastingInfo(self.unit)
    local isChanneled = false

    if name == nil then
        name, text, texture, startTime, endTime, _, notInterruptible, spellID = UnitChannelInfo(self.unit)
        isChanneled = true
    end

    if name ~= nil then
        if C_Secrets then
            -- 12.0.0+ secret-safe duration handling
            local duration
            if isChanneled then
                duration = UnitChannelDuration(self.unit)  -- Returns secret-safe duration object
            else
                duration = UnitCastingDuration(self.unit)  -- Returns secret-safe duration object
            end
            -- SetTimerDuration accepts secret duration objects
            self.statusBar:SetTimerDuration(duration)
            self.interruptMarker:SetMinMaxValues(0, duration:GetTotalDuration())

            if self.showInterruptMarker then
                local spellID = GetInterruptSpell()
                if spellID then
                    self:SetScript("OnUpdate", function()
                        -- C_Spell.GetSpellCooldownDuration returns secret-safe duration
                        local duration = C_Spell.GetSpellCooldownDuration(spellID)
                        self.interruptMarker:SetValue(duration:GetRemainingDuration())
                        -- SetAlphaFromBoolean handles secret values at C++ level
                        self.interruptMarker:SetAlphaFromBoolean(
                            duration:IsZero(),
                            0,
                            C_CurveUtil.EvaluateColorValueFromBoolean(notInterruptible, 0, 1)
                        )
                    end)
                end
            end
        else
            -- Pre-12.0.0 handling with direct arithmetic
            self.statusBar:SetMinMaxValues(0, (endTime - startTime) / 1000)
            -- ... standard OnUpdate with GetTime() arithmetic
        end
    end
end
```

#### Auras.lua - C_UnitAuras Secret-Safe APIs
```lua
function addonTable.Display.AurasManagerMixin:FullRefresh()
    self:Reset()

    if C_UnitAuras.GetUnitAuraInstanceIDs then
        local important, crowdControl = self.GetImportantAuras()
        if self.buffsDetails then
            local all = C_UnitAuras.GetUnitAuras(self.unit, self.buffFilter, nil, self.buffSort, self.buffOrder)
            for _, aura in ipairs(all) do
                -- GetAuraApplicationDisplayCount returns a secret string during combat.
                -- Pass directly to SetText() — do NOT use Lua string/number ops on it.
                aura.applicationsString = C_UnitAuras.GetAuraApplicationDisplayCount(
                    self.unit, aura.auraInstanceID, 2, 1000
                )
                -- GetAuraDuration returns secret-safe duration object
                aura.durationSecret = C_UnitAuras.GetAuraDuration(self.unit, aura.auraInstanceID)
                aura.kind = "buffs"
                self.auraData[aura.auraInstanceID] = aura
            end
        end
    end
end
```

#### Initialize.lua - Nameplate SetAlpha(0) Pattern
```lua
-- CRITICAL: Use SetAlpha(0) instead of Hide() to keep HitTestFrame active
hooksecurefunc(NamePlateDriverFrame, "OnNamePlateAdded", function(_, unit)
    local nameplate = C_NamePlate.GetNamePlateForUnit(unit, issecure())
    if nameplate and unit and addonTable.Constants.IsMidnight then
        -- SetAlpha(0) makes UnitFrame invisible but keeps HitTestFrame active
        -- This is ESSENTIAL for click targeting in 12.0.0+
        nameplate.UnitFrame:SetAlpha(0)

        -- Move aura frames to hidden parent (they don't affect click targeting)
        nameplate.UnitFrame.AurasFrame.DebuffListFrame:SetParent(addonTable.hiddenFrame)
        nameplate.UnitFrame.AurasFrame.BuffListFrame:SetParent(addonTable.hiddenFrame)

        -- Hook SetAlpha to prevent Blizzard from making it visible again
        hooksecurefunc(nameplate.UnitFrame, "SetAlpha", function(UF)
            if not UF:IsForbidden() then
                UF:SetAlpha(0)
            end
        end)
    end
end)
```

#### StatusBar FillStyle Enum Pattern
```lua
-- 12.0.0 uses Enum, pre-12.0.0 uses strings
function frame:SetReverseFill(value)
    if value then
        if addonTable.Constants.IsMidnight then
            frame.statusBar:SetFillStyle(Enum.StatusBarFillStyle.Reverse)
        else
            frame.statusBar:SetFillStyle("REVERSE")
        end
    else
        if addonTable.Constants.IsMidnight then
            frame.statusBar:SetFillStyle(Enum.StatusBarFillStyle.Standard)
        else
            frame.statusBar:SetFillStyle("STANDARD")
        end
    end
end
```

### Blizzard UI Patterns

**Pools.lua** - Rejecting secrets in pools:
```lua
function ObjectPoolBaseMixin:Release(object, canFailToFindObject)
    -- Prevent secrets from being stored as keys
    if issecretvalue(object) then
        assertsafe(false, "attempted to release a secret value into a pool: %s", tostring(object))
        return active
    end
end
```

**SecureTypes.lua** - SecureMap preventing secret storage:
```lua
function SecureMap:SetValue(key, value)
    assert(not issecretvalue(key), "attempted to store a secret key in a SecureMap")
    assert(not issecretvalue(value), "attempted to store a secret value in a SecureMap")
    self.tbl[key] = value
end
```

**Dump.lua** - Handling secrets in /dump:
```lua
if (IsSimpleType(valType)) then
    local format = issecretvalue(val) and FORMATS.simpleValueSecret or FORMATS.simpleValue
    context:Write(string.format(format, firstPrefix, prepSimple(val, context), suffix))
elseif (not canaccessvalue(val) or (valType == "table" and not canaccesstable(val))) then
    context:Write(string.format(FORMATS.opaqueTypeValSecret, firstPrefix, valType, suffix))
end
```

**RestrictedInfrastructure.lua** - Blocking secrets in restricted tables:
```lua
if issecretvalue(k) or issecretvalue(v) then
    error("Attempted to use a secret key or value in a restricted table assignment")
end
```

---

## Migration Checklist

### For Existing Addons

1. [ ] Search for all uses of `UnitHealth`, `UnitHealthMax`, `UnitPower`, `UnitPowerMax`
2. [ ] Add `issecretvalue` checks before arithmetic/comparisons
3. [ ] Replace arithmetic with `UnitHealthPercent`/`UnitPowerPercent` where possible
4. [ ] Use native StatusBar frames for health/power display when possible
5. [ ] Search for `C_ActionBar` usage and add secret guards
6. [ ] Search for aura filtering by `name`, `spellId`, or other data fields -- these CANNOT work during combat
7. [ ] Replace `AuraUtil.FindAuraByName()` and `C_UnitAuras.GetUnitAuraBySpellID()` -- both return nil during combat
8. [ ] Replace `C_Spell.GetSpellInfo("spellName")` with numeric spell ID lookups
9. [ ] Use `C_UnitAuras.GetAuraDuration()` and `C_UnitAuras.GetAuraApplicationDisplayCount()` for aura display
10. [ ] Use API-level filter strings (`HELPFUL|PLAYER`, `INCLUDE_NAME_PLATE_ONLY`) instead of Lua-side filtering
11. [ ] Test with `secretCombatRestrictionsForced` CVar enabled
12. [ ] Test in actual combat scenarios (including open world, not just instances)
13. [ ] Add version checks for `issecretvalue` existence

### For New Addons

1. [ ] Always check `issecretvalue` before operations on combat data
2. [ ] Use `UnitHealthPercent`/`UnitPowerPercent` for custom bars
3. [ ] Use native StatusBar for automatic secret handling
4. [ ] Do NOT attempt custom aura filtering by name/spellId during combat -- use API filter strings
5. [ ] Use `C_UnitAuras.GetAuraDuration()` for aura cooldown display, not `auraData.duration`/`expirationTime`
6. [ ] Use `C_UnitAuras.GetAuraApplicationDisplayCount()` for stack display, not `auraData.applications`
7. [ ] Design UI to gracefully handle unavailable data
8. [ ] Document which features may be limited during combat
9. [ ] Use numeric spell IDs (not spell name strings) for all spell lookups

---

## Quick Reference Table

### Detection Functions

| Function | Purpose | Returns |
|----------|---------|---------|
| `issecretvalue(val)` | Check if value is secret | boolean |
| `canaccessvalue(val)` | Check if caller can use value | boolean |
| `canaccessallvalues(...)` | Check if ALL values accessible | boolean |
| `hasanysecretvalues(...)` | Check if ANY value is secret | boolean |
| `canaccesssecrets()` | Check caller's secret permissions | boolean |
| `canaccesstable(tbl)` | Check if table is accessible | boolean |
| `issecrettable(tbl)` | Check if table/contents are secret | boolean |

### Manipulation Functions

| Function | Purpose | Notes |
|----------|---------|-------|
| `scrubsecretvalues(...)` | Replace secrets with nil | Safe for addon use |
| `scrub(...)` | Replace secrets + non-primitives | More aggressive |
| `mapvalues(fn, ...)` | Apply function to values | Useful for transforms |
| `secretwrap(...)` | Convert to secrets | Restricted |
| `secretunwrap(...)` | Convert from secrets | Restricted |
| `dropsecretaccess()` | Drop caller's permissions | Blizzard use |

### Secure Execution

| Function | Purpose |
|----------|---------|
| `securecallfunction(fn, ...)` | Call function securely |
| `secureexecuterange(tbl, fn, ...)` | Secure table iteration |
| `forceinsecure()` | Force tainted execution |

### SecureTypes Containers

| Type | Creator | Use Case |
|------|---------|----------|
| SecureMap | `SecureTypes.CreateSecureMap()` | Key-value storage |
| SecureArray | `SecureTypes.CreateSecureArray()` | Indexed storage |
| SecureStack | `SecureTypes.CreateSecureStack()` | LIFO stack |
| SecureValue | `SecureTypes.CreateSecureValue(v)` | Single value wrapper |
| SecureNumber | `SecureTypes.CreateSecureNumber(v)` | Number with arithmetic |
| SecureBoolean | `SecureTypes.CreateSecureBoolean(v)` | Boolean wrapper |
| SecureFunction | `SecureTypes.CreateSecureFunction()` | Function wrapper |

---

## See Also

- [12_API_Migration_Guide.md](12_API_Migration_Guide.md) - Full migration guide including secret values in combat context
- [01_API_Reference.md](01_API_Reference.md) - General API reference
- [10_Advanced_Techniques.md](10_Advanced_Techniques.md) - Advanced addon patterns

---

*Last Updated: 2026-02-05*
*WoW Version: 12.0.0 (Midnight)*
*Added: Comprehensive aura data secret values documentation (all fields secret except auraInstanceID), C_Spell.GetSpellInfo spell-name-lookup removal, pre-combat caching limitations, practical implications table for aura addon developers*
