# File Organization Reference

This directory contains categorized lists of all WoW UI source files for easy reference.

## File Lists

### Addon Lists
- `00_ALL_BLIZZARD_ADDONS.txt` - Complete list of all Blizzard addons (281 addons)

### By Category
- `UI_Core_Systems.md` - Core UI frameworks (SharedXML, ActionBar, UnitFrame, etc.)
- `UI_Windows_Panels.md` - Major windows and panels (Character, Spellbook, Talents, etc.)
- `Game_Systems.md` - Game mechanics (Quest, Loot, Trade, Mail, etc.)
- `Social_Communication.md` - Social features (Chat, Guild, Friends, Communities, etc.)
- `Content_Features.md` - Expansion and content features (Professions, PvP, Delves, etc.)
- `Developer_Tools.md` - Developer and debug tools (Console, EventTrace, APIDocumentation, etc.)

## How to Use

1. Find the category that matches what you're looking for
2. Open the relevant `.md` or `.txt` file
3. Locate the specific addon or file
4. Reference the full path in the WoW UI source directory

## Source Locations

**Blizzard UI Source:**
```
D:\Games\World of Warcraft\_retail_\Interface\+wow-ui-source+ (11.2.7)\Interface\AddOns\
```

**Community Addons:**
```
D:\Games\World of Warcraft\_retail_\Interface\AddOns\
```

## Quick Reference

### Most Important Addons for Development

#### Core Framework
- `Blizzard_SharedXML` - Shared UI components, mixins, utilities
- `Blizzard_FrameXML` - Base UI framework (if exists)

#### Common UI Patterns
- `Blizzard_ActionBar` - Action button implementation
- `Blizzard_BuffFrame` - Buff/debuff display
- `Blizzard_UnitFrame` - Unit frame system
- `Blizzard_Tooltip` or `Blizzard_UIPanels_Game` - Tooltip system

#### Data and Lists
- `Blizzard_AuctionHouseUI` - Modern scroll box usage
- `Blizzard_ObjectiveTracker` - Quest tracking
- `Blizzard_Collections` - Mount/pet collection UI

#### API Reference
- `Blizzard_APIDocumentationGenerated` - Complete API documentation (513 files)
- `Blizzard_APIDocumentation` - API documentation UI

---

**Version:** 1.0 - Based on WoW 11.2.7 (The War Within)
**Last Updated:** 2025-10-19
