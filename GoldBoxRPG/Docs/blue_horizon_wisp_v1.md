# Combat Grid: Replace Cube Walls with SM_Wall_Stone (Edge-Based)

## Objective
Replace `BP_CombatGrid`'s cube wall mesh with `SM_Wall_Stone` using edge-based wall placement (matching the dungeon generator's approach) so walls appear only at boundaries between blocked and unblocked tiles.

## Changes
- `/Game/Blueprints/Combat/BP_CombatGrid` - WallMeshISM defaults + RebuildTileInstances graph logic

## Relevant Assets and Quirks
- **Exploration reference**: `BP_DungeonGenerator` uses `SM_Wall_Stone` at scale `(1,1,1)` with `TileSize=400`, placing walls at tile edges with direction-specific offsets and yaw rotations (North: offset Y=-200, Yaw=90°)
- **Combat current**: `BP_CombatGrid.WallMeshISM` uses `Cube`, `RelativeScale3D=(200,200,200)`, `OverrideMaterials=WorldGridMaterial`, instance scale `(2,2,2)`, rotation `(0,0,0)` filling entire blocked tiles
- **Tile size mismatch**: Exploration tiles are 400cm, combat tiles are 200cm — SM_Wall_Stone base dimensions fit 400cm, so combat walls need scale `(0.5, 1.0, 0.5)` per instance
- **Wall direction pattern** (verified in BP_DungeonGenerator.SpawnNorthWall):
  - North edge: offset `(0, -TileSize/2, 0)`, rotation Yaw=90°
  - East edge: offset `(TileSize/2, 0, 0)`, rotation Yaw=0°
  - South edge: offset `(0, TileSize/2, 0)`, rotation Yaw=90°
  - West edge: offset `(-TileSize/2, 0, 0)`, rotation Yaw=0°

## Steps
- [ ] 1. Update WallMeshISM component defaults: set `StaticMesh` to `/Game/Art/Meshes/SM_Wall_Stone`, `RelativeScale3D` to `(1,1,1)`, clear `OverrideMaterials` (use mesh's own material) [depends: none]
- [ ] 2. Modify `RebuildTileInstances`: replace the blocked-tile-fill logic (lines around K2Node_IfThenElse_0 → K2Node_CallFunction_8) with edge-detection: for each tile, check all 4 neighbors via `IsBlocked`; if neighbor is NOT blocked or out of bounds, spawn a direction-specific wall instance at that edge [depends: 1]
- [ ] 3. Add wall edge spawning: per direction, compute offset `(±TileSize/2, 0)` or `(0, ±TileSize/2)`, set rotation `(0,0,0)` or `(0,0,90)`, and scale `(0.5, 1.0, 0.5)` — wire to `AddInstance` on `WallMeshISM` [depends: 2]
- [ ] 4. Add `Print String` debug logging to log how many wall instances were spawned during `RebuildTileInstances` [depends: 2,3]

## Verify
- PIE in TestDungeon, trigger combat encounter: blocked tiles should show stone wall panels at edges facing unblocked neighbors, not cubes filling entire tiles
- Walls should use the stone material (not WorldGridMaterial wireframe)
