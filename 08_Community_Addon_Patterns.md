# Community Addon Development Patterns

## Table of Contents
1. [Popular Addon Frameworks](#popular-addon-frameworks)
2. [Common Addon Structures](#common-addon-structures)
3. [Configuration and Options](#configuration-and-options)
4. [Slash Commands](#slash-commands)
5. [LibStub and Library Management](#libstub-and-library-management)
6. [Localization Patterns](#localization-patterns)
7. [Profile Systems](#profile-systems)
8. [Update and Notification Systems](#update-and-notification-systems)
9. [Performance Best Practices](#performance-best-practices)
10. [Distribution and Packaging](#distribution-and-packaging)
11. [Addon Categories and 12.0 Compatibility](#addon-categories-and-120-compatibility)
12. [Library Updates for 12.0](#library-updates-for-120)

---

## Popular Addon Frameworks

### Ace3 Framework

**Most popular addon development framework in the WoW community.**

**Core Libraries:**
- **AceAddon-3.0** - Addon management
- **AceConsole-3.0** - Slash commands
- **AceConfig-3.0** - Configuration GUI
- **AceDB-3.0** - Database management with profiles
- **AceEvent-3.0** - Event handling
- **AceHook-3.0** - Function hooking
- **AceGUI-3.0** - GUI widgets
- **AceLocale-3.0** - Localization

**Basic Ace3 Addon Structure:**
```lua
-- MyAddon.lua
local AddonName = "MyAddon";
local MyAddon = LibStub("AceAddon-3.0"):NewAddon(AddonName, "AceConsole-3.0", "AceEvent-3.0");

-- Defaults
local defaults = {
    profile = {
        enabled = true,
        message = "Hello, World!",
    },
};

function MyAddon:OnInitialize()
    -- Database
    self.db = LibStub("AceDB-3.0"):New("MyAddonDB", defaults, true);

    -- Register slash command
    self:RegisterChatCommand("myaddon", "ChatCommand");
end

function MyAddon:OnEnable()
    -- Register events
    self:RegisterEvent("PLAYER_LOGIN");
end

function MyAddon:OnDisable()
    -- Cleanup
end

function MyAddon:PLAYER_LOGIN()
    self:Print("MyAddon loaded!");
end

function MyAddon:ChatCommand(input)
    if not input or input:trim() == "" then
        self:Print("Usage: /myaddon <command>");
    else
        self:Print("You said:", input);
    end
end
```

---

## Common Addon Structures

### Module-Based Architecture

**Pattern used by large addons like WeakAuras, DBM, BigWigs:**

```
MyAddon/
├── MyAddon.toc
├── Core.lua              # Main addon initialization
├── Config.lua            # Configuration/defaults
├── Options.lua           # Options GUI
├── Locales/
│   ├── enUS.lua
│   ├── deDE.lua
│   └── frFR.lua
├── Modules/
│   ├── Module1.lua       # Feature module 1
│   ├── Module2.lua       # Feature module 2
│   └── Module3.lua       # Feature module 3
├── UI/
│   ├── MainFrame.xml
│   ├── MainFrame.lua
│   └── Templates.xml
└── Libs/
    ├── LibStub/
    ├── AceAddon-3.0/
    └── AceDB-3.0/
```

**Module Pattern:**
```lua
-- Core.lua
MyAddon = LibStub("AceAddon-3.0"):NewAddon("MyAddon");
MyAddon.modules = {};

function MyAddon:RegisterModule(name, module)
    self.modules[name] = module;
    module.parent = self;
end

-- Module1.lua
local Module1 = {};

function Module1:OnEnable()
    print("Module1 enabled");
end

function Module1:OnDisable()
    print("Module1 disabled");
end

MyAddon:RegisterModule("Module1", Module1);
```

### Single-File Addon

**Simple addons can be just one file:**

```
MySimpleAddon/
├── MySimpleAddon.toc
└── MySimpleAddon.lua
```

**MySimpleAddon.toc:**
```
## Interface: 120000
## Title: My Simple Addon
## Author: Your Name
## Version: 1.0.0
## SavedVariables: MySimpleAddonDB

MySimpleAddon.lua
```

**MySimpleAddon.lua:**
```lua
local AddonName = "MySimpleAddon";
local DB;

-- Initialize
local frame = CreateFrame("Frame");
frame:RegisterEvent("ADDON_LOADED");
frame:RegisterEvent("PLAYER_LOGIN");

frame:SetScript("OnEvent", function(self, event, ...)
    if event == "ADDON_LOADED" then
        local addonName = ...;
        if addonName == AddonName then
            DB = MySimpleAddonDB or {};
            MySimpleAddonDB = DB;
        end
    elseif event == "PLAYER_LOGIN" then
        print(AddonName, "loaded!");
    end
end);
```

---

## Configuration and Options

### Ace3Config Pattern

**Most addons use Ace3 for configuration:**

```lua
local options = {
    name = "MyAddon",
    handler = MyAddon,
    type = "group",
    args = {
        enabled = {
            type = "toggle",
            name = "Enable",
            desc = "Enable/disable the addon",
            get = function(info) return MyAddon.db.profile.enabled; end,
            set = function(info, value)
                MyAddon.db.profile.enabled = value;
            end,
        },
        message = {
            type = "input",
            name = "Message",
            desc = "Custom message to display",
            get = function(info) return MyAddon.db.profile.message; end,
            set = function(info, value)
                MyAddon.db.profile.message = value;
            end,
        },
        color = {
            type = "color",
            name = "Color",
            desc = "Pick a color",
            hasAlpha = true,
            get = function(info)
                local c = MyAddon.db.profile.color;
                return c.r, c.g, c.b, c.a;
            end,
            set = function(info, r, g, b, a)
                local c = MyAddon.db.profile.color;
                c.r, c.g, c.b, c.a = r, g, b, a;
            end,
        },
    },
};

-- Register options
LibStub("AceConfig-3.0"):RegisterOptionsTable("MyAddon", options);
LibStub("AceConfigDialog-3.0"):AddToBlizOptions("MyAddon", "MyAddon");
```

### Blizzard Settings Panel (11.0+)

**Modern WoW settings integration:**

```lua
local category = Settings.RegisterVerticalLayoutCategory("MyAddon");

-- Boolean setting
local variable = "MyAddon_Enabled";
local name = "Enable MyAddon";
local tooltip = "Enable or disable the addon";
local defaultValue = true;

local setting = Settings.RegisterAddOnSetting(category, variable, variable, Settings.VarType.Boolean, name, defaultValue);
Settings.CreateCheckbox(category, setting, tooltip);

-- Dropdown setting
local function GetOptions()
    local container = Settings.CreateControlTextContainer();
    container:Add("option1", "Option 1");
    container:Add("option2", "Option 2");
    container:Add("option3", "Option 3");
    return container:GetData();
end

local variable = "MyAddon_DropdownValue";
local name = "Dropdown Setting";
local tooltip = "Select an option";
local defaultValue = "option1";

local setting = Settings.RegisterAddOnSetting(category, variable, variable, Settings.VarType.String, name, defaultValue);
Settings.CreateDropdown(category, setting, GetOptions, tooltip);

-- Register category
Settings.RegisterAddOnCategory(category);
```

---

## Slash Commands

### Basic Slash Command

```lua
SLASH_MYADDON1 = "/myaddon";
SLASH_MYADDON2 = "/ma";

SlashCmdList["MYADDON"] = function(msg, editBox)
    local command, rest = msg:match("^(%S*)%s*(.-)$");

    if command == "show" then
        MyAddon:ShowFrame();
    elseif command == "hide" then
        MyAddon:HideFrame();
    elseif command == "config" then
        MyAddon:OpenConfig();
    else
        print("MyAddon commands:");
        print("  /myaddon show - Show main frame");
        print("  /myaddon hide - Hide main frame");
        print("  /myaddon config - Open configuration");
    end
end;
```

### Advanced Command Parsing

```lua
local commands = {
    ["show"] = function()
        MyAddon:ShowFrame();
    end,
    ["hide"] = function()
        MyAddon:HideFrame();
    end,
    ["set"] = function(rest)
        local key, value = rest:match("^(%S+)%s+(.+)$");
        if key and value then
            MyAddon.db.profile[key] = value;
            print("Set", key, "to", value);
        else
            print("Usage: /myaddon set <key> <value>");
        end
    end,
    ["get"] = function(rest)
        local key = rest:match("^(%S+)");
        if key then
            print(key, "=", tostring(MyAddon.db.profile[key]));
        else
            print("Usage: /myaddon get <key>");
        end
    end,
};

SlashCmdList["MYADDON"] = function(msg)
    local command, rest = msg:match("^(%S*)%s*(.-)$");
    command = command:lower();

    local handler = commands[command];
    if handler then
        handler(rest);
    else
        print("Unknown command:", command);
        print("Available commands:");
        for cmd in pairs(commands) do
            print("  /myaddon", cmd);
        end
    end
end;
```

---

## LibStub and Library Management

### LibStub Pattern

**Standard library loader:**

```lua
-- Check if library already loaded
local lib = LibStub:GetLibrary("MyLib-1.0", true);
if lib then
    return lib;  -- Already loaded
end

-- Create new library
local lib = LibStub:NewLibrary("MyLib-1.0", 1);  -- Name, version
if not lib then
    return;  -- Older version already loaded
end

-- Library implementation
function lib:DoSomething()
    print("Library function called!");
end
```

### Embedding Libraries

**Pattern used by Ace3 libraries:**

```lua
-- MyLib.lua
local MAJOR, MINOR = "MyLib-1.0", 1;
local lib = LibStub:NewLibrary(MAJOR, MINOR);
if not lib then return; end

-- Embed mixin
lib.embeds = lib.embeds or {};

function lib:Embed(target)
    for k, v in pairs(self) do
        if type(v) == "function" and k ~= "Embed" then
            target[k] = v;
        end
    end
    self.embeds[target] = true;
end

-- Usage
MyAddon = {};
LibStub("MyLib-1.0"):Embed(MyAddon);
MyAddon:DoSomething();  -- Library method now available
```

---

## Localization Patterns

### Basic Localization

```lua
-- Locales/enUS.lua
local L = LibStub("AceLocale-3.0"):NewLocale("MyAddon", "enUS", true);
if not L then return; end

L["WELCOME_MESSAGE"] = "Welcome to MyAddon!";
L["OPTION_ENABLED"] = "Enabled";
L["OPTION_DISABLED"] = "Disabled";

-- Locales/deDE.lua
local L = LibStub("AceLocale-3.0"):NewLocale("MyAddon", "deDE");
if not L then return; end

L["WELCOME_MESSAGE"] = "Willkommen bei MyAddon!";
L["OPTION_ENABLED"] = "Aktiviert";
L["OPTION_DISABLED"] = "Deaktiviert";

-- Usage in code
local L = LibStub("AceLocale-3.0"):GetLocale("MyAddon");
print(L["WELCOME_MESSAGE"]);
```

### Simple Localization (No Library)

```lua
local L = {};

if GetLocale() == "deDE" then
    L["WELCOME"] = "Willkommen!";
elseif GetLocale() == "frFR" then
    L["WELCOME"] = "Bienvenue!";
else
    L["WELCOME"] = "Welcome!";
end

print(L["WELCOME"]);
```

---

## Profile Systems

### Ace3 Profile Pattern

```lua
-- Database with profiles
local defaults = {
    profile = {
        setting1 = true,
        setting2 = "value",
    },
};

MyAddon.db = LibStub("AceDB-3.0"):New("MyAddonDB", defaults, true);

-- Profile options
local profileOptions = LibStub("AceDBOptions-3.0"):GetOptionsTable(MyAddon.db);

-- Add to config
options.args.profiles = profileOptions;

-- Access current profile
local profile = MyAddon.db.profile;
print(profile.setting1);

-- Change profile
MyAddon.db:SetProfile("ProfileName");

-- Create new profile
MyAddon.db:SetProfile("NewProfile");

-- Delete profile
MyAddon.db:DeleteProfile("ProfileName");

-- Copy profile
MyAddon.db:CopyProfile("SourceProfile");

-- Reset profile
MyAddon.db:ResetProfile();
```

---

## Update and Notification Systems

### Version Check Pattern

```lua
local CURRENT_VERSION = "1.2.3";

-- Parse version string
local function ParseVersion(versionString)
    local major, minor, patch = versionString:match("(%d+)%.(%d+)%.(%d+)");
    return tonumber(major), tonumber(minor), tonumber(patch);
end

-- Compare versions
local function IsNewerVersion(v1, v2)
    local major1, minor1, patch1 = ParseVersion(v1);
    local major2, minor2, patch2 = ParseVersion(v2);

    if major1 > major2 then return true; end
    if major1 < major2 then return false; end
    if minor1 > minor2 then return true; end
    if minor1 < minor2 then return false; end
    return patch1 > patch2;
end

-- Check for updates
function MyAddon:CheckVersion()
    if not MyAddonDB.lastVersion then
        -- First install
        MyAddonDB.lastVersion = CURRENT_VERSION;
        self:ShowWelcomeMessage();
    elseif IsNewerVersion(CURRENT_VERSION, MyAddonDB.lastVersion) then
        -- Updated
        self:ShowUpdateMessage(MyAddonDB.lastVersion, CURRENT_VERSION);
        MyAddonDB.lastVersion = CURRENT_VERSION;
    end
end
```

### Changelog Notification

```lua
local CHANGELOG = {
    ["1.2.3"] = {
        "Added new feature X",
        "Fixed bug Y",
        "Improved performance",
    },
    ["1.2.2"] = {
        "Fixed critical bug",
    },
};

function MyAddon:ShowChangelog(fromVersion, toVersion)
    print(format("|cff00ff00%s updated from %s to %s|r", AddonName, fromVersion, toVersion));

    for version, changes in pairs(CHANGELOG) do
        if IsNewerVersion(version, fromVersion) and not IsNewerVersion(version, toVersion) then
            print(format("|cffFFFF00Version %s:|r", version));
            for _, change in ipairs(changes) do
                print(format("  - %s", change));
            end
        end
    end
end
```

---

## Performance Best Practices

### Throttling Updates

```lua
-- Throttle function calls
local function CreateThrottle(delay)
    local lastUpdate = 0;
    return function(callback)
        local now = GetTime();
        if now - lastUpdate >= delay then
            lastUpdate = now;
            callback();
        end
    end;
end

-- Usage
local throttledUpdate = CreateThrottle(0.5);  -- Max once per 0.5 seconds

frame:SetScript("OnUpdate", function()
    throttledUpdate(function()
        -- Expensive update code
        MyAddon:RefreshDisplay();
    end);
end);
```

### Debouncing

```lua
-- Debounce: Only call after activity stops
local function CreateDebounce(delay)
    local timer;
    return function(callback)
        if timer then
            timer:Cancel();
        end
        timer = C_Timer.NewTimer(delay, callback);
    end;
end

-- Usage
local debouncedSave = CreateDebounce(1.0);  -- Wait 1 second after last change

function MyAddon:OnSettingChanged()
    debouncedSave(function()
        MyAddon:SaveSettings();
    end);
end
```

### Using New Profiler Metrics (12.0+)

```lua
-- 12.0 introduces C_ScriptProfiler for addon performance analysis
-- Use this to identify bottlenecks

local function ProfileFunction(funcName, func)
    return function(...)
        local startTime = debugprofilestop();
        local results = {func(...)};
        local elapsed = debugprofilestop() - startTime;

        if elapsed > 1 then  -- Log slow calls (>1ms)
            print(format("%s took %.2fms", funcName, elapsed));
        end

        return unpack(results);
    end;
end

-- Wrap expensive functions
MyAddon.ExpensiveFunction = ProfileFunction("ExpensiveFunction", MyAddon.ExpensiveFunction);
```

---

## Distribution and Packaging

### TOC File Best Practices

```
## Interface: 120000
## Title: My Addon
## Notes: Short description of what the addon does
## Author: Your Name
## Version: @project-version@  # Auto-filled by packager
## X-Category: Interface Enhancements
## X-Website: https://github.com/yourname/myaddon
## X-Curse-Project-ID: 12345
## X-Wago-ID: yourwagoid

## SavedVariables: MyAddonDB
## SavedVariablesPerCharacter: MyAddonCharDB

# Libraries
Libs\LibStub\LibStub.lua
Libs\AceAddon-3.0\AceAddon-3.0.lua

# Localization
Locales\enUS.lua
Locales\deDE.lua

# Core
Core.lua
Config.lua

# Modules
Modules\Module1.lua
Modules\Module2.lua

# UI
UI\Templates.xml
UI\MainFrame.lua
UI\MainFrame.xml
```

### .pkgmeta for CurseForge/Wago

```yaml
package-as: MyAddon

enable-nolib-creation: no

externals:
    Libs/LibStub:
        url: https://repos.curseforge.com/wow/libstub/trunk
    Libs/AceAddon-3.0:
        url: https://repos.curseforge.com/wow/ace3/trunk/AceAddon-3.0
    Libs/AceDB-3.0:
        url: https://repos.curseforge.com/wow/ace3/trunk/AceDB-3.0

ignore:
    - README.md
    - .git
    - .github
```

---

## Addon Categories and 12.0 Compatibility

The 12.0.0 (Midnight) expansion introduces significant API changes that affect most addon categories. This section documents the impact and migration strategies for each major category.

### Damage Meters (Details, Skada, Recount)

**12.0 Changes:**
- **Official C_DamageMeter API** - Blizzard now provides native damage meter functionality
- **C_CombatLog namespace required** - Direct combat log parsing still works but must use namespaced API
- **Secret values system** - Real-time combat data may show approximated values during encounters

**Migration Strategy:**
```lua
-- Check for official damage meter API
local hasOfficialMeter = C_DamageMeter ~= nil;

-- Modern combat log event registration (12.0+)
local function RegisterCombatLogEvents()
    -- Use C_CombatLog namespace for parsing
    if C_CombatLog and C_CombatLog.GetCurrentEventInfo then
        -- Modern approach
        frame:RegisterEvent("COMBAT_LOG_EVENT_UNFILTERED");
    end
end

-- Handle combat log events with secret value awareness
local function OnCombatLogEvent()
    local timestamp, subevent, hideCaster, sourceGUID, sourceName,
          sourceFlags, sourceRaidFlags, destGUID, destName, destFlags,
          destRaidFlags = CombatLogGetCurrentEventInfo();

    -- Note: Damage values may be approximated (secrets) in 12.0
    -- Consider integrating with C_DamageMeter for accurate totals
    if subevent == "SWING_DAMAGE" then
        local amount, overkill, school, resisted, blocked, absorbed,
              critical, glancing, crushing, isOffHand = select(12, CombatLogGetCurrentEventInfo());

        -- amount may be a secret value - handle gracefully
        if type(amount) == "number" then
            ProcessDamage(sourceGUID, destGUID, amount);
        end
    end
end

-- Optional: Integrate with official meter
local function GetOfficialMeterData()
    if C_DamageMeter and C_DamageMeter.GetEncounterData then
        return C_DamageMeter.GetEncounterData();
    end
    return nil;
end
```

### Boss Mods (DBM, BigWigs, LittleWigs)

**12.0 Changes:**
- **C_EncounterTimeline API** - Official boss timeline system with phase tracking
- **C_EncounterWarnings API** - Native boss warnings that addons can supplement
- **Secret values** - Unit health in boss encounters may show approximated values

**Migration Strategy:**
```lua
-- Check for official encounter systems
local hasEncounterTimeline = C_EncounterTimeline ~= nil;
local hasEncounterWarnings = C_EncounterWarnings ~= nil;

-- Modern boss mod initialization
function BossMod:OnEncounterStart(encounterID, encounterName, difficultyID, groupSize)
    -- Check if official timeline is available
    if hasEncounterTimeline then
        local phases = C_EncounterTimeline.GetEncounterPhases(encounterID);
        if phases then
            self:IntegrateOfficialPhases(phases);
        end
    end

    -- Register for official warnings to avoid duplicates
    if hasEncounterWarnings then
        self:RegisterCallback(C_EncounterWarnings, "OnWarningDisplayed", "HandleOfficialWarning");
    end
end

-- Handle secret values for boss health
function BossMod:GetBossHealth(unit)
    local health = UnitHealth(unit);
    local maxHealth = UnitHealthMax(unit);

    -- In 12.0, use UnitHealthPercent for more reliable data
    local percent = UnitHealthPercent and UnitHealthPercent(unit);
    if percent then
        return percent;  -- Returns 0-100, handles secrets gracefully
    end

    -- Fallback for older API
    if maxHealth > 0 then
        return (health / maxHealth) * 100;
    end
    return 0;
end

-- Supplement rather than replace official warnings
function BossMod:DisplayWarning(warningType, spellID, text)
    -- Check if official warning system will handle this
    if hasEncounterWarnings and C_EncounterWarnings.HasWarningForSpell(spellID) then
        -- Official system handles it - optionally add supplemental info
        return;
    end

    -- Display custom warning
    self:ShowWarningFrame(warningType, text);
end
```

### Action Bar Addons (Bartender, ElvUI, Dominos)

**12.0 Changes - BREAKING:**
- **Global action bar functions REMOVED** - Must use C_ActionBar namespace exclusively
- **Major rewrite required** for any addon using legacy action bar APIs
- **New profiler metrics** available for optimization

**Migration Strategy:**
```lua
-- 12.0 REQUIRED: All action bar functions must use C_ActionBar
-- Legacy functions like GetActionInfo(), PickupAction() are REMOVED

-- Modern action bar slot management
local function GetActionSlotInfo(slot)
    -- 12.0: Must use C_ActionBar namespace
    if C_ActionBar.GetActionInfo then
        local actionType, id, subType = C_ActionBar.GetActionInfo(slot);
        return actionType, id, subType;
    end
    return nil;
end

-- Modern action pickup
local function PickupActionSlot(slot)
    if C_ActionBar.PickupAction then
        C_ActionBar.PickupAction(slot);
    end
end

-- Modern action placement
local function PlaceActionInSlot(slot)
    if C_ActionBar.PlaceAction then
        C_ActionBar.PlaceAction(slot);
    end
end

-- Modern action bar page management
local function GetCurrentActionBarPage()
    if C_ActionBar.GetCurrentPage then
        return C_ActionBar.GetCurrentPage();
    end
    return 1;
end

-- Check if action has range
local function IsActionInRange(slot, unit)
    if C_ActionBar.IsActionInRange then
        return C_ActionBar.IsActionInRange(slot, unit);
    end
    return nil;
end

-- Get action cooldown
local function GetActionCooldownInfo(slot)
    if C_ActionBar.GetActionCooldown then
        return C_ActionBar.GetActionCooldown(slot);
    end
    return 0, 0, 0;
end

-- Full compatibility wrapper for action bar addons
local ActionBarCompat = {};

function ActionBarCompat:Initialize()
    -- Verify required APIs exist
    assert(C_ActionBar, "C_ActionBar namespace required for 12.0+");
    assert(C_ActionBar.GetActionInfo, "C_ActionBar.GetActionInfo required");
    assert(C_ActionBar.HasAction, "C_ActionBar.HasAction required");
end

function ActionBarCompat:HasAction(slot)
    return C_ActionBar.HasAction(slot);
end

function ActionBarCompat:GetActionTexture(slot)
    return C_ActionBar.GetActionTexture(slot);
end

function ActionBarCompat:GetActionText(slot)
    return C_ActionBar.GetActionText(slot);
end

function ActionBarCompat:GetActionCharges(slot)
    return C_ActionBar.GetActionCharges(slot);
end

function ActionBarCompat:IsUsableAction(slot)
    local usable, noMana, noRange = C_ActionBar.IsUsableAction(slot);
    return usable, noMana;
end
```

### Unit Frame Addons (ElvUI, Shadowed Unit Frames, Pitbull)

**12.0 Changes:**
- **Secret values system** - Health and power values may be approximated during combat
- **New Unit functions** - UnitHealthPercent(), UnitPowerPercent() provide reliable percentages
- **Health prediction changes** - Must handle secrets in prediction calculations

**Migration Strategy:**
```lua
-- Modern unit health display with secret handling
local function GetUnitHealthDisplay(unit)
    -- 12.0: Use percentage functions for reliable display
    local percent = UnitHealthPercent and UnitHealthPercent(unit);

    if percent then
        -- Reliable percentage even with secrets
        local maxHealth = UnitHealthMax(unit);
        local displayHealth = math.floor(maxHealth * percent / 100);
        return displayHealth, maxHealth, percent;
    end

    -- Fallback
    local health = UnitHealth(unit);
    local maxHealth = UnitHealthMax(unit);
    local calcPercent = maxHealth > 0 and (health / maxHealth * 100) or 0;
    return health, maxHealth, calcPercent;
end

-- Modern power display
local function GetUnitPowerDisplay(unit, powerType)
    local percent = UnitPowerPercent and UnitPowerPercent(unit, powerType);

    if percent then
        local maxPower = UnitPowerMax(unit, powerType);
        local displayPower = math.floor(maxPower * percent / 100);
        return displayPower, maxPower, percent;
    end

    local power = UnitPower(unit, powerType);
    local maxPower = UnitPowerMax(unit, powerType);
    local calcPercent = maxPower > 0 and (power / maxPower * 100) or 0;
    return power, maxPower, calcPercent;
end

-- Health bar update with secrets awareness
function UnitFrameMixin:UpdateHealth()
    local displayHealth, maxHealth, percent = GetUnitHealthDisplay(self.unit);

    self.healthBar:SetMinMaxValues(0, maxHealth);
    self.healthBar:SetValue(displayHealth);

    -- Display percentage for accuracy during secrets
    if self.healthText then
        if self.showPercentage then
            self.healthText:SetText(format("%.0f%%", percent));
        else
            self.healthText:SetText(AbbreviateNumber(displayHealth));
        end
    end
end

-- Incoming heal prediction with secrets handling
function UnitFrameMixin:UpdateHealPrediction()
    local _, maxHealth, percent = GetUnitHealthDisplay(self.unit);
    local incomingHeal = UnitGetIncomingHeals(self.unit) or 0;
    local absorb = UnitGetTotalAbsorbs(self.unit) or 0;

    -- Calculate based on percentages for consistency
    local currentPercent = percent / 100;
    local healPercent = maxHealth > 0 and (incomingHeal / maxHealth) or 0;
    local absorbPercent = maxHealth > 0 and (absorb / maxHealth) or 0;

    self:SetHealPredictionBars(currentPercent, healPercent, absorbPercent);
end
```

### Transmog Addons (Narcissus, BetterWardrobe, Mog It)

**12.0 Changes - MAJOR OVERHAUL:**
- **C_TransmogOutfitInfo namespace** - Complete new API for outfit management
- **Old outfit APIs removed** - C_TransmogSets heavily restructured
- **Custom sets system** - New player-created outfit functionality

**Migration Strategy:**
```lua
-- Check for 12.0 transmog API
local hasNewTransmogAPI = C_TransmogOutfitInfo ~= nil;

-- Modern outfit management
local TransmogCompat = {};

function TransmogCompat:GetOutfits()
    if hasNewTransmogAPI and C_TransmogOutfitInfo.GetOutfits then
        return C_TransmogOutfitInfo.GetOutfits();
    end
    -- Legacy fallback
    return C_TransmogCollection and C_TransmogCollection.GetOutfits() or {};
end

function TransmogCompat:GetOutfitInfo(outfitID)
    if hasNewTransmogAPI and C_TransmogOutfitInfo.GetOutfitInfo then
        return C_TransmogOutfitInfo.GetOutfitInfo(outfitID);
    end
    return nil;
end

function TransmogCompat:SaveOutfit(name, sources)
    if hasNewTransmogAPI and C_TransmogOutfitInfo.SaveOutfit then
        return C_TransmogOutfitInfo.SaveOutfit(name, sources);
    end
    -- Legacy
    if C_TransmogCollection and C_TransmogCollection.SaveOutfit then
        return C_TransmogCollection.SaveOutfit(name, sources);
    end
    return false;
end

function TransmogCompat:DeleteOutfit(outfitID)
    if hasNewTransmogAPI and C_TransmogOutfitInfo.DeleteOutfit then
        return C_TransmogOutfitInfo.DeleteOutfit(outfitID);
    end
    if C_TransmogCollection and C_TransmogCollection.DeleteOutfit then
        return C_TransmogCollection.DeleteOutfit(outfitID);
    end
end

function TransmogCompat:GetSlotSources(slotID)
    if hasNewTransmogAPI and C_TransmogOutfitInfo.GetSlotSources then
        return C_TransmogOutfitInfo.GetSlotSources(slotID);
    end
    if C_TransmogCollection then
        return C_TransmogCollection.GetAppearanceSources(slotID);
    end
    return {};
end

-- Apply outfit
function TransmogCompat:ApplyOutfit(outfitID)
    if hasNewTransmogAPI and C_TransmogOutfitInfo.ApplyOutfit then
        C_TransmogOutfitInfo.ApplyOutfit(outfitID);
        return true;
    end
    return false;
end
```

### Bag/Inventory Addons (Bagnon, AdiBags, ArkInventory)

**12.0 Changes:**
- **Void Storage REMOVED** (11.2.0) - Must remove or gracefully handle missing API
- **Socket APIs moved** to C_ItemSocketInfo namespace (11.2.5+)
- **Bank system restructured** - New Warband bank integration

**Migration Strategy:**
```lua
-- Check for removed/moved APIs
local hasVoidStorage = C_VoidStorage ~= nil;
local hasNewSocketAPI = C_ItemSocketInfo ~= nil;

-- Socket API compatibility
local function GetItemSockets(itemLink)
    if hasNewSocketAPI and C_ItemSocketInfo.GetItemSockets then
        return C_ItemSocketInfo.GetItemSockets(itemLink);
    end
    -- Legacy fallback
    if GetItemStats then
        local stats = GetItemStats(itemLink);
        return stats and stats["EMPTY_SOCKET_RED"] or 0;
    end
    return 0;
end

function GetSocketInfo(socketIndex)
    if hasNewSocketAPI and C_ItemSocketInfo.GetSocketInfo then
        return C_ItemSocketInfo.GetSocketInfo(socketIndex);
    end
    -- Legacy
    return GetExistingSocketInfo and GetExistingSocketInfo(socketIndex);
end

-- Void Storage handling - gracefully removed
local function HasVoidStorageAccess()
    if not hasVoidStorage then
        return false;  -- Feature removed in 11.2.0
    end
    return CanUseVoidStorage and CanUseVoidStorage();
end

-- Bank compatibility with Warband bank
local function GetBankSlotInfo(bagIndex, slotIndex)
    -- Check for Warband bank API
    if C_Bank and C_Bank.GetBankSlotInfo then
        return C_Bank.GetBankSlotInfo(bagIndex, slotIndex);
    end
    -- Legacy
    return GetContainerItemInfo(bagIndex, slotIndex);
end

-- Modern container iteration
local function IterateBagSlots(bagID, callback)
    local numSlots = C_Container.GetContainerNumSlots(bagID);
    for slot = 1, numSlots do
        local itemInfo = C_Container.GetContainerItemInfo(bagID, slot);
        callback(slot, itemInfo);
    end
end

-- Inventory addon initialization
function InventoryAddon:OnInitialize()
    -- Register for relevant events
    self:RegisterEvent("BAG_UPDATE");
    self:RegisterEvent("BANKFRAME_OPENED");
    self:RegisterEvent("BANKFRAME_CLOSED");

    -- Remove Void Storage tab if feature is gone
    if not hasVoidStorage then
        self:RemoveVoidStorageTab();
    end

    -- Check for Warband bank
    if C_Bank and C_Bank.HasWarbandBank then
        self:SetupWarbandBankTab();
    end
end
```

### Housing Addons (NEW CATEGORY in 12.0)

**12.0 introduces player housing with the C_Housing system. This enables an entirely new addon category.**

**Reference:** See `12_Housing_System_Guide.md` for complete API documentation.

```lua
-- Housing addon base structure
local HousingAddon = LibStub("AceAddon-3.0"):NewAddon("MyHousingAddon", "AceEvent-3.0");

-- Check for housing availability
local hasHousingSystem = C_Housing ~= nil;

function HousingAddon:OnInitialize()
    if not hasHousingSystem then
        self:Print("Housing system not available - addon disabled");
        return;
    end

    self.db = LibStub("AceDB-3.0"):New("MyHousingAddonDB", self.defaults, true);
end

function HousingAddon:OnEnable()
    if not hasHousingSystem then return; end

    -- Register housing events
    self:RegisterEvent("HOUSING_ENTERED");
    self:RegisterEvent("HOUSING_EXITED");
    self:RegisterEvent("HOUSING_DECOR_PLACED");
    self:RegisterEvent("HOUSING_DECOR_REMOVED");
    self:RegisterEvent("HOUSING_DECOR_MOVED");
    self:RegisterEvent("HOUSING_LAYOUT_CHANGED");
end

-- Housing state management
function HousingAddon:HOUSING_ENTERED(event, houseID)
    local houseInfo = C_Housing.GetHouseInfo(houseID);
    self.currentHouse = houseInfo;
    self:RefreshDecorList();
end

function HousingAddon:HOUSING_EXITED(event)
    self.currentHouse = nil;
end

-- Decor management
function HousingAddon:GetPlacedDecor()
    if not self.currentHouse then return {}; end
    return C_Housing.GetPlacedDecor(self.currentHouse.houseID);
end

function HousingAddon:PlaceDecor(decorID, position, rotation)
    if not C_Housing.CanPlaceDecor(decorID) then
        self:Print("Cannot place this decor item");
        return false;
    end

    return C_Housing.PlaceDecor(decorID, position, rotation);
end

-- Layout save/load
function HousingAddon:SaveLayout(layoutName)
    local decor = self:GetPlacedDecor();
    local layout = {
        name = layoutName,
        houseID = self.currentHouse.houseID,
        decor = {},
    };

    for _, item in ipairs(decor) do
        table.insert(layout.decor, {
            decorID = item.decorID,
            position = item.position,
            rotation = item.rotation,
        });
    end

    self.db.profile.layouts[layoutName] = layout;
    self:Print("Layout saved:", layoutName);
end

function HousingAddon:LoadLayout(layoutName)
    local layout = self.db.profile.layouts[layoutName];
    if not layout then
        self:Print("Layout not found:", layoutName);
        return;
    end

    -- Clear current decor first
    C_Housing.ClearAllDecor();

    -- Place saved decor
    for _, item in ipairs(layout.decor) do
        self:PlaceDecor(item.decorID, item.position, item.rotation);
    end

    self:Print("Layout loaded:", layoutName);
end

-- Neighborhood features
function HousingAddon:GetNeighborhood()
    if C_Housing.GetNeighborhoodInfo then
        return C_Housing.GetNeighborhoodInfo();
    end
    return nil;
end

function HousingAddon:VisitNeighbor(playerGUID)
    if C_Housing.CanVisitHouse and C_Housing.CanVisitHouse(playerGUID) then
        C_Housing.VisitHouse(playerGUID);
        return true;
    end
    return false;
end
```

---

## Library Updates for 12.0

Several community libraries may need updates or have native replacements in 12.0.

### Native Replacements for Common Libraries

**C_EncodingUtil (replaces some LibSerialize/LibCompress functionality):**
```lua
-- 12.0 native encoding utilities
if C_EncodingUtil then
    -- Serialize data to string
    local encoded = C_EncodingUtil.EncodeTable(myTable);

    -- Deserialize string to data
    local decoded = C_EncodingUtil.DecodeTable(encoded);

    -- Compress string
    local compressed = C_EncodingUtil.CompressString(largeString);

    -- Decompress string
    local decompressed = C_EncodingUtil.DecompressString(compressed);
else
    -- Fallback to libraries
    local LibSerialize = LibStub("LibSerialize");
    local LibDeflate = LibStub("LibDeflate");
end
```

**Native Table Functions (reduce utility library needs):**
```lua
-- 12.0 native Lua extensions
-- table.create(narr, nrec) - Pre-allocate table memory
local t = table.create(100, 0);  -- 100 array slots, 0 hash slots

-- table.count(t) - Count all entries (both array and hash)
local count = table.count(myTable);

-- These reduce need for custom implementations
local function CreatePooledTable(size)
    return table.create(size, 0);
end

local function GetTableSize(t)
    if table.count then
        return table.count(t);
    end
    -- Fallback
    local count = 0;
    for _ in pairs(t) do count = count + 1; end
    return count;
end
```

### Library Compatibility Checklist

**Libraries that need 12.0 updates:**
- **LibDBIcon** - May need updates for minimap changes
- **LibDataBroker** - Check for compatibility with new UI
- **LibActionButton** - CRITICAL: Must update for C_ActionBar changes
- **LibCompress/LibSerialize** - Consider C_EncodingUtil as alternative
- **LibQTip** - Verify tooltip system compatibility
- **LibRangeCheck** - May need updates for range API changes

**Libraries likely still compatible:**
- **LibStub** - Core loader, should work unchanged
- **Ace3 suite** - Community maintains compatibility
- **CallbackHandler** - Pure Lua, no API dependencies
- **AceLocale** - No API dependencies
- **AceDB** - SavedVariables system unchanged

**Checking Library Compatibility:**
```lua
-- Verify libraries load without error
local function CheckLibraryCompat(libName)
    local success, lib = pcall(function()
        return LibStub(libName);
    end);

    if success and lib then
        print(format("%s loaded successfully", libName));
        return true;
    else
        print(format("%s FAILED to load: %s", libName, tostring(lib)));
        return false;
    end
end

-- Check critical libraries
CheckLibraryCompat("AceAddon-3.0");
CheckLibraryCompat("AceDB-3.0");
CheckLibraryCompat("AceConfig-3.0");
CheckLibraryCompat("LibDBIcon-1.0");
```

### Migration Helper

```lua
-- Helper module for detecting 12.0 API changes
local APICompat = {};

APICompat.features = {
    hasHousing = C_Housing ~= nil,
    hasEncodingUtil = C_EncodingUtil ~= nil,
    hasNewActionBar = C_ActionBar ~= nil and C_ActionBar.GetActionInfo ~= nil,
    hasNewTransmog = C_TransmogOutfitInfo ~= nil,
    hasOfficialDamageMeter = C_DamageMeter ~= nil,
    hasEncounterTimeline = C_EncounterTimeline ~= nil,
    hasEncounterWarnings = C_EncounterWarnings ~= nil,
    hasVoidStorage = C_VoidStorage ~= nil,
    hasNewSocketAPI = C_ItemSocketInfo ~= nil,
    hasTableCreate = table.create ~= nil,
    hasTableCount = table.count ~= nil,
};

function APICompat:PrintCompatReport()
    print("=== 12.0 API Compatibility Report ===");
    for feature, available in pairs(self.features) do
        local status = available and "|cff00ff00YES|r" or "|cffff0000NO|r";
        print(format("  %s: %s", feature, status));
    end
end

function APICompat:RequiresFeature(featureName)
    if not self.features[featureName] then
        error(format("Required feature '%s' not available in this WoW version", featureName));
    end
end

-- Export for other addons
_G.APICompat = APICompat;
```

---

<!-- CLAUDE_SKIP_START -->
## Common Community Patterns Summary

### Do's:
1. Use LibStub for library management
2. Implement profile system for settings
3. Provide localization support
4. Use slash commands for user interaction
5. Version your database for migrations
6. Throttle/debounce expensive operations
7. Follow community naming conventions
8. Provide clear documentation
9. Use semantic versioning (1.2.3)
10. Test with different locales/UI scales

### Don'ts:
1. Don't hardcode English strings
2. Don't embed entire libraries if not needed
3. Don't pollute global namespace
4. Don't break on updates
5. Don't ignore user feedback
6. Don't use deprecated APIs without fallbacks
7. Don't create UI every frame (pool!)
8. Don't save temporary data
9. Don't ignore memory usage
10. Don't ship debug code in release

---

**Version:** 2.0 - Updated for WoW 12.0.0 (Midnight)
**Last Updated:** 2026-01-20
**Community Patterns:** Ace3, LibStub, and common addon frameworks
**Breaking Changes:** Action bar APIs, Transmog APIs, Void Storage removal

<!-- CLAUDE_SKIP_END -->
