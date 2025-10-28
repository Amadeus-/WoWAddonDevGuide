# Core UI Systems - Blizzard Addons

Essential UI framework components that form the foundation of WoW's interface.

## Core Framework

### Blizzard_SharedXML
**Most important addon for development - contains all shared UI components**
- Mixins, utilities, base templates
- Frame systems, layout frames, scroll boxes
- Data providers, event utilities
- NineSlice, backdrop, animation templates

**Key Files:**
- `EventUtil.lua` - Event handling utilities
- `TableBuilder.lua` - Table/grid construction
- `DataProvider.lua` - Data provider pattern
- `LayoutFrame.lua` - Layout frame system
- `ScrollBox.lua` - Modern scroll system
- `NineSlice.lua` - Frame borders
- `SortUtil.lua` - Sorting utilities

### Blizzard_FrameXML
**Base frame XML definitions** (if present in version)

## Action Bars & Buttons

### Blizzard_ActionBar
**Complete action button implementation**
- Action button templates
- Cooldown tracking
- Range checking
- Keybinding display
- Pet action bar, stance bar, possess bar

**Variants:**
- `Blizzard_ActionBar_Mainline.toc`
- `Blizzard_ActionBar_Mists.toc`

**Key Files:**
- `Shared\ActionButton.lua` - Core button logic
- `Mainline\ActionButtonTemplate.xml` - Button template
- `Shared\SpellFlyout.lua` - Spell flyout system

### Blizzard_ActionBarController
**Action bar management**
- Bar visibility
- Combat state handling
- Vehicle UI transitions

### Blizzard_ActionStatus
**Action feedback system**
- Spell alerts
- Proc notifications

## Unit Frames

### Blizzard_UnitFrame
**Player, target, party, raid frames**
- Health/mana bars
- Auras (buffs/debuffs)
- Cast bars
- Target of target

**Key Files:**
- `UnitFrame.lua` - Core unit frame logic
- `ArenaFrame.lua` - Arena frames
- `PartyFrame.lua` - Party frames
- `RaidFrame.lua` - Raid frames

### Blizzard_CompactRaidFrames
**Compact raid frame system**
- Modern raid frames
- Sorting and filtering
- Indicators and debuffs

### Blizzard_BuffFrame
**Buff/debuff display**
- Aura templates
- Timer management
- Debuff type coloring

**Key Files:**
- `BuffFrameTemplates.xml` - Aura button templates
- `BuffFrame.lua` - Buff frame logic

## Tooltip System

### Blizzard_UIPanels_Game
**Contains GameTooltip and related systems**
- Tooltip formatting
- Comparison tooltips
- Hyperlink handling

### Blizzard_TooltipInfo
**Tooltip data handling** (if present)

## Layout & Positioning

### Blizzard_UIParent
**Root UI container**
- UI scaling
- Screen layout
- Panel management

### Blizzard_EditMode
**Edit mode for UI customization** (11.0+)
- Drag-and-drop UI elements
- Grid snapping
- Layout presets

## Input & Keybindings

### Blizzard_BindingUI
**Keybinding interface**
- Key assignment
- Conflict detection

### Blizzard_ClickBindingUI
**Click-to-cast bindings** (if present)

## Secure Templates

### Blizzard_SecureTemplates
**Combat-safe templates** (if present as separate addon)
- Secure action buttons
- Secure handlers
- Combat lockdown handling

## Accessibility

### Blizzard_AccessibilityTemplates
**Accessibility features**
- Text scaling
- Color blind mode
- UI element templates

## Deprecated / Legacy

### Blizzard_ClassicBindings
**Classic era bindings** (Classic/Cataclysm only)

### Blizzard_ClassicUITemplate
**Classic-style UI** (if present)

---

## Usage Notes

**For addon development, START with:**
1. `Blizzard_SharedXML` - Learn mixins, utilities, patterns
2. `Blizzard_ActionBar` - See complete button implementation
3. `Blizzard_BuffFrame` - See aura handling
4. `Blizzard_UnitFrame` - See unit data handling

**All paths relative to:**
```
D:\Games\World of Warcraft\_retail_\Interface\+wow-ui-source+ (11.2.7)\Interface\AddOns\
```

---

**Version:** 1.0 - Based on WoW 11.2.7 (The War Within)
**Last Updated:** 2025-10-19
