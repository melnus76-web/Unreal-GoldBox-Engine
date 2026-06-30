# Gold Box UE5.7 — Solo Dev Build Order
## Optimised for Earliest Playable Vertical Slice

---

## How to Read This Document

Each phase ends with something you can **actually run and test**. Never build a system you cannot immediately verify. For a solo dev, untestable work is invisible debt.

Tickets marked **[VS]** are on the critical path to the Vertical Slice.
Tickets marked **[STUB]** should be built thin first — just enough to unblock the next system — then fleshed out in a later phase.
Tickets marked **[DEFER]** are not needed until after Milestone 1.

**Ruleset notice:** GB-079 is now **essentially complete** — ability scores, the entire combat math, XP thresholds, saving throws, and the class/condition enum renames have all been migrated to **The Threshold System** (`Threshold_Ruleset_v1.md`) and tested. Only `ESpellSchool`'s 4→2 shrink remains, deliberately deferred since the magic system isn't built yet. See the GB-079 row in Phase 6 below for full detail.

---

## Phase 0 — Foundations (No Skipping)
*Everything else depends on this. Build it once, build it right.*
*Testable output: project opens, enums exist, GameManager is accessible from any Blueprint.*

| Order | Ticket | Notes |
|---|---|---|
| 1 | **GB-001** | Project structure, folders, naming conventions, source control commit |
| 2 | **GB-002** | All enums in one sitting — EGameState, EDirection, ECharacterClass, ERace, EAlignment, ECombatAction, ECombatAction_AI, ECondition, ESpellSchool, ESpellEffect, ETriggerType |
| 3 | **GB-017** | SCharacter struct — the single most-depended-on data structure in the project. Get every field right now, it is painful to change later |
| 4 | **GB-021** | SMonster struct — needed before any combat ticket |
| 5 | **GB-032** | SCombatant struct — needed before combat grid and turn manager |
| 6 | **GB-003** | BP_GameManager — stub: ChangeGameState only, story flags map, AwardXP placeholder |
| 7 | **GB-035** | BP_DiceSystem — pure functions, no dependencies, test immediately with Print String nodes |

**Why this order:** Enums must exist before structs reference them. Structs must exist before managers hold references to them. DiceSystem is pure and dependency-free — build it early and test it in isolation.

---

## Phase 1 — Hardcode a Test Party
*Testable output: BP_PartyManager holds four characters you can inspect in the debugger.*

| Order | Ticket | Notes |
|---|---|---|
| 8 | **GB-018** ⚠ partial | BP_CharacterRules — ComputeTHAC0, ComputeAC, GetStrengthBonus, ComputeSavingThrows, as originally built. Populate DT_LevelProgression and DT_SavingThrows data tables now — you will need them in combat. **GB-079 update: GetStrengthBonus/ComputeAC deleted and replaced. ComputeSavingThrows migrated to BPL_RulesLibrary (3-category formula: Fortitude/Reflex/Willpower) — see GB-079 row below for current status.** |
| 9 | **GB-020** | BP_PartyManager — AddCharacter, GetLivingMembers, IsPartyWiped. Do NOT build character creation yet |
| 10 | **GB-020a** | [STUB] Hardcode a test party of 4 in BP_GameManager BeginPlay: one Fighter, one Cleric, one Magic-User, one Thief. Fill all SCharacter fields manually. This replaces GB-019 for the VS |

**Why defer GB-019 (Character Creation)?** A full creation flow takes significant UI time. A hardcoded party unblocks every downstream system immediately. Creation UI comes in Phase 6.

---

## Phase 2 — Dungeon You Can Walk Through
*Testable output: Launch the game, walk through a hand-built dungeon, read a tile message.*

| Order | Ticket | Notes |
|---|---|---|
| 11 | **GB-007** | SDungeonTile and SDungeonMap structs. Also create DT_DungeonMaps and hand-author one small test level (8x8, a few corridors, one message tile, one exit tile, one encounter trigger tile) |
| 12 | **GB-008** | BP_MapManager — LoadMap, GetTile, CheckWall, UpdatePlayerPosition, GetExitTile. Leave IncrementStepCounter as a stub (just increments, no encounter check yet) |
| 13 | **GB-005** | BP_ExplorerPawn — camera, grid coords, facing direction |
| 14 | **GB-006** | Movement system — MoveForward, MoveBackward, TurnLeft, TurnRight with timeline-based smooth movement and wall collision check |
| 15 | **GB-010** | Placeholder dungeon meshes — one wall mesh, one floor mesh is enough to see where you are. Full mesh variants come later |
| 16 | **GB-011** | BP_DungeonGenerator — SpawnGeometryFromMap. Keep it simple: spawn wall and floor meshes per tile, no culling yet |
| 17 | **GB-012** | WBP_ExplorationHUD — layout only: four regions (viewport, party panel, text log, command menu). Panels can be empty boxes for now |
| 18 | **GB-013** | WBP_PartyPanel — show name and HP for each party member. Pulls from BP_PartyManager |
| 19 | **GB-014** | WBP_TextLog + BP_TextManager — PostMessage and display. Wire up tile message display from map data |
| 20 | **GB-015** | WBP_CommandMenu — Move commands only for now. Exploration state only |

**Test checkpoint:** You can walk a corridor, bump into walls, read a message when you step on a trigger tile, and see your party HP panel. Everything visible is functional.

---

## Phase 3 — Scripted Encounter
*Testable output: Walk to a trigger tile, encounter screen appears, make a choice, choose Attack.*

| Order | Ticket | Notes |
|---|---|---|
| 21 | **GB-025** | ETriggerType framework and STriggerEvent struct. Create DT_TriggerEvents with two entries for your test level: one message trigger, one encounter trigger |
| 22 | **GB-026** | BP_EventManager — EvaluateTriggers called on every tile move. Wire to BP_TextManager for messages and to BP_EncounterManager for encounter trigger |
| 23 | **GB-027** | DT_Encounters — create one test encounter: a Goblin patrol. One portrait, intro text, three choices: Nice (flee outcome), Sly (flee outcome), Attack (combat outcome) |
| 24 | **GB-028** | BP_EncounterManager — StartEncounter, ResolveChoice, TriggerCombatFromEncounter |
| 25 | **GB-029** | WBP_EncounterScreen — portrait area, text area, dynamic choice buttons. Wire to BP_EncounterManager |
| 26 | **GB-016** | WBP_EncounterPortrait — [STUB] just show a placeholder texture for now. Real portraits come with art pass |

**Test checkpoint:** Walk to encounter tile -> screen transitions -> read text -> click Attack -> encounter manager calls StartCombat (which doesn't exist yet, so just print "Combat would start here").

---

## Phase 4 — Combat You Can Win
*Testable output: Full combat loop — initiative, player attacks, enemy attacks, someone dies, XP awarded, return to dungeon.*
*This is the longest phase. Do not attempt to do it in one session.*

### Phase 4a — Combat Grid and Camera

| Order | Ticket | Notes |
|---|---|---|
| 27 | **GB-030** ✅ | BP_CombatGrid — ISM-based, integer-keyed OccupancyMap + BlockedTiles Set, Chebyshev throughout. **Built:** TileToIndex, IndexToTile, TileToWorld, IsTileInBounds, IsBlocked, SetBlocked, IsTraversable, IsOccupied, GetOccupant, SetOccupant, ClearOccupant, GetAdjacentTiles, GetTilesInRange, GetTilesInSpellAoE (Radius only), SpawnGrid, RebuildTileInstances, GenerateGridFromDungeon. All tests passing. 256 instances (full grid), 40 instances (corridor at player 0,3) confirmed. |
| 28 | **GB-031** ✅ | BP_CombatCamera — transition from first-person to top-down orthographic on combat start. This is a significant UE camera blend; test it in isolation before wiring to combat |

### Phase 4b — Combat Data and Turn Flow

| Order | Ticket | Notes |
|---|---|---|
| 29 | **GB-022** ✅ | DT_Monsters — 5 monsters seeded: Goblin (ID=1), Orc (ID=2), Skeleton (ID=3, IsUndead), Zombie (ID=4, IsUndead), Giant Rat (ID=5, HasPoison, HitDice=0.5 for sweep attack testing). SMonster struct fixed: MoraleRating + XPValue changed from Boolean to Integer |
| 30 | **GB-033** ✅ | BP_CombatManager — StartCombat wired end to end. Spawned by BP_GameManager. CombatGridRef + CombatCameraRef set from Level Blueprint. Party CombatantIDs start at 100. CompleteMovement in BP_ExplorerPawn now updates GameManager CurrentPlayerTileX/Y. C key test trigger removed — StartCombat owns camera switch. Full flow verified: encounter -> Attack! -> combat camera -> corridor arena -> movement locked |
| 31 | **GB-034** ✅ | BP_CombatTurnManager — StartPlayerTurn, StartEnemyTurn, OnActionComplete, round counter |

### Phase 4c — Attack Resolution

| Order | Ticket | Notes |
|---|---|---|
| 32 | **GB-036** ✅ -> ✅ migrated under GB-079 | ResolveAttack added to BPL_RulesLibrary, originally built with THAC0-AC d20 math. **Fully rebuilt during GB-079 as a percentage-based roll — see GB-079 row below for details.** |
| 33 | **GB-043** ✅ | WBP_AttackAction — target selection highlight, call ResolveAttack, apply damage, death check, log result to TextLog |
| 34 | **GB-042** ✅ | Move Action (VS scope — on BP_CombatManager not WBP): M key -> EnterMoveMode -> WASD moves combatant one tile. BP_TileHighlight actor for cyan tile highlights. GetValidMoveTiles filters traversable+unoccupied. TilesMovedThisTurn tracking (resets each turn). EndPlayerTurn (T key). RefreshMoveHighlights after each move. Return Node fix in OnActionComplete resolved duplicate turn message. MovementRange=3 for VS (rules redesign post-VS). |

### Phase 4d — Enemy AI (Minimum Viable)

| Order | Ticket | Notes |
|---|---|---|
| 35 | **GB-045** ✅ | BP_EnemyAI — VS stub complete. Spawned by BP_GameManager, CombatManagerRef set by StartEnemyTurn. FindNearestPartyMember (Chebyshev distance, skips monsters + dead), MoveOneStepToward (GetAdjacentTiles -> IsTraversable + IsOccupied filter -> closest tile), ExecuteEnemyAttack (ResolveAttack -> ApplyDamage -> CheckVictory). Shared functions added to BP_CombatManager: UpdateCombatantGridPosition, MoveMarkerToTile, ApplyDamage. Verified: goblin moves one tile per turn, respects traversability and occupancy, attacks nearest party member, combat continues and ends correctly. Full priority logic (incapacitated targets, AoE opportunity, morale flee) comes post-VS. |

### Phase 4e — Combat HUD, XP, and End State

| Order | Ticket | Notes |
|---|---|---|
| 36 | **GB-004a** ✅ | WBP_CombatHUD — VS stub complete. Bottom action bar matching ExplorationHUD command menu position (full width, Y=-108, H=108). Buttons: Move (M), Attack (A), End Turn (T), Flee (stub). Round counter and turn indicator text placeholders (wiring deferred). Shown/hidden directly by BP_CombatManager.StartCombat/EndCombat via CombatHUDRef. Button clicks wire to BP_CombatManager functions. Verified: HUD appears on combat start, all buttons functional, HUD disappears on combat end. Note: OnGameStateUpdated binding pattern does NOT work for widgets not yet in viewport — use direct manager ref pattern instead (same as WBP_EncounterScreen). |
| 37 | **GB-041** ✅ -> ✅ fully migrated under GB-079 | BP_XPManager — AwardCombatXP, DistributeXP, CheckLevelUp, ApplyLevelUp, as originally built. **GB-079 update: Constitution/GetConstitutionHPBonus replaced by Vigor/GetVigorBonus, THAC0 references replaced by StrikeNumber throughout, XPRequired table lookup replaced by GetXPThreshold formula.** Original verification: goblin defeat awards 20 XP split 4 ways; forced high-XP test confirmed correct level-up for all four characters. New formula verified separately: 2500 XP per character correctly leveled all four to exactly level 2, not level 3. |
| 38 | **GB-046** ✅ (partial) | Combat victory/defeat handling — XP summary display, loot placeholder (just gold for now), return to exploration state. **Built for VS:** CheckVictory + EndCombat (victory detection, XP award via GB-041, camera/state cleanup, return to exploration). **Deferred:** defeat handling (all party dead), loot drop, retreat system |

### Phase 4f — Minimum Conditions

| Order | Ticket | Notes |
|---|---|---|
| 39 | **GB-039** ✅ | Condition system — 8/12 complete. **Built:** Dead, Restrained, Poisoned, Blinded, Quickened, Slowed, Paralysed, Unconscious. Skip-turn OR-chain in `OnActionComplete` covers Dead/Restrained/Paralysed/Unconscious. Auto-hit OR-chain in both attack functions covers Restrained/Paralysed/Unconscious. Blinded penalty wired into `ResolveAttack` (attacker +4 SN, defender +4 DR) and `ResolveSave` (+4 Reflex). `GetMovementModifier` on `BPL_RulesLibrary` handles Quickened (+1)/Slowed (−1) for both movement (`EnterMoveMode`) and attacks (`GetAttacksThisRound`, with min-1 clamp for Slowed). `GetBlindPenalty` on `BPL_RulesLibrary` returns 4 if Blinded. **Deferred:** Diseased (game-time system needed), Sapped (level-down logic needed), Petrified (damage-type system needed). Live HP sync during combat, full defeat/game-over (GB-046), multi-condition display, and downed marker polish all still deferred. (renamed from Held under GB-079). **Built:** `ApplyCondition`, `RemoveCondition`, `HasCondition`, `SetMarkerDowned` as shared functions on `BP_CombatManager`. Dead applied on kill in both `ExecutePlayerAttack` and `ExecuteEnemyAttack`. `OnActionComplete` guard clause skips Dead and Restrained combatants (with critical ID-match gate fix — without it the For Each loop was dispatching start-turn logic for every combatant instead of just the matching one, causing rapid infinite recursion). `CheckDefeat` stub added to `OnActionComplete` (calls `EndCombat` when all party members have Dead — proper game-over handling deferred to GB-046). Restrained auto-hit wired into both attack paths with separate "(Auto Hit)" message path bypassing the roll-display message. `CharacterID` added to `SCombatant` struct; `SCharacter.Conditions` changed from single `ECondition` to `Array of ECondition`; `ApplyDeathToCharacter` built on `BP_PartyManager` (sets HP=0, adds Dead, fires `OnPartyUpdated`); erroneous `RefreshPartyPanel` call removed from `OnMessagePosted_Event` in `WBP_ExplorationHUD` (was causing infinite loop on all-party-dead). **Deferred:** live HP sync during combat (all hits, not just death — new ticket needed), full defeat/game-over screen (GB-046), multi-condition display in party panel, visual polish for downed markers. |

**Test checkpoint:** Full VS loop works end to end. Walk dungeon -> encounter trigger -> choose Attack -> combat starts -> move and attack -> enemy attacks back -> one side dies -> XP awarded -> return to dungeon -> reach exit tile.

### Phase 4g — Blueprint Refactor (ForEach to FindByID) ✅

**All ForEach-loop-by-ID patterns in BP_CombatManager replaced with FindCombatantByID and FindMarkerByID.**

15 caller functions refactored: ExecutePlayerAttack (rewritten from scratch), ApplyDamage, ApplyCondition, RemoveCondition, HasCondition, UpdateCombatantGridPosition, ExecuteCombatantMove, EnterMoveMode, EndPlayerTurn, CheckVictory, CheckDefeat, FindStartCombatant, FindActiveCombatant, OnActionComplete, StartPlayerTurn, StartEnemyTurn.

3 marker functions refactored to use FindMarkerByID: RemoveMarkerForCombatant, MoveMarkerToTile, SetMarkerDowned.

Bugs fixed: miss messages reconnected, 240 damage display bug, FindStartCombatant missing CurrentCombatantID.

---

## Phase 5 — Vertical Slice Complete
*Run the full VS definition checklist from the roadmap:*

- [ ] Start game
- [ ] Load a dungeon level (hardcoded test party)
- [ ] Move through dungeon tile-by-tile
- [ ] Turn 90 left/right
- [ ] Read tile-triggered dungeon messages
- [ ] Trigger a scripted encounter (dialogue choice)
- [ ] Enter tactical combat on a grid
- [ ] Execute attack actions, resolve percentage-based hit chance, apply damage *(Threshold System — migrated under GB-079)*
- [ ] Defeat all enemies
- [ ] Award XP, check level-up
- [ ] Return to exploration
- [ ] Reach dungeon exit

**Before moving to Phase 6, do a complete codebase review.** Rename anything inconsistently named. Document every Blueprint with a description. Commit to source control with a VS tag. The post-VS systems build on everything here — clean foundations now save serious pain later.

---

## Phase 6 — Rules Engine Completion
*Replace the legacy placeholder math with The Threshold System, then fill in everything that was stubbed during Phase 4. Combat becomes tactically deep.*

| Order | Ticket | Notes |
|---|---|---|
| 40a | **GB-079** ✅ essentially complete | Migrate legacy rules to The Threshold System. See Threshold_Ruleset_v1.md 11 for the full status table. **Done:** ability scores, GetMightBonus/GetVigorBonus/GetAbilityModifier, AC->DefenseRating and THAC0->StrikeNumber renamed everywhere, ResolveAttack rebuilt as a percentage-based roll (see Threshold_Ruleset_v1.md 4 revision note), all 5 DT_Monsters rows converted, GetXPThreshold formula replacing DT_LevelProgression's XPRequired lookup, ComputeSavingThrows rebuilt with the 3-category formula (also relocated from BP_CharacterRules — found there unexpectedly, same as GetStrengthBonus before it; a full audit of BP_CharacterRules's remaining contents is worth doing at some point), ECondition renamed (Restrained/Quickened/Sapped), and ECharacterClass fully renamed (Warden/Devout/Adept/Rogue + 5 hybrid slots) including the knock-on rename of all 20 DT_LevelProgression row names. **Deliberately deferred:** ESpellSchool's 4->2 shrink — not urgent since magic isn't built. **Test suite note:** DT_THAC0Tests/DT_StrengthBonusTests/DT_AttackResolutionTests/DT_XPThresholdTests/DT_SavingThrowTests/DT_LevelCapTests are all stale against the new math — decided to delete and recreate these fresh at a later date rather than patch them individually. |
| 40 | **GB-037** ✅ (player) ⚠ (monsters pending) | Multiple attacks — Threshold System breakpoints (Warden 1->2->2+reroll at L5/L8, Rogue/Devout 1->2 at L8, Adept always 1). No fractional/partial-round tracking, no Sweep Attack subsystem (dropped). **Done:** `GetAttacksThisRound` built on `BPL_RulesLibrary`, `RerollOnesOnDamage` wired into `ExecutePlayerAttack` per-hit `ApplyDamage` chain, `Completed` pin routed to `CheckVictory`. **TODO:** mirror multi-attack into `BP_EnemyAI.ExecuteEnemyAttack` — add `GetAttacksThisRound` call, For Loop wrapping `ResolveAttack`, per-hit `ApplyDamage` + reroll chain (same pattern as `ExecutePlayerAttack`). Current monsters all have `NumAttacks=1` (harmless) but must be done before any multi-attack monsters are added. |
| 41 | **GB-038** ✅ (function ready, consumers pending) | Saving throw system — `ComputeSavingThrows` already built and migrated under GB-079. **Done:** `E_SaveType` enum created (Fortitude/Reflex/Willpower), `ResolveSave(Character, SaveType, DifficultyModifier)` built on `BPL_RulesLibrary` — Break SCharacter -> ComputeSavingThrows -> Switch(SaveType) -> SET ThisSaveTarget -> +DifficultyModifier -> DiceRoll(1d20) -> return bSuccess, RollResult, SaveTarget. **Pending:** wire into actual gameplay triggers — GB-039 (poison ticks call Fortitude), GB-044 (Ambush Strike may call Reflex), GB-046 (retreat calls Reflex). |
| 42 | **GB-039** ✅ (8/12 complete) | Full condition system. **Done:** Poisoned (Fortitude save or 1d4 damage, wired in `TickConditions` → `OnActionComplete`), Blinded (−4 SN/DR via `ResolveAttack`, −4 Reflex via `ResolveSave`, `GetBlindPenalty` helper), Quickened (+1 attack via `GetAttacksThisRound`, +1 movement via `EnterMoveMode`/`GetMovementModifier`), Slowed (−1 attack min 1 via `GetAttacksThisRound`, −1 movement via same `GetMovementModifier`), Paralysed/Unconscious (skip-turn via `OnActionComplete` OR-chain, auto-hit via both attack functions). `GetAttacksThisRound` refactored — all class paths converge through variable sets, then Quickened/Slowed modifiers applied. **Deferred:** Diseased (game-time system needed), Sapped (level-down logic needed), Petrified (damage-type system needed). |
| 43 | **GB-040** | Morale system — 2D6 + leader's Presence modifier vs fixed thresholds, flee behaviour, trigger conditions |
| 44 | **GB-044** ✅ | Ambush Strike (renamed from Backstab) — Rogue condition check, Threshold System multiplier table (L1-3 x2, L4-6 x3, L7-9 x4, L10 x5), wired to player and enemy attack flows. `ResolveAttack` extended with `SituationalModifier` (+4 for Ambush). `IsFlanked` and `GetAmbushMultiplier` built on `BPL_RulesLibrary`. Hybrid classes deferred |
| 45 | **GB-045** | Enemy AI — full logic: incapacitated target priority, intelligent monster spellcasting, AoE opportunity check, morale-based flee |
| 46 | **GB-046** | Combat retreat — Reflex check, partial flee, full party retreat handling |

---

## Phase 7 — Character Creation and Full Party System

| Order | Ticket | Notes |
|---|---|---|
| 47 | **GB-019** | Full character creation flow — all 7 steps, race/class filter, portrait selection (no exceptional-ability-score step — dropped in The Threshold System) |
| 48 | **GB-018a** | Race/class restrictions and level caps — wire into creation flow and XP system |
| 49 | **GB-052** | WBP_CharacterScreen — full character sheet view |

---

## Phase 8 — Magic System

| Order | Ticket | Notes |
|---|---|---|
| 50 | **GB-047** | DT_Spells — Arcane (Adept) and Divine (Devout) lists, original spell names per Threshold_Ruleset_v1.md 9 (Force Dart, Slumber, Cinder Burst, Mend Wounds, Blessing, etc.) |
| 51 | **GB-048** | Spell slot tracking per character — formula-driven (`clamp(CasterLevel - 2x(SpellLevel-1), 0, 4)`), no data table; arrays sized 1-5 not 1-7 |
| 52 | **GB-049** | BP_SpellCaster — CanCast, CastSpell, slot deduction, dispatch to effect handler |
| 53 | **GB-050** | BP_SpellEffectHandler — damage, Restrained, Mend Wounds, AoE with friendly fire, Rekindle (-1 Vigor), Renewal |
| 54 | **GB-051** | WBP_SpellAoEPreview — grid overlay before cast confirmation |

---

## Phase 9 — Camp, Save, and Inventory

| Order | Ticket | Notes |
|---|---|---|
| 55 | **GB-057** | BP_GoldBoxSave — serialise full party, dungeon position, story flags, gold |
| 56 | **GB-058** | LoadFromSlot |
| 57 | **GB-059** | AutoSave — trigger on level transition and post-combat |
| 58 | **GB-060** | WBP_CampScreen |
| 59 | **GB-061** | Rest and spell memorisation — 4hr + 15min/level timing, slot restore, random encounter roll |
| 60 | **GB-062** | Party reordering |
| 61 | **GB-063** | Random rest encounter |
| 62 | **GB-053** | SItem struct + DT_Items |
| 63 | **GB-054** | BP_EquipmentManager — equip/unequip, Defense Rating recompute, alignment check |
| 64 | **GB-055** | BP_LootManager — loot generation, loot screen, distribution |

---

## Phase 10 — Random Encounters and Event System Completion

| Order | Ticket | Notes |
|---|---|---|
| 65 | **GB-023** | BP_StepCounterManager — full multi-counter logic, weighted random trigger |
| 66 | **GB-024** | DT_RandomEncounters — populate for test dungeon |
| 67 | **GB-026a** | Extend BP_EventManager to handle full ETriggerType range (OnSearchTile, OnStoryFlag, OnDayNight, OnMoonPhase) |
| 68 | **GB-064** | Alignment tracking — wire to item equip and encounter outcomes |
| 69 | **GB-065** | Story flag system — already stubbed in GB-003, now wire to triggers and encounters fully |

---

## Phase 11 — Dungeon Level Transitions and Full Exploration

| Order | Ticket | Notes |
|---|---|---|
| 70 | **GB-009** | Dungeon level transitions — StairsUp/Down tile handling, LoadDungeonLevel, level index tracking |
| 71 | **GB-010a** | Full modular dungeon mesh set — crumbling walls, tapestried walls, iron bars, multiple door types |
| 72 | **GB-011a** | Geometry culling — only render tiles within view distance |

---

## Phase 12 — Overland Map and World Structure

| Order | Ticket | Notes |
|---|---|---|
| 73 | **GB-066** | WBP_OverlandMap + DT_OverlandLocations |
| 74 | **GB-067** | BP_OverlandManager — movement between locations, step counter |
| 75 | **GB-068** | Overland random encounters |
| 76 | **GB-070** | Weather system — EWeather enum, BP_WeatherManager, encounter rate modifier |

---

## Phase 13 — Extended Systems (Post Alpha)

| Order | Ticket | Notes |
|---|---|---|
| 77 | **GB-056** | Shop / merchant system |
| 78 | **GB-069** | NPC party members |
| 79 | **GB-071** | Character import / export |
| 80 | **GB-072** | BP_AudioManager |
| 81 | **GB-073** | DT_MusicTracks + DT_SFX |

---

## Phase 14 — Content Pipeline (Milestone 3)

| Order | Ticket | Notes |
|---|---|---|
| 82 | **GB-074** | Map editor tools |
| 83 | **GB-075** | Encounter authoring |
| 84 | **GB-076** | Dialogue authoring |
| 85 | **GB-077** | Dungeon themes |
| 86 | **GB-078** | Monster group templates |
| 87 | **GB-022** | Expand DT_Monsters to full 127 rows |

---

## Summary: Vertical Slice Critical Path

These 39 tickets are all you need to hit Milestone 1. Everything else is post-VS.

```
Phase 0:  GB-001, GB-002, GB-017, GB-021, GB-032, GB-003, GB-035
Phase 1:  GB-018, GB-020, GB-020a (hardcoded party)
Phase 2:  GB-007, GB-008, GB-005, GB-006, GB-010, GB-011,
          GB-012, GB-013, GB-014, GB-015
Phase 3:  GB-025, GB-026, GB-027, GB-028, GB-029, GB-016
Phase 4a: GB-030 ✅, GB-031 ✅
Phase 4b: GB-022 ✅ (5 monsters), GB-033 ✅, GB-034 ✅
Phase 4c: GB-036 ✅, GB-043 ✅, GB-042 ✅
Phase 4d: GB-045 ✅ (stub complete)
Phase 4e: GB-004a ✅ (stub), GB-041 ✅, GB-046 ✅ (partial)
Phase 4h: GB-039 ✅ complete. Multiple-monster spawning built. Initiative fixed. GB-040 (Morale) next.
```

---

## Key Principles for Solo Dev

**Build thin, test immediately.** Every phase ends with something you can run. If you cannot test it, you are not done building it.

**Stub aggressively.** A stub that logs "CastSpell called" to the screen is better than a half-built spell system that breaks combat. Mark every stub with a `// TODO` comment and a ticket reference.

**Data tables before logic.** DT_LevelProgression, DT_SavingThrows, and DT_Monsters contain the numbers the rules engine consults. Populate them before writing the Blueprint logic that reads them — you need real data to verify your formulas. *(Post-GB-079, DT_LevelProgression and DT_SavingThrows go away entirely in favour of formulas — DT_Monsters stays, since per-monster stats genuinely need authored data.)*

**SCharacter is load-bearing.** It is referenced by Party Manager, Combat Manager, Rules Engine, Save System, and UI simultaneously. If you need to add a field to it after Phase 0, every system that copies or passes the struct may need updating. Think carefully before you finalise it.

**Never build two untested systems at once.** If combat is broken and the encounter system is also new, you cannot isolate the bug. Finish and verify one system before starting the next.

**Commit after every phase.** Tag VS completion in source control. You need a known-good state to roll back to when post-VS work breaks something.
