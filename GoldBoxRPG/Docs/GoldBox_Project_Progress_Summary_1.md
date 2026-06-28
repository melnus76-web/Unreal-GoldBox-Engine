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
- ✅ **GB-045** — BP_EnemyAI stub complete (VS scope). Spawned by BP_GameManager, CombatManagerRef set by StartEnemyTurn. Three functions: FindNearestPartyMember (Chebyshev distance, skips IsMonster + HP≤0), MoveOneStepToward (GetAdjacentTiles → IsTraversable + IsOccupied filter → pick closest tile → ClearOccupant/SetOccupant), ExecuteEnemyAttack (ResolveAttack → ApplyDamage → CheckVictory). Two shared utility functions added to BP_CombatManager: UpdateCombatantGridPosition and MoveMarkerToTile (reusable by both player and enemy movement). ApplyDamage added to BP_CombatManager (updates SCombatant HP in Combatants array). Verified: goblin moves one tile per turn toward nearest party member, respects traversability and occupancy, attacks correct target, combat continues correctly after hit, combat ends correctly on goblin death.

### Phase 4e — Combat HUD and XP (EPIC 008/EPIC 004)
- ✅ **GB-004a** — WBP_CombatHUD stub complete. Bottom action bar (Move/Attack/EndTurn/Flee buttons, round counter and turn indicator placeholders) shown/hidden directly by BP_CombatManager via CombatHUDRef — OnGameStateUpdated binding does not work for widgets not yet in viewport, so direct manager-driven show/hide is used instead (same pattern as WBP_EncounterScreen).
- ✅ **GB-041** — BP_XPManager complete. **GB-079:** GetConstitutionHPBonus replaced by GetVigorBonus, THAC0 references replaced by StrikeNumber. XP thresholds now use formula-based `GetXPThreshold(Class, Level)` — BP_XPManager complete (VS scope), as originally built. **GB-079 update:** GetConstitutionHPBonus replaced by GetVigorBonus, THAC0 field references replaced by StrikeNumber throughout the level-up flow. XPRequired column in DT_LevelProgression is still legacy — not yet migrated to a formula. Original verification: goblin defeat (XPValue=20) splits 5 XP to each of 4 party members; forced high-XP test confirmed all four characters correctly leveled up.

---

### Phase 4f — Minimum Conditions (GB-039) ✅

- ✅ **Three shared functions built on `BP_CombatManager`:** `ApplyCondition(CombatantID, NewCondition)`, `RemoveCondition(CombatantID, ConditionToRemove)`, `HasCondition(CombatantID, ConditionToCheck) → bHasCondition`. All use the same find-by-ID/break/promote-to-local-variable/modify/make-struct/set-array-elem pattern established by `ApplyDamage`. `HasCondition` exits the loop immediately on match via Return Node, with a fallback Return `false` on Completed.
- ✅ **`SetMarkerDowned` built on `BP_CombatManager`:** duplicated from `RemoveMarkerForCombatant`, swapped `Destroy Actor` for `Set Actor Scale 3D (0.5, 0.5, 0.5)`. Visual placeholder — material tint/icon deferred to polish pass.
- ✅ **Dead condition applied in `ExecutePlayerAttack`:** existing `NewCurrentHP <= 0` Branch extended via Sequence — Then 0 = existing marker destroy logic, Then 1 = `ApplyCondition(CombatantID, Dead)`.
- ✅ **Dead condition applied in `ExecuteEnemyAttack`:** new death check built from scratch (none existed previously). `NewHP = CurrentHP (from Break SCombatant) - Damage`, Branch ≤0 → Sequence: Then 0 = `CombatManagerRef.ApplyCondition(Dead)`, Then 1 = `CombatManagerRef.SetMarkerDowned(CombatantID)`, Then 2 = `PartyManagerRef.ApplyDeathToCharacter(CharacterID)`.
- ✅ **Turn-order guard clause in `OnActionComplete`:** `HasCondition(Dead) OR HasCondition(Restrained)` → True: `OnActionComplete` (recurse/skip), False: `IsPlayerControlled` → `StartPlayerTurn`/`StartEnemyTurn`. **Critical bug found and fixed:** the For Each loop that determines whose turn it is was missing an ID-match Branch, so it was dispatching start-turn logic for every single combatant in the array rather than just the matching one. With 5 combatants, this caused exponential recursion (5 → 25 → 125 calls) hitting UE's infinite-loop detector. Added `PlayerOrMonsterID == CombatantID` Branch as a gate inside the loop, Continue on False.
- ✅ **`CheckDefeat` stub added to `OnActionComplete`:** checks if all non-monster combatants have `Dead` condition before turn dispatch — if so, posts "Defeat" message and calls `EndCombat`. Stub — proper game-over/defeat screen deferred to GB-046.
- ✅ **Restrained auto-hit wired into `ExecutePlayerAttack`:** Branch on `HasCondition(DefenderID, Restrained)` inserted after "Set Defender ID" step. True path: `Make SAttackResult(bHit=true, RollResult=0, RollNeeded=0, Damage=RollDice(1,6))` → Set `ResolveAttackResult` → feeds same `Break SAttackResult` as normal path. `bWasAutoHit` Boolean controls which message displays: normal path shows roll-based message, auto-hit path shows `"[Attacker] hits for [Damage] damage! (Auto Hit)"`.
- ✅ **Restrained auto-hit wired into `ExecuteEnemyAttack`:** same pattern. `Break SCombatant` on Target → `CombatManagerRef.HasCondition(Restrained)` → Branch before `ResolveAttack`. Auto-hit path sets local `Attack Result` variable directly. `bWasAutoHit` controls message.
- ✅ **`CharacterID` field added to `SCombatant` struct:** populated from `SCharacter.CharacterID` in `BuildCombatants` for party members (left at default 0 for monsters). Required to correlate combat-side data back to `BP_PartyManager`'s `SCharacter` array for death sync.
- ✅ **`SCharacter.Conditions` changed from single `ECondition` to `Array of ECondition`:** breaking change that required updating `UpdateCharSlot` in `WBP_ExplorationHUD` (now uses `Contains(Dead)` check rather than direct enum match). Multi-condition display deferred post-VS.
- ✅ **`ApplyDeathToCharacter` built on `BP_PartyManager`:** loops `PartyMembers`, matches by `CharacterID`, adds `Dead` to Conditions array, sets `CurrentHP=0`, writes back, fires `OnPartyUpdated` → `WBP_ExplorationHUD.RefreshPartyPanel`. `PartyManagerRef` added to `BP_EnemyAI`, set in `StartEnemyTurn` alongside existing `CombatManagerRef`.
- ✅ **`OnMessagePosted_Event` loop bug fixed:** an erroneous `RefreshPartyPanel` call had been added to `OnMessagePosted_Event` in `WBP_ExplorationHUD` in a prior session. When all party members died, this created an infinite loop (message posted → refresh → condition check posted another message → loop). Removed.
- **Deferred:** live HP sync for all hits during combat (only death is currently synced to `SCharacter` — a new ticket needed), full defeat/game-over screen (GB-046), multi-condition display in party panel, downed marker visual polish.

---

## Phase 3 Test Checkpoint — PASSING ✅

- ✅ Walk to encounter tile (X=4, Y=3) → encounter screen appears
- ✅ Intro text displays in encounter screen text area
- ✅ Three choice buttons appear: "Speak Nicely", "Threaten Them", "Attack!"
- ✅ "Speak Nicely" → flee text posts to log, screen disappears, player can move
- ✅ "Threaten Them" → same
- ✅ "Attack!" → TriggerCombatFromEncounter stub fires (print confirms call)
- ✅ Encounter screen scales correctly with resolution changes
- ✅ Player movement blocked during encounter (CanMove = false)
- ✅ Mouse cursor visible and clickable during encounter

---

### GB-079 — Ruleset Migration to The Threshold System (Complete)
- ✅ **Ability scores migrated.** `SCharacter` renamed Strength→Might, Intelligence→Acuity, Wisdom→Resolve, Dexterity→Reflex, Constitution→Vigor, Charisma→Presence. `StrengthExceptional` deleted entirely. Verified via `DT_TestParty` — all four characters' data carried through the rename cleanly, no loss.
- ✅ **GetMightBonus and GetVigorBonus built** on `BPL_RulesLibrary`, replacing `GetStrengthBonus`/`GetConstitutionHPBonus` (deleted, not kept as wrappers). New breakpoints, no exceptional-score tier. `GetVigorBonus` wired into `BP_XPManager.CheckLevelUp`'s HP roll, tested via level-up.
- ✅ **Combat math fully rebuilt.** `AC`→`DefenseRating`, `THAC0`→`StrikeNumber` renamed on `SCharacter`, `SCombatant`, `SMonster`, and the `SLevelProgression` column. **Design correction caught before implementation:** the original ruleset doc specified `Required Roll = Strike Number − Defense Rating` on a d20 — this was backwards (ascending Defense Rating would make better-armored targets easier to hit). Decided to switch to a percentage-based system entirely rather than just fix the operator, since a d20's 20-point range was too tight to calibrate against future bonus stacking (gear, spells, conditions). New formula: `Final Hit Chance = clamp(StrikeNumber + MightBonus + WeaponBonus + SituationalModifier − DefenseRating, 5, 95)`, roll 1d100, hit if roll ≤ Final Hit Chance. Natural-20/natural-1 special-casing removed — the clamp does that job now.
- ✅ **ResolveAttack rewritten from scratch** on `BPL_RulesLibrary` rather than surgically edited, since the logic shape changed enough (different die, no special-case branches) that a clean rebuild was cleaner than patching.
- ✅ **Data converted and tested.** `DT_TestParty`'s four characters set to StrikeNumber 50 (new level-1 baseline, same for every class). `DT_LevelProgression`'s StrikeNumber column updated to the new ascending percentages for all 20 rows (Fighter/Cleric/MagicUser/Thief × levels 1–5) — `XPRequired` column deliberately left untouched, that's separate unmigrated work. All 5 monsters in `DT_Monsters` converted using a derived linear formula anchored to the goblin's first conversion (`New DR = (10 − OldAC) × 2.5`, `New SN = 50 + (20 − OldTHAC0) × 5`): Goblin 10/50, Orc 10/55, Skeleton 8/55, Zombie 5/50, Giant Rat 3/50.
- ✅ **Verified live in combat** — fought the goblin with the new math, hit/miss rates felt right (~40% player-vs-goblin, ~50% goblin-vs-unarmored-party), damage and combat-end still correct.
- ✅ **XP thresholds migrated to a formula.** Built `BPL_RulesLibrary.GetXPThreshold(Class, Level)` implementing `500 × Level × (Level−1) × ClassMultiplier` (Fighter 1.0, Cleric 1.1, MagicUser 1.2, Thief 0.9, others default 1.0 unused) via a Select node keyed on the class enum. `CheckLevelUp` now compares XP against this formula's result instead of reading `XPRequired` from the `DT_LevelProgression` row — that column is now unused, left in the table for reference only rather than restructuring the struct. `ApplyLevelUp`'s parameter renamed `NewTHAC0`→`NewStrikeNumber` to match (a lingering rename gap from the earlier combat math migration, caught and fixed during this step). Tested: temporarily set goblin XPValue to 10000 (2500 XP per character) — correctly leveled all four characters to exactly level 2 (thresholds 900–1200 depending on class) without overshooting to level 3 (thresholds 2700–3600), confirming the formula's per-level math is correct.
- ⚠ **Side effect discovered:** `BP_RulesTestSuite`'s `DT_THAC0Tests`, `DT_StrengthBonusTests`, `DT_AttackResolutionTests`, and `DT_XPThresholdTests` now test deleted/changed functions and are stale. **Decision: delete and recreate these test tables fresh at a later date rather than patch them** — `DT_SavingThrowTests` and `DT_LevelCapTests` are likely also affected by the saves/class migration below and are candidates for the same treatment.
- ✅ **ECondition renamed** — Held→Restrained, Hasted→Quickened, LevelDrained→Sapped. Clean compile, no errors (low blast radius since conditions are barely built yet — only Dead + Held/Restrained wired up for VS).
- ✅ **Saving throws fully migrated.** `ESaveType` restructured from 5 categories (VsPoison/VsWands/VsPetrification/VsBreathWeapon/VsSpells) to 3 (Fortitude/Reflex/Willpower) — no 1:1 mapping existed, so this was a clean delete-and-recreate rather than a rename. Built a new shared `BPL_RulesLibrary.GetAbilityModifier(Score)` function (same bonus shape as GetMightBonus's ToHit column) for the formula's ability modifier term. Rebuilt `ComputeSavingThrows(Class, Level, Vigor, Reflex, Resolve) → FortitudeSave, ReflexSave, WillpowerSave` implementing `16 − floor(Level/2) − GetAbilityModifier(relevant score) − ClassBonus`, with class bonuses (+2 to one save) via three Select nodes: Fighter→Fortitude, Cleric→Willpower, MagicUser→Willpower, Thief→Reflex. **Found `ComputeSavingThrows` living in `BP_CharacterRules` rather than `BPL_RulesLibrary`** — second occurrence of this doc/reality mismatch after `GetStrengthBonus` earlier in the migration. Deleted from `BP_CharacterRules`, rebuilt in `BPL_RulesLibrary`. Flagged a full audit of `BP_CharacterRules`'s remaining contents as worth doing, not yet done. Not yet tested live since nothing currently triggers a saving throw (GB-038 not built) — ready and waiting.
- ✅ **ECharacterClass fully renamed** — all 9 values: Fighter→Warden, Cleric→Devout, MagicUser→Adept, Thief→Rogue (core 4), and Paladin→Templar, Ranger→Skirmisher, Druid→Sylvan, Illusionist→Shadowpriest, Assassin→Infiltrator (5 hybrid slots — renamed rather than deleted, since the ruleset's 5 sketched hybrid roles exactly matched the 5 available non-core slots). Clean compile. **Knock-on fix required:** `DT_LevelProgression`'s row names are built at runtime via `Enum to String(Class)` inside `CheckLevelUp` — renaming the enum changed what that produces (e.g. "Fighter_2"→"Warden_2"), so all 20 existing rows had to be renamed to match (Fighter_1-5→Warden_1-5, Cleric_1-5→Devout_1-5, MagicUser_1-5→Adept_1-5, Thief_1-5→Rogue_1-5) or every level-up lookup would have failed with Row Not Found. Verified: full level-up flow re-tested after the row rename, works correctly.
- **Still outstanding for GB-079:** only `ESpellSchool`'s 4→2 shrink remains, deliberately deferred since the magic system isn't built yet. Everything else is migrated.
- `Threshold_Ruleset_v1.md` updated to match — the broken formula corrected, the percentage system documented as what's actually built, and a build-status note added distinguishing implemented sections from design-only ones.

---

## Current State (End of Session)

- Full Phase 3 encounter system working end to end
- Dungeon renders correctly in PIE
- **Full VS combat loop verified end to end:**
  - Walk dungeon → trigger encounter → choose Attack! → combat starts
  - Party markers at bottom, monster at top, all on traversable tiles
  - Player turn message appears once per turn
  - M key → cyan highlights show valid move tiles
  - WASD → combatant moves, highlights refresh
  - T key → end turn, advances to next combatant
  - Goblin moves one tile toward nearest party member each turn
  - Goblin respects traversability and occupancy (won't enter blocked or occupied tiles)
  - Goblin attacks nearest party member — hit/miss/damage logged correctly
  - Combat continues correctly after goblin hits party member
  - Round counter increments correctly
  - A key → select target → click monster → attack resolves
  - Hit/miss/damage logged correctly
  - Monster defeated → marker removed → Victory! → return to exploration
  - Encounter does not re-trigger after combat
  - Combat grid renders blocked tiles as visible wall blocks (WallMeshISM, brick placeholder), giving clear corridor boundaries on the tactical grid
  - XP awarded on victory, split evenly across living party members, each character checked for level-up; level-up correctly updates Level/THAC0/MaxHP/CurrentHP and posts a message

---

## Known Issues / Outstanding Bugs

| Issue | Location | Notes |
|---|---|---|
| Player camera not centred in viewport | BP_ExplorerPawn | Deferred — polish task |
| Standalone mode dungeon not rendering | BP_GameManager → InitialiseWorld | DungeonGenerator None error — works in PIE Selected Viewport |
| CommandMenu visible during Encounter | WBP_ExplorationHUD → RefreshCommandMenu | Deferred to polish pass |
| Portrait area white placeholder | WBP_EncounterScreen | Deferred to art pass with GB-016 |
| Monster spawn zone | BP_CombatManager → BuildCombatants | FindNearestTraversableTile starts at (14,8) — may land on wrong side for some maps. Proper spawn zone logic post-VS |

### Resolved
| Issue | Resolution |
|---|---|
| Party showing 8 members instead of 4 | Fixed — duplicate InitialiseTestParty call removed |
| Movement stutters briefly | Fixed — input now uses Started trigger for forward movement |
| Video memory warning | Not a bug — laptop with 16GB shared memory, not a project issue |
| SpawnGrid showing 288 instances | Fixed — Subtract node was Add (Height+1=17 instead of Height−1=15) |
| IsTileInBounds not found in search | Fixed — typo IsTIleInBounds renamed to IsTileInBounds |
| GetAdjacentTiles not found in search | Fixed — typo GetAdjacentTIles renamed to GetAdjacentTiles |
| SetBlocked/IsBlocked/IsTraversable missing | Rebuilt — lost during earlier session, all three recreated and verified |
| Wrong NOT node (Not Equal instead of NOT Boolean) | Fixed — replaced != with single-pin NOT Boolean node |
| GenerateGridFromDungeon instances = 0 | Fixed — Subtract nodes had −1 instead of 1 (PlayerTileY−(−1) = +1, giving wrong window origin) |
| MapManagerRef None error | Fixed — Level Blueprint Delay added (0.1s) to ensure GameManager SpawnManagers completes before GenerateGridFromDungeon fires |
| Markers appearing off grid | Fixed — FindNearestTraversableTile added; GenerateGridFromDungeon now runs before BuildCombatants |
| Initiative order always empty | Fixed — inner For Each Loop Body exec not connected to Branch; outer loop Index not wired to GetTilesInRange Range pin |
| Markers all same colour | Fixed — wrong IsMonster bool being passed to Initialise function |
| StartCombat too complex (50+ nodes) | Refactored — extracted BuildCombatants, SpawnCombatantMarkers, TransitionToCombat as separate functions |
| Two combatants stacking on same tile | Fixed — IsOccupied check added to FindNearestTraversableTile; SetOccupant called after each placement |
| Party markers in wrong position | Fixed — party spawns at bottom of arena (PreferredX=0, PreferredY=4+Index) in party order |
| Duplicate player turn message after enemy turn | Fixed — Return Node added to OnActionComplete after StartEnemyTurn call; prevents double OnActionComplete execution |
| Goblin attacks after being defeated | Fixed — HP > 0 check added in OnActionComplete before calling StartEnemyTurn |
| Wrong marker destroyed on death | Fixed — Destroy Actor target was Self (CombatManager) instead of As BP_CombatantMarker cast result |
| Attacker/Defender swapped in ExecutePlayerAttack | Fixed — first For Each matched TargetCombatantID for both; corrected to CurrentCombatantID for attacker |
| GetValidMoveTiles returning blocked tiles | Fixed — GetValidMoveTiles filters IsTraversable AND IsOccupied before adding to result |
| Movement range too large causing infinite loop | Fixed — MovementRange reduced to 3 for VS testing (rules system to be redesigned post-VS) |
| ClearHighlights only clearing one highlight | Fixed — Clear Array was inside Loop Body (fired after first destroy); moved to For Each Completed |
| IsMovementLocked not blocking input during combat | Fixed — CanMove was incorrectly used for combat lock; replaced with IsMovementLocked |
| CheckVictory always returning false | Fixed — IsMonster=false (party members) was routing to SET AllEnemiesDead=false; party members now skipped entirely, only IsMonster=true AND CurrentHP>0 sets the flag |
| ExecutePlayerAttack death check reading wrong HP | Fixed — death check was reading CurrentHP from Break SCombatant (old value) instead of CurrentHP-Damage (new value); promoted subtraction result to local variable NewHP, used for both Make SCombatant and the ≤0 death check |
| Damage calculated before For Each loop found correct combatant | Fixed — moved damage calculation inside the loop, after the matching combatant is found |
| MoveMarkerToTile crash after goblin death | Fixed — cast to BP_CombatantMarker hit a stale/destroyed reference; added Is Valid check after cast to skip destroyed markers |
| Wall blocks overlapping walkable corridor tiles | Fixed — RebuildTileInstances only cleared TileMeshISM (floor), never WallMeshISM (walls added during wall visibility polish); wall instances accumulated across rebuilds and bled into tiles that should have been clear. Added a second Clear Instances call targeting WallMeshISM at the start of RebuildTileInstances |
| GetLivingMembers function silently renamed to GetPartyMembers | Found during GB-041 — function still correctly filtered dead members internally but name no longer matched behaviour. Renamed back to GetLivingMembers for clarity |
| XPManagerRef Accessed None in EndCombat | Fixed — Level Blueprint's Set XPManagerRef node had Target wired to BP_GameManager instead of BP_CombatManager (wrong variable owner); a stray Set node had been created against the wrong Blueprint. Deleted and recreated targeting BP_CombatManager correctly |
| Threshold_Ruleset_v1.md combat formula backwards | Caught before implementation, during GB-079 — doc specified `Strike Number − Defense Rating` on an ascending Defense Rating, which would make better-armored targets easier to hit. Resolved by switching to a percentage-based system entirely (see GB-079 section above) rather than just flipping the operator, since the d20's narrow range was a separate calibration risk |

### Known Constraint
- **SCharacter.Name is not guaranteed unique** — any character-matching logic (XP awards, level-up, future inventory/spell systems) must use CharacterID, never Name. CharacterID is auto-assigned sequentially by BP_PartyManager.AddCharacter based on array length at time of insertion.

---

## Architecture Decisions

### Key Blueprint Structure

```
BP_GameManager (Game Instance)
  ├── SpawnManagers()
  │     ├── Spawns BP_PartyManager
  │     ├── Spawns BP_MapManager
  │     ├── Spawns BP_TextManager
  │     ├── Spawns BP_EventManager
  │     │     ├── Sets MapManagerRef
  │     │     └── Sets TextManagerRef
  │     ├── Spawns BP_DungeonGenerator
  │     ├── Spawns BP_EncounterManager
  │     │     ├── Sets TextManagerRef
  │     │     └── Sets GameManagerRef = Self
  │     ├── Spawns BP_CombatManager
  │     │     ├── Sets GameManagerRef = Self
  │     │     ├── Sets TextManagerRef = TextManager
  │     │     └── Sets PartyManagerRef = PartyManager
  │     ├── Spawns BP_EnemyAI
  │     └── Spawns BP_XPManager
  ├── InitialiseParty() → BP_PartyManager.InitialiseTestParty()
  ├── InitialiseWorld() → BP_MapManager.LoadMap(1)
  └── OnGameStateUpdated (Event Dispatcher)

Level Blueprint (Event Begin Play)
  → Create WBP_ExplorationHUD → Add to Viewport (Z=0)
  → Create WBP_EncounterScreen (NOT added to viewport here)
  → SET BP_EncounterManager.EncounterScreenRef
  → WBP_EncounterScreen.Initialise
  → Create WBP_CombatHUD (NOT added to viewport here)
  → SET BP_CombatManager.CombatHUDRef
  → SET BP_CombatManager.CombatGridRef
  → SET BP_CombatManager.CombatCameraRef
  → SET BP_CombatManager.EnemyAIRef
  → SET BP_CombatManager.XPManagerRef

BP_ExplorerPawn
  └── CompleteMovement()
        ├── SET GridX, GridY
        ├── UpdatePlayerPosition (BP_MapManager)
        ├── EvaluateTriggers (BP_EventManager)
        └── IncrementStepCounter (BP_MapManager)

BP_EncounterManager
  ├── StartEncounter(EncounterID)
  │     ├── Loads S_Encounter from DT_Encounters
  │     ├── ChangeGameState(Encounter)
  │     ├── PostMessage(IntroText)
  │     ├── SetEncounterData on WBP_EncounterScreen
  │     ├── Add WBP_EncounterScreen to Viewport (Z=1)
  │     └── BP_ExplorerPawn.CanMove = false
  └── ResolveChoice(ChoiceIndex)
        ├── Flee    → PostMessage + ChangeGameState(Exploration) + Remove screen + CanMove = true
        ├── TextOnly → PostMessage only
        ├── Reward  → GrantReward + PostMessage + ChangeGameState + Remove screen + CanMove = true
        └── Combat  → TriggerCombatFromEncounter(MonsterGroupID) → StartCombat

BP_CombatManager
  ├── StartCombat(MonsterGroupID)
  │     ├── SET CurrentMonsterGroupID = MonsterGroupID
  │     ├── BuildCombatants, BuildInitiativeOrder, SpawnCombatantMarkers, TransitionToCombat
  │     └── CombatHUDRef.Add to Viewport (Z=1)
  ├── StartPlayerTurn / StartEnemyTurn (→ BP_EnemyAI.RunEnemyTurn) / OnActionComplete
  ├── ExecutePlayerAttack / EnemyAI.ExecuteEnemyAttack → ResolveAttack → ApplyDamage → CheckVictory
  └── EndCombat
        ├── XPManagerRef.AwardCombatXP(CurrentMonsterGroupID) → TotalXP
        ├── XPManagerRef.DistributeXP(TotalXP) → (per living member) AddXPToCharacter + CheckLevelUp
        ├── PostMessage("Victory!"), clear combat state
        ├── CombatHUDRef.Remove from Parent
        └── ChangeGameState(Exploration)

WBP_EncounterScreen
  ├── Initialise() — sets refs from Game Instance, binds OnGameStateUpdated
  ├── SetEncounterData(IntroText, Choices) — sets text, calls PopulateChoices
  ├── PopulateChoices(Choices) — creates WBP_ChoiceButton per choice
  └── ClearChoices() — clears VerticalBox_Choices

WBP_ChoiceButton
  └── OnClicked → Get Game Instance → Get EncounterManager → ResolveChoice(ChoiceIndex)
      [EncounterManagerRef retrieved fresh from Game Instance at click time — not stored]

WBP_CombatHUD
  ├── Initialise() — sets GameManagerRef from Game Instance
  ├── Shown/hidden directly by BP_CombatManager.StartCombat/EndCombat (not via OnGameStateUpdated —
  │     binding does not fire reliably for widgets not yet in viewport)
  └── Button clicks → GameManagerRef.CombatManagerRef.(EnterMoveMode / EnterTargetSelectMode / EndPlayerTurn)
```

### Widget Initialisation Pattern

WBP_EncounterScreen does NOT use Event Construct because Event Construct only fires when a widget is added to viewport. Since the encounter screen adds itself conditionally, it uses an explicit `Initialise` function called from the Level Blueprint instead.

This is the correct pattern for any widget that:
- Is created at startup but not immediately added to viewport
- Needs manager refs set before it can function
- Adds/removes itself based on game state

### CanMove Pattern

`BP_ExplorerPawn.CanMove` is a class variable checked at the start of `CanInitiateMove`. Setting it false blocks all movement without touching individual movement functions. Set false on encounter start, set true on encounter resolution.

---

## Data Tables

| Table | Struct | Purpose | Status |
|---|---|---|---|
| DT_LevelProgression | SLevelProgression | THAC0, XP, hit dice by class/level | ✅ Populated to L5 |
| DT_SavingThrows | SSavingThrowRow | Save values by class/level | ✅ Populated to L5 |
| DT_Monsters | SMonster | 5 monsters seeded for VS | ✅ Complete (Goblin, Orc, Skeleton, Zombie, Giant Rat) |
| DT_TriggerEvents | STriggerEvent | Scripted dungeon triggers | ✅ 2 rows |
| DT_Encounters | S_Encounter | Scripted encounters | ✅ 1 row (Goblin patrol, 3 choices) |
| DT_MonsterPortraits | S_MonsterPortrait | Monster portrait textures | ✅ 1 stub row (no art yet) |
| DT_DungeonMaps | SDungeonMap | Dungeon level data | ✅ Empty (GenerateTestMap used) |
| DT_HPFractionTests | SRulesTestCase | Unit tests | ✅ 8 tests passing |
| DT_THAC0Tests | SThAC0TestCase | Unit tests | ✅ 14 tests passing |
| DT_SavingThrowTests | SSavingThrowTestCase | Unit tests | ✅ Rows ready |
| DT_StrengthBonusTests | SStrengthBonusTestCase | Unit tests | ✅ 14 rows ready |
| DT_AttackResolutionTests | SAttackTestCase | Unit tests | ✅ Rows ready |
| DT_MultiAttackTests | SMultiAttackTestCase | Unit tests | ✅ Rows ready |
| DT_XPThresholdTests | SXPThresholdTestCase | Unit tests | ✅ Rows ready |
| DT_LevelCapTests | SLevelCapTestCase | Unit tests | ✅ Rows ready |

---

## Test Framework (BP_RulesTestSuite / L_TestSuite)

- Separate test level `L_TestSuite` with `BP_GameManager` level name gate
- `BP_RulesTestSuite` actor with `AssertEqual_Float`, `AssertEqual_Integer`, `AssertEqual_Bool`
- `RunDataTableTests(DataTable, Category)` — generic data table driven test runner
- Total: **30 tests passing**
- `PrintSummary` — shows pass/fail count

---

## Enums

| Enum | Values | Notes |
|---|---|---|
| EGameState | MainMenu, Exploration, Encounter, Combat, Camp, CharacterScreen | |
| EDirection | North, East, South, West | |
| ECharacterClass | Warden, Skirmisher, Templar, Devout, Sylvan, Adept, Shadowpriest, Rogue, Infiltrator | Renamed GB-079 |
| ERace | Human, Elf, Dwarf, Gnome, HalfElf, Halfling, HalfOrc | |
| EAlignment | LawfulGood through ChaoticEvil (9 values) | |
| ECombatAction | Move, Attack, Cast, UseItem, Guard, Flee | |
| ECombatAction_AI | MoveToTarget, AttackNearest, AttackIncapacitated, CastSpell, Flee | |
| ECondition | Normal, Restrained, Paralysed, Poisoned, Blinded, Quickened, Slowed, Diseased, Sapped, Petrified, Dead, Unconscious | Renamed GB-079 |
| ESpellSchool | MagicUser, Illusionist, Cleric, Druid | |
| ESpellEffect | Damage, Heal, AoEDamage, Entangle, Hold, Haste, Slow, Blind, LevelDrain, Summon, Utility | |
| ETriggerType | OnEnterTile, OnSearchTile, OnStoryFlag, OnTimedCondition, OnCombatEnd, OnItemUsed, OnDayNight, OnMoonPhase | |
| ETriggerActionType | ShowMessage, StartEncounter, StartCombat, TransitionLevel, SetStoryFlag, SpawnLoot | |
| EMessageType | Exploration, Combat, System, Loot, Encounter | |
| EOutcomeType | Combat, Reward, Flee, TextOnly | |
| ETileType | Floor, Pit, Water, Teleport, StairsUp, StairsDown | |
| ESaveType | Fortitude, Reflex, Willpower | Restructured GB-079 (5 to 3 categories) |
| ETestType | Float, Integer, Bool | |

---

## Structs

| Struct | Purpose |
|---|---|
| SCharacter | Full character data (stats, HP, conditions, spells, inventory) |
| SMonster | Monster data (HD, attacks, special abilities) |
| SCombatant | Runtime combat state |
| SDungeonTile | Single tile (walls, passable, triggers, type) |
| SDungeonMap | Full dungeon level (tile grid, step counters) |
| STriggerEvent | Scripted event definition |
| S_Encounter | Scripted encounter (portrait, intro text, choices array) |
| S_EncounterChoice | Single encounter choice (label, outcome type, payload) |
| SLevelProgression | XP, THAC0, hit dice per class/level |
| SSavingThrowRow | Save values per class/level |
| SRulesTestCase | Generic float/int/bool test case |
| SThAC0TestCase | THAC0 test case |
| SAttackTestCase | Attack resolution test case |
| SSavingThrowTestCase | Saving throw test case |
| SStrengthBonusTestCase | Strength bonus test case |
| SMultiAttackTestCase | Multiple attacks test case |
| SXPThresholdTestCase | XP threshold test case |
| SLevelCapTestCase | Level cap test case |

---

## Next Steps

**Phase 4a-4f complete. VS loop fully functional end-to-end.**

Phase 4 critical path:
```
Phase 4a: GB-030 ✅, GB-031 ✅
Phase 4b: GB-022 ✅, GB-033 ✅, GB-034 ✅
Phase 4c: GB-036 ✅, GB-043 ✅, GB-042 ✅, GB-046 ✅ (partial)
Phase 4d: GB-045 ✅ (Enemy AI stub complete)
Phase 4e: GB-004a ✅ (Combat HUD stub), GB-041 ✅ (XP Manager)
Phase 4f: GB-039 ✅ (Dead + Restrained conditions — VS subset)
```

**New ticket needed:** live HP sync during combat (all hits, not just death)

  ### Phase 1-3 Refactor (Complete)

  All ForEach-loop-by-ID patterns in BP_CombatManager replaced with FindCombatantByID and FindMarkerByID:
  - 15 caller functions refactored to use FindCombatantByID(CombatantID) to OutCombatant + bFound
  - ExecutePlayerAttack rewritten from scratch: cleaner graph, ForEach loops removed, 240 damage display bug fixed
  - 3 marker functions (RemoveMarkerForCombatant, MoveMarkerToTile, SetMarkerDowned) refactored to use FindMarkerByID
  - Miss messages reconnected in ExecutePlayerAttack
  - FindStartCombatant missing CurrentCombatantID assignment fixed. `ApplyDamage` currently updates `SCombatant` only — `SCharacter` in `BP_PartyManager` doesn't reflect HP changes until combat ends. Party panel shows stale HP for living characters taking damage mid-combat. Consider adding as **GB-039a** or a new Phase 4 ticket before VS is considered done.

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
- **GB-079 already complete — only ESpellSchool 4-to-2 shrink deferred (magic system not built yet).** (Phase 6, before commercial release): replaces THAC0/AC math, GetStrengthBonus/GetConstitutionHPBonus tables, DT_LevelProgression and DT_SavingThrows tables, and renames ECharacterClass/ECondition/ESpellSchool values. See `Threshold_Ruleset_v1.md` §11 for the full mapping.

---

*Document updated: GB-079 ruleset migration complete. Phase 1-3 refactor complete (ForEach to FindByID on BP_CombatManager). Phase 4 combat loop verified end-to-end.*
*Next: Phase 5/6 tickets (GB-037/038/040/044) or live HP sync (GB-039a).*
*UE5.7 · Blueprints Only · Solo Dev*
