# Gold Box RPG — UE5.7 Blueprint Project Progress Summary
## Session Reference Document

---

## Project Overview

A Gold Box-inspired RPG built in **Unreal Engine 5.7**, Blueprints only. Tactical grid combat, six-character party management, and an original ruleset called **The Threshold System** — see Ruleset Notice below.

> **Ruleset Notice:** GB-079 (the migration to The Threshold System, `Threshold_Ruleset_v1.md`) is now **essentially complete**, with one item deliberately deferred. Done and tested: ability scores, the entire attack resolution system, XP thresholds, saving throws (3 categories, formula-based), and the `ECharacterClass`/`ECondition` enum renames. Deliberately deferred: `ESpellSchool`'s 4→2 shrink, since the magic system isn't built yet. Renaming an enum isn't the same as building mechanics — the class enum reads the new names now, but per-class mechanics and condition tick logic are still separate, not-yet-built tickets.

**Source documents in project:**
- `SSI_Gold_Box_Engine.md` — technical reference (historical research on the original SSI engine — not affected by the ruleset migration, since it documents a different product)
- `Threshold_Ruleset_v1.md` — the original ruleset replacing the legacy placeholder math (target design)
- `GoldBox_UE56_Blueprint_Roadmap_1.md` — full feature roadmap (updated to target The Threshold System for not-yet-built systems)
- `GoldBox_Solo_Dev_Build_Order.md` — phased build order (same)
- `GoldBox_Blueprint_Reference.md` — live Blueprint architecture reference (still accurate to current legacy-math implementation, flagged inline)

---

## Completed Tickets

### Phase 0 — Foundations
- ✅ **GB-001** — Project structure, folders, naming conventions
- ✅ **GB-002** — All core enums created
- ✅ **GB-003** — BP_GameManager (Game Instance)
- ✅ **GB-017** — SCharacter struct
- ✅ **GB-021** — SMonster struct
- ✅ **GB-032** — SCombatant struct
- ✅ **GB-035** — BP_DiceSystem

### Phase 1 — Hardcoded Test Party
- ✅ **GB-018** ⚠ partial — BP_CharacterRules (ComputeTHAC0, ComputeAC, GetStrengthBonus, ComputeSavingThrows) as originally built. **GB-079 update: GetStrengthBonus/ComputeAC deleted and replaced by GetMightBonus/the new ResolveAttack — see GB-079 entry below. ComputeSavingThrows still legacy, not yet migrated.**
- ✅ **GB-020** — BP_PartyManager (AddCharacter, GetLivingMembers, IsPartyWiped)
- ✅ **GB-020a** — Hardcoded test party (Thorin/Fighter, Aldric/Cleric, Seraphel/MagicUser, Mira/Thief)

### Phase 2 — Dungeon Exploration
- ✅ **GB-007** — SDungeonTile, SDungeonMap structs, DT_DungeonMaps
- ✅ **GB-008** — BP_MapManager (LoadMap, GetTile, CheckWall, UpdatePlayerPosition, etc.)
- ✅ **GB-005** — BP_ExplorerPawn (camera, grid coords, facing direction)
- ✅ **GB-006** — Movement system (MoveForward, MoveBackward, TurnLeft, TurnRight, wall collision)
- ✅ **GB-010** — Placeholder dungeon meshes
- ✅ **GB-011** — BP_DungeonGenerator (SpawnGeometryFromMap)

### Phase 2 — Exploration UI (EPIC 003)
- ✅ **GB-012** — WBP_ExplorationHUD (revised layout: automap top-right, party below)
- ✅ **GB-013** — Party Status Panel (built into GB-012)
- ✅ **GB-014** — WBP_TextLog + BP_TextManager
- ✅ **GB-015** — WBP_CommandMenu

### Phase 3 — Scripted Encounter (EPIC 007)
- ✅ **GB-025** — ETriggerType framework, STriggerEvent struct, DT_TriggerEvents
- 🔄 **GB-026** — BP_EventManager (partial — Phase 3 scope complete, Phase 10 deferred)
- ✅ **GB-027** — S_Encounter struct, S_EncounterChoice struct, EOutcomeType enum, DT_Encounters (1 row)
- ✅ **GB-028** — BP_EncounterManager (StartEncounter, ResolveChoice, TriggerCombatFromEncounter stub, GrantReward stub)
- ✅ **GB-029** — WBP_EncounterScreen, WBP_ChoiceButton — fully implemented and tested
- ✅ **GB-016** — WBP_EncounterPortrait stub — S_MonsterPortrait, DT_MonsterPortraits, ShowPortrait/HidePortrait wired

### Phase 4a — Combat Grid and Camera (EPIC 008)
- ✅ **GB-030** — BP_CombatGrid (all functions built and tested — corridor-shaped arena verified at 40 instances for player position (0,3))
- ✅ **GB-031** — CombatCameraActor (instant cut transition verified, full arena visible, exploration camera returns to correct height)

### Phase 4b — Combat Data and Turn Flow (EPIC 008)
- ✅ **GB-022** — DT_Monsters seeded: Goblin (ID=1), Orc (ID=2), Skeleton (ID=3, IsUndead), Zombie (ID=4, IsUndead), Giant Rat (ID=5, HasPoison, HitDice=0.5). SMonster struct fixed: MoraleRating and XPValue changed from Boolean to Integer
- ✅ **GB-033** — BP_CombatManager (StartCombat refactored into sub-functions: BuildCombatants, BuildInitiativeOrder, SortInitiativeOrder, SpawnCombatantMarkers, TransitionToCombat. Full flow verified end to end)
- ✅ **GB-034** — Combat Turn Flow (StartPlayerTurn, StartEnemyTurn, OnActionComplete, FindStartCombatant, BuildInitiativeOrder, SortInitiativeOrder). BP_CombatantMarker visual debug markers. FindNearestTraversableTile added to BP_CombatGrid. RollDice moved to BPL_RulesLibrary. Full turn flow verified.

### Phase 4c — Attack Resolution (EPIC 008/009)
- ✅ **GB-036** ✅ migrated under GB-079 — ResolveAttack on BPL_RulesLibrary, originally built with THAC0−AC d20 math. **Fully rewritten during GB-079**: now a percentage-based roll (`Strike Number + modifiers − Defense Rating`, clamped 5–95%, 1d100), no more natural-20/natural-1 special-casing (the clamp replaces it). Verified live against the goblin and all four other VS monsters.
- ✅ **GB-043** — Attack Action (VS scope): A key → EnterTargetSelectMode → click monster marker → ExecutePlayerAttack. Full attack flow verified. IsMovementLocked used for combat lock. Encounter re-trigger prevented via Story Flag.
- ✅ **GB-042** — Move Action (VS scope): M key → EnterMoveMode → WASD moves combatant → highlights refresh. BP_TileHighlight actor for visual tile highlighting. GetValidMoveTiles filters blocked/occupied tiles. TilesMovedThisTurn tracking. EndPlayerTurn (T key) to pass turn. Return Node fix resolves duplicate turn message after enemy turn.
- ✅ **GB-046 (partial)** — CheckVictory + EndCombat. Victory when all monsters HP≤0. Returns camera, unlocks movement, clears combat state.

### Phase 4d — Enemy AI (EPIC 008)
- ✅ **GB-045** — BP_EnemyAI full movement AI complete. RunEnemyTurn dispatches via ChooseAction (MoveToTarget/MeleeAttack/CastSpell/Flee/SkipTurn/CastAOESpell). BFS pathfinding (FindPathBFS) navigates to nearest party member and fleeing edge tiles with GetAdjacentTiles/BFS stack/OccupancyMap, avoiding blocked and occupied tiles (target tile exempted via NOT IsOccupied OR Neighbour==End gate). Three movement branches (Morale Fleeing, MoveToTarget, Switch Flee) all ClearOccupant before FindPathBFS, grab path[1] for actual next step. **TODO:** Cast Spell and Cast AOESpell are PrintString stubs only; Incapacitated target priority and AoE opportunity detection deferred.

### Phase 4e — Combat HUD and XP (EPIC 008/EPIC 004)
- ✅ **GB-004a** — WBP_CombatHUD stub complete. Bottom action bar (Move/Attack/EndTurn/Flee buttons, round counter and turn indicator placeholders) shown/hidden directly by BP_CombatManager via CombatHUDRef — OnGameStateUpdated binding does not work for widgets not yet in viewport, so direct manager-driven show/hide is used instead (same pattern as WBP_EncounterScreen).
- ✅ **GB-041** — BP_XPManager complete. **GB-079:** GetConstitutionHPBonus replaced by GetVigorBonus, THAC0 references replaced by StrikeNumber. XP thresholds now use formula-based `GetXPThreshold(Class, Level)` — BP_XPManager complete (VS scope), as originally built. **GB-079 update:** GetConstitutionHPBonus replaced by GetVigorBonus, THAC0 field references replaced by StrikeNumber throughout the level-up flow. XPRequired column in DT_LevelProgression is still legacy — not yet migrated to a formula. Original verification: goblin defeat (XPValue=20) splits 5 XP to each of 4 party members; forced high-XP test confirmed all four characters correctly leveled up.

### Phase 4f — Minimum Conditions (GB-039) ✅
- (see original document for full condition system details — unchanged from prior session)

### Phase 4h — Multiple Monster Spawning + Initiative Fix
- (see original document — unchanged from prior session)

### Phase 4i — Ambush Strike (GB-044) ✅
- (see original document — unchanged from prior session)

---

### Phase 7 — Character Creation (GB-019) 🚧 IN PROGRESS

**WBP_CharacterCreation** — 7-step wizard widget built on a WidgetSwitcher pattern.

✅ **Steps 0–4 complete:**
- **Step 0 — RollStatsPanel:** 6 ability score rows (Might through Presence), each with DiceRoll(3,6) button, 3 re-rolls, disabled at 0 remaining. `ValidateRollStats()` function gates step 6.
- **Step 1 — ChooseRacePanel:** 7 race buttons (Human/Elf/Dwarf/Gnome/Halfling/Half-Elf/Half-Orc), `SelectRace(E_Race)` function with highlight logic, `bChooseRace` gate flag.
- **Step 2 — ChooseClassPanel:** 9 class buttons (Warden through Infiltrator), `SelectClass()` with highlight, `bChooseClass` gate flag.
- **Step 3 — ChooseAlignmentPanel:** 7 morality buttons (Righteous through Ruthless), `SelectAlignment()` with highlight, `bChooseAlignment` gate flag.
- **Step 4 — NameEntryPanel:** EditableTextBox with Random/Clear buttons, `GenerateRandomName()` Pure function (syllable-based), `OnTextChanged` updates `CharacterName` (Text). Gate checks non-empty.

🚧 **Step 6 — ConfirmPanel:** Two-column summary layout (Character Summary left, Ability Scores right). Widget tree complete with 10 variable TextBlocks. Three Pure enum-to-text helpers built: `GetRaceDisplayName`, `GetClassDisplayName`, `GetAlignmentDisplayName`.

❌ **Remaining:** `PopulateConfirmPanel` function, Next button branch for step 6 save flow, `Make S_Character` → `Add Data Table Row(DT_CharacterRoster)` → `Remove from Parent`. PortraitPanel (step 5) still empty placeholder.

**DataTable:** `DT_CharacterRoster` created at `/Game/Blueprints/Data/DT_CharacterRoster` with `S_Character` row struct (empty).

**Next button gate:** Full AND-chain: `(step!=6 OR ValidateRollStats) AND (step!=1 OR bChooseRace) AND (step!=2 OR bChooseClass) AND (step!=3 OR bChooseAlignment) AND (step!=4 OR CharacterName!="")`.

---

### GB-079 — Ruleset Migration to The Threshold System (Complete)
- (see original document for full GB-079 details — unchanged from prior session)

---

### Phase 3 Test Checkpoint — PASSING ✅
- (see original document — unchanged from prior session)

---

## Current State (End of Session)

- Phase 7 character creation: Steps 0–4 fully built, step 6 widget tree done. Remaining: PopulateConfirmPanel, save logic, PortraitPanel
- Encounter system fully working end-to-end
- Full VS combat loop verified
- Dungeon renders correctly in PIE
- Ambush Strike (GB-044) complete
- GB-079 ruleset migration complete (ESpellSchool deferred)

---

## Known Issues / Outstanding Bugs

| Issue | Location | Notes |
|---|---|---|
| Character creation save logic | WBP_CharacterCreation | PopulateConfirmPanel + save flow not yet built |
| PortraitPanel | WBP_CharacterCreation index 5 | Empty placeholder |
| Player camera not centred in viewport | BP_ExplorerPawn | Deferred — polish task |
| Standalone mode dungeon not rendering | BP_GameManager → InitialiseWorld | DungeonGenerator None error — works in PIE Selected Viewport |
| CommandMenu visible during Encounter | WBP_ExplorationHUD → RefreshCommandMenu | Deferred to polish pass |
| Portrait area white placeholder | WBP_EncounterScreen | Deferred to art pass with GB-016 |
| Monster spawn zone | BP_CombatManager → BuildCombatants | FindNearestTraversableTile starts at (14,8) — may land on wrong side for some maps. Proper spawn zone logic post-VS |

### Resolved
| Issue | Resolution |
|---|---|
| Party showing 8 members instead of 4 | Fixed — duplicate InitialiseTestParty call removed |
| (see original document for full resolved list — unchanged) |

---

### Deferred Polish Items (post-VS)
- CommandMenu visibility during Encounter state
- Portrait art — populate DT_MonsterPortraits with real textures
- Player camera centring in viewport
- Standalone mode dungeon rendering fix
- HUD font sizes too small in exploration mode
- **Viewport rendering overhaul (major post-VS task):** Currently both the dungeon camera and combat camera render full-screen behind the HUD overlay. Switch to Render Target 2D approach. Deferred to art/polish pass.
- Movement range rules system redesign (currently hardcoded to 3 for VS testing)
- Adjacency check for attacks (currently can attack from any distance)
- Diagonal movement consideration (Chebyshev allows 8-directional)
- **GB-079 already complete — only ESpellSchool 4-to-2 shrink deferred (magic system not built yet).**

---

*Document updated: GB-019 character creation in progress (Steps 0-4 complete, ConfirmPanel widget tree built, DT_CharacterRoster created, enum-to-text helpers built). Next: PopulateConfirmPanel + save logic.*
*UE5.7 · Blueprints Only · Solo Dev*
