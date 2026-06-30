# Gold Box RPG — TODO / Deferred Items
## UE5.7 · Blueprints Only · Solo Dev

---

## Recent Fixes

| Item | Notes |
|---|---|
| GetAdjacentTiles DX/DY swap | NX was X+DY, NY was Y+DX (swapped). Fixed to NX=X+DX, NY=Y+DY. BFS now produces correct paths |
| FindPathBFS target tile exemption | IsTraversable blocked ALL occupied tiles including targets. Added NOT IsOccupied OR (Neighbour==End) gate so BFS can step onto target's tile |
| ClearOccupant before FindPathBFS | All three RunEnemyTurn movement branches now clear start tile before BFS runs, preventing self-blocking |
| GetArrayItem index 0→1 | All three branches grab path[1] instead of path[0] (which was the start tile, causing no-ops) |
| Enemies occupying same tile | IsOccupied added to IsTraversable; start tile cleared before BFS; target tile exempted from occupancy check |
| BuildInitiativeOrder Loop Body disconnected | ForEach Loop Body was not wired to DiceRoll after refactor. All combatants had Initiative=0. Reconnected. |

---

## Conditions (GB-039) — 3 Remaining

| Condition | Blocker | Notes |
|---|---|---|
| **Diseased** | Game-time/calendar system | Vigor reduced by 1 per day until cured. Needs a day/night or rest system to trigger |
| **Sapped** | Level-down logic | Permanent loss of 1 character level. Needs a reverse-level-up procedure (recompute SN, MaxHP, remove spell slots, etc.) |
| **Petrified** | Damage-type system | Treat as Paralysed, immune to damage except force/sonic. Needs a damage-type enum and resistance checks |

---

## Combat — Deferred from Phase 4

| Item | Ticket | Notes |
|---|---|---|
| Live HP sync during combat | New (GB-039a) | `SCharacter` in `BP_PartyManager` only updates on death, not on hits. Party panel shows stale HP for living damaged characters |
| Full defeat/game-over screen | GB-046 | `CheckDefeat` stub calls `EndCombat(false)` — needs proper game-over UI |
| Multi-condition display | GB-039 | Party panel only checks `Contains(Dead)`. Needs multi-icon or scrolling condition display |
| Downed marker visual polish | GB-039 | Currently 0.5 scale only — needs material tint, icon, or animation |
| Monster AI multi-attack | GB-037 | `GetAttacksThisRound` works for player but `ExecuteEnemyAttack` doesn't use it yet. Current monsters all have `NumAttacks=1` (harmless) |
| Adjacency check for attacks | GB-042 | Complete. S_Monster/S_Combatant.AttackRange fields, GetCombatDistance in BPL_RulesLibrary, range gate in ExecutePlayerAttack and ExecuteEnemyAttack. Monsters default melee (range=1), players hardcoded range=1 until weapon tables exist |
| Dynamic combat camera | GB-047 | **Partially built:** IA_CombatPan (Axis2D), IA_CombatZoom (Axis1D), IMC_Combat (arrow keys pan + mouse wheel zoom), BP_CombatCamera (SpringArm + Camera + Tick pan/zoom bounds clamp + SnapToTarget), TransitionToCombat spawn/IMC/SetViewTarget wired. StartPlayerTurn SnapToTarget + EndCombat destroy/remove IMC not yet wired. **Blocked:** CombatHUD overlays the viewport behind UI panels. Needs proper viewport/UI partitioning system (SubViewport or similar) first. | **Plan:** (1) Create `IA_CombatPan` (Value2D) + `IA_CombatZoom` (1D Axis). (2) Create `IMC_Combat` mapping context (WASD/arrows=pan, mouse wheel=zoom, higher priority than IMC_Dungeon). (3) Create `BP_CombatCamera` Actor: SpringArm (pitch -60, length 2000) + Camera + Tick pan/zoom logic + bounds clamp (16x16x200uu arena) + `SnapToTarget` function. (4) `TransitionToCombat`: spawn camera, add IMC, SetViewTarget. (5) `StartPlayerTurn`: FindMarkerByID -> SnapToTarget. (6) `EndCombat`: destroy camera, remove IMC. (7) Lock pan/zoom during enemy turns. Plan: `Saved/.Aura/plans/dynamic_combat_camera_v1.md` |
| Movement range redesign | GB-042 | Currently hardcoded to 3 for VS. Needs rules-based system (class/race/armour-based) |
| Hybrid Ambush classes (Skirmisher/Shadowpriest/Infiltrator) | GB-044 | Half multiplier for Skirmisher/Shadowpriest, one tier ahead for Infiltrator. Deferred until hybrid class system built |
| Party-only filter for IsFlanked | GB-044 (polish) | Currently passes all Combatants (party + monsters) — works fine but a filtered party-only array would be cleaner |
| Full Enemy AI | GB-045 | ✅ Movement complete: BFS pathfinding to nearest target, occupancy-aware traversability, flee-to-edge pathfinding. TODO: Incapacitated target priority, intelligent spellcasting, AoE opportunity detection |

---

## Spell System — Stubs (GB-045)

| Branch | Function | Current state |
|---|---|---|
| Cast Spell | RunEnemyTurn -> ChooseAction -> SwitchEnum | TODO: PrintString "casts a spell" only. Needs spell selection, target selection, damage/heal/effect resolution |
| Cast AOESpell | RunEnemyTurn -> ChooseAction -> SwitchEnum | TODO: PrintString "casts AoE spell" only. Needs AoE target pattern, multi-target resolution |

---

## Rules Engine — Deferred

| Item | Ticket | Notes |
|---|---|---|
| ResolveSave consumer wiring | GB-038 | `ResolveSave` built but nothing calls it yet. GB-039 poison ticks call Fortitude. Still needs: Ambush Strike (GB-044) Reflex, retreat (GB-046) Reflex |
| Morale system | GB-040 | ✅ Core complete: E_MoraleState enum, S_Combatant.MoraleState + GroupID, ResolveMorale(GroupID) in BP_CombatManager, Normal->Shaken->Fleeing transitions, Fleeing monsters skip turn. Remaining: flee-to-map-edge + removal from combat. **Polish:** base morale check on total morale rating of the monster group (sum all monsters' MoraleRating) rather than per-monster |
| Flee-to-map-edge | GB-040 | Fleeing monsters currently skip turn in place. Full flee mechanic (move toward nearest map edge, exit combat on arrival) still deferred |
| Ambush Strike | GB-044 | Rogue condition check, Threshold System multiplier table (L1-3 x2, L4-6 x3, L7-9 x4, L10 x5) |
| Full Enemy AI | GB-045 | Incapacitated target priority, intelligent spellcasting, AoE opportunity, morale flee |
| Combat retreat | GB-046 | Reflex check, partial flee, full party retreat handling |

---

## Test Suite

| Item | Notes |
|---|---|
| DT_THAC0Tests, DT_StrengthBonusTests, DT_AttackResolutionTests, DT_XPThresholdTests, DT_SavingThrowTests, DT_LevelCapTests | All stale after GB-079 migration. Decided to delete and recreate fresh rather than patch individually |

---

## Ruleset Migration (GB-079)

| Item | Notes |
|---|---|
| ESpellSchool 4->2 shrink | Currently MagicUser/Illusionist/Cleric/Druid. Threshold target: Arcane/Divine. Deferred until magic system built (Phase 8) |

---

## Deferred Polish Items

| Item | Notes |
|---|---|
| Player camera centring in viewport | BP_ExplorerPawn — polish task |
| Standalone mode dungeon rendering | DungeonGenerator None error, works in PIE Selected Viewport |
| CommandMenu visible during Encounter | WBP_ExplorationHUD -> RefreshCommandMenu |
| Portrait area white placeholder | WBP_EncounterScreen — deferred to art pass |
| Monster spawn zone logic | BP_CombatManager -> BuildCombatants — FindNearestTraversableTile starts at (14,8), may land wrong side for some maps |
| HUD font sizes | Too small in exploration mode |
| Viewport rendering overhaul | Switch to Render Target 2D approach instead of full-screen behind HUD |
| BP_CharacterRules audit | Two functions found living there instead of BPL_RulesLibrary (GetStrengthBonus, ComputeSavingThrows). Worth checking what else remains there |

---

## Named CharacterID Constraint

`SCharacter.Name` is not guaranteed unique — any character-matching logic must use `CharacterID`, never `Name`. CharacterID is auto-assigned sequentially by `BP_PartyManager.AddCharacter`.

---

*Updated: Enemy AI BFS movement fixes + spell stubs added.*