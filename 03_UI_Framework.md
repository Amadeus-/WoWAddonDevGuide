# WoW UI Framework - Comprehensive Guide

## Table of Contents
1. [XML UI Structure and Patterns](#xml-ui-structure-and-patterns)
2. [Frame Scripting Patterns](#frame-scripting-patterns)
3. [Widget and Region Types](#widget-and-region-types)
4. [New Widget Methods (11.1.5 - 12.0.0)](#new-widget-methods-1115---1200)
5. [Secret Values System (12.0.0)](#secret-values-system-1200)
6. [Common Blizzard Templates](#common-blizzard-templates)
7. [Frame Pooling and Object Reuse](#frame-pooling-and-object-reuse)
8. [Data Provider Pattern](#data-provider-pattern)
9. [Practical Examples](#practical-examples)
10. [Best Practices](#best-practices)

---

## XML UI Structure and Patterns

### Core Frame Types and Inheritance

The WoW UI framework uses XML for defining visual structures. Frames can inherit from multiple templates and bind to Lua mixins for behavior.

**Key Concepts:**
- **virtual="true"** - Template not instantiated, only inherited
- **inherits** - Comma-separated list of parent templates
- **mixin** - Lua table(s) to merge with frame instance
- **parentKey** - Named access to child elements (e.g., `frame.Icon` instead of `_G["frameName" .. "Icon"]`)

**Example: Action Button Template**
```xml
<CheckButton name="ActionButtonTemplate"
             inherits="ActionButtonSpellFXTemplate, FlyoutButtonTemplate"
             mixin="BaseActionButtonMixin"
             virtual="true">
    <KeyValues>
        <KeyValue key="enableSpellFX" value="true" type="boolean"/>
        <KeyValue key="popupDirection" value="UP" type="string"/>
    </KeyValues>
    <Size x="45" y="45"/>
    <Layers>
        <Layer level="BACKGROUND">
            <Texture name="$parentIcon" parentKey="icon" />
            <MaskTexture parentKey="IconMask" atlas="UI-HUD-ActionBar-IconFrame-Mask"
                         hWrapMode="CLAMPTOBLACKADDITIVE" vWrapMode="CLAMPTOBLACKADDITIVE">
                <Anchors>
                    <Anchor point="CENTER" relativeKey="$parent.icon"/>
                </Anchors>
                <MaskedTextures>
                    <MaskedTexture childKey="icon"/>
                </MaskedTextures>
            </MaskTexture>
        </Layer>
        <Layer level="ARTWORK" textureSubLevel="1">
            <Texture name="$parentFlash" parentKey="Flash"
                     atlas="UI-HUD-ActionBar-IconFrame-Flash" hidden="true">
                <Anchors>
                    <Anchor point="TOPLEFT"/>
                </Anchors>
            </Texture>
        </Layer>
    </Layers>
    <Animations>
        <AnimationGroup parentKey="SpellHighlightAnim" looping="REPEAT">
            <Alpha childKey="SpellHighlightTexture" smoothing="OUT"
                   duration=".35" fromAlpha="0" toAlpha="1"/>
        </AnimationGroup>
    </Animations>
    <Scripts>
        <OnLoad method="BaseActionButtonMixin_OnLoad"/>
        <OnEnter method="BaseActionButtonMixin_OnEnter"/>
        <OnLeave method="BaseActionButtonMixin_OnLeave"/>
    </Scripts>
</CheckButton>
```

**Source:** `Blizzard_ActionBar\Mainline\ActionButtonTemplate.xml`

### 3-Tier Template Inheritance Pattern

Blizzard commonly uses a 3-tier pattern for reusable components:

1. **Art Template** - Visual structure only (no code)
2. **Code Template** - Adds mixin for behavior
3. **Full Template** - Adds scripts that wire to mixin methods

**Example: Aura Button Template**
```xml
<!-- Tier 1: Pure Art Template -->
<Frame name="AuraButtonArtTemplate" virtual="true">
    <Size x="30" y="40"/>
    <Layers>
        <Layer level="BACKGROUND">
            <Texture parentKey="Icon" file="Interface\ICONS\INV_Misc_QuestionMark.blp">
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
        </Layer>
    </Layers>
</Frame>

<!-- Tier 2: Code Template (Mixin) -->
<Button name="AuraButtonCodeTemplate"
        inherits="AuraButtonArtTemplate"
        mixin="AuraButtonMixin"
        virtual="true"/>

<!-- Tier 3: Full Template (with scripts) -->
<Button name="AuraButtonTemplate" inherits="AuraButtonCodeTemplate" virtual="true">
    <Scripts>
        <OnLoad method="OnLoad"/>
        <OnClick method="OnClick"/>
        <OnEnter method="OnEnter"/>
        <OnLeave method="OnLeave"/>
        <OnUpdate method="OnUpdate"/>
    </Scripts>
</Button>
```

**Benefits:**
- Art can be reused without code dependencies
- Code can be tested independently
- Full template provides complete component
- Easy to create variants by mixing different tiers

**Source:** `Blizzard_BuffFrame\BuffFrameTemplates.xml`

### Naming Conventions

**$parent Token:**
- `$parent` - Replaced with parent frame name
- `$parentIcon` - Creates name like "MyFrameIcon"
- Used for global frame names (less common now)

**parentKey (Modern):**
- `parentKey="Icon"` - Accessible as `frame.Icon`
- No global namespace pollution
- Preferred for new code

**relativeKey:**
- `relativeKey="$parent.Icon"` - Anchor to sibling
- Navigation within frame hierarchy

**childKey:**
- Used in animations to target child elements
- `childKey="SpellHighlightTexture"`

---

## Frame Scripting Patterns

### Mixin System

WoW uses a composition-based approach with mixins instead of inheritance.

**Basic Mixin Creation:**
```lua
MyAddonFrameMixin = {};

function MyAddonFrameMixin:OnLoad()
    self:RegisterEvent("PLAYER_LOGIN");
    self:RegisterEvent("PLAYER_LOGOUT");
end

function MyAddonFrameMixin:OnEvent(event, ...)
    if event == "PLAYER_LOGIN" then
        self:Initialize();
    elseif event == "PLAYER_LOGOUT" then
        self:SaveData();
    end
end

function MyAddonFrameMixin:Initialize()
    print("MyAddon initialized!");
end
```

**Applying to XML:**
```xml
<Frame name="MyAddonFrame" mixin="MyAddonFrameMixin">
    <Scripts>
        <OnLoad method="OnLoad"/>
        <OnEvent method="OnEvent"/>
    </Scripts>
</Frame>
```

### CreateFromMixins Pattern

Combine multiple mixins for composition:

```lua
ScrollBoxListViewMixin = CreateFromMixins(ScrollBoxViewMixin, CallbackRegistryMixin);

ScrollBoxListViewMixin:GenerateCallbackEvents({
    "OnDataChanged",
    "OnDataProviderReassigned",
    "OnAcquiredFrame",
    "OnInitializedFrame",
    "OnReleasedFrame",
});

function ScrollBoxListViewMixin:Init()
    CallbackRegistryMixin.OnLoad(self);
    ScrollBoxViewMixin.Init(self);

    self.frameFactory = CreateFrameFactory();
    self.initializers = {};
end
```

**Source:** `Blizzard_SharedXML\Shared\Scroll\ScrollBoxListView.lua`

**Key Points:**
- Methods from all mixins are merged
- Later mixins override earlier ones
- `GenerateCallbackEvents()` creates event system
- Multiple initializers may need to be called

### Intrinsic Methods Pattern

Blizzard uses `_Intrinsic` suffix for framework internals:

```lua
EventFrameMixin = CreateFromMixins(CallbackRegistryMixin);

EventFrameMixin:GenerateCallbackEvents({
    "OnHide",
    "OnShow",
    "OnSizeChanged",
});

function EventFrameMixin:OnLoad_Intrinsic()
    CallbackRegistryMixin.OnLoad(self);
end

function EventFrameMixin:OnHide_Intrinsic()
    self:TriggerEvent("OnHide");
end

function EventFrameMixin:OnShow_Intrinsic()
    self:TriggerEvent("OnShow");
end

function EventFrameMixin:OnSizeChanged_Intrinsic(width, height)
    self:TriggerEvent("OnSizeChanged", width, height);
end
```

**Source:** `Blizzard_SharedXML\Shared\Frame\EventFrame.lua`

**Pattern:**
- `_Intrinsic` methods are called by XML scripts
- They trigger callback events
- Enables event-driven architecture
- Addons register callbacks instead of overriding scripts

### Layout Frame with Dirty Marking

Efficient layout updates using dirty marking:

```lua
BaseLayoutMixin = {};

function BaseLayoutMixin:OnShow()
    if not self.skipLayoutOnShow then
        self:Layout();
    end
end

function BaseLayoutMixin:MarkDirty()
    self.dirty = true;

    -- Only set OnUpdate while marked dirty for performance
    self:SetScript("OnUpdate", self.OnUpdate);

    -- Propagate to parent layout frames
    local parent = self:GetParent();
    while parent do
        if IsLayoutFrame(parent) then
            parent:MarkDirty();
            return;
        end
        parent = parent:GetParent();
    end
end

function BaseLayoutMixin:OnUpdate()
    if self:IsDirty() then
        self:Layout();
    end
end

function BaseLayoutMixin:Layout()
    self.dirty = false;
    self:SetScript("OnUpdate", nil);  -- Remove OnUpdate when clean

    -- Perform actual layout...
end
```

**Source:** `Blizzard_SharedXML\LayoutFrame.lua`

**Performance Benefits:**
- OnUpdate only set when needed
- Lazy layout updates
- Batch multiple changes into one layout
- Propagates dirty state up hierarchy

---

## Widget and Region Types

### Layer Draw Order

Layers control the z-order (draw order) of visual elements:

```
BACKGROUND (z=0) - Backgrounds and base textures
    ↓
BORDER (z=1) - Borders and separators
    ↓
ARTWORK (z=2) - Main artwork and decorations
    ↓
OVERLAY (z=3) - Text and overlays
    ↓
HIGHLIGHT (z=4) - Mouseover highlights
```

**textureSubLevel** adds fractional ordering within a layer (0-7):

```xml
<Layers>
    <Layer level="BACKGROUND">
        <Texture parentKey="BaseTexture"/>  <!-- Drawn first -->
    </Layer>

    <Layer level="ARTWORK" textureSubLevel="1">
        <Texture parentKey="Flash"/>  <!-- ARTWORK + 0.1 -->
    </Layer>

    <Layer level="ARTWORK" textureSubLevel="2">
        <Texture parentKey="Overlay"/>  <!-- ARTWORK + 0.2 (on top of Flash) -->
    </Layer>

    <Layer level="OVERLAY">
        <FontString parentKey="Name"/>  <!-- On top of all artwork -->
    </Layer>
</Layers>
```

**Source:** `Blizzard_ActionBar\Mainline\ActionButtonTemplate.xml`

### Common Widget Types

| Widget Type | Purpose | Key Methods |
|-------------|---------|-------------|
| **Frame** | Base container | `SetPoint()`, `SetSize()`, `Show()`, `Hide()` |
| **Button** | Clickable button | `SetText()`, `Click()`, `RegisterForClicks()` |
| **CheckButton** | Toggle button | `SetChecked()`, `GetChecked()` |
| **EditBox** | Text input | `SetText()`, `GetText()`, `SetMaxLetters()` |
| **ScrollFrame** | Scrollable container | `SetScrollChild()`, `GetHorizontalScroll()` |
| **Slider** | Value slider | `SetValue()`, `GetValue()`, `SetMinMaxValues()` |
| **StatusBar** | Progress bar | `SetStatusBarTexture()`, `SetValue()` |
| **Texture** | Image display | `SetTexture()`, `SetAtlas()`, `SetTexCoord()` |
| **FontString** | Text display | `SetText()`, `SetFont()`, `SetTextColor()` |
| **Model** | 3D model display | `SetModel()`, `SetCamera()` |
| **ModelScene** | 3D scene | `CreateActor()`, `SetLight()` |

### Texture Atlas and TexCoords

**Atlas (Modern):**
```xml
<Texture parentKey="Icon" atlas="UI-HUD-ActionBar-IconFrame-Background"
         useAtlasSize="true">
    <Anchors>
        <Anchor point="CENTER"/>
    </Anchors>
</Texture>
```

```lua
-- In code:
frame.Icon:SetAtlas("UI-HUD-ActionBar-IconFrame-Background");
frame.Icon:SetAtlas("UI-HUD-ActionBar-IconFrame-Flash", true);  -- useAtlasSize
```

**TexCoords (Legacy):**
```xml
<Texture parentKey="Border" file="Interface\Buttons\UI-Debuff-Overlays">
    <Size x="33" y="32"/>
    <Anchors>
        <Anchor point="CENTER"/>
    </Anchors>
    <TexCoords left="0.296875" right="0.5703125" top="0" bottom="0.515625"/>
</Texture>
```

```lua
-- In code:
frame.Border:SetTexture("Interface\\Buttons\\UI-Debuff-Overlays");
frame.Border:SetTexCoord(0.296875, 0.5703125, 0, 0.515625);
```

**Source:** `Blizzard_BuffFrame\BuffFrameTemplates.xml`

### Anchor System

Anchors position and size frames relative to others:

**Single Anchor (Position):**
```xml
<Frame name="MyFrame">
    <Size x="100" y="50"/>
    <Anchors>
        <Anchor point="CENTER" x="0" y="0"/>
    </Anchors>
</Frame>
```

**Multiple Anchors (Position + Size):**
```xml
<Frame name="MyScrollbar">
    <Size x="20" y="0"/>  <!-- y ignored, controlled by anchors -->
    <Anchors>
        <!-- Anchor top-left to parent's top-right -->
        <Anchor point="TOPLEFT" relativePoint="TOPRIGHT" x="5" y="-10"/>
        <!-- Anchor bottom-left to parent's bottom-right -->
        <Anchor point="BOTTOMLEFT" relativePoint="BOTTOMRIGHT" x="5" y="10"/>
    </Anchors>
</Frame>
```

**Relative to Sibling:**
```xml
<Texture parentKey="Icon"/>
<FontString parentKey="Count">
    <Anchors>
        <Anchor point="BOTTOMRIGHT" relativeKey="$parent.Icon" x="-2" y="2"/>
    </Anchors>
</FontString>
```

**Fill Parent:**
```xml
<Texture setAllPoints="true"/>
<!-- Same as: -->
<Texture>
    <Anchors>
        <Anchor point="TOPLEFT"/>
        <Anchor point="BOTTOMRIGHT"/>
    </Anchors>
</Texture>
```

**In Code:**
```lua
-- Single anchor
frame:SetPoint("CENTER", UIParent, "CENTER", 0, 0);

-- Multiple anchors
scrollbar:SetPoint("TOPLEFT", parent, "TOPRIGHT", 5, -10);
scrollbar:SetPoint("BOTTOMLEFT", parent, "BOTTOMRIGHT", 5, 10);

-- Relative to sibling
frame.Count:SetPoint("BOTTOMRIGHT", frame.Icon, "BOTTOMRIGHT", -2, 2);

-- Clear and reset
frame:ClearAllPoints();
frame:SetAllPoints(parent);
```

**Anchor Points:**
```
TOPLEFT -------- TOP -------- TOPRIGHT
   |                              |
   |                              |
  LEFT          CENTER          RIGHT
   |                              |
   |                              |
BOTTOMLEFT --- BOTTOM --- BOTTOMRIGHT
```

**Source:** `Blizzard_SharedXML\HybridScrollFrame.xml`

---

## New Widget Methods (11.1.5 - 12.0.0)

This section covers new widget methods added in recent patches. For the secret values system, see the dedicated section below.

### FrameScriptObject Methods

**Secret Value Methods (12.0.0):**
```lua
-- Check if any secret aspects are set on this object
local hasAnySecret = frame:HasAnySecretAspect()

-- Check for a specific secret aspect
local hasTextSecret = frame:HasSecretAspect("Text")
local hasValueSecret = frame:HasSecretAspect("Value")

-- Check if the object has any secret values
local hasSecretValues = frame:HasSecretValues()

-- Check/Set prevention of secret values
local isPreventing = frame:IsPreventingSecretValues()
frame:SetPreventSecretValues(true)  -- Block secrets from propagating

-- Clear all secret aspects and reset to defaults
frame:SetToDefaults()
```

### ScriptRegion Methods

```lua
-- Check if anchoring involves secret values (12.0.0)
local isSecretAnchored = region:IsAnchoringSecret()

-- Clear all scripts from this region (11.2.0)
region:ClearScripts()

-- Check button pass-through state (11.2.0)
local passThrough = region:ShouldButtonPassThrough()
```

### Region Methods

**Secret Value Display (12.0.0):**
```lua
-- Set alpha based on boolean (for secret values)
-- When value is secret, this allows conditional visibility
region:SetAlphaFromBoolean(shouldShow)

-- Set vertex color based on boolean (for secret values)
region:SetVertexColorFromBoolean(shouldShow)
```

### Frame Methods

**Event Callbacks (12.0.0):**
```lua
-- Register a callback for an event (alternative to OnEvent script)
frame:RegisterEventCallback("PLAYER_ENTERING_WORLD", function(event, ...)
    print("Player entered world!")
end)

-- Register a unit event callback
frame:RegisterUnitEventCallback("UNIT_HEALTH", function(event, unit)
    print(unit, "health changed")
end, "player")
```

**Attribute Management (11.2.0):**
```lua
-- Clear a specific attribute
frame:ClearAttribute("unit")

-- Clear all attributes
frame:ClearAttributes()
```

**Alpha Gradients (11.1.5, 11.2.0):**
```lua
-- Apply alpha gradient (fades from startAlpha to endAlpha across the frame)
frame:SetAlphaGradient(startX, startY, endX, endY)

-- Check if frame has an alpha gradient
local hasGradient = frame:HasAlphaGradient()  -- 11.2.0

-- Clear the alpha gradient
frame:ClearAlphaGradient()  -- 11.2.0
```

**Hyperlink Propagation (11.0.5):**
```lua
-- Check if hyperlinks propagate to parent
local propagates = frame:DoesHyperlinkPropagateToParent()

-- Enable/disable hyperlink propagation to parent frame
frame:SetHyperlinkPropagateToParent(true)
```

**Other Frame Methods:**
```lua
-- Check if frame is a framebuffer (11.2.0)
local isBuffer = frame:IsFrameBuffer()

-- Check if highlight is locked (11.2.0)
local isLocked = frame:IsHighlightLocked()

-- Get highest frame level among children (11.1.7)
local highestLevel = frame:GetHighestFrameLevel()

-- Control whether children affect bounds calculation (12.0.0)
local ignoring = frame:IsIgnoringChildrenForBounds()
frame:SetIgnoringChildrenForBounds(true)
```

### FontString Methods

**Font Height (11.2.0):**
```lua
-- Get the actual rendered font height
local height = fontString:GetFontHeight()

-- Set the font height directly
fontString:SetFontHeight(14)
```

**Scale Animation Mode (12.0.0):**
```lua
-- Get current scale animation mode
local mode = fontString:GetScaleAnimationMode()

-- Set scale animation mode
fontString:SetScaleAnimationMode(mode)
```

**Alpha Gradients (11.2.0):**
```lua
-- Get current alpha gradient settings
local startX, startY, endX, endY = fontString:GetAlphaGradient()

-- Clear the alpha gradient
fontString:ClearAlphaGradient()
```

**Color Overrides (11.1.5):**
```lua
-- Callback when colors are updated (e.g., item quality colors changed)
function MyFontStringMixin:OnColorsUpdated()
    -- Refresh any cached color values
    self:UpdateTextColor()
end
```

### StatusBar Methods

**Interpolated Values (12.0.0):**
```lua
-- Get the current interpolated value (during smooth transitions)
local interpolatedValue = statusBar:GetInterpolatedValue()

-- Check if currently interpolating
local isInterpolating = statusBar:IsInterpolating()

-- Jump directly to target value (skip interpolation)
statusBar:SetToTargetValue(100)
```

**Timer Duration (12.0.0):**
```lua
-- Get current timer duration
local duration = statusBar:GetTimerDuration()

-- Set timer duration (used with secret duration values)
statusBar:SetTimerDuration(durationObject)
```

### Cooldown Methods

**Countdown FontString (12.0.0):**
```lua
-- Get the FontString used for countdown display
local fontString = cooldown:GetCountdownFontString()
```

**Duration Objects (12.0.0):**
```lua
-- Set cooldown from a Duration object (for secret durations)
cooldown:SetCooldownFromDurationObject(durationObject)

-- Set cooldown from expiration time
cooldown:SetCooldownFromExpirationTime(expirationTime)
```

**Cooldown Control:**
```lua
-- Pause/unpause the cooldown animation (12.0.0)
cooldown:SetPaused(true)

-- Set the edge color (11.2.0)
cooldown:SetEdgeColor(r, g, b, a)

-- Get/Set minimum countdown duration for text display (11.2.5)
local minDuration = cooldown:GetMinimumCountdownDuration()
cooldown:SetMinimumCountdownDuration(3)  -- Only show countdown for 3+ seconds
```

### TextureBase Methods

```lua
-- Reset texture coordinates to default (12.0.0)
texture:ResetTexCoord()

-- Set texture to a specific sprite sheet cell (12.0.0)
texture:SetSpriteSheetCell(cellIndex)

-- Clear all vertex offsets (11.2.0)
texture:ClearVertexOffsets()
```

### Model and ModelScene Methods

```lua
-- Set gradient mask on model (11.2.7)
model:SetGradientMask(texturePath, blendMode)

-- ModelSceneActor methods (11.2.7)
modelSceneActor:SetGradientMask(texturePath, blendMode)

-- Check collision bounds preference (11.2.7)
local prefersCollision = modelSceneActor:IsPreferringModelCollisionBounds()

-- Set collision bounds preference (11.2.7)
modelSceneActor:SetPreferModelCollisionBounds(true)
```

---

## Secret Values System (12.0.0)

The Secret Values system was introduced in 12.0.0 (Midnight) to hide sensitive information from addons while still allowing the UI to display it. This is used for competitive integrity features.

### Understanding Secret Values

Secret values are special protected data types that:
- Cannot be read by addon code directly
- Can be displayed in UI widgets (FontString, StatusBar, etc.)
- Propagate through the UI hierarchy
- Mark widgets with "secret aspects" when applied

### StatusBar Frames and Secret Values (Critical for Health Bars)

**Native StatusBar frames** are designed to handle secret values at the C++ level:

```lua
-- This WORKS with secret values from UnitHealth/UnitHealthMax:
local healthBar = CreateFrame("StatusBar", nil, parent)
healthBar:SetMinMaxValues(0, UnitHealthMax("target"))  -- Secret value OK!
healthBar:SetValue(UnitHealth("target"))               -- Secret value OK!
-- The bar displays correctly even though values are secret
```

**Custom texture-based bars CANNOT use secret values:**

```lua
-- THIS FAILS with secret values:
local health = UnitHealth("target")      -- Secret during combat
local maxHealth = UnitHealthMax("target") -- Secret during combat
local percent = health / maxHealth        -- ERROR: arithmetic on secret value!
texture:SetWidth(percent * BAR_WIDTH)     -- Never reached
```

**Solution: Use UnitHealthPercent for custom bars:**

```lua
-- UnitHealthPercent returns NON-SECRET percentage (0-100)
local healthPercent = UnitHealthPercent("target", false, CurveConstants.ScaleTo100)
local width = (healthPercent / 100) * BAR_WIDTH  -- Safe arithmetic!
texture:SetWidth(width)
```

See `12a_Secret_Safe_APIs.md` for the complete secret values API reference, and `12_API_Migration_Guide.md` for migration patterns and examples.

### Secret Aspects

When a secret value is applied to a widget, it gains a "secret aspect":

```lua
-- Check if a FontString has secret text
if fontString:HasSecretAspect("Text") then
    -- The text was set from a secret value
    -- Calling GetText() will return nil or empty
end

-- Check if a StatusBar has secret values
if statusBar:HasAnySecretAspect() then
    -- GetValue() may return unexpected results
end

-- Check if any object has secrets
if frame:HasSecretValues() then
    -- Some child elements contain secret data
end
```

### Secret Anchoring

When a frame's position depends on secret values, it becomes "secret anchored":

```lua
-- Check if anchoring involves secrets
if region:IsAnchoringSecret() then
    -- Position cannot be reliably determined
end
```

### Preventing Secret Values

Some UI elements should never display secrets:

```lua
-- Prevent secrets from being set on this element
frame:SetPreventSecretValues(true)

-- Check if prevention is enabled
local isPreventing = frame:IsPreventingSecretValues()

-- Reset to defaults (clears secret aspects)
frame:SetToDefaults()
```

### Boolean Display Methods

For conditional UI based on secret values:

```lua
-- Show/hide based on a boolean that may be secret
region:SetAlphaFromBoolean(shouldBeVisible)

-- Color based on a boolean that may be secret
region:SetVertexColorFromBoolean(isActive)
```

### Curve Objects (12.0.0)

Curve objects allow animated values that may contain secrets:

```lua
-- Create a curve for value interpolation
local curve = C_CurveUtil.CreateCurve()
curve:AddControlPoint(0, 0)      -- At input 0, output 0
curve:AddControlPoint(0.5, 0.8)  -- At input 0.5, output 0.8
curve:AddControlPoint(1, 1)      -- At input 1, output 1

-- Use with StatusBar for smooth transitions
statusBar:SetMinMaxValues(curve)
```

### ColorCurve Objects (12.0.0)

Color curves allow color transitions with secret support:

```lua
-- Create a color curve
local colorCurve = C_CurveUtil.CreateColorCurve()
colorCurve:AddColorStop(0, 1, 0, 0, 1)    -- Red at start
colorCurve:AddColorStop(0.5, 1, 1, 0, 1)  -- Yellow at middle
colorCurve:AddColorStop(1, 0, 1, 0, 1)    -- Green at end

-- Apply to texture or other color-supporting widgets
```

### Duration Objects (12.0.0)

Duration objects encapsulate time values that may be secret:

```lua
-- Create a duration object
local duration = C_DurationUtil.CreateDuration()
duration:SetStartTime(GetTime())
duration:SetDuration(10)  -- 10 seconds

-- Use with Cooldown frame
cooldown:SetCooldownFromDurationObject(duration)

-- Use with StatusBar timer
statusBar:SetTimerDuration(duration)
```

### Color Overrides (11.1.5)

Item quality colors can now be user-configured:

```lua
-- PREFERRED: Use ColorManager for item quality colors
local color = ColorManager.GetColorDataForItemQuality(qualityID)

-- The ITEM_QUALITY_COLORS table may not reflect user preferences
-- Using the markup is now preferred:
local text = "|cnIQ" .. quality .. ":" .. itemName .. "|r"
-- Example: "|cnIQ4:Epic Sword|r" for epic quality text

-- FontString callback for color updates
function MyFontStringMixin:OnColorsUpdated()
    -- Called when ITEM_QUALITY_COLORS or other color settings change
    self:RefreshColors()
end
```

### Font Scaling (11.2.0)

New accessibility features for font scaling:

```lua
-- User font scale is controlled by the userFontScale CVar
-- Check if a font can be user-scaled via FontScriptInfo structure

-- Get and set font height directly
local height = fontString:GetFontHeight()
fontString:SetFontHeight(16)  -- Set to 16 pixels

-- FontScriptInfo now includes canBeUserScaled field
local fontInfo = fontString:GetFontObject():GetFontInfo()
if fontInfo.canBeUserScaled then
    -- This font will scale with user preferences
end
```

---

## Common Blizzard Templates

### Layout Frame Templates

Modern WoW uses layout frames for automatic positioning:

**Vertical Layout:**
```xml
<Frame inherits="VerticalLayoutFrame">
    <KeyValues>
        <KeyValue key="spacing" value="5" type="number"/>
        <KeyValue key="align" value="center" type="string"/>
    </KeyValues>
    <Frames>
        <Button>
            <KeyValues>
                <KeyValue key="layoutIndex" value="1" type="number"/>
            </KeyValues>
        </Button>
        <Button>
            <KeyValues>
                <KeyValue key="layoutIndex" value="2" type="number"/>
                <KeyValue key="topPadding" value="10" type="number"/>
            </KeyValues>
        </Button>
    </Frames>
</Frame>
```

**Grid Layout:**
```xml
<Frame inherits="GridLayoutFrame">
    <KeyValues>
        <KeyValue key="stride" value="4" type="number"/>
        <KeyValue key="childXPadding" value="5" type="number"/>
        <KeyValue key="childYPadding" value="5" type="number"/>
        <KeyValue key="isHorizontal" value="true" type="boolean"/>
    </KeyValues>
</Frame>
```

**Resize Layout:**
```xml
<Frame inherits="ResizeLayoutFrame">
    <!-- Automatically resizes to fit children -->
    <Frames>
        <Button/>
        <Button/>
        <Frame>
            <KeyValues>
                <!-- Exclude from layout calculations -->
                <KeyValue key="ignoreInLayout" value="true" type="boolean"/>
            </KeyValues>
        </Frame>
    </Frames>
</Frame>
```

**Layout KeyValues:**
- `layoutIndex` - Order in layout (required for VerticalLayout)
- `expand` - Fill available space (boolean)
- `topPadding`, `bottomPadding`, `leftPadding`, `rightPadding` - Spacing
- `align` - "center" or "right" (default left)
- `ignoreInLayout` - Exclude from layout
- `includeAsLayoutChildWhenHidden` - Layout hidden frames

**Source:** `Blizzard_SharedXML\LayoutFrame.xml`

### ScrollBox (Modern Scrolling)

Replaced HybridScrollFrame in recent versions:

```lua
local scrollBox = CreateFrame("Frame", nil, parent, "WowScrollBoxList");
local scrollBar = CreateFrame("EventFrame", nil, parent, "MinimalScrollBar");

scrollBox:SetPoint("TOPLEFT", 10, -10);
scrollBox:SetPoint("BOTTOMRIGHT", scrollBar, "BOTTOMLEFT", -5, 10);
scrollBar:SetPoint("TOPRIGHT", -10, -10);
scrollBar:SetPoint("BOTTOMRIGHT", -10, 10);

local view = CreateScrollBoxListLinearView();
view:SetElementInitializer("MyButtonTemplate", function(button, elementData)
    button:SetText(elementData.name);
    button.data = elementData;
end);

ScrollUtil.InitScrollBoxListWithScrollBar(scrollBox, scrollBar, view);

local dataProvider = CreateDataProvider();
dataProvider:Insert({name = "Item 1"});
dataProvider:Insert({name = "Item 2"});
scrollBox:SetDataProvider(dataProvider);
```

**Source:** `Blizzard_SharedXML\Shared\Scroll\ScrollBox.lua`

### Hybrid Scroll Frame (Legacy)

Older scroll system still widely used:

```xml
<ScrollFrame name="MyScrollFrame" inherits="HybridScrollFrameTemplate">
    <Anchors>
        <Anchor point="TOPLEFT" x="10" y="-10"/>
        <Anchor point="BOTTOMRIGHT" x="-30" y="10"/>
    </Anchors>
    <Frames>
        <Slider name="$parentScrollBar" inherits="HybridScrollBarTemplate">
            <Anchors>
                <Anchor point="TOPLEFT" relativePoint="TOPRIGHT" x="0" y="-17"/>
                <Anchor point="BOTTOMLEFT" relativePoint="BOTTOMRIGHT" x="0" y="12"/>
            </Anchors>
        </Slider>
    </Frames>
</ScrollFrame>
```

```lua
local function buttonInit(button, elementData)
    button:SetText(elementData.name);
end

HybridScrollFrame_SetDoNotHideScrollBar(MyScrollFrame, true);
HybridScrollFrame_CreateButtons(MyScrollFrame, "MyButtonTemplate", 0, -1, "TOPLEFT", "TOPLEFT", 0, -1, "TOP", "BOTTOM");

local scrollData = {};
for i = 1, 100 do
    table.insert(scrollData, {name = "Item " .. i});
end

local scrollOffset = HybridScrollFrame_GetOffset(MyScrollFrame);
HybridScrollFrame_Update(MyScrollFrame, #scrollData * 20, MyScrollFrame:GetHeight());

for i = 1, #MyScrollFrame.buttons do
    local button = MyScrollFrame.buttons[i];
    local dataIndex = i + scrollOffset;
    if dataIndex <= #scrollData then
        buttonInit(button, scrollData[dataIndex]);
        button:Show();
    else
        button:Hide();
    end
end
```

**Source:** `Blizzard_SharedXML\HybridScrollFrame.xml`, `Blizzard_SharedXML\HybridScrollFrame.lua`

### StaticPopup (Updated 11.2.0)

**Important Changes:**
- `StaticPopup_DisplayedFrames` table removed in 11.2.0
- `STATICPOPUP_NUMDIALOGS` constant removed in 11.2.0
- Use `StaticPopup_ForEachShownDialog()` instead for iterating dialogs

**Old Pattern (Pre-11.2.0):**
```lua
-- DEPRECATED - Do not use
for i = 1, STATICPOPUP_NUMDIALOGS do
    local dialog = _G["StaticPopup" .. i]
    if dialog and dialog:IsShown() then
        -- process dialog
    end
end
```

**New Pattern (11.2.0+):**
```lua
-- Iterate all shown dialogs
StaticPopup_ForEachShownDialog(function(dialog, data)
    -- dialog is the frame
    -- data is the popup data
    print("Dialog shown:", dialog.which)
end)

-- Check if a specific dialog is shown
local isShown = StaticPopup_Visible("CONFIRM_DELETE_ITEM")
```

### Backdrop and NineSlice

**Legacy Backdrop:**
```lua
local backdrop = {
    bgFile = "Interface\\DialogFrame\\UI-DialogBox-Background",
    edgeFile = "Interface\\DialogFrame\\UI-DialogBox-Border",
    tile = true,
    tileEdge = true,
    tileSize = 32,
    edgeSize = 32,
    insets = {left = 11, right = 12, top = 12, bottom = 11},
};

frame:SetBackdrop(backdrop);
frame:SetBackdropColor(0.1, 0.1, 0.1, 0.5);
frame:SetBackdropBorderColor(1, 1, 1, 1);
```

**Modern NineSlice:**
```lua
local layout = {
    TopLeftCorner = {atlas = "UI-Frame-TopLeft"},
    TopEdge = {atlas = "UI-Frame-Top", tileHorizontal = true},
    TopRightCorner = {atlas = "UI-Frame-TopRight"},
    LeftEdge = {atlas = "UI-Frame-Left", tileVertical = true},
    Center = {atlas = "UI-Frame-Center", tileHorizontal = true, tileVertical = true},
    RightEdge = {atlas = "UI-Frame-Right", tileVertical = true},
    BottomLeftCorner = {atlas = "UI-Frame-BottomLeft"},
    BottomEdge = {atlas = "UI-Frame-Bottom", tileHorizontal = true},
    BottomRightCorner = {atlas = "UI-Frame-BottomRight"},
    mirrorLayout = true,
};

NineSliceUtil.ApplyLayout(frame, layout);
```

**Source:** `Blizzard_SharedXML\NineSlice.lua`, `Blizzard_SharedXML\Backdrop.lua`

### Common Button Templates

| Template | Description | Inherits From |
|----------|-------------|---------------|
| `UIPanelButtonTemplate` | Standard UI button | - |
| `UIPanelCloseButton` | Red X close button | - |
| `UIPanelSquareButton` | Square icon button | - |
| `GameMenuButtonTemplate` | Game menu style | `UIPanelButtonTemplate` |
| `MagicButtonTemplate` | Stylized action button | - |
| `SecureActionButtonTemplate` | Combat-safe button | - |
| `SecureHandlerClickTemplate` | Click handler (combat) | - |

**Source:** `Blizzard_SharedXML\Mainline\SharedUIPanelTemplates.xml`

---

## Frame Pooling and Object Reuse

Frame pooling is critical for performance when creating many short-lived frames.

### Frame Factory Pattern

```lua
-- Create factory
local factory = CreateFrameFactory();

-- Acquire frame from pool or create new
local frame, isNew = factory:Create(parent, "Button", resetterFunc);

if isNew then
    -- Initialize new frame
    frame:SetSize(100, 30);
end

-- Use frame...
frame:SetText("Hello");
frame:Show();

-- Release back to pool
factory:Release(frame);

-- Release all frames
factory:ReleaseAll();
```

**Custom Resetter Function:**
```lua
local function ResetButton(pool, button)
    button:Hide();
    button:ClearAllPoints();
    button:SetText("");
    button:SetEnabled(true);
    button.data = nil;
end

local factory = CreateFrameFactory();
factory:SetResetterFunction(ResetButton);
```

**Source:** `Blizzard_SharedXML\Shared\Scroll\ScrollBoxListView.lua`

### Frame Accessor Pattern

Temporary accessors link pooled frames to their data:

```lua
function ScrollBoxListViewMixin:AssignAccessors(frame, elementData)
    local view = self;

    -- Get underlying data
    frame.GetData = function(self)
        return view:TranslateElementDataToUnderlyingData(elementData);
    end;

    -- Get element data
    frame.GetElementData = function(self)
        return elementData;
    end;

    -- Get index
    frame.GetElementDataIndex = function(self)
        return view:FindElementDataIndex(elementData);
    end;

    -- Match element data
    frame.ElementDataMatches = function(self, elementData)
        return self:GetElementData() == elementData;
    end;
end

function ScrollBoxListViewMixin:UnassignAccessors(frame)
    frame.GetElementData = nil;
    frame.GetData = nil;
    frame.ElementDataMatches = nil;
    frame.GetOrderIndex = nil;
end
```

**Pattern:**
- Assign accessors when frame is acquired
- Accessors close over view and data
- Clear accessors before releasing to pool
- Prevents memory leaks from closures

**Source:** `Blizzard_SharedXML\Shared\Scroll\ScrollBoxListView.lua`

---

## Data Provider Pattern

Data providers separate data from UI:

```lua
DataProviderMixin = CreateFromMixins(CallbackRegistryMixin);

DataProviderMixin:GenerateCallbackEvents({
    "OnSizeChanged",
    "OnInsert",
    "OnRemove",
    "OnSort",
    "OnMove",
});

function DataProviderMixin:Init(tbl)
    CallbackRegistryMixin.OnLoad(self);
    self.collection = {};

    if tbl then
        self:InsertTable(tbl);
    end
end

function DataProviderMixin:Insert(...)
    local count = select("#", ...);
    for index = 1, count do
        local value = select(index, ...);
        self:InsertInternal(value);
    end

    if count > 0 then
        self:TriggerEvent(DataProviderMixin.Event.OnSizeChanged);
    end

    self:Sort();
end

function DataProviderMixin:Remove(...)
    -- Remove elements...
    self:TriggerEvent(DataProviderMixin.Event.OnSizeChanged);
end

function DataProviderMixin:Enumerate(indexBegin, indexEnd)
    return CreateTableEnumerator(self.collection, indexBegin, indexEnd);
end

function DataProviderMixin:SetSortComparator(sortComparator, skipSort)
    self.sortComparator = sortComparator;
    if not skipSort then
        self:Sort();
    end
end
```

**Usage:**
```lua
local dataProvider = CreateDataProvider();

-- Listen for changes
dataProvider:RegisterCallback(DataProviderMixin.Event.OnSizeChanged, function()
    print("Data changed!");
end);

-- Insert data
dataProvider:Insert({name = "Item 1", value = 10});
dataProvider:Insert({name = "Item 2", value = 5});
dataProvider:Insert({name = "Item 3", value = 15});

-- Set sort
dataProvider:SetSortComparator(function(a, b)
    return a.value < b.value;
end);

-- Enumerate
for index, data in dataProvider:Enumerate() do
    print(index, data.name, data.value);
end
```

**Source:** `Blizzard_SharedXML\DataProvider.lua`

---

## Practical Examples

### Complete Addon Frame Example

**MyAddon.toc:**
```
## Interface: 120000
## Title: My Addon
## Author: Your Name
## Version: 1.0.0

MyAddonFrame.xml
MyAddonFrame.lua
```

**MyAddonFrame.xml:**
```xml
<Ui xmlns="http://www.blizzard.com/wow/ui/"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://www.blizzard.com/wow/ui/
                        https://raw.githubusercontent.com/Gethe/wow-ui-source/live/Interface/AddOns/Blizzard_SharedXML/UI.xsd">

    <!-- Item Button Template -->
    <Button name="MyAddonItemButtonTemplate" virtual="true" mixin="MyAddonItemButtonMixin">
        <Size x="200" y="30"/>
        <Layers>
            <Layer level="BACKGROUND">
                <Texture parentKey="Icon">
                    <Size x="24" y="24"/>
                    <Anchors>
                        <Anchor point="LEFT" x="5" y="0"/>
                    </Anchors>
                </Texture>
            </Layer>
            <Layer level="ARTWORK">
                <FontString parentKey="Name" inherits="GameFontNormal">
                    <Anchors>
                        <Anchor point="LEFT" relativeKey="$parent.Icon" relativePoint="RIGHT" x="5" y="0"/>
                    </Anchors>
                </FontString>
            </Layer>
        </Layers>
        <HighlightTexture alpha="0.3" file="Interface\Buttons\UI-Common-MouseHilight"/>
        <Scripts>
            <OnLoad method="OnLoad"/>
            <OnClick method="OnClick"/>
            <OnEnter method="OnEnter"/>
            <OnLeave method="OnLeave"/>
        </Scripts>
    </Button>

    <!-- Main Frame -->
    <Frame name="MyAddonFrame" parent="UIParent" mixin="MyAddonFrameMixin" enableMouse="true" movable="true">
        <Size x="300" y="400"/>
        <Anchors>
            <Anchor point="CENTER"/>
        </Anchors>

        <Layers>
            <Layer level="BACKGROUND">
                <Texture setAllPoints="true">
                    <Color r="0" g="0" b="0" a="0.8"/>
                </Texture>
            </Layer>
            <Layer level="ARTWORK">
                <FontString parentKey="Title" inherits="GameFontNormalLarge">
                    <Anchors>
                        <Anchor point="TOP" x="0" y="-10"/>
                    </Anchors>
                </FontString>
            </Layer>
        </Layers>

        <Frames>
            <!-- Scroll Frame -->
            <ScrollFrame name="$parentScrollFrame" parentKey="ScrollFrame" inherits="HybridScrollFrameTemplate">
                <Anchors>
                    <Anchor point="TOPLEFT" x="10" y="-40"/>
                    <Anchor point="BOTTOMRIGHT" x="-30" y="40"/>
                </Anchors>
                <Frames>
                    <Slider name="$parentScrollBar" inherits="HybridScrollBarTemplate">
                        <Anchors>
                            <Anchor point="TOPLEFT" relativePoint="TOPRIGHT" x="0" y="-17"/>
                            <Anchor point="BOTTOMLEFT" relativePoint="BOTTOMRIGHT" x="0" y="12"/>
                        </Anchors>
                    </Slider>
                </Frames>
            </ScrollFrame>

            <!-- Close Button -->
            <Button name="$parentCloseButton" parentKey="CloseButton" inherits="UIPanelCloseButton">
                <Anchors>
                    <Anchor point="TOPRIGHT" x="-5" y="-5"/>
                </Anchors>
            </Button>
        </Frames>

        <Scripts>
            <OnLoad method="OnLoad"/>
            <OnShow method="OnShow"/>
            <OnHide method="OnHide"/>
            <OnDragStart>self:StartMoving()</OnDragStart>
            <OnDragStop>self:StopMovingOrSizing()</OnDragStop>
        </Scripts>
    </Frame>
</Ui>
```

**MyAddonFrame.lua:**
```lua
-- Item Button Mixin
MyAddonItemButtonMixin = {};

function MyAddonItemButtonMixin:OnLoad()
    -- Initialize button
end

function MyAddonItemButtonMixin:SetData(data)
    self.data = data;
    self.Icon:SetTexture(data.icon);
    self.Name:SetText(data.name);
end

function MyAddonItemButtonMixin:OnClick()
    print("Clicked:", self.data.name);
end

function MyAddonItemButtonMixin:OnEnter()
    GameTooltip:SetOwner(self, "ANCHOR_RIGHT");
    GameTooltip:SetText(self.data.name);
    GameTooltip:AddLine(self.data.description, 1, 1, 1, true);
    GameTooltip:Show();
end

function MyAddonItemButtonMixin:OnLeave()
    GameTooltip:Hide();
end

-- Main Frame Mixin
MyAddonFrameMixin = {};

function MyAddonFrameMixin:OnLoad()
    self:RegisterForDrag("LeftButton");
    self.Title:SetText("My Addon");

    -- Setup scroll frame
    HybridScrollFrame_SetDoNotHideScrollBar(self.ScrollFrame, true);
    HybridScrollFrame_CreateButtons(
        self.ScrollFrame,
        "MyAddonItemButtonTemplate",
        0, -1,  -- spacing
        "TOPLEFT", "TOPLEFT",
        0, -1,
        "TOP", "BOTTOM"
    );

    self.data = {};
end

function MyAddonFrameMixin:OnShow()
    self:Refresh();
end

function MyAddonFrameMixin:OnHide()
    -- Cleanup
end

function MyAddonFrameMixin:SetData(data)
    self.data = data;
    self:Refresh();
end

function MyAddonFrameMixin:Refresh()
    local scrollFrame = self.ScrollFrame;
    local offset = HybridScrollFrame_GetOffset(scrollFrame);
    local buttons = scrollFrame.buttons;

    local itemHeight = 30;
    local totalHeight = #self.data * itemHeight;

    HybridScrollFrame_Update(scrollFrame, totalHeight, scrollFrame:GetHeight());

    for i = 1, #buttons do
        local button = buttons[i];
        local dataIndex = i + offset;

        if dataIndex <= #self.data then
            button:SetData(self.data[dataIndex]);
            button:Show();
        else
            button:Hide();
        end
    end
end

-- Example usage
function MyAddon_ShowFrame()
    local data = {
        {name = "Item 1", icon = "Interface\\Icons\\INV_Misc_QuestionMark", description = "First item"},
        {name = "Item 2", icon = "Interface\\Icons\\INV_Misc_QuestionMark", description = "Second item"},
        {name = "Item 3", icon = "Interface\\Icons\\INV_Misc_QuestionMark", description = "Third item"},
    };

    MyAddonFrame:SetData(data);
    MyAddonFrame:Show();
end
```

---

## Best Practices

### 1. Use 3-Tier Template Pattern
Separate art, code, and script binding for reusability:
```xml
<Frame name="MyArtTemplate" virtual="true">...</Frame>
<Frame name="MyCodeTemplate" inherits="MyArtTemplate" mixin="MyMixin" virtual="true"/>
<Frame name="MyFullTemplate" inherits="MyCodeTemplate" virtual="true">
    <Scripts>...</Scripts>
</Frame>
```

### 2. Prefer parentKey over Named Frames
```xml
<!-- Good -->
<Texture parentKey="Icon"/>
<!-- Access: frame.Icon -->

<!-- Avoid -->
<Texture name="$parentIcon"/>
<!-- Access: _G[frame:GetName() .. "Icon"] -->
```

### 3. Use Mixins for Composition
```lua
-- Good: Compose behavior
MyFrameMixin = CreateFromMixins(BaseFrameMixin, EventListenerMixin);

-- Avoid: Global functions
function MyFrame_OnLoad(self)
    -- Pollutes global namespace
end
```

### 4. Lazy Script Assignment
Only set scripts when needed:
```lua
function MyMixin:MarkDirty()
    self.dirty = true;
    self:SetScript("OnUpdate", self.OnUpdate);  -- Set only when dirty
end

function MyMixin:OnUpdate()
    if self:IsDirty() then
        self:Layout();
        self:SetScript("OnUpdate", nil);  -- Remove when clean
    end
end
```

### 5. Pool Frames for Performance
```lua
-- Create pool
local pool = CreateFramePool("Button", parent, "MyButtonTemplate");

-- Acquire from pool
local button = pool:Acquire();
button:SetText("Hello");
button:Show();

-- Release to pool
pool:Release(button);

-- Release all
pool:ReleaseAll();
```

### 6. Use Layout Frames
Let the system handle positioning:
```xml
<Frame inherits="VerticalLayoutFrame">
    <KeyValues>
        <KeyValue key="spacing" value="5" type="number"/>
    </KeyValues>
    <Frames>
        <Button><KeyValues><KeyValue key="layoutIndex" value="1" type="number"/></KeyValues></Button>
        <Button><KeyValues><KeyValue key="layoutIndex" value="2" type="number"/></KeyValues></Button>
    </Frames>
</Frame>
```

### 7. Separate Data from UI
Use data providers:
```lua
local dataProvider = CreateDataProvider();
dataProvider:Insert(item1, item2, item3);
scrollBox:SetDataProvider(dataProvider);
```

### 8. Layer Properly
Respect draw order:
```
BACKGROUND → BORDER → ARTWORK → OVERLAY → HIGHLIGHT
```

### 9. Anchor Wisely
Use multiple anchors for flexible sizing:
```xml
<Anchors>
    <Anchor point="TOPLEFT" x="10" y="-10"/>
    <Anchor point="BOTTOMRIGHT" x="-10" y="10"/>
</Anchors>
```

### 10. Virtual Templates
Mark templates as virtual to prevent instantiation:
```xml
<Button name="MyTemplate" virtual="true">
    <!-- Only used for inheritance -->
</Button>
```

---

<!-- CLAUDE_SKIP_START -->
<!-- CLAUDE_SKIP_START -->
## Reference Files

| Topic | File Path |
|-------|-----------|
| Mixin System | `Blizzard_SharedXML\Shared\Frame\EventFrame.lua` |
| Layout Frames | `Blizzard_SharedXML\LayoutFrame.lua` |
| ScrollBox (Modern) | `Blizzard_SharedXML\Shared\Scroll\ScrollBox.lua` |
| Data Providers | `Blizzard_SharedXML\DataProvider.lua` |
| Frame Pooling | `Blizzard_SharedXML\Shared\Scroll\ScrollBoxListView.lua` |
| NineSlice | `Blizzard_SharedXML\NineSlice.lua` |
| Backdrop | `Blizzard_SharedXML\Backdrop.lua` |
| Action Button | `Blizzard_ActionBar\Mainline\ActionButtonTemplate.xml` |
| Buff Frame | `Blizzard_BuffFrame\BuffFrameTemplates.xml` |
| UI Panel Templates | `Blizzard_SharedXML\Mainline\SharedUIPanelTemplates.lua` |
| Hybrid Scroll | `Blizzard_SharedXML\HybridScrollFrame.xml` |

All paths relative to: `D:\Games\World of Warcraft\_retail_\Interface\+wow-ui-source+ (11.2.7)\Interface\AddOns\`

---

**Version:** 2.0 - Updated for WoW 12.0.0 (Midnight)
**Last Updated:** 2026-01-20

<!-- CLAUDE_SKIP_END -->

<!-- CLAUDE_SKIP_END -->
