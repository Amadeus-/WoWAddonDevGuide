<\!-- CLAUDE_SKIP_START -->
# WoW Addon Development Knowledge Base - Update Log

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

**Log Version:** 2.0
**Created:** 2025-10-19
**Last Updated:** 2026-01-20
**Maintained By:** AI-assisted documentation team
**Knowledge Base Status:** Active, Version 2.0 (12.0.0 Midnight)
<\!-- CLAUDE_SKIP_END -->
