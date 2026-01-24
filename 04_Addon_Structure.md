# WoW Addon Structure Guide

## Table of Contents
1. [TOC File Overview](#toc-file-overview)
2. [TOC Directives Reference](#toc-directives-reference)
3. [File Organization Best Practices](#file-organization-best-practices)
4. [Load Order](#load-order)
5. [C_AddOns API Reference](#c_addons-api-reference)
6. [Addon Profiling (11.0.7+)](#addon-profiling-1107)
7. [Security Changes in 12.0.0](#security-changes-in-1200-midnight)
8. [Dependencies Management](#dependencies-management)
9. [LoadOnDemand Addons](#loadondemand-addons)
10. [Addon Initialization Patterns](#addon-initialization-patterns)
11. [Module Organization](#module-organization)
12. [Library Integration](#library-integration)
13. [Localization Structure](#localization-structure)
14. [Example Addon Structures](#example-addon-structures)

---

## TOC File Overview

The TOC (Table of Contents) file is the manifest for your addon. It must:
- Have the same name as the addon folder with `.toc` extension
- Use UTF-8 encoding
- List all files to be loaded in order
- Define metadata and configuration through directives

**File naming convention:**
```
AddonName/
    AddonName.toc
    AddonName.lua
    ...
```

**Basic TOC structure:**
```
## Title: My Addon Name
## Interface: 120000
## Author: Your Name
## Version: 1.0.0
## Notes: Brief description of what the addon does
## SavedVariables: MyAddonDB
## Dependencies: SomeOtherAddon
## OptionalDeps: OptionalAddon

# Core files
Core.lua
Utils.lua

# Modules
Modules\Config.lua
Modules\UI.lua

# UI Files
UI.xml
```

---

## TOC Directives Reference

### Essential Directives

#### `## Interface: <version>`
Specifies which game version the addon is compatible with.
```
## Interface: 120000        # Version 12.0.0 (Midnight)
## Interface: 120000        # Version 12.0.0
## Interface: 0             # Works with any version (Blizzard addons only)
```
**Format:** XXYYZZ where XX=expansion, YY=major, ZZ=minor
- Example: 120000 = 12.00.00 (Midnight 12.0.0)
- Example: 110207 = 11.02.07 (The War Within 11.2.7)

#### Comma-Separated Interface Versions (10.1.0+)

Since Patch 10.1.0 (May 2023), TOC files support **comma-separated Interface versions**. This allows a single TOC file to declare compatibility with multiple WoW versions:

```
## Interface: 120000, 110207, 50503, 40402, 11508
```

**When to use comma-separated versions (single TOC file):**
- Your addon code is **identical** across all WoW versions (Retail, Classic, etc.)
- You don't need to load different files for different game clients
- Simpler maintenance with one TOC file

**When to use multiple TOC files:**
- You need to load **different files** for different game versions
- Retail version needs different features than Classic version
- You have version-specific optimizations or code paths

**Example - Single TOC with Multiple Versions:**
```
## Interface: 120000, 110207, 50503, 40402, 11508
## Title: My Universal Addon
## Version: 1.0.0

# Same files load on all versions
Core.lua
Utils.lua
```

**Example - Multiple TOC Files (when needed):**
```
MyAddon/
    MyAddon_Mainline.toc    # ## Interface: 120000
    MyAddon_Cata.toc        # ## Interface: 40402
    MyAddon_Vanilla.toc     # ## Interface: 11508
    Core.lua                # Shared
    Retail/Features.lua     # Only in Mainline TOC
    Classic/Features.lua    # Only in Classic TOCs
```

**Current Interface Versions Reference:**

| Game Version | Interface Number | Notes |
|-------------|------------------|-------|
| Retail 12.0.0 (Midnight) | 120000 | Current expansion |
| Retail 12.0.1 | 120001 | Future patches |
| TWW 11.2.7 | 110207 | Previous expansion |
| Classic Cata 4.4.2 | 40402 | Cataclysm Classic |
| Classic SoD/Era 1.15.8 | 11508 | Season of Discovery / Classic Era |
| Wrath Classic 3.4.3 | 30403 | WotLK Classic |
| TBC Classic 2.5.4 | 20504 | Burning Crusade Classic |

**Best Practice Recommendation:**
For most addons, use **comma-separated versions** in a single TOC file unless you specifically need different file loading per version. This simplifies maintenance and ensures consistent behavior across clients.

#### `## Title: <name>`
Display name shown in the addon list.
```
## Title: My Awesome Addon
## Title: |cff00ff00Green|r Colored Title    # With color codes
```

#### `## Author: <name>`
Addon author/creator.
```
## Author: YourName
## Author: Blizzard Entertainment
```

#### `## Version: <version>`
Version number for your addon.
```
## Version: 1.0.0
## Version: 2.3.1-beta
```

#### `## Notes: <description>`
Short description shown in the addon list.
```
## Notes: Provides enhanced raid frames
## Notes: All aboard for da' island! (Island Queue Screen)
```

---

### SavedVariables Directives

#### `## SavedVariables: <variables>`
Global saved variables (account-wide, all characters).
```
## SavedVariables: MyAddonDB
## SavedVariables: GlobalSettings, GlobalCache
```

Variables are automatically saved/loaded to `WTF/Account/<account>/SavedVariables/AddonName.lua`

#### `## SavedVariablesPerCharacter: <variables>`
Per-character saved variables.
```
## SavedVariablesPerCharacter: MyAddonCharDB
## SavedVariablesPerCharacter: CharSettings, CharData
```

Variables are saved to `WTF/Account/<account>/<realm>/<character>/SavedVariables/AddonName.lua`

#### `## SavedVariablesMachine: <variables>`
Per-computer saved variables (not synced between machines).
```
## SavedVariablesMachine: LocalCache
## SavedVariablesMachine: g_addonCategoriesCollapsed
```

Variables are saved to `WTF/Config.wtf` area (machine-specific).

**Example from Blizzard addons:**
```
## SavedVariables: g_auctionHouseSortsBySearchContext
## SavedVariablesPerCharacter: g_activeBidAuctionIDs
```

---

### Dependency Directives

#### `## Dependencies: <addon1>, <addon2>, ...`
Required addons that must load first. Addon fails to load if dependencies are missing.
```
## Dependencies: LibStub, AceAddon-3.0
## Dependencies: Blizzard_SharedXML, Blizzard_Colors
```

#### `## OptionalDeps: <addon1>, <addon2>, ...`
Also written as `## OptionalDep:`
Optional addons that will load first if present, but not required.
```
## OptionalDeps: LibSharedMedia-3.0, AceGUI-3.0
## OptionalDep: Blizzard_UIParentPanelManager
```

#### `## RequiredDep: <addon1>, <addon2>, ...`
Alternative syntax for Dependencies (used in some Blizzard addons).
```
## RequiredDep: Blizzard_MapCanvas, Blizzard_SharedMapDataProviders
```

**Difference between Dependencies and OptionalDeps:**
- `Dependencies`: Hard requirement - addon won't load without them
- `OptionalDeps`: Soft requirement - loads if available, ensures load order
- Use `OptionalDeps` for libraries you can detect and work without

---

### Load Control Directives

#### `## LoadOnDemand: <0|1>`
Controls when the addon loads.
```
## LoadOnDemand: 0    # Load at startup (default)
## LoadOnDemand: 1    # Load only when requested
```

When `LoadOnDemand: 1`, the addon must be loaded via:
```lua
-- Check if loaded
if IsAddOnLoaded("AddonName") then
    -- Use addon
end

-- Load the addon
local loaded, reason = LoadAddOn("AddonName")
if loaded then
    -- Initialize
else
    print("Failed to load:", reason)
end
```

#### `## LoadWith: <addon1>, <addon2>, ...`
Load this addon when specified addons load (only for LoadOnDemand addons).
```
## LoadOnDemand: 1
## LoadWith: Blizzard_UIParent
```

#### `## LoadFirst: 1`
Ensures this addon loads before others (Blizzard addons only).
```
## LoadFirst: 1
```

#### `## LoadSavedVariablesFirst: 1`
Loads saved variables before the addon code executes (11.1.5+).
```
## LoadSavedVariablesFirst: 1
```

This is particularly useful when:
- Your addon needs saved data during the initial script execution
- You want to access settings before module initialization
- You need to configure the addon based on user preferences at load time

Without this directive, saved variables are available only after all addon files have loaded (at `ADDON_LOADED` event).

---

### Game Environment Directives

#### `## AllowLoad: <Game|Glue|Both>`
Controls where the addon can load.
```
## AllowLoad: Game      # In-game only
## AllowLoad: Glue      # Login screen only
## AllowLoad: Both      # Both game and glue
```

- `Game`: Normal in-game environment (most addons)
- `Glue`: Login/character select screen
- `Both`: Works in both environments

#### `## AllowLoadGameType: <type1>, <type2>, ...`
Restricts loading to specific game variants. As of 11.1.5+, this directive is available to all addons (was previously secure-only).
```
## AllowLoadGameType: mainline                    # Retail only
## AllowLoadGameType: classic                     # Classic Era
## AllowLoadGameType: vanilla, tbc, wrath         # Multiple versions
## AllowLoadGameType: plunderstorm               # Special game modes
```

**Available game types:**
- `mainline` - Retail WoW
- `classic` - Classic Era
- `vanilla` - Classic Vanilla
- `tbc` - Burning Crusade Classic
- `wrath` - Wrath of the Lich King Classic
- `cata` - Cataclysm Classic
- `mists` - Mists of Pandaria
- `plunderstorm` - Plunderstorm game mode
- `wowhack` - Special development mode

**Per-file loading restrictions (11.1.5+):**
```
# In TOC file - inline AllowLoadGameType for specific files
MainlineFeature.lua [AllowLoadGameType mainline]
ClassicFeature.lua [AllowLoadGameType classic]
SharedFeature.lua
```

---

### Addon Organization Directives (11.1.0+)

#### `## Category: <name>`
Groups addons under collapsible headers in the addon list (11.1.0+).
```
## Category: My Addon Category
## Category-deDE: Meine Addon-Kategorie
## Category-frFR: Ma Categorie d'Addon
```

Use localized variants with `Category-<locale>:` for different languages. This helps users organize related addons together in their addon management interface.

#### `## Group: <ParentAddonName>`
Links related addons together under a parent addon (11.1.0+).
```
## Group: MyMainAddon
```

This is useful for addon suites where multiple addons share functionality:
```
# MyAddon_Options.toc
## Title: MyAddon - Options
## Group: MyAddon
## LoadOnDemand: 1

# MyAddon_Debug.toc
## Title: MyAddon - Debug Tools
## Group: MyAddon
## LoadOnDemand: 1
```

---

### Display and Behavior Directives

#### `## DefaultState: <enabled|disabled>`
Default state in addon list.
```
## DefaultState: enabled     # Enabled by default
## DefaultState: disabled    # Disabled by default
```

#### `## Secure: 1`
Marks addon as secure (can use secure functions in combat).
```
## Secure: 1
```
Only applies to Blizzard signed addons.

#### `## ShowInAddOnList: <0|1>`
Controls visibility in addon list.
```
## ShowInAddOnList: 0    # Hidden from list
## ShowInAddOnList: 1    # Visible (default)
```

#### `## ShowInDebugList: <0|1>`
Controls visibility in debug addon list.
```
## ShowInDebugList: 1    # Show in debug list
```

---

### Development and Testing Directives

#### `## OnlyBetaAndPTR: 1`
Only loads on Beta/PTR realms.
```
## OnlyBetaAndPTR: 1
```

#### `## EscalateErrorDuringLoad: 1`
Treats any error during load as fatal.
```
## EscalateErrorDuringLoad: 1
```

#### `## UseSecureEnvironment: 1`
Forces secure execution environment.
```
## UseSecureEnvironment: 1
```

#### `## SuppressLocalTableRef: 1`
Suppresses local table reference warnings.
```
## SuppressLocalTableRef: 1
```

#### `## AllowAddOnTableAccess: 1`
Allows other addons to access this addon's namespace table via `C_AddOns.GetAddOnLocalTable()` (11.1.7+).
```
## AllowAddOnTableAccess: 1
```

When enabled, other addons can retrieve your addon's local table:
```lua
-- In another addon
local otherAddonTable = C_AddOns.GetAddOnLocalTable("MyAddon")
if otherAddonTable then
    -- Access shared data/functions
    otherAddonTable.SomeFunction()
end
```

This provides a cleaner alternative to exposing global tables for addon interoperability.

---

### Special Metadata Directives

#### `## IconTexture: <path>`
Icon shown in addon list.
```
## IconTexture: Interface\ICONS\inv_misc_note_06
```

#### Custom/Informational Directives
Any directive starting with `## X-` is custom and ignored by WoW:
```
## X-Website: https://example.com
## X-Email: author@example.com
## X-License: MIT
## X-Curse-Project-ID: 12345
## X-WoWI-ID: 67890
```

---

### TOC Inline Variables (11.1.5+)

TOC files support inline variables that expand based on the game environment. This allows a single TOC to load different files for different game variants.

#### Family Variable `[Family]`
Expands based on game family (Mainline vs Classic):
```
[Family]\Init.lua
# Expands to:
# - Mainline\Init.lua (on Retail)
# - Classic\Init.lua (on Classic versions)
```

#### Game Variable `[Game]`
Expands based on specific game version:
```
[Game]\Features.lua
# Expands to:
# - Standard\Features.lua (Retail)
# - Mists\Features.lua (MoP Remix)
# - Vanilla\Features.lua (Classic Era)
# - Wrath\Features.lua (WotLK Classic)
# - Cata\Features.lua (Cata Classic)
```

#### TextLocale Variable `[TextLocale]`
Expands based on client locale:
```
Locales\[TextLocale].lua
# Expands to:
# - Locales\enUS.lua (English US)
# - Locales\deDE.lua (German)
# - Locales\frFR.lua (French)
# - Locales\esES.lua (Spanish Spain)
# - Locales\zhCN.lua (Chinese Simplified)
# etc.
```

#### Conditional File Loading
Combine inline variables with AllowLoadGameType conditions:
```
# Load different files based on game type
MainlineFeatures.lua [AllowLoadGameType mainline]
ClassicFeatures.lua [AllowLoadGameType vanilla, tbc, wrath, cata]

# Using Family variable
[Family]\Core.lua
[Family]\UI.lua

# Conditional with variables
RetailUI.lua [AllowLoadGameType mainline]
ClassicUI.lua [AllowLoadGameType classic]
```

**Complete example using inline variables:**
```
## Interface: 120000
## Title: MultiPlatform Addon
## Version: 1.0.0

# Shared core
Core.lua

# Family-specific initialization
[Family]\Init.lua

# Locale-specific strings (loads only matching locale file)
Locales\[TextLocale].lua

# Conditional modules
RetailModule.lua [AllowLoadGameType mainline]
ClassicModule.lua [AllowLoadGameType vanilla, tbc, wrath]
```

---

## File Organization Best Practices

### Single-File Addon (Simple)
```
MyAddon/
    MyAddon.toc
    MyAddon.lua
```

### Basic Multi-File Structure
```
MyAddon/
    MyAddon.toc          # TOC file
    Core.lua              # Core initialization
    Config.lua            # Configuration
    Utils.lua             # Utility functions
    UI.lua                # UI code
    Locales.lua           # Localization
```

### Modular Structure (Recommended)
```
MyAddon/
    MyAddon.toc

    # Core
    Core.lua              # Addon initialization
    Constants.lua         # Constants and enums

    # Libraries (embed if needed)
    Libs/
        LibStub.lua
        CallbackHandler-1.0.lua
        AceAddon-3.0.lua

    # Localization
    Locales/
        enUS.lua          # English (default)
        deDE.lua          # German
        frFR.lua          # French
        esES.lua          # Spanish
        zhCN.lua          # Chinese Simplified

    # Modules
    Modules/
        Database.lua
        Events.lua
        Commands.lua
        Config.lua

    # UI
    UI/
        MainFrame.lua
        MainFrame.xml
        OptionsPanel.lua
        OptionsPanel.xml

    # Data
    Data/
        Items.lua
        Spells.lua
```

### Large Addon Structure
```
MyAddon/
    MyAddon.toc
    MyAddon_Mainline.toc   # Retail-specific TOC
    MyAddon_Classic.toc     # Classic-specific TOC

    Init.lua               # Pre-initialization
    Core.lua               # Main initialization

    Libs/                  # Embedded libraries
        LibStub/
        AceAddon-3.0/
        AceDB-3.0/
        AceConfig-3.0/
        AceGUI-3.0/

    Locales/
        Locale.lua         # Locale loader
        enUS.lua
        [other locales...]

    Core/                  # Core systems
        Database.lua
        Events.lua
        Utilities.lua
        API.lua

    Modules/               # Feature modules
        Module1/
            Module1.lua
            Module1.xml
        Module2/
            Module2.lua
            Submodule.lua

    UI/                    # User interface
        Templates/
            Templates.xml
        Frames/
            MainFrame.lua
            MainFrame.xml
        Dialogs/
            Dialogs.lua
            Dialogs.xml

    Config/                # Configuration
        Defaults.lua
        Options.lua
        Profiles.lua

    Data/                  # Static data
        Constants.lua
        Tables.lua
```

---

## Load Order

### General Load Order Rules

1. **Dependencies load first**
   - `Dependencies` addons load before your addon
   - `OptionalDeps` load before your addon (if present)
   - `LoadFirst` ensures priority loading

2. **Files load in TOC order**
   - Files are loaded in the exact order listed in the TOC
   - Lua files load before XML files at the same level
   - XML files can reference Lua objects already loaded

3. **Saved variables load timing**
   - Without `LoadSavedVariablesFirst`: Variables available after all files load
   - With `LoadSavedVariablesFirst`: Variables available before first file loads

### File Type Loading

**Standard pattern:**
```
# TOC file - typical loading pattern
Init.lua              # Loads first
Core.lua              # Then core
Utils.lua             # Then utilities
Templates.xml         # XML after Lua
MainFrame.lua         # Frame logic
MainFrame.xml         # Frame XML
```

**Lua before XML rule:**
- Always list `.lua` files before corresponding `.xml` files
- XML can reference Lua functions/tables, not vice versa
- Templates in XML must be defined before use

### Load Order Example

```
## MyAddon.toc

# Libraries (load first)
Libs\LibStub.lua
Libs\AceAddon-3.0.lua

# Core initialization (needs libraries)
Init.lua
Core.lua

# Localization (before modules need it)
Locales\enUS.lua
Locales\Locale.lua

# Data (before modules)
Data\Constants.lua
Data\Items.lua

# Modules (can use core, locales, data)
Modules\Database.lua
Modules\Events.lua

# UI (last - uses everything)
UI\Templates.xml          # XML templates
UI\MainFrame.lua          # Frame logic
UI\MainFrame.xml          # Frame definition
```

### Initialization Timing

```lua
-- File load time (when file is parsed)
local MyAddon = {}

-- Fires for each addon after its files load
function MyAddon:OnInitialize()
    -- Saved variables are available here (if LoadSavedVariablesFirst)
end

-- Fires when all addons are loaded
local frame = CreateFrame("Frame")
frame:RegisterEvent("ADDON_LOADED")
frame:SetScript("OnEvent", function(self, event, addonName)
    if addonName == "MyAddon" then
        -- This addon just loaded
        -- Saved variables are definitely available
    end
end)

-- Fires when player enters world
frame:RegisterEvent("PLAYER_LOGIN")
frame:SetScript("OnEvent", function(self, event)
    if event == "PLAYER_LOGIN" then
        -- Player is fully loaded
        -- All game data available
    end
end)
```

---

## C_AddOns API Reference

The `C_AddOns` namespace provides functions for querying and managing addon information programmatically.

### Core Functions

```lua
-- Get number of addons
local count = C_AddOns.GetNumAddOns()

-- Get addon info by index or name
local name, title, notes, loadable, reason, security = C_AddOns.GetAddOnInfo(indexOrName)

-- Check if addon is loaded
local isLoaded = C_AddOns.IsAddOnLoaded(indexOrName)

-- Load an addon (LoadOnDemand addons)
local loaded, reason = C_AddOns.LoadAddOn(name)

-- Enable/disable addon for next session
C_AddOns.EnableAddOn(name, character)
C_AddOns.DisableAddOn(name, character)

-- Check enable state
local enabled = C_AddOns.GetAddOnEnableState(name, character)
```

### New Functions (11.x and 12.0)

```lua
-- Get addon interface version (11.0.5+)
local interfaceVersion = C_AddOns.GetAddOnInterfaceVersion(name)

-- Access another addon's namespace table (11.1.7+)
-- Requires ## AllowAddOnTableAccess: 1 in target addon's TOC
local addonTable = C_AddOns.GetAddOnLocalTable("OtherAddonName")

-- Check for load errors (11.0.5+)
local hasError = C_AddOns.DoesAddOnHaveLoadError(name)

-- Check if addon is default enabled (11.1.0+)
local isDefaultEnabled = C_AddOns.IsAddOnDefaultEnabled(name)

-- Individual info functions (cleaner than GetAddOnInfo)
local name = C_AddOns.GetAddOnName(index)
local notes = C_AddOns.GetAddOnNotes(indexOrName)
local security = C_AddOns.GetAddOnSecurity(indexOrName)
local title = C_AddOns.GetAddOnTitle(indexOrName)
```

### Usage Examples

```lua
-- List all loaded addons
local count = C_AddOns.GetNumAddOns()
for i = 1, count do
    local name = C_AddOns.GetAddOnName(i)
    if C_AddOns.IsAddOnLoaded(name) then
        local title = C_AddOns.GetAddOnTitle(name)
        local version = C_AddOns.GetAddOnInterfaceVersion(name)
        print(format("%s (Interface: %d)", title, version))
    end
end

-- Access another addon's API (requires AllowAddOnTableAccess)
local DBM = C_AddOns.GetAddOnLocalTable("DBM-Core")
if DBM and DBM.RegisterCallback then
    DBM.RegisterCallback(myCallback)
end
```

---

## Addon Profiling (11.0.7+)

The `C_AddOnProfiler` namespace provides performance metrics for addon debugging and optimization.

### Enabling Profiling

Addon profiling must be enabled via:
1. `/console scriptProfile 1` (requires UI reload)
2. Or enable in Interface Options > AddOns > Addon CPU Usage

### Profiler Functions

```lua
-- Check if profiling is enabled
local isEnabled = C_AddOnProfiler.IsEnabled()

-- Get specific metric for an addon
local value = C_AddOnProfiler.GetAddOnMetric(addonName, metric)

-- Get overall metric across all addons
local value = C_AddOnProfiler.GetOverallMetric(metric)

-- Measure a specific function call (11.1.7+)
local elapsed = C_AddOnProfiler.MeasureCall(func, ...)

-- Get profiler tick rate (11.1.7+)
local ticksPerSecond = C_AddOnProfiler.GetTicksPerSecond()
```

### Available Metrics

| Metric | Description |
|--------|-------------|
| `SessionAverageTime` | Average time per frame across session |
| `RecentAverageTime` | Average time per frame (recent samples) |
| `EncounterAverageTime` | Average time during boss encounters |
| `LastTime` | Time taken in last frame |
| `PeakTime` | Maximum time recorded |
| `CountTimeOver1Ms` | Frames exceeding 1ms |
| `CountTimeOver5Ms` | Frames exceeding 5ms |
| `CountTimeOver10Ms` | Frames exceeding 10ms |
| `CountTimeOver50Ms` | Frames exceeding 50ms |

### Profiling Examples

```lua
-- Check addon performance
if C_AddOnProfiler.IsEnabled() then
    local avgTime = C_AddOnProfiler.GetAddOnMetric("MyAddon", "RecentAverageTime")
    local peakTime = C_AddOnProfiler.GetAddOnMetric("MyAddon", "PeakTime")
    local slowFrames = C_AddOnProfiler.GetAddOnMetric("MyAddon", "CountTimeOver5Ms")

    print(format("MyAddon - Avg: %.3fms, Peak: %.3fms, Slow frames: %d",
        avgTime * 1000, peakTime * 1000, slowFrames))
end

-- Measure specific function performance (11.1.7+)
local function HeavyOperation()
    -- Complex calculation
end

local elapsed = C_AddOnProfiler.MeasureCall(HeavyOperation)
print(format("Operation took %.3fms", elapsed * 1000))
```

### Performance Optimization Tips

Based on profiler data:
- `RecentAverageTime > 1ms`: Consider optimizing frequent operations
- `CountTimeOver5Ms > 0`: Look for expensive operations causing hitches
- `PeakTime` high: Investigate startup or event burst handling
- `EncounterAverageTime` high: Optimize combat-related code

---

## Security Changes in 12.0.0 (Midnight)

The 12.0.0 expansion introduces significant security changes affecting addon development:

### Secret Values System

12.0.0 expands the "secret values" system for combat security:
- Certain API return values become "tainted" during combat
- Tainted values cannot be used for secure operations
- Addons may need to cache values before combat or handle nil returns

### Impact on Addons

- **Action button addons**: May need additional checks for tainted values
- **Unit frame addons**: Some unit information may be restricted during combat
- **Macro/automation addons**: Increased restrictions on programmatic actions

### Recommendations

1. Test addons thoroughly on PTR before 12.0.0 launch
2. Cache important values during `PLAYER_REGEN_ENABLED`
3. Check `InCombatLockdown()` before secure operations
4. Reference `11_API_Migration_Guide.md` for detailed migration steps

---

## Dependencies Management

### Choosing Dependency Types

**Use `Dependencies` when:**
- Addon cannot function without it
- You use embedded libraries that must exist
- You extend another addon's functionality

**Use `OptionalDeps` when:**
- You can detect and work without the dependency
- You want to add features if a library is present
- You need load order but not requirement

**Use `RequiredDep` when:**
- Same as `Dependencies` (Blizzard convention)
- Seen in newer Blizzard addons

### Dependency Examples

#### Hard Dependency
```
## Dependencies: LibStub, AceAddon-3.0

# In code - safe to use immediately
local AceAddon = LibStub("AceAddon-3.0")
local MyAddon = AceAddon:NewAddon("MyAddon")
```

#### Optional Dependency
```
## OptionalDeps: LibSharedMedia-3.0

# In code - check before use
local LSM = LibStub:GetLibrary("LibSharedMedia-3.0", true)
if LSM then
    -- Use LSM features
    LSM:Register("font", "MyFont", [[Fonts\MyFont.ttf]])
else
    -- Fallback without LSM
    print("LibSharedMedia not available")
end
```

#### Embedding Libraries
```
# TOC file - embed AceAddon
Libs\LibStub\LibStub.lua
Libs\CallbackHandler-1.0\CallbackHandler-1.0.lua
Libs\AceAddon-3.0\AceAddon-3.0.lua

# No Dependencies directive needed - libraries are embedded
```

#### Cross-Addon Dependencies
```
## MyTooltipAddon.toc
## Title: My Tooltip Addon
## Dependencies: MyMainAddon

# Lua file
local MyMainAddon = _G.MyMainAddon
if MyMainAddon then
    -- Extend main addon
    MyMainAddon:RegisterTooltipProvider(MyTooltipAddon)
end
```

### Circular Dependencies

**Problem:**
```
AddonA: ## Dependencies: AddonB
AddonB: ## Dependencies: AddonA
```
Result: Neither loads!

**Solution:** Use OptionalDeps for one direction
```
AddonA: ## Dependencies: AddonB
AddonB: ## OptionalDeps: AddonA
```

---

## LoadOnDemand Addons

### When to Use LoadOnDemand

**Good uses:**
- UI panels that aren't always needed (options, configuration)
- Feature modules users may not use
- Memory-intensive components
- Integration with other addons

**Bad uses:**
- Core functionality
- Event handlers needed at login
- Small addons (overhead not worth it)

### LoadOnDemand Implementation

#### Basic LoadOnDemand
```
## MyAddon_Options.toc
## Title: MyAddon Options
## LoadOnDemand: 1
## Dependencies: MyAddon
## OptionalDeps: AceConfig-3.0

Options.lua
OptionsUI.xml
```

```lua
-- In main addon - load when needed
local function ShowOptions()
    local loaded, reason = LoadAddOn("MyAddon_Options")
    if loaded then
        MyAddon_Options:Show()
    else
        print("Failed to load options:", reason)
    end
end
```

#### LoadWith Example
```
## MyAddon_GuildFeatures.toc
## Title: MyAddon Guild Features
## LoadOnDemand: 1
## LoadWith: Blizzard_GuildUI
## Dependencies: MyAddon

GuildFeatures.lua
```

Automatically loads when guild frame opens.

#### Manual Load Control
```lua
-- Check if already loaded
if not IsAddOnLoaded("MyAddon_Options") then
    -- Check if available
    local name, title, notes, loadable, reason = GetAddOnInfo("MyAddon_Options")

    if loadable then
        LoadAddOn("MyAddon_Options")
    else
        print("Cannot load:", reason)
        -- Reasons: "DISABLED", "MISSING", "INCOMPATIBLE", "CORRUPT"
    end
end

-- Check if load-on-demand
local _, _, _, loadOnDemand = GetAddOnInfo("MyAddon_Options")
if loadOnDemand then
    print("This is a load-on-demand addon")
end
```

### LoadOnDemand Event Handling

```lua
-- Detect when LOD addon loads
local frame = CreateFrame("Frame")
frame:RegisterEvent("ADDON_LOADED")
frame:SetScript("OnEvent", function(self, event, addonName)
    if addonName == "MyAddon_Options" then
        -- Options addon just loaded
        MyAddon_Options:Initialize(MyAddon)
        frame:UnregisterEvent("ADDON_LOADED")
    end
end)
```

---

## Addon Initialization Patterns

### Pattern 1: Simple Global Table

**Best for:** Small addons, simple functionality

```lua
-- MyAddon.lua
MyAddon = {}
MyAddon.version = "1.0.0"

function MyAddon:Initialize()
    print("MyAddon loaded!")
end

-- Create event frame
local frame = CreateFrame("Frame")
frame:RegisterEvent("ADDON_LOADED")
frame:SetScript("OnEvent", function(self, event, addonName)
    if addonName == "MyAddon" then
        MyAddon:Initialize()
        self:UnregisterEvent("ADDON_LOADED")
    end
end)
```

### Pattern 2: AceAddon Framework

**Best for:** Medium to large addons, using Ace3 libraries

```lua
-- Core.lua
local AceAddon = LibStub("AceAddon-3.0")
local MyAddon = AceAddon:NewAddon("MyAddon", "AceEvent-3.0", "AceConsole-3.0")

function MyAddon:OnInitialize()
    -- Called when addon is first loaded
    self.db = LibStub("AceDB-3.0"):New("MyAddonDB", defaults)
end

function MyAddon:OnEnable()
    -- Called when addon is enabled
    self:RegisterEvent("PLAYER_LOGIN")
    self:RegisterChatCommand("myaddon", "SlashCommand")
end

function MyAddon:OnDisable()
    -- Called when addon is disabled
end

function MyAddon:PLAYER_LOGIN()
    self:Print("Player logged in!")
end
```

### Pattern 3: Namespaced Module Pattern

**Best for:** Large addons, avoiding global pollution

```lua
-- Init.lua - Create namespace
local ADDON_NAME, ns = ...
ns.version = "1.0.0"

-- Core.lua - Use namespace
local ADDON_NAME, ns = ...

ns.Core = {}

function ns.Core:Initialize()
    print(ADDON_NAME .. " v" .. ns.version)
end

-- Events.lua - Add to namespace
local ADDON_NAME, ns = ...

local frame = CreateFrame("Frame")
frame:RegisterEvent("ADDON_LOADED")
frame:SetScript("OnEvent", function(self, event, addonName)
    if addonName == ADDON_NAME then
        ns.Core:Initialize()
    end
end)
```

### Pattern 4: Object-Oriented Module System

**Best for:** Very large addons, clean separation of concerns

```lua
-- Core.lua
local ADDON_NAME, ns = ...

local MyAddon = {}
ns.MyAddon = MyAddon

MyAddon.modules = {}

function MyAddon:RegisterModule(name, module)
    self.modules[name] = module
    if module.OnInitialize then
        module:OnInitialize(self)
    end
end

function MyAddon:GetModule(name)
    return self.modules[name]
end

-- Module1.lua
local ADDON_NAME, ns = ...
local MyAddon = ns.MyAddon

local Module1 = {}

function Module1:OnInitialize(core)
    self.core = core
    print("Module1 initialized")
end

function Module1:DoSomething()
    -- Module functionality
end

MyAddon:RegisterModule("Module1", Module1)

-- Usage
local module = MyAddon:GetModule("Module1")
module:DoSomething()
```

### Pattern 5: Event-Driven Initialization

**Best for:** Complex initialization sequences

```lua
local MyAddon = {}
local initialized = false
local loginComplete = false

-- Stage 1: Addon files loaded
local function OnAddonLoaded()
    -- Saved variables available
    MyAddon.db = MyAddonDB or {}
    MyAddon:LoadModules()
end

-- Stage 2: Player enters world
local function OnPlayerLogin()
    -- Game data available
    MyAddon:InitializeUI()
    loginComplete = true
end

-- Stage 3: Player entering world complete
local function OnPlayerEnteringWorld()
    if not initialized then
        MyAddon:CompleteInitialization()
        initialized = true
    end
end

local frame = CreateFrame("Frame")
frame:RegisterEvent("ADDON_LOADED")
frame:RegisterEvent("PLAYER_LOGIN")
frame:RegisterEvent("PLAYER_ENTERING_WORLD")

frame:SetScript("OnEvent", function(self, event, ...)
    if event == "ADDON_LOADED" and select(1, ...) == "MyAddon" then
        OnAddonLoaded()
    elseif event == "PLAYER_LOGIN" then
        OnPlayerLogin()
    elseif event == "PLAYER_ENTERING_WORLD" then
        OnPlayerEnteringWorld()
    end
end)
```

---

## Module Organization

### Module Registration System

```lua
-- Core.lua
local ADDON_NAME, ns = ...

ns.modules = {}
ns.moduleOrder = {}

function ns:RegisterModule(name, module)
    if self.modules[name] then
        error("Module already registered: " .. name)
    end

    self.modules[name] = module
    table.insert(self.moduleOrder, name)

    if module.Initialize then
        module:Initialize()
    end
end

function ns:GetModule(name)
    return self.modules[name]
end

function ns:InitializeAllModules()
    for _, name in ipairs(self.moduleOrder) do
        local module = self.modules[name]
        if module.OnEnable then
            module:OnEnable()
        end
    end
end

-- Module template
local function CreateModule(name)
    local module = {
        name = name,
        enabled = false
    }

    function module:Initialize()
        -- Setup
    end

    function module:OnEnable()
        self.enabled = true
        -- Enable module
    end

    function module:OnDisable()
        self.enabled = false
        -- Cleanup
    end

    return module
end
```

### Module Examples

#### Database Module
```lua
-- Modules/Database.lua
local ADDON_NAME, ns = ...

local Database = CreateModule("Database")

function Database:Initialize()
    self.cache = {}
end

function Database:Set(key, value)
    self.cache[key] = value
end

function Database:Get(key)
    return self.cache[key]
end

ns:RegisterModule("Database", Database)
```

#### Events Module
```lua
-- Modules/Events.lua
local ADDON_NAME, ns = ...

local Events = CreateModule("Events")

function Events:Initialize()
    self.frame = CreateFrame("Frame")
    self.callbacks = {}
end

function Events:RegisterCallback(event, callback)
    if not self.callbacks[event] then
        self.callbacks[event] = {}
        self.frame:RegisterEvent(event)
    end
    table.insert(self.callbacks[event], callback)
end

function Events:OnEnable()
    self.frame:SetScript("OnEvent", function(frame, event, ...)
        if self.callbacks[event] then
            for _, callback in ipairs(self.callbacks[event]) do
                callback(event, ...)
            end
        end
    end)
end

ns:RegisterModule("Events", Events)
```

---

## Library Integration

### LibStub Pattern

LibStub is the standard library management system for WoW addons.

```lua
-- Embedding LibStub
## TOC file
Libs\LibStub\LibStub.lua
Libs\CallbackHandler-1.0\CallbackHandler-1.0.lua
Libs\AceAddon-3.0\AceAddon-3.0.lua

-- Using LibStub
local LibStub = LibStub
local AceAddon = LibStub("AceAddon-3.0")

-- Check if library exists (safe)
local AceDB = LibStub("AceDB-3.0", true)  -- true = silent fail
if AceDB then
    -- Use AceDB
end
```

### Ace3 Integration

#### AceAddon
```lua
local AceAddon = LibStub("AceAddon-3.0")
local MyAddon = AceAddon:NewAddon("MyAddon", "AceEvent-3.0", "AceConsole-3.0")

function MyAddon:OnInitialize()
    -- Called when addon loads
end

function MyAddon:OnEnable()
    -- Called when addon is enabled
end
```

#### AceDB (Database)
```lua
local AceDB = LibStub("AceDB-3.0")

local defaults = {
    profile = {
        enabled = true,
        scale = 1.0,
        position = { x = 0, y = 0 }
    },
    global = {
        version = "1.0.0"
    }
}

function MyAddon:OnInitialize()
    self.db = AceDB:New("MyAddonDB", defaults, true)
    -- true = use default profile
end

-- Access data
local enabled = MyAddon.db.profile.enabled
MyAddon.db.profile.scale = 1.5
```

#### AceConfig (Options)
```lua
local AceConfig = LibStub("AceConfig-3.0")
local AceConfigDialog = LibStub("AceConfigDialog-3.0")

local options = {
    type = "group",
    name = "MyAddon",
    args = {
        enabled = {
            type = "toggle",
            name = "Enabled",
            desc = "Enable the addon",
            get = function() return MyAddon.db.profile.enabled end,
            set = function(_, value) MyAddon.db.profile.enabled = value end,
        },
        scale = {
            type = "range",
            name = "Scale",
            desc = "UI scale",
            min = 0.5,
            max = 2.0,
            step = 0.1,
            get = function() return MyAddon.db.profile.scale end,
            set = function(_, value) MyAddon.db.profile.scale = value end,
        }
    }
}

AceConfig:RegisterOptionsTable("MyAddon", options)
AceConfigDialog:AddToBlizOptions("MyAddon", "MyAddon")
```

### LibSharedMedia

```lua
## OptionalDeps: LibSharedMedia-3.0

local LSM = LibStub("LibSharedMedia-3.0", true)

if LSM then
    -- Register custom media
    LSM:Register("font", "MyFont", [[Fonts\MyFont.ttf]])
    LSM:Register("background", "MyTexture", [[Textures\Background.tga]])
    LSM:Register("sound", "MySound", [[Sounds\Alert.ogg]])

    -- Use registered media
    local fontPath = LSM:Fetch("font", "MyFont")
    fontString:SetFont(fontPath, 12, "OUTLINE")
end
```

### LibDataBroker

```lua
local LDB = LibStub("LibDataBroker-1.1", true)

if LDB then
    local dataObject = LDB:NewDataObject("MyAddon", {
        type = "data source",
        icon = "Interface\\Icons\\inv_misc_note_01",
        label = "MyAddon",
        text = "0/100",

        OnClick = function(self, button)
            if button == "LeftButton" then
                -- Handle click
            end
        end,

        OnTooltipShow = function(tooltip)
            tooltip:AddLine("MyAddon")
            tooltip:AddLine("Click to toggle")
        end
    })

    -- Update data
    dataObject.text = "50/100"
end
```

---

## Localization Structure

### Basic Localization

```lua
-- Locales/enUS.lua (default locale)
local L = {}
L["Hello"] = "Hello"
L["Goodbye"] = "Goodbye"
L["Welcome %s"] = "Welcome %s"

-- Store in addon namespace
local ADDON_NAME, ns = ...
ns.L = L
```

```lua
-- Locales/deDE.lua (German)
local L = ns.L  -- Get existing table

if GetLocale() == "deDE" then
    L["Hello"] = "Hallo"
    L["Goodbye"] = "Auf Wiedersehen"
    L["Welcome %s"] = "Willkommen %s"
end
```

### AceLocale Integration

```lua
-- Locales/enUS.lua
local L = LibStub("AceLocale-3.0"):NewLocale("MyAddon", "enUS", true)
if not L then return end

L["Hello"] = true
L["Goodbye"] = true
L["Welcome %s"] = true
```

```lua
-- Locales/deDE.lua
local L = LibStub("AceLocale-3.0"):NewLocale("MyAddon", "deDE")
if not L then return end

L["Hello"] = "Hallo"
L["Goodbye"] = "Auf Wiedersehen"
```

```lua
-- Core.lua - Usage
local L = LibStub("AceLocale-3.0"):GetLocale("MyAddon")

print(L["Hello"])
print(string.format(L["Welcome %s"], playerName))
```

### Locale File Organization

```
## TOC file
Locales\Locale.xml      # Load all locales

# Or load individually
Locales\enUS.lua
Locales\deDE.lua
Locales\frFR.lua
Locales\esES.lua
Locales\zhCN.lua
Locales\zhTW.lua
Locales\koKR.lua
Locales\ruRU.lua
```

```xml
<!-- Locales/Locale.xml -->
<Ui xmlns="http://www.blizzard.com/wow/ui/">
    <Script file="enUS.lua"/>
    <Script file="deDE.lua"/>
    <Script file="frFR.lua"/>
    <Script file="esES.lua"/>
    <Script file="zhCN.lua"/>
    <Script file="zhTW.lua"/>
    <Script file="koKR.lua"/>
    <Script file="ruRU.lua"/>
</Ui>
```

---

## Example Addon Structures

### Example 1: Simple Addon

**Purpose:** Display a message on login

```
SimpleAddon/
    SimpleAddon.toc
    SimpleAddon.lua
```

**SimpleAddon.toc:**
```
## Interface: 120000
## Title: Simple Addon
## Author: YourName
## Version: 1.0.0
## Notes: Shows a welcome message

SimpleAddon.lua
```

**SimpleAddon.lua:**
```lua
local frame = CreateFrame("Frame")
frame:RegisterEvent("PLAYER_LOGIN")
frame:SetScript("OnEvent", function(self, event)
    print("Welcome to SimpleAddon!")
end)
```

---

### Example 2: Medium Addon with Modules

**Purpose:** Track player gold across characters

```
GoldTracker/
    GoldTracker.toc
    Core.lua
    Database.lua
    UI.lua
    Locales.lua
```

**GoldTracker.toc:**
```
## Interface: 120000
## Title: Gold Tracker
## Author: YourName
## Version: 1.0.0
## SavedVariablesPerCharacter: GoldTrackerDB
## SavedVariables: GoldTrackerGlobal

Core.lua
Database.lua
Locales.lua
UI.lua
```

**Core.lua:**
```lua
local ADDON_NAME, ns = ...

GoldTracker = {}
ns.GoldTracker = GoldTracker

local function OnAddonLoaded()
    GoldTracker.db = GoldTrackerDB or {}
    GoldTracker.global = GoldTrackerGlobal or {}
end

local function OnPlayerMoney()
    local money = GetMoney()
    local playerName = UnitName("player")
    local realmName = GetRealmName()

    GoldTracker.global[realmName] = GoldTracker.global[realmName] or {}
    GoldTracker.global[realmName][playerName] = money
end

local frame = CreateFrame("Frame")
frame:RegisterEvent("ADDON_LOADED")
frame:RegisterEvent("PLAYER_MONEY")
frame:SetScript("OnEvent", function(self, event, ...)
    if event == "ADDON_LOADED" and select(1, ...) == ADDON_NAME then
        OnAddonLoaded()
    elseif event == "PLAYER_MONEY" then
        OnPlayerMoney()
    end
end)
```

---

### Example 3: Complex Addon with Ace3

**Purpose:** Full-featured raid frame addon

```
RaidFrames/
    RaidFrames.toc

    # Libraries
    Libs\
        LibStub\LibStub.lua
        CallbackHandler-1.0\CallbackHandler-1.0.lua
        AceAddon-3.0\AceAddon-3.0.lua
        AceEvent-3.0\AceEvent-3.0.lua
        AceDB-3.0\AceDB-3.0.lua
        AceConfig-3.0\AceConfig-3.0.lua
        AceGUI-3.0\AceGUI-3.0.lua

    # Core
    Core.lua
    Defaults.lua

    # Localization
    Locales\
        Locale.xml
        enUS.lua
        deDE.lua

    # Modules
    Modules\
        UnitFrames.lua
        RaidFrames.lua
        PartyFrames.lua

    # UI
    UI\
        Templates.xml
        Frames.lua
        Frames.xml

    # Config
    Config\
        Options.lua
```

**RaidFrames.toc:**
```
## Interface: 120000
## Title: Raid Frames
## Author: YourName
## Version: 2.0.0
## Notes: Advanced raid frame addon
## SavedVariables: RaidFramesDB
## OptionalDeps: LibSharedMedia-3.0

# Libraries
Libs\LibStub\LibStub.lua
Libs\CallbackHandler-1.0\CallbackHandler-1.0.lua
Libs\AceAddon-3.0\AceAddon-3.0.lua
Libs\AceEvent-3.0\AceEvent-3.0.lua
Libs\AceDB-3.0\AceDB-3.0.lua
Libs\AceConfig-3.0\AceConfig-3.0.lua
Libs\AceGUI-3.0\AceGUI-3.0.lua

# Core
Defaults.lua
Core.lua

# Localization
Locales\Locale.xml

# Modules
Modules\UnitFrames.lua
Modules\RaidFrames.lua
Modules\PartyFrames.lua

# UI
UI\Templates.xml
UI\Frames.lua
UI\Frames.xml

# Config
Config\Options.lua
```

**Core.lua:**
```lua
local AceAddon = LibStub("AceAddon-3.0")
local RaidFrames = AceAddon:NewAddon("RaidFrames", "AceEvent-3.0")

function RaidFrames:OnInitialize()
    self.db = LibStub("AceDB-3.0"):New("RaidFramesDB", self.defaults, true)

    -- Register options
    local AceConfig = LibStub("AceConfig-3.0")
    AceConfig:RegisterOptionsTable("RaidFrames", self.options)

    local AceConfigDialog = LibStub("AceConfigDialog-3.0")
    self.optionsFrame = AceConfigDialog:AddToBlizOptions("RaidFrames", "RaidFrames")
end

function RaidFrames:OnEnable()
    self:RegisterEvent("PLAYER_ENTERING_WORLD")
    self:RegisterEvent("GROUP_ROSTER_UPDATE")
    self:CreateFrames()
end

function RaidFrames:PLAYER_ENTERING_WORLD()
    self:UpdateFrames()
end

function RaidFrames:GROUP_ROSTER_UPDATE()
    self:UpdateRoster()
end
```

---

### Example 4: Multi-Version Addon (Retail + Classic)

**Purpose:** Support multiple WoW versions

#### Option A: Single TOC with Comma-Separated Versions (Recommended)

**Use this when:** Your addon code is the same across all versions.

```
UniversalAddon/
    UniversalAddon.toc
    Core.lua
    Utils.lua
```

**UniversalAddon.toc:**
```
## Interface: 120000, 110207, 50503, 40402, 11508
## Title: Universal Addon
## Author: YourName
## Version: 1.0.0
## Notes: Works on Retail, Cata Classic, and Classic Era

Core.lua
Utils.lua
```

**Core.lua:**
```lua
local isMainline = (WOW_PROJECT_ID == WOW_PROJECT_MAINLINE)
local isClassic = (WOW_PROJECT_ID == WOW_PROJECT_CLASSIC)

UniversalAddon = {
    version = "1.0.0",
    isMainline = isMainline,
    isClassic = isClassic
}

-- Shared functionality with version detection for API differences
function UniversalAddon:Initialize()
    print("UniversalAddon loaded for " .. (isMainline and "Mainline" or "Classic"))

    -- Handle API differences in code
    if isMainline then
        -- Use C_ActionBar namespace (12.0.0+)
        self.GetActionInfo = C_ActionBar and C_ActionBar.GetActionInfo or GetActionInfo
    else
        -- Use legacy globals on Classic
        self.GetActionInfo = GetActionInfo
    end
end
```

#### Option B: Multiple TOC Files (When Needed)

**Use this when:** You need to load different files for different game versions.

```
MultiVersion/
    MultiVersion_Mainline.toc    # Retail
    MultiVersion_Cata.toc        # Cata Classic
    MultiVersion_Vanilla.toc     # Classic Era

    # Shared core
    Core.lua

    # Version-specific files
    Mainline\
        Features.lua
    Classic\
        Features.lua
```

**MultiVersion_Mainline.toc:**
```
## Interface: 120000
## Title: MultiVersion
## Author: YourName
## Version: 1.0.0

Core.lua
Mainline\Features.lua
```

**MultiVersion_Vanilla.toc:**
```
## Interface: 11508
## Title: MultiVersion
## Author: YourName
## Version: 1.0.0

Core.lua
Classic\Features.lua
```

**When Multiple TOCs Are Necessary:**
- Loading different Lua/XML files per version
- Using version-specific libraries
- Significant code path differences that can't be handled with runtime detection

---

<!-- CLAUDE_SKIP_START -->
## Best Practices Summary

### TOC File
- Always specify `## Interface` matching your target version
- Use descriptive `## Title` and `## Notes`
- List files in logical load order
- Comment your TOC file for clarity

### Dependencies
- Embed libraries when possible (reduces user hassle)
- Use `OptionalDeps` for soft dependencies
- Avoid circular dependencies
- Document why dependencies exist

### File Organization
- Group related files in folders
- Load Lua before XML
- Load core before modules
- Load data before code that uses it

### Initialization
- Use proper initialization events (ADDON_LOADED, PLAYER_LOGIN)
- Don't do heavy work at load time
- Initialize saved variables safely
- Handle missing/corrupt saved variables

### Modules
- Keep modules independent when possible
- Provide clear module APIs
- Use registration systems for discoverability
- Document module dependencies

### Localization
- Always provide enUS as default
- Load locale files after core
- Use locale keys, not hardcoded strings
- Test with different locales

### Performance
- Use LoadOnDemand for optional features
- Lazy-load heavy modules
- Cache frequently-used data
- Unregister events when not needed

---

## References

### Official Resources
- FrameXML Source: `Interface\FrameXML`
- Blizzard Addons: `Interface\AddOns\Blizzard_*`
- API Documentation: https://wowpedia.fandom.com/

### Community Standards
- WoW Interface: https://www.wowinterface.com/
- CurseForge: https://www.curseforge.com/wow/addons
- Wago: https://addons.wago.io/

---

**Document Version:** 2.0.0
**Game Version:** 12.0.0 (Midnight)
**Last Updated:** 2026-01-20

<!-- CLAUDE_SKIP_END -->
