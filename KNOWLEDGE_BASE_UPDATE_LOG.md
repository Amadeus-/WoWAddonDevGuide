<\!-- CLAUDE_SKIP_START -->
# WoW Addon Development Knowledge Base - Update Log

## Version 2.5 - 2026-01-28

### Comprehensive Secret-Safe API Documentation (12.0.0+)

**Summary:**
Created a new comprehensive documentation file (`12a_Secret_Safe_APIs.md`) that provides complete coverage of WoW 12.0.0's secret values system - one of the most significant API changes in WoW addon history.

**Files Created:**
- `12a_Secret_Safe_APIs.md` - Complete 1000+ line reference for secret values system

**Files Updated:**
- `00_MASTER_PROMPT.md` - Added reference to new file
- `01_API_Reference.md` - Updated cross-reference
- `03_UI_Framework.md` - Updated cross-reference
- `README.md` - Added to documentation list (now 14 guides)
- `QUICK_START_GUIDE.md` - Added to guide list
- `WoWAddon-Expert.md` (agent file) - Added quick-reference section

**Research Sources:**
1. **Platynator addon** - Real-world addon using `issecretvalue()` for absorb text, guild text, MSP integration
2. **Blizzard UI Source (12.0.0)** - 50+ files using secret-safe patterns including:
   - `RestrictedInfrastructure.lua` - Core secret handling
   - `SecureTypes.lua` - SecureMap, SecureArray, SecureNumber containers
   - `Pools.lua` - Object pool secret rejection patterns
   - `Dump.lua` - Handling secrets in /dump command
   - `AuraUtil.lua` - `securecallfunction` patterns
   - `CallbackRegistry.lua` - Secure callback execution
3. **FrameScriptDocumentation.lua** - Official API documentation for all secret functions

**Documentation Includes:**

1. **Core Detection Functions:**
   - `issecretvalue(val)` - Check if value is secret
   - `canaccessvalue(val)` - Check caller access
   - `canaccessallvalues(...)` - Check multiple values
   - `hasanysecretvalues(...)` - Check for any secrets
   - `canaccesssecrets()` - Check caller permissions
   - `canaccesstable(tbl)` - Check table access
   - `issecrettable(tbl)` - Check if table is secret

2. **Manipulation Functions:**
   - `scrubsecretvalues(...)` - Replace secrets with nil
   - `scrub(...)` - Aggressive scrubbing
   - `mapvalues(fn, ...)` - Transform values
   - `secretwrap(...)` - Create secrets (restricted)
   - `secretunwrap(...)` - Unwrap secrets (restricted)
   - `dropsecretaccess()` - Drop privileges

3. **Secure Execution Functions:**
   - `securecallfunction(fn, ...)` - Call function securely
   - `secureexecuterange(tbl, fn, ...)` - Secure table iteration
   - `forceinsecure()` - Force tainted context

4. **Table Security System:**
   - `SetTableSecurityOption(tbl, option)`
   - `TableSecurityOption.DisallowTaintedAccess`
   - `TableSecurityOption.DisallowSecretKeys`
   - `TableSecurityOption.SecretWrapContents`

5. **SecureTypes Containers:**
   - `SecureTypes.CreateSecureMap()`
   - `SecureTypes.CreateSecureArray()`
   - `SecureTypes.CreateSecureStack()`
   - `SecureTypes.CreateSecureValue()`
   - `SecureTypes.CreateSecureNumber()`
   - `SecureTypes.CreateSecureBoolean()`
   - `SecureTypes.CreateSecureFunction()`

6. **Practical Patterns:**
   - Check-before-use pattern
   - Defer-to-out-of-combat pattern
   - Native StatusBar for secret display
   - Percentage-based custom bars
   - Real-world examples from Platynator and Blizzard UI

7. **Testing Section:**
   - CVars for forcing restrictions
   - Test commands
   - Verification checklist

8. **Migration Checklist:**
   - For existing addons
   - For new addons

**Why This Was Needed:**
The secret values system is the most significant API change in WoW addon history. Without proper documentation, addon developers face:
- Mysterious runtime errors ("attempt to compare a secret value")
- Broken functionality during combat
- No clear migration path from pre-12.0 patterns
- Limited understanding of which APIs are affected

This guide provides a single authoritative reference for all secret-safe API patterns.

---

## Version 2.4 - 2026-01-24

### Closing Blizzard Settings Panel Documentation (12.0.0)

**Summary:**
Added documentation about the correct way to programmatically close the Blizzard Settings panel from addon code. This is a common requirement for addons that use AceConfig or similar libraries for standalone configuration dialogs.

**Files Updated:**
- `12_API_Migration_Guide.md` - New section "Closing the Blizzard Settings Panel (12.0.0+)"

**The Issue:**
When addon code attempts to close the Blizzard Settings panel before opening its own config dialog, common approaches fail:

1. **`Settings.CloseUI()`** - Does not exist (nil error)
2. **`SettingsPanel:Close()`** - Triggers `ADDON_ACTION_BLOCKED` because it internally calls `Commit()` which triggers `SaveBindings()` - a protected function

**Correct Approach:**
```lua
if SettingsPanel and SettingsPanel:IsShown() then
    HideUIPanel(SettingsPanel)
end
```

**Why This Was Added:**
Discovered during real-world addon development when trying to create a workflow where clicking on an addon's entry in the Blizzard Interface Options would close the Settings panel and open a standalone AceConfig dialog instead. The natural approaches (`Close()` method, hypothetical `CloseUI()` function) either don't exist or trigger protected function errors.

**Use Cases Documented:**
- Opening AceConfig standalone dialogs
- Redirecting from Interface Options to standalone config
- General pattern for hiding built-in panels safely

---

## Version 2.3 - 2026-01-24

### Nameplate Click Targeting Changes Documentation (12.0.0)

**Summary:**
Added critical documentation about WoW 12.0.0's change to how nameplate click targeting works. This is essential for any addon that replaces or modifies nameplates (TidyPlates, NeatPlates, Plater, KuiNameplates, etc.).

**Files Updated:**
- `12_API_Migration_Guide.md` - New section "Nameplate Click Targeting Changes (12.0.0)"

**The Issue:**
In WoW 12.0.0, Blizzard moved nameplate click detection from Lua-level frame handling to C++ level using a `HitTestFrame` child of the nameplate's UnitFrame. This breaks click targeting for any addon that uses `Hide()` on the Blizzard UnitFrame.

**Key Points Documented:**
1. **Problem**: `UnitFrame:Hide()` also hides the HitTestFrame child, breaking click targeting
2. **Solution**: Use `SetAlpha(0)` instead of `Hide()` to make Blizzard plates invisible while keeping click targeting active
3. **New API**: `C_NamePlateManager.SetNamePlateHitTestFrame(frame)` exists but may be restricted to Blizzard code only
4. **Migration Checklist**: Step-by-step guide for updating nameplate addons
5. **Debug Tip**: Code to check if HitTestFrame is properly visible

**Why This Was Added:**
Discovered during real-world testing of NeatPlates nameplate addon in WoW 12.0.0. Click targeting was completely broken until the Hide()/SetAlpha(0) pattern was identified. This is a common issue that will affect many nameplate addons.

**Documentation Location:**
Added immediately after the "Testing Secret Value Handling" section in the 12.0.0 changes, keeping all nameplate-related content together (secret values for health bars + click targeting).

---

## Version 2.2 - 2026-01-22

### Critical Discovery: C_DamageMeter API Data is SECRET-PROTECTED

**Summary:**
Through real-world testing of the Recount damage meter addon, we discovered that **C_DamageMeter API data is protected as "secret values"** and cannot be used by third-party addons. This means **no third-party damage meters can function in WoW 12.0.0+**.

**Files Updated:**
- `12_API_Migration_Guide.md` - Complete rewrite of C_DamageMeter section and Example 5

**What Was Discovered:**

When attempting to use C_DamageMeter API (after combat log was blocked), we found:

1. **`C_DamageMeter.IsDamageMeterAvailable()` returns `true`** - The API exists and claims to be available
2. **`C_DamageMeter.GetCombatSessionFromType()` returns data** - You get a table with entries
3. **BUT the actual data values are SECRET:**
   ```lua
   source.name           -- SECRET: "attempt to compare (a secret value)"
   source.totalAmount    -- SECRET: <no value>
   source.amountPerSecond -- SECRET: <no value>
   source.sourceGUID     -- SECRET: <no value>
   ```
4. **Only cosmetic fields are accessible:**
   ```lua
   source.classFilename  -- Works: "DEATHKNIGHT"
   source.isLocalPlayer  -- Works: true
   source.specIconID     -- Works: 135770
   ```

**Error Message When Trying to Use Secret Values:**
```
attempt to compare local 'name' (a secret value)
```

**What This Means:**
| Blocked Path | Error |
|--------------|-------|
| Combat log events | `ADDON_ACTION_FORBIDDEN` on registration |
| C_DamageMeter data | "secret value" protection on key fields |

**Conclusion:**
- C_DamageMeter API is designed **ONLY for Blizzard's built-in UI**
- Third-party damage meters (Recount, Skada, Details!) **cannot function in 12.0.0+**
- There is NO workaround - players must use Blizzard's meter
- This is intentional as part of Blizzard's "addon disarmament"

**Documentation Changes:**
- Marked C_DamageMeter section as "⛔ SECRET-PROTECTED (UNUSABLE BY ADDONS)"
- Added table showing which fields are secret vs accessible
- Rewrote Example 5 to show there is NO valid migration path
- Added code example showing how to display warning to users

**Verified By:**
- Real-world testing with Recount addon in WoW 12.0.0
- Error captured by BugGrabber/BugSack showing secret value details

---

## Version 2.1 - 2026-01-22

### Critical Fix: Combat Log Events BLOCKED Documentation

**Summary:**
Updated `12_API_Migration_Guide.md` to accurately document that **combat log events are completely blocked for third-party addons in WoW 12.0.0**. This is a critical correction - the previous documentation showed API migration patterns that **do not work** because event registration itself is blocked.

**Files Updated:**
- `12_API_Migration_Guide.md` - Added critical warning section, fixed Example 5

**What Was Discovered:**

When attempting to update the Recount damage meter addon for 12.0.0 compatibility, we discovered that:

1. **`Frame:RegisterEvent("COMBAT_LOG_EVENT_UNFILTERED")` throws `ADDON_ACTION_FORBIDDEN`** - The event registration itself is blocked, not just the API function calls
2. **This applies to ALL registration methods:**
   - `frame:RegisterEvent()` - BLOCKED
   - `EventRegistry:RegisterFrameEventAndCallback()` - BLOCKED
   - `RegisterEventCallback()` - BLOCKED
3. **Using `pcall()` does NOT prevent the error** - The game still raises ADDON_ACTION_FORBIDDEN
4. **This is intentional** - Blizzard officially stated: "Combat Log Events are no longer available to addons"
5. **Traditional damage meters CANNOT function in 12.0.0** without major refactoring to use C_DamageMeter API

**Previous (Incorrect) Documentation Implied:**
```lua
-- This was shown as working in 12.0.0:
local frame = CreateFrame("Frame")
frame:RegisterEvent("COMBAT_LOG_EVENT_UNFILTERED")  -- DOES NOT WORK!
```

**Corrected Documentation Now States:**
```lua
-- THIS CODE THROWS ADDON_ACTION_FORBIDDEN IN 12.0.0:
local frame = CreateFrame("Frame")
frame:RegisterEvent("COMBAT_LOG_EVENT_UNFILTERED")  -- BLOCKED!
-- You MUST use C_DamageMeter API instead
```

**Why This Matters:**
- Developers relying on the previous documentation would waste time trying to make combat log parsing work
- Traditional damage meters (Recount, Skada, Details!) need complete architectural rewrites
- The only path forward is using `C_DamageMeter` API for damage/healing data

**Sources:**
- [Combat Addon Restrictions Eased in Midnight - Icy Veins](https://www.icy-veins.com/wow/news/combat-addon-restrictions-eased-in-midnight/)
- [Patch 12.0.0/Planned API changes - Warcraft Wiki](https://warcraft.wiki.gg/wiki/Patch_12.0.0/Planned_API_changes)
- Real-world testing with Recount addon in WoW 12.0.0

---

## Version 2.0 - 2026-01-20

### Major Update: 12.0.0 (Midnight) Compatibility

**Summary:**
Comprehensive update of the entire WoW Addon Development Knowledge Base for the Midnight expansion (12.0.0). This update documents the "Addon Apocalypse" - the largest API overhaul in WoW history with 432 new APIs, 140+ removed APIs, and the introduction of the Secret Values security system.

**New Files Added:**
- `12_Housing_System_Guide.md` - Comprehensive guide for player housing APIs (287+ new functions)

**Files Updated (All 13 Guides):**
- `00_MASTER_PROMPT.md` - Updated for 12.0.0, new namespaces, Secret Values warning
- `01_API_Reference.md` - Major expansion with 12.0.0 namespaces, Secret Values, new functions
- `02_Event_System.md` - 75+ new events, callback-based registration, removed events
- `03_UI_Framework.md` - Secret value widgets, Curve/Duration objects, new widget methods
- `04_Addon_Structure.md` - New TOC directives, C_AddOns updates, profiling
- `05_Patterns_And_Best_Practices.md` - Secret values handling, encoding, housing patterns
- `06_Data_Persistence.md` - C_EncodingUtil, LoadSavedVariablesFirst, Warband data
- `07_Blizzard_UI_Examples.md` - New Blizzard addons (DamageMeter, Housing, etc.)
- `08_Community_Addon_Patterns.md` - 12.0 migration guides per addon category
- `09_Addon_Libraries_Guide.md` - Native API alternatives, library compatibility
- `10_Advanced_Techniques.md` - Secret Values system, profiling, encounter integration
- `11_API_Migration_Guide.md` - Complete patch-by-patch documentation 11.0.2-12.0.1
- `README.md` - Updated version, added Housing guide
- `QUICK_START_GUIDE.md` - 12.0.0 critical changes section

**Major Topics Added:**
1. **Secret Values System (12.0.0)** - Complete documentation of Blizzard's new security system that obscures combat-sensitive data from addons
2. **C_DamageMeter** - Official damage meter API that replaces combat log parsing for damage/healing data
3. **C_EncounterTimeline / C_EncounterWarnings** - New boss timeline system for encounter addons
4. **C_Housing (287+ functions)** - Complete player housing system API
5. **C_ActionBar namespace** - Replaces global action bar functions (REMOVED in 12.0.0)
6. **C_CombatLog namespace** - Replaces global combat log functions (REMOVED in 12.0.0)
7. **C_EncodingUtil (11.1.5)** - Native compression/serialization APIs
8. **New TOC directives** - Category, Group, LoadSavedVariablesFirst, AllowAddOnTableAccess
9. **Callback-based event registration (12.0.0)** - New EventUtil patterns
10. **Lua extensions** - table.create, table.count, string.concat

**API Changes Documented:**
- Patches: 11.0.2, 11.0.5, 11.0.7, 11.1.0, 11.1.5, 11.1.7, 11.2.0, 11.2.5, 11.2.7, 12.0.0, 12.0.1
- New APIs: 432+ (12.0.0) + hundreds more across 11.x patches
- Removed APIs: 140+ (12.0.0) + many deprecations in 11.x
- New Events: 100+ (housing, combat, encounter, transmog)
- New CVars: 100+ (secret testing, encounter, housing)

**Statistics:**
- Total guides: 14 (was 12)
- API Migration Guide: Expanded from ~630 lines to ~1,950 lines
- Housing System Guide: New file, ~1,766 lines
- All other guides significantly expanded with 12.0.0 content

**Breaking Changes Documented:**
- Global action bar functions REMOVED (use C_ActionBar)
- Combat log globals REMOVED (use C_CombatLog)
- Transmog APIs completely redesigned (C_TransmogCollection/Sets overhaul)
- Void Storage REMOVED (11.2.0)
- Reagent Bank REMOVED (11.2.0)
- Socket APIs moved (11.2.5)
- UnitAura finally removed (use C_UnitAuras)

**Why This Update:**
The Midnight expansion (12.0.0) represents the largest API overhaul in WoW history. Blizzard implemented sweeping security changes ("Secret Values") that fundamentally change how combat addons work. The combat log no longer exposes real damage numbers to addons - instead, addons must use C_DamageMeter to access damage/healing data. Without updated documentation, addon developers would struggle to migrate existing addons or create new ones for the Midnight expansion.

**Impact:**
This update ensures the knowledge base remains the authoritative reference for WoW addon development through the Midnight expansion era. Developers can now:
- Understand and implement Secret Values handling
- Migrate combat addons to use C_DamageMeter
- Build player housing addons using C_Housing
- Navigate the extensive API removals and replacements
- Use new encoding/compression APIs for data handling

**Directories Removed (2026-01-20 Cleanup):**
The following directories contained outdated 11.x extracted data and were removed to prevent confusion. All useful information from these directories is now incorporated into the main guide files in a more organized and useful format:

- `api_extracted/` - Contained raw API index files (513 entries from 11.x). Redundant with `01_API_Reference.md` which has better organized, example-rich API documentation.
- `events_extracted/` - Contained alphabetical event list (1,645 events from 11.2.7). Redundant with `02_Event_System.md` which includes event payloads, examples, and 12.0.0 updates.
- `file_lists/` - Contained Blizzard addon folder listings from 11.2.7. Low practical value for addon development.

**Rationale:** These directories were originally created as raw data dumps during the initial knowledge base creation. The main guide files (00-12) now contain all relevant information in a curated, example-rich format that's more useful for actual addon development. Keeping outdated extracted data risked causing confusion or misinformation.

---

## Version 1.1 - 2025-10-19

### Major Addition: API Migration Guide

**Files Added:**
- ✅ `11_API_Migration_Guide.md` - Comprehensive guide for API version migration and compatibility

**Files Updated:**
- ✅ `README.md` - Updated to reflect 12 guides, added migration guide to all relevant sections
- ✅ `00_MASTER_PROMPT.md` - Added reference to new migration guide

**External Files Updated:**
- ✅ `C:\Dev\++Claude AI++\prompts\update_wow_addon_dev_guide.md` - Updated prompt template

### What's New in 11_API_Migration_Guide.md

This comprehensive new guide addresses the common problem of updating addons when WoW releases new patches or expansions.

**Topics Covered:**

1. **Where to Find API Changes**
   - Warcraft Wiki links (primary source)
   - WoW Forums
   - GitHub UI source
   - CurseForge/Wago.io comments

2. **Major API Changes by Expansion**
   - The War Within (11.0+): GetSpellInfo/GetSpellCooldown removed
   - Dragonflight (10.x): UnitAura deprecated, TextStatusBar changes
   - Shadowlands (9.0+): C_Container namespace introduction
   - Complete version history table

3. **Common Migration Patterns**
   - Single function to namespace (GetSpellInfo → C_Spell.GetSpellName)
   - Multiple return values to table (cooldown APIs)
   - Compatibility wrapper pattern (backward compatibility)
   - Version-specific code blocks
   - Safe API wrapper with fallback

4. **Version Detection & Compatibility**
   - TOC version detection
   - Client type detection (Retail vs Classic)
   - Function existence checks
   - Expansion detection patterns

5. **Automated Migration Checklist**
   - Pre-migration tasks
   - TOC file updates
   - Code audit with search patterns
   - Testing methodology
   - Documentation requirements

6. **Real-World Migration Examples**
   - Archaeology addon (10.0 → 11.0)
   - Bag addon (8.0 → 9.0)
   - Aura tracking (10.2 → 11.0)
   - Complete code examples with before/after

7. **Migration Quick Reference Table**
   - Old API → New API mappings
   - Migration patterns for each change

8. **Best Practices for Future-Proof Addons**
   - 8 key strategies for maintaining compatibility
   - Resource links and bookmarks
   - Community resources

### Why This Was Added

**Problem Identified:** When helping update the ArchyShadowlands addon from version 10.x to 11.x, it became clear that:

1. API changes between WoW versions are common and predictable
2. Knowing WHERE to find API change documentation is critical
3. Common migration patterns exist (wrappers, table returns, namespace moves)
4. Developers need a systematic approach to version updates

**Solution:** A dedicated guide that teaches:
- Where to research API changes (Warcraft Wiki is primary source)
- How to detect and fix deprecated APIs
- Proven compatibility wrapper patterns
- Real-world migration examples from actual addon updates
- Automated checklists to ensure complete migration

### Key Features

**Comprehensive Coverage:**
- ✅ Links to official API change documentation for every major patch
- ✅ Version history table (expansions, interface versions, major changes)
- ✅ Complete code examples for all migration patterns
- ✅ Real-world case studies from actual addon migrations
- ✅ Automated audit scripts and search patterns
- ✅ Testing methodology and verification steps

**Practical Tools:**
- ✅ Quick reference table of API changes
- ✅ Copy-paste compatibility wrappers
- ✅ Version detection code snippets
- ✅ Search patterns to find deprecated APIs
- ✅ Complete migration checklist

**Educational Value:**
- ✅ Teaches the "why" behind API changes
- ✅ Shows the evolution of WoW's addon API
- ✅ Explains Blizzard's deprecation patterns
- ✅ Demonstrates best practices for future-proofing

### Impact

This guide will help addon developers:

1. **Proactively prepare** for new WoW patches/expansions
2. **Quickly migrate** existing addons to new API versions
3. **Understand** where to find official API change documentation
4. **Implement** compatibility strategies that work across versions
5. **Avoid** common migration pitfalls

### Statistics

- **File Size:** ~24 KB
- **Word Count:** ~6,500 words
- **Code Examples:** 25+ complete examples
- **Real-World Case Studies:** 3 detailed migrations
- **External Resources:** 10+ official documentation links
- **Migration Patterns:** 5 proven patterns with code

### Integration

The new guide has been integrated into:

- ✅ README.md "I want to..." section
- ✅ README.md table of contents and learning path
- ✅ README.md directory structure
- ✅ README.md Week 5+ curriculum
- ✅ 00_MASTER_PROMPT.md reference list
- ✅ C:\Dev\++Claude AI++\prompts\update_wow_addon_dev_guide.md

### Validation

The guide was validated against:
- ✅ Real-world addon migration (ArchyShadowlands 10.0 → 11.0)
- ✅ Warcraft Wiki API change documentation
- ✅ WoW Forums community discussions
- ✅ Production addon patterns (Archy, ArkInventory, ElvUI)

---

## Previous Versions

### Version 1.0 - 2025-10-19 (Initial Release)

**Initial Knowledge Base Contents:**
- 11 core documentation guides (00-10)
- QUICK_START_GUIDE.md
- README.md
- api_extracted/ directory (513 API files)
- events_extracted/ directory
- file_lists/ directory

**Guides Included:**
1. 00_MASTER_PROMPT.md
2. 01_API_Reference.md
3. 02_Event_System.md
4. 03_UI_Framework.md
5. 04_Addon_Structure.md
6. 05_Patterns_And_Best_Practices.md
7. 06_Data_Persistence.md
8. 07_Blizzard_UI_Examples.md
9. 08_Community_Addon_Patterns.md
10. 09_Addon_Libraries_Guide.md
11. 10_Advanced_Techniques.md

---

## Future Roadmap

### Completed in Version 2.0
- Major 12.0.0 (Midnight) documentation - COMPLETE
- Secret Values system documentation - COMPLETE
- Player Housing APIs (C_Housing) - COMPLETE
- API Migration Guide expansion - COMPLETE
- Combat log modernization (C_DamageMeter) - COMPLETE

### Potential Additions for Version 2.1+

**Suggested Topics:**
- Cross-addon communication patterns
- WeakAuras integration guide
- Classic Era / Season of Discovery compatibility guide
- Performance optimization deep-dive (profiling with C_AddOnProfiler)
- UI animation and effects guide (Curve/Duration objects)
- Housing addon examples and tutorials

**Maintenance Tasks:**
- Update for WoW 12.0.5 / 12.1.0 when released
- Monitor for new API changes and deprecations
- Track Secret Values system evolution
- Add new library documentation as libraries adapt to 12.0.0
- Expand real-world 12.0.0 migration examples
- Document community addon adaptations to Secret Values

---

## How to Use This Log

When updating the knowledge base:

1. Document version number and date
2. List all files added, modified, or removed
3. Explain WHY the changes were made
4. Detail WHAT problems the changes solve
5. Provide statistics and metrics
6. Note integration points with other guides
7. Include validation methods used

This helps future maintainers understand the evolution and rationale behind the knowledge base structure.

---

**Log Version:** 2.4
**Created:** 2025-10-19
**Last Updated:** 2026-01-24
**Maintained By:** AI-assisted documentation team
**Knowledge Base Status:** Active, Version 2.4 (12.0.0 Midnight)
<\!-- CLAUDE_SKIP_END -->
