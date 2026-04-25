# WoW Cooldown Viewer / Cooldown Manager System Guide

## Table of Contents

1. [Overview](#overview)
2. [C_CooldownViewer API](#c_cooldownviewer-api)
3. [Events](#events)
4. [Enums](#enums)
5. [Architecture](#architecture)
6. [Mixin Hierarchy](#mixin-hierarchy)
7. [Spell ID Resolution](#spell-id-resolution)
8. [Cooldown Tracking](#cooldown-tracking)
9. [Aura Tracking](#aura-tracking)
10. [Alert System](#alert-system)
11. [Layout System](#layout-system)
12. [Serialization](#serialization)
13. [Settings UI](#settings-ui)
14. [Viewer Frame Templates](#viewer-frame-templates)
15. [Pandemic Alert Animation](#pandemic-alert-animation)
16. [CooldownFrame Widget API](#cooldownframe-widget-api)
17. [Addon Integration Patterns](#addon-integration-patterns)
18. [Key Blizzard Source Files Reference](#key-blizzard-source-files-reference)
19. [Taint Avoidance for CooldownViewer Frames](#taint-avoidance-for-cooldownviewer-frames)

---

## Overview

The **Cooldown Viewer** (user-facing name) or **Cooldown Manager** (internal name / `CDM`) is a built-in HUD system introduced in **WoW 12.0.0 (Midnight)**. It is retail-only (`AllowLoadGameType: standard`) and depends on `Blizzard_EditMode`.

The system provides four viewer frames, each displaying a different category of player abilities:

| Viewer | Category | Purpose |
|--------|----------|---------|
| **Essential Cooldowns** | `Essential` | Major cooldowns (defensive/offensive CDs) |
| **Utility Cooldowns** | `Utility` | Minor utility spells |
| **Tracked Buff Icons** | `TrackedBuff` | Self-buff/debuff tracking as icons |
| **Tracked Buff Bars** | `TrackedBar` | Self-buff/debuff tracking as status bars |

All four viewers are integrated with the Edit Mode system, allowing players to reposition, resize, and configure them. The first three are managed by `UIParentBottomManagedFrameTemplate` for automatic bottom-bar layout; the Buff Bar viewer is not.

### CVars

| CVar | Type | Description |
|-------|------|-------------|
| `cooldownViewerEnabled` | bool | Master toggle for all Cooldown Viewer frames |
| `cooldownViewerShowUnlearned` | bool | Show unlearned spells in the Settings UI |

---

## C_CooldownViewer API

All functions in the `C_CooldownViewer` namespace. Functions annotated with `SecretArguments = "AllowedWhenUntainted"` will reject calls from tainted (addon) execution contexts. See [Secret Safe APIs](12a_Secret_Safe_APIs.md) for details.

| Function | Parameters | Returns | Secret |
|----------|-----------|---------|--------|
| GetCooldownViewerCategorySet | category (CooldownViewerCategory), allowUnlearned (bool, default false) | cooldownIDs (table) | Untainted |
| GetCooldownViewerCooldownInfo | cooldownID (number) | cooldownInfo (CooldownViewerCooldown) or nothing | Untainted |
| GetLayoutData | *(none)* | data (string) | *(none)* |
| SetLayoutData | data (string) | *(none)* | Untainted |
| GetValidAlertTypes | cooldownID (number) | validAlertTypes (table) | Untainted |
| IsCooldownViewerAvailable | *(none)* | isAvailable (bool), failureReason (string) | *(none)* |

**Notes:**
- **Secret = Untainted** means `SecretArguments = "AllowedWhenUntainted"` — these functions reject calls from tainted (addon) execution contexts.
- `GetCooldownViewerCategorySet`: when `allowUnlearned` is true, includes spells the player has not yet learned.
- `GetCooldownViewerCooldownInfo`: may return nothing if the cooldownID is invalid.
- `GetLayoutData` / `SetLayoutData`: read/write the raw serialized layout string from the account data store.
- `GetValidAlertTypes`: returns which alert event types are valid for a cooldown (e.g., a spell without charges won't include ChargeGained).

### CooldownViewerCooldown Structure

Returned by `GetCooldownViewerCooldownInfo`:

| Field | Type | Nilable | Description |
|-------|------|---------|-------------|
| `cooldownID` | number | No | Unique identifier for this cooldown entry |
| `spellID` | number | No | The base spell ID |
| `overrideSpellID` | number | Yes | Current override spell (talent morphs, etc.) |
| `overrideTooltipSpellID` | number | Yes | Tooltip-specific override (linked spell acting as display) |
| `linkedSpellIDs` | table\<number\> | No | Up to 4 related spell IDs (e.g., DoT applied by the ability) |
| `selfAura` | bool | No | Whether this cooldown applies a self-buff |
| `hasAura` | bool | No | Whether this cooldown has any associated aura |
| `charges` | bool | No | Whether this spell uses the charge system |
| `isKnown` | bool | No | Whether the player currently knows this spell |
| `flags` | CooldownSetSpellFlags | No | Bitfield flags (HideAura, HideByDefault) |
| `category` | CooldownViewerCategory | No | Default category assignment |

---

## Events

| Event | Payload | Description |
|-------|---------|-------------|
| `COOLDOWN_VIEWER_DATA_LOADED` | *(none)* | Fires synchronously when cooldown viewer data is loaded and ready. |
| `COOLDOWN_VIEWER_SPELL_OVERRIDE_UPDATED` | `baseSpellID` (number), `overrideSpellID` (number or nil) | Fires synchronously when a spell override is added or removed. A nil `overrideSpellID` means the override was removed from the base spell. |
| `COOLDOWN_VIEWER_TABLE_HOTFIXED` | *(none)* | Fires synchronously when the server hotfixes the cooldown data tables. Triggers a full data refresh. |

The system also uses the `EventRegistry` for internal communication:

| EventRegistry Callback | Argument | Description |
|------------------------|----------|-------------|
| `CooldownViewerSettings.OnShow` | settings frame | Settings window opened |
| `CooldownViewerSettings.OnHide` | settings frame | Settings window closed |
| `CooldownViewerSettings.OnDataChanged` | *(none)* | Layout data changed (profile switch, setting change) |
| `CooldownViewerSettings.OnPendingChanges` | *(none)* | Layout manager has unsaved changes |
| `CooldownViewerSettings.OnEnterItem` | *(none)* | Mouse entered a settings item (drag-and-drop) |

---

## Enums

### CooldownViewerCategory

| Value | Name | Description |
|-------|------|-------------|
| 0 | `Essential` | Major cooldowns viewer |
| 1 | `Utility` | Utility cooldowns viewer |
| 2 | `TrackedBuff` | Tracked buff icons viewer |
| 3 | `TrackedBar` | Tracked buff bars viewer |
| -1 | `HiddenSpell` | Pseudo-category: disabled spells (not in any viewer) |
| -2 | `HiddenAura` | Pseudo-category: disabled auras (not in any viewer) |

`HiddenSpell` and `HiddenAura` are injected at runtime in `CooldownViewerSettingsConstants.lua` and are NOT part of the C++ enum. They exist so that disabled cooldowns can use the same category system as visible ones.

### CooldownViewerAlertEventType

| Value | Name | Description |
|-------|------|-------------|
| 1 | `Available` | Cooldown finished (non-GCD, duration > 0.75s) |
| 2 | `PandemicTime` | Pandemic refresh window started (target debuff) |
| 3 | `OnCooldown` | Ability just went on cooldown |
| 4 | `ChargeGained` | A charge was gained |

### CooldownViewerAlertType

| Value | Name | Description |
|-------|------|-------------|
| 1 | `Sound` | Play a sound effect or TTS |
| 2 | `Visual` | Show a visual alert animation |

### CooldownSetSpellFlags

Bitfield flags on `CooldownViewerCooldown.flags`:

| Value | Name | Description |
|-------|------|-------------|
| 1 | `HideAura` | Do not use aura data for cooldown display |
| 2 | `HideByDefault` | Start in the hidden/disabled category |

### CooldownSetLinkedSpellFlags

| Value | Name | Description |
|-------|------|-------------|
| 1 | `UseAsTooltip` | Use this linked spell as the tooltip override |

### CooldownViewerAddAlertStatus

| Value | Name | Description |
|-------|------|-------------|
| 0 | `Success` | Alert added successfully |
| 1 | `InvalidAlertType` | Alert type is not recognized |
| 2 | `InvalidEventType` | Alert event type is out of range |
| 3 | `DuplicateAlert` | An identical alert already exists |

### CooldownLayoutType

| Value | Name | Description |
|-------|------|-------------|
| 1 | `Character` | Per-character layout |
| 2 | `Account` | Account-wide layout (not yet implemented) |

### CooldownLayoutStatus

| Value | Name | Description |
|-------|------|-------------|
| 0 | `Success` | Operation succeeded |
| 1 | `InvalidLayoutName` | Layout name is invalid |
| 2 | `TooManyLayouts` | Maximum layout count reached |
| 3 | `AttemptToModifyDefaultLayoutWouldCreateTooManyLayouts` | Cannot auto-create layout |
| 4 | `TooManyAlerts` | Maximum alert count reached |
| 5 | `InvalidOrderChange` | Reorder operation is invalid |
| 6 | `NoValidAlerts` | No valid alert types for this cooldown |

### CooldownLayoutAction

| Value | Name | Description |
|-------|------|-------------|
| 0 | `ChangeOrder` | Reorder cooldowns |
| 1 | `ChangeCategory` | Move cooldown to different category |
| 2 | `AddLayout` | Create a new layout |
| 3 | `AddAlert` | Add an alert to a cooldown |

### CooldownViewerVisibleSetting

| Value | Name | Description |
|-------|------|-------------|
| 0 | `Always` | Always visible |
| 1 | `InCombat` | Only visible during combat |
| 2 | `Hidden` | Never visible (unless in Edit Mode) |

### CooldownViewerBarContent

| Value | Name | Description |
|-------|------|-------------|
| 0 | `IconAndName` | Show icon and spell name on buff bars |
| 1 | `IconOnly` | Show only the icon |
| 2 | `NameOnly` | Show only the spell name |

### CooldownViewerOrientation

| Value | Name | Description |
|-------|------|-------------|
| 0 | `Horizontal` | Icons arranged horizontally |
| 1 | `Vertical` | Icons arranged vertically |

### CooldownViewerIconDirection

| Value | Name | Description |
|-------|------|-------------|
| 0 | `Left` | Icons grow leftward (or upward when vertical) |
| 1 | `Right` | Icons grow rightward (or downward when vertical) |

### CDMLayoutMode

Lua-only enum controlling whether layout operations can create new layouts:

| Value | Name | Description |
|-------|------|-------------|
| `false` | `AccessOnly` | Read-only access, do not create layouts |
| `true` | `AllowCreate` | Allow creating layouts if none exists |

### UI Constants

| Constant | Value | Description |
|----------|-------|-------------|
| `COOLDOWN_VIEWER_LINKED_SPELLS_SIZE` | 4 | Maximum linked spells per cooldown |
| `COOLDOWN_VIEWER_CATEGORY_SET_SIZE` | 16 | Maximum cooldowns per category |

---

## Architecture

### File Listing

| File | Lines | Purpose |
|------|-------|---------|
| `Blizzard_CooldownViewer.toc` | 27 | TOC file; load order, dependencies |
| `CooldownViewerSettingsConstants.lua` | 140 | Enums, pseudo-categories, sound/visual enum definitions |
| `CooldownViewerUtil.lua` | 53 | Class+spec tag encoding, utility helpers |
| `CooldownViewerVisualAlert.lua` | 84 | Base visual alert mixin (anchor, target, lifecycle) |
| `CooldownViewerVisualAlertTemplates.lua` | 65 | MarchingAnts and Flash alert mixin implementations |
| `CooldownViewerVisualAlertTemplates.xml` | 88 | XML templates for all 10 visual alert variants |
| `CooldownViewerVisualAlertsManager.lua` | 32 | Pool-based visual alert manager (acquire/release) |
| `CooldownViewerVisualAlertsManager.xml` | 8 | XML frame for the visual alerts manager singleton |
| `CooldownViewerVisualAlertTarget.lua` | 58 | Target mixin for frames that can receive visual alerts |
| `CooldownViewerSoundAlertData.lua` | 87 | Sound alert data (67 sounds across 6 categories + TTS) |
| `CooldownViewerVisualAlertData.lua` | 14 | Visual alert data (10 visual types) |
| `CooldownViewerAlert.lua` | 254 | Alert data structure, play logic, TTS/sound/visual dispatch |
| `CooldownViewerSettingsAlerts.lua` | 220 | Alert editing panel (3 dropdowns, add/edit logic) |
| `CooldownViewerSettingsAlerts.xml` | 88 | XML for the alert editing panel |
| `CooldownViewerSettingsDataStoreSerialization.lua` | 443 | CBOR/Deflate/Base64 serialization, version migration (v1-v4) |
| `CooldownViewerSettingsLayoutManager.lua` | 978 | Layout CRUD, undo, import/export, alert management |
| `CooldownViewerSettingsDataProvider.lua` | 342 | Display data provider, category filtering, reordering |
| `CooldownViewerItemData.lua` | 471 | Item data mixin (spell resolution, aura lookup, tooltip) |
| `CooldownViewerSettingsDialogs.lua` | 58 | Import/export dialog mixins |
| `CooldownViewerSettingsDialogs.xml` | 14 | XML for settings dialogs |
| `CooldownViewerSettings.lua` | 1717 | Full settings UI frame (tabs, search, drag-drop, categories) |
| `CooldownViewerSettings.xml` | 261 | XML for the settings frame (399x609, ButtonFrameTemplate) |
| `CooldownViewer.lua` | 2043 | Core viewer/item mixins, cooldown/aura tracking, event dispatch |
| `PandemicAlertAnimation.xml` | 82 | Pandemic alert animation templates (icon pulse, bar rotation) |
| `CooldownViewer.xml` | 334 | XML for all 4 viewer frames and item templates |

**Total: 27 files, ~7,934 lines**

### Load Order (from .toc)

1. `CooldownViewerSettingsConstants.lua` -- Enums and constants
2. `CooldownViewerUtil.lua` -- Utility functions
3. `CooldownViewerVisualAlert.lua` -- Visual alert base
4. `CooldownViewerVisualAlertTemplates.lua` -- Visual alert implementations
5. `CooldownViewerVisualAlertTemplates.xml` -- Visual alert XML templates
6. `CooldownViewerVisualAlertsManager.lua` -- Alert pool manager
7. `CooldownViewerVisualAlertsManager.xml` -- Manager XML frame
8. `CooldownViewerVisualAlertTarget.lua` -- Alert target mixin
9. `CooldownViewerSoundAlertData.lua` -- Sound data
10. `CooldownViewerVisualAlertData.lua` -- Visual data
11. `CooldownViewerAlert.lua` -- Alert core logic
12. `CooldownViewerSettingsAlerts.lua` -- Alert editing UI
13. `CooldownViewerSettingsAlerts.xml` -- Alert editing XML
14. `CooldownViewerSettingsDataStoreSerialization.lua` -- Serialization
15. `CooldownViewerSettingsLayoutManager.lua` -- Layout management
16. `CooldownViewerSettingsDataProvider.lua` -- Data provider
17. `CooldownViewerItemData.lua` -- Item data mixin
18. `CooldownViewerSettingsDialogs.lua` -- Dialog mixins
19. `CooldownViewerSettingsDialogs.xml` -- Dialog XML
20. `CooldownViewerSettings.lua` -- Settings UI
21. `CooldownViewerSettings.xml` -- Settings XML
22. `CooldownViewer.lua` -- Core viewer logic
23. `PandemicAlertAnimation.xml` -- Pandemic animations
24. `CooldownViewer.xml` -- Viewer frame definitions

### Dependencies

- **Required:** `Blizzard_EditMode` -- provides `EditModeCooldownViewerSystemMixin` and `EditModeCooldownViewerSystemTemplate`
- **Load restriction:** `AllowLoadGameType: standard` (retail only)

---

## Mixin Hierarchy

### Item Mixins (individual cooldown/buff slots)

```
CooldownViewerItemDataMixin           -- Data layer: spell ID resolution, aura lookup, tooltip
  + CooldownViewerVisualAlertTargetMixin -- Can receive visual alerts
  = CooldownViewerItemMixin            -- Display layer: OnUpdate, alerts, pandemic, shown state
    |
    +-- CooldownViewerCooldownItemMixin -- Cooldown-specific: charges, range check, icon color, glow
    |   +-- CooldownViewerEssentialItemMixin  -- (empty, inherits all)
    |   +-- CooldownViewerUtilityItemMixin    -- (empty, inherits all)
    |
    +-- CooldownViewerBuffItemMixin     -- Buff-specific: aura active state, totem support
        +-- CooldownViewerBuffIconItemMixin   -- Icon display: reverse swipe, applications
        +-- CooldownViewerBuffBarItemMixin    -- Bar display: StatusBar, pip, name, duration text
```

### Viewer Mixins (container frames)

```
CooldownViewerMixin                    -- Base: item pool, aura map, layout, visibility
  |
  +-- CooldownViewerCooldownMixin      -- Adds spell activation, range, usable events
  |   + EditModeCooldownViewerSystemMixin
  |   + UIParentManagedFrameMixin
  |   + GridLayoutFrameMixin
  |   = EssentialCooldownViewerMixin
  |
  |   + EditModeCooldownViewerSystemMixin
  |   + UIParentManagedFrameMixin
  |   + GridLayoutFrameMixin
  |   = UtilityCooldownViewerMixin
  |
  +-- CooldownViewerBuffMixin          -- Buff-specific viewer base (no extra events)
      + EditModeCooldownViewerSystemMixin
      + UIParentManagedFrameMixin
      + GridLayoutFrameMixin
      = BuffIconCooldownViewerMixin    -- Single row/column stride
      
      + EditModeCooldownViewerSystemMixin
      + GridLayoutFrameMixin
      = BuffBarCooldownViewerMixin     -- Bar width scaling, bar content mode, NO UIParentManaged
```

---

## Spell ID Resolution

`CooldownViewerItemDataMixin:GetSpellID()` resolves the "active" spell ID using a priority chain. This determines which spell is used for cooldown display, aura lookup, tooltip, and texture.

### Priority Chain (highest to lowest)

| Priority | Source | Field | When Used |
|----------|--------|-------|-----------|
| 1 | **Active Aura** | `self.auraSpellID` | An aura matching any associated spell is currently active |
| 2 | **Linked Spell** | `cooldownInfo.linkedSpellID` | A linked spell has an active aura (set via `UpdateLinkedSpell`) |
| 3 | **Override Tooltip** | `cooldownInfo.overrideTooltipSpellID` | A linked spell with `UseAsTooltip` flag is configured |
| 4 | **Override Spell** | `cooldownInfo.overrideSpellID` | Talent morphs or temporary spell overrides are active |
| 5 | **Base Spell** | `cooldownInfo.spellID` | Default fallback |

### Texture Resolution

`GetSpellTexture()` follows a similar chain but with one key difference: when falling through to the base spell, it calls `C_Spell.GetSpellTexture(baseSpellID)` which internally handles override spell textures. This means the texture chain does not need to explicitly check `overrideSpellID`.

```lua
-- Texture priority:
-- 1. auraSpellID texture
-- 2. linkedSpellID texture
-- 3. overrideTooltipSpellID texture
-- 4. C_Spell.GetSpellTexture(baseSpellID) -- handles overrides internally
-- 5. Fallback (edit mode placeholder icon)
```

---

## Cooldown Tracking

### CacheCooldownValues Priority

`CooldownViewerCooldownItemMixin:CacheCooldownValues()` checks multiple data sources in priority order. Each source can set `wasSetFrom*` flags; later sources skip updating values if a higher-priority source already ran.

| Priority | Source | Method | When Active |
|----------|--------|--------|-------------|
| 1 | **Charges** | `CheckCacheCooldownValuesFromCharges` | Spell has charges AND charge cooldown is active AND charges > 0 |
| 2 | **Spell Cooldown** | `CheckCacheCooldownValuesFromSpellCooldown` | `C_Spell.GetSpellCooldown` returns data AND charges did not claim priority |
| 3 | **Aura** | `CheckCacheCooldownValuesFromAura` | Self-buff or totem is active AND `HideAura` flag is not set |
| 4 | **Edit Mode** | `CheckCacheCooldownValuesFromEditMode` | In Edit Mode AND no real spell data was set |

### GCD Detection

The system uses spell ID **61304** (a dummy GCD spell) to detect whether a spell's cooldown is actually just the GCD:

```lua
local gcdInfo = C_Spell.GetSpellCooldown(61304)
-- If spell's startTime and duration match the GCD info exactly, it's on GCD
local isOnGCD = (spellCooldownInfo.startTime == gcdInfo.startTime)
            and (spellCooldownInfo.duration == gcdInfo.duration)
```

**Simpler alternative (12.0.1+):** `SpellCooldownInfo.isOnGCD` is a `NeverSecret` boolean field on the return from `C_Spell.GetSpellCooldown()`. This eliminates the need to query the dummy spell 61304:

```lua
local spellCooldownInfo = C_Spell.GetSpellCooldown(spellID)
if spellCooldownInfo.isOnGCD then
    -- This is just the GCD, not a real cooldown
end
```

The `Available` alert is suppressed for GCD-only cooldowns and cooldowns shorter than 0.75 seconds.

### Non-Secret Cooldown Fields for Logic

As of 12.0.1, `SpellCooldownInfo.isActive` and `SpellCooldownInfo.isOnGCD` are both `NeverSecret`. Use these for branching logic (e.g., deciding whether to show a swipe, whether to desaturate an icon), reserving DurationObjects only for display. This avoids secret value issues entirely for decision-making code.

### GCD Swipe Suppression

Use `isOnGCD` to distinguish GCD-only swipes from real cooldowns. When the cooldown is GCD-only, suppress the swipe by setting the CooldownFrame's alpha to zero rather than hiding or clearing it:

```lua
if spellCooldownInfo.isOnGCD then
    cooldownFrame:SetAlpha(0)  -- Hide GCD swipe visually
else
    cooldownFrame:SetAlpha(1)  -- Show real cooldown swipe
end
```

### Active Swipe Suppression with Addon-Owned CooldownFrame

When you need independent control over cooldown display (e.g., showing a spell cooldown swipe instead of Blizzard's aura swipe), suppress Blizzard's CooldownFrame with `SetAlpha(0)` and create your own:

```lua
-- Suppress Blizzard's CooldownFrame
child.Cooldown:SetAlpha(0)

-- Create addon-owned CooldownFrame
local addonCD = CreateFrame("Cooldown", nil, child, "CooldownFrameTemplate")
addonCD:SetAllPoints(child.Icon)
addonCD:SetDrawEdge(false)
addonCD:SetDrawBling(false)

-- Copy timer font from Blizzard's countdown FontString
local blizzFont = child.Cooldown:GetCountdownFontString()
if blizzFont then
    addonCD:SetCountdownFont(blizzFont:GetFont())
end

-- Raise charge/stack count frames above the addon CD
local addonCDLevel = addonCD:GetFrameLevel()
if child.Count then
    child.Count:GetParent():SetFrameLevel(addonCDLevel + 4)
end

-- Use DurationObject for secret-safe display
local dur = C_Spell.GetSpellCooldownDuration(spellID)
if dur then
    addonCD:SetCooldownFromDurationObject(dur)
end
```

### Charge Cooldown Duration

`C_Spell.GetSpellChargeDuration(spellID, ignoreGCD)` returns a `DurationObject` for the charge recharge timer. Use this alongside `C_Spell.GetSpellCharges()` when `maxCharges > 1`. For charge-based spells, prefer the charge recharge duration over the spell cooldown duration for the swipe display:

```lua
local chargeInfo = C_Spell.GetSpellCharges(spellID)
if chargeInfo and chargeInfo.maxCharges > 1 then
    -- Use charge duration for swipe
    local chargeDur = C_Spell.GetSpellChargeDuration(spellID)
    if chargeDur then
        cooldownFrame:SetCooldownFromDurationObject(chargeDur)
    end
else
    -- Use spell cooldown duration for swipe
    local spellDur = C_Spell.GetSpellCooldownDuration(spellID)
    if spellDur then
        cooldownFrame:SetCooldownFromDurationObject(spellDur)
    end
end
```

**12.0.5 update — `ignoreGCD` parameter:** The charge-duration-constructing APIs gained a second `ignoreGCD` boolean parameter in 12.0.5. Pass `true` to exclude the global cooldown from the returned duration. The same parameter is available on `C_ActionBar.GetActionChargeDuration(slot, ignoreGCD)` and `C_SpellBook.GetSpellBookItemChargeDuration(spellBookItemSlotIndex, spellBookItemType, ignoreGCD)`.

**12.0.5 update — zero-span duration at max charges:** These charge-duration APIs now consistently return a **zero-span duration object** when the spell/action is at maximum charges (previously nil or a live-counting duration). Zero-span durations are treated as fully elapsed by the cooldown framework, so UI code can rely on `SetCooldownFromDurationObject(dur)` producing a "nothing to count down" display without special-casing.

### Charge Detection Gap (Caveat)

There is currently no clean non-secret API to determine if a charged spell is specifically on charge cooldown vs GCD. The `CooldownFlash:IsShown()` heuristic (checking if the charge-spent flipbook is visible) is used by addons as a workaround but is acknowledged as unreliable. The 12.0.1 `isEnabled` and `maxCharges` non-secret fields on cooldown info may provide better approaches in the future.

### Inventory Item Cooldowns

`GetInventoryItemCooldown("player", slotID)` tracks trinket and consumable cooldowns. Filter out GCD by checking the duration:

```lua
local start, duration, enable = GetInventoryItemCooldown("player", slotID)
if enable == 1 and duration > 1.5 then
    -- Real cooldown, not GCD
    local dur = C_DurationUtil.CreateDuration()
    dur:SetTimeFromStart(start, duration, 1)
    cooldownFrame:SetCooldownFromDurationObject(dur)
end
```

### Range Checking

Range checking is opt-in per spell:

1. On `CooldownIDSet`, check `C_Spell.SpellHasRange(baseSpellID)`
2. If true, call `C_Spell.EnableSpellRangeCheck(spellID, true)` and register `SPELL_RANGE_CHECK_UPDATE`
3. On range update, set `self.spellOutOfRange` and refresh icon color
4. On `CooldownIDCleared`, call `C_Spell.EnableSpellRangeCheck(spellID, false)` and unregister the event

### Icon Coloring

Four states, checked in order:

| State | Color | RGB | Condition |
|-------|-------|-----|-----------|
| Out of Range | Red | `(0.64, 0.15, 0.15)` | `spellOutOfRange == true` |
| Usable | White | `(1.0, 1.0, 1.0)` | `C_Spell.IsSpellUsable` returns true |
| Not Enough Mana | Blue | `(0.5, 0.5, 1.0)` | `notEnoughMana` is true |
| Not Usable | Gray | `(0.4, 0.4, 0.4)` | Default fallback |

An out-of-range overlay texture (`UI-CooldownManager-OORshadow`) is also shown at 50% alpha.

### Overlay Glow

`ActionButtonSpellAlertManager` handles proc overlay glow effects. The system checks `C_SpellActivationOverlay.IsSpellOverlayed(spellID)` on data refresh and responds to `SPELL_ACTIVATION_OVERLAY_GLOW_SHOW` / `SPELL_ACTIVATION_OVERLAY_GLOW_HIDE` events.

### Desaturation

The icon texture is desaturated (`SetDesaturated(true)`) when the spell is on actual cooldown (`isOnActualCooldown = not isOnGCD and cooldownIsActive`). Desaturation is cleared when the cooldown ends or when an aura is being displayed instead.

**Desaturation override for addons:** Blizzard's `RefreshIconDesaturation` uses the secret `isOnActualCooldown` field internally, which can cause issues in tainted contexts. Override with plain non-secret booleans:

```lua
local isRealCD = spellCooldownInfo.isActive and not spellCooldownInfo.isOnGCD
local hasRemainingCharges = chargeInfo and chargeInfo.currentCharges > 0
child.Icon:SetDesaturated(isRealCD == true and not hasRemainingCharges)
```

---

## Aura Tracking

### Self-Buff Tracking

The system registers `UNIT_AURA` for both `"player"` and `"target"` via `RegisterUnitEvent`.

For **player auras**, the system uses the `unitAuraUpdateInfo` payload for efficient updates:

- **Added:** Iterate `addedAuras`, check if any match associated spell IDs
- **Updated:** Look up item frames via `auraInstanceIDToItemFramesMap`
- **Removed:** Look up item frames via `auraInstanceIDToItemFramesMap`, clear aura state

The `auraInstanceIDToItemFramesMap` provides O(1) lookup from `auraInstanceID` to the item frame(s) tracking that aura. Items register/unregister when their aura instance changes.

### Target Debuff Tracking

For **target auras**, any change triggers a full refresh of all active frames (`RefreshActiveFramesForTargetChange`). Target debuffs are cached per-frame using `GetTargetAurasCached()` which queries `C_UnitAuras.GetUnitAuras("target", "HARMFUL|PLAYER")` once per `GetTime()` tick.

On `UNIT_TARGET` (player changes target), all items reset their state via `OnNewTarget()`:
- Active state forced to false
- Linked spell cleared
- Pandemic alert state cleared
- Full data refresh

### Totem Tracking

`PLAYER_TOTEM_UPDATE` is handled by calling `GetTotemInfo(slot)` and passing totem data to each item. Totem data includes slot, name, expirationTime, duration, and modRate. Totems take priority over aura data when both are present.

### Linked Spells

Each cooldown can have up to 4 linked spells (`linkedSpellIDs`). When a linked spell gains an active aura, it takes precedence over the base spell for display purposes. The `UpdateLinkedSpell` method checks incoming spell IDs against the linked list and promotes/demotes them.

### Pandemic Alert Calculation

The pandemic window is calculated using two APIs:

```lua
local extendedDuration = C_UnitAuras.GetRefreshExtendedDuration("target", auraInstanceID, spellID)
local baseDuration = C_UnitAuras.GetAuraBaseDuration("target", auraInstanceID, spellID)
local carriedOverToNewCast = extendedDuration - baseDuration
```

If `carriedOverToNewCast > 0`, the pandemic refresh window starts at `expirationTime - carriedOverToNewCast`. The pandemic alert only applies to **target debuffs** and only when the `PandemicTime` alert event type is configured.

### Custom Pandemic Overlay System

Blizzard's built-in `ShowPandemicStateFrame` / `CheckPandemicTimeDisplay` only calculates pandemic timing for **target auras**, making it unreliable for player buffs. Addons can suppress Blizzard's glow and drive custom pandemic overlays.

**Suppressing Blizzard's pandemic glow:**

```lua
child.PandemicIcon:SetAlpha(0)  -- Hide Blizzard's pandemic indicator
```

**Driving custom overlays with DurationObjects and color curves:**

```lua
-- Get DurationObject for the aura
local durObj = C_UnitAuras.GetAuraDuration(unit, auraInstanceID)

-- Create a step color curve for pandemic thresholds
local pandemicCurve = C_CurveUtil.CreateColorCurve()
pandemicCurve:SetType(Enum.LuaCurveType.Step)
pandemicCurve:AddPoint(0, CreateColor(1, 0, 0, 1))     -- <15%: red (pandemic window)
pandemicCurve:AddPoint(0.15, CreateColor(1, 0.5, 0, 1)) -- 15-30%: orange (approaching)
pandemicCurve:AddPoint(0.3, CreateColor(0, 0, 0, 0))    -- >30%: invisible (safe)
```

These curves are **secret-safe** because `DurationObject:EvaluateRemainingPercent(colorCurve)` evaluation happens at the C++ level.

**Evaluating the curve on each frame:**

```lua
-- In OnUpdate handler (throttled to ~10/sec)
local color = durObj:EvaluateRemainingPercent(pandemicCurve)
local finalAlpha = C_CurveUtil.EvaluateColorValueFromBoolean(
    durObj:IsZero(), 0, color.a
)
pandemicOverlay:SetAlpha(finalAlpha)
pandemicOverlay:SetVertexColor(color.r, color.g, color.b)
```

**Throttling:** DurationObjects are snapshots that must be re-fetched each tick. Limit evaluations to approximately 10 per second per child (0.1s interval) to avoid unnecessary overhead:

```lua
local PANDEMIC_THROTTLE = 0.1
local lastPandemicUpdate = 0

frame:SetScript("OnUpdate", function(self, elapsed)
    lastPandemicUpdate = lastPandemicUpdate + elapsed
    if lastPandemicUpdate < PANDEMIC_THROTTLE then return end
    lastPandemicUpdate = 0
    -- Re-fetch DurationObject and evaluate curve
end)
```

### Pandemic Overlay Styles

Three common styles for pandemic overlays:

| Style | Template/Approach | Description |
|-------|-------------------|-------------|
| Blizzard | CooldownPandemicFXTemplate | The built-in 3-texture pulse effect |
| Border | BackdropTemplate | Pulsing border alpha driven by curve evaluation |
| Proc | FlipBook animation | Uses `UI-HUD-ActionBar-Proc-Loop-Flipbook` atlas for a glowing proc-style effect |

---

## Alert System

### Alert Data Structure

Alerts are stored as compact 3-element arrays for serialization efficiency:

```lua
-- { type, event, payload }
-- Index 1: CooldownViewerAlertType (Sound or Visual)
-- Index 2: CooldownViewerAlertEventType (Available, PandemicTime, OnCooldown, ChargeGained)
-- Index 3: Payload (CooldownViewerSound enum or CooldownViewerVisual enum)
```

Accessor functions: `CooldownViewerAlert_GetType`, `CooldownViewerAlert_GetEvent`, `CooldownViewerAlert_GetPayload`.

### Alert Event Types and Triggers

| Event Type | Triggers When | Conditions |
|-----------|---------------|------------|
| `Available` | Cooldown finishes | Non-GCD, duration > 0.75s, `cooldownEnabled` is true |
| `PandemicTime` | Pandemic refresh window opens | Target debuff, `carriedOverToNewCast > 0` |
| `OnCooldown` | Ability goes on cooldown | Transitions from GCD/off-CD to actual cooldown |
| `ChargeGained` | A charge is predicted to be gained | `predictedChargeGainTime <= GetTime()` |

Alert dispatch is handled in `CooldownViewerItemMixin:OnUpdate()` which checks trigger times each frame.

### Sound Alerts

**68 total options:** 67 sound effects across 6 categories, plus Text-to-Speech.

| Category | Count | Examples |
|----------|-------|---------|
| Animals | 10 | Cat, Chicken, Cow, Gnoll, Goat, Lion, Panther, Rattlesnake, Sheep, Wolf |
| Devices | 11 | Boat Horn, Air Horn, Bike Horn, Cash Register, Jackpot Bell/Coins/Fail, Rotary Phone Dial/Ring, Stove Pipe, Trashcan Lid |
| Impacts | 10 | Anvil Strike, Bubble Smash, Low Thud, Metal Clanks/Rattle/Scrape/Warble, Pop Click, Strange Clang, Sword Scrape |
| Instruments | 12 | Bell Ring/Trill, Brass, Chime Ascending, Guitar Chug/Pinch, Pitch Pipe Distressed/Note, Synth Big/Buzz/High, Warhorn |
| War2 | 12 | Abstract Whoosh, Choir, Construction, Magic Chimes, Pig Squeal, Saws, Seal, Slow, Smith, Synth Stinger, Trumpet Rally, Zippy Magic |
| War3 | 12 | Bell, Crunchy Bell, Drum Splash, Error, Fanfare, Gate Open, Gold, Magic Shimmer, Ringout, Rooster, Shimmer Bell, Wolf Howl |
| TTS | 1 | Text-to-Speech (reads spell name via `TextToSpeechFrame_PlayCooldownAlertMessage`) |

Each sound entry maps to a `soundKitID` which is played via `PlaySound()`.

**IMPORTANT:** The `CooldownViewerSound` enum values must never be changed -- they are persisted in saved layout data. New sounds should only be appended.

### Visual Alerts

**10 visual types** in 2 families, each with 5 color variants:

| Visual Type | Template | Color | Hex |
|-------------|----------|-------|-----|
| MarchingAnts | `CDMVISMarchingAntsBase` | Gold | #FFD700 |
| MarchingAntsCyan | `CDMVISMarchingAntsCyan` | Cyan | #3FBFD4 |
| MarchingAntsRed | `CDMVISMarchingAntsRed` | Red | #CA1A1A |
| MarchingAntsGreen | `CDMVISMarchingAntsGreen` | Green | #4DAF63 |
| MarchingAntsBlue | `CDMVISMarchingAntsBlue` | Blue | #3F6EC7 |
| Flash | `CDMVISFlashBase` | Gold | #FFD700 |
| FlashCyan | `CDMVISFlashCyan` | Cyan | #3FBFD4 |
| FlashRed | `CDMVISFlashRed` | Red | #CA1A1A |
| FlashGreen | `CDMVISFlashGreen` | Green | #4DAF63 |
| FlashBlue | `CDMVISFlashBlue` | Blue | #3F6EC7 |

All visual alerts have a **2-second duration** and are managed by a pool-based system (`CooldownViewerVisualAlertsManager`). MarchingAnts use a flipbook animation (30 frames, 6x5 grid, 1s per cycle). Flash uses a bouncing alpha animation (0.25 to 1.0, 0.5s per half-cycle).

The visual alerts inherit from `AnimateWhileShownTemplate` and are anchored to the target item frame with an 8px expansion on each side.

### TTS

Text-to-speech uses `TextToSpeechFrame_PlayCooldownAlertMessage(alert, formattedName, allowOverlappedSpeech)` with the spell name. Overlapped speech is enabled so multiple alerts can play simultaneously.

### CDM Visual Alert Re-Anchoring

Hook `CooldownViewerVisualAlertsManager:AcquireAlert()` to re-anchor marching ants alert frames from Blizzard's default -8/+9 px overhang to custom sizing. This is useful when custom FlipBook pulse overlays use different insets:

```lua
hooksecurefunc(CooldownViewerVisualAlertsManager, "AcquireAlert", function(self, alert, target)
    -- Re-anchor the alert to match your custom overlay insets
    local alertFrame = target:GetVisualAlertFrame()
    if alertFrame then
        alertFrame:ClearAllPoints()
        alertFrame:SetPoint("TOPLEFT", target, "TOPLEFT", -4, 4)
        alertFrame:SetPoint("BOTTOMRIGHT", target, "BOTTOMRIGHT", 4, -4)
    end
end)
```

### Multi-Layer Glow Effect System

Addons can implement multiple independent glow styles on icons simultaneously. Each layer serves a different purpose:

| Layer | Source | Atlas / Technique |
|-------|--------|-------------------|
| Flash | One-shot proc burst | `UI-HUD-ActionBar-Proc-Start-Flipbook` FlipBook |
| Pulse | Looping proc glow | `UI-HUD-ActionBar-Proc-Loop-Flipbook` FlipBook with optional timer |
| Approaching | Subtle border pulse | Wrapper frame with secret-safe alpha from curve evaluation |
| Border | Alert border pulse | Same wrapper pattern, different trigger (e.g., low health) |
| Active | State indicator | Active trinket/buff indicator pulse |

**Secret-safe wrapper technique for duration-driven glows:**

Create a plain frame wrapper. Set its alpha via secret-returning curve evaluation (since `SetAlpha` accepts secrets natively at the C++ level). The wrapper alpha then multiplies with the animated border alpha inside it:

```lua
-- Create wrapper whose alpha is driven by DurationObject curve
local glowWrapper = CreateFrame("Frame", nil, iconFrame)
glowWrapper:SetAllPoints()

-- Animated border inside the wrapper
local border = glowWrapper:CreateTexture(nil, "OVERLAY")
border:SetAllPoints()
-- ... set up pulse animation ...

-- In OnUpdate: wrapper alpha comes from DurationObject evaluation
local dur = C_UnitAuras.GetAuraDuration("player", auraInstanceID)
local color = dur:EvaluateRemainingPercent(approachingCurve)
glowWrapper:SetAlpha(color.a)  -- Secret-safe: SetAlpha handles secrets
```

### SetAlphaFromBoolean (C++-Level Secret-Safe Method)

Glow frames expose a C++-level method for mapping secret booleans to alpha values:

```lua
glow:SetAlphaFromBoolean(secretBoolValue, falseAlpha, trueAlpha)
```

Maps a boolean value (which may be a secret) to an alpha value without exposing the boolean to Lua. Useful for conditionally showing/hiding glow effects based on secret combat state. Since this executes at the C++ level, it bypasses secret value restrictions entirely.

---

## Layout System

> **WARNING — Internal Blizzard UI infrastructure, not a public API.** `CooldownViewerLayoutManagerMixin`, `CooldownViewerSettingsDataProviderMixin`, and `CooldownViewerSettings:GetLayoutManager()` are internal mixins on Blizzard's settings UI, not part of the addon-callable surface.
>
> **Read-only methods are safe.** `GetLayout`, `GetLayoutByName`, and `EnumerateLayouts` are pure-Lua iterations over an internal table and behave correctly when called from addon code.
>
> **State-changing methods are NOT safe to call from addon code.** Any method that mutates layout state — `SetActiveLayout`, `SetActiveLayoutByID`, `AddLayout`, `RemoveLayout`, `RenameLayout`, `WriteCooldownInfo_*`, `ImportLayout`, `CopyLayout`, `ResetCurrentToDefaults`, `UseDefaultLayout` — synchronously calls `SetHasPendingChanges(true)`, which fires `EventRegistry:TriggerEvent("CooldownViewerSettings.OnPendingChanges", ...)` and `EventRegistry:TriggerEvent("CooldownViewerSettings.OnDataChanged")` on the caller's stack frame. The `OnDataChanged` listeners (`CooldownViewerMixin:RefreshLayout` and the settings-frame refresh) eventually call `C_CooldownViewer.GetCooldownViewerCategorySet` and `C_CooldownViewer.GetCooldownViewerCooldownInfo`, both of which are flagged `SecretArguments = "AllowedWhenUntainted"` in `CooldownViewerDocumentation.lua`. From any addon-tainted Lua stack frame, those C++ entry points refuse to operate and emit force-taint / secret-value errors. `securecallfunction` does **not** wash the C++ event taint attribution for these calls.
>
> **There is no public API for switching the active layout.** The `C_CooldownViewer` namespace exposes only six functions (`GetCooldownViewerCategorySet`, `GetCooldownViewerCooldownInfo`, `GetLayoutData`, `GetValidAlertTypes`, `IsCooldownViewerAvailable`, `SetLayoutData`); none of them switch layouts, and all but `GetLayoutData` and `IsCooldownViewerAvailable` are themselves `AllowedWhenUntainted`. **User-driven layout selection through Blizzard's cooldown-viewer settings UI is the only supported flow.** See [Layout Switching from Addon Code](#layout-switching-from-addon-code) below for details.
>
> Source citations: `+wow-ui-source+ (12.0.0)\Interface\AddOns\Blizzard_CooldownViewer\CooldownViewerSettingsLayoutManager.lua` lines 184, 540, 775; `CooldownViewerSettingsDataProvider.lua` lines 75-90; `+wow-ui-source+ (12.0.0)\Interface\AddOns\Blizzard_APIDocumentationGenerated\CooldownViewerDocumentation.lua` lines 13, 30, 79.
>
> **See also:** [12a_Secret_Safe_APIs.md — Predicates Table in API Documentation](12a_Secret_Safe_APIs.md#predicates-table-in-api-documentation-new-in-1205) for an explanation of the `AllowedWhenUntainted` flag and how to check it on any other API.

### Layout Object Structure

A layout is a plain Lua table created by `CooldownManagerLayout_Create`:

| Field | Type | Description |
|-------|------|-------------|
| `layoutID` | number | Unique identifier (0 = default layout, never stored) |
| `layoutName` | string | User-assigned name (nil/empty = auto-generated from class+spec) |
| `classAndSpecTag` | number | Encoded as `classID * 10 + specIndex` |
| `orderedCooldownIDs` | table | Ordered list of all cooldown IDs for this spec |
| `cooldownInfo` | table | Per-cooldown overrides: `{ [cooldownID] = { category = ..., alerts = {...} } }` |
| `isDefault` | bool | True if this is an auto-created "starter" layout |

### Per-Character, Per-Spec Storage

The class+spec tag is encoded as: `classID * 10 + specIndex` (e.g., Warrior Arms = `11`, Mage Frost = `83`). This limits spec indices to single digits (0-9), which is validated at runtime.

### Layout Limits

- Maximum **5 layouts** per character (not yet fully enforced in the code, marked as TODO)
- The **default layout ID** is `0` and represents "no custom layout" -- it is never stored

### CooldownManagerLayout_* Helper Functions

| Function | Description |
|----------|-------------|
| `CooldownManagerLayout_Create(id, name, tag)` | Create a new layout object |
| `CooldownManagerLayout_GetID(layout)` | Get layout ID |
| `CooldownManagerLayout_GetType(layout)` | Always returns `Character` (for now) |
| `CooldownManagerLayout_GetName(layout)` | Get display name (explicit or auto-generated) |
| `CooldownManagerLayout_GetNameExplicit(layout)` | Get explicit name only (nil if auto-generated) |
| `CooldownManagerLayout_SetName(layout, name)` | Set the layout name |
| `CooldownManagerLayout_GetClassAndSpecTag(layout)` | Get the class+spec tag |
| `CooldownManagerLayout_GetOrderedCooldownIDs(layout)` | Get ordered cooldown ID list |
| `CooldownManagerLayout_SetOrderedCooldownIDs(layout, ids)` | Set ordered cooldown ID list |
| `CooldownManagerLayout_GetCooldownInfo(layout, allowCreate)` | Get per-cooldown override table |
| `CooldownManagerLayout_IsDefaultLayout(layout)` | Check if auto-created starter layout |
| `CooldownManagerLayout_SetIsDefault(layout, isDefault)` | Set default flag |

### Layout Manager CRUD Operations

`CooldownViewerLayoutManagerMixin` provides the methods listed below. **Read the warning at the top of [Layout System](#layout-system) before calling any of these from addon code.** The two tables below split the surface by safety:

**Read-only operations (safe to call from addon code):**

| Operation | Method | Description |
|-----------|--------|-------------|
| Read | `GetLayout(id)` / `GetLayoutByName(name, specTag)` | Retrieve a layout (pure-Lua table lookup) |
| List | `EnumerateLayouts()` | Iterate all layouts (`pairs` over internal table) |

**Internal Blizzard UI methods (will trigger taint cascade if called from addon code):**

| Operation | Method | Description |
|-----------|--------|-------------|
| Create | `AddLayout(name, specTag, desiredID)` | Add a new layout, auto-assigns ID if needed |
| Update | `WriteCooldownInfo_Category(info, category)` | Update a cooldown's category |
| Delete | `RemoveLayout(id)` | Remove a layout and clear active state |
| Activate | `SetActiveLayout(layout)` / `SetActiveLayoutByID(id)` | Set the current layout |
| Import | `ImportLayout(name, layout)` | Import a copied layout |
| Copy | `CopyLayout(layout)` | Deep-copy with new ID |

Each method in the second table fires the `OnDataChanged` cascade described in the warning above. Use the cooldown-viewer settings UI for any user-driven mutation; see [Layout Switching from Addon Code](#layout-switching-from-addon-code) for the addon-code limitations.

### Undo Support

`CooldownViewerSettingsDataProviderMixin:ResetToRestorePoint()` reverts to the last saved state. The layout manager's `CreateRestorePoint` / `ResetToRestorePoint` pattern is used by the settings UI to allow "Revert Changes" via a `StaticPopupDialog`.

### Layout Switching from Addon Code

**There is no supported addon-code path for switching the active cooldown viewer layout in 12.0.0.** The taint architecture of the cooldown viewer subsystem means every available mechanism either runs the `OnDataChanged` cascade through addon-tainted stack frames or hits an `AllowedWhenUntainted` C++ entry point. Specifically:

| Approach | Why it fails from addon code |
|----------|------------------------------|
| `layoutManager:SetActiveLayout(layout)` | Synchronous `OnDataChanged` cascade lands on `GetCooldownViewerCategorySet` / `GetCooldownViewerCooldownInfo`, both `AllowedWhenUntainted`. |
| `layoutManager:SetActiveLayoutByID(id)` | Same cascade as `SetActiveLayout`. |
| `CooldownViewerSettings:SetActiveLayoutByID(id)` | Wraps the data-provider call, same end result. |
| `C_CooldownViewer.SetLayoutData(blob)` | The C++ entry itself is `SecretArguments = "AllowedWhenUntainted"` (`CooldownViewerDocumentation.lua:79`). Rewriting the serialized blob to flip the active layout ID and writing it back is blocked. |
| Custom non-secure button calling `CooldownViewerSettings:SetActiveLayoutByID(id)` | Click handler is addon-tainted; relocating the call site does not wash the originating taint. |

**The only `C_CooldownViewer` function without `AllowedWhenUntainted`** is `GetLayoutData()` (read-only). It returns the current serialized blob without raising taint, but there is no companion non-tainted writer.

**Recommended addon-code patterns instead of programmatic switching:**

1. **Lean on the built-in spec-based auto-switch.** `CooldownViewerLayoutManagerMixin:SwitchToBestLayoutForSpec` (`CooldownViewerSettingsLayoutManager.lua:259`) fires automatically from the data provider's `TRAIT_CONFIG_UPDATED` handler (`CooldownViewerSettingsDataProvider.lua:10-15`). If your goal is "switch layouts when the player changes spec," Blizzard already does that; no addon code needed.
2. **Direct the user to the cooldown-viewer settings UI.** Open the panel with `ShowUIPanel(CooldownViewerSettings)` and let the user pick a layout from the dropdown. Layout switches initiated by Blizzard's own UI run on a clean stack and do not taint.
3. **Read-only inspection is fine.** Calling `CooldownViewerSettings:GetLayoutManager():GetLayoutByName(name)` to discover layouts, or `C_CooldownViewer.GetLayoutData()` to snapshot the blob to saved-variables for backup, both work from addon code without taint. Restoring a backup, however, requires the user to invoke the import flow through the settings UI — there is no addon-code path to write the blob back.

### Layout Data Sync Across Characters

> **Caveat:** the snippet below uses `C_CooldownViewer.SetLayoutData(saved)`, which is `AllowedWhenUntainted` in 12.0.0. Calling it from an addon's login handler will fault with a force-taint / secret-value error. The read side (`GetLayoutData`) is safe; the write side is not. Until Blizzard exposes a non-tainted writer, this pattern works only for **read/backup**, not for restore. Track per-class blobs in saved variables for the user's reference, but the user must re-import them through the cooldown-viewer settings UI.

```lua
-- SAFE: snapshot current layout per class to saved variables (read-only side)
function addon:SaveLayoutForClass()
    local classID = select(3, UnitClass("player"))
    local rawData = C_CooldownViewer.GetLayoutData()
    if rawData then
        MyAddonDB.layoutsByClass = MyAddonDB.layoutsByClass or {}
        MyAddonDB.layoutsByClass[classID] = rawData
    end
end

-- BLOCKED IN 12.0.0: restoring on login faults because SetLayoutData is AllowedWhenUntainted.
-- Kept here for reference; do not wire this to login events from addon code.
function addon:RestoreLayoutForClass()
    local classID = select(3, UnitClass("player"))
    local saved = MyAddonDB.layoutsByClass and MyAddonDB.layoutsByClass[classID]
    if saved then
        C_CooldownViewer.SetLayoutData(saved) -- AllowedWhenUntainted -- will fault from addon code
    end
end
```

### Layout Cache Invalidation

The layout manager caches serialized data internally. To force it to re-read from the C API (e.g., after `SetLayoutData()`), invalidate the cache:

```lua
TextureLoadingGroupMixin.RemoveTexture(serializer, "cachedSerializedData")
TextureLoadingGroupMixin.AddTexture({textures = dataProvider}, "displayDataDirty")
```

### Import/Export

Layouts can be exported to a string via `SerializeLayouts(singleLayoutID)` and imported via `DeserializeLayouts(serializedData, isUserInput)`. The import flow:

1. User pastes serialized string into import dialog
2. `ProcessImportText` deserializes and creates temporary layouts
3. Layouts are deleted from the manager (cached in dialog)
4. User names the layout and confirms
5. `ImportLayout` adds the layout with the chosen name

---

## Serialization

### Wire Format

```
<encodingVersion>|<encodedPayload>
```

The pipe character (`|`) separates the encoding version from the encoded data.

### Encoding Pipeline

```
Lua Table  -->  CBOR  -->  Deflate  -->  Base64  -->  String
                (C_EncodingUtil.SerializeCBOR)
                          (C_EncodingUtil.CompressString, Deflate)
                                        (C_EncodingUtil.EncodeBase64)
```

Decoding reverses the pipeline: Base64 decode, Deflate decompress, CBOR deserialize.

### Data Format v4 Structure

The CBOR payload deserializes to a Lua table with these field IDs:

| Field ID | Key | Description |
|----------|-----|-------------|
| 1 | `SAVE_FIELD_ID_VERSION` | Data format version (currently 4) |
| 2 | `SAVE_FIELD_ID_ACTIVE_LAYOUT_NAMES` | `{ [specTag] = layoutID }` mapping |
| 3 | `SAVE_FIELD_ID_LAYOUTS` | `{ [specTag] = { [layoutID] = layoutData } }` |
| 4 | `SAVE_FIELD_ID_LAYOUT_ID_DATA` | `{ [layoutID] = layoutName }` mapping |

Each `layoutData` contains:

| Field ID | Key | Description |
|----------|-----|-------------|
| 1 | `SAVE_FIELD_ID_COOLDOWN_ORDER` | Ordered list of cooldown IDs |
| 2 | `SAVE_FIELD_ID_CATEGORY_OVERRIDES` | `{ [category] = { cooldownID, ... } }` |
| 3 | `SAVE_FIELD_ID_ALERT_OVERRIDES` | `{ [cooldownID] = { alert, ... } }` |

### Version History

| Version | Changes |
|---------|---------|
| v1 | Initial format. Active layout determined per-layout (last written = active). |
| v2 | Added explicit `ACTIVE_LAYOUT_NAMES` field. `specTag -> layoutName` mapping. |
| v3 | Added `ALERT_OVERRIDES` field for per-cooldown alert configurations. |
| v4 | Changed active layout tracking from name-based to ID-based. Added `LAYOUT_ID_DATA` for name storage. Layouts keyed by ID instead of name. |

### Persistence

Layout data is stored in the WoW account data system:

- `C_CooldownViewer.GetLayoutData()` reads from account data
- `C_CooldownViewer.SetLayoutData(data)` writes to account data

The `AccountData` enum entries are:
- `CooldownManager` (17)
- `CooldownManager2` (18)

The serialization layer supports pluggable persistence objects (any object implementing `SetSerializedData`, `GetSerializedData`, `ClearSerializedData`), but defaults to the C_CooldownViewer APIs.

---

## Settings UI

### Frame Structure

The settings frame (`CooldownViewerSettings`) uses `ButtonFrameTemplate` with a size of **399x609** pixels. It is registered as a UI panel via `SetUIPanelAttribute`.

### Tabs

| Tab | Content |
|-----|---------|
| **Spells** | Essential + Utility cooldowns, Hidden Spells |
| **Auras** | Tracked Buff + Tracked Bar, Hidden Auras |

### Display Categories

Six display categories shown as collapsible containers:

1. **Essential** -- Major cooldowns
2. **Utility** -- Minor cooldowns
3. **Tracked Buffs** -- Self-buff icons
4. **Tracked Bars** -- Self-buff bars
5. **Not in Bar (Spells)** -- Hidden spell pseudo-category
6. **Not in Bar (Auras)** -- Hidden aura pseudo-category

### Search Filtering

A search box filters cooldowns by name. Results are matched against `C_Spell.GetSpellName(spellID)` and the `cooldownViewerShowUnlearned` CVar controls whether unlearned spells are included.

### Drag-and-Drop Reordering

The settings UI supports drag-and-drop between categories:

1. `OnMouseDown` starts drag, creates a cursor icon frame (`CooldownViewerSettingsDraggedItemTemplate`)
2. `OnEnter` of target items triggers reorder marker display
3. `OnMouseUp` calls `ChangeOrderIndex(source, dest, reorderOffset)` on the data provider
4. Category changes happen automatically when dragging between categories

### Context Menu

Right-clicking a cooldown item opens a context menu with:

- **Move to [Category]** -- reassign to a different viewer category
- **Add Alert** -- opens the alert configuration panel
- **Edit Alert** / **Remove Alert** -- for existing alerts

### Alert Configuration Dialog

The alert editing panel (`CooldownViewerSettingsEditAlertMixin`) attaches to the right side of the settings frame with three dropdowns:

1. **Type** -- Sound or Visual
2. **When** -- Available, Pandemic Time, On Cooldown, Charge Gained (filtered by valid types for the cooldown)
3. **Payload** -- Specific sound or visual effect (Sound dropdown has nested sub-menus by category, with sample play buttons)

### Layout Management

A dropdown at the top of the settings frame manages layouts:

- **Use Default** -- deactivate all custom layouts
- **Switch Layout** -- select from existing layouts
- **New Layout** / **Rename** / **Delete**
- **Import** / **Export** -- clipboard-based serialized data
- **Revert Changes** -- undo to last saved state
- **Reset to Default** -- delete the active layout

---

## Viewer Frame Templates

### Four Viewer Frames

| Frame | Size | Category | Managed | Layout |
|-------|------|----------|---------|--------|
| EssentialCooldownViewer | 50x50 | Essential | Yes | 10 |
| UtilityCooldownViewer | 30x30 | Utility | Yes | 11 |
| BuffIconCooldownViewer | 40x40 | TrackedBuff | Yes | 9 |
| BuffBarCooldownViewer | 220x30 | TrackedBar | No | *(none)* |

**Frame details:**

| Frame | Mixin | Item Template | EditMode Index |
|-------|-------|---------------|----------------|
| Essential | EssentialCooldownViewerMixin | CooldownViewerEssentialItemTemplate | .Essential |
| Utility | UtilityCooldownViewerMixin | CooldownViewerUtilityItemTemplate | .Utility |
| BuffIcon | BuffIconCooldownViewerMixin | CooldownViewerBuffIconItemTemplate | .BuffIcon |
| BuffBar | BuffBarCooldownViewerMixin | CooldownViewerBuffBarItemTemplate | .BuffBar |

EditMode indices are under `Enum.EditModeCooldownViewerSystemIndices`.

All viewers inherit `CooldownViewerTemplate` which itself inherits `EditModeCooldownViewerSystemTemplate` and `GridLayoutFrame`. All have `ignoreInLayoutWhenActionBarIsOverriden = true` (except BuffBar which does not use UIParentBottomManaged).

### Viewer Frame Accessible Properties

Addon code can read these properties on CDM viewer frames:

| Property | Type | Description |
|----------|------|-------------|
| `viewer.isHorizontal` | boolean | True if viewer uses horizontal layout |
| `viewer.iconDirection` | number | Icon growth direction |
| `viewer.iconLimit` | number | Maximum icons shown |
| `viewer.iconScale` | number | Current icon scale factor |
| `viewer.childXPadding` | number | Horizontal padding between icons |
| `viewer.childYPadding` | number | Vertical padding between icons |
| `viewer.visibleSetting` | CooldownViewerVisibleSetting | Visibility mode (Always/InCombat/Hidden) |
| `viewer.settingMap` | table | Maps EditMode settings to their values (e.g., Opacity) |

| Method | Returns | Description |
|--------|---------|-------------|
| `viewer:IsInitialized()` | boolean | Returns false if viewer hasn't completed setup yet |

**Important:** Always check `viewer:IsInitialized()` before operating on viewers. Also check `EditModeManagerFrame.layoutApplyInProgress` -- defer operations while Blizzard is applying layouts.

### Item Template Details

**Essential Item** (`CooldownViewerEssentialItemTemplate`, 50x50):
- Icon with `UI-HUD-CoolDownManager-Mask` mask
- `UI-HUD-CoolDownManager-IconOverlay` frame overlay (-9,+8 / +9,-8)
- Out-of-range overlay (`UI-CooldownManager-OORshadow`, 50% alpha)
- Cooldown frame with custom swipe texture (`UI-HUD-CoolDownManager-Icon-Swipe`)
- Charge count (bottom-right, `NumberFontNormal`)
- Cooldown flash flipbook animation (22 frames, 0.75s)
- Countdown font: `GameFontHighlightHugeOutline`

**Utility Item** (`CooldownViewerUtilityItemTemplate`, 30x30):
- Same structure as Essential but smaller
- Overlay offset: -6,+5 / +6,-5
- Countdown font: `GameFontHighlightOutline`
- Charge count uses `NumberFontNormalSmall`

**Buff Icon Item** (`CooldownViewerBuffIconItemTemplate`, 40x40):
- Icon with mask and overlay (-8,+7 / +8,-7)
- **Reverse** cooldown swipe (fills up instead of draining)
- Swipe color: black at 70% alpha (vs white at 100% for cooldown items)
- Debuff border frame (`CooldownViewerItemDebuffBorderTemplate`)
- Applications count (bottom-right, `NumberFontNormal`)
- `allowHideWhenInactive = true` (hides when no active aura)

**Buff Bar Item** (`CooldownViewerBuffBarItemTemplate`, 220x30):
- Icon frame (30x30) at left with mask and overlay
- Debuff border frame
- Applications count on icon (bottom-right, `NumberFontNormalSmall`)
- StatusBar with `UI-HUD-CoolDownManager-Bar` atlas, default color `(1.0, 0.5, 0.25)`
- Background: `UI-HUD-CoolDownManager-Bar-BG`
- Pip texture: `UI-HUD-CoolDownManager-Bar-Pip` anchored to bar fill right edge
- Name font string (left-aligned, `NumberFontNormal`)
- Duration font string (right-aligned, `NumberFontNormal`, format: `COOLDOWN_DURATION_SEC`)
- `allowHideWhenInactive = true`
- StatusBar frame level: 511, Icon frame level: 512

### Child Frame (Icon) Accessible Properties

Addon code can read these properties on CDM icon child frames:

| Property | Type | Description |
|----------|------|-------------|
| `child.cooldownID` | number | Unique ID for `C_CooldownViewer.GetCooldownViewerCooldownInfo()` |
| `child.layoutIndex` | number | Position index in the viewer's layout |
| `child.wasSetFromAura` | boolean | True when icon shows aura state (vs cooldown) |
| `child.cooldownChargesShown` | boolean | True when charges display is active |
| `child.cooldownShowSwipe` | boolean | Whether swipe animation is enabled |
| `child.CooldownFlash` | frame | Flipbook frame shown when charges are spent |

| Method | Returns | Description |
|--------|---------|-------------|
| `child:GetCooldownInfo()` | table or nil | Returns cooldown info; nil for uninitialized frames |
| `child:GetBaseSpellID()` | number | Returns the base spell ID for this icon |

**Notes:**

- `child.cooldownID` is the primary non-secret way addons identify what spell a CDM icon represents. Pass it to `C_CooldownViewer.GetCooldownViewerCooldownInfo(cooldownID)` to get spell details.
- `child:GetCooldownInfo()` can return nil for frames that haven't been fully initialized -- always check for nil before using the return value.

---

## Pandemic Alert Animation

### Icon Pandemic (`CooldownPandemicFXTemplate`)

Used by Essential, Utility, and Buff Icon viewers.

- Inherits `AnimateWhileShownTemplate`
- Contains a `Border` frame with `UI-CooldownManager-PandemicBorder` atlas
- Contains an `FX` frame with 3 overlapping pulse textures:
  - `UI-CooldownManager-PandemicFX-Icon01`
  - `UI-CooldownManager-PandemicFX-Icon02`
  - `UI-CooldownManager-PandemicFX-Icon03`
- All 3 textures masked by `UI-CooldownManager-PandemicBorder-Mask`
- Animation: **looping REPEAT**, 2-second total cycle per texture, staggered by 1.5s:
  - Scale from 0.25 to 1.5 over 2s with IN_OUT smoothing
  - Alpha: 0 to 1 (0.5s), hold at 1 (1s), 1 to 0 (0.5s)
- Anchored to item with -6,+6 / +6,-6 offset

### Bar Pandemic (`CooldownPandemicBarFXTemplate`)

Used by Buff Bar viewer.

- Border frame with `UI-CooldownManager-PandemicBorderBar` atlas
- Single rotating texture: `UI-CooldownManager-PandemicFX-Bar`
  - Oversized: anchored at -256,+95 / +251,-199 relative to parent
  - Masked by `UI-CooldownManager-PandemicBorderBar-Mask`
- Animation: **looping REPEAT**, 360-degree rotation over **5 seconds**
- Anchored to bar with -9,+10 / +9,-10 offset, frame level = bar level + 1

---

## CooldownFrame Widget API

The `Cooldown` frame type is a specialized frame widget that displays radial cooldown animations. For full secret value implications, see [Secret Safe APIs](12a_Secret_Safe_APIs.md) and [API Migration Guide](12_API_Migration_Guide.md).

### Setting Cooldown Values

| Method | Parameters | Status |
|--------|-----------|--------|
| SetCooldown | start, duration, modRate=1 | **RESTRICTED (tainted+secret)** |
| SetCooldownDuration | duration, modRate=1 | **RESTRICTED (tainted+secret)** |
| SetCooldownFromExpirationTime | expirationTime, duration, modRate=1 | **RESTRICTED (tainted+secret)** |
| SetCooldownUNIX | start, duration, modRate=1 | **RESTRICTED (tainted+secret)** |
| SetCooldownFromDurationObject | duration (LuaDurationObject), clearIfZero=true | **SAFE** |

The four restricted methods have `SecretArguments = "AllowedWhenTainted"` and add the Cooldown secret aspect — they fail from tainted addon code with secret values in 12.0.1+. `SetCooldownFromDurationObject` has `SecretArguments = "AllowedWhenUntainted"` and is the **only** method usable from tainted code. Use `C_Spell.GetSpellCooldownDuration()` or `C_DurationUtil.CreateDuration()` to obtain Duration objects.

### Reading Cooldown Values (Secret-Returning)

| Method | Returns | Secret Aspect | Description |
|--------|---------|--------------|-------------|
| `GetCooldownDisplayDuration` | `duration` (ms, unaffected by modRate) | Cooldown | May return secret value |
| `GetCooldownDuration` | `duration` (ms, multiplied by modRate) | Cooldown | May return secret value |
| `GetCooldownTimes` | `start`, `duration` | Cooldown | Both may be secret |

### Non-Secret Getters

| Method | Returns | Description |
|--------|---------|-------------|
| `GetCountdownFontString` | SimpleFontString | The countdown text FontString |
| `GetDrawBling` | bool | Whether bling animation plays on completion |
| `GetDrawEdge` | bool | Whether the sweeping edge is drawn |
| `GetDrawSwipe` | bool | Whether the swipe overlay is drawn |
| `GetEdgeScale` | number | Scale of the edge texture |
| `GetHideCountdownNumbers` | bool | Whether countdown numbers are hidden |
| `GetMinimumCountdownDuration` | number (ms) | Minimum duration to show countdown text |
| `GetReverse` | bool | Whether swipe direction is reversed |
| `GetRotation` | number (radians) | Rotation offset |
| `GetUseAuraDisplayTime` | bool | Whether to use aura-specific display timing |
| `IsPaused` | bool | Whether the cooldown is paused |

### Control Methods

| Method | Parameters | Description |
|--------|-----------|-------------|
| `Clear` | *(none)* | Remove the cooldown display |
| `Pause` | *(none)* | Pause the cooldown animation |
| `Resume` | *(none)* | Resume a paused cooldown |

### Appearance Methods

All appearance setters have `SecretArguments = "AllowedWhenUntainted"`:

| Method | Parameters | Description |
|--------|-----------|-------------|
| `SetBlingTexture` | texture, r, g, b, a | Set the completion bling texture |
| `SetCountdownAbbrevThreshold` | seconds | Threshold for abbreviated countdown text |
| `SetCountdownFont` | fontName | Set the countdown font |
| `SetCountdownFormatter` *(12.0.5+)* | formatter (NumericFormatter, Nilable) | Plug in a `NumericFormatter` subclass (e.g., `AbbreviatedNumberFormatter`, `SecondsFormatter`, `NumericRuleFormatter`) to control how the countdown renders. Formatter `Format` functions accept secret numbers natively. Pass `nil` to clear. See [Formatter-Based Countdown Text](#formatter-based-countdown-text-1205) below. |
| `SetCountdownMillisecondsThreshold` *(12.0.5+)* | seconds | Below this threshold, cooldown numbers display with one decimal place (e.g., "6.7") instead of integer seconds. |
| `SetDrawBling` | bool | Enable/disable bling animation |
| `SetDrawEdge` | bool | Enable/disable sweeping edge |
| `SetDrawSwipe` | bool | Enable/disable swipe overlay |
| `SetEdgeColor` | r, g, b, a? | Set edge color |
| `SetEdgeScale` | scale | Set edge texture scale |
| `SetEdgeTexture` | texture, r, g, b, a | Set edge texture |
| `SetHideCountdownNumbers` | bool | Show/hide countdown numbers |
| `SetMinimumCountdownDuration` | ms | Minimum total duration to show countdown |
| `SetPaused` | bool | Pause/resume |
| `SetReverse` | bool | Reverse swipe direction |
| `SetRotation` | radians | Rotate the cooldown display |
| `SetSwipeColor` | r, g, b, a? | Set swipe color |
| `SetSwipeTexture` | texture, r, g, b, a | Set swipe texture |
| `SetTexCoordRange` | low (Vector2D), high (Vector2D) | Set texture coordinate range |
| `SetUseAuraDisplayTime` | bool | Toggle aura display timing |
| `SetUseCircularEdge` | bool | Use circular edge shape |

### Utility Functions

| Function | Parameters | Description |
|----------|-----------|-------------|
| `CooldownFrame_Set` | `self`, `start`, `duration`, `enable`, `forceShowDrawEdge`, `modRate` | Convenience wrapper: if enabled and start/duration > 0, calls `SetCooldown`; otherwise calls `CooldownFrame_Clear`. **Inherits `SetCooldown` restrictions** — not safe from tainted code with secret values in 12.0.1+; use `SetCooldownFromDurationObject` instead in that case. |
| `CooldownFrame_Clear` | `self` | Calls `self:Clear()` |
| `CooldownFrame_SetDisplayAsPercentage` | `self`, `percentage` | Pauses the cooldown and sets it to display a static percentage (0-1). **Inherits `SetCooldown` restrictions** — not safe from tainted code with secret values in 12.0.1+. |

### Formatter-Based Countdown Text (12.0.5+)

Patch 12.0.5 added two new CooldownFrame methods that let addons customize how countdown text is rendered — and critically, the new formatter types accept **secret numbers natively**. This is the preferred replacement for `pcall(string.format, "%.0f", secretValue)` workarounds in 12.0.5+.

**New CooldownFrame methods:**

- `cooldown:SetCountdownFormatter(formatter)` — plug in a `NumericFormatter` subclass to control countdown text. Pass `nil` to clear. `SecretArguments = "AllowedWhenUntainted"`.
- `cooldown:SetCountdownMillisecondsThreshold(seconds)` — below this threshold, numbers render with one decimal place (e.g., "6.7"). `SecretArguments = "AllowedWhenUntainted"`.

**New numeric formatter types (all accept secret numbers via their `Format` function on duration objects):**

| Formatter | Use case |
|-----------|----------|
| `AbbreviatedNumberFormatter` | Human-readable abbreviations (e.g., "1.2M") |
| `NumericRuleFormatter` | Locale-aware pluralization and rule-based formatting |
| `SecondsFormatter` | Duration / time formatting |

Base class: `NumericFormatter`. API documentation for each formatter type lives in `Interface\AddOns\Blizzard_APIDocumentationGenerated\` under `AbbreviatedNumberFormatterAPIDocumentation.lua`, `NumericFormatterAPIDocumentation.lua`, `NumericRuleFormatterAPIDocumentation.lua`, and `SecondsFormatterAPIDocumentation.lua` in the 12.0.5 client source.

**Usage:**

```lua
-- 12.0.5+ preferred pattern for secret-safe countdown text
local cooldown = myButton.Cooldown
local dur = C_Spell.GetSpellCooldownDuration(spellID)
if dur then
    cooldown:SetCooldownFromDurationObject(dur)
    cooldown:SetCountdownFormatter(SecondsFormatter)
    cooldown:SetCountdownMillisecondsThreshold(10)  -- show "6.7" style under 10s
end
```

**Cross-client fallback (pre-12.0.5):**

```lua
-- Check for 12.0.5+ formatter support
local tocVersion = select(4, GetBuildInfo())
if tocVersion >= 120005 and cooldown.SetCountdownFormatter and SecondsFormatter then
    cooldown:SetCountdownFormatter(SecondsFormatter)
    cooldown:SetCountdownMillisecondsThreshold(10)
else
    -- Older clients — fall back to the pcall(string.format) pattern on the text FontString:
    -- local ok, text = pcall(string.format, "%.0f", secretSeconds)
    -- cooldown.Text:SetText(ok and text or "?")
end
```

See [12_API_Migration_Guide.md](12_API_Migration_Guide.md) for the full 12.0.5 delta, and [12a_Secret_Safe_APIs.md](12a_Secret_Safe_APIs.md) for complete secret-value handling patterns.

---

## Addon Integration Patterns

### Querying Available Cooldowns

```lua
-- Check if the system is available
local isAvailable, failureReason = C_CooldownViewer.IsCooldownViewerAvailable()
if not isAvailable then
    -- failureReason explains why (e.g., character level too low)
    return
end

-- Get all Essential cooldowns for the current spec
local essentialIDs = C_CooldownViewer.GetCooldownViewerCategorySet(
    Enum.CooldownViewerCategory.Essential,
    false  -- only learned spells
)

for _, cooldownID in ipairs(essentialIDs) do
    local info = C_CooldownViewer.GetCooldownViewerCooldownInfo(cooldownID)
    if info then
        local spellName = C_Spell.GetSpellName(info.spellID)
        local texture = C_Spell.GetSpellTexture(info.spellID)
        -- Use cooldownID, spellName, texture...
    end
end
```

### Getting Cooldown Info for a Specific Spell

```lua
-- You need the cooldownID, not the spellID
-- Search through all categories to find it
local function FindCooldownIDForSpell(targetSpellID)
    local categories = {
        Enum.CooldownViewerCategory.Essential,
        Enum.CooldownViewerCategory.Utility,
        Enum.CooldownViewerCategory.TrackedBuff,
        Enum.CooldownViewerCategory.TrackedBar,
    }
    
    for _, category in ipairs(categories) do
        local cooldownIDs = C_CooldownViewer.GetCooldownViewerCategorySet(category, true)
        for _, cooldownID in ipairs(cooldownIDs) do
            local info = C_CooldownViewer.GetCooldownViewerCooldownInfo(cooldownID)
            if info and info.spellID == targetSpellID then
                return cooldownID, info
            end
        end
    end
    return nil
end
```

### Working with Layout Data

```lua
-- Read the raw serialized layout data
local rawData = C_CooldownViewer.GetLayoutData()

-- The data is in the format: "encodingVersion|base64EncodedCBOR"
-- To decode: Base64 -> Deflate decompress -> CBOR deserialize
-- Use C_EncodingUtil APIs:
local delimiterIndex = string.find(rawData, "|", 1, true)
if delimiterIndex then
    local payload = string.sub(rawData, delimiterIndex + 1)
    local decoded = C_EncodingUtil.DecodeBase64(payload)
    local inflated = C_EncodingUtil.DecompressString(decoded, Enum.CompressionMethod.Deflate)
    local dataTable = C_EncodingUtil.DeserializeCBOR(inflated)
    -- dataTable now contains the layout structure
end
```

### Working with CooldownFrame Safely (Secret Values)

In WoW 12.0.1+, the standard `SetCooldown`, `SetCooldownDuration`, `SetCooldownFromExpirationTime`, and `SetCooldownUNIX` methods are **restricted** when called from tainted (addon) code with secret values. Use `SetCooldownFromDurationObject` instead.

```lua
-- SAFE: Using Duration Objects (12.0.1+)
local function SetupCooldownDisplay(cooldownFrame, spellID)
    -- Get a Duration object (not a raw number)
    local durationObj = C_Spell.GetSpellCooldownDuration(spellID)
    if durationObj then
        cooldownFrame:SetCooldownFromDurationObject(durationObj)
    else
        cooldownFrame:Clear()
    end
end

-- For item cooldowns (no native Duration API), synthesize manually:
local function SetupItemCooldown(cooldownFrame, startTime, duration, modRate)
    local dur = C_DurationUtil.CreateDuration()
    dur:SetTimeFromStart(startTime, duration, modRate)
    cooldownFrame:SetCooldownFromDurationObject(dur)
end

-- UNSAFE in 12.0.1+ (will error with secret values from tainted code):
-- cooldownFrame:SetCooldown(start, duration, modRate)
-- CooldownFrame_Set(cooldownFrame, start, duration, enable, drawEdge, modRate)
```

See [Secret Safe APIs](12a_Secret_Safe_APIs.md) for comprehensive secret value handling patterns and [Advanced Techniques](10_Advanced_Techniques.md) for more complex cooldown display scenarios.

### Enabling CooldownViewer Programmatically

Before interacting with the CooldownViewer system, ensure it is enabled via the `cooldownViewerEnabled` CVar:

```lua
if GetCVar("cooldownViewerEnabled") ~= "1" then
    SetCVar("cooldownViewerEnabled", "1")
end
```

Guard against redundant `SetCVar` calls to avoid CVar taint propagation.

### Creating a Private Cooldown Frame

```lua
-- Create a Cooldown frame programmatically
local myFrame = CreateFrame("Frame", nil, UIParent)
myFrame:SetSize(50, 50)
myFrame:SetPoint("CENTER")

local icon = myFrame:CreateTexture(nil, "ARTWORK")
icon:SetAllPoints()

local cooldown = CreateFrame("Cooldown", nil, myFrame, "CooldownFrameTemplate")
cooldown:SetAllPoints()
cooldown:SetDrawEdge(false)
cooldown:SetDrawBling(true)
cooldown:SetHideCountdownNumbers(false)

-- Use Duration objects for secret-safe cooldown display
local function UpdateCooldown(spellID)
    local durationObj = C_Spell.GetSpellCooldownDuration(spellID)
    if durationObj then
        cooldown:SetCooldownFromDurationObject(durationObj)
    else
        cooldown:Clear()
    end
    
    local texture = C_Spell.GetSpellTexture(spellID)
    icon:SetTexture(texture)
end
```

### Squarifying CooldownViewer Icons

Replace the rounded mask and overlay with square equivalents for a cleaner look:

```lua
-- Replace rounded mask with square
local maskTexture = child.Icon:GetMaskTexture(1)
if maskTexture then
    child.Icon:RemoveMaskTexture(maskTexture)
    -- Apply a square mask or leave unmasked
end

-- Replace mask texture (FileID 6707800) with solid square
child.IconMask:SetTexture("Interface\\Buttons\\WHITE8x8")

-- Hide overlay atlas (do NOT remove it)
child.IconOverlay:SetAlpha(0)

-- Replace CooldownFrame swipe texture with square
child.Cooldown:SetSwipeTexture("Interface\\Buttons\\WHITE8x8")

-- Hide DebuffBorder via alpha, NEVER nil it (see Taint section)
if child.DebuffBorder then
    child.DebuffBorder:SetAlpha(0)
end
```

### Squarifying Buff Bars

Replace bar textures with solid rectangles for consistent theming:

```lua
-- Replace bar fill atlas with solid color
child.Bar:SetStatusBarTexture("Interface\\Buttons\\WHITE8x8")
child.Bar:SetStatusBarColor(0.8, 0.6, 0.2, 1.0)

-- Replace background atlas with solid dark rectangle
child.Bar.BG:SetTexture("Interface\\Buttons\\WHITE8x8")
child.Bar.BG:SetVertexColor(0.1, 0.1, 0.1, 0.8)

-- Scale pip proportionally to bar height
local barHeight = child.Bar:GetHeight()
child.Bar.Pip:SetSize(2, barHeight)
```

### Icon Texture Override with SetTexture Guard Hook

Apply custom textures and install a `hooksecurefunc` on `Icon:SetTexture` to re-apply after Blizzard's `RefreshSpellTexture` resets them. Use a recursion guard to prevent infinite loops:

```lua
local isApplyingTexture = false

local function ApplyCustomTexture(child, textureID)
    if isApplyingTexture then return end
    isApplyingTexture = true
    child.Icon:SetTexture(textureID)
    isApplyingTexture = false
end

hooksecurefunc(child.Icon, "SetTexture", function(self, tex)
    if isApplyingTexture then return end
    -- Re-apply custom texture when Blizzard resets it
    local customTex = GetCustomTextureForChild(child)
    if customTex then
        ApplyCustomTexture(child, customTex)
    end
end)
```

### Keybind Text Overlay

Resolve `spellID` to a keybind and display it on icons. Create FontStrings on a **separate overlay frame** (parented to your container, not Blizzard frames) to avoid taint:

```lua
-- Resolve spellID to keybind
local function GetKeybindForSpell(spellID)
    -- Try action bar buttons first
    local slots = C_ActionBar.FindSpellActionButtons(spellID)
    if slots and #slots > 0 then
        local command = "ACTIONBUTTON" .. slots[1]
        local key = GetBindingKey(command)
        if key then return key end
    end
    -- Fall back to direct spell binding
    local spellName = C_Spell.GetSpellName(spellID)
    if spellName then
        return GetBindingKey("SPELL " .. spellName)
    end
    return nil
end

-- Create overlay frame on addon container (not on Blizzard frame)
local keybindOverlay = CreateFrame("Frame", nil, addonContainer)
keybindOverlay:SetAllPoints(child)
local keybindText = keybindOverlay:CreateFontString(nil, "OVERLAY", "NumberFontNormalSmallGray")
keybindText:SetPoint("TOPLEFT", 2, -2)

-- Invalidate on binding changes
local bindingFrame = CreateFrame("Frame")
bindingFrame:RegisterEvent("UPDATE_BINDINGS")
bindingFrame:RegisterEvent("ACTIONBAR_SLOT_CHANGED")
bindingFrame:SetScript("OnEvent", function() RefreshAllKeybinds() end)
```

### Button Press Visual Mirroring

Mirror action bar "pushed" visuals onto tracker icons. Hook `ActionButtonDown`/`ActionButtonUp` and LibActionButton-1.0 `PreClick` to resolve the pressed spell and find the matching CooldownViewer child:

```lua
-- Hook standard action buttons
hooksecurefunc("ActionButtonDown", function(id)
    local actionType, actionID = GetActionInfo(id)
    if actionType == "spell" then
        local child = FindChildBySpellID(actionID)
        if child then
            child.Icon:SetVertexColor(0.6, 0.6, 0.6)  -- Dim for "pressed"
        end
    end
end)

hooksecurefunc("ActionButtonUp", function(id)
    -- Restore normal color
    local actionType, actionID = GetActionInfo(id)
    if actionType == "spell" then
        local child = FindChildBySpellID(actionID)
        if child then
            child.Icon:SetVertexColor(1, 1, 1)
        end
    end
end)
```

### Cross-Parent Rendering

CooldownViewer children remain parented to the viewer but can be positioned via `SetPoint` to external container frames. Use `SetIgnoreParentAlpha(true)` to prevent parent alpha from affecting routed children:

```lua
-- Child stays parented to viewer but is positioned by container
child:SetIgnoreParentAlpha(true)
child:ClearAllPoints()
child:SetPoint("BOTTOMLEFT", addonContainer, "BOTTOMLEFT", xOffset, yOffset)
```

This pattern keeps the Blizzard secure parent chain intact while giving the addon full control over positioning and visibility.

### Spell Override Resolution

Check for spell overrides when routing spells to icons. For example, Divine Toll may override to Hammer of Light:

```lua
local effectiveSpellID = C_Spell.GetOverrideSpell(baseSpellID)
-- effectiveSpellID is the currently active version of the spell
-- Falls back to baseSpellID if no override is active
```

### Skyriding Ability Override Mode

Detect skyriding via `C_ActionBar.GetBonusBarIndex()` (bonus bar 11, offset 5). Scan action bar slots 121-132 for skyriding abilities and create standalone icon frames:

```lua
local SKYRIDING_BONUS_BAR = 11
local SKYRIDING_SLOT_OFFSET = 120  -- Slots 121-132

local function IsSkyriding()
    return C_ActionBar.GetBonusBarIndex() == SKYRIDING_BONUS_BAR
end

local function GetSkyridingAbilities()
    local abilities = {}
    for slot = SKYRIDING_SLOT_OFFSET + 1, SKYRIDING_SLOT_OFFSET + 12 do
        local actionType, id = GetActionInfo(slot)
        if actionType == "spell" and id then
            table.insert(abilities, { slot = slot, spellID = id })
        end
    end
    return abilities
end

-- Create standalone icon frames with DurationObject-based cooldowns
local function UpdateSkyridingCooldowns()
    for _, ability in ipairs(GetSkyridingAbilities()) do
        local dur = C_Spell.GetSpellCooldownDuration(ability.spellID)
        if dur then
            skyridingIcons[ability.slot].Cooldown:SetCooldownFromDurationObject(dur)
        end
    end
end
```

### Custom Bar Content Enum Value

Define values outside Blizzard's `Enum.CooldownViewerBarContent` range for custom display modes:

```lua
-- Blizzard values: 0=IconAndName, 1=IconOnly, 2=NameOnly
-- Custom addon value (outside Blizzard range):
local BAR_CONTENT_BAR_ONLY = 99  -- Bar only, no name or icon

-- Usage in your layout logic:
if barContent == BAR_CONTENT_BAR_ONLY then
    child.Icon:SetAlpha(0)
    child.Name:SetAlpha(0)
end
```

### Context Menu Hook Point (MENU_COOLDOWN_SETTINGS_ITEM)

Addons can add custom options to the CDM icon right-click context menu using the `MENU_COOLDOWN_SETTINGS_ITEM` hook:

```lua
Menu.ModifyMenu("MENU_COOLDOWN_SETTINGS_ITEM", function(owner, rootDescription, contextData)
    -- owner is the CDM icon frame
    -- owner:GetCooldownInfo() returns the cooldown info
    -- owner:GetBaseSpellID() returns the spell ID
    
    rootDescription:CreateDivider()
    rootDescription:CreateTitle("My Addon Settings")
    
    local spellID = owner:GetBaseSpellID()
    rootDescription:CreateCheckbox("Custom Option", function()
        return MyAddon:IsOptionEnabled(spellID)
    end, function()
        MyAddon:ToggleOption(spellID)
    end)
end)
```

### Opening CDM Settings Programmatically

There are two ways to open the settings panel from addon code, with different trade-offs:

- **`:ShowUIPanel(fromEditMode)`** (the mixin method) calls the global `ShowUIPanel(self)` internally. This integrates the panel with the UIPanelWindows system — proper layout (won't overlap other panels), ESC closes it, edit-mode handoff is performed when `fromEditMode=true`. Historically the UIPanelWindows system has been a taint vector when invoked from addon code in certain scenarios.
- **`:Show()`** bypasses the UIPanelWindows system entirely. Safer from UIPanel taint risk, but you lose layout management, ESC-to-close, and edit-mode handoff.

```lua
if InCombatLockdown() then
    -- Show() is always safe in combat (no UIPanel interaction at all).
    CooldownViewerSettings:Show()
else
    -- Out of combat, the mixin method gives proper UIPanelWindows
    -- integration. fromEditMode = false here.
    CooldownViewerSettings:ShowUIPanel(false)
end
```

### Checking Texture FileIDs with Secret Values

Texture values read from frame properties can be secret in tainted contexts. Always check with `issecretvalue()` before comparing FileIDs:

```lua
local tex = child.Icon:GetTexture()
if issecretvalue and issecretvalue(tex) then
    -- Cannot compare, use fallback logic
else
    if tex == 6707800 then  -- Known mask FileID
        -- Handle specific texture
    end
end
```

### C_AssistedCombat Integration

The Assisted Combat API powers the "Assisted Combat Rotation" feature that highlights suggested spells on CDM icons:

```lua
-- Get the list of spells in the current rotation suggestion
local rotationSpells = C_AssistedCombat.GetRotationSpells()
-- Returns array of { spellID = number }

-- Get the next suggested spell to cast
local nextSpell = C_AssistedCombat.GetNextCastSpell()
-- Returns { spellID = number } or nil

-- Hook to sync with Blizzard's highlight system
hooksecurefunc(AssistedCombatManager, "UpdateAllAssistedHighlightFramesForSpell",
    function(self, spellID, isActive)
        -- spellID: the spell being highlighted/unhighlighted
        -- isActive: true if this spell should be highlighted
    end
)
```

Addons can use this to show rotation suggestions on CDM icons by matching `C_AssistedCombat.GetRotationSpells()` against `child.cooldownID` resolved via `C_CooldownViewer.GetCooldownViewerCooldownInfo()`.

### Utility Dimming with C_CurveUtil

Secret-safe dimming technique to reduce opacity of off-cooldown icons:

```lua
-- Create a curve that maps remaining duration to opacity
-- When duration is 0 (off cooldown): opacity = dimAlpha
-- When duration > 0.1 (on cooldown): opacity = 1.0
local dimCurve = C_CurveUtil.CreateCurve()
dimCurve:AddPoint(0, dimAlpha)   -- at 0 seconds remaining: dim
dimCurve:AddPoint(0.1, 1.0)      -- at 0.1+ seconds remaining: full opacity

-- Apply to a cooldown's duration object. Cooldown frames do NOT expose a
-- GetDurationObject method; resolve the spellID from the CDM child's
-- cooldownID and ask C_Spell for a LuaDurationObject.
local cooldownInfo = C_CooldownViewer.GetCooldownViewerCooldownInfo(child.cooldownID)
local durationObj = cooldownInfo and C_Spell.GetSpellCooldownDuration(cooldownInfo.spellID)
if durationObj then
    -- There is no SetAlphaFromCurve method on any frame type. Evaluate the
    -- curve against the remaining duration ourselves and call SetAlpha with
    -- the numeric result. For a scalar curve (C_CurveUtil.CreateCurve) the
    -- result is a number; EvaluateRemainingDuration is non-secret when the
    -- curve itself is non-secret (SecretWhenCurveSecret = true).
    child:SetAlpha(durationObj:EvaluateRemainingDuration(dimCurve))
end
```

This technique is fully secret-safe because `EvaluateRemainingDuration` handles secret duration values at the C++ level.

---

## Key Blizzard Source Files Reference

All paths are relative to `Interface/AddOns/Blizzard_CooldownViewer/` within the Blizzard UI source.

| File | Lines | Description |
|------|-------|-------------|
| `Blizzard_CooldownViewer.toc` | 27 | TOC: load order, dependencies, retail-only restriction |
| `CooldownViewerSettingsConstants.lua` | 140 | Enums (CooldownViewerSound, CooldownViewerVisual), pseudo-categories, CDMLayoutMode |
| `CooldownViewerUtil.lua` | 53 | Class+spec tag encoding/decoding, `IsDisabledCategory` helper |
| `CooldownViewerVisualAlert.lua` | 84 | `CooldownViewerVisualAlertMixin`: alert target, anchoring, priority |
| `CooldownViewerVisualAlertTemplates.lua` | 65 | `CDMVISBaseMixin`, `CDMVISMarchingAntsBaseMixin`, `CDMVISFlashBaseMixin`: vertex color, duration, anchors |
| `CooldownViewerVisualAlertTemplates.xml` | 88 | XML templates for all 10 visual alert variants (5 MarchingAnts + 5 Flash) |
| `CooldownViewerVisualAlertsManager.lua` | 32 | `CooldownViewerVisualAlertsManagerMixin`: pool-based alert acquire/release |
| `CooldownViewerVisualAlertsManager.xml` | 8 | XML frame for `CooldownViewerVisualAlertsManager` singleton |
| `CooldownViewerVisualAlertTarget.lua` | 58 | `CooldownViewerVisualAlertTargetMixin`: alert container, sorting, frame level management |
| `CooldownViewerSoundAlertData.lua` | 87 | 67 sound entries across 6 categories with soundKitIDs |
| `CooldownViewerVisualAlertData.lua` | 14 | 10 visual alert entries mapping enums to animation templates |
| `CooldownViewerAlert.lua` | 254 | Alert data structure (3-field array), play dispatch (sound/visual/TTS), status validation |
| `CooldownViewerSettingsAlerts.lua` | 220 | `CooldownViewerSettingsEditAlertMixin`: 3-dropdown alert config panel, add/edit logic |
| `CooldownViewerSettingsAlerts.xml` | 88 | XML for the alert editing panel UI |
| `CooldownViewerSettingsDataStoreSerialization.lua` | 443 | `CooldownViewerDataStoreSerializationMixin`: CBOR/Deflate/Base64, v1-v4 migration readers |
| `CooldownViewerSettingsLayoutManager.lua` | 978 | `CooldownViewerLayoutManagerMixin`: CRUD, undo, import/export, alert/category management |
| `CooldownViewerSettingsDataProvider.lua` | 342 | `CooldownViewerSettingsDataProviderMixin`: display data build, category filtering, reorder |
| `CooldownViewerItemData.lua` | 471 | `CooldownViewerItemDataMixin`: spell ID resolution, aura/totem data, tooltip, alert types |
| `CooldownViewerSettingsDialogs.lua` | 58 | `CooldownViewerBaseDialogMixin`, `CooldownViewerImportLayoutDialogMixin`: import/export dialogs |
| `CooldownViewerSettingsDialogs.xml` | 14 | XML for settings dialog frames |
| `CooldownViewerSettings.lua` | 1717 | Main settings UI: tabs, search, drag-drop reordering, category containers, context menus |
| `CooldownViewerSettings.xml` | 261 | XML for settings frame (399x609, ButtonFrameTemplate, tab bar) |
| `CooldownViewer.lua` | 2043 | Core: all item/viewer mixins, cooldown caching, GCD detection, range check, icon coloring |
| `PandemicAlertAnimation.xml` | 82 | Pandemic animation templates (icon pulse 3-texture, bar rotation) |
| `CooldownViewer.xml` | 334 | XML: 4 viewer frames, 4 item templates, debuff border template |

### Related Files Outside the Addon

| File | Description |
|------|-------------|
| .../CooldownViewerDocumentation.lua | C_CooldownViewer API docs |
| .../CooldownViewerConstantsDocumentation.lua | Enum definitions |
| .../EditModeManagerConstantsDocumentation.lua | Edit Mode enums (VisibleSetting, BarContent) |
| .../FrameAPICooldownDocumentation.lua | CooldownFrame widget API |
| .../Cooldown.lua | CooldownFrame_Set, _Clear, _SetDisplayAsPercentage |

All paths are relative to `Blizzard_APIDocumentationGenerated/` except `Cooldown.lua` which is in `Blizzard_FrameXMLUtil/`.

---

## Taint Avoidance for CooldownViewer Frames

CooldownViewer frames inherit `EditModeCooldownViewerSystemMixin`, making them implicitly protected. This introduces unique taint challenges for addon code that modifies these frames. This section documents critical pitfalls and their solutions.

### Addon-Owned Container Pattern for Combat-Safe Operations

Because viewer frames are implicitly protected, `SetWidth`, `SetHeight`, and `SetPoint` are restricted during combat. Solution: create **unprotected container frames** and anchor viewer icon children to them via `SetPoint` **without reparenting** the viewer.

**Critical:** Reparenting a protected frame to a container makes the container implicitly protected per WoW's taint rules.

```lua
-- CORRECT: Anchor children to container WITHOUT reparenting
local container = CreateFrame("Frame", nil, UIParent)
container:SetSize(400, 100)
container:SetPoint("CENTER")

-- Child remains parented to viewer, but positioned relative to container:
child:SetPoint("BOTTOMLEFT", container, "BOTTOMLEFT", x, y)

-- WRONG: Never reparent the viewer or its children
-- viewer:SetParent(container)   -- Makes container implicitly protected!
-- child:SetParent(container)    -- Same problem!
```

### Never Nil Out Frame Table Properties

Any insecure write to a Blizzard frame table taints the key. For example, nilling out `DebuffBorder` causes `RefreshIconBorder` to read the tainted `nil` during secure `RefreshData()` execution, contaminating the secure context. This propagation chain can be devastating:

1. Addon sets `child.DebuffBorder = nil` (tainted write)
2. Blizzard's `RefreshIconBorder` reads `child.DebuffBorder` in secure context
3. Taint propagates through child iteration to `CheckAllowOnCooldown`
4. The file-local `wasOnGCDLookup` table becomes permanently tainted
5. All subsequent cooldown operations produce "attempted to index a forbidden table" warnings

```lua
-- WRONG: Taints the frame table key
child.DebuffBorder = nil

-- CORRECT: Hide visually without modifying the table
child.DebuffBorder:SetAlpha(0)
```

This rule applies to **all** Blizzard frame properties: `DebuffBorder`, `PandemicIcon`, `IconOverlay`, etc.

### Never Hook Child Mixin Methods

Hooks on child-level methods like `OnActiveStateChanged`, `OnUnitAuraAddedEvent`, or `NeedsAddedAuraUpdate` inject insecure addon code into Blizzard's secure aura-processing chain. This taints sibling `spellID` / `sourceUnit` comparisons in `NeedsAddedAuraUpdate`.

**Safe to hook (viewer-level):**
- `Layout`
- `RefreshLayout`
- `OnUnitAura`
- `ProcessCooldownData`

**Risky to hook (child-level):**
- `OnActiveStateChanged`
- `OnUnitAuraAddedEvent`
- `NeedsAddedAuraUpdate`
- `RefreshData`
- `RefreshIconBorder`
- `RefreshIconDesaturation`

> **Practical note:** Popular addons like CooldownManagerCentered hook `OnActiveStateChanged`, `OnUnitAuraAddedEvent`, and `OnUnitAuraRemovedEvent` on CDM child frames successfully using `hooksecurefunc`. While these hooks carry theoretical taint risk (especially in secure execution contexts), they work in practice for post-processing operations. The TaintLess library can mitigate some taint propagation. Use with caution and test thoroughly.

### Coalesced Layout via Dirty-Flag OnUpdate

Instead of running layout on every event or hook, create a permanent watcher frame whose `OnUpdate` checks a dirty flag. Multiple events and hooks all just set `layoutDirty = true`. The OnUpdate fires once per render frame:

```lua
local layoutDirty = false

-- Multiple triggers all just set the flag
hooksecurefunc(viewer, "RefreshLayout", function()
    layoutDirty = true
end)

hooksecurefunc(viewer, "OnUnitAura", function()
    layoutDirty = true
end)

-- Single watcher processes once per frame
local watcher = CreateFrame("Frame")
watcher:SetScript("OnUpdate", function()
    if not layoutDirty then return end
    layoutDirty = false
    addon:ApplyLayout(viewer)
end)
```

This avoids redundant layout passes when multiple events fire in the same frame.

### Viewer Alpha as Hide/Show Signal

Use `SetIgnoreParentAlpha(true)` on all viewer icon children so the viewer's alpha does NOT cascade to them. Manage per-icon alpha directly. Use the viewer's alpha only as a binary signal (0=hidden, 1=shown):

```lua
-- Set up all children to ignore parent alpha
for _, child in ipairs(viewer.activeFrames) do
    child:SetIgnoreParentAlpha(true)
end

-- Monitor viewer alpha as a visibility signal
hooksecurefunc(viewer, "SetAlpha", function(self, alpha)
    if alpha == 0 then
        addon:OnViewerHidden(self)
    else
        addon:OnViewerShown(self)
    end
end)
```

### SetAlpha(0) Not Hide() for Cosmetic Suppression

Always use `SetAlpha(0)` instead of `Hide()` for suppressing Blizzard-managed CooldownViewer elements. `Hide()` can trigger combat lockdown issues on protected frames, and also disrupts Blizzard's internal show/hide state tracking:

```lua
-- CORRECT:
child.IconOverlay:SetAlpha(0)
child.DebuffBorder:SetAlpha(0)
child.PandemicIcon:SetAlpha(0)
child.Cooldown:SetAlpha(0)

-- WRONG: Can cause combat lockdown errors
-- child.IconOverlay:Hide()
-- child.DebuffBorder:Hide()
```

This pattern applies to overlays, borders, pandemic icons, cooldown frames, and any other sub-element of a protected CooldownViewer child.
