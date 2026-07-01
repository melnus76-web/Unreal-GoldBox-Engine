# Functions List — Project Audit

Generated: 2026-07-01

---

## 🔴 BP_CombatManager (34 functions)

| Function | Nodes | Resp | Verdict |
|----------|-------|------|---------|
| `StartCombat` | ~30 | 1 | ✅ Orchestration chain |
| `BuildInitiativeOrder` | ~20 | 1 | ✅ Rolls + updates combatants |
| `SortInitiativeOrder` | ~25 | 1 |  ✅Selection sort |
| `BuildCombatants` | ~25 | 1 | ✅ Uses new AddCombatantToGrid helper |
| `SpawnCombatantMarkers` | ~15 | 1 | ✅ |
| `TransitionToCombat` | ~20 | 1 | ✅ Camera + input + pawn |
| `StartPlayerTurn` | ~15 | 1 | ✅ |
| `StartEnemyTurn` | ~5 | 1 | ✅ Delegates to EnemyAI |
| **`OnActionComplete`** | **~40** | **5** | **🔴 Defeat check • While loop turn advance • Round msg • Condition filter • Dispatch** |
| `FindStartCombatant` | ~15 | 1 | ✅ |
| `EnterTargetSelectMode` | ~8 | 1 | ✅ |
| **`ExecutePlayerAttack`** | **~60** | **3** | **🟡 Range check • Attack resolution • 3x message-building chains** |
| `RemoveMarkerForCombatant` | ~8 | 1 | ✅ |
| `CheckVictory` | ~12 | 1 | ✅ |
| **`EndCombat`** | **~35** | **3** | **🟡 Victory: XP+loot+msg • Defeat: menu+widget • Common cleanup** |
| `EnterMoveMode` | ~5 | 1 | ✅ |
| `ExecuteCombatantMove` | ~30 | 2 | ⚠️ Move logic + marker sync |
| `RefreshMoveHighlights` | ~10 | 1 | ✅ |
| `EndPlayerTurn` | ~8 | 1 | ✅ |
| `UpdateCombatantGridPosition` | ~10 | 1 | ✅ |
| `MoveMarkerToTile` | ~8 | 1 | ✅ |
| `ApplyDamage` | ~10 | 1 | ✅ |
| `ApplyCondition` | ~8 | 1 | ✅ |
| `ConditionToRemove` | ~8 | 1 | ✅ |
| `HasCondition` | ~5 | 1 | ✅ |
| `SetMarkerDowned` | ~5 | 1 | ✅ |
| `CheckDefeat` | ~8 | 1 | ✅ |
| `FindActiveCombatant` | ~15 | 1 | ✅ |
| `FindCombatantByID` | ~8 | 1 | ✅ |
| `FindMarkerByID` | ~8 | 1 | ✅ |
| `TickConditions` | ~20 | 1 | ✅ One condition type (Poisoned) |
| `ResolveMorale` | ~35 | 1 | ✅ Complex but single concern |
| `ProcessLootDrops` | ~15 | 1 | ✅ |
| `RetreatCharacter` | ~8 | 1 | ✅ |
| `FinishEnemyTurn` | ~5 | 1 | ✅ |

---

## 🔵 BP_EnemyAI (13 functions)

| Function | Nodes | Resp | Verdict |
|----------|-------|------|---------|
| `RunEnemyTurn` | ~20 | 1 | ✅ (refactored!) |
| `ExecuteEnemyAttack` | ~40 | 2 | ⚠️ Attack resolve + auto-hit check + distance check + rogue ambush + message |
| `ChooseAction` | ~18 | 1 | ✅ |
| `FindNearestPartyMember` | ~25 | 1 | ✅ Manhattan distance |
| `MoveOneStepToward` | ~25 | 1 | ✅ |
| `FindPathBFS` | ~45 | 1 | ✅ Complex algorithm, single concern |
| `CheckAOEOpportunity` | ~25 | 1 | ✅ |
| `GetFleeingEdgeTile` | ~15 | 1 | ✅ |
| `SkipCurrentTurn` | ~5 | 1 | ✅ (new) |
| `ExecuteFleeMovement` | ~20 | 1 | ✅ (new) |
| `MoveToNearestPartyMember` | ~25 | 1 | ✅ (new) |
| `ExecuteMeleeAttack` | ~8 | 1 | ✅ (new) |

---

## 🟢 BP_GameManager (11 functions)

| Function | Nodes | Resp | Verdict |
|----------|-------|------|---------|
| `ChangeGameState` | ~5 | 1 | ✅ |
| `SetStoryFlag` | ~5 | 1 | ✅ |
| `CheckStoryFlag` | ~5 | 1 | ✅ |
| `LoadDungeonLevel` | ~15 | 1 | ✅ Orchestration |
| `StartCombat` | ~2 | 1 | ✅ Stub |
| `EndCombat` | ~2 | 1 | ✅ Stub |
| `AwardXP` | ~5 | 1 | ✅ |
| `InitialiseGameState` | ~2 | 1 | ✅ |
| `SpawnManagers` | ~30 | 1 | ✅ Spawning chain |
| `InitialiseParty` | ~3 | 1 | ✅ |
| `InitialiseWorld` | ~10 | 1 | ✅ |

---

## 🟢 BP_DungeonGenerator (12 functions)

| Function | Nodes | Resp | Verdict |
|----------|-------|------|---------|
| `GenerateDungeon` | ~8 | 1 | ✅ For Loop → ProcessRow |
| `ProcessRow` | ~8 | 1 | ✅ For Loop → ProcessTile |
| `ProcessTile` | ~10 | 1 | ✅ GetTile → spawn branches |
| `SpawnTileWalls` | ~6 | 1 | ✅ |
| `SpawnNorthWall` | ~5 | 1 | ✅ |
| `SpawnEastWall` | ~5 | 1 | ✅ |
| `SpawnSouthWall` | ~5 | 1 | ✅ |
| `SpawnWestWall` | ~5 | 1 | ✅ |
| `SpawnFloorAndCeiling` | ~6 | 1 | ✅ |
| `AddFloorInstance` | ~5 | 1 | ✅ |
| `AddCeilingInstance` | ~5 | 1 | ✅ |
| `TileToWorldLocation` | ~5 | 1 | ✅ |

---

## 🟢 BP_MapManager (11 functions)

| Function | Nodes | Resp | Verdict |
|----------|-------|------|---------|
| `GenerateTestMap` | ~20 | 1 | ✅ AddTileToMap chain |
| `AddTileToMap` | ~8 | 1 | ✅ |
| `LoadMap` | ~10 | 1 | ✅ |
| `GetTile` | ~8 | 1 | ✅ |
| `CheckWall` | ~5 | 1 | ✅ |
| `CheckDoor` | ~5 | 1 | ✅ |
| `UpdatePlayerPosition` | ~5 | 1 | ✅ |
| `GetExitTile` | ~5 | 1 | ✅ |
| `IncrementStepCounter` | ~5 | 1 | ✅ |
| `HandleLevelTransition` | ~8 | 1 | ✅ |
| `FillMapWithEmptyTiles` | ~5 | 1 | ✅ |

---

## 🟢 BP_CombatGrid (21 functions)

All grid utility functions — single responsibility, ≤10 nodes each. ✅

---

## 🟢 BP_CombatantMarker (2 functions)

| Function | Nodes | Resp | Verdict |
|----------|-------|------|---------|
| `Initialise` | ~15 | 1 | ✅ Config visual based on type |
| `Construction Script` | — | 1 | ✅ |

---

## 🟢 BP_ExplorerPawn (7 functions)

All movement/utility functions — single responsibility, ≤10 nodes each. ✅

---

## 🟢 Manager Blueprints

| BP | Functions | Verdict |
|----|-----------|---------|
| BP_PartyManager | 9 | ✅ All single responsibility |
| BP_TextManager | 2 | ✅ |
| BP_XPManager | 4 | ✅ |
| BP_EncounterManager | 4 | ✅ |
| BP_EventManager | 2 | ✅ |

---

## 🟢 BPL_RulesLibrary (22 functions)

Pure function library — all single responsibility. ✅

---

## Summary

| Priority | Function | BP | Issues |
|----------|----------|-----|--------|
| 🔴 | `OnActionComplete` | BP_CombatManager | 5 responsibilities in ~40 nodes |
| 🟡 | `ExecutePlayerAttack` | BP_CombatManager | 3-branch duplicate message chain |
| 🟡 | `EndCombat` | BP_CombatManager | Victory vs Defeat paths intertwined |
| 🟡 | `ExecuteEnemyAttack` | BP_EnemyAI | Attack + auto-hit + distance + rogue + message |
| ⚠️ | `ExecuteCombatantMove` | BP_CombatManager | Move logic + marker sync |

**116 total functions across 14 BPs. 18 functions had issues, 6 are multi-responsibility, 2 already refactored.**
