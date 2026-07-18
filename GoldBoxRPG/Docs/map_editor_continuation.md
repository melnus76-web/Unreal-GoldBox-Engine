# Map Editor Continuation — Steps 18-22

## Current State (completed)

All prior steps are done:
- `BP_DungeonMapPack` PrimaryDataAsset exists at `/Game/Blueprints/Dungeon/` with `DungeonName` (FText) and `Levels` (TArray<BP_DungeonMapData*>)
- `E_BrushMode` enum at `/Game/Blueprints/Editor/` with values: TileType, WallNorth, WallEast, WallSouth, WallWest, DoorNorth, DoorEast, DoorSouth, DoorWest, Decoration, Special
- `BP_MapEditorWidget` EditorUtilityWidget at `/Game/Blueprints/Editor/` with full widget tree:
  - Toolbar: Btn_LoadPack, Btn_Save, Btn_NewLevel, Cmb_LevelSelect, Spin_Width, Spin_Height, Btn_ApplySize
  - Brush sidebar: 17 buttons (Btn_Floor, Btn_WallN/E/S/W, Btn_DoorN/E/S/W, Btn_Torch, Btn_Pit, Btn_Water, Btn_Stairs, Btn_Dark, Btn_Trap, Btn_Exit)
  - Grid: Grid_ScrollBox > Grid_UniformGrid (UniformGridPanel)
  - Properties: Prop_VBox with Chk_Passable, Cmb_TileType, Chk_WallN/E/S/W, Chk_DoorN/E/S/W, Chk_DecoN/E/S/W (CheckBoxes — binary Torch/None), Chk_IsDark, Chk_HasTrap, Chk_HasSecretDoor, Chk_IsExit
- All 17 brush buttons wired: each sets ActiveBrushMode and optionally ActiveTileType/ActiveDecoration
- Variables on BP_MapEditorWidget:
  - CurrentPack (BP_DungeonMapPack*, Object Reference)
  - WorkingMapData (S_DungeonMap, Struct)
  - ActiveBrushMode (E_BrushMode)
  - ActiveTileType (E_TileType)
  - ActiveDecoration (E_WallDecoration)
  - SelectedTileX, SelectedTileY (Integer)
  - GridWidth, GridHeight (Integer)
- `RebuildGrid()` function complete:
  - ClearChildren on Grid_UniformGrid
  - Nested ForLoop (Row 0..GridHeight-1, Col 0..GridWidth-1)
  - Computes tile index: Row * GridWidth + Col
  - Gets tile from WorkingMapData.TileGrid via GetArrayItem
  - Break S_DungeonTile → Switch on E_TileType → Set local `TileColour` variable per type
  - Branch on NOT Passable → override TileColour to dark grey (0.15,0.15,0.15)
  - GenericCreateObject(Border) → Cast to Border → SetBrushColor(TileColour)
  - AddChild(Grid_UniformGrid, Border) → Cast slot to UniformGridSlot → SetColumn(Col) → SetRow(Row)
- Event Construct:
  - Set WorkingMapData = Make S_DungeonMap (LevelID=1, Width=8, Height=8, defaults)
  - ForLoop 0..63: Make S_DungeonTile (all defaults, Passable=False) → Array_Add to WorkingMapData.TileGrid
  - Completed → Set GridWidth=8 → Set GridHeight=8 → RebuildGrid()

## Files on Disk

| Asset | Path |
|---|---|
| BP_MapEditorWidget | /Game/Blueprints/Editor/BP_MapEditorWidget.BP_MapEditorWidget |
| BP_DungeonMapPack | /Game/Blueprints/Dungeon/BP_DungeonMapPack.BP_DungeonMapPack |
| E_BrushMode | /Game/Blueprints/Editor/E_BrushMode.E_BrushMode |
| BP_DungeonMapData | /Game/Blueprints/Dungeon/BP_DungeonMapData.BP_DungeonMapData |
| BP_MapManager | /Game/Blueprints/Dungeon/BP_MapManager.BP_MapManager |
| BP_GameManager | /Game/Blueprints/Core/BP_GameManager.BP_GameManager |
| S_DungeonMap struct | /Game/Blueprints/Structs/S_DungeonMap.S_DungeonMap |
| S_DungeonTile struct | /Game/Blueprints/Structs/S_DungeonTile.S_DungeonTile |

---

## Step 18 — PaintTile function

Create a new function on `BP_MapEditorWidget`:

**Name:** `PaintTile`
**Inputs:**
- `TileX` (Integer)
- `TileY` (Integer)

**Logic:**

1. Compute tile index: `TileY * GridWidth + TileX`
2. Get current tile from array: break WorkingMapData → TileGrid → **GET** at index
3. Use **Switch on E_BrushMode** on `ActiveBrushMode`. Wire each case:

   **TileType case:**
   - Make S_DungeonTile copying all values from the current tile (use **Set members in S_DungeonTile** node)
   - OR: Break the current tile, then use Make S_DungeonTile re-pinning each field except TileType
   - Set `TileType` = ActiveTileType
   - If setting to Floor: set Passable = True
   - If setting to Pit/Water: set Passable = False
   - **Array_Set** at tile index with the new struct

   **WallNorth/East/South/West cases:**
   - Break current tile
   - Toggle the corresponding Wall bool (NOT current value)
   - Re-make the tile with the toggled value
   - Array_Set

   **DoorNorth/East/South/West cases:**
   - Break current tile
   - Toggle the Door bool
   - If adding a door, also set the corresponding Wall = False (doors replace walls)
   - Array_Set

   **Decoration case:**
   - Break current tile
   - Set all four WallDecoration fields to ActiveDecoration
   - Array_Set

   **Special case:**
   - Break current tile
   - Cycle through special flags: first click sets IsDark=True, second click sets HasTrap=True, third sets HasSecretDoor=True, fourth sets IsExitTile=True, fifth clears all
   - Implementation: check current state, cycle to next
   - Array_Set

4. After Array_Set on all cases, call **RebuildGrid**

---

## Step 19 — SelectTile + Property Inspector

### 19a — SelectTile function

Create function:

**Name:** `SelectTile`
**Inputs:** `TileX` (Integer), `TileY` (Integer)

**Logic:**
1. Set SelectedTileX = TileX
2. Set SelectedTileY = TileY
3. Compute index: TileY * GridWidth + TileX
4. Get tile from WorkingMapData.TileGrid
5. Break S_DungeonTile
6. Populate property widgets:
   - Set Txt_TilePos text to "Tile: (X, Y)"
   - Chk_Passable → Set Is Checked = Passable
   - Cmb_TileType → Set Selected Option = TileType (need to map enum to string)
   - Chk_WallN/E/S/W → Set Is Checked from respective bools
   - Chk_DoorN/E/S/W → same
   - Chk_DecoN/E/S/W → Set Is Checked = (WallDecoration == Torch)
   - Chk_IsDark/HasTrap/HasSecretDoor/IsExit → same

### 19b — Property change handlers

For each property widget, add an OnCheckStateChanged or OnSelectionChanged event that calls an `ApplyProperty` function:

**ApplyProperty** function:
1. Compute index from SelectedTileX/Y
2. Get current tile
3. Read all property widget values
4. Rebuild the tile with new values
5. Array_Set at index
6. Call RebuildGrid (optional — could be batched)

Alternatively, simpler: each property change handler directly reads the widget, builds a new tile, and writes it back. No separate ApplyProperty function needed.

---

## Step 20 — Click Handling on Grid Tiles

Since individual Border widgets in a UniformGridPanel don't have click events directly, use this approach:

### 20a — Overlay Button

Add a transparent **Button** on top of the Grid_UniformGrid (same size, position, overlapping). Wire its **OnClicked** event:

1. Get mouse position: use **Get Mouse Position on Viewport**
2. Convert to local space relative to Grid_UniformGrid: **Absolute to Local** with Grid_UniformGrid's geometry
3. Compute tile coordinates: `TileX = LocalX / TileSize`, `TileY = LocalY / TileSize` (TileSize = 48 or whatever pixel size each Border is)
4. Check if the mouse button was left or right:
   - For left-click: call **PaintTile(TileX, TileY)**
   - For right-click: call **SelectTile(TileX, TileY)**

For right-click detection, use **OnMouseButtonDown** override instead of OnClicked (OnClicked only fires for left-click). Add an override for **OnMouseButtonDown** on the main widget or the overlay button, and check the Mouse Event's **Pressed Button** pin.

### 20b — Tile size constant

Add a variable: `TilePixelSize` (Integer, default = 48). Each Border in RebuildGrid should have its size set via **Set Size Override** on the slot: Width = TilePixelSize, Height = TilePixelSize.

In RebuildGrid, after Cast to UniformGridSlot → SetColumn/SetRow, also call **Set Size** → X=48, Y=48.

---

## Step 21 — Save / Load

### 21a — Btn_ApplySize OnClicked

Wire Btn_ApplySize's OnClicked:
1. Read Spin_Width.Value → Set GridWidth
2. Read Spin_Height.Value → Set GridHeight
3. Update WorkingMapData.Width and WorkingMapData.Height
4. Resize WorkingMapData.TileGrid:
   - If new grid is larger: while TileGrid length < Width*Height, add default tiles (Passable=False, TileType=Floor)
   - If smaller: truncate TileGrid (remove excess entries)
5. Call RebuildGrid

### 21b — Btn_Save OnClicked

Wire Btn_Save's OnClicked:
1. Get CurrentPack
2. Get selected level index from Cmb_LevelSelect
3. Get CurrentPack.Levels[Index] → the BP_DungeonMapData to save
4. Set its MapData = WorkingMapData
5. Call **EditorAssetLibrary::SaveLoadedAsset**(DataAsset)
6. Print notification: "Saved {Level Name}"

### 21c — Btn_LoadPack OnClicked

Wire Btn_LoadPack's OnClicked:
1. Open asset picker for BP_DungeonMapPack type
2. Load selected pack → Set CurrentPack
3. Populate Cmb_LevelSelect with level names
4. Auto-select first level
5. Load its MapData into WorkingMapData
6. Set GridWidth/Height from loaded data
7. Call RebuildGrid

For the asset picker, use **EditorUtilityLibrary::EditorAssetLibrary::SyncBrowserToObjects** or write a custom file dialog. In EditorUtilityWidgets, you can use:**Load Object** with a class filter.

Simplest: add a **BP_DungeonMapPack** variable exposed as Instance Editable on the widget itself, and the user drags a pack asset into it before opening. Then Btn_LoadPack reads that variable.

### 21d — Btn_NewLevel OnClicked

1. Create new BP_DungeonMapData asset at /Game/Data/DungeonMaps/DA_{PackName}_Level{N}
2. Initialize its MapData with current Width/Height, empty tiles
3. Add to CurrentPack.Levels array
4. Refresh Cmb_LevelSelect
5. Select the new level

---

## Step 22 — BP_GameManager Integration

### 22a — CurrentMapPack variable

Add to `BP_GameManager`:
- `CurrentMapPack` (BP_DungeonMapPack*, Object Reference, Instance Editable)

### 22b — Update LoadDungeonLevel

In BP_GameManager.LoadDungeonLevel:
1. Remove the placeholder PrintString
2. If CurrentMapPack is valid AND LevelId is within CurrentMapPack.Levels range:
   - Get CurrentMapPack.Levels[LevelId]
   - Call BP_MapManager.LoadMap(MapDataAsset = that DataAsset)
3. If invalid: fall back to LoadMap with None (triggers GenerateTestMap)

### 22c — Assign pack in editor

In the TestDungeon level or BP_GameMode defaults, set BP_GameManager's CurrentMapPack to the authored pack. This gets cooked into the build.

---

## Quick Testing Checklist

1. Double-click BP_MapEditorWidget → editor window opens
2. A grid of 64 dark-grey tiles appears (8×8, all Passable=False)
3. Click Floor brush in sidebar → left-click a tile → turns tan (floor)
4. Click Wall N brush → click tile → wall appears on north edge
5. Right-click tile → property panel shows tile data
6. Click Apply after changing Width/Height → grid resizes
7. Save should produce a valid BP_DungeonMapData asset

---

## Note for the Assistant

The user is a novice. Give instructions one small piece at a time. Verify after each piece using `get_asset_graph` or `get_asset_meta`. Do not assume UE widget nodes exist — verify behavior in UE 5.7 specifically. The decoration properties use CheckBoxes (not ComboBoxes) because E_WallDecoration only has None/Torch — this is intentional and correct.
