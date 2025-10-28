# WoW API Migration & Version Compatibility Guide

## Table of Contents
1. [Overview](#overview)
2. [Where to Find API Changes](#where-to-find-api-changes)
3. [Major API Changes by Expansion](#major-api-changes-by-expansion)
4. [Common Migration Patterns](#common-migration-patterns)
5. [Version Detection & Compatibility](#version-detection--compatibility)
6. [Automated Migration Checklist](#automated-migration-checklist)
7. [Testing Across Versions](#testing-across-versions)
8. [Real-World Migration Examples](#real-world-migration-examples)

---

## Overview

When Blizzard releases new WoW expansions or major patches, addon APIs frequently change. Functions get deprecated, removed, or replaced with new alternatives. This guide helps you:

- Find official API change documentation
- Understand common migration patterns
- Create version-compatible code
- Migrate existing addons to new API versions

**Key Principle**: Always check API changes when a new expansion or major patch releases!

---

## Where to Find API Changes

### Official Resources

#### 1. Warcraft Wiki (PRIMARY SOURCE)
The most comprehensive and up-to-date resource for API changes.

**Main API Change Pages:**
```
https://warcraft.wiki.gg/wiki/Patch_[VERSION]/API_changes
```

**Examples:**
- Patch 11.0.0: https://warcraft.wiki.gg/wiki/Patch_11.0.0/API_changes
- Patch 11.0.2: https://warcraft.wiki.gg/wiki/Patch_11.0.2/API_changes
- Patch 10.2.5: https://warcraft.wiki.gg/wiki/Patch_10.2.5/API_changes

**What You'll Find:**
- New APIs added
- Deprecated APIs (still work, but scheduled for removal)
- Removed APIs (no longer work)
- Changed API signatures
- New namespaces (C_* APIs)

#### 2. WoW Forums - UI & Macro Forum
```
https://us.forums.blizzard.com/en/wow/c/ui-and-macro/
```

Search for threads like:
- "API changes in [patch]"
- "Deprecated API post [patch]"
- "Addon broke after [patch]"

#### 3. GitHub - WoW UI Source
```
https://github.com/Gethe/wow-ui-source
```

Browse commits to see changes between versions.

#### 4. CurseForge/Wago.io Addon Comments
Check popular addons' update notes for insights on what changed.

### Version History Quick Reference

| Expansion | Interface Version | Major API Changes |
|-----------|------------------|-------------------|
| **The War Within** | 11.0.0 - 11.x.x | UIDropDownMenu deprecated, GetSpellInfo removed, C_Spell changes |
| **Dragonflight** | 10.0.0 - 10.2.7 | UnitAura/UnitBuff/UnitDebuff deprecated, TextStatusBar changes |
| **Shadowlands** | 9.0.0 - 9.2.7 | C_Container namespace, many containerAPIs moved |
| **Battle for Azeroth** | 8.0.0 - 8.3.x | Major UI scale changes, backdrop changes |
| **Legion** | 7.0.0 - 7.3.x | Protected functions expanded, macro changes |

---

## Major API Changes by Expansion

### The War Within (11.0.0+)

#### ❌ Removed APIs
```lua
-- REMOVED in 11.0.0
GetSpellInfo(spellID)  -- Use C_Spell.GetSpellInfo() or C_Spell.GetSpellName()
GetSpellCooldown(spellID)  -- Use C_Spell.GetSpellCooldown()
```

#### 🆕 New/Changed APIs
```lua
-- NEW: C_Spell namespace (11.0.0)
C_Spell.GetSpellInfo(spellID)  -- Returns table
C_Spell.GetSpellName(spellID)  -- Returns name only
C_Spell.GetSpellCooldown(spellID)  -- Returns table

-- Example return values:
local spellInfo = C_Spell.GetSpellInfo(spellID)
-- spellInfo = {
--   name = "Spell Name",
--   iconID = 123456,
--   castTime = 1500,
--   minRange = 0,
--   maxRange = 40,
--   spellID = 12345
-- }

local cooldownInfo = C_Spell.GetSpellCooldown(spellID)
-- cooldownInfo = {
--   startTime = 123456.789,
--   duration = 30,
--   isEnabled = true,
--   modRate = 1.0
-- }
```

#### ⚠️ Deprecated (Still Works)
```lua
-- DEPRECATED in 11.0.0
UIDropDownMenu  -- Use new menu system or keep for compatibility
```

### Dragonflight (10.0.0 - 10.2.7)

#### ❌ Deprecated (Removed in 11.0)
```lua
-- DEPRECATED in 10.2.5
UnitAura(unit, index)  -- Use C_UnitAuras.GetAuraDataByIndex()
UnitBuff(unit, index)  -- Use C_UnitAuras.GetBuffDataByIndex()
UnitDebuff(unit, index)  -- Use C_UnitAuras.GetDebuffDataByIndex()

-- DEPRECATED in 10.2.7
TextStatusBar global functions  -- Use methods on StatusBar objects
```

### Shadowlands (9.0.0+)

#### 🆕 New Namespaces
```lua
-- Container APIs moved to C_Container (9.0.0)
C_Container.GetContainerNumSlots(bagID)  -- Was GetContainerNumSlots()
C_Container.GetContainerItemID(bagID, slot)  -- Was GetContainerItemID()
C_Container.GetContainerItemInfo(bagID, slot)  -- Was GetContainerItemInfo()
```

---

## Common Migration Patterns

### Pattern 1: Single Function to Namespace

**Before (Old API):**
```lua
local name = GetSpellInfo(spellID)
```

**After (New API):**
```lua
local name = C_Spell.GetSpellName(spellID)
```

### Pattern 2: Multiple Return Values to Table

**Before (Old API):**
```lua
local start, duration, enabled = GetSpellCooldown(spellID)
```

**After (New API):**
```lua
local cooldownInfo = C_Spell.GetSpellCooldown(spellID)
local start = cooldownInfo.startTime
local duration = cooldownInfo.duration
local enabled = cooldownInfo.isEnabled
```

### Pattern 3: Compatibility Wrapper (Best Practice)

Create wrappers for backward compatibility with old code and libraries:

```lua
-- In your Constants.lua or init file (loaded first)

-- Compatibility wrapper for GetSpellInfo
if not _G.GetSpellInfo then
    _G.GetSpellInfo = function(spellID)
        if not spellID then return nil end
        local spellInfo = C_Spell.GetSpellInfo(spellID)
        if spellInfo then
            return spellInfo.name, nil, spellInfo.iconID,
                   spellInfo.castTime, spellInfo.minRange,
                   spellInfo.maxRange, spellInfo.spellID,
                   spellInfo.originalIconID
        end
        return nil
    end
end

-- Compatibility wrapper for GetSpellCooldown
if not _G.GetSpellCooldown then
    _G.GetSpellCooldown = function(spellID)
        local cooldownInfo = C_Spell.GetSpellCooldown(spellID)
        if cooldownInfo then
            return cooldownInfo.startTime, cooldownInfo.duration,
                   cooldownInfo.isEnabled, cooldownInfo.modRate
        end
        return 0, 0, false, 1
    end
end
```

**Benefits:**
- ✅ Old code continues to work
- ✅ Embedded libraries don't need updates
- ✅ Gradual migration possible
- ✅ One-file fix

### Pattern 4: Version-Specific Code Blocks

```lua
local tocVersion = select(4, GetBuildInfo())

if tocVersion >= 110000 then  -- 11.0.0+
    -- Use new API
    local name = C_Spell.GetSpellName(spellID)
else  -- 10.x or earlier
    -- Use old API
    local name = GetSpellInfo(spellID)
end
```

### Pattern 5: Safe API Wrapper with Fallback

```lua
-- Wrapper that works across all versions
local function GetSpellNameSafe(spellID)
    if C_Spell and C_Spell.GetSpellName then
        return C_Spell.GetSpellName(spellID)
    elseif GetSpellInfo then
        return GetSpellInfo(spellID)
    else
        return nil
    end
end
```

---

## Version Detection & Compatibility

### TOC Version Detection

```lua
-- Get current game version
local tocVersion = select(4, GetBuildInfo())
local majorVersion = math.floor(tocVersion / 10000)

-- Expansion detection
local isWarWithin = (tocVersion >= 110000)
local isDragonflight = (tocVersion >= 100000 and tocVersion < 110000)
local isShadowlands = (tocVersion >= 90000 and tocVersion < 100000)
local isBFA = (tocVersion >= 80000 and tocVersion < 90000)

-- Use in code
if isWarWithin then
    -- War Within specific code
elseif isDragonflight then
    -- Dragonflight specific code
end
```

### Client Type Detection

```lua
-- Detect Retail vs Classic
local isRetail = (WOW_PROJECT_ID == WOW_PROJECT_MAINLINE)
local isClassic = (WOW_PROJECT_ID == WOW_PROJECT_CLASSIC)
local isWrathClassic = (WOW_PROJECT_ID == WOW_PROJECT_WRATH_CLASSIC)
local isCataClassic = (WOW_PROJECT_ID == WOW_PROJECT_CATACLYSM_CLASSIC)

if isRetail then
    -- Use Retail-only APIs
end
```

### Function Existence Check (Safest)

```lua
-- Check if API exists before using
if C_Spell and C_Spell.GetSpellName then
    local name = C_Spell.GetSpellName(spellID)
elseif GetSpellInfo then
    local name = GetSpellInfo(spellID)
else
    error("No spell API available!")
end
```

---

## Automated Migration Checklist

Use this checklist when migrating an addon to a new WoW version:

### Pre-Migration

- [ ] Check Warcraft Wiki for API changes in new version
- [ ] Read patch notes for UI/API section
- [ ] Search WoW forums for common issues
- [ ] Check if addon uses embedded libraries (may need updates)
- [ ] Create backup of working addon

### TOC File Updates

- [ ] Update `## Interface:` version number
- [ ] Update `## Version:` string
- [ ] Check `## Dependencies:` are compatible
- [ ] Test without `## OptionalDeps:` first

### Code Audit

Run these searches in your addon folder:

#### Search for Deprecated Spell APIs (11.0+)
```bash
# Search for these patterns:
GetSpellInfo\(
GetSpellCooldown\(
```

#### Search for Deprecated Aura APIs (10.2.5+)
```bash
# Search for these patterns:
UnitAura\(
UnitBuff\(
UnitDebuff\(
```

#### Search for Old Container APIs (9.0+)
```bash
# Search for these patterns:
GetContainerNumSlots\(
GetContainerItemID\(
GetContainerItemInfo\(
GetContainerItemLink\(
```

### Testing

- [ ] Test addon loads without errors (`/reload`)
- [ ] Enable script errors: `/console scriptErrors 1`
- [ ] Test core functionality
- [ ] Check for deprecation warnings in chat
- [ ] Test with other popular addons
- [ ] Test in different zones/scenarios

### Documentation

- [ ] Update README with supported versions
- [ ] Document API changes in CHANGELOG
- [ ] Update CurseForge/Wago description
- [ ] Add migration notes for other developers

---

## Testing Across Versions

### In-Game Testing Commands

```lua
-- Check game version
/run print(select(4, GetBuildInfo()))

-- Check for function existence
/run print(C_Spell ~= nil)
/run print(C_Spell.GetSpellName ~= nil)

-- Enable error display
/console scriptErrors 1

-- Reload UI
/reload

-- Monitor events
/eventtrace

-- Check frame stack
/fstack
```

### Testing Methodology

1. **Fresh Character Test**
   - Create new character
   - Test with no saved variables
   - Verify first-run experience

2. **Migrated Data Test**
   - Use character with old saved variables
   - Check data migration works
   - Verify no corruption

3. **Compatibility Test**
   - Test with other popular addons
   - Check for conflicts
   - Test Ace3 compatibility

---

## Real-World Migration Examples

### Example 1: Archaeology Addon (10.0 → 11.0)

**Issues Found:**
- `GetSpellInfo()` removed in 11.0
- `GetSpellCooldown()` changed signature

**Files Modified:**
```
Archy_Mainline.toc  - Update Interface version
Constants.lua       - Add compatibility wrappers
Archy.lua           - Update 3 locations
Race.lua            - Update 1 location
```

**Solution Applied:**
```lua
-- Added to Constants.lua (loaded first)
if not _G.GetSpellInfo then
    _G.GetSpellInfo = function(spellID)
        if not spellID then return nil end
        local spellInfo = C_Spell.GetSpellInfo(spellID)
        if spellInfo then
            return spellInfo.name, nil, spellInfo.iconID,
                   spellInfo.castTime, spellInfo.minRange,
                   spellInfo.maxRange, spellInfo.spellID,
                   spellInfo.originalIconID
        end
        return nil
    end
end

-- Direct API updates in addon code
local name = C_Spell.GetSpellName(SURVEY_SPELL_ID)

-- Updated cooldown checks
local cooldownInfo = C_Spell.GetSpellCooldown(spellID)
local duration = cooldownInfo and cooldownInfo.duration or 0
```

**Result:** ✅ Addon works, embedded libraries work via wrapper

### Example 2: Bag Addon (8.0 → 9.0)

**Issue:** Container APIs moved to `C_Container` namespace

**Before:**
```lua
local numSlots = GetContainerNumSlots(bagID)
local itemID = GetContainerItemID(bagID, slot)
```

**After:**
```lua
local numSlots = C_Container.GetContainerNumSlots(bagID)
local itemID = C_Container.GetContainerItemID(bagID, slot)
```

**Compatibility Wrapper:**
```lua
-- Support both old and new APIs
local GetNumSlots = C_Container and C_Container.GetContainerNumSlots
                    or GetContainerNumSlots
local GetItemID = C_Container and C_Container.GetContainerItemID
                  or GetContainerItemID

-- Use in code
local numSlots = GetNumSlots(bagID)
```

### Example 3: Aura Tracking (10.2 → 11.0)

**Issue:** `UnitAura()` deprecated in 10.2.5

**Before (Old):**
```lua
local name, icon, count, debuffType, duration, expirationTime =
    UnitAura("player", i, "HELPFUL")
```

**After (New):**
```lua
local auraData = C_UnitAuras.GetAuraDataByIndex("player", i, "HELPFUL")
if auraData then
    local name = auraData.name
    local icon = auraData.icon
    local count = auraData.applications
    local debuffType = auraData.dispelName
    local duration = auraData.duration
    local expirationTime = auraData.expirationTime
end
```

**Compatibility Solution:**
```lua
local function GetAuraInfo(unit, index, filter)
    -- Try new API first (10.2.5+)
    if C_UnitAuras and C_UnitAuras.GetAuraDataByIndex then
        local auraData = C_UnitAuras.GetAuraDataByIndex(unit, index, filter)
        if auraData then
            return auraData.name, auraData.icon, auraData.applications,
                   auraData.dispelName, auraData.duration,
                   auraData.expirationTime
        end
    -- Fall back to old API
    elseif UnitAura then
        return UnitAura(unit, index, filter)
    end
    return nil
end
```

---

## Migration Quick Reference Table

| Old API (Pre-11.0) | New API (11.0+) | Migration Pattern |
|-------------------|----------------|-------------------|
| `GetSpellInfo(id)` | `C_Spell.GetSpellName(id)` | Direct replacement |
| `GetSpellCooldown(id)` | `C_Spell.GetSpellCooldown(id)` | Returns table |
| `UnitAura(unit, i)` | `C_UnitAuras.GetAuraDataByIndex(unit, i)` | Returns table |
| `GetContainerNumSlots(bag)` | `C_Container.GetContainerNumSlots(bag)` | Namespace move |
| `GetItemInfo(id)` | `C_Item.GetItemInfo(id)` or `GetItemInfo(id)` | Both work |

---

<!-- CLAUDE_SKIP_START -->
## Best Practices for Future-Proof Addons

### 1. Use Compatibility Wrappers
Create wrapper functions for commonly used APIs that might change.

### 2. Check Function Existence
Always check if new APIs exist before using them.

### 3. Version-Gate Features
Use TOC version checks for expansion-specific features.

### 4. Document Version Requirements
Clearly state minimum/maximum supported versions.

### 5. Monitor API Deprecation Warnings
Test with `/console scriptErrors 1` to catch warnings.

### 6. Subscribe to Update Notifications
- Follow Warcraft Wiki
- Join WoW Addon Discord
- Monitor CurseForge/GitHub issues

### 7. Test on PTR
Test your addon on Public Test Realm before patches go live.

### 8. Keep Libraries Updated
Regularly update embedded libraries (Ace3, LibStub, etc.).

---

## Resources & Links

### Essential Bookmarks

**Warcraft Wiki - API Changes**
```
https://warcraft.wiki.gg/wiki/Category:API_changes
```

**WoW API Documentation**
```
https://warcraft.wiki.gg/wiki/World_of_Warcraft_API
```

**WoW Forums - UI & Macro**
```
https://us.forums.blizzard.com/en/wow/c/ui-and-macro/
```

**GitHub - WoW UI Source**
```
https://github.com/Gethe/wow-ui-source
```

### Community Resources

- **WoW AddOn Discord**: Real-time help with API changes
- **CurseForge Forums**: Addon-specific discussions
- **Wago.io**: Addon hosting with version tracking

---

## Conclusion

API migrations are a regular part of WoW addon development. By:
- Checking Warcraft Wiki for API changes
- Using compatibility wrappers
- Testing thoroughly
- Following migration patterns

You can keep your addons working across expansions with minimal effort.

**Remember:** Check API changes BEFORE each major patch, not after your addon breaks!

---

**Related Guides:**
- [10_Advanced_Techniques.md](10_Advanced_Techniques.md) - Cross-client compatibility
- [04_Addon_Structure.md](04_Addon_Structure.md) - TOC file versioning
- [05_Patterns_And_Best_Practices.md](05_Patterns_And_Best_Practices.md) - Code patterns

**Next Steps:**
1. Bookmark Warcraft Wiki API changes pages
2. Create compatibility wrapper template
3. Set up automated testing for new patches
4. Join WoW Addon Discord for early warnings

<!-- CLAUDE_SKIP_END -->
