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
    - [Unit Tooltip Identity: GameTooltip:GetUnit() and the inline-helper technique](#unit-tooltip-identity-gametooltipgetunit-returns-secrets-and-the-helper-has-no-internal-guards)
10. [APIs That Accept Secret Values](#apis-that-accept-secret-values)
    - [AbbreviateNumbers (general-purpose secret-safe formatter)](#abbreviatenumbers-general-purpose-secret-safe-numeric-formatter-1200)
    - [C_StringUtil.TruncateWhenZero / WrapString](#c_stringutiltruncatewhenzero-1200)
11. [Common Patterns and Solutions](#common-patterns-and-solutions)
    - [Pattern 10: Cache Last-Known Index for Secret Comparisons](#pattern-10-cache-last-known-index-for-ordered-comparisons-on-secret-numbers)
    - [Pattern 11: Centralized NotSecretValue / IsSecretValue Guards](#pattern-11-centralized-notsecretvalue--issecretvalue-guard-helpers)
12. [Testing and Debugging](#testing-and-debugging)
13. [Real-World Examples](#real-world-examples)
14. [CooldownViewer Frame Taint Pitfalls](#cooldownviewer-frame-taint-pitfalls)
15. [Migration Checklist](#migration-checklist)
16. [Quick Reference Table](#quick-reference-table)

---

## Overview

WoW 12.0.0 (Midnight) introduced the **Secret Values** system as part of Blizzard's "addon disarmament" initiative. This system fundamentally changes how addons interact with combat-sensitive data by making certain API return values unusable by tainted (addon) code during combat or restricted states.

**Key Concept:** Secret values are special Lua values that LOOK normal but CANNOT be used in arithmetic, comparisons, concatenations, or most other operations when called from tainted (addon) code.

**Scope:** Secret values are active during **ALL combat contexts**, including open-world combat, not just instanced content (dungeons, raids, Mythic+, PvP). Any time a player is in combat anywhere in the game, affected APIs return secret values.

**IMPORTANT: Taint Propagation Beyond Combat:** Secret values also appear in **any tainted execution context**, not exclusively during `InCombatLockdown()`. For example, addon-created frames parented to secure Blizzard frames (like `WorldMapFrame:GetCanvas()`) can propagate taint into Blizzard's tooltip code, causing "secret number value" errors even outside combat. `InCombatLockdown()` is NOT a sufficient guard against secret value errors -- always use `issecretvalue()` checks regardless of combat state.

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
  - *(12.0.1: This is now the ONLY `CooldownFrame` method that accepts secret values. `SetCooldown`, `SetCooldownFromExpirationTime`, `SetCooldownDuration`, and `SetCooldownUNIX` are restricted.)*
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
2. Limit damage meters from parsing raw combat data in real-time (see [12_API_Migration_Guide.md](12_API_Migration_Guide.md#c_damagemeter-namespace---secret-protected-usable-with-workarounds)) — though workarounds exist for display purposes
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
local chargeInfo = C_ActionBar.GetActionCharges(slot)
if canaccessvalue(chargeInfo) then
    -- Safe to use
    ProcessChargeInfo(chargeInfo)
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
local cooldownInfo = C_ActionBar.GetActionCooldown(slot)
local chargeInfo = C_ActionBar.GetActionCharges(slot)
if canaccessallvalues(cooldownInfo, chargeInfo) then
    -- All values are accessible
    ProcessActionState(cooldownInfo, chargeInfo)
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

These functions allow addon code to call Blizzard functions in a way that avoids taint propagation from the caller's context.

### securecallfunction(func, ...)

Executes `func` in **the function's own security context**, not the caller's context.

- For **Blizzard Lua functions**, this restores the callee's native security level -- useful when calling a Blizzard helper that reads protected state, so the tainted caller context doesn't propagate into the callee's Lua execution.
- **However**, for **C++ API calls**, `securecallfunction` does NOT prevent C++ event taint attribution. Events fired by C++ APIs (e.g., `QUEST_WATCH_LIST_CHANGED` from `C_QuestLog.AddWorldQuestWatch()`) are still attributed to the addon regardless of wrapping. See [12_API_Migration_Guide.md](12_API_Migration_Guide.md#what-does-not-work) for details.
- For **addon Lua functions**, it still runs as tainted because the function itself is tainted -- `securecallfunction` does not grant security, it restores the callee's native security level.

```lua
-- Signature
local result1, result2, ... = securecallfunction(func, arg1, arg2, ...)

-- Parameters:
--   func: Function to call in its own security context
--   ...: Arguments to pass to the function

-- Returns:
--   ...: Return values from the function

-- Primary use case: calling Blizzard Lua functions so their internal
-- reads of protected state execute in the callee's secure context.
-- NOTE: Does NOT prevent C++ event taint attribution (see migration guide).
securecallfunction(ShowUIPanel, SomeBlizzardPanel)

-- Useful for safe table access in tainted contexts:
securecallfunction(rawget, someTable, someKey)

-- Blizzard also uses this in SecureTypes to safely access values:
local function GetValueSecure(self)
    return self.value
end

function SomeObject:GetValue()
    return securecallfunction(GetValueSecure, self)
end
```

**Key Use Case:** Calling Blizzard APIs (e.g., `ShowUIPanel`, `C_QuestLog.AddWorldQuestWatch`, `WorldMapFrame:SetMapID()`) from addon code so their side effects execute in the secure context, preventing taint propagation through Blizzard's internal call chains.

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

### table.freeze(tbl) / table.isfrozen(tbl) (NEW in 12.0.5)

Patch 12.0.5 added Lua-level read-only table support. These functions are usable from addon code (not restricted).

```lua
-- Make a table read-only
local config = table.freeze({
    maxRetries = 3,
    timeoutSeconds = 5,
    debugMode = false,
})

-- Subsequent writes raise an error
-- config.maxRetries = 5  -- ERROR: attempt to modify a frozen table

-- Inspect
if table.isfrozen(config) then
    -- Table is read-only
end
```

**Use cases:**
- Shipping immutable constant tables between modules or to other addons (prevents accidental mutation downstream).
- Catching bugs where shared config tables are being mutated from unexpected call sites.
- Freezing lookup tables after initialization to lock their shape for the lifetime of the addon.

**Not related to secret values or taint directly** — this is a Lua-level immutability primitive. But it fits naturally here as part of the table-security story.

### Predicates Table in API Documentation (NEW in 12.0.5)

Patch 12.0.5 added a `Predicates` table to generated Lua API documentation. Each entry describes per-API restrictions — specifically, whether an API accepts secret arguments, whether it's combat-restricted, and related metadata.

**How to use it for addon development:** When you're unsure whether a given API accepts secret arguments, open the corresponding `*APIDocumentation.lua` file in `Interface\AddOns\Blizzard_APIDocumentationGenerated\` and look for the `Predicates` entries on the function you're interested in. This is the authoritative reference — more reliable than inferring from behavior.

Common `Predicates` labels you'll see in the 12.0.5 documentation:
- `SecretArguments = "AllowedWhenUntainted"` — function accepts secret arguments, but ONLY from untainted (Blizzard) code. From addon code, passing a secret will fail.
- `SecretArguments = "AllowedWhenTainted"` — function accepts secret arguments from any caller, including tainted addon code.
- `HasRestrictions = true` — function has broader restrictions; typically means combat-restricted or requires a specific security context.
- `SecretReturns` entries indicate which return values may be secret.

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

##### Defensive `pcall()` on Percentage APIs

While `UnitHealthPercent()` and `UnitPowerPercent()` return non-secret percentage values, they can still error when called with certain arguments (e.g., color curve parameters) in deeply tainted execution contexts. For maximum resilience, wrap calls in `pcall()`:

```lua
local ok, pct = pcall(UnitHealthPercent, unit, true, ScaleTo100)
if ok then
    -- use pct for color gradient
else
    pct = 0  -- fallback
end
```

This pattern is used by major addons (ElvUI/oUF) for all health/power gradient color computations.

### Unit Comparison and Identity APIs

Several unit APIs return secret values during combat that cause errors if used directly in boolean tests, comparisons, or as table keys.

| API | Returns | Secret During Combat | Notes |
|-----|---------|---------------------|-------|
| `UnitIsUnit(unit1, unit2)` | boolean | **YES** | Cannot use in `if` directly |
| `UnitIsDead(unit)` | boolean | **Sometimes** | May be secret in restricted contexts |
| `UnitName(unit)` | string | **Sometimes** | Cannot use as table key or in string ops |
| `UnitClass(unit)` | string, string | **Sometimes** | Class token may be secret |
| `UnitGUID(unit)` | string | **Sometimes** | Cannot use as table key |
| `UnitDetailedThreatSituation(unit, mob)` | isTanking, status, scaledPercent, rawPercent, threatValue | **YES** | `scaledPercent`, `rawPercent`, `threatValue` are secret |

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

> **12.0.5 update — stricter unit identity rules.** In Patch 12.0.5, several `Unit*` APIs became stricter about secret unit identities. Patterns that worked by coincidence on 12.0.1 may now return `nil` or secret:
>
> - **`UnitName(unit)`** no longer accepts secret unit tokens — passing a secret-token unit returns `nil`.
> - **`UnitSpellTargetName(unit)`** only returns names for player units.
> - **`UnitTokenFromGUID(guid)`** will NOT return `arena*`, `nameplate*`, `boss*`, `party*`, `raid*`, `targettarget`, or `focustarget` tokens when identities are secret.
> - **`UnitIsUnit(a, b)`** received a significant rewrite:
>   - Returns a SECRET value if either argument is `targettarget` or `focustarget`.
>   - Returns `nil` if the comparison between the two input tokens is forbidden.
>   - **Allowed comparisons:** either unit is `player`, `pet`, `vehicle`, `mouseover`, `target`, `softenemy`, `softfriend`, `softinteract`, `focus`, `none`, `npc`, or `questnpc`.
>   - **Also allowed:** one unit is a party/raid token (or its pet) AND the other is NOT a compound token, nameplate token, `targettarget`, or `focustarget`.
>   - **All other comparisons return nil.**
>
> Practical impact: code like `UnitIsUnit("targettarget", "player")` for "is a mob targeting me?" detection may now fail with `nil` or a secret. Prefer unit events (`UNIT_TARGET`, `UNIT_THREAT_LIST_UPDATE`) or the `softenemy`/`softfriend` tokens when you can.
>
> **12.0.5 update — player stat APIs now inherit aura secrecy.** APIs returning player stats (power, speed, attack power, etc.) now return secret values **if the auras influencing them are secret**. In 12.0.1 these were unconditionally non-secret; in 12.0.5 they may inherit secrecy from aura state. Guard accordingly.
>
> **12.0.5 update — `Ambiguate` accepts secrets.** `Ambiguate(name, context)` now accepts secret string arguments from addons. Previously this was a minor friction point when processing names returned by some combat/secret APIs.

### Action Bar APIs

All C_ActionBar functions may return secret values during combat:

| API | Returns | Notes |
|-----|---------|-------|
| `C_ActionBar.GetActionTexture(slot)` | texture | May be secret |
| `C_ActionBar.IsUsableAction(slot)` | usable | May be secret |
| `C_ActionBar.GetActionCharges(slot)` | chargeInfo | May be secret when cooldown-restricted |

### C_DamageMeter APIs

Key data fields from C_DamageMeter are SECRET during combat, but **proven workarounds exist** for building functional damage meter displays:

| Field | Status | Workaround |
|-------|--------|------------|
| `name` | SECRET during combat | `pcall(string.format, "%s", name)` extracts as text |
| `totalAmount` | SECRET during combat | `StatusBar:SetValue()` accepts directly; `pcall(string.format, "%.0f", val)` extracts as text |
| `amountPerSecond` | SECRET during combat | Same as totalAmount |
| `sourceGUID` | SECRET during combat | Not needed for display; use array index for ordering |
| `classFilename` | Always accessible | Direct use |
| `isLocalPlayer` | Always accessible | Direct use |
| `specIconID` | Always accessible | Direct use |

**After combat ends** (`PLAYER_REGEN_ENABLED` + short delay), all values become non-secret and fully readable for post-combat analysis.

#### Workarounds for Display During Combat

**1. Format secret numbers/strings into displayable text via pcall:**

`pcall(string.format, ...)` executes at the C++ level, which can format secret values into normal strings:

```lua
-- Display secret DPS value as text
local ok, text = pcall(string.format, "%.0f", source.amountPerSecond)
if ok then
    fontString:SetText(text .. " DPS")
end

-- Display secret player name
local ok, nameText = pcall(string.format, "%s", source.name)
if ok then
    nameFontString:SetText(nameText)
end
```

> **12.0.5 alternative — numeric formatter types.** Patch 12.0.5 added three new Lua formatter object types whose `Format` functions accept secret numbers natively (no `pcall` wrapper required): `AbbreviatedNumberFormatter` (e.g., "1.2M"), `NumericRuleFormatter` (locale-aware pluralization), and `SecondsFormatter` (duration/time). These share a base class `NumericFormatter`. For cooldown frames specifically, plug a formatter in via `cooldown:SetCountdownFormatter(formatter)` plus `cooldown:SetCountdownMillisecondsThreshold(seconds)` — see [13_Cooldown_Viewer_Guide.md](13_Cooldown_Viewer_Guide.md) for the cooldown-specific path. The `pcall(string.format)` pattern above remains necessary for pre-12.0.5 cross-client addons and for non-cooldown display paths that don't have formatter hooks yet.

> **12.0.5 clarification — format precision still restricted.** String-format precision / field-width modifiers (e.g., `%.5s`, `%10s`, `%.1s`) remain restricted with secret string inputs (this rule was brought forward from 12.0.5 to 12.0.1 and re-stated in 12.0.5 release notes). `%s` without a modifier continues to work via `SetFormattedText` at the C++ level.

**2. Use StatusBar for proportional bar display:**

StatusBar widgets accept secret values at the C++ level:

```lua
-- Use secret values directly for bar scaling
pcall(statusBar.SetMinMaxValues, statusBar, 0, session.maxAmount)
pcall(statusBar.SetValue, statusBar, source.totalAmount)
```

**3. Use array index for sort order:**

`GetCombatSessionFromType()` returns `combatSources` sorted by amount (highest first). The array index itself is not secret:

```lua
local session = C_DamageMeter.GetCombatSessionFromType(sessionType, meterType)
for i, source in ipairs(session.combatSources) do
    -- i=1 is highest damage/healing, i=2 is second, etc.
    -- Use index to derive a synthetic non-secret ranking value
    local syntheticValue = math.max(1, 1000 - i)
    -- source.isLocalPlayer is always accessible for highlighting
end
```

**4. Post-combat full data access:**

```lua
local frame = CreateFrame("Frame")
frame:RegisterEvent("PLAYER_REGEN_ENABLED")
frame:SetScript("OnEvent", function(self, event)
    -- Short delay to ensure secrets are fully cleared
    C_Timer.After(0.5, function()
        local session = C_DamageMeter.GetCombatSessionFromType(
            Enum.DamageMeterSessionType.LastFight,
            Enum.DamageMeterType.DamageDone
        )
        for _, source in ipairs(session.combatSources) do
            -- All values are now non-secret and fully readable
            local name = source.name
            local total = source.totalAmount
            local dps = source.amountPerSecond
            -- Full arithmetic, comparisons, and string operations work
        end
    end)
end)
```

##### APIs Now Returning Nil Instead of Secrets (12.0.1)

Some APIs now return `nil` rather than secret values when the underlying data is secret:

| API | Behavior Change |
|-----|----------------|
| `UnitCreatureID(unit)` | Returns `nil` when unit identity is secret (previously returned a secret value) |
| `Frame:GetEffectiveAlpha()` | Returns `nil` if alpha involves secret aspects |
| `StatusBar:IsStatusBarDesaturated()` | Returns `nil` if desaturation involves secret aspects |
| `Texture:IsDesaturated()` | Returns `nil` if desaturation involves secret aspects |

Handle these by checking for `nil` returns:

```lua
local creatureID = UnitCreatureID(unit)
if creatureID then  -- nil when secret, number when accessible
    -- use creatureID
end
```

### Tooltip Data

`C_TooltipInfo` line fields (`line.leftText`, `line.rightText`) can themselves be secret values — not just `FontString:GetText()` calls from the legacy path. Any addon using `C_TooltipInfo` still needs `issecretvalue()` guards before performing **any** Lua string operation on the text. The error comes from the string operation on the secret (e.g., `string.match`, `string.gsub`, `WrapTextInColorCode()`, concatenation with `..`), not from display — `FontString:SetText()` accepts secrets natively (see below).

```lua
-- WRONG: String operations on potentially secret tooltip text
local tooltipData = C_TooltipInfo.GetUnit(unit)
local line = tooltipData.lines[2]
local cleaned = line.leftText:gsub("%[(.-)%]", "%1")  -- ERROR if secret!
local colored = leftColor:WrapTextInColorCode(line.leftText)  -- ERROR if secret!

-- CORRECT: Guard with issecretvalue() BEFORE any string operation
local tooltipData = C_TooltipInfo.GetUnit(unit)
local line = tooltipData.lines[2]
if not issecretvalue(line.leftText) then
    local cleaned = line.leftText:gsub("%[(.-)%]", "%1")
    local colored = leftColor:WrapTextInColorCode(line.leftText)
end
```

**Key point:** Migrating from legacy `_G["GameTooltipTextLeft"..i]:GetText()` to `C_TooltipInfo` does NOT eliminate the need for `issecretvalue()` guards. Both paths can return secret values in tainted execution contexts.

**Important:** Secret values in tooltip data can exist at the TABLE level, not just individual fields. `line.leftColor` can be a secret TABLE (not just secret `.r`, `.g`, `.b` fields), and `line.type` can also be secret. Always check the outermost container before indexing:

```lua
local color = line.leftColor
if issecretvalue(color) then return end  -- Check the TABLE itself first
-- Only then is it safe to access color.r, color.g, color.b

local lineType = line.type
if issecretvalue(lineType) then return end  -- Check before comparing
```

#### Legacy Tooltip Font String Pattern

The legacy pattern of scanning tooltip text via global font string names (`_G["GameTooltipTextLeft"..i]`) is especially vulnerable to secret values in tainted contexts:

```lua
-- LEGACY PATTERN (vulnerable to secret values):
for i = 1, GameTooltip:NumLines() do
    local line = _G["GameTooltipTextLeft"..i]
    local text = line:GetText()
    -- Both `line` and `text` can be secret values in tainted contexts!
    if issecretvalue(text) then
        -- Cannot parse this line — skip or bail out
        break
    end
    -- Also check the line object itself
    if issecretvalue(line) then break end
end
```

**Recommended migration:** Use `C_TooltipInfo` APIs (which return structured data) or create a private scanning tooltip. The `_G["GameTooltipTextLeft"..i]` pattern reads from the shared `GameTooltip`, which inherits taint from any prior addon interaction. If you must use this pattern, always guard with `issecretvalue()` checks on both the font string object and its text content.

#### Unit Tooltip Identity: `GameTooltip:GetUnit()` Returns Secrets and the Helper Has No Internal Guards

`GameTooltip:GetUnit()` is a thin wrapper around `TooltipUtil.GetDisplayedUnit(tooltip)`. The implementation lives at `+wow-ui-source+ (12.0.0)\Interface\AddOns\Blizzard_SharedXMLGame\Tooltip\TooltipUtil.lua:34-42` and is short enough to quote in full:

```lua
function TooltipUtil.GetDisplayedUnit(tooltip)
    if tooltip:IsTooltipType(Enum.TooltipDataType.Unit) then
        local tooltipData = tooltip:GetPrimaryTooltipData()
        local guid = tooltipData.guid
        local unit = guid and UnitTokenFromGUID(guid)
        local name = unit and UnitName(unit)
        return name, unit, guid
    end
end
```

The helper returns `name, unit, guid` straight from `tooltipData` with **zero internal `canaccessvalue` / `issecretvalue` guards**. Any of the three returns can be a secret value when the underlying unit identity is restricted (PvP, restricted-aura, nameplate, party/raid token contexts — see the [Unit Comparison and Identity APIs](#unit-comparison-and-identity-apis) section for the broader rules on which unit tokens are secret-restricted, and especially the 12.0.5 update tightening `UnitName` and `UnitTokenFromGUID`).

##### Why the obvious fix fails

This is a case the existing "use `issecretvalue` before string ops" rule does **NOT** fully cover. A typical reader following that rule would write something like:

```lua
-- WRONG: looks correct, throws anyway
local name, unit = tooltip:GetUnit()
if unit ~= "none" and UnitExists(unit) and UnitIsBattlePet(unit) then
    -- ... use name and unit ...
end
```

The comparison `unit ~= "none"` itself throws when `unit` is a secret string, **before** any `issecretvalue` guard you might add afterward could run. Adding `if not issecretvalue(unit) then ...` after the multi-return is too late: the multi-return has already produced the secret value, and the very next operation (`unit ~= "none"`, or `name .. ":"`, or `unitTable[name]`) taints. The error surfaces at the comparison/concat/index, not at the `GetUnit()` call itself.

You also can't rescue this by guarding only the final `name`/`unit` returns either, because the helper performed each chained API call (`tooltipData.guid` → `UnitTokenFromGUID(guid)` → `UnitName(unit)`) inside its own body without any short-circuit. By the time the helper returns, the damage is done — secret intermediates have already been chained through API boundaries that may, in some Blizzard internal paths, propagate taint further.

##### The canonical pattern: inline-reconstruct with per-step guards

The fix is to **replace the helper call with its own source body** in your addon, then insert `canaccessvalue()` guards on each intermediate before feeding it to the next API or comparing it:

```lua
local tooltipData, guid, unit, name
if tooltip:IsTooltipType(Enum.TooltipDataType.Unit) then
    tooltipData = tooltip:GetPrimaryTooltipData()
    if tooltipData and canaccessvalue(tooltipData) then
        guid = tooltipData.guid
        if guid and canaccessvalue(guid) then
            unit = UnitTokenFromGUID(guid)
            if unit and canaccessvalue(unit) then
                name = UnitName(unit)
            end
        end
    end
end

-- Now safe — `unit` and `name` are either nil or non-secret accessible values
if unit and unit ~= "none" and UnitExists(unit) and UnitIsBattlePet(unit) then
    -- ... use name and unit ...
end
```

Why each step's guard matters:

| Step | Why the guard is needed |
|------|------------------------|
| `canaccessvalue(tooltipData)` | `GetPrimaryTooltipData()` can return a secret table when the tooltip's primary data is sourced from a restricted unit. Indexing a secret table for `.guid` taints. |
| `canaccessvalue(guid)` | `tooltipData.guid` can be a secret string. Passing a secret string to `UnitTokenFromGUID` and then doing anything with the result can chain taint. |
| `canaccessvalue(unit)` | `UnitTokenFromGUID(guid)` can return a secret unit token (especially for `arena*`, `nameplate*`, `boss*`, `party*`, `raid*`, `targettarget`, `focustarget` — see the 12.0.5 stricter rules in [Unit Comparison and Identity APIs](#unit-comparison-and-identity-apis)). Comparing a secret token against a literal string (`unit ~= "none"`) is the operation that throws. |
| `canaccessvalue(name)` (implicit, via the chain) | `UnitName(unit)` can return a secret string. Concatenation, table-keying, or string-format on the name will throw. The chain stops here naturally because nothing further is done with `name` until the calling code uses it — at which point `name` is either nil or accessible. |

##### General technique: inline the Blizzard helper when its intermediates are secret-prone

The pattern above generalises beyond just `GameTooltip:GetUnit()`. **When a Blizzard convenience helper bundles multiple operations on potentially-secret values into a single call (multi-return or chained internal lookups), and the helper has no internal `canaccessvalue` guards of its own, you sometimes need to replace the helper call with its own source body in your addon**, then insert per-step `canaccessvalue` guards on each intermediate. This is more thorough than guarding only the final return values, because the comparison / concat / format that taints often runs on an intermediate, not the final value.

**When to apply this technique:**
- Any time you find a Blizzard helper that returns "ready-to-use" data derived from secret-prone unit, aura, or identity APIs.
- Open the helper's source file in `+wow-ui-source+ (12.0.0)\Interface\AddOns\` and check whether it guards its intermediates.
- If there are no internal `canaccessvalue` / `issecretvalue` calls visible in the helper body, it is a candidate for inlining.

**When NOT to apply it:**
- Helpers that operate purely on non-secret data (UI math, color conversion, string formatting on user-authored input).
- Helpers that already pass their inputs straight to engine APIs with `AllowedWhenTainted` (e.g., `FontString:SetText`, `StatusBar:SetValue`) — those handle secrets natively at the C++ level.
- Helpers wrapped by Blizzard with `securecallfunction` or that route through `TooltipDataProcessor` callbacks — those are designed for tainted callers.

Don't inline reflexively. Verify the helper actually consumes secrets first; otherwise you are duplicating Blizzard code for no benefit and adding maintenance debt across patches.

**Source-of-truth for checking helpers:** read the helper's `.lua` file under `+wow-ui-source+ (X.Y.Z)\Interface\AddOns\` to see whether it guards intermediates. The path layout under `+wow-ui-source+` mirrors the live `Interface\AddOns\` tree.

##### Provenance / canonical attribution

This pattern is taken from ArkInventory v31212.01 (one of the most-installed inventory addons), specifically the `Core\ArkInventoryTooltip.lua:2032-2048` fix released in response to issue #2147. ArkInventory's author landed on the inline-reconstruct approach after observing repeat real-world bug reports against the addon's unit-tooltip hook (`HookOnTooltipSetUnit`) — the prior code used `tooltip:GetUnit()` directly and threw secret-value errors on hover over restricted-identity units. Treat ArkInventory as a canonical reference for taint-handling patterns in inventory and tooltip-heavy contexts.

##### Provenance caveats — verified vs. observed

To be precise about what is established fact vs. empirical observation:

- **Verified from Blizzard source:** The `TooltipUtil.GetDisplayedUnit` helper at `+wow-ui-source+ (12.0.0)\Interface\AddOns\Blizzard_SharedXMLGame\Tooltip\TooltipUtil.lua:34-42` has no internal `canaccessvalue` or `issecretvalue` guards. `GameTooltip:GetUnit()` at `+wow-ui-source+ (12.0.0)\Interface\AddOns\Blizzard_GameTooltip\Mainline\GameTooltip.lua:1018-1019` is a one-line forwarder to it.
- **Empirically observed (not exhaustively reproduced):** The frequency with which the un-guarded `tooltip:GetUnit()` path actually throws for any given addon is workload-dependent. ArkInventory's author observed it firing routinely against unit tooltips in their addon's code path; other addons may not hit the same throw rate depending on which tooltips they hook and how often the hover targets carry secret identities.

Do not overstate this as "every call to `tooltip:GetUnit()` will fail." The accurate framing is: the helper has no internal guards, so the failure mode exists and will surface intermittently in the field. Apply the inline pattern defensively in any addon that hooks unit tooltips and processes the returned `name` / `unit` / `guid`.

#### Cross-Client Compatibility

For addons that support both retail (12.0.0+) and classic clients, the global `issecretvalue()` function only exists in retail. Wrap it in a safe function to avoid errors on classic:

```lua
-- Safe wrapper for cross-client addons
function MyAddon.SafeIsSecretValue(value)
    if issecretvalue then
        return issecretvalue(value)
    end
end

-- Usage in tooltip scanning:
local text = line.leftText  -- or _G["MyTooltipTextLeft"..i]:GetText()
if not MyAddon.SafeIsSecretValue(text) then
    -- Safe to perform string operations
    local cleaned = text:gsub("%[(.-)%]", "%1")
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

### Numeric Formatters (NEW in 12.0.5)

Patch 12.0.5 added three Lua-level formatter object types whose `Format` functions accept secret numbers natively via duration objects. They share a base class `NumericFormatter`. These replace the `pcall(string.format)` workaround for display paths that support pluggable formatters.

| Formatter | Use case | Example output |
|-----------|----------|----------------|
| `AbbreviatedNumberFormatter` | Human-readable abbreviations | "1.2M", "34.5K" |
| `NumericRuleFormatter` | Locale-aware pluralization and rule-based formatting | varies by locale/rule |
| `SecondsFormatter` | Duration / time formatting | "3m 42s", "0:42" |

**Plug-in points:** Any API that accepts a `NumericFormatter` (including nil-to-clear). The most visible in 12.0.5 is cooldown frames:

```lua
-- 12.0.5+ cooldown countdown text configuration
cooldown:SetCountdownFormatter(SecondsFormatter)         -- accepts secret durations
cooldown:SetCountdownMillisecondsThreshold(10)           -- decimal rendering under 10s
```

See [13_Cooldown_Viewer_Guide.md](13_Cooldown_Viewer_Guide.md) for the cooldown-specific usage and cross-client fallback pattern, and [12_API_Migration_Guide.md](12_API_Migration_Guide.md) for the full 12.0.5 delta.

**Charge duration APIs gained `ignoreGCD` parameter (12.0.5):** `C_Spell.GetSpellChargeDuration(spellID, ignoreGCD)`, `C_ActionBar.GetActionChargeDuration(slot, ignoreGCD)`, and `C_SpellBook.GetSpellBookItemChargeDuration(spellBookItemSlotIndex, spellBookItemType, ignoreGCD)` accept an `ignoreGCD` boolean to exclude the global cooldown from the returned duration. At max charges, these APIs now consistently return a zero-span duration object (treated as fully elapsed by the framework).

### AbbreviateNumbers: General-Purpose Secret-Safe Numeric Formatter (12.0.0)

`AbbreviateNumbers(value, breakpoint)` is a **general-purpose** numeric formatter marked `SecretArguments = "AllowedWhenTainted"` in Blizzard's API documentation (`Blizzard_APIDocumentationGenerated/LocalizationDocumentation.lua`). It accepts secret numbers as input and returns a **non-secret** formatted string. This makes it strictly more powerful than the `UnitHealthPercent` / `UnitPowerPercent` escape hatches (see [Non-Secret Alternatives](#non-secret-alternatives)) — those are limited to two specific APIs, whereas `AbbreviateNumbers` works on any numeric (crit rating, mastery bonus, movement speed, mana regen, etc.).

**Recommendation:** When you need to display a ratio, percentage, or formatted number derived from an arithmetic expression that might involve secret operands, reach for `AbbreviateNumbers` **first**. Fall back to `UnitHealthPercent` / `UnitPowerPercent` only for direct health/power display.

**Signature and `breakpointData` table shape:**

```lua
-- AbbreviateNumbers(value, breakpoint) -> formatted string
-- breakpoint = { breakpointData = { <NumberAbbrevData entries> }, locale = <optional>, config = <optional cached> }
-- Each NumberAbbrevData entry:
--   breakpoint              number  -- value threshold at or above which this entry applies
--   abbreviation            cstring -- abbreviation name (global string lookup if abbreviationIsGlobal=true)
--   significandDivisor      number  -- value / significandDivisor = the displayed significand
--   fractionDivisor         number  -- controls decimal precision (10^decimalPlaces)
--   abbreviationIsGlobal    bool    -- false to use raw abbreviation string; true to look up in globals
```

The math: `displayed = (value / significandDivisor) / fractionDivisor`, then `fractionDivisor` scales the display digits back so decimal places appear correctly. Pick `significandDivisor` so that `value / significandDivisor` equals the target number you want to show.

**Worked example — movement speed as percentage of base:**

```lua
local BASE_MOVEMENT_SPEED = BASE_MOVEMENT_SPEED  -- 7 yards/sec

-- We want: speed=7 -> "100", speed=3.5 -> "50", speed=10.5 -> "150"
-- Formula: percent = speed * 100 / 7 = speed / (7 * 0.01)
-- So significandDivisor = BASE_MOVEMENT_SPEED * 0.01

local data = {
    breakpoint = 0,
    abbreviation = '',
    fractionDivisor = 1,              -- integer display (no decimals)
    significandDivisor = BASE_MOVEMENT_SPEED * 0.01,
    abbreviationIsGlobal = false,
}
local breakpoint = { breakpointData = { data } }

-- Later, in OnEvent (speed may be secret in combat):
local speed = GetUnitSpeed("player")
local percent = AbbreviateNumbers(speed, breakpoint)  -- Returns "100" etc. as a non-secret string
fontString:SetText(percent .. "%")                    -- Concatenation works (percent is not secret)
```

**Worked example — mastery bonus as percentage:**

```lua
local data = {
    breakpoint = 0,
    abbreviation = '',
    fractionDivisor = 1,
    significandDivisor = 1,           -- placeholder; set per-frame because coeffect varies
    abbreviationIsGlobal = false,
}
local breakpoint = { breakpointData = { data } }

local function UpdateMastery()
    local _, coeffect = GetMasteryEffect()
    local rating     = GetCombatRatingBonus(CR_MASTERY)
    if issecretvalue and not issecretvalue(coeffect) then
        -- rating * coeffect = displayed bonus percentage
        data.significandDivisor = 1 / coeffect
    end
    local text = AbbreviateNumbers(rating or 0, breakpoint)  -- safe even if rating is secret
    fontString:SetText(text .. "%")
end
```

**Pattern notes:**
- Mutate the per-addon `data` table live (e.g., when user changes `decimalLength`: `data.fractionDivisor = 10 ^ decimalLength`). Retain the same outer `breakpoint` object; Blizzard caches against it.
- `breakpointData` can hold multiple entries sorted largest-first for size-gated abbreviations (e.g., ">=1M show 'M', >=1K show 'K'"). For simple ratio formatting a single `breakpoint = 0` entry is sufficient.
- Returns a string — no further `format()` or arithmetic is needed for display.
- Used in production by ElvUI 15.13+ (MovementSpeed, Mastery, AttackPower, ManaRegen, Versatility datatexts; `speed:percent*` tags) as the taint-safe replacement for `value / constant * 100`.

### C_StringUtil.TruncateWhenZero (12.0.0)

`C_StringUtil.TruncateWhenZero(number) -> string` is marked `SecretArguments = "AllowedWhenTainted"` in `Blizzard_APIDocumentationGenerated/StringUtilDocumentation.lua`. Blizzard's documentation: *"Formats the given number to a string as an integer (rounding down). If the integer is zero, returns an empty string."*

This is the **taint-safe replacement for `val > 0 and tostring(val) or ""`** — the `val > 0` comparison errors on a secret `val`, but `TruncateWhenZero` accepts the secret natively and returns a non-secret string.

```lua
-- OLD (breaks on secret val):
local text = val > 0 and format("%.0f", val) or ""

-- NEW (12.0.0+, secret-safe):
local text = C_StringUtil.TruncateWhenZero(val)  -- "42" or "" ; never errors
fontString:SetText(text)
```

**Companion API — `C_StringUtil.WrapString`:** Also `AllowedWhenTainted`. Signature: `C_StringUtil.WrapString(infix, prefix, suffix) -> string`. Returns `prefix .. infix .. suffix` when infix is non-empty, else an empty string. This is the taint-safe replacement for `prefix and infix and suffix and (prefix..infix..suffix) or ""` with conditional wrapping. Used internally by oUF's `CreateTagFunc` for secret-string tag results.

```lua
-- Conditional " (count)" suffix when count is secret and may be zero-display:
local countText = C_StringUtil.TruncateWhenZero(count)           -- "" or "N"
local suffix    = C_StringUtil.WrapString(countText, " (", ")") -- "" or " (N)"
fontString:SetText(baseLabel .. suffix)  -- concatenation is safe: suffix is non-secret
```

See [Quick Reference Table](#quick-reference-table) for consolidated listing with `AbbreviateNumbers`.

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

##### `string.format` Precision Specifiers Restricted (12.0.1)

As of 12.0.1, `string.format` precision specifiers (e.g., `"%.1s"`, `"%.3s"`) are restricted with secret string inputs. This prevents addons from extracting partial secret string content character-by-character:

```lua
-- BROKEN in 12.0.1+:
local firstChar = format("%.1s", secretString)  -- Error

-- Still works (full string, no truncation):
format("%s", secretString)  -- OK (but result is still secret)

-- For display, use SetFormattedText which handles secrets at C++ level:
fontString:SetFormattedText("%s", secretValue)  -- OK
```

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

During combat, most aura data fields are secret. Several classification booleans are non-secret (and more became non-secret in 12.0.5 — see the update note below).

| Aura Data Field | Secret During Combat? | Notes |
|-----------------|----------------------|-------|
| `auraInstanceID` | **NO** | Stable instance identifier. See 12.0.5 re-randomization note below. |
| `isHelpful` | **NO** (12.0.5+) | Buff flag. Safe to read/compare in Lua during combat as of 12.0.5. |
| `isHarmful` | **NO** (12.0.5+) | Debuff flag. Safe in 12.0.5+. |
| `isRaid` | **NO** (12.0.5+) | Raid-applicable flag. Safe in 12.0.5+. |
| `isNameplateOnly` | **NO** (12.0.5+) | Nameplate-visibility-only. Safe in 12.0.5+. |
| `isFromPlayerOrPlayerPet` | **NO** (12.0.5+) | "My" aura flag. Safe in 12.0.5+. |
| `name` | **YES** | Cannot filter/compare by name |
| `spellId` | **YES** | Cannot filter/compare by spell ID |
| `icon` | **YES** | Cannot use for custom icon logic |
| `duration` | **YES** | Cannot do arithmetic |
| `expirationTime` | **YES** | Cannot calculate remaining time |
| `sourceUnit` | **YES** | Cannot identify caster |
| `dispelName` | **YES** | Cannot filter by dispel type |
| `applications` (stacks) | **YES** | Cannot read stack count |
| `timeMod` | **YES** | Cannot read time modification |
| `points` | **YES** | Cannot read aura values |

> **12.0.5 update — classification fields non-secret.** As of Patch 12.0.5, the five classification booleans above (`isHelpful`, `isHarmful`, `isRaid`, `isNameplateOnly`, `isFromPlayerOrPlayerPet`) can be read from tainted Lua during combat. This lets addons perform Lua-side classification filtering ("is this my aura?", "is this a nameplate-only aura?") on results from `C_UnitAuras.GetAuraDataByIndex` / `GetAuraDataByAuraInstanceID` without needing additional API calls with filter strings. The identifier fields (`name`, `spellId`, `icon`, durations) remain secret. See [12_API_Migration_Guide.md](12_API_Migration_Guide.md) for the full 12.0.5 delta.
>
> **12.0.5 update — `auraInstanceID` re-randomizes at encounter boundaries.** `auraInstanceID` values are now re-randomized when entering encounters, Mythic+, and PvP. Any instance-id-keyed cache MUST be invalidated on `ENCOUNTER_START`, `CHALLENGE_MODE_START`, and arena/battleground boundary events.

#### All Aura APIs Return Secret Data During Combat

Every method of querying aura data is affected. On 12.0.5+, classification booleans are an exception — see the note above.

```lua
-- During combat, the identifier and numeric fields on these aura objects are secret.
-- Non-secret fields: auraInstanceID (all clients), isHelpful/isHarmful/isRaid/
-- isNameplateOnly/isFromPlayerOrPlayerPet (12.0.5+).

-- Method 1: GetUnitAuras (bulk query)
local auras = C_UnitAuras.GetUnitAuras(unit, filter)
for _, aura in ipairs(auras) do
    -- aura.auraInstanceID       -- NON-SECRET (usable on all clients)
    -- aura.isHelpful            -- NON-SECRET on 12.0.5+; secret on older
    -- aura.isHarmful            -- NON-SECRET on 12.0.5+; secret on older
    -- aura.name                 -- SECRET (unusable for comparisons)
    -- aura.spellId              -- SECRET (unusable for comparisons)
    -- aura.icon                 -- SECRET
    -- aura.duration             -- SECRET
    -- Other identifier/numeric fields -- SECRET
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

### C_LossOfControl.GetActiveLossOfControlDuration() (NEW in 12.0.1)

Returns a secret-safe duration object for loss-of-control cooldown display:
```lua
local duration = C_LossOfControl.GetActiveLossOfControlDuration(unitToken, index)
cooldown:SetCooldownFromDurationObject(duration)
```

### GetTotemDuration() (NEW in 12.0.1)

Returns a secret-safe duration object for totem cooldown display:
```lua
local duration = GetTotemDuration(slot)
cooldown:SetCooldownFromDurationObject(duration)
```

### Manual Duration Objects for Item Cooldowns

Unlike spells (`C_Spell.GetSpellCooldownDuration()`) and LoC effects (`C_LossOfControl.GetActiveLossOfControlDuration()`), item cooldowns have no native Duration API. Synthesize a DurationObject manually using `C_DurationUtil.CreateDuration()`:

```lua
local start, duration, enable, modRate = C_Container.GetItemCooldown(itemID)
local dur = C_DurationUtil.CreateDuration()
dur:SetTimeFromStart(start, duration, modRate or 1)
cooldown:SetCooldownFromDurationObject(dur)
```

`SetTimeFromStart(startTime, duration, modRate)` is a convenience method that configures all timing properties in one call (alternative to calling `SetStartTime()` and `SetDuration()` separately).

### SetCooldownFromDurationObject and SetUseAuraDisplayTime

**IMPORTANT:** Do NOT call `SetUseAuraDisplayTime(true)` when using `SetCooldownFromDurationObject()`. `SetCooldownFromDurationObject` already handles aura-style display timing internally. Combining both causes the countdown text to display **1 second LESS** than expected (e.g., showing "4" when there are 5 seconds remaining).

| Method | SetUseAuraDisplayTime? | Why |
|--------|----------------------|-----|
| `SetCooldownFromDurationObject(durationObj)` | **NO** -- Do NOT set it | Timing handled internally by the duration object |
| `SetCooldown(start, duration)` for **auras** | **YES** -- Set to `true` | Adjusts floor-to-ceil rounding so "5s" shows as "5" not "4" |
| `SetCooldown(start, duration)` for **cooldowns** | **NO** -- Do NOT set it | Floor rounding is correct for ability cooldowns |

> **12.0.1 Restriction:** `SetCooldown(start, duration)` is now restricted from tainted addon code with secret values. For aura cooldowns, use `SetCooldownFromDurationObject` with `C_UnitAuras.GetAuraDuration()`. For spell/action cooldowns, use `SetCooldownFromDurationObject` with `C_Spell.GetSpellCooldownDuration()`.

```lua
-- CORRECT: Aura cooldown with duration object (Platynator pattern)
local duration = C_UnitAuras.GetAuraDuration(unit, auraInstanceID)
cooldown:SetCooldownFromDurationObject(duration)
-- Do NOT call cooldown:SetUseAuraDisplayTime(true) here!

-- RESTRICTED in 12.0.1: SetCooldown with secret values (Blizzard nameplate pattern)
-- This previously worked but is now blocked for tainted addon code with secrets.
-- Use SetCooldownFromDurationObject instead.
local expirationTime = auraData.expirationTime
local auraDuration = auraData.duration
cooldown:SetUseAuraDisplayTime(true)
cooldown:SetCooldown(expirationTime - auraDuration, auraDuration)  -- RESTRICTED in 12.0.1

-- WRONG: Combining both (countdown shows 1 second less than expected)
cooldown:SetUseAuraDisplayTime(true)  -- Don't do this with duration objects!
cooldown:SetCooldownFromDurationObject(duration)
```

**Cooldown API Non-Secret Fields (12.0.1):** `isEnabled`, `maxCharges`, and the new `isActive` boolean are all non-secret. Use `isActive` to determine whether a cooldown display should be rendered, without needing to check secret `startTime`/`duration` values.

> **See also:** [Cooldown Viewer Guide](13_Cooldown_Viewer_Guide.md#cooldown-tracking) for how Blizzard's own Cooldown Viewer handles secret-safe cooldown display.

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
    local texture = C_ActionBar.GetActionTexture(slot)

    if issecretvalue(texture) then
        -- Queue for later
        pendingUpdates[slot] = true
        return
    end

    ProcessActionButton(slot, texture)
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

### Pattern 5: Secret-Safe Absorb Text Display

Display absorb values that may be secret during combat. The secret check must happen
**before** any Lua arithmetic, concatenation, or function calls on the value:

```lua
function addonTable.Display.AbsorbTextMixin:UpdateText()
    if UnitIsDeadOrGhost(self.unit) then
        self.text:SetText("+0")
        self.text:SetAlpha(0)
    else
        local raw = UnitGetTotalAbsorbs(self.unit)
        if issecretvalue and issecretvalue(raw) then
            -- Secret: use SetFormattedText which handles secrets at C++ level
            self.text:SetFormattedText("+%d", raw)
            -- SetAlpha accepts secret values natively
            self.text:SetAlpha(raw)
        else
            -- Not secret: safe to use Lua functions on the value
            local absolute = (AbbreviateNumbersAlt or AbbreviateNumbers)(raw)
            self.text:SetText("+" .. absolute)
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
- `SetWidth(0)` on the first FontString so its RIGHT edge matches the text edge (see [03_UI_Framework.md](03_UI_Framework.md) - FontString Width and Anchoring)
- Use `SetFormattedText("(%s)", secret)` to add formatting around secret values without Lua concatenation
- Match fonts between FontStrings with `secretText:SetFont(staticText:GetFont())`

### Pattern 8: C++ APIs as Secret-Safe Fallbacks for Lua Lookups

When a unit API returns a secret value, you cannot use it as a Lua table key -- `RAID_CLASS_COLORS[classToken]` will error with "attempt to index with secret value." However, C++ API functions accept secret arguments natively. Use the C++ equivalent as a fallback:

```lua
local _, className = UnitClass(unit)
local r, g, b
if issecretvalue(className) then
    -- C++ API handles secrets natively
    local color = C_ClassColor.GetClassColor(className)
    if color then
        r, g, b = color:GetRGB()
    end
else
    -- Lua table lookup works when value is not secret
    local color = RAID_CLASS_COLORS[className]
    if color then
        r, g, b = color.r, color.g, color.b
    end
end
```

This pattern applies broadly: whenever a Lua-side operation (table lookup, string manipulation, arithmetic) fails with a secret value, check if an equivalent C++ API exists that can handle the same input.

### Pattern 9: Table-Level Secret Values

Secret values can exist at ANY level -- the table itself, individual fields, or nested fields. Always check the outermost container first:

```lua
-- The color TABLE itself might be secret (not just color.r)
local color = line.leftColor
if issecretvalue(color) then return end  -- Check table before indexing
if issecretvalue(color.r) then return end  -- Then check fields

-- Common with tooltip data where entire structures can be tainted
-- Also applies to any API that returns tables during combat
```

### Pattern 10: Cache Last-Known Index for Ordered Comparisons on Secret Numbers

When you need to find the max/min across several values that may be secret (e.g., "which of melee/ranged/spell crit is highest?"), the naive `if a > b` comparison errors during combat. The idiom used by ElvUI v15.13 (Crit.lua) is to **cache the last successfully-resolved index in a module-level upvalue** and, when the inputs are secret, skip the comparison entirely and re-display the current value at the cached index.

```lua
-- Module-level state (retained across calls):
local ratingIndex, spellIndex = CR_CRIT_MELEE, 2  -- defaults

local function OnEvent(panel)
    local critMelee  = GetCritChance()
    local minSpell   = GetSpellCritChance(2)  -- holy

    if issecretvalue and issecretvalue(minSpell) then
        -- Secret phase: DO NOT compare. Use cached ratingIndex from last safe phase.
        local text
        if     ratingIndex == CR_CRIT_MELEE  then text = critMelee
        elseif ratingIndex == CR_CRIT_RANGED then text = GetRangedCritChance()
        else                                     text = GetSpellCritChance(spellIndex)
        end
        panel.text:SetFormattedText(displayString, text)
        return
    end

    -- Safe phase: scan and update cached indices.
    for i = 3, MAX_SPELL_SCHOOLS do
        local chance = GetSpellCritChance(i)
        if chance < minSpell then
            minSpell, spellIndex = chance, i
        end
    end
    local critRanged = GetRangedCritChance()
    local best
    if     minSpell   >= critRanged and minSpell >= critMelee then ratingIndex, best = CR_CRIT_SPELL,  minSpell
    elseif critRanged >= critMelee                            then ratingIndex, best = CR_CRIT_RANGED, critRanged
    else                                                           ratingIndex, best = CR_CRIT_MELEE,  critMelee end
    panel.text:SetFormattedText(displayString, best)
end
```

**Why this works:** `ratingIndex` is always a plain integer set in a previous non-combat / non-secret update, so it's never secret. When secret values block the comparison, you still know *which* value to show — you just can't re-rank the categories until out of combat. The display stays accurate for the dominant category (which rarely changes moment-to-moment during a fight).

**Applicability:** Any "pick the highest / lowest of N secret numbers" logic. Common cases: best crit school, highest cooldown among tracked spells, top-DPS target selection, highest-threat unit. Do the ordering in your last known good state and cache the index; display the current value at that index during the secret phase.

### Pattern 11: Centralized NotSecretValue / IsSecretValue Guard Helpers

Sprinkling `issecretvalue(x)` throughout the codebase works but creates noise at every guard site. The idiom adopted by ElvUI (v15.13) and oUF is to define thin wrappers at the library level once, then use them liberally at arithmetic/string/comparison boundaries:

```lua
-- In a library init file (once):
function oUF:IsSecretValue(value)
    return issecretvalue and issecretvalue(value)
end

function oUF:NotSecretValue(value)
    return not oUF:IsSecretValue(value)
end

-- Optionally re-expose on your addon's main table for terseness:
E.NotSecretValue = oUF.NotSecretValue
E.IsSecretValue  = oUF.IsSecretValue
```

This gives you two benefits:
1. **Cross-client safety** — `issecretvalue` doesn't exist pre-12.0.0. The wrapper returns false there, so `if E:NotSecretValue(x) then <do arithmetic> end` degrades gracefully on Classic / Mists / TBC / Wrath.
2. **Uniform guard idiom** — call sites read naturally: `if E:NotSecretValue(speed) and speed > 0 then ...`. The Lua short-circuit ensures the comparison is only evaluated when safe.

**Where to place guards (from ElvUI v15.13 real-world usage):**

| Boundary | Example | Notes |
|---|---|---|
| Before arithmetic | `if E:NotSecretValue(baseAP) then totalAP = baseAP + posAP + negAP end` | Any `+` `-` `*` `/` `%` on a secret errors |
| Before comparison | `if E:NotSecretValue(speed) and speed > 0 then ...` | `>` `<` `>=` `<=` errors on secrets |
| Before string ops | `local lower = E:NotSecretValue(text) and strlower(text)` | `strlower`, `strsub`, `utf8sub`, `==` on strings error |
| Before table indexing | `if E:NotSecretValue(className) then return RAID_CLASS_COLORS[className] end` | Index with secret errors; fall back to `C_ClassColor.GetClassColor()` |
| Before `format`/`string.format` | See [Pattern 8](#pattern-8-c-apis-as-secret-safe-fallbacks-for-lua-lookups) | Use `FontString:SetFormattedText` for display instead |

The intent is: **treat every arithmetic or string operation that crosses the addon/API boundary as a potential secret-value site**, and guard explicitly. This is verbose but predictable — matches the "guard at the edge, trust the interior" principle for taint hygiene.

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

-- Check that line and its text are accessible (not secret)
local canUseLine = false
if not issecretvalue then
    -- Pre-12.0: no secrets, just check line exists
    canUseLine = (line ~= nil)
else
    -- 12.0+: verify neither line nor leftText is secret
    canUseLine = line and not issecretvalue(line) and not issecretvalue(line.leftText)
end

if canUseLine then
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

#### Cast Target Names

The target name returned by `UnitCastingInfo()` / `UnitChannelInfo()` was initially treated as potentially secret in 12.0.0, but subsequent builds (12.0.1+) appear to have made it non-secret. Major addons (ElvUI, oUF) have removed their `issecretvalue()` guards on cast target names. If targeting older builds, retain checks as a safety measure.

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

##### Private Aura APIs -- Combat-Restricted (12.0.1)

The following `C_UnitAuras` APIs can no longer be called during combat:
- `AddPrivateAuraAnchor()`
- `RemovePrivateAuraAnchor()`
- `SetPrivateWarningTextAnchor()`
- `AddPrivateAuraAppliedSound()`
- `RemovePrivateAuraAppliedSound()`

Set up all private aura anchors during initialization (`ADDON_LOADED` / `PLAYER_ENTERING_WORLD`), not dynamically during combat.

---

## CooldownViewer Frame Taint Pitfalls

Blizzard's CooldownViewer (`EditModeCooldownViewerSystemMixin`) is a protected frame managed by EditMode. Addons that customize its appearance or hook its behavior face several non-obvious taint pitfalls. These patterns apply broadly to **any Blizzard protected frame**, not just CooldownViewer.

> **See also:** [Cooldown Viewer Guide](13_Cooldown_Viewer_Guide.md#taint-avoidance-for-cooldownviewer-frames) for the full CooldownViewer taint reference.

### Never Nil Out Blizzard Frame Table Properties

Any insecure write to a Blizzard frame table taints the key. Nilling out a property like `DebuffBorder` on a CooldownViewer icon causes `RefreshIconBorder` to read the tainted nil during secure `RefreshData()` execution, contaminating the secure context. This propagates through the child iteration loop to `CheckAllowOnCooldown`, permanently tainting the file-local `wasOnGCDLookup` table and causing "attempted to index a forbidden table" warnings on EditMode exit.

```lua
-- WRONG: Taints the frame table key
child.DebuffBorder = nil

-- CORRECT: Hide visually without modifying the table
child.DebuffBorder:SetAlpha(0)
```

This rule applies to ALL Blizzard frames -- never write nil (or any value) to keys on Blizzard-owned frame tables from addon code. Use visual hiding (`SetAlpha(0)`, `SetSize(0.001, 0.001)`) instead of structural modification.

### Never hooksecurefunc on Child Mixin Methods

Hooks on CooldownViewer child mixin methods (`OnActiveStateChanged`, `OnUnitAuraAddedEvent`, etc.) inject insecure addon code into Blizzard's secure aura-processing chain. This taints sibling `spellID` / `sourceUnit` comparisons in `NeedsAddedAuraUpdate`, causing cascading taint through the entire aura update pipeline.

**Safe to hook:** Viewer-level methods like `Layout`, `RefreshLayout`, `OnUnitAura`.

**Never hook:** Individual child (icon) mixin methods that participate in secure aura/cooldown data processing.

### Implicit Protection via Reparenting

Reparenting a protected frame (CooldownViewer inherits `EditModeCooldownViewerSystemMixin`) to an addon-owned container makes that container implicitly protected. `SetWidth`, `SetHeight`, `SetPoint` on the container then fail during combat with "ADDON_ACTION_BLOCKED" errors. Solution: anchor children to the container via `SetPoint` WITHOUT reparenting the viewer.

```lua
-- CORRECT: Anchor children without reparenting
local container = CreateFrame("Frame", nil, UIParent)
child:SetPoint("BOTTOMLEFT", container, "BOTTOMLEFT", x, y)

-- WRONG: Never reparent a protected frame to your container
-- viewer:SetParent(container)  -- Makes container implicitly protected!
```

### issecretvalue() on Texture FileIDs

Texture values read from CooldownViewer frame properties (e.g., via `GetTexture()`) can be secret in tainted contexts. Always check `issecretvalue(tex)` before comparing texture FileIDs or using them as table keys:

```lua
local tex = icon.Icon:GetTexture()
if issecretvalue and issecretvalue(tex) then
    -- Cannot compare or use as table key -- skip or use fallback
    return
end
if tex == SOME_EXPECTED_TEXTURE then
    -- Safe to compare
end
```

### Secret-Safe Alpha Wrapper Technique

For glow or alert overlays that need to respond to secret-returning duration evaluations, create a plain wrapper frame and set its alpha via the secret-returning API (since `SetAlpha()` accepts secrets natively). The wrapper alpha multiplies with animated child alpha, making the overlay invisible when the wrapper alpha is 0:

```lua
local wrapper = CreateFrame("Frame", nil, parent)
local glow = CreateFrame("Frame", nil, wrapper)
-- ... set up glow animation on `glow` ...

-- When duration evaluation returns a secret, use it directly for visibility:
-- SetAlpha accepts secrets at C++ level
wrapper:SetAlpha(C_CurveUtil.EvaluateColorValueFromBoolean(
    duration:IsZero(), 0, 1
))
-- wrapper alpha 0 = glow invisible, wrapper alpha 1 = glow visible
-- No Lua-level secret comparison needed
```

### Non-Secret Cooldown Fields for Logic Decisions

`SpellCooldownInfo.isActive` and `SpellCooldownInfo.isOnGCD` are NeverSecret per 12.0.1. Use these for branching logic (if/else decisions), reserving DurationObjects only for display. This avoids secret value issues entirely for decision-making code:

```lua
local cooldownInfo = C_Spell.GetSpellCooldown(spellID)
-- isActive and isOnGCD are NEVER secret -- safe for if/else
if cooldownInfo.isActive and not cooldownInfo.isOnGCD then
    -- Show cooldown display using duration object (secret-safe for display)
    local duration = C_Spell.GetSpellCooldownDuration(spellID)
    cooldown:SetCooldownFromDurationObject(duration)
else
    cooldown:Clear()
end
```

### C_CurveUtil for Secret-Safe Duration Evaluation

`C_CurveUtil.CreateColorCurve()` combined with `DurationObject:EvaluateRemainingPercent(curve)` evaluates remaining time at the C++ level, producing secret-safe color and alpha values. Combined with `C_CurveUtil.EvaluateColorValueFromBoolean()`, this enables threshold-based visibility without Lua-level secret comparisons:

```lua
-- Create a curve that maps remaining-percent to alpha
-- e.g., alpha=1 when >10% remaining, fade to 0 below 10%
local fadeCurve = C_CurveUtil.CreateColorCurve(
    { percent = 0, value = 0 },   -- fully expired = invisible
    { percent = 10, value = 1 },  -- 10% remaining = fully visible
    { percent = 100, value = 1 }  -- full duration = fully visible
)

-- Evaluate entirely at C++ level -- result is secret-safe
local alpha = duration:EvaluateRemainingPercent(fadeCurve)
frame:SetAlpha(alpha)  -- SetAlpha accepts the result natively
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
14. [ ] Replace `SetCooldown`/`SetCooldownFromExpirationTime`/`SetCooldownDuration`/`SetCooldownUNIX` with `SetCooldownFromDurationObject` for secret values
15. [ ] Replace `ActionButton_ApplyCooldown` usage with `isActive`/`shouldReplaceNormalCooldown` fields + duration objects
16. [ ] Update LoC cooldown code to use new "Info" suffix APIs (e.g., `GetSpellLossOfControlCooldownInfo`)
17. [ ] Check for `format("%.Ns", secret)` precision specifier usage
18. [ ] Update code reading `UnitCreatureID` to handle `nil` returns
19. [ ] Move private aura API calls (`AddPrivateAuraAnchor`, etc.) out of combat paths
20. [ ] Check for `GetEffectiveAlpha()`, `IsStatusBarDesaturated()`, `IsDesaturated()` nil returns

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

### Secret-Safe Formatters (`AllowedWhenTainted`)

| API | Version | Purpose |
|-----|---------|---------|
| `AbbreviateNumbers(val, breakpoint)` | 12.0.0 | General numeric formatter; accepts secret number, returns non-secret string. See [AbbreviateNumbers: General-Purpose Secret-Safe Numeric Formatter (12.0.0)](#abbreviatenumbers-general-purpose-secret-safe-numeric-formatter-1200). |
| `C_StringUtil.TruncateWhenZero(num)` | 12.0.0 | Taint-safe replacement for `val > 0 and tostring(val) or ""`. See [C_StringUtil.TruncateWhenZero (12.0.0)](#c_stringutiltruncatewhenzero-1200). |
| `C_StringUtil.WrapString(infix, prefix, suffix)` | 12.0.0 | Taint-safe conditional wrapping when infix may be empty. |
| `FontString:SetFormattedText(fmt, ...)` | 12.0.0 | Secret-safe format-and-display for FontStrings. See [FontString:SetFormattedText() with Secret Values](#fontstringsetformattedtext-with-secret-values). |
| `AbbreviatedNumberFormatter` / `SecondsFormatter` / `NumericRuleFormatter` | 12.0.5 | Formatter objects for cooldown countdown text. See [Numeric Formatters (NEW in 12.0.5)](#numeric-formatters-new-in-1205). |

---

## See Also

- [12_API_Migration_Guide.md](12_API_Migration_Guide.md) - Full migration guide including secret values in combat context
- [13_Cooldown_Viewer_Guide.md](13_Cooldown_Viewer_Guide.md) - Cooldown Viewer system guide with full taint avoidance reference
- [01_API_Reference.md](01_API_Reference.md) - General API reference
- [10_Advanced_Techniques.md](10_Advanced_Techniques.md) - Advanced addon patterns

---

