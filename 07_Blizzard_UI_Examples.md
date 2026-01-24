# Blizzard UI Examples - Real-World Code References

## Table of Contents
1. [Action Buttons](#action-buttons)
2. [Buff/Debuff Frames](#buffdebuff-frames)
3. [Scroll Frames and Lists](#scroll-frames-and-lists)
4. [Dropdown Menus](#dropdown-menus)
5. [Tooltips](#tooltips)
6. [Quest Tracking](#quest-tracking)
7. [Map and Minimap](#map-and-minimap)
8. [Chat and Communication](#chat-and-communication)
9. [Auction House](#auction-house)
10. [Unit Frames](#unit-frames)
11. [Damage Meter](#damage-meter)
12. [Encounter Timeline](#encounter-timeline)
13. [Housing UI](#housing-ui)
14. [Accessibility Features](#accessibility-features)
15. [Transmog and Outfits](#transmog-and-outfits)

---

## Important Notes for 12.0.0 (Midnight)

### Structural Changes
- **FrameXML restructured**: Many core UI files have been reorganized. Check `Interface\FrameXML` and `Interface\GlueXML` for updated paths.
- **Global functions moved to C_* namespaces**: Continued migration of legacy globals to namespaced APIs.
- **Blizzard_ActionBar is now the primary source** for all action bar related code and templates.
- **New addon loading system**: Performance monitoring integrated at the addon level.

### New Blizzard AddOns in 12.0.0
| Addon | Purpose |
|-------|---------|
| `Blizzard_DamageMeter` | Official damage/healing meter |
| `Blizzard_EncounterTimeline` | Boss ability timeline display |
| `Blizzard_EncounterWarnings` | Encounter warning system (DBM-like) |
| `Blizzard_Housing` | Core player housing UI |
| `Blizzard_HousingEditor` | Housing placement/editing interface |
| `Blizzard_HousingFurniture` | Furniture catalog and management |
| `Blizzard_AccessibilityTemplates` | Accessibility UI components |
| `Blizzard_AddOnPerformance` | Addon performance monitoring |
| `Blizzard_CombatAudioAlerts` | Text-to-speech combat alerts |

---

## Action Buttons

### Complete Action Button Implementation

**Source Files:**
- `Blizzard_ActionBar\Mainline\ActionButtonTemplate.xml`
- `Blizzard_ActionBar\Shared\ActionButton.lua`

**Key Features:**
- Cooldown tracking
- Range checking
- Out-of-mana indication
- Keybinding display
- Spell flyouts
- Drag-and-drop
- Charge-based abilities (12.0+)

**Reference Structure:**
```xml
<CheckButton name="ActionButtonTemplate"
             inherits="ActionButtonSpellFXTemplate, FlyoutButtonTemplate"
             mixin="BaseActionButtonMixin"
             virtual="true">
    <Size x="45" y="45"/>
    <Layers>
        <Layer level="BACKGROUND">
            <Texture parentKey="icon"/>
            <Texture parentKey="SlotBackground" atlas="UI-HUD-ActionBar-IconFrame-Background"/>
        </Layer>
        <Layer level="ARTWORK">
            <Texture parentKey="Flash" atlas="UI-HUD-ActionBar-IconFrame-Flash"/>
            <FontString parentKey="Name" inherits="GameFontHighlightSmallOutline"/>
            <FontString parentKey="HotKey" inherits="NumberFontNormalSmallGray"/>
        </Layer>
        <Layer level="OVERLAY">
            <FontString parentKey="Count" inherits="NumberFontNormal"/>
        </Layer>
    </Layers>
    <Cooldown parentKey="cooldown" inherits="CooldownFrameTemplate"/>
</CheckButton>
```

**Key Code Patterns (12.0 Updated):**
```lua
function ActionButton_Update(self)
    local action = self.action
    if not action then
        return
    end

    -- Icon (use C_ActionBar namespace when available)
    local icon = C_ActionBar.GetActionTexture and C_ActionBar.GetActionTexture(action) or GetActionTexture(action)
    self.icon:SetTexture(icon)

    -- Count
    local count = GetActionCount(action)
    self.Count:SetText(count > 1 and count or "")

    -- Cooldown (modern API with charges support)
    local cooldownInfo = C_ActionBar.GetActionCooldown and C_ActionBar.GetActionCooldown(action)
    if cooldownInfo then
        CooldownFrame_Set(self.cooldown, cooldownInfo.startTime, cooldownInfo.duration, cooldownInfo.enable, cooldownInfo.modRate)

        -- Handle charges
        if cooldownInfo.chargeModRate then
            self.cooldown:SetDrawBling(cooldownInfo.chargeModRate > 0)
        end
    else
        -- Fallback for older API
        local start, duration, enable = GetActionCooldown(action)
        CooldownFrame_Set(self.cooldown, start, duration, enable)
    end

    -- Range indicator
    local inRange = C_ActionBar.IsActionInRange and C_ActionBar.IsActionInRange(action) or IsActionInRange(action)
    if inRange == false then
        self.icon:SetVertexColor(0.8, 0.1, 0.1)
    else
        self.icon:SetVertexColor(1.0, 1.0, 1.0)
    end

    -- Usability (mana, requirements)
    local isUsable, notEnoughMana = IsUsableAction(action)
    if isUsable then
        self.icon:SetDesaturated(false)
    elseif notEnoughMana then
        self.icon:SetVertexColor(0.5, 0.5, 1.0)
    else
        self.icon:SetDesaturated(true)
    end
end
```

**Useful Patterns:**
- Icon masking for rounded corners
- Layer-based visual hierarchy
- Mixin composition for features
- Event-driven updates (ACTIONBAR_UPDATE_COOLDOWN, etc.)
- Charge-based spell handling (12.0+)

---

## Buff/Debuff Frames

### Aura Button System

**Source Files:**
- `Blizzard_BuffFrame\BuffFrameTemplates.xml`
- `Blizzard_BuffFrame\BuffFrame.lua`

**3-Tier Template Pattern:**
```xml
<!-- Tier 1: Visual Structure -->
<Frame name="AuraButtonArtTemplate" virtual="true">
    <Size x="30" y="40"/>
    <Layers>
        <Layer level="BACKGROUND">
            <Texture parentKey="Icon">
                <Size x="30" y="30"/>
                <Anchors>
                    <Anchor point="TOP"/>
                </Anchors>
            </Texture>
            <FontString parentKey="Count" inherits="NumberFontNormal">
                <Anchors>
                    <Anchor point="BOTTOMRIGHT" relativeKey="$parent.Icon" x="-2" y="2"/>
                </Anchors>
            </FontString>
            <FontString parentKey="Duration" inherits="GameFontNormalSmall" hidden="true">
                <Anchors>
                    <Anchor point="TOP" relativeKey="$parent.Icon" relativePoint="BOTTOM"/>
                </Anchors>
            </FontString>
        </Layer>
        <Layer level="OVERLAY">
            <Texture parentKey="DebuffBorder"/>
        </Layer>
    </Layers>
</Frame>

<!-- Tier 2: Add Mixin -->
<Button name="AuraButtonCodeTemplate"
        inherits="AuraButtonArtTemplate"
        mixin="AuraButtonMixin"
        virtual="true"/>

<!-- Tier 3: Add Scripts -->
<Button name="AuraButtonTemplate"
        inherits="AuraButtonCodeTemplate"
        virtual="true">
    <Scripts>
        <OnLoad method="OnLoad"/>
        <OnUpdate method="OnUpdate"/>
        <OnEnter method="OnEnter"/>
        <OnLeave method="OnLeave"/>
    </Scripts>
</Button>
```

**Aura Update Pattern (12.0 Updated):**
```lua
function AuraButton_Update(button, unit, index, filter)
    local aura = C_UnitAuras.GetAuraDataByIndex(unit, index, filter)
    if not aura then
        button:Hide()
        return
    end

    -- Icon
    button.Icon:SetTexture(aura.icon)

    -- Count (applications)
    if aura.applications > 1 then
        button.Count:SetText(aura.applications)
        button.Count:Show()
    else
        button.Count:Hide()
    end

    -- Duration
    if aura.duration > 0 then
        button.expirationTime = aura.expirationTime
        button:SetScript("OnUpdate", AuraButton_OnUpdate)
    else
        button.Duration:Hide()
        button:SetScript("OnUpdate", nil)
    end

    -- Border (debuff type coloring)
    if aura.isHarmful then
        local color = DebuffTypeColor[aura.dispelName] or DebuffTypeColor["none"]
        button.DebuffBorder:SetVertexColor(color.r, color.g, color.b)
        button.DebuffBorder:Show()
    else
        button.DebuffBorder:Hide()
    end

    -- New in 12.0: Aura source tracking
    if aura.sourceUnit then
        button.sourceUnit = aura.sourceUnit
    end

    button:Show()
end

-- Alternative: Query by spell ID (12.0 preferred method)
function AuraButton_UpdateBySpellID(button, unit, spellID)
    local aura = C_UnitAuras.GetAuraDataBySpellName(unit, spellID)
    if aura then
        AuraButton_Update(button, unit, aura.auraInstanceID)
    end
end
```

---

## Scroll Frames and Lists

### Modern ScrollBox Pattern

**Source Files:**
- `Blizzard_SharedXML\Shared\Scroll\ScrollBox.lua`
- `Blizzard_AuctionHouseUI\Shared\Blizzard_AuctionHouseCommoditiesList.lua`

**Setup Pattern:**
```lua
-- Create scroll box and scrollbar
local scrollBox = CreateFrame("Frame", nil, parent, "WowScrollBoxList")
local scrollBar = CreateFrame("EventFrame", nil, parent, "MinimalScrollBar")

-- Position
scrollBox:SetPoint("TOPLEFT", 10, -10)
scrollBox:SetPoint("BOTTOMRIGHT", scrollBar, "BOTTOMLEFT", -5, 10)
scrollBar:SetPoint("TOPRIGHT", -10, -10)
scrollBar:SetPoint("BOTTOMRIGHT", -10, 10)

-- Create view
local view = CreateScrollBoxListLinearView()

-- Set element initializer
view:SetElementInitializer("MyItemButtonTemplate", function(button, elementData)
    button.Name:SetText(elementData.name)
    button.Icon:SetTexture(elementData.icon)
    button.data = elementData
end)

-- Initialize
ScrollUtil.InitScrollBoxListWithScrollBar(scrollBox, scrollBar, view)

-- Set data
local dataProvider = CreateDataProvider()
for i = 1, 100 do
    dataProvider:Insert({
        name = "Item " .. i,
        icon = "Interface\\Icons\\INV_Misc_QuestionMark",
    })
end

scrollBox:SetDataProvider(dataProvider)
```

**Data Provider Pattern:**
```lua
-- Create provider
local dataProvider = CreateDataProvider()

-- Listen for changes
dataProvider:RegisterCallback(DataProviderMixin.Event.OnSizeChanged, function()
    -- React to data changes
    print("Data changed!")
end)

-- Add data
dataProvider:Insert(item1, item2, item3)

-- Remove data
dataProvider:Remove(item)

-- Find data
local index = dataProvider:FindIndex(function(data)
    return data.id == searchID
end)

-- Sort
dataProvider:SetSortComparator(function(a, b)
    return a.level > b.level
end)

-- Enumerate
for index, data in dataProvider:Enumerate() do
    print(index, data.name)
end
```

---

## Dropdown Menus

### Dropdown Menu Pattern

**Source Files:**
- `Blizzard_SharedXML\Mainline\UIDropDownMenu.lua`
- `Blizzard_Menu\MenuUtil.lua` (12.0 new menu system)

**Modern Menu System (12.0 Preferred):**
```lua
-- New MenuUtil system (12.0+)
local function CreateMyMenu(ownerRegion, rootDescription)
    rootDescription:CreateTitle("Select an Option")
    rootDescription:CreateDivider()

    local options = {"Option 1", "Option 2", "Option 3"}
    for i, option in ipairs(options) do
        local button = rootDescription:CreateButton(option, function()
            print("Selected:", option)
            MyAddonDB.selectedOption = i
        end)

        -- Check mark for current selection
        if MyAddonDB.selectedOption == i then
            button:SetIsSelected(function() return true end)
        end
    end

    -- Submenu
    local submenu = rootDescription:CreateButton("More Options")
    submenu:CreateTitle("Submenu")
    submenu:CreateButton("Sub-option A", function() print("A") end)
    submenu:CreateButton("Sub-option B", function() print("B") end)
end

-- Show menu
MenuUtil.CreateContextMenu(myButton, CreateMyMenu)
```

**Legacy Dropdown (Still Supported):**
```lua
-- Initialize dropdown
local function InitializeDropdown(self, level)
    local info = UIDropDownMenu_CreateInfo()

    -- Add title
    info.text = "Select an Option"
    info.isTitle = true
    info.notCheckable = true
    UIDropDownMenu_AddButton(info, level)

    -- Add separator
    info = UIDropDownMenu_CreateInfo()
    info.isTitle = false
    info.notCheckable = true
    info.disabled = true
    UIDropDownMenu_AddSeparator(level)

    -- Add selectable options
    local options = {"Option 1", "Option 2", "Option 3"}
    for i, option in ipairs(options) do
        info = UIDropDownMenu_CreateInfo()
        info.text = option
        info.value = i
        info.func = function(self)
            print("Selected:", option)
            UIDropDownMenu_SetSelectedValue(dropdown, i)
        end
        info.checked = (UIDropDownMenu_GetSelectedValue(dropdown) == i)
        UIDropDownMenu_AddButton(info, level)
    end
end

-- Setup dropdown
UIDropDownMenu_Initialize(dropdown, InitializeDropdown)
UIDropDownMenu_SetWidth(dropdown, 150)
UIDropDownMenu_SetButtonWidth(dropdown, 150)
UIDropDownMenu_JustifyText(dropdown, "LEFT")
```

---

## Tooltips

### Tooltip Patterns

**Source Files:**
- `Blizzard_SharedXML\SharedTooltipTemplates.lua`
- `Blizzard_UIPanels_Game\Mainline\GameTooltip.lua`

**Basic Tooltip:**
```lua
function MyButton_OnEnter(self)
    GameTooltip:SetOwner(self, "ANCHOR_RIGHT")
    GameTooltip:SetText("Title Text", 1, 1, 1)  -- RGB
    GameTooltip:AddLine("Description text", NORMAL_FONT_COLOR.r, NORMAL_FONT_COLOR.g, NORMAL_FONT_COLOR.b, true)  -- wrap
    GameTooltip:Show()
end

function MyButton_OnLeave(self)
    GameTooltip:Hide()
end
```

**Advanced Tooltip (12.0 Updated):**
```lua
function ShowItemTooltip(button)
    GameTooltip:SetOwner(button, "ANCHOR_RIGHT")

    -- Set item (prefer new API)
    if C_TooltipInfo and C_TooltipInfo.GetItemByID then
        local tooltipData = C_TooltipInfo.GetItemByID(itemID)
        if tooltipData then
            GameTooltip:ProcessInfo(tooltipData)
        end
    else
        GameTooltip:SetItemByID(itemID)
    end

    -- Or set spell
    if C_TooltipInfo and C_TooltipInfo.GetSpellByID then
        local tooltipData = C_TooltipInfo.GetSpellByID(spellID)
        if tooltipData then
            GameTooltip:ProcessInfo(tooltipData)
        end
    else
        GameTooltip:SetSpellByID(spellID)
    end

    -- Add custom lines
    GameTooltip:AddLine(" ")  -- Blank line
    GameTooltip:AddDoubleLine("Label:", "Value", nil, nil, nil, 1, 1, 1)
    GameTooltip:AddLine("Red text", 1, 0, 0)

    -- Add texture
    GameTooltip:AddTexture("Interface\\Icons\\INV_Misc_QuestionMark")

    GameTooltip:Show()
end
```

**Shopping Tooltip:**
```lua
-- Show comparison tooltips
function ShowComparisonTooltip(button, itemLink)
    GameTooltip:SetOwner(button, "ANCHOR_RIGHT")
    GameTooltip:SetHyperlink(itemLink)

    -- Show comparison
    if IsModifiedClick("COMPAREITEMS") or C_CVar.GetCVarBool("alwaysCompareItems") then
        GameTooltip_ShowCompareItem()
    end

    GameTooltip:Show()
end
```

---

## Quest Tracking

### Quest Objectives Pattern

**Source Files:**
- `Blizzard_ObjectiveTracker\Blizzard_ObjectiveTracker.lua`
- `Blizzard_QuestNavigation\QuestDataProvider.lua`

**Quest Info Retrieval:**
```lua
-- Get quest log info
local numQuests = C_QuestLog.GetNumQuestLogEntries()

for i = 1, numQuests do
    local info = C_QuestLog.GetInfo(i)

    if info and not info.isHeader and not info.isHidden then
        -- Quest ID
        local questID = info.questID

        -- Quest details
        local title = info.title
        local level = info.level
        local isComplete = C_QuestLog.IsComplete(questID)
        local isOnMap = C_QuestLog.IsOnMap(questID)

        -- Objectives
        local objectives = C_QuestLog.GetQuestObjectives(questID)
        for j, objective in ipairs(objectives) do
            local text = objective.text
            local finished = objective.finished
            local numFulfilled = objective.numFulfilled
            local numRequired = objective.numRequired

            print(format("%s: %d/%d", text, numFulfilled, numRequired))
        end
    end
end
```

---

## Map and Minimap

### Map Pin System

**Source Files:**
- `Blizzard_SharedMapDataProviders\MapDataProviderBase.lua`
- `Blizzard_SharedMapDataProviders\QuestDataProvider.lua`

**Custom Map Pin:**
```lua
-- Define pin mixin
MyAddonMapPinMixin = CreateFromMixins(MapCanvasPinMixin)

function MyAddonMapPinMixin:OnLoad()
    self:UseFrameLevelType("PIN_FRAME_LEVEL_TOPMOST")
end

function MyAddonMapPinMixin:OnAcquired(data)
    self.data = data
    self.Texture:SetAtlas(data.atlas)
    self:SetPosition(data.x, data.y)
end

-- Data provider
MyAddonDataProviderMixin = CreateFromMixins(MapCanvasDataProviderMixin)

function MyAddonDataProviderMixin:OnAdded()
    MapCanvasDataProviderMixin.OnAdded(self)
    self:GetMap():RegisterCallback("SetFocusedQuestID", self.OnQuestChanged, self)
end

function MyAddonDataProviderMixin:RefreshAllData()
    self:RemoveAllData()

    -- Add pins
    for i, location in ipairs(myLocations) do
        self:GetMap():AcquirePin("MyAddonMapPinTemplate", location)
    end
end
```

### Minimap Button

**Source Files:**
- `Blizzard_SharedXML\Minimap.lua`

**Minimap Button Pattern:**
```lua
local button = CreateFrame("Button", "MyAddonMinimapButton", Minimap)
button:SetSize(32, 32)
button:SetFrameStrata("MEDIUM")
button:SetFrameLevel(8)

-- Texture
button.icon = button:CreateTexture(nil, "BACKGROUND")
button.icon:SetSize(20, 20)
button.icon:SetTexture("Interface\\Icons\\INV_Misc_QuestionMark")
button.icon:SetPoint("CENTER", 0, 1)

-- Border
button.border = button:CreateTexture(nil, "OVERLAY")
button.border:SetSize(52, 52)
button.border:SetTexture("Interface\\Minimap\\MiniMap-TrackingBorder")
button.border:SetPoint("TOPLEFT")

-- Position around minimap
local function UpdatePosition()
    local angle = math.rad(MyAddonDB.minimapAngle or 0)
    local x = math.cos(angle) * 80
    local y = math.sin(angle) * 80
    button:SetPoint("CENTER", Minimap, "CENTER", x, y)
end

-- Dragging
button:RegisterForDrag("LeftButton")
button:SetScript("OnDragStart", function(self)
    self:SetScript("OnUpdate", function(self)
        local mx, my = Minimap:GetCenter()
        local px, py = GetCursorPosition()
        local scale = Minimap:GetEffectiveScale()
        px, py = px / scale, py / scale

        local angle = math.atan2(py - my, px - mx)
        MyAddonDB.minimapAngle = math.deg(angle)
        UpdatePosition()
    end)
end)

button:SetScript("OnDragStop", function(self)
    self:SetScript("OnUpdate", nil)
end)
```

---

## Chat and Communication

### Chat Frame Usage

**Source Files:**
- `Blizzard_ChatFrameBase\Shared\ChatFrame.lua`

**Print to Chat:**
```lua
-- Default chat frame
DEFAULT_CHAT_FRAME:AddMessage("Hello, World!", 1, 1, 0)  -- Yellow

-- Specific chat frame
ChatFrame3:AddMessage("Debug info", 0, 1, 0)  -- Green

-- With icon
ChatFrame1:AddMessage(format("|T%s:16|t %s",
    "Interface\\Icons\\Achievement_Boss_Ragnaros",
    "Achievement earned!"))
```

**Chat Links:**
```lua
-- Item link
local itemLink = select(2, C_Item.GetItemInfo(itemID))
print("Item:", itemLink)

-- Achievement link
local achievementLink = GetAchievementLink(achievementID)
print("Achievement:", achievementLink)

-- Spell link (12.0 uses C_Spell)
local spellLink = C_Spell.GetSpellLink(spellID)
print("Spell:", spellLink)

-- Custom link
local customLink = format("|Hgarrmission:missionID:%d|h[Mission Name]|h", missionID)
```

**Hyperlink Clicks:**
```lua
-- Handle hyperlink clicks
local frame = CreateFrame("Frame")
frame:SetScript("OnHyperlinkClick", function(self, link, text, button)
    local linkType, data = link:match("(%w+):(.+)")

    if linkType == "item" then
        local itemID = tonumber(data)
        -- Handle item click
    elseif linkType == "spell" then
        local spellID = tonumber(data)
        -- Handle spell click
    end
end)
```

---

## Auction House

### Auction House Search

**Source Files:**
- `Blizzard_AuctionHouseUI\Shared\Blizzard_AuctionData.lua`

**Search Pattern:**
```lua
-- Search for items
C_AuctionHouse.SearchForItemKeys(itemKeys, sorts)

-- Event: AUCTION_HOUSE_BROWSE_RESULTS_UPDATED
local frame = CreateFrame("Frame")
frame:RegisterEvent("AUCTION_HOUSE_BROWSE_RESULTS_UPDATED")
frame:SetScript("OnEvent", function(self, event)
    if event == "AUCTION_HOUSE_BROWSE_RESULTS_UPDATED" then
        local numResults = C_AuctionHouse.GetNumBrowseResults()

        for i = 1, numResults do
            local result = C_AuctionHouse.GetBrowseResultInfo(i)

            if result then
                print(format("%s x%d - %s",
                    result.itemKey.itemName,
                    result.quantity,
                    C_CurrencyInfo.GetCoinTextureString(result.minPrice)))
            end
        end
    end
end)
```

---

## Unit Frames

### Unit Frame Pattern

**Source Files:**
- `Blizzard_UnitFrame\UnitFrame.lua`

**Basic Unit Frame:**
```lua
local frame = CreateFrame("Button", "MyUnitFrame", UIParent, "SecureUnitButtonTemplate")
frame:SetSize(100, 50)
frame:SetPoint("CENTER")
frame:SetAttribute("unit", "player")

-- Health bar
frame.healthBar = CreateFrame("StatusBar", nil, frame)
frame.healthBar:SetSize(100, 20)
frame.healthBar:SetPoint("TOP")
frame.healthBar:SetStatusBarTexture("Interface\\TargetingFrame\\UI-StatusBar")
frame.healthBar:SetStatusBarColor(0, 1, 0)

-- Update health
frame:RegisterEvent("UNIT_HEALTH")
frame:SetScript("OnEvent", function(self, event, unit)
    if unit == self:GetAttribute("unit") then
        local health = UnitHealth(unit)
        local healthMax = UnitHealthMax(unit)
        self.healthBar:SetMinMaxValues(0, healthMax)
        self.healthBar:SetValue(health)
    end
end)
```

---

## Damage Meter

### Official Damage Meter (12.0 New) - ⛔ SECRET-PROTECTED

> **⚠️ CRITICAL (Verified January 2026):** While `Blizzard_DamageMeter` addon and `C_DamageMeter` API exist, **the data is protected as "secret values"** and cannot be accessed by third-party addons. The code examples below show the API structure but **WILL NOT WORK** in practice.

**Source Files:**
- `Blizzard_DamageMeter\Blizzard_DamageMeter.lua`
- `Blizzard_DamageMeter\Blizzard_DamageMeterTemplates.xml`

**Why Blizzard's Code Works But Yours Won't:**
Blizzard's own UI code runs in a secure execution context and can access secret values. Third-party addon code runs in an insecure context and cannot.

**What Happens When You Try:**
```lua
-- The API exists and IsDamageMeterAvailable() returns true
if C_DamageMeter then
    local available = C_DamageMeter.IsDamageMeterAvailable() -- Returns true!

    -- You can get session data...
    local sessionData = C_DamageMeter.GetCombatSessionFromType(
        Enum.DamageMeterSessionType.Overall,
        Enum.DamageMeterType.DamageDone
    )

    -- ...but the VALUES inside are SECRET:
    for _, source in ipairs(sessionData.combatSources) do
        -- These throw "attempt to compare (a secret value)" errors:
        if source.name == "SomePlayer" then end  -- ERROR!

        -- These return <no value> (secret):
        local damage = source.totalAmount     -- SECRET
        local dps = source.amountPerSecond    -- SECRET
        local guid = source.sourceGUID        -- SECRET

        -- Only cosmetic data works (useless without names/amounts):
        local class = source.classFilename    -- Works: "WARRIOR"
        local isMe = source.isLocalPlayer     -- Works: true
    end
end
```

**The Bottom Line:**
- `C_DamageMeter` is for Blizzard's built-in meter ONLY
- Third-party addons CANNOT create custom damage meters in 12.0.0+
- Players must use Blizzard's meter (Shift+P or Encounter Journal)

**Events (Fire But Contain No Usable Data):**
```lua
-- These events fire, but you can't do anything useful with them
local frame = CreateFrame("Frame")
frame:RegisterEvent("DAMAGE_METER_COMBAT_SESSION_UPDATED")
frame:RegisterEvent("DAMAGE_METER_CURRENT_SESSION_UPDATED")
frame:RegisterEvent("DAMAGE_METER_RESET")
frame:SetScript("OnEvent", function(self, event, ...)
    -- Events fire, but when you try to read the data, it's all secret
    print("Event fired:", event)  -- This works
    -- Actually using the data? Nope, it's secret.
end)
```

---

## Encounter Timeline

### Boss Ability Timeline (12.0 New)

**Source Files:**
- `Blizzard_EncounterTimeline\Blizzard_EncounterTimeline.lua`
- `Blizzard_EncounterWarnings\Blizzard_EncounterWarnings.lua`

**Key APIs:**
```lua
-- Encounter timeline data
if C_EncounterTimeline then
    -- Get abilities for current encounter
    local encounterID = C_EncounterJournal.GetActiveEncounterID()
    if encounterID then
        local abilities = C_EncounterTimeline.GetEncounterAbilities(encounterID)

        for _, ability in ipairs(abilities) do
            print(format("Ability: %s (Spell ID: %d)", ability.name, ability.spellID))
            print(format("  First cast: %.1fs, Cooldown: %.1fs", ability.firstCastTime, ability.cooldown))
            print(format("  Phase: %d, Interruptible: %s", ability.phase, tostring(ability.interruptible)))
        end
    end

    -- Get current phase
    local currentPhase = C_EncounterTimeline.GetCurrentPhase()

    -- Get time until next ability
    local nextAbility = C_EncounterTimeline.GetNextScheduledAbility()
    if nextAbility then
        local timeUntil = nextAbility.scheduledTime - GetTime()
        print(format("%s in %.1f seconds", nextAbility.name, timeUntil))
    end
end

-- Encounter warnings
if C_EncounterWarnings then
    -- Register for specific ability warnings
    C_EncounterWarnings.RegisterSpellWarning(spellID, function(warningInfo)
        print(format("WARNING: %s incoming!", warningInfo.spellName))
        -- Play custom alert
    end)

    -- Get warning settings
    local settings = C_EncounterWarnings.GetWarningSettings()
    print("Audio alerts:", settings.audioEnabled)
    print("Screen flash:", settings.screenFlashEnabled)
end
```

**Custom Warning Integration:**
```lua
-- Create custom encounter warning frame
local warningFrame = CreateFrame("Frame", "MyEncounterWarning", UIParent)
warningFrame:SetSize(400, 50)
warningFrame:SetPoint("TOP", 0, -200)

warningFrame.text = warningFrame:CreateFontString(nil, "OVERLAY", "GameFontNormalHuge")
warningFrame.text:SetPoint("CENTER")

warningFrame.animation = warningFrame:CreateAnimationGroup()
local fadeIn = warningFrame.animation:CreateAnimation("Alpha")
fadeIn:SetFromAlpha(0)
fadeIn:SetToAlpha(1)
fadeIn:SetDuration(0.2)
fadeIn:SetOrder(1)

local fadeOut = warningFrame.animation:CreateAnimation("Alpha")
fadeOut:SetFromAlpha(1)
fadeOut:SetToAlpha(0)
fadeOut:SetDuration(0.5)
fadeOut:SetStartDelay(2)
fadeOut:SetOrder(2)

function warningFrame:ShowWarning(text, color)
    self.text:SetText(text)
    self.text:SetTextColor(color.r, color.g, color.b)
    self:Show()
    self.animation:Play()
end

warningFrame.animation:SetScript("OnFinished", function()
    warningFrame:Hide()
end)

-- Hook to Blizzard warnings
if C_EncounterWarnings then
    hooksecurefunc(C_EncounterWarnings, "ShowWarning", function(spellID, text)
        -- Optionally show custom warning alongside official one
        warningFrame:ShowWarning(text, {r=1, g=0.5, b=0})
    end)
end
```

---

## Housing UI

### Player Housing System (12.0 New)

For comprehensive housing system documentation, see: **12_Housing_System_Guide.md**

**Source Files:**
- `Blizzard_Housing\Blizzard_Housing.lua`
- `Blizzard_HousingEditor\Blizzard_HousingEditor.lua`
- `Blizzard_HousingFurniture\Blizzard_HousingFurniture.lua`

**Key APIs Overview:**
```lua
-- Check if housing is available
if C_Housing then
    -- Get player's housing plot info
    local plotInfo = C_Housing.GetPlayerPlotInfo()
    if plotInfo then
        print("Plot ID:", plotInfo.plotID)
        print("Plot Name:", plotInfo.name)
        print("Zone:", plotInfo.zoneName)
    end

    -- Get placed furniture
    local furniture = C_Housing.GetPlacedFurniture()
    for _, item in ipairs(furniture) do
        print(format("Furniture: %s at (%.1f, %.1f, %.1f)",
            item.name, item.x, item.y, item.z))
    end

    -- Check if in edit mode
    local isEditing = C_Housing.IsInEditMode()

    -- Get furniture catalog
    local catalog = C_Housing.GetFurnitureCatalog()
    for categoryID, items in pairs(catalog) do
        for _, itemInfo in ipairs(items) do
            print(format("  %s (ID: %d) - %s",
                itemInfo.name, itemInfo.furnitureID,
                itemInfo.owned and "Owned" or "Not Owned"))
        end
    end
end
```

**Housing Events:**
```lua
local frame = CreateFrame("Frame")
frame:RegisterEvent("HOUSING_ENTERED")
frame:RegisterEvent("HOUSING_EXITED")
frame:RegisterEvent("HOUSING_EDIT_MODE_CHANGED")
frame:RegisterEvent("HOUSING_FURNITURE_PLACED")
frame:RegisterEvent("HOUSING_FURNITURE_REMOVED")
frame:RegisterEvent("HOUSING_FURNITURE_MOVED")

frame:SetScript("OnEvent", function(self, event, ...)
    if event == "HOUSING_ENTERED" then
        local plotID, ownerName = ...
        print(format("Entered %s's housing plot", ownerName))
    elseif event == "HOUSING_EDIT_MODE_CHANGED" then
        local isEditing = ...
        if isEditing then
            print("Edit mode enabled")
        else
            print("Edit mode disabled")
        end
    elseif event == "HOUSING_FURNITURE_PLACED" then
        local furnitureID, instanceID = ...
        print("Furniture placed:", furnitureID)
    end
end)
```

**Custom Housing UI Integration:**
```lua
-- Create a furniture counter widget
local furnitureCounter = CreateFrame("Frame", "MyFurnitureCounter", UIParent, "BackdropTemplate")
furnitureCounter:SetSize(150, 30)
furnitureCounter:SetPoint("TOPLEFT", HousingFrame, "TOPRIGHT", 10, 0)

furnitureCounter.text = furnitureCounter:CreateFontString(nil, "OVERLAY", "GameFontNormal")
furnitureCounter.text:SetPoint("CENTER")

local function UpdateCounter()
    if not C_Housing then return end

    local furniture = C_Housing.GetPlacedFurniture()
    local limit = C_Housing.GetFurnitureLimit()
    furnitureCounter.text:SetText(format("Furniture: %d / %d", #furniture, limit))

    if #furniture >= limit then
        furnitureCounter.text:SetTextColor(1, 0.3, 0.3)
    else
        furnitureCounter.text:SetTextColor(1, 1, 1)
    end
end

furnitureCounter:RegisterEvent("HOUSING_FURNITURE_PLACED")
furnitureCounter:RegisterEvent("HOUSING_FURNITURE_REMOVED")
furnitureCounter:RegisterEvent("HOUSING_ENTERED")
furnitureCounter:SetScript("OnEvent", UpdateCounter)
```

---

## Accessibility Features

### Accessibility Templates (12.0 New)

**Source Files:**
- `Blizzard_AccessibilityTemplates\Blizzard_AccessibilityTemplates.lua`
- `Blizzard_CombatAudioAlerts\Blizzard_CombatAudioAlerts.lua`

**Key APIs:**
```lua
-- Accessibility settings
if C_Accessibility then
    -- Check TTS settings
    local ttsEnabled = C_Accessibility.IsTextToSpeechEnabled()
    local ttsRate = C_Accessibility.GetTextToSpeechRate()

    -- Speak text (for addon accessibility)
    if ttsEnabled then
        C_Accessibility.SpeakText("Important notification", Enum.TtsVoiceType.Standard)
    end

    -- Check colorblind mode
    local colorblindMode = C_Accessibility.GetColorblindMode()
    -- 0 = None, 1 = Protanopia, 2 = Deuteranopia, 3 = Tritanopia

    -- Get motion sickness reduction setting
    local reduceMotion = C_Accessibility.GetReduceMotionEnabled()
end

-- Combat audio alerts
if C_CombatAudioAlerts then
    -- Check if combat TTS is enabled
    local combatTTSEnabled = C_CombatAudioAlerts.IsEnabled()

    -- Get alert settings
    local settings = C_CombatAudioAlerts.GetSettings()
    print("Announce interrupts:", settings.announceInterrupts)
    print("Announce low health:", settings.announceLowHealth)
    print("Low health threshold:", settings.lowHealthThreshold)
end
```

**Creating Accessible Addons:**
```lua
-- Accessible button template
local function CreateAccessibleButton(parent, name, text)
    local button = CreateFrame("Button", name, parent, "UIPanelButtonTemplate, AccessibleButtonTemplate")
    button:SetText(text)

    -- Set accessibility label (for screen readers)
    if button.SetAccessibilityLabel then
        button:SetAccessibilityLabel(text)
    end

    -- High contrast support
    button:HookScript("OnEnter", function(self)
        if C_Accessibility and C_Accessibility.IsHighContrastEnabled() then
            self:GetFontString():SetTextColor(1, 1, 0)  -- Yellow for visibility
        end
    end)

    return button
end

-- Announce important events with TTS
local function AnnounceToPlayer(message, priority)
    -- Visual notification
    UIErrorsFrame:AddMessage(message, 1, 1, 0)

    -- Audio notification for accessibility
    if C_Accessibility and C_Accessibility.IsTextToSpeechEnabled() then
        local voiceType = priority == "HIGH" and Enum.TtsVoiceType.Alternate or Enum.TtsVoiceType.Standard
        C_Accessibility.SpeakText(message, voiceType)
    end
end
```

---

## Transmog and Outfits

### Transmog System (12.0 Updated)

**Note:** The transmog system was significantly overhauled in 12.0. Old transmog APIs have been replaced with `C_TransmogOutfitInfo`.

**Source Files:**
- `Blizzard_Collections\Blizzard_Wardrobe.lua`
- `Blizzard_Collections\Blizzard_TransmogOutfits.lua`

**Key APIs (12.0):**
```lua
-- Get outfit information
if C_TransmogOutfitInfo then
    -- Get all saved outfits
    local outfits = C_TransmogOutfitInfo.GetOutfits()
    for _, outfitID in ipairs(outfits) do
        local name = C_TransmogOutfitInfo.GetOutfitName(outfitID)
        local icon = C_TransmogOutfitInfo.GetOutfitIcon(outfitID)
        print(format("Outfit: %s (ID: %d)", name, outfitID))
    end

    -- Get outfit sources
    local outfitID = outfits[1]
    if outfitID then
        local sources = C_TransmogOutfitInfo.GetOutfitSources(outfitID)
        for slotID, sourceInfo in pairs(sources) do
            if sourceInfo then
                print(format("Slot %d: Appearance %d", slotID, sourceInfo.appearanceID))
            end
        end
    end

    -- Create new outfit from current appearance
    local newOutfitID = C_TransmogOutfitInfo.CreateOutfitFromCurrentAppearance("My New Outfit")

    -- Apply outfit
    C_TransmogOutfitInfo.ApplyOutfit(outfitID)

    -- Delete outfit
    C_TransmogOutfitInfo.DeleteOutfit(outfitID)

    -- Rename outfit
    C_TransmogOutfitInfo.RenameOutfit(outfitID, "New Name")
end

-- Check appearance collection status
if C_TransmogCollection then
    -- Check if an appearance is collected
    local appearanceID = 12345
    local isCollected = C_TransmogCollection.PlayerHasTransmog(appearanceID)

    -- Get appearance sources
    local sources = C_TransmogCollection.GetAppearanceSources(appearanceID)
    for _, sourceInfo in ipairs(sources) do
        print(format("Source: %s (Item ID: %d)", sourceInfo.name, sourceInfo.itemID))
    end

    -- Get slot appearances
    local slotAppearances = C_TransmogCollection.GetSlotAppearances(Enum.TransmogSlot.Head)
    print(format("Head slot has %d appearances", #slotAppearances))
end
```

**Transmog Events (12.0):**
```lua
local frame = CreateFrame("Frame")
frame:RegisterEvent("TRANSMOG_COLLECTION_UPDATED")
frame:RegisterEvent("TRANSMOG_OUTFIT_UPDATED")
frame:RegisterEvent("TRANSMOG_OUTFITS_CHANGED")
frame:RegisterEvent("TRANSMOG_SETS_UPDATE_FAVORITE")

frame:SetScript("OnEvent", function(self, event, ...)
    if event == "TRANSMOG_COLLECTION_UPDATED" then
        local collectionSlot, newAppearanceID = ...
        if newAppearanceID then
            print("New appearance collected:", newAppearanceID)
        end
    elseif event == "TRANSMOG_OUTFIT_UPDATED" then
        local outfitID = ...
        print("Outfit updated:", outfitID)
    elseif event == "TRANSMOG_OUTFITS_CHANGED" then
        -- Refresh outfit list
        print("Outfits changed")
    end
end)
```

**Custom Outfit Manager:**
```lua
-- Create simple outfit quick-swap buttons
local function CreateOutfitQuickSwap()
    local container = CreateFrame("Frame", "MyOutfitQuickSwap", UIParent)
    container:SetSize(200, 40)
    container:SetPoint("BOTTOM", 0, 100)

    if not C_TransmogOutfitInfo then
        return
    end

    local outfits = C_TransmogOutfitInfo.GetOutfits()
    local buttonWidth = 36
    local spacing = 4

    for i, outfitID in ipairs(outfits) do
        if i > 5 then break end  -- Max 5 buttons

        local button = CreateFrame("Button", nil, container)
        button:SetSize(buttonWidth, buttonWidth)
        button:SetPoint("LEFT", (i-1) * (buttonWidth + spacing), 0)

        local icon = C_TransmogOutfitInfo.GetOutfitIcon(outfitID)
        button.Icon = button:CreateTexture(nil, "ARTWORK")
        button.Icon:SetAllPoints()
        button.Icon:SetTexture(icon)

        button.outfitID = outfitID
        button:SetScript("OnClick", function(self)
            C_TransmogOutfitInfo.ApplyOutfit(self.outfitID)
        end)

        button:SetScript("OnEnter", function(self)
            GameTooltip:SetOwner(self, "ANCHOR_TOP")
            GameTooltip:SetText(C_TransmogOutfitInfo.GetOutfitName(self.outfitID))
            GameTooltip:Show()
        end)

        button:SetScript("OnLeave", GameTooltip_Hide)
    end

    return container
end
```

---

<!-- CLAUDE_SKIP_START -->
## Key Blizzard Addon References

| Feature | Source Addon | Key Files |
|---------|--------------|-----------|
| Action Bars | `Blizzard_ActionBar` | `ActionButton.lua`, `ActionButtonTemplate.xml` |
| Buffs/Debuffs | `Blizzard_BuffFrame` | `BuffFrame.lua`, `BuffFrameTemplates.xml` |
| Scroll Lists | `Blizzard_AuctionHouseUI` | `Blizzard_AuctionHouseCommoditiesList.lua` |
| Quest Tracking | `Blizzard_ObjectiveTracker` | `Blizzard_ObjectiveTracker.lua` |
| Map Pins | `Blizzard_SharedMapDataProviders` | `QuestDataProvider.lua` |
| Tooltips | `Blizzard_UIPanels_Game` | `GameTooltip.lua` |
| Dropdown Menus | `Blizzard_SharedXML`, `Blizzard_Menu` | `UIDropDownMenu.lua`, `MenuUtil.lua` |
| Unit Frames | `Blizzard_UnitFrame` | `UnitFrame.lua` |
| Damage Meter | `Blizzard_DamageMeter` | `Blizzard_DamageMeter.lua` |
| Encounter Timeline | `Blizzard_EncounterTimeline` | `Blizzard_EncounterTimeline.lua` |
| Encounter Warnings | `Blizzard_EncounterWarnings` | `Blizzard_EncounterWarnings.lua` |
| Housing | `Blizzard_Housing`, `Blizzard_HousingEditor` | `Blizzard_Housing.lua` |
| Accessibility | `Blizzard_AccessibilityTemplates` | `Blizzard_AccessibilityTemplates.lua` |
| Combat Audio | `Blizzard_CombatAudioAlerts` | `Blizzard_CombatAudioAlerts.lua` |
| Performance | `Blizzard_AddOnPerformance` | `Blizzard_AddOnPerformance.lua` |
| Transmog/Outfits | `Blizzard_Collections` | `Blizzard_Wardrobe.lua`, `Blizzard_TransmogOutfits.lua` |

All paths relative to: `Interface\AddOns\`

---

**Version:** 2.0 - Updated for WoW 12.0.0 (Midnight)
**Interface Version:** 120000
**Last Updated:** 2026-01-20

<!-- CLAUDE_SKIP_END -->
