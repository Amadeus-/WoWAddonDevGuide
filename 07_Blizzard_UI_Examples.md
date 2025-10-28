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

**Key Code Patterns:**
```lua
function ActionButton_Update(self)
    local action = self.action;
    if not action then
        return;
    end

    -- Icon
    local icon = GetActionTexture(action);
    self.icon:SetTexture(icon);

    -- Count
    local count = GetActionCount(action);
    self.Count:SetText(count > 1 and count or "");

    -- Cooldown
    local start, duration = GetActionCooldown(action);
    CooldownFrame_Set(self.cooldown, start, duration, 1);

    -- Range indicator
    local inRange = IsActionInRange(action);
    if inRange == false then
        self.icon:SetVertexColor(0.8, 0.1, 0.1);
    else
        self.icon:SetVertexColor(1.0, 1.0, 1.0);
    end

    -- Usability (mana, requirements)
    local isUsable, notEnoughMana = IsUsableAction(action);
    if isUsable then
        self.icon:SetDesaturated(false);
    elseif notEnoughMana then
        self.icon:SetVertexColor(0.5, 0.5, 1.0);
    else
        self.icon:SetDesaturated(true);
    end
end
```

**Useful Patterns:**
- Icon masking for rounded corners
- Layer-based visual hierarchy
- Mixin composition for features
- Event-driven updates (ACTIONBAR_UPDATE_COOLDOWN, etc.)

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

**Aura Update Pattern:**
```lua
function AuraButton_Update(button, unit, index, filter)
    local aura = C_UnitAuras.GetAuraDataByIndex(unit, index, filter);
    if not aura then
        button:Hide();
        return;
    end

    -- Icon
    button.Icon:SetTexture(aura.icon);

    -- Count
    if aura.applications > 1 then
        button.Count:SetText(aura.applications);
        button.Count:Show();
    else
        button.Count:Hide();
    end

    -- Duration
    if aura.duration > 0 then
        button.expirationTime = aura.expirationTime;
        button:SetScript("OnUpdate", AuraButton_OnUpdate);
    else
        button.Duration:Hide();
        button:SetScript("OnUpdate", nil);
    end

    -- Border (debuff type coloring)
    if aura.isHarmful then
        local color = DebuffTypeColor[aura.dispelName] or DebuffTypeColor["none"];
        button.DebuffBorder:SetVertexColor(color.r, color.g, color.b);
        button.DebuffBorder:Show();
    else
        button.DebuffBorder:Hide();
    end

    button:Show();
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
local scrollBox = CreateFrame("Frame", nil, parent, "WowScrollBoxList");
local scrollBar = CreateFrame("EventFrame", nil, parent, "MinimalScrollBar");

-- Position
scrollBox:SetPoint("TOPLEFT", 10, -10);
scrollBox:SetPoint("BOTTOMRIGHT", scrollBar, "BOTTOMLEFT", -5, 10);
scrollBar:SetPoint("TOPRIGHT", -10, -10);
scrollBar:SetPoint("BOTTOMRIGHT", -10, 10);

-- Create view
local view = CreateScrollBoxListLinearView();

-- Set element initializer
view:SetElementInitializer("MyItemButtonTemplate", function(button, elementData)
    button.Name:SetText(elementData.name);
    button.Icon:SetTexture(elementData.icon);
    button.data = elementData;
end);

-- Initialize
ScrollUtil.InitScrollBoxListWithScrollBar(scrollBox, scrollBar, view);

-- Set data
local dataProvider = CreateDataProvider();
for i = 1, 100 do
    dataProvider:Insert({
        name = "Item " .. i,
        icon = "Interface\\Icons\\INV_Misc_QuestionMark",
    });
end

scrollBox:SetDataProvider(dataProvider);
```

**Data Provider Pattern:**
```lua
-- Create provider
local dataProvider = CreateDataProvider();

-- Listen for changes
dataProvider:RegisterCallback(DataProviderMixin.Event.OnSizeChanged, function()
    -- React to data changes
    print("Data changed!");
end);

-- Add data
dataProvider:Insert(item1, item2, item3);

-- Remove data
dataProvider:Remove(item);

-- Find data
local index = dataProvider:FindIndex(function(data)
    return data.id == searchID;
end);

-- Sort
dataProvider:SetSortComparator(function(a, b)
    return a.level > b.level;
end);

-- Enumerate
for index, data in dataProvider:Enumerate() do
    print(index, data.name);
end
```

---

## Dropdown Menus

### Dropdown Menu Pattern

**Source Files:**
- `Blizzard_SharedXML\Mainline\UIDropDownMenu.lua`
- `Blizzard_AddOnList\AddonList.lua`

**Basic Dropdown:**
```lua
-- Initialize dropdown
local function InitializeDropdown(self, level)
    local info = UIDropDownMenu_CreateInfo();

    -- Add title
    info.text = "Select an Option";
    info.isTitle = true;
    info.notCheckable = true;
    UIDropDownMenu_AddButton(info, level);

    -- Add separator
    info = UIDropDownMenu_CreateInfo();
    info.isTitle = false;
    info.notCheckable = true;
    info.disabled = true;
    UIDropDownMenu_AddSeparator(level);

    -- Add selectable options
    local options = {"Option 1", "Option 2", "Option 3"};
    for i, option in ipairs(options) do
        info = UIDropDownMenu_CreateInfo();
        info.text = option;
        info.value = i;
        info.func = function(self)
            print("Selected:", option);
            UIDropDownMenu_SetSelectedValue(dropdown, i);
        end;
        info.checked = (UIDropDownMenu_GetSelectedValue(dropdown) == i);
        UIDropDownMenu_AddButton(info, level);
    end
end

-- Setup dropdown
UIDropDownMenu_Initialize(dropdown, InitializeDropdown);
UIDropDownMenu_SetWidth(dropdown, 150);
UIDropDownMenu_SetButtonWidth(dropdown, 150);
UIDropDownMenu_JustifyText(dropdown, "LEFT");
```

**Nested Menus:**
```lua
local function InitializeDropdown(self, level, menuList)
    local info = UIDropDownMenu_CreateInfo();

    if level == 1 then
        -- Top level menu
        info.text = "Submenu 1";
        info.hasArrow = true;
        info.menuList = "submenu1";
        info.notCheckable = true;
        UIDropDownMenu_AddButton(info, level);

    elseif menuList == "submenu1" then
        -- Submenu items
        info.text = "Submenu Item 1";
        info.func = function() print("Clicked!"); end;
        UIDropDownMenu_AddButton(info, level);
    end
end
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
    GameTooltip:SetOwner(self, "ANCHOR_RIGHT");
    GameTooltip:SetText("Title Text", 1, 1, 1);  -- RGB
    GameTooltip:AddLine("Description text", NORMAL_FONT_COLOR.r, NORMAL_FONT_COLOR.g, NORMAL_FONT_COLOR.b, true);  -- wrap
    GameTooltip:Show();
end

function MyButton_OnLeave(self)
    GameTooltip:Hide();
end
```

**Advanced Tooltip:**
```lua
function ShowItemTooltip(button)
    GameTooltip:SetOwner(button, "ANCHOR_RIGHT");

    -- Set item
    GameTooltip:SetItemByID(itemID);

    -- Or set spell
    GameTooltip:SetSpellByID(spellID);

    -- Or set unit
    GameTooltip:SetUnit("player");

    -- Add custom lines
    GameTooltip:AddLine(" ");  -- Blank line
    GameTooltip:AddDoubleLine("Label:", "Value", nil, nil, nil, 1, 1, 1);
    GameTooltip:AddLine("Red text", 1, 0, 0);

    -- Add texture
    GameTooltip:AddTexture("Interface\\Icons\\INV_Misc_QuestionMark");

    GameTooltip:Show();
end
```

**Shopping Tooltip:**
```lua
-- Show comparison tooltips
function ShowComparisonTooltip(button, itemLink)
    GameTooltip:SetOwner(button, "ANCHOR_RIGHT");
    GameTooltip:SetHyperlink(itemLink);

    -- Show comparison
    if IsModifiedClick("COMPAREITEMS") or GetCVarBool("alwaysCompareItems") then
        GameTooltip_ShowCompareItem();
    end

    GameTooltip:Show();
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
local numQuests = C_QuestLog.GetNumQuestLogEntries();

for i = 1, numQuests do
    local info = C_QuestLog.GetInfo(i);

    if info and not info.isHeader and not info.isHidden then
        -- Quest ID
        local questID = info.questID;

        -- Quest details
        local title = info.title;
        local level = info.level;
        local isComplete = C_QuestLog.IsComplete(questID);
        local isOnMap = C_QuestLog.IsOnMap(questID);

        -- Objectives
        local objectives = C_QuestLog.GetQuestObjectives(questID);
        for j, objective in ipairs(objectives) do
            local text = objective.text;
            local finished = objective.finished;
            local numFulfilled = objective.numFulfilled;
            local numRequired = objective.numRequired;

            print(format("%s: %d/%d", text, numFulfilled, numRequired));
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
MyAddonMapPinMixin = CreateFromMixins(MapCanvasPinMixin);

function MyAddonMapPinMixin:OnLoad()
    self:UseFrameLevelType("PIN_FRAME_LEVEL_TOPMOST");
end

function MyAddonMapPinMixin:OnAcquired(data)
    self.data = data;
    self.Texture:SetAtlas(data.atlas);
    self:SetPosition(data.x, data.y);
end

-- Data provider
MyAddonDataProviderMixin = CreateFromMixins(MapCanvasDataProviderMixin);

function MyAddonDataProviderMixin:OnAdded()
    MapCanvasDataProviderMixin.OnAdded(self);
    self:GetMap():RegisterCallback("SetFocusedQuestID", self.OnQuestChanged, self);
end

function MyAddonDataProviderMixin:RefreshAllData()
    self:RemoveAllData();

    -- Add pins
    for i, location in ipairs(myLocations) do
        self:GetMap():AcquirePin("MyAddonMapPinTemplate", location);
    end
end
```

### Minimap Button

**Source Files:**
- `Blizzard_SharedXML\Minimap.lua`

**Minimap Button Pattern:**
```lua
local button = CreateFrame("Button", "MyAddonMinimapButton", Minimap);
button:SetSize(32, 32);
button:SetFrameStrata("MEDIUM");
button:SetFrameLevel(8);

-- Texture
button.icon = button:CreateTexture(nil, "BACKGROUND");
button.icon:SetSize(20, 20);
button.icon:SetTexture("Interface\\Icons\\INV_Misc_QuestionMark");
button.icon:SetPoint("CENTER", 0, 1);

-- Border
button.border = button:CreateTexture(nil, "OVERLAY");
button.border:SetSize(52, 52);
button.border:SetTexture("Interface\\Minimap\\MiniMap-TrackingBorder");
button.border:SetPoint("TOPLEFT");

-- Position around minimap
local function UpdatePosition()
    local angle = math.rad(MyAddonDB.minimapAngle or 0);
    local x = math.cos(angle) * 80;
    local y = math.sin(angle) * 80;
    button:SetPoint("CENTER", Minimap, "CENTER", x, y);
end

-- Dragging
button:RegisterForDrag("LeftButton");
button:SetScript("OnDragStart", function(self)
    self:SetScript("OnUpdate", function(self)
        local mx, my = Minimap:GetCenter();
        local px, py = GetCursorPosition();
        local scale = Minimap:GetEffectiveScale();
        px, py = px / scale, py / scale;

        local angle = math.atan2(py - my, px - mx);
        MyAddonDB.minimapAngle = math.deg(angle);
        UpdatePosition();
    end);
end);

button:SetScript("OnDragStop", function(self)
    self:SetScript("OnUpdate", nil);
end);
```

---

## Chat and Communication

### Chat Frame Usage

**Source Files:**
- `Blizzard_ChatFrameBase\Shared\ChatFrame.lua`

**Print to Chat:**
```lua
-- Default chat frame
DEFAULT_CHAT_FRAME:AddMessage("Hello, World!", 1, 1, 0);  -- Yellow

-- Specific chat frame
ChatFrame3:AddMessage("Debug info", 0, 1, 0);  -- Green

-- With icon
ChatFrame1:AddMessage(format("|T%s:16|t %s",
    "Interface\\Icons\\Achievement_Boss_Ragnaros",
    "Achievement earned!"));
```

**Chat Links:**
```lua
-- Item link
local itemLink = select(2, GetItemInfo(itemID));
print("Item:", itemLink);

-- Achievement link
local achievementLink = GetAchievementLink(achievementID);
print("Achievement:", achievementLink);

-- Spell link
local spellLink = GetSpellLink(spellID);
print("Spell:", spellLink);

-- Custom link
local customLink = format("|Hgarrmission:missionID:%d|h[Mission Name]|h", missionID);
```

**Hyperlink Clicks:**
```lua
-- Handle hyperlink clicks
local frame = CreateFrame("Frame");
frame:SetScript("OnHyperlinkClick", function(self, link, text, button)
    local linkType, data = link:match("(%w+):(.+)");

    if linkType == "item" then
        local itemID = tonumber(data);
        -- Handle item click
    elseif linkType == "spell" then
        local spellID = tonumber(data);
        -- Handle spell click
    end
end);
```

---

## Auction House

### Auction House Search

**Source Files:**
- `Blizzard_AuctionHouseUI\Shared\Blizzard_AuctionData.lua`

**Search Pattern:**
```lua
-- Search for items
C_AuctionHouse.SearchForItemKeys(itemKeys, sorts);

-- Event: AUCTION_HOUSE_BROWSE_RESULTS_UPDATED
local frame = CreateFrame("Frame");
frame:RegisterEvent("AUCTION_HOUSE_BROWSE_RESULTS_UPDATED");
frame:SetScript("OnEvent", function(self, event)
    if event == "AUCTION_HOUSE_BROWSE_RESULTS_UPDATED" then
        local numResults = C_AuctionHouse.GetNumBrowseResults();

        for i = 1, numResults do
            local result = C_AuctionHouse.GetBrowseResultInfo(i);

            if result then
                print(format("%s x%d - %s",
                    result.itemKey.itemName,
                    result.quantity,
                    C_CurrencyInfo.GetCoinTextureString(result.minPrice)));
            end
        end
    end
end);
```

---

## Unit Frames

### Unit Frame Pattern

**Source Files:**
- `Blizzard_UnitFrame\UnitFrame.lua`

**Basic Unit Frame:**
```lua
local frame = CreateFrame("Button", "MyUnitFrame", UIParent, "SecureUnitButtonTemplate");
frame:SetSize(100, 50);
frame:SetPoint("CENTER");
frame:SetAttribute("unit", "player");

-- Health bar
frame.healthBar = CreateFrame("StatusBar", nil, frame);
frame.healthBar:SetSize(100, 20);
frame.healthBar:SetPoint("TOP");
frame.healthBar:SetStatusBarTexture("Interface\\TargetingFrame\\UI-StatusBar");
frame.healthBar:SetStatusBarColor(0, 1, 0);

-- Update health
frame:RegisterEvent("UNIT_HEALTH");
frame:SetScript("OnEvent", function(self, event, unit)
    if unit == self:GetAttribute("unit") then
        local health = UnitHealth(unit);
        local healthMax = UnitHealthMax(unit);
        self.healthBar:SetMinMaxValues(0, healthMax);
        self.healthBar:SetValue(health);
    end
end);
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
| Dropdown Menus | `Blizzard_SharedXML` | `UIDropDownMenu.lua` |
| Unit Frames | `Blizzard_UnitFrame` | `UnitFrame.lua` |

All paths relative to: `D:\Games\World of Warcraft\_retail_\Interface\+wow-ui-source+ (11.2.7)\Interface\AddOns\`

---

**Version:** 1.0 - Based on WoW 11.2.7 (The War Within)
**Last Updated:** 2025-10-19

<!-- CLAUDE_SKIP_END -->
