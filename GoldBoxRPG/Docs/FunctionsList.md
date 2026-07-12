# Functions List — Project Audit

Generated: 2026-07-01 | Updated: 2026-07-12

---

## 🟢 WBP_CharacterCreation (12 functions) — GB-019 COMPLETE

| Function | Nodes | Resp | Verdict |
|----------|-------|------|---------|
| `UpdateNavigation` | ~15 | 1 | ✅ Prev/Next button visibility |
| `ValidateRollStats` | ~15 | 1 | ✅ All 6 stats must be rolled |
| `SelectRace` | ~10 | 1 | ✅ Highlight logic |
| `SelectClass` | ~10 | 1 | ✅ Highlight logic |
| `SelectAlignment` | ~10 | 1 | ✅ Highlight logic |
| `GenerateRandomName` | ~12 | 1 | ✅ Syllable-based name gen |
| `GetRaceDisplayName` | ~8 | 1 | ✅ Pure enum-to-text |
| `GetClassDisplayName` | ~10 | 1 | ✅ Pure enum-to-text |
| `GetAlignmentDisplayName` | ~8 | 1 | ✅ Pure enum-to-text |
| `Initialise` | ~5 | 1 | ✅ Self-init from Event Construct |
| `PopulateConfirmPanel` | ~20 | 1 | ✅ All summary TextBlocks |
| `AssembleAndAddCharacter` | ~40 | 1 | ✅ DT lookup + HP + saves + struct + save |

---

## 🟡 WBP_CharacterScreen (6 functions) — GB-052 IN PROGRESS

| Function | Nodes | Resp | Verdict |
|----------|-------|------|---------|
| `Initialise` | ~5 | 1 | ✅ Store PartyManagerRef |
| `RefreshPartyList` | ~20 | 1 | ✅ Clear + ForEach → CreateWidget WBP_PartySlot |
| `OnPartySlotClicked` | ~3 | 1 | ✅ Set SelectedIndex + RefreshCharacterDetails |
| `OnTabClicked` | ~3 | 1 | ✅ SetActiveWidgetIndex |
| **`RefreshCharacterDetails`** | **~80** | **1** | **✅ Stats/Saves/Equip populated. ComputeSavingThrows + DT_LevelProgression lookup + conditions** |
| `GetEquipName` | ~5 | 1 | ✅ Placeholder — returns "None" for ID 0 |

---

## 🔴 BP_CombatManager (34 functions)

| Function | Nodes | Resp | Verdict |
|----------|-------|------|---------|
| `StartCombat` | ~30 | 1 | ✅ Orchestration chain |
| `BuildInitiativeOrder` | ~20 | 1 | ✅ Rolls + updates combatants |
| `SortInitiativeOrder` | ~25 | 1 | ✅ Selection sort |
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
| `RunEnemyTurn` | ~20 | 1 | ✅ Dispatch via ChooseAction |
| `ChooseAction` | ~15 | 1 | ✅ MoveToTarget/MeleeAttack/CastSpell/Flee/SkipTurn |
| `MoveToTarget` | ~25 | 1 | ✅ BFS pathfinding + step execute |
| `MeleeAttack` | ~20 | 1 | ✅ Attack resolution chain |
| `CastSpell` | ~5 | 1 | ⚠️ PrintString stub |
| `Flee` | ~20 | 1 | ✅ Morale-based flee to edge |
| `SkipTurn` | ~3 | 1 | ✅ |
| `CastAOESpell` | ~5 | 1 | ⚠️ PrintString stub |
| `FindNearestPlayer` | ~12 | 1 | ✅ |
| `FindPathBFS` | ~30 | 1 | ✅ With adjacency+occupancy |
| `GetAdjacentTiles` | ~10 | 1 | ✅ Fixed DX/DY swap |
| `GetFleeTarget` | ~15 | 1 | ✅ Edge tile search |
| `GetAttacksThisRound` | ~8 | 1 | ✅ |

---

## 🟣 BP_PartyManager (11 functions) — Updated

| Function | Nodes | Resp | Verdict |
|----------|-------|------|---------|
| `AddCharacter` | ~5 | 1 | ✅ |
| `GetLivingMembers` | ~8 | 1 | ✅ |
| `IsPartyWiped` | ~5 | 1 | ✅ |
| `GetActiveSpeaker` | ~5 | 1 | ✅ |
| `ReorderParty` | ~5 | 1 | ✅ |
| `AwardXPToParty` | ~10 | 1 | ✅ |
| `InitialiseTestParty` | ~20 | 1 | ✅ Hardcoded 4-party creation |
| `AddXPToCharacter` | ~8 | 1 | ✅ |
| `ApplyDeathToCharacter` | ~5 | 1 | ✅ |
| `UpdateCharacterHP` | ~5 | 1 | ✅ |
| **`LoadSavedParty`** | **~15** | **1** | **✅ NEW — Checks save exists, ForEach adds to PartyMembers** |
| **`SavePartyToDisk`** | **~15** | **1** | **✅ NEW — Appends last member to existing save or creates new** |

---

*Updated: 2026-07-12 — Added WBP_CharacterCreation (12 functions), WBP_CharacterScreen (6 functions), new BP_PartyManager save/load functions*
