# WoW Housing System API Guide

## Table of Contents
1. [Overview](#overview)
2. [Core Namespaces Reference](#core-namespaces-reference)
3. [Key Events](#key-events)
4. [CVars](#cvars)
5. [Practical Examples](#practical-examples)
6. [Best Practices](#best-practices)
7. [Related Systems](#related-systems)

---

## Overview

The Player Housing system was introduced in **Patch 11.2.7** (The War Within) and significantly expanded in **Patch 12.0.0** (Midnight). It allows players to own, customize, and decorate personal houses within shared neighborhoods.

### Key Concepts

**Houses**
- Personal dwellings that players can own and customize
- Located on plots within neighborhoods
- Have interior and exterior spaces
- Level up through "House Favor" (house XP) to unlock more features

**Neighborhoods**
- Shared community spaces containing multiple plots
- Can be Charter-based (friends) or Guild-based
- Have managers and residents with different permissions
- Players can visit each other's houses within neighborhoods

**Plots**
- Individual land parcels within neighborhoods
- Can be purchased at cornerstone NPCs
- Houses are built on plots
- Exterior decor is placed on plots

**Decor**
- Furniture and decorative items
- Stored in "House Chest" storage
- Has budget costs (different sizes cost different amounts)
- Can be dyed with color customizations

**Editor Modes**
- **Basic Mode** - Simple drag-and-drop placement
- **Expert Mode** - Precision placement with gizmos for translate/rotate/scale
- **Layout Mode** - Room arrangement and floor management
- **Customize Mode** - Applying dyes, themes, and wallpapers
- **Cleanup Mode** - Bulk removal of decor
- **Exterior Customization Mode** - House exterior and fixtures

---

## Core Namespaces Reference

### C_Housing (45+ functions)
*Main housing control and information API*

#### Location Checks
```lua
-- Check if player is inside any house
local isInside = C_Housing.IsInsideHouse()

-- Check if inside house OR on a plot
local isInHouseOrPlot = C_Housing.IsInsideHouseOrPlot()

-- Check if inside player's own house
local isInOwnHouse = C_Housing.IsInsideOwnHouse()

-- Check if on a plot (exterior)
local isOnPlot = C_Housing.IsInsidePlot()

-- Check if on a neighborhood map
local isOnNeighborhoodMap = C_Housing.IsOnNeighborhoodMap()
```

#### House Information
```lua
-- Get info about current house (returns HouseInfo or nil)
local houseInfo = C_Housing.GetCurrentHouseInfo()
-- HouseInfo fields:
--   plotID, houseName, ownerName, plotCost, neighborhoodName,
--   moveOutTime, plotReserved, neighborhoodGUID, houseGUID

-- Get current neighborhood GUID
local neighborhoodGUID = C_Housing.GetCurrentNeighborhoodGUID()

-- Get player's owned houses (triggers PLAYER_HOUSE_LIST_UPDATED event)
C_Housing.GetPlayerOwnedHouses()

-- Get another player's houses
C_Housing.GetOthersOwnedHouses(playerGUID, bnetID, isInPlayersGuild)

-- Get max house level
local maxLevel = C_Housing.GetMaxHouseLevel()

-- Get house level favor (XP) for a specific level
local favor = C_Housing.GetHouseLevelFavorForLevel(level)

-- Get tracked house GUID
local trackedGUID = C_Housing.GetTrackedHouseGuid()

-- Set tracked house
C_Housing.SetTrackedHouseGuid(houseGUID) -- or nil to clear
```

#### Teleportation and Visiting
```lua
-- Teleport to player's home
C_Housing.TeleportHome(neighborhoodGUID, houseGUID, plotID)

-- Visit another player's house
C_Housing.VisitHouse(neighborhoodGUID, houseGUID, plotID)

-- Leave current house
C_Housing.LeaveHouse()

-- Return to previous location after visiting
C_Housing.ReturnAfterVisitingHouse()

-- Get visit cooldown info
local cooldownInfo = C_Housing.GetVisitCooldownInfo() -- SpellCooldownInfo or nothing
```

#### Neighborhood and Charter Functions
```lua
-- Create a guild neighborhood
C_Housing.CreateGuildNeighborhood(neighborhoodName)

-- Create a neighborhood charter (friend-based)
C_Housing.CreateNeighborhoodCharter(neighborhoodName)

-- Edit existing charter
C_Housing.EditNeighborhoodCharter(neighborhoodName)

-- Validate neighborhood name (triggers NEIGHBORHOOD_NAME_VALIDATED)
C_Housing.ValidateNeighborhoodName(neighborhoodName)

-- Check if player can edit charter
local canEdit = C_Housing.CanEditCharter()

-- Accept/decline neighborhood ownership transfer
C_Housing.AcceptNeighborhoodOwnership()
C_Housing.DeclineNeighborhoodOwnership()

-- Rename neighborhood
C_Housing.TryRenameNeighborhood(neighborhoodName)
```

#### House Finder Functions
```lua
-- Request available neighborhoods
C_Housing.HouseFinderRequestNeighborhoods()

-- Request specific neighborhood data
C_Housing.RequestHouseFinderNeighborhoodData(neighborhoodGuid, neighborhoodName)

-- Request plot reservation and teleport
C_Housing.HouseFinderRequestReservationAndPort(neighborhoodGuid, plotID)

-- Decline neighborhood invitation
C_Housing.HouseFinderDeclineNeighborhoodInvitation()

-- Search friend neighborhoods
local isValid = C_Housing.SearchBNetFriendNeighborhoods(bnetName)
local isValid = C_Housing.SearchBNetFriendNeighborhoodsByID(bnetID)
```

#### Service Availability
```lua
-- Check if housing service is enabled
local isAvailable = C_Housing.IsHousingServiceEnabled()

-- Check if housing market is enabled
local isEnabled = C_Housing.IsHousingMarketEnabled()

-- Check if housing market shop is enabled
local isShopEnabled = C_Housing.IsHousingMarketShopEnabled()

-- Check if player has housing expansion access
local hasAccess = C_Housing.HasHousingExpansionAccess()
```

#### House Settings
```lua
-- Get housing access flags
local flags = C_Housing.GetHousingAccessFlags() -- HouseSettingFlags

-- Save house settings
C_Housing.SaveHouseSettings(playerGUID, accessFlags)

-- Relinquish (give up) a house
C_Housing.RelinquishHouse(houseGUID)

-- Get refund amount for current house
local refund = C_Housing.GetCurrentHouseRefundAmount()
```

---

### C_HouseEditor (9 functions)
*Editor mode management*

```lua
-- Enter house editor (default mode)
local result = C_HouseEditor.EnterHouseEditor()
-- Returns HousingResult - Success means entering or already active

-- Leave house editor
C_HouseEditor.LeaveHouseEditor()

-- Activate specific editor mode
local result = C_HouseEditor.ActivateHouseEditorMode(editMode)
-- editMode: Enum.HouseEditorMode (None, BasicDecor, ExpertDecor, Layout,
--           Customize, Cleanup, ExteriorCustomization)

-- Get current active mode
local editMode = C_HouseEditor.GetActiveHouseEditorMode()

-- Check if editor is active (any mode)
local isActive = C_HouseEditor.IsHouseEditorActive()

-- Check if specific mode is active
local isModeActive = C_HouseEditor.IsHouseEditorModeActive(editMode)

-- Get editor availability
local result = C_HouseEditor.GetHouseEditorAvailability()

-- Get specific mode availability
local result = C_HouseEditor.GetHouseEditorModeAvailability(editMode)

-- Check if editor status is available (ready to process)
local isReady = C_HouseEditor.IsHouseEditorStatusAvailable()
```

**HouseEditorMode Enum Values:**
```lua
Enum.HouseEditorMode = {
    None = 0,
    BasicDecor = 1,
    ExpertDecor = 2,
    Layout = 3,
    Customize = 4,
    Cleanup = 5,
    ExteriorCustomization = 6,
}
```

---

### C_HouseExterior (14 functions)
*Exterior customization - house type, size, and fixtures*

```lua
-- Get current house exterior size
local size = C_HouseExterior.GetCurrentHouseExteriorSize() -- HousingFixtureSize or nil

-- Get current house exterior type
local typeID, typeName = C_HouseExterior.GetCurrentHouseExteriorType()

-- Get available size options
local options = C_HouseExterior.GetHouseExteriorSizeOptions()
-- Returns HouseExteriorSizeOptionsInfo with selectedSize and options table

-- Get available type options
local options = C_HouseExterior.GetHouseExteriorTypeOptions()
-- Returns HouseExteriorTypeOptionsInfo with selectedExteriorType and options table

-- Set house exterior size
C_HouseExterior.SetHouseExteriorSize(size) -- HousingFixtureSize enum

-- Set house exterior type
C_HouseExterior.SetHouseExteriorType(houseExteriorTypeID)

-- Fixture management
local info = C_HouseExterior.GetSelectedFixturePointInfo() -- HousingFixturePointInfo or nil
local hasSelection = C_HouseExterior.HasSelectedFixturePoint()
local hasHovered = C_HouseExterior.HasHoveredFixture()

-- Get core fixture options (roof, door, etc.)
local info = C_HouseExterior.GetCoreFixtureOptionsInfo(coreFixtureType) -- HousingFixtureType

-- Select fixture
C_HouseExterior.SelectFixtureOption(fixtureID)
C_HouseExterior.SelectCoreFixtureOption(fixtureID)

-- Remove fixture from selected point
C_HouseExterior.RemoveFixtureFromSelectedPoint()

-- Cancel exterior editing
C_HouseExterior.CancelActiveExteriorEditing()
```

**HousingFixtureType Enum:**
```lua
Enum.HousingFixtureType = {
    None = 0,
    Base = 9,
    Roof = 10,
    Door = 11,
    Window = 12,
    RoofDetail = 13,
    RoofWindow = 14,
    Tower = 15,
    Chimney = 16,
}
```

---

### C_HousingDecor (27 functions)
*Decor placement and management - shared across modes*

#### Selection and Hover State
```lua
-- Check if decor is currently selected
local isSelected = C_HousingDecor.IsDecorSelected()

-- Get selected decor info
local info = C_HousingDecor.GetSelectedDecorInfo() -- HousingDecorInstanceInfo or nil

-- Check if hovering over decor
local isHovering = C_HousingDecor.IsHoveringDecor()

-- Get hovered decor info
local info = C_HousingDecor.GetHoveredDecorInfo()

-- Get decor instance info by GUID
local info = C_HousingDecor.GetDecorInstanceInfoForGUID(decorGUID)
```

**HousingDecorInstanceInfo Structure:**
```lua
-- Fields:
--   decorGUID (WOWGUID)
--   decorID (number)
--   name (string)
--   isLocked (bool) - being edited by someone else
--   canBeCustomized (bool) - supports dyes
--   canBeRemoved (bool) - can be returned to chest
--   isAllowedOutdoors (bool)
--   isAllowedIndoors (bool)
--   isRefundable (bool)
--   dyeSlots (table of HousingDecorDyeSlot)
--   dataTagsByID (table)
--   size (HousingCatalogEntrySize)
```

#### Budget and Placement Counts
```lua
-- Get max placement budget
local maxBudget = C_HousingDecor.GetMaxPlacementBudget()

-- Check if budget system is active
local hasBudget = C_HousingDecor.HasMaxPlacementBudget()

-- Get spent budget
local spent = C_HousingDecor.GetSpentPlacementBudget()

-- Get number of placed decor items
local numPlaced = C_HousingDecor.GetNumDecorPlaced()
```

#### Decor Operations
```lua
-- Remove currently selected decor (return to chest)
C_HousingDecor.RemoveSelectedDecor()

-- Cancel active editing
C_HousingDecor.CancelActiveEditing()

-- Commit decor movement
C_HousingDecor.CommitDecorMovement()

-- Get all placed decor (RESTRICTED - may be expensive)
local placedDecor = C_HousingDecor.GetAllPlacedDecor()
-- Returns table of HousingDecorInstanceListEntry: {decorGUID, name}

-- Remove placed decor by GUID (RESTRICTED)
C_HousingDecor.RemovePlacedDecorEntry(decorGUID)

-- Set placed decor hovered/selected (RESTRICTED)
C_HousingDecor.SetPlacedDecorEntryHovered(decorGUID, hovered)
C_HousingDecor.SetPlacedDecorEntrySelected(decorGUID, selected)
```

#### Grid and Helpers
```lua
-- Grid visibility
local isVisible = C_HousingDecor.IsGridVisible()
C_HousingDecor.SetGridVisible(visible)

-- Get decor name/icon by ID
local name = C_HousingDecor.GetDecorName(decorID)
local icon = C_HousingDecor.GetDecorIcon(decorID)

-- House exterior hover checks
local isHovered = C_HousingDecor.IsHouseExteriorHovered()
local isDoorHovered = C_HousingDecor.IsHouseExteriorDoorHovered()
```

#### Preview State
```lua
-- Preview mode for catalog shopping
C_HousingDecor.EnterPreviewState()
C_HousingDecor.ExitPreviewState()
local isPreview = C_HousingDecor.IsPreviewState()
local numPreview = C_HousingDecor.GetNumPreviewDecor()
local isDisabled = C_HousingDecor.IsModeDisabledForPreviewState(mode)
```

---

### C_HousingBasicMode (22 functions)
*Basic editing mode - drag and drop placement*

#### Selection and Hover
```lua
-- Decor selection
local isSelected = C_HousingBasicMode.IsDecorSelected()
local info = C_HousingBasicMode.GetSelectedDecorInfo()

-- Hover state
local isHovering = C_HousingBasicMode.IsHoveringDecor()
local info = C_HousingBasicMode.GetHoveredDecorInfo()

-- House exterior
local isHouseHovered = C_HousingBasicMode.IsHouseExteriorHovered()
local isHouseSelected = C_HousingBasicMode.IsHouseExteriorSelected()
```

#### Placing New Decor
```lua
-- Start placing from catalog
C_HousingBasicMode.StartPlacingNewDecor(catalogEntryID)

-- Start placing preview decor
C_HousingBasicMode.StartPlacingPreviewDecor(decorRecordID, bundleCatalogShopProductID)

-- Check if placing
local isPlacing = C_HousingBasicMode.IsPlacingNewDecor()

-- Finish placing
C_HousingBasicMode.FinishPlacingNewDecor()
```

#### Movement and Rotation
```lua
-- Commit changes
C_HousingBasicMode.CommitDecorMovement()
C_HousingBasicMode.CommitHouseExteriorPosition()

-- Cancel
C_HousingBasicMode.CancelActiveEditing()

-- Remove
C_HousingBasicMode.RemoveSelectedDecor()

-- Rotation (degrees)
C_HousingBasicMode.RotateDecor(rotDegrees)
C_HousingBasicMode.RotateHouseExterior(rotDegrees)
```

#### Grid and Free Place
```lua
-- Grid snap
local isEnabled = C_HousingBasicMode.IsGridSnapEnabled()
C_HousingBasicMode.SetGridSnapEnabled(enabled)

-- Grid visibility
local isVisible = C_HousingBasicMode.IsGridVisible()
C_HousingBasicMode.SetGridVisible(visible)

-- Free place (ignore collisions)
local isEnabled = C_HousingBasicMode.IsFreePlaceEnabled()
C_HousingBasicMode.SetFreePlaceEnabled(enabled)
```

---

### C_HousingExpertMode (18 functions)
*Expert editing mode - precision placement with gizmos*

#### Selection and State
```lua
-- Decor
local isSelected = C_HousingExpertMode.IsDecorSelected()
local info = C_HousingExpertMode.GetSelectedDecorInfo()

-- Hover
local isHovering = C_HousingExpertMode.IsHoveringDecor()
local info = C_HousingExpertMode.GetHoveredDecorInfo()

-- House exterior
local isHouseHovered = C_HousingExpertMode.IsHouseExteriorHovered()
local isHouseSelected = C_HousingExpertMode.IsHouseExteriorSelected()

-- Grid
local isVisible = C_HousingExpertMode.IsGridVisible()
C_HousingExpertMode.SetGridVisible(visible)
```

#### Precision Submodes
```lua
-- Get current submode
local submode = C_HousingExpertMode.GetPrecisionSubmode()
-- Returns HousingPrecisionSubmode or nil

-- Set submode
C_HousingExpertMode.SetPrecisionSubmode(subMode)

-- Check submode restrictions
local restriction = C_HousingExpertMode.GetPrecisionSubmodeRestriction(subMode)
```

**HousingPrecisionSubmode Enum:**
```lua
Enum.HousingPrecisionSubmode = {
    Translate = 0,  -- Move
    Rotate = 1,     -- Rotate
    Scale = 2,      -- Scale
}
```

#### Precision Controls
```lua
-- Incremental changes (hold for continuous)
C_HousingExpertMode.SetPrecisionIncrementingActive(incrementType, active)

-- Select next rotation axis (X -> Y -> Z -> X...)
C_HousingExpertMode.SelectNextRotationAxis()

-- Reset transforms
C_HousingExpertMode.ResetPrecisionChanges(activeSubmodeOnly)
-- If true, only resets current submode; false resets all
```

**HousingIncrementType Enum:**
```lua
Enum.HousingIncrementType = {
    Left = 1,
    Right = 2,
    Forward = 4,
    Back = 8,
    Up = 16,
    Down = 32,
    RotateLeft = 64,
    RotateRight = 128,
    ScaleUp = 256,
    ScaleDown = 512,
}
```

#### Commit/Cancel
```lua
C_HousingExpertMode.CommitDecorMovement()
C_HousingExpertMode.CommitHouseExteriorPosition()
C_HousingExpertMode.CancelActiveEditing()
C_HousingExpertMode.RemoveSelectedDecor()
```

---

### C_HousingCustomizeMode (28 functions)
*Customization mode - dyes, themes, wallpapers*

#### Target Selection
```lua
-- Decor
local isSelected = C_HousingCustomizeMode.IsDecorSelected()
local info = C_HousingCustomizeMode.GetSelectedDecorInfo()
local isHovering = C_HousingCustomizeMode.IsHoveringDecor()
local info = C_HousingCustomizeMode.GetHoveredDecorInfo()

-- Room components
local isSelected = C_HousingCustomizeMode.IsRoomComponentSelected()
local info = C_HousingCustomizeMode.GetSelectedRoomComponentInfo()
local isHovering = C_HousingCustomizeMode.IsHoveringRoomComponent()
local info = C_HousingCustomizeMode.GetHoveredRoomComponentInfo()

-- House exterior door
local isDoorHovered = C_HousingCustomizeMode.IsHouseExteriorDoorHovered()

-- Deselect
C_HousingCustomizeMode.ClearTargetRoomComponent()
C_HousingCustomizeMode.CancelActiveEditing()
```

#### Dye System
```lua
-- Apply dye to selected decor (preview)
C_HousingCustomizeMode.ApplyDyeToSelectedDecor(dyeSlotID, dyeColorID)
-- Pass nil for dyeColorID to clear the slot

-- Get preview dyes
local previewDyes = C_HousingCustomizeMode.GetPreviewDyesOnSelectedDecor()
-- Returns table of {dyeColorID, dyeSlotID}

-- Clear all preview dyes
C_HousingCustomizeMode.ClearDyesForSelectedDecor()

-- Commit dyes (save)
local hasChanges = C_HousingCustomizeMode.CommitDyesForSelectedDecor()

-- Get dye costs
local numToSpend = C_HousingCustomizeMode.GetNumDyesToSpendOnSelectedDecor()
local numToRemove = C_HousingCustomizeMode.GetNumDyesToRemoveOnSelectedDecor()

-- Recently used dyes
local recentDyes = C_HousingCustomizeMode.GetRecentlyUsedDyes()
```

#### Themes (Styles)
```lua
-- Apply theme to room
C_HousingCustomizeMode.ApplyThemeToRoom(themeSetID)

-- Apply theme to selected component only
C_HousingCustomizeMode.ApplyThemeToSelectedRoomComponent(themeSetID)

-- Get theme info
local name = C_HousingCustomizeMode.GetThemeSetInfo(themeSetID)

-- Recently used themes
local recentThemes = C_HousingCustomizeMode.GetRecentlyUsedThemeSets()
```

#### Wallpapers (Materials/Textures)
```lua
-- Apply wallpaper to all walls
C_HousingCustomizeMode.ApplyWallpaperToAllWalls(roomComponentTextureRecID)

-- Apply to selected component
C_HousingCustomizeMode.ApplyWallpaperToSelectedRoomComponent(roomComponentTextureRecID)

-- Get available wallpapers for component type
local wallpapers = C_HousingCustomizeMode.GetWallpapersForRoomComponentType(type)
-- Returns table of {name, roomComponentTextureRecID}

-- Recently used wallpapers
local recentWallpapers = C_HousingCustomizeMode.GetRecentlyUsedWallpapers()
```

#### Room Component Variants
```lua
-- Check variant support
local supported = C_HousingCustomizeMode.RoomComponentSupportsVariant(componentID, variant)

-- Set ceiling type
C_HousingCustomizeMode.SetRoomComponentCeilingType(roomGUID, componentID, ceilingType)
-- ceilingType: Enum.HousingRoomComponentCeilingType.Flat or .Vaulted

-- Set door type
C_HousingCustomizeMode.SetRoomComponentDoorType(roomGUID, componentID, doorType)
-- doorType: Enum.HousingRoomComponentDoorType.None, .Doorway, or .Threshold
```

---

### C_HousingLayout (32 functions)
*Room layout management - room placement and floor management*

#### Room Budget
```lua
-- Get room placement budget
local budget = C_HousingLayout.GetRoomPlacementBudget()
local hasBudget = C_HousingLayout.HasRoomPlacementBudget()
local spent = C_HousingLayout.GetSpentPlacementBudget()

-- Get number of active rooms
local numRooms = C_HousingLayout.GetNumActiveRooms()
```

#### Selection State
```lua
-- Room selection
local hasRoom = C_HousingLayout.HasSelectedRoom()
local roomGUID = C_HousingLayout.GetSelectedRoom() -- may return nothing

-- Door selection
local hasDoor = C_HousingLayout.HasSelectedDoor()
local doorComponentID, roomGUID = C_HousingLayout.GetSelectedDoor() -- may return nothing

-- Floorplan selection (from chest)
local hasFloorplan = C_HousingLayout.HasSelectedFloorplan()
local roomID = C_HousingLayout.GetSelectedFloorplan()

-- Any selections
local hasAny = C_HousingLayout.HasAnySelections()
```

#### Selection Control
```lua
-- Select floorplan from chest
C_HousingLayout.SelectFloorplan(roomID)
C_HousingLayout.DeselectFloorplan()

-- Deselect room or door
C_HousingLayout.DeselectRoomOrDoor()
```

#### Room Operations
```lua
-- Remove room (return to chest)
C_HousingLayout.RemoveRoom(roomGUID)

-- Rotate room
C_HousingLayout.RotateRoom(roomGUID, isLeft)
C_HousingLayout.RotateFocusedRoom(isLeft) -- selected or dragged

-- Check if base room (cannot be removed)
local isBase = C_HousingLayout.IsBaseRoom(roomGUID)

-- Check for stairs
local hasStairs = C_HousingLayout.HasStairs(roomRecordID)
```

#### Dragging
```lua
-- Check drag state
local isDragging, isAccessibleDrag = C_HousingLayout.IsDraggingRoom()

-- Control drag
C_HousingLayout.StartDrag()
C_HousingLayout.StopDrag()
C_HousingLayout.StopDraggingRoom()

-- Move dragged room to connection point
C_HousingLayout.MoveDraggedRoom(sourceDoorIndex, destRoom, destDoorIndex)
```

#### Floor Management
```lua
-- Get/set viewed floor
local floor = C_HousingLayout.GetViewedFloor()
C_HousingLayout.SetViewedFloor(floor)

-- Check if any rooms on floor
local anyRooms = C_HousingLayout.AnyRoomsOnFloor(floor)

-- Stair direction confirmation
C_HousingLayout.ConfirmStairChoice(direction) -- HousingLayoutStairDirection or nil to cancel

-- Get stairwell room count for selected
local count = C_HousingLayout.GetSelectedStairwellRoomCount()
```

#### Connection Validation
```lua
-- Check if connection is valid
local canPlace = C_HousingLayout.HasValidConnection(roomGUID, componentID, roomId)
```

#### Camera Control
```lua
-- Move layout camera
C_HousingLayout.MoveLayoutCamera(direction, isPressed)
-- direction: Enum.HousingLayoutCameraDirection (None, Up, Down, Left, Right)

-- Zoom
local changed = C_HousingLayout.ZoomLayoutCamera(zoomIn)
```

#### Utility
```lua
-- Cancel all layout editing
C_HousingLayout.CancelActiveLayoutEditing()
```

---

### C_HousingCatalog (22 functions)
*Decor catalog and storage management*

#### Entry Information
```lua
-- Get catalog entry info by ID
local info = C_HousingCatalog.GetCatalogEntryInfo(entryID)
-- entryID is HousingCatalogEntryID structure

-- Get by item
local info = C_HousingCatalog.GetCatalogEntryInfoByItem(itemInfo, tryGetOwnedInfo)
-- itemInfo: ItemID, name, or link

-- Get by record ID
local info = C_HousingCatalog.GetCatalogEntryInfoByRecordID(entryType, recordID, tryGetOwnedInfo)

-- Get category/subcategory info
local catInfo = C_HousingCatalog.GetCatalogCategoryInfo(categoryID)
local subInfo = C_HousingCatalog.GetCatalogSubcategoryInfo(subcategoryID)
```

**HousingCatalogEntryInfo Structure:**
```lua
-- Key fields:
--   entryID (HousingCatalogEntryID)
--   itemID, name, asset, iconTexture/iconAtlas
--   categoryIDs, subcategoryIDs
--   size (HousingCatalogEntrySize)
--   placementCost, showQuantity, quantity
--   remainingRedeemable, numPlaced
--   isUniqueTrophy, isAllowedOutdoors, isAllowedIndoors
--   canCustomize, isPrefab, quality
--   customizations, dyeIDs
--   marketInfo, firstAcquisitionBonus, sourceText
```

#### Storage Counts
```lua
-- Max decor storage
local max = C_HousingCatalog.GetDecorMaxOwnedCount()

-- Total owned count
local total, exempt = C_HousingCatalog.GetDecorTotalOwnedCount()
-- exempt = decor that doesn't count against limit
```

#### Category Browsing
```lua
-- Search categories
local categoryIDs = C_HousingCatalog.SearchCatalogCategories(searchParams)
local subcategoryIDs = C_HousingCatalog.SearchCatalogSubcategories(searchParams)

-- searchParams: HousingCategorySearchInfo
--   withOwnedEntriesOnly (bool)
--   includeFeaturedCategory (bool)
--   editorModeContext (HouseEditorMode, optional)

-- Get filter tags
local tagGroups = C_HousingCatalog.GetAllFilterTagGroups()
```

#### Featured Content
```lua
-- Get featured entries
local hasEntries = C_HousingCatalog.HasFeaturedEntries()
local bundles = C_HousingCatalog.GetFeaturedBundles()
local decor = C_HousingCatalog.GetFeaturedDecor()
```

#### Bundle Information
```lua
-- Get bundle info
local bundleInfo = C_HousingCatalog.GetBundleInfo(bundleCatalogShopProductID)
-- Returns: price, originalPrice, productID, decorEntries, canPreview
```

#### Entry Management
```lua
-- Check if entry can be destroyed
local canDelete = C_HousingCatalog.CanDestroyEntry(entryID)

-- Destroy entry from storage
C_HousingCatalog.DestroyEntry(entryID, destroyAll)

-- Get refund timestamp
local timestamp = C_HousingCatalog.GetCatalogEntryRefundTimeStampByRecordID(entryType, recordID)
```

#### Catalog Searcher
```lua
-- Create a searcher instance for async filtering
local searcher = C_HousingCatalog.CreateCatalogSearcher()
-- Returns HousingCatalogSearcher frame with methods for filtering
```

#### Preview Cart
```lua
-- Preview cart management
C_HousingCatalog.SetPreviewCartItemShown(decorGUID, shown)
local isShown = C_HousingCatalog.IsPreviewCartItemShown(decorGUID)
C_HousingCatalog.DeletePreviewCartDecor(decorGUID)
C_HousingCatalog.PromotePreviewDecor(decorID, previewDecorGUID)
local limit = C_HousingCatalog.GetCartSizeLimit()
```

#### Market Requests
```lua
C_HousingCatalog.RequestHousingMarketInfoRefresh()
C_HousingCatalog.RequestHousingMarketRefundInfo()
```

---

### C_HousingNeighborhood (23 functions)
*Neighborhood management - roster, invitations, plot purchasing*

#### Neighborhood Info
```lua
-- Get neighborhood name
local name = C_HousingNeighborhood.GetNeighborhoodName()

-- Get plot name
local plotName = C_HousingNeighborhood.GetNeighborhoodPlotName(plotIndex)

-- Get neighborhood map data
local plots = C_HousingNeighborhood.GetNeighborhoodMapData()
-- Returns table of NeighborhoodPlotMapInfo

-- Request updated info
C_HousingNeighborhood.RequestNeighborhoodInfo()

-- Get texture suffix for theming
local suffix = C_HousingNeighborhood.GetCurrentNeighborhoodTextureSuffix()
```

#### Role Checks
```lua
-- Check if player is neighborhood owner
local isOwner = C_HousingNeighborhood.IsNeighborhoodOwner()

-- Check if player is manager
local isManager = C_HousingNeighborhood.IsNeighborhoodManager()

-- Check if in another player's plot
local isInOtherPlot = C_HousingNeighborhood.IsPlayerInOtherPlayersPlot()
```

#### Roster Management (Bulletin Board)
```lua
-- Request roster
C_HousingNeighborhood.RequestNeighborhoodRoster()

-- Request pending invites
C_HousingNeighborhood.RequestPendingNeighborhoodInvites()

-- Invite player
C_HousingNeighborhood.InvitePlayerToNeighborhood(playerName)

-- Cancel invite
C_HousingNeighborhood.CancelInviteToNeighborhood(playerName)

-- Promote to manager
C_HousingNeighborhood.PromoteToManager(playerGUID)

-- Demote to resident
C_HousingNeighborhood.DemoteToResident(playerGUID)

-- Transfer ownership
C_HousingNeighborhood.TransferNeighborhoodOwnership(playerGUID)

-- Evict player
C_HousingNeighborhood.TryEvictPlayer(plotID)

-- Close bulletin board
C_HousingNeighborhood.OnBulletinBoardClosed()
```

#### Cornerstone Functions (Plot Purchase/Move)
```lua
-- Get cornerstone info
local houseInfo = C_HousingNeighborhood.GetCornerstoneHouseInfo()
local neighborhoodInfo = C_HousingNeighborhood.GetCornerstoneNeighborhoodInfo()
local purchaseMode = C_HousingNeighborhood.GetCornerstonePurchaseMode()
-- CornerstonePurchaseMode: Basic, Import, Move

-- Check permissions
local reason = C_HousingNeighborhood.HasPermissionToPurchase()
-- Returns PurchaseHouseDisabledReason

-- Check plot availability
local isAvailable = C_HousingNeighborhood.IsPlotAvailableForPurchase()
local isPlayerOwned = C_HousingNeighborhood.IsPlotOwnedByPlayer()

-- Purchase/move
C_HousingNeighborhood.TryPurchasePlot()
C_HousingNeighborhood.TryMoveHouse()

-- Move costs
local movePrice = C_HousingNeighborhood.GetDiscountedMovePrice()
local cooldown = C_HousingNeighborhood.GetMoveCooldownTime()

-- Previous house identifier (for moves)
local prevID = C_HousingNeighborhood.GetPreviousHouseIdentifier()
```

#### Return After Visit
```lua
local canReturn = C_HousingNeighborhood.CanReturnAfterVisitingHouse()
```

---

### C_HousingCleanupMode (3 functions)
*Cleanup mode - bulk decor removal*

```lua
-- Hover state
local isHovering = C_HousingCleanupMode.IsHoveringDecor()
local info = C_HousingCleanupMode.GetHoveredDecorInfo()

-- Remove selected decor
C_HousingCleanupMode.RemoveSelectedDecor()
```

---

### C_DyeColor (8 functions)
*Dye color system*

```lua
-- Get all dye colors
local dyeColorIDs = C_DyeColor.GetAllDyeColors(ownedColorsOnly)

-- Get dye color info
local info = C_DyeColor.GetDyeColorInfo(dyeColorID)
-- Returns DyeColorDisplayInfo

-- Check if dye is owned
local isOwned = C_DyeColor.IsDyeColorOwned(dyeColorID)

-- Get dye color for item
local dyeColorID = C_DyeColor.GetDyeColorForItem(itemLinkOrID)
local dyeColorID = C_DyeColor.GetDyeColorForItemLocation(itemLocation)

-- Categories
local categoryIDs = C_DyeColor.GetAllDyeColorCategories()
local categoryInfo = C_DyeColor.GetDyeColorCategoryInfo(categoryID)
local dyeIDs = C_DyeColor.GetDyeColorsInCategory(categoryID, ownedColorsOnly)
```

---

## Key Events

### House State Events
```lua
-- House information
"CURRENT_HOUSE_INFO_RECIEVED"      -- Payload: houseInfo (HouseInfo)
"CURRENT_HOUSE_INFO_UPDATED"       -- Payload: houseInfo (HouseInfo)
"HOUSE_INFO_UPDATED"               -- No payload
"PLAYER_HOUSE_LIST_UPDATED"        -- Payload: houseInfos (table of HouseInfo)

-- House level
"HOUSE_LEVEL_CHANGED"              -- Payload: newHouseLevelInfo (HouseLevelInfo or nil)
"HOUSE_LEVEL_FAVOR_UPDATED"        -- Payload: houseLevelFavor (HouseLevelFavor)
"RECEIVED_HOUSE_LEVEL_REWARDS"     -- Payload: level, rewards (table)

-- Plot state
"HOUSE_PLOT_ENTERED"               -- No payload
"HOUSE_PLOT_EXITED"                -- No payload

-- Market
"HOUSING_MARKET_AVAILABILITY_UPDATED"
"HOUSING_SERVICES_AVAILABILITY_UPDATED"

-- New items
"NEW_HOUSING_ITEM_ACQUIRED"        -- Payload: itemType, itemName, icon
```

### Editor Mode Events
```lua
-- Mode changes
"HOUSE_EDITOR_AVAILABILITY_CHANGED"
"HOUSE_EDITOR_MODE_CHANGED"        -- Payload: currentEditMode (HouseEditorMode)
"HOUSE_EDITOR_MODE_CHANGE_FAILURE" -- Payload: result (HousingResult)
```

### Decor Events
```lua
-- Placement
"HOUSING_DECOR_PLACE_SUCCESS"      -- Payload: decorGUID, size, isNew, isPreview
"HOUSING_DECOR_PLACE_FAILURE"      -- Payload: housingResult
"HOUSING_DECOR_REMOVED"            -- Payload: decorGUID
"HOUSE_DECOR_ADDED_TO_CHEST"       -- Payload: decorGUID, decorID

-- Selection
"HOUSING_DECOR_SELECT_RESPONSE"    -- Payload: result (HousingResult)

-- Count changes
"HOUSING_NUM_DECOR_PLACED_CHANGED"

-- Grid
"HOUSING_DECOR_GRID_VISIBILITY_STATUS_CHANGED"  -- Payload: isGridVisible
"HOUSING_DECOR_GRID_SNAP_STATUS_CHANGED"        -- Payload: isGridSnapEnabled
"HOUSING_DECOR_GRID_SNAP_OCCURRED"

-- Preview
"HOUSING_DECOR_PREVIEW_STATE_CHANGED"  -- Payload: isPreviewState
```

### Basic Mode Events
```lua
"HOUSING_BASIC_MODE_HOVERED_TARGET_CHANGED"   -- Payload: hasHoveredTarget, targetType
"HOUSING_BASIC_MODE_SELECTED_TARGET_CHANGED"  -- Payload: hasSelectedTarget, targetType, isPreview
"HOUSING_BASIC_MODE_PLACEMENT_FLAGS_UPDATED"  -- Payload: targetType, activeFlags
"HOUSING_DECOR_FREE_PLACE_STATUS_CHANGED"     -- Payload: isFreePlaceEnabled
```

### Expert Mode Events
```lua
"HOUSING_EXPERT_MODE_HOVERED_TARGET_CHANGED"  -- Payload: hasHoveredTarget, targetType
"HOUSING_EXPERT_MODE_SELECTED_TARGET_CHANGED" -- Payload: hasSelectedTarget, targetType
"HOUSING_EXPERT_MODE_PLACEMENT_FLAGS_UPDATED" -- Payload: targetType, activeFlags
"HOUSING_DECOR_PRECISION_SUBMODE_CHANGED"     -- Payload: activeSubmode
"HOUSING_DECOR_PRECISION_MANIPULATION_STATUS_CHANGED" -- Payload: isManipulatingSelection
"HOUSING_DECOR_PRECISION_MANIPULATION_EVENT"  -- Payload: event (TransformManipulatorEvent)
```

### Customize Mode Events
```lua
"HOUSING_CUSTOMIZE_MODE_HOVERED_TARGET_CHANGED"  -- Payload: hasHoveredTarget, targetType
"HOUSING_CUSTOMIZE_MODE_SELECTED_TARGET_CHANGED" -- Payload: hasSelectedTarget, targetType
"HOUSING_DECOR_CUSTOMIZATION_CHANGED"            -- Payload: decorGUID
"HOUSING_DECOR_DYE_FAILURE"                      -- Payload: decorGUID, housingResult
"HOUSING_ROOM_COMPONENT_CUSTOMIZATION_CHANGED"   -- Payload: roomGUID, componentID
"HOUSING_ROOM_COMPONENT_CUSTOMIZATION_CHANGE_FAILED" -- Payload: roomGUID, componentID, housingResult
```

### Cleanup Mode Events
```lua
"HOUSING_CLEANUP_MODE_HOVERED_TARGET_CHANGED" -- Payload: hasHoveredTarget
"HOUSING_CLEANUP_MODE_TARGET_SELECTED"        -- No payload
```

### Layout Events
```lua
-- Selection
"HOUSING_LAYOUT_ROOM_SELECTION_CHANGED"       -- Payload: hasSelection
"HOUSING_LAYOUT_DOOR_SELECTION_CHANGED"       -- Payload: hasSelection
"HOUSING_LAYOUT_DOOR_SELECTED"                -- Payload: roomGUID, componentID
"HOUSING_LAYOUT_FLOORPLAN_SELECTION_CHANGED"  -- Payload: hasSelection, roomID

-- Room operations
"HOUSING_LAYOUT_ROOM_RECEIVED"                -- Payload: prevNumFloors, currNumFloors, isUpstairs
"HOUSING_LAYOUT_ROOM_REMOVED"
"HOUSING_LAYOUT_ROOM_MOVED"
"HOUSING_LAYOUT_ROOM_RETURNED"
"HOUSING_LAYOUT_ROOM_MOVE_INVALID"
"HOUSING_LAYOUT_ROOM_SNAPPED"

-- Dragging
"HOUSING_LAYOUT_DRAG_TARGET_CHANGED"          -- Payload: isDraggingRoom

-- Floor changes
"HOUSING_LAYOUT_VIEWED_FLOOR_CHANGED"         -- Payload: floor
"HOUSING_LAYOUT_NUM_FLOORS_CHANGED"           -- Payload: prevNumFloors, numFloors

-- Stairs
"SHOW_STAIR_DIRECTION_CONFIRMATION"

-- Pin frames
"HOUSING_LAYOUT_PIN_FRAME_ADDED"              -- Payload: pinFrame
"HOUSING_LAYOUT_PIN_FRAME_RELEASED"           -- Payload: pinFrame
"HOUSING_LAYOUT_PIN_FRAMES_RELEASED"

-- Theme
"HOUSING_LAYOUT_ROOM_COMPONENT_THEME_SET_CHANGED" -- Payload: roomGUID, componentID, newThemeSet, result
```

### Exterior Events
```lua
-- House exterior
"HOUSE_EXTERIOR_POSITION_SUCCESS"
"HOUSE_EXTERIOR_POSITION_FAILURE"             -- Payload: housingResult
"HOUSE_EXTERIOR_TYPE_UNLOCKED"                -- Payload: fixtureID
"HOUSING_SET_EXTERIOR_HOUSE_SIZE_RESPONSE"    -- Payload: result
"HOUSING_SET_EXTERIOR_HOUSE_TYPE_RESPONSE"    -- Payload: result

-- Fixtures
"HOUSING_CORE_FIXTURE_CHANGED"                -- Payload: coreFixtureType
"HOUSING_FIXTURE_HOVER_CHANGED"               -- Payload: anyHovered
"HOUSING_FIXTURE_POINT_SELECTION_CHANGED"     -- Payload: hasSelection
"HOUSING_FIXTURE_UNLOCKED"                    -- Payload: fixtureID
"HOUSING_SET_FIXTURE_RESPONSE"                -- Payload: result

-- Pin frames
"HOUSING_FIXTURE_POINT_FRAME_ADDED"           -- Payload: pointFrame
"HOUSING_FIXTURE_POINT_FRAME_RELEASED"        -- Payload: pointFrame
"HOUSING_FIXTURE_POINT_FRAMES_RELEASED"
```

### Neighborhood Events
```lua
-- Info updates
"NEIGHBORHOOD_INFO_UPDATED"                   -- Payload: neighborhoodInfo
"NEIGHBORHOOD_NAME_UPDATED"                   -- Payload: neighborhoodGuid, neighborhoodName
"NEIGHBORHOOD_MAP_DATA_UPDATED"
"NEIGHBORHOOD_LIST_UPDATED"                   -- Payload: result, neighborhoodInfos
"B_NET_NEIGHBORHOOD_LIST_UPDATED"             -- Payload: result, neighborhoodInfos

-- Roster
"UPDATE_BULLETIN_BOARD_ROSTER"                -- Payload: neighborhoodInfo, rosterMemberList
"UPDATE_BULLETIN_BOARD_ROSTER_STATUSES"       -- Payload: rosterMemberList
"UPDATE_BULLETIN_BOARD_MEMBER_TYPE"           -- Payload: player, residentType

-- Invites
"NEIGHBORHOOD_INVITE_RESPONSE"                -- Payload: result
"CANCEL_NEIGHBORHOOD_INVITE_RESPONSE"         -- Payload: result, playerName
"PENDING_NEIGHBORHOOD_INVITES_RECIEVED"       -- Payload: result, pendingInviteList

-- Charter
"ADD_NEIGHBORHOOD_CHARTER_SIGNATURE"          -- Payload: signature
"REMOVE_NEIGHBORHOOD_CHARTER_SIGNATURE"       -- Payload: signature
"CREATE_NEIGHBORHOOD_RESULT"                  -- Payload: result, neighborhoodName
"NEIGHBORHOOD_NAME_VALIDATED"                 -- Payload: approved
"NEIGHBORHOOD_GUILD_SIZE_VALIDATED"           -- Payload: approved
"OPEN_NEIGHBORHOOD_CHARTER"                   -- Payload: neighborhoodInfo, signatures, requiredSignatures
"OPEN_NEIGHBORHOOD_CHARTER_SIGNATURE_REQUEST" -- Payload: neighborhoodInfo

-- Cornerstone/Plot
"OPEN_PLOT_CORNERSTONE"
"CLOSE_PLOT_CORNERSTONE"
"PURCHASE_PLOT_RESULT"                        -- Payload: result

-- Dialogs
"SHOW_PLAYER_EVICTED_DIALOG"
"SHOW_NEIGHBORHOOD_OWNERSHIP_TRANSFER_DIALOG" -- Payload: neighborhoodName, cosmeticOwnerName

-- UI
"OPEN_CREATE_CHARTER_NEIGHBORHOOD_UI"         -- Payload: locationName
"CLOSE_CREATE_CHARTER_NEIGHBORHOOD_UI"
"OPEN_CREATE_GUILD_NEIGHBORHOOD_UI"           -- Payload: locationName
"CLOSE_CREATE_GUILD_NEIGHBORHOOD_UI"
"OPEN_CHARTER_CONFIRMATION_UI"                -- Payload: neighborhoodName, locationName
"CLOSE_CHARTER_CONFIRMATION_UI"
```

### Catalog Events
```lua
"HOUSING_CATALOG_CATEGORY_UPDATED"            -- Payload: categoryID
"HOUSING_CATALOG_SUBCATEGORY_UPDATED"         -- Payload: subcategoryID
"HOUSING_STORAGE_UPDATED"
"HOUSING_STORAGE_ENTRY_UPDATED"               -- Payload: entryID
"HOUSING_REFUND_LIST_UPDATED"

-- Preview
"HOUSING_DECOR_ADD_TO_PREVIEW_LIST"           -- Payload: previewItemData
"HOUSING_DECOR_PREVIEW_LIST_UPDATED"
"HOUSING_DECOR_PREVIEW_LIST_REMOVE_FROM_WORLD" -- Payload: decorGUID
```

### Dye Events
```lua
"DYE_COLOR_UPDATED"                           -- Payload: dyeColorID
"DYE_COLOR_CATEGORY_UPDATED"                  -- Payload: dyeColorCategoryID
```

### House Finder Events
```lua
"HOUSE_FINDER_NEIGHBORHOOD_DATA_RECIEVED"     -- Payload: neighborhoodPlots
"HOUSE_RESERVATION_RESPONSE_RECIEVED"         -- Payload: result
"DECLINE_NEIGHBORHOOD_INVITATION_RESPONSE"    -- Payload: success
"FORCE_REFRESH_HOUSE_FINDER"
"VIEW_HOUSES_LIST_RECIEVED"                   -- Payload: houseInfos
"PLAYER_CHARACTER_LIST_UPDATED"               -- Payload: characterInfos, ownerListIndex
"TRACKED_HOUSE_CHANGED"                       -- Payload: trackedHouse
"MOVE_OUT_RESERVATION_UPDATED"
```

---

## CVars

### Grid and Snap Settings
```lua
-- Grid snap enabled
SetCVar("housingDecorGridSnapEnabled", 1)  -- 0 or 1

-- Grid visible
SetCVar("housingDecorGridVisible", 1)      -- 0 or 1
```

### UI Settings
```lua
-- Storage panel collapsed state
SetCVar("housingStoragePanelCollapsed", 0) -- 0 or 1

-- Tutorials enabled
SetCVar("housingTutorialsEnabled", 1)      -- 0 or 1
```

### Expert Mode Gizmo Settings
```lua
-- Gizmo scale (Expert mode)
SetCVar("housingExpertGizmoScale", 1.0)    -- float

-- Increment amounts
SetCVar("housingExpertTranslateIncrement", 0.1)
SetCVar("housingExpertRotateIncrement", 5.0)
SetCVar("housingExpertScaleIncrement", 0.05)
```

---

## Practical Examples

### Example 1: Check if Player is in Their Own House
```lua
local function IsInOwnHouse()
    if not C_Housing.IsInsideHouse() then
        return false
    end
    return C_Housing.IsInsideOwnHouse()
end

-- More detailed check
local function GetOwnHouseStatus()
    if not C_Housing.IsInsideHouseOrPlot() then
        return "not_in_housing"
    end

    if C_Housing.IsInsidePlot() and not C_Housing.IsInsideHouse() then
        return "on_plot_exterior"
    end

    if C_Housing.IsInsideOwnHouse() then
        return "own_house"
    end

    return "visiting_house"
end
```

### Example 2: List Placed Decor with Budget Info
```lua
local function ShowDecorBudgetStatus()
    if not C_Housing.IsInsideHouseOrPlot() then
        print("Not in a house or plot!")
        return
    end

    local numPlaced = C_HousingDecor.GetNumDecorPlaced()
    local spent = C_HousingDecor.GetSpentPlacementBudget()
    local max = C_HousingDecor.GetMaxPlacementBudget()

    print(string.format("Decor: %d items placed", numPlaced))
    print(string.format("Budget: %d / %d used", spent, max))

    -- Get all placed decor (be careful - this can be expensive!)
    local allDecor = C_HousingDecor.GetAllPlacedDecor()
    if allDecor then
        print("Placed items:")
        for i, entry in ipairs(allDecor) do
            print(string.format("  %d. %s", i, entry.name))
            if i >= 10 then
                print(string.format("  ... and %d more", #allDecor - 10))
                break
            end
        end
    end
end
```

### Example 3: Create a Housing Event Monitor Addon
```lua
-- HousingMonitor.lua
local addonName = "HousingMonitor"
local HousingMonitor = CreateFrame("Frame")

-- Storage for callbacks
HousingMonitor.callbacks = {}

-- Register for housing events
local housingEvents = {
    "HOUSE_EDITOR_MODE_CHANGED",
    "HOUSING_DECOR_PLACE_SUCCESS",
    "HOUSING_DECOR_REMOVED",
    "HOUSE_PLOT_ENTERED",
    "HOUSE_PLOT_EXITED",
    "CURRENT_HOUSE_INFO_UPDATED",
    "HOUSE_LEVEL_CHANGED",
}

for _, event in ipairs(housingEvents) do
    HousingMonitor:RegisterEvent(event)
end

-- Event handler
HousingMonitor:SetScript("OnEvent", function(self, event, ...)
    -- Call any registered callbacks
    if self.callbacks[event] then
        for _, callback in ipairs(self.callbacks[event]) do
            callback(...)
        end
    end

    -- Default logging
    if event == "HOUSE_EDITOR_MODE_CHANGED" then
        local mode = ...
        local modeName = "Unknown"
        for name, value in pairs(Enum.HouseEditorMode) do
            if value == mode then
                modeName = name
                break
            end
        end
        print(string.format("[Housing] Editor mode: %s", modeName))

    elseif event == "HOUSING_DECOR_PLACE_SUCCESS" then
        local decorGUID, size, isNew, isPreview = ...
        local decorInfo = C_HousingDecor.GetDecorInstanceInfoForGUID(decorGUID)
        if decorInfo then
            print(string.format("[Housing] Placed: %s%s",
                decorInfo.name,
                isNew and " (new)" or " (moved)"))
        end

    elseif event == "HOUSE_LEVEL_CHANGED" then
        local levelInfo = ...
        if levelInfo then
            print(string.format("[Housing] House leveled up to %d!", levelInfo.level))
        end
    end
end)

-- Public API
function HousingMonitor:RegisterCallback(event, callback)
    if not self.callbacks[event] then
        self.callbacks[event] = {}
    end
    table.insert(self.callbacks[event], callback)
end

-- Check housing availability
function HousingMonitor:IsHousingAvailable()
    return C_Housing.IsHousingServiceEnabled() and
           C_Housing.HasHousingExpansionAccess()
end

-- Get current editor mode name
function HousingMonitor:GetEditorModeName()
    local mode = C_HouseEditor.GetActiveHouseEditorMode()
    for name, value in pairs(Enum.HouseEditorMode) do
        if value == mode then
            return name
        end
    end
    return "None"
end

_G["HousingMonitor"] = HousingMonitor
```

### Example 4: Decor Placement Helper
```lua
local function PlaceDecorWithFeedback(catalogEntryID)
    -- Check prerequisites
    if not C_Housing.IsInsideHouseOrPlot() then
        return false, "Not in a house or plot"
    end

    if not C_HouseEditor.IsHouseEditorActive() then
        local result = C_HouseEditor.EnterHouseEditor()
        if result ~= Enum.HousingResult.Success then
            return false, "Cannot enter editor mode"
        end
    end

    -- Make sure we're in basic mode
    local currentMode = C_HouseEditor.GetActiveHouseEditorMode()
    if currentMode ~= Enum.HouseEditorMode.BasicDecor then
        local result = C_HouseEditor.ActivateHouseEditorMode(Enum.HouseEditorMode.BasicDecor)
        if result ~= Enum.HousingResult.Success then
            return false, "Cannot activate Basic Decor mode"
        end
    end

    -- Check budget
    local spent = C_HousingDecor.GetSpentPlacementBudget()
    local max = C_HousingDecor.GetMaxPlacementBudget()

    -- Get entry info to check placement cost
    local entryInfo = C_HousingCatalog.GetCatalogEntryInfo(catalogEntryID)
    if entryInfo then
        if spent + entryInfo.placementCost > max then
            return false, string.format("Not enough budget (%d/%d, need %d more)",
                spent, max, entryInfo.placementCost)
        end

        if entryInfo.quantity <= 0 and entryInfo.remainingRedeemable <= 0 then
            return false, "No items available to place"
        end
    end

    -- Start placing
    C_HousingBasicMode.StartPlacingNewDecor(catalogEntryID)
    return true, "Started placing - click to position"
end
```

### Example 5: Dye Management Helper
```lua
local DyeHelper = {}

-- Get all owned dyes organized by category
function DyeHelper:GetOwnedDyesByCategory()
    local result = {}
    local categories = C_DyeColor.GetAllDyeColorCategories()

    for _, categoryID in ipairs(categories) do
        local catInfo = C_DyeColor.GetDyeColorCategoryInfo(categoryID)
        local ownedDyes = C_DyeColor.GetDyeColorsInCategory(categoryID, true)

        if #ownedDyes > 0 then
            result[categoryID] = {
                name = catInfo and catInfo.name or "Unknown",
                dyes = {}
            }

            for _, dyeID in ipairs(ownedDyes) do
                local dyeInfo = C_DyeColor.GetDyeColorInfo(dyeID)
                if dyeInfo then
                    table.insert(result[categoryID].dyes, {
                        id = dyeID,
                        name = dyeInfo.name,
                        color = dyeInfo.color,
                    })
                end
            end
        end
    end

    return result
end

-- Check if selected decor can be dyed
function DyeHelper:CanDyeSelectedDecor()
    local decorInfo = C_HousingCustomizeMode.GetSelectedDecorInfo()
    if not decorInfo then
        return false, "No decor selected"
    end

    if not decorInfo.canBeCustomized then
        return false, "This decor cannot be customized"
    end

    if #decorInfo.dyeSlots == 0 then
        return false, "This decor has no dye slots"
    end

    return true, decorInfo.dyeSlots
end

-- Apply dye to all available slots
function DyeHelper:ApplyDyeToAllSlots(dyeColorID)
    local canDye, slotsOrError = self:CanDyeSelectedDecor()
    if not canDye then
        return false, slotsOrError
    end

    for _, slot in ipairs(slotsOrError) do
        C_HousingCustomizeMode.ApplyDyeToSelectedDecor(slot.ID, dyeColorID)
    end

    return true
end

_G["DyeHelper"] = DyeHelper
```

### Example 6: Neighborhood Visitor Counter
```lua
local NeighborhoodTracker = CreateFrame("Frame")
NeighborhoodTracker.visitors = {}

NeighborhoodTracker:RegisterEvent("HOUSE_PLOT_ENTERED")
NeighborhoodTracker:RegisterEvent("NEIGHBORHOOD_INFO_UPDATED")
NeighborhoodTracker:RegisterEvent("UPDATE_BULLETIN_BOARD_ROSTER")

NeighborhoodTracker:SetScript("OnEvent", function(self, event, ...)
    if event == "HOUSE_PLOT_ENTERED" then
        local houseInfo = C_Housing.GetCurrentHouseInfo()
        if houseInfo then
            print(string.format("Entered %s's plot in %s",
                houseInfo.ownerName or "Unknown",
                houseInfo.neighborhoodName or "Unknown Neighborhood"))
        end

    elseif event == "UPDATE_BULLETIN_BOARD_ROSTER" then
        local neighborhoodInfo, roster = ...
        local online = 0
        for _, member in ipairs(roster) do
            if member.isOnline then
                online = online + 1
            end
        end
        print(string.format("Neighborhood %s: %d/%d online",
            neighborhoodInfo.neighborhoodName,
            online, #roster))
    end
end)

-- Get current plot owner info
function NeighborhoodTracker:GetCurrentPlotOwner()
    if not C_Housing.IsInsidePlot() then
        return nil
    end

    local houseInfo = C_Housing.GetCurrentHouseInfo()
    return houseInfo and houseInfo.ownerName
end

-- Check if we're in our own neighborhood
function NeighborhoodTracker:IsInOwnNeighborhood()
    return C_HousingNeighborhood.IsNeighborhoodOwner() or
           C_HousingNeighborhood.IsNeighborhoodManager()
end

_G["NeighborhoodTracker"] = NeighborhoodTracker
```

---

## Best Practices

### 1. Always Check Location Before Housing Operations
```lua
-- Good
if C_Housing.IsInsideHouseOrPlot() then
    local houseInfo = C_Housing.GetCurrentHouseInfo()
    -- Proceed with operations
end

-- Bad - may fail silently or error
local houseInfo = C_Housing.GetCurrentHouseInfo()
```

### 2. Use Events Instead of Polling
```lua
-- Good: React to events
local frame = CreateFrame("Frame")
frame:RegisterEvent("HOUSING_DECOR_PLACE_SUCCESS")
frame:SetScript("OnEvent", function(self, event, decorGUID, ...)
    -- Handle placement success
end)

-- Bad: Polling for changes
local function OnUpdate(self, elapsed)
    local count = C_HousingDecor.GetNumDecorPlaced()
    if count ~= self.lastCount then
        -- Handle change
    end
end
```

### 3. Respect Editor Mode States
```lua
-- Check mode before mode-specific operations
local function SafeDecorOperation()
    local mode = C_HouseEditor.GetActiveHouseEditorMode()

    if mode == Enum.HouseEditorMode.BasicDecor then
        -- Use C_HousingBasicMode functions
    elseif mode == Enum.HouseEditorMode.ExpertDecor then
        -- Use C_HousingExpertMode functions
    elseif mode == Enum.HouseEditorMode.Customize then
        -- Use C_HousingCustomizeMode functions
    end
end
```

### 4. Handle API Results Properly
```lua
-- Many housing functions return HousingResult
local result = C_HouseEditor.EnterHouseEditor()
if result == Enum.HousingResult.Success then
    -- Entered successfully (or already active)
elseif result == Enum.HousingResult.ActionLockedByCombat then
    -- Cannot enter during combat
elseif result == Enum.HousingResult.NotInsideHouse then
    -- Not in a valid location
else
    -- Handle other error cases
    print("Editor failed:", result)
end
```

### 5. Be Careful with Expensive Operations
```lua
-- GetAllPlacedDecor can be expensive
-- Cache results and avoid calling frequently
local cachedDecor = nil
local cacheTime = 0

local function GetPlacedDecorCached()
    local now = GetTime()
    if not cachedDecor or (now - cacheTime) > 5 then
        cachedDecor = C_HousingDecor.GetAllPlacedDecor()
        cacheTime = now
    end
    return cachedDecor
end
```

### 6. Handle Permissions Appropriately
```lua
-- Check permissions before roster operations
local function CanManageRoster()
    return C_HousingNeighborhood.IsNeighborhoodOwner() or
           C_HousingNeighborhood.IsNeighborhoodManager()
end

if CanManageRoster() then
    C_HousingNeighborhood.InvitePlayerToNeighborhood(playerName)
else
    print("You don't have permission to invite players")
end
```

---

## Related Systems

### C_CatalogShop
Used for purchasing housing items from the in-game shop.
```lua
-- Housing items are purchased through the catalog shop system
-- This integrates with C_HousingCatalog for market info
```

### Bulletin Board System
The bulletin board provides neighborhood management UI:
- Roster viewing and management
- Invitation management
- Ownership transfer
- Access via `C_HousingNeighborhood` functions that require bulletin board interaction

### House Chest
Virtual storage for housing items:
- Decor items stored here when not placed
- Rooms available for placement
- Accessed through `C_HousingCatalog` functions

### Housing Tutorial System
```lua
-- Start housing tutorial
C_Housing.StartTutorial()

-- Control tutorial visibility
SetCVar("housingTutorialsEnabled", 1)
```

---

## Enumerations Reference

### HousingResult (90 values)
Common result codes:
- `Success` (0) - Operation succeeded
- `ActionLockedByCombat` (1) - Cannot perform during combat
- `CannotAfford` (5) - Not enough currency
- `NotInsideHouse` (62) - Must be inside house
- `PermissionDenied` (66) - No permission
- `MaxDecorReached` (48) - Budget exhausted

### HousingCatalogEntrySize
- `None` (0)
- `Tiny` (65)
- `Small` (66)
- `Medium` (67)
- `Large` (68)
- `Huge` (69)

### ResidentType
- Owner
- Manager
- Resident

### NeighborhoodType
- Charter (friend-created)
- Guild

---

*Last Updated: Patch 12.0.0 (Midnight)*
