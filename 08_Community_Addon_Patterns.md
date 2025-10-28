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
## Interface: 110207
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

---

## Distribution and Packaging

### TOC File Best Practices

```
## Interface: 110207
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

<!-- CLAUDE_SKIP_START -->
## Common Community Patterns Summary

### Do's:
1. ✅ Use LibStub for library management
2. ✅ Implement profile system for settings
3. ✅ Provide localization support
4. ✅ Use slash commands for user interaction
5. ✅ Version your database for migrations
6. ✅ Throttle/debounce expensive operations
7. ✅ Follow community naming conventions
8. ✅ Provide clear documentation
9. ✅ Use semantic versioning (1.2.3)
10. ✅ Test with different locales/UI scales

### Don'ts:
1. ❌ Don't hardcode English strings
2. ❌ Don't embed entire libraries if not needed
3. ❌ Don't pollute global namespace
4. ❌ Don't break on updates
5. ❌ Don't ignore user feedback
6. ❌ Don't use deprecated APIs without fallbacks
7. ❌ Don't create UI every frame (pool!)
8. ❌ Don't save temporary data
9. ❌ Don't ignore memory usage
10. ❌ Don't ship debug code in release

---

**Version:** 1.0 - Based on WoW 11.2.7 (The War Within)
**Last Updated:** 2025-10-19
**Community Patterns:** Ace3, LibStub, and common addon frameworks

<!-- CLAUDE_SKIP_END -->
