# Gold Box Inspired RPG Engine
## Unreal Engine 5.6.4 Blueprint Development Plan

---

# Project Goal

Create a modern Gold Box-inspired RPG using:

- Unreal Engine 5.6.4
- Blueprints only
- First-person dungeon exploration (tile-based, discrete movement)
- Tactical top-down grid combat
- Six-character party management
- An original tactical ruleset — **The Threshold System** — covering combat resolution, saving throws, and Vancian-style spell memorisation
- Dialogue and scripted encounter system

---

# Ruleset Notice

This project originally prototyped its rules using AD&D-derived placeholder math (THAC0, descending AC, five-category saves, doubling XP tables) to get the vertical slice working quickly. That placeholder math has since been replaced in design by **The Threshold System** — a wholly original ruleset (see `Threshold_Ruleset_v1.md`) designed specifically to avoid reproducing any published rulebook's tables or terminology, since the long-term goal is a sellable commercial product.

**Migration status:** The Vertical Slice (Milestone 1) was built and tested end-to-end using the original placeholder math — that work is correct and complete, just running on rules that need replacing before any commercial release. All **new** rules work from Phase 6 onward targets The Threshold System directly. The already-built legacy systems (attack resolution, ability score bonus tables, XP/level progression) are tracked for migration under **GB-079** (see EPIC 009).

Tickets below are marked accordingly: ✅ = built (legacy math, working), no mark = not yet built (descriptions already updated to target The Threshold System).

---

# Vertical Slice Definition

A successful vertical slice allows the player to:

1. Start game
2. Create or load a test party of four characters
3. Load a dungeon level
4. Move through dungeon tile-by-tile
5. Turn 90° left/right
6. Read tile-triggered dungeon messages
7. Trigger a scripted encounter (dialogue choice)
8. Enter tactical combat on a grid
9. Execute attack actions, resolve Strike Number vs Defense Rating (built under legacy THAC0/AC math — see Ruleset Notice), apply damage
10. Defeat all enemies
11. Award XP, return to exploration
12. Reach dungeon exit

Everything else comes later.

---

# EPIC 001 — Project Foundation

## GB-001 Create Unreal project structure
- Create UE 5.6.4 Blueprint-only project
- Create folder structure: Blueprints, Data, UI, Maps, Audio, Art
- Configure source control (Git or Perforce)
- Create naming conventions document (BP_, DT_, WBP_, E_, S_ prefixes)

## GB-002 Create core enums
- EGameState (MainMenu, Exploration, Encounter, Combat, Camp, CharacterScreen)
- EDirection (North, East, South, West)
- ECharacterClass (Warden, Skirmisher, Templar, Devout, Sylvan, Adept, Shadowpriest, Rogue, Infiltrator) — ✅ **renamed under GB-079 (from Fighter/Ranger/Paladin/Cleric/Druid/MagicUser/Illusionist/Thief/Assassin). Mechanics not yet built — only the names exist**
- ERace (Human, Elf, Dwarf, Gnome, HalfElf, Halfling, HalfOrc)
- EAlignment (LawfulGood, NeutralGood, ChaoticGood, LawfulNeutral, TrueNeutral, ChaoticNeutral, LawfulEvil, NeutralEvil, ChaoticEvil)
- ECombatAction (Move, Attack, Cast, UseItem, Guard, Flee)
- ECombatAction_AI (MoveToTarget, AttackNearest, AttackIncapacitated, CastSpell, Flee)
- ECondition (Normal, Restrained, Paralysed, Poisoned, Blinded, Quickened, Slowed, Diseased, Sapped, Petrified, Dead, Unconscious) — ✅ **renamed under GB-079 (from Held/Hasted/LevelDrained). Tick/effect logic not yet built — VS scope only wires up Dead + Restrained**
- ESpellSchool (MagicUser, Illusionist, Cleric, Druid) — ⚠ **built as listed; Threshold System target is 2 schools, Arcane/Divine, see GB-079**
- ESpellEffect (Damage, Heal, AoEDamage, Entangle, Hold, Haste, Slow, Blind, LevelDrain, Summon, Utility)
- ETriggerType (OnEnterTile, OnSearchTile, OnStoryFlag, OnTimedCondition, OnCombatEnd, OnItemUsed)

## GB-003 Create Game Manager
- BP_GameManager (GameState singleton)
- ChangeGameState(EGameState)
- LoadDungeonLevel(LevelID, EntryTile)
- StartCombat(CombatSetup)
- EndCombat(Victory bool)
- SetStoryFlag(FlagName, Value)
- CheckStoryFlag(FlagName) → bool
- AwardXP(Amount) — distributes to living party members

## GB-004 Create UI Manager
- WBP_MainMenu
- WBP_ExplorationHUD
- WBP_CombatHUD
- WBP_EncounterScreen
- WBP_CampScreen
- WBP_CharacterScreen
- ShowScreen / HideScreen routing functions

---

# EPIC 002 — Dungeon Exploration Foundation

## GB-005 Create Grid Explorer Pawn
- BP_ExplorerPawn
- First-person camera (fixed height, no free-look)
- Current grid coordinates (X, Y, Level)
- Current facing direction (EDirection)
- Movement locked during transitions

## GB-006 Create movement system
- MoveForward — advance one tile in facing direction
- MoveBackward — retreat one tile
- TurnLeft — rotate facing 90° CCW
- TurnRight — rotate facing 90° CW
- Timeline-based smooth movement (not teleport)
- Wall collision check before moving (consult Map Manager)
- Step counter increment on each tile moved (feeds random encounter system)

## GB-007 Create dungeon map structures
- SDungeonTile struct:
  - WallNorth, WallEast, WallSouth, WallWest (bool)
  - DoorNorth, DoorEast, DoorSouth, DoorWest (bool)
  - Passable (bool)
  - TileType (Floor, Pit, Water, Teleport, StairsUp, StairsDown)
  - EventTriggerID (int, 0 = none)
  - MessageID (int, 0 = none)
  - SpecialFlags (bitmask: Secret Door, Trap, Dark, etc.)
- SDungeonMap struct:
  - LevelID (int)
  - Width, Height (int)
  - TileGrid (Array of SDungeonTile)
  - StepCounterThresholds (Array: encounter type → step threshold)
  - RandomEncounterTableID (int)
  - AmbientLighting (enum)

## GB-008 Create Map Manager
- BP_MapManager
- LoadMap(LevelID) — reads DT_DungeonMaps data table
- GetTile(X, Y) → SDungeonTile
- CheckWall(X, Y, EDirection) → bool
- CheckDoor(X, Y, EDirection) → bool
- UpdatePlayerPosition(X, Y)
- GetExitTile → Vector2D
- IncrementStepCounter(EncounterType) — triggers random encounter check
- HandleLevelTransition(StairsUp/Down) — load adjacent dungeon level

## GB-009 Create dungeon level transitions
- StairsUp / StairsDown tile types in SDungeonTile
- OnStairsEntered → prompt player, call LoadDungeonLevel
- Track current level index (supports up to 36 levels per adventure)

## GB-010 Create modular dungeon meshes
- SM_Wall_Stone, SM_Wall_Crumbling, SM_Wall_Tapestried, SM_Wall_IronBars
- SM_Floor_Stone
- SM_Door_Wood, SM_Door_Iron
- SM_Pillar
- SM_Ceiling
- BP_DungeonWallSpawner — selects mesh variant from tile data

## GB-011 Create dungeon geometry generator
- BP_DungeonGenerator
- SpawnGeometryFromMap(SDungeonMap) — instantiates meshes per tile
- CullNonVisibleGeometry — only render tiles within view distance
- UpdateVisibleGeometry on player move

---

# EPIC 003 — Exploration User Interface

## GB-012 Create Exploration HUD
- WBP_ExplorationHUD layout matching original Gold Box screen regions:
  - Top Left: Dungeon Viewport (first-person 3D view)
  - Top Right: Party Status Panel
  - Bottom: Text Log
  - Bottom: Command Menu

## GB-013 Create Party Status Panel
- WBP_PartyPanel
- Six character slots (name, current HP / max HP, condition icon)
- Colour coding: healthy (white), wounded (yellow), critical (red), dead (grey)
- Condition display: Paralysed, Poisoned, Hasted, etc.
- Updates on any HP or condition change

## GB-014 Create Text Log
- WBP_TextLog
- Scrolling message display (last N messages visible)
- Message types: Exploration, Encounter, Combat, System
- BP_TextManager — central message bus:
  - PostMessage(Text, EMessageType)
  - ClearLog

## GB-015 Create Command Menu
- WBP_CommandMenu
- Context-sensitive: changes options based on EGameState
- Exploration commands: Move (WASD), Search, Encamp, View Party, Items, Cast, Open Door
- Keyboard-primary input; optional mouse support

## GB-016 Create Encounter Portrait System
- WBP_EncounterPortrait — full-panel portrait display in viewport area
- DT_MonsterPortraits — maps MonsterID to portrait texture
- ShowPortrait(MonsterID) before combat or scripted encounter
- HidePortrait on combat start or choice resolution

---

# EPIC 004 — Party & Character System

## GB-017 Create Character Struct (SCharacter)
- Identity: Name (string), Race (ERace), Class (ECharacterClass), Class2 (for multi-class), Alignment (EAlignment), Gender
- Ability scores: Strength (int), StrengthExceptional (int, 0–100 for 18/xx), Intelligence, Wisdom, Dexterity, Constitution, Charisma — ⚠ **built as listed (legacy AD&D-style names); Threshold System target is Might/Vigor/Reflex/Acuity/Resolve/Presence, no StrengthExceptional field, see GB-079**
- Derived stats: CurrentHP, MaxHP, AC, THAC0 (computed), XP, Level, Level2 (multi-class) — ⚠ **AC/THAC0 field names can stay internally; see GB-079 for the value/formula migration**
- Condition: ECondition (current status)
- Saving throws: SaveVsPoison, SaveVsWands, SaveVsPetrification, SaveVsBreath, SaveVsSpells (all int, by class/level table) — ⚠ **Threshold System target is 3 saves (Fortitude/Reflex/Willpower), formula-driven not table-driven, see GB-079**
- Spell slots: SpellSlotsLevel1–7 (current/max pairs, by spell school)
- Memorised spells: Array of SpellID per slot level
- Equipment slots: Weapon, Shield, Armour, Helmet, Ring1, Ring2, Cloak, Misc
- Inventory: Array of SItem (up to 8 carried)
- Thief skills (if class = Thief/Assassin): PickPockets, OpenLocks, FindTraps, RemoveTraps, MoveSilently, HideInShadows, HearNoise, ClimbWalls, ReadLanguages (all int %) — ⚠ **Threshold System target is 5 consolidated Rogue skills (Stealth, Sleight of Hand, Disable Device, Climb, Perception), see GB-079**
- Portrait texture reference
- IsPlayerControlled (bool) — false for NPCs/monsters sharing this struct

## GB-018 Create Character Rules Library — *(target: The Threshold System)*
- BPL_RulesLibrary (pure function library)
- ComputeStrikeNumber(Class, Level) → int — formula: `16 − floor(Level / ClassSNDivisor)`, no data table needed
- ComputeDefenseRating(ArmorValue, ShieldValue, ReflexModifier) → int — ascending, see Threshold_Ruleset_v1.md §4
- ComputeSavingThrows(Class, Level, Might/Vigor/Reflex/Acuity/Resolve/Presence) → FSavingThrows struct (Fortitude, Reflex, Willpower) — formula: `16 − floor(Level/2) − AbilityMod − ClassBonus`
- GetMightBonus(Might) → FMightBonus (ToHit, Damage) — see Threshold_Ruleset_v1.md §1, no exceptional-score subsystem
- ComputeMaxHP(Class, Level, Vigor) → int (rolls hit dice + Vigor bonus)
- GetSpellSlots(CasterLevel) → Array of int (slots per spell level 1–5) — formula: `clamp(CasterLevel − 2×(SpellLevel−1), 0, 4)`, no data table needed
- GetLevelCap(Race, Class) → int (race/class restriction table)
- IsMultiClassAllowed(Race, Class1, Class2) → bool
- GetXPThreshold(Class, Level) → int — formula: `500 × Level × (Level−1) × ClassMultiplier`, no data table needed

## GB-019 Create Character Creation Flow
- WBP_CharacterCreation
- Step 1: Roll ability scores (call BP_DiceSystem.Roll(3, D6) × 6, allow re-roll)
- Step 2: Choose race (filters available classes based on race/class restrictions)
- Step 3: Choose class (or multi-class if race allows)
- Step 4: Choose alignment
- Step 5: Name entry
- Step 6: Portrait / combat icon selection
- Step 7: Confirm and add to party
- OnConfirm: call BPL_RulesLibrary to compute derived stats, initialise spell slots
- *(Threshold System has no exceptional-ability-score subsystem — the old Step 4 "auto-roll exceptional STR" is removed entirely, see Threshold_Ruleset_v1.md §1)*

## GB-020 Create Party Manager
- BP_PartyManager
- PartyMembers: Array of SCharacter (max 6)
- AddCharacter(SCharacter)
- RemoveCharacter(Index)
- GetActiveSpeaker → SCharacter (first living member)
- GetLivingMembers → Array of SCharacter
- IsPartyWiped → bool (all members dead)
- ReorderParty(FromIndex, ToIndex)
- AwardXPToParty(Amount) — splits XP, checks for level-up
- CheckLevelUp(CharacterIndex) — compares XP to DT_LevelProgression threshold

## GB-021 Create Monster Struct (SMonster)
- Shares core fields with SCharacter (HP, AC, THAC0, Conditions)
- MonsterID (int) — index into DT_Monsters (127 records)
- HitDice (float, supports fractional HD like 0.5)
- NumAttacks (int)
- DamagePerAttack (Array of FDiceRoll)
- SpecialAbilities (bitmask: BreathWeapon, LevelDrain, Poison, Petrify, SpellCasting, MagicResistance)
- MagicResistance (int %)
- MoralRating (int 0–12)
- XPValue (int)
- SpellList (Array of SpellID, if caster)
- PortraitID (int)
- IsUndead, IsGiantClass, IsDemonic (bool flags)

## GB-022 Create Monster Data Table
- DT_Monsters — target roster size TBD per content plan, one row per monster type
- Columns mirror SMonster fields
- All monster names, stat blocks, and special abilities are original creations — generic folklore monster names (goblin, orc, skeleton, zombie, giant rat, etc.) are fine to reuse freely; avoid transcribing stat blocks directly from any published bestiary

---

# EPIC 005 — Random Encounter System

## GB-023 Create Step Counter System
- BP_StepCounterManager
- StepCounters: Map of EncounterTableID → current step count
- IncrementAll — called on every tile move
- CheckThresholds — on each increment, roll against encounter probability
- TriggerRandomEncounter(EncounterTableID)
- Multiple counters run simultaneously (different encounter types)
- EncounterTable entry: MonsterGroupID, MinCount, MaxCount, Probability weight

## GB-024 Create Random Encounter Tables
- DT_RandomEncounters — keyed by DungeonLevelID
- Each row: EncounterTableID, Array of FEncounterEntry
- FEncounterEntry: MonsterID, MinCount, MaxCount, Weight
- Weighted random selection on trigger

---

# EPIC 006 — Event Trigger System

## GB-025 Create Trigger Type Framework
- ETriggerType enum (OnEnterTile, OnSearchTile, OnStoryFlag, OnTimedCondition, OnCombatEnd, OnItemUsed, OnDayNight, OnMoonPhase)
- STriggerEvent struct: TriggerType, ConditionFlagName, ConditionFlagValue, ActionType, ActionPayload (text/encounter/combat/transition)
- DT_TriggerEvents — all scripted events for a dungeon level

## GB-026 Create Event Manager
- BP_EventManager
- EvaluateTriggers(TileX, TileY, ETriggerType) — checks all triggers for current tile/condition
- ExecuteTrigger(STriggerEvent) — dispatches to appropriate handler:
  - ShowMessage → BP_TextManager
  - StartEncounter → BP_EncounterManager
  - StartCombat → BP_CombatManager
  - TransitionLevel → BP_MapManager
  - SetStoryFlag → BP_GameManager
  - SpawnLoot
- StoryFlags Map (string → bool) — persistent within session

---

# EPIC 007 — Encounter System

## GB-027 Create Encounter Data
- DT_Encounters — scripted non-combat encounters
- SEncounter struct: EncounterID, PortraitID, IntroText, Array of SEncounterChoice
- SEncounterChoice: ChoiceLabel (Nice/Sly/Threaten/Attack/Bribe), OutcomeType (Combat/Reward/Flee/TextOnly), OutcomePayload

## GB-028 Create Encounter Manager
- BP_EncounterManager
- StartEncounter(EncounterID) — shows portrait, displays text and choices
- ResolveChoice(ChoiceIndex) — executes outcome
- TriggerCombatFromEncounter(MonsterGroupID)
- GrantReward(ItemID or XP)

## GB-029 Create Encounter Widget
- WBP_EncounterScreen
- Portrait display (left viewport)
- Encounter text (text area)
- Choice buttons (dynamic, driven by SEncounterChoice array)
- Supports variable number of choices per encounter

---

# EPIC 008 — Combat Foundation

## GB-030 Create Combat Grid ✅ *(all steps complete — verified)*
- BP_CombatGrid — single actor with Instanced Static Mesh component (NOT per-tile actors)
- 16×16 combat grid = 4×4 dungeon-tile window around player, each dungeon tile subdivided into 4×4 combat sub-tiles
- Grid spawns at a fixed arena world location; layout generated from local dungeon data (content reflects vicinity, position does not)
- **EAoEShape enum** added: None, Radius, Cone, Line — used by GetTilesInSpellAoE; reused by GB-047/GB-051
- **TileToIndex(X, Y) / IndexToTile(Index)** — ✅ row-major helpers, all grid logic routes through them
- **TileToWorld(X, Y) → Vector** — ✅ world position of tile centre (GridOrigin + half-tile offset)
- **IsTileInBounds / IsBlocked / SetBlocked / IsTraversable** — ✅ IsTraversable = InBounds AND NOT Blocked
- **IsOccupied / GetOccupant / SetOccupant / ClearOccupant** — ✅ OccupancyMap: Map Integer → Integer (TileIndex → CombatantID)
- **GetAdjacentTiles(X, Y)** — ✅ 8-neighbour Chebyshev, bounds-filtered
- **GetTilesInRange(OriginX, OriginY, Range)** — ✅ Chebyshev square; geometric only — does not filter blocked/occupied
- **GetTilesInSpellAoE(OriginX, OriginY, Shape, Size)** — ✅ Radius working; Cone/Line stubbed with TODO until Phase 8
- **SpawnGrid(Width, Height)** — ✅ sets dims, clears OccupancyMap + BlockedTiles, calls RebuildTileInstances
- **RebuildTileInstances** — ✅ *added during build* — clears ISM instances, nested loops, skips blocked tiles, instances non-blocked tiles at TileToWorld positions. Called by both SpawnGrid and GenerateGridFromDungeon
- **GenerateGridFromDungeon(PlayerTileX, PlayerTileY)** — ✅ samples 4×4 dungeon window, blocks impassable sub-blocks, calls RebuildTileInstances. Verified: 40 instances at player (0,3), corridor shape correct

## GB-031 Create Combat Camera ✅
- CombatCameraActor — dedicated Camera Actor placed in level (NOT a camera component on BP_CombatGrid — UE doesn't expose active camera selection for components on non-pawn actors)
- Location: X=−8400, Y=1600, Z=6000 · Rotation: Pitch=−90, Yaw=0, Roll=0 (above arena centre, looking straight down)
- Perspective camera at near-vertical angle (not orthographic — perspective at −90 pitch is visually identical with none of the orthographic rendering quirks)
- Instant cut transition (Blend Time=0) via Set View Target With Blend on Player Controller
- C key test trigger in Level Blueprint: C Pressed → cut to CombatCameraActor, C Released → cut back to BP_ExplorerPawn camera
- Return target is BP_ExplorerPawn (Get Player Pawn) not pawn root — returns to correct eye height
- Pan/zoom and tile highlight deferred to GB-042/GB-051 per build order
- Verified: cut to combat view shows full arena, cut back restores exploration camera at correct height

## GB-032 Create Combatant Data (SCombatant)
- CombatantID (int)
- CharacterID (int) — **GB-039: added to enable party-panel death sync. Populated from SCharacter.CharacterID in BuildCombatants for party members; left at default 0 for monsters (unused)**
- SourceCharacter (SCharacter or SMonster reference)
- GridPosition (Vector2D)
- CurrentHP, MaxHP
- AC, THAC0 — ⚠ **field names fine to keep internally; values/formulas migrate to Defense Rating / Strike Number under GB-079**
- Initiative (int, set at round start)
- ActionsRemaining (int per turn)
- IsPlayerControlled (bool)
- Conditions (Array of ECondition)
- MovementRange (int tiles)

## GB-033 Create Combat Manager ✅ *(VS scope complete)*
- BP_CombatManager — spawned by BP_GameManager (consistent with all other managers)
- StartCombat(MonsterGroupID) — loads monster from DT_Monsters, builds SCombatant for monster and all party members, generates grid from player tile, cuts to combat camera, locks player movement
- Party CombatantIDs start at 100 to avoid collision with monster IDs
- CombatGridRef and CombatCameraRef set from Level Blueprint after SpawnManagers completes
- CurrentPlayerTileX/Y read from GameManagerRef — BP_ExplorerPawn.CompleteMovement now updates these
- TriggerCombatFromEncounter wired to StartCombat — full flow verified end to end
- Deferred: BuildInitiativeOrder (GB-034), PlaceCombatants on grid, CheckVictory/CheckDefeat (GB-046), EndCombat (GB-046), full MonsterGroup table (Phase 10)

## GB-034 Create Combat Turn Flow
- BP_CombatTurnManager
- StartPlayerTurn(CombatantID) — enable player action input
- StartEnemyTurn(CombatantID) — run AI
- OnActionComplete — advance to next combatant
- Round counter — tracks partial multiple-attack accumulation

---

# EPIC 009 — Rules Engine

## GB-035 Create Dice System
- BP_DiceSystem (pure function library)
- Roll(NumDice, DieType) → int — supports D4, D6, D8, D10, D12, D20, D100
- RollWithModifier(NumDice, DieType, Modifier) → int
- PercentileRoll → int (1–100, for Rogue skill checks)
- FDiceRoll struct: NumDice (int), DieType (int), Modifier (int)

## GB-036 Create THAC0 Attack Resolution ✅ *(legacy math — see GB-079 for Threshold System migration)*
- ResolveAttack added to **BPL_RulesLibrary** (not a separate actor — pure calculation, no state)
- **SAttackResult struct** created: bHit, RollResult, RollNeeded, Damage
- Roll D20 → natural 20 always hits, natural 1 always misses
- RollNeeded = ModifiedTHAC0 − Defender.AC
- Hit if D20Roll >= RollNeeded
- Damage = 1d6 default for VS (stub — real weapon dice post GB-053)
- STR bonus stub: STR=10, ExSTR=0 → ToHitBonus=0 (TODO GB-053)
- WeaponBonus stub: returns 0 (TODO GB-053)
- SituationalModifier stub: returns 0 (TODO GB-039)
- Monster STR/DEX post-VS note: recommend pre-baking bonuses into DT_Monsters THAC0/DamagePerAttack rather than adding ability scores to SMonster
- **GB-079 will replace the RollNeeded formula with `Strike Number − Defense Rating` (ascending DR) — see Threshold_Ruleset_v1.md §4**

## GB-037 Create Multiple Attack System — *(target: The Threshold System)*
- Extra attacks per round by class and level, clean integer breakpoints — no fractional/partial-round bookkeeping
- GetAttacksThisRound(Class, Level) → int
  - Warden: 1 (levels 1–4), 2 (levels 5–7), 2 + reroll 1s on damage (levels 8–10)
  - Rogue / Devout: 1 (levels 1–7), 2 (levels 8–10)
  - Adept: always 1
- No Sweep Attack subsystem — dropped from the Threshold System for simplicity (see Threshold_Ruleset_v1.md §3)

## GB-038 Create Saving Throw System — *(underlying formula ✅ built under GB-079; ResolveSave/gameplay wiring not yet built)*
- `BPL_RulesLibrary.ComputeSavingThrows` ✅ already implements the formula below and returns FortitudeSave/ReflexSave/WillpowerSave for a given Class/Level/Vigor/Reflex/Resolve. What's still needed for this ticket: `ResolveSave(Combatant, ESaveCategory) → bool` — actually rolling a d20 against the computed target and calling it from somewhere (nothing triggers a save yet)
  - ESaveType: Fortitude, Reflex, Willpower (3 categories, down from the original 5) — ✅ already restructured
  - Formula, no data table: `Save Target = 16 − floor(Level/2) − AbilityMod − ClassBonus`
  - Fortitude←Vigor, Reflex←Reflex, Willpower←Resolve
  - Class bonus (+2 to one save): Warden→Fortitude, Devout→Willpower, Adept→Willpower, Rogue→Reflex
- No DT_SavingThrows table needed — formula-driven, see Threshold_Ruleset_v1.md §5

## GB-039 Create Condition / Status Effect System — *(VS subset ✅ complete; full system post-VS)*
- **VS subset built:** `ApplyCondition`, `RemoveCondition`, `HasCondition`, `SetMarkerDowned` as shared functions on `BP_CombatManager` (not a separate `BP_ConditionManager` actor — consistent with `ApplyDamage` pattern, instance-state functions rather than pure library functions). Dead and Restrained fully wired into both attack paths and turn order. See `GoldBox_Project_Progress_Summary_1.md` Phase 4f for full detail.
- **Also built as part of VS:** `ApplyDeathToCharacter` on `BP_PartyManager` (party panel sync on death), `CheckDefeat` stub on `BP_CombatManager` (ends combat when all party members Dead — replace with proper game-over in GB-046). `SCharacter.Conditions` changed from single `ECondition` to `Array of ECondition`.
- **Full system (post-VS):**
  - TickConditions(CombatantID) — called each round:
    - Poison: Fortitude save or lose 1d4 HP
    - Diseased: Vigor score reduced by 1 per day until cured
    - Quickened (renamed from Hasted): +1 extra attack this round, +1 tile movement
    - Restrained (renamed from Held) / Paralysed: skip turn, attackers auto-hit ✅ skip/auto-hit already built
    - Blinded: +4 to effective Strike Number needed against this target, −4 Defense Rating, −4 on Reflex saves
    - Sapped (renamed from LevelDrained): permanent loss of 1 character level (rare; powerful undead-equivalent only)
  - GetConditionModifiers(CombatantID) → FModifierBundle (Strike Number, Defense Rating, save adjustments)
  - Multi-condition display in party panel (currently shows Dead only via Contains check)
  - Downed marker visual polish (currently 0.5 scale only)
  - Full condition list and renames: Threshold_Ruleset_v1.md §7

## GB-040 Create Morale System — *(target: The Threshold System — mechanic unchanged, generic wargame convention)*
- BP_MoraleResolver
- CheckMorale(MonsterGroup) → EMoraleResult (HoldFirm, HoldDefensive, Flee)
- MoraleRoll: 2D6 + group leader's Presence Modifier vs fixed thresholds (≤6 flee, 7–9 hold defensive at −2 to hit, ≥10 hold firm)
- Triggers: first casualty in group, >50% of group dead, leader killed
- OnFleeResult: monster moves away from party each turn, exits grid

## GB-041 Create XP & Levelling System ✅ *(legacy math — see GB-079 for Threshold System migration)*
- BP_XPManager
- AwardCombatXP(MonsterGroup) → int total XP from DT_Monsters.XPValue
- DistributeXP(Total, LivingParty) — divide equally
- CheckLevelUp(SCharacter) — compare XP to DT_LevelProgression
- OnLevelUp: recompute THAC0, saving throws, HP (roll new hit die + CON bonus), spell slots
- DT_LevelProgression — XP thresholds, THAC0, saves, HP dice per class/level
- **GB-079 will replace DT_LevelProgression's XP/THAC0 columns with formulas (`500 × Level × (Level−1) × ClassMultiplier` and `16 − floor(Level/ClassSNDivisor)`) — current table closely mirrors a published AD&D XP progression and should not ship commercially as-is. See Threshold_Ruleset_v1.md §6.**

## GB-079 Migrate Legacy Rules to The Threshold System ✅ *(essentially complete)*
- One-time refactor pass covering every legacy-math system flagged ⚠ above.
- ✅ **DONE:** GetStrengthBonus/GetConstitutionHPBonus replaced by GetMightBonus/GetVigorBonus (deleted, not kept as wrappers). Ability scores renamed throughout. AC/THAC0 renamed to DefenseRating/StrikeNumber on SCharacter, SCombatant, SMonster, and SLevelProgression. ResolveAttack fully rebuilt — turned out to need a bigger change than originally planned: percentage-based (1d100 vs a clamped 5–95% hit chance) rather than a fixed d20 formula, since the d20's narrow range made calibration too fragile once Defense Rating's polarity was corrected. See `Threshold_Ruleset_v1.md` §4 for the as-built version and its revision note. All 5 DT_Monsters rows and DT_TestParty's 4 characters converted and tested live in combat. GetXPThreshold formula built, replacing DT_LevelProgression's XPRequired table lookup (column now unused, kept for reference). New shared GetAbilityModifier function built; ComputeSavingThrows rebuilt with the 3-category formula (also relocated — found living in BP_CharacterRules unexpectedly, same situation as GetStrengthBonus before it; a full audit of BP_CharacterRules's remaining contents is worth doing). ECondition renamed (Held→Restrained, Hasted→Quickened, LevelDrained→Sapped). ECharacterClass fully renamed — all 9 values, including the 5 hybrid slots (Paladin→Templar, Ranger→Skirmisher, Druid→Sylvan, Illusionist→Shadowpriest, Assassin→Infiltrator) — renamed rather than retired, since the ruleset's 5 sketched hybrid roles exactly matched the 5 available slots. Knock-on fix: DT_LevelProgression's 20 row names renamed to match (built via Enum-to-String at runtime), verified via full level-up re-test.
- ⏸ **DELIBERATELY DEFERRED:** ESpellSchool shrink (4→2, Arcane/Divine) — not urgent since the magic system isn't built yet.
- DT_Monsters: existing rows (Goblin/Orc/Skeleton/Zombie/Giant Rat) reviewed and converted — generic folklore names needed no rename, only the numeric DefenseRating/StrikeNumber values needed migrating
- Struct field names (THAC0, AC on SCombatant/SCharacter) ended up renamed anyway (DefenseRating/StrikeNumber) rather than kept internal-only — decided full rename was worth the consistency once underway
- ⚠ **Side effect discovered:** BP_RulesTestSuite's DT_THAC0Tests, DT_StrengthBonusTests, DT_AttackResolutionTests, DT_XPThresholdTests, DT_SavingThrowTests, and DT_LevelCapTests are all stale against the migrated math to varying degrees — decided to delete and recreate them fresh at a later date rather than patch individually

---

# EPIC 010 — Combat Gameplay

## GB-042 Create Move Action ✅ *(VS scope complete)*
- Built as functions on BP_CombatManager (not WBP — no Combat HUD yet)
- M key → EnterMoveMode (bMoveMode=true, ClearHighlights, GetValidMoveTiles, HighlightTiles cyan)
- WASD → ExecuteCombatantMove(DeltaX, DeltaY) — validates IsTraversable+IsOccupied, updates OccupancyMap, moves marker, RefreshMoveHighlights
- T key → EndPlayerTurn (bMoveMode=false, ClearHighlights, OnActionComplete)
- BP_TileHighlight actor — flat plane with M_CombatTile material (EmissiveColour parameter for glow)
- GetValidMoveTiles on BP_CombatGrid — filters IsTraversable AND IsOccupied (not just geometric)
- TileIndexToInstanceIndex Map added to BP_CombatGrid (RebuildTileInstances populates it)
- TilesMovedThisTurn added to SCombatant — resets to 0 in StartPlayerTurn
- RefreshMoveHighlights separate function — prevents EnterMoveMode recursion
- MovementRange=3 for VS (rules redesign post-VS — placeholder value)
- Return Node added to OnActionComplete after StartEnemyTurn — fixes duplicate turn message

## GB-043 Create Attack Action ✅ *(VS scope complete)*
- Built as functions on BP_CombatManager (not WBP — no Combat HUD yet)
- A key → EnterTargetSelectMode (bSelectingTarget=true, posts "[Name] — select a target")
- Click BP_CombatantMarker → OnClicked → ExecutePlayerAttack(CombatantID)
- ExecutePlayerAttack: finds Attacker (CurrentCombatantID) + Defender (TargetCombatantID) → ResolveAttack → apply damage → death check → RemoveMarkerForCombatant → CheckVictory → EndCombat or OnActionComplete
- Click detection: Enable Mouse Click Events on PlayerController + OnClicked on MarkerMesh component
- IsMovementLocked used for combat input lock (CanMove reserved for tile traversability checks)
- Encounter re-trigger prevented via Story Flag (Encounter_[TriggerID]) in BP_EventManager

## GB-046 Create Combat Victory / Defeat Handling ✅ *(partial — VS scope)*
- CheckVictory: loops Combatants, returns true if all IsMonster combatants at HP≤0
- EndCombat: clears Combatants/InitiativeOrder/SpawnedMarkers arrays, resets bools, returns camera to BP_ExplorerPawn, IsMovementLocked=false, ChangeGameState(Exploration)
- Deferred: XP award (GB-041), loot drop, defeat handling, retreat system

## GB-044 Create Ambush Strike Action (Rogue) — *(renamed from Backstab, target: The Threshold System)*
- BP_AmbushResolver
- Conditions: target unaware OR Rogue moved with Stealth success last turn
- MultiplierByLevel: L1–3 ×2, L4–6 ×3, L7–9 ×4, L10 ×5 (and +4 to hit)
- ResolveAmbushStrike(Rogue, Target) → FAttackResult (with multiplied damage)

## GB-045 Create Enemy AI
- BP_EnemyAI
- ChooseAction(MonsterCombatant) → ECombatAction_AI
- Target selection priority:
  1. Nearest living party member (default)
  2. Incapacitated/held party members (finish them off)
  3. Highest-threat party member (spellcasters if monster is intelligent)
- Movement: pathfind toward chosen target (simple BFS on grid)
- Spell use: if monster has spells and slots remaining, prefer area spells against clustered targets
- CheckAoEOpportunity — fire AoE spell if ≥ 2 party members in radius
- Morale check at round start if group is broken
- Flee behaviour: move toward grid edge each turn

## GB-046 Create Combat Victory / Defeat Handling ✅ *(partial — VS scope)*
- CheckVictory: loops Combatants, returns true if all IsMonster combatants at HP≤0
- EndCombat: clears Combatants/InitiativeOrder/SpawnedMarkers arrays, resets bools (bCombatActive, bSelectingTarget, bMoveMode), returns camera to BP_ExplorerPawn, IsMovementLocked=false, ChangeGameState(Exploration)
- Deferred: XP award (GB-041), loot drop, defeat handling, retreat system

---

# MILESTONE 1 — VERTICAL SLICE

Player can:

- Load a dungeon level and explore tile-by-tile
- Read tile-triggered messages
- Trigger a scripted encounter with dialogue choices
- Enter tactical grid combat
- Execute attacks using THAC0/AC resolution *(legacy math, as actually built — see Ruleset Notice and GB-079)*
- Experience status conditions (at least: Held, Poisoned, Dead)
- Defeat enemies and receive XP
- Level up if XP threshold met
- Return to dungeon exploration
- Reach dungeon exit and transition

---

# EPIC 011 — Magic System

## GB-047 Create Spell Data — *(target: The Threshold System — 2 schools, formula-driven slots)*
- SSpell struct: SpellID, Name, School (ESpellSchool: Arcane/Divine), Level (int 1–5), CastingTime (int segments), Range (int tiles), Duration (int rounds), AoEShape (None/Radius/Cone/Line), AoESize (int), EffectType (ESpellEffect), EffectPayload (damage dice, condition, etc.)
- DT_Spells — full spell list for Arcane (Adept) and Divine (Devout), original names throughout:
  - Arcane: Force Dart, Slumber, Mesmerize, Binding Grasp, Tangling Vines, Choking Vapor, Cinder Burst, Arc of Lightning, Quicken/Sluggish, Frost Cone, Unmaking, Words of Ending
  - Divine: Mend Wounds I–IV, Blessing, Veil of Shadows, Binding Grasp, Curse, Wall of Blades, Wither, Renewal, Rekindle
  - Full list with levels and effects: Threshold_Ruleset_v1.md §9

## GB-048 Create Spell Slot Tracking — *(target: The Threshold System — formula-driven, no data table)*
- Per character: SpellSlotsCurrent[5], SpellSlotsMax[5] (arrays indexed by spell level 1–5 — array shrunk from 7 since level cap is 10 and max spell level is 5)
- MemorizedSpells[5]: Array of SpellID per spell level
- Slots populated by BPL_RulesLibrary.GetSpellSlots(CasterLevel) — formula: `clamp(CasterLevel − 2×(SpellLevel−1), 0, 4)`, no DT_SpellSlots table needed
- OnCastSpell: decrement SpellSlotsCurrent for that level
- OnRest: restore slots (handled in Camp system)

## GB-049 Create Spell Casting
- BP_SpellCaster
- CanCast(Character, SpellID) → bool (has slot, in combat or exploration as appropriate)
- CastSpell(Caster SCombatant, SpellID, TargetTile or TargetCombatant)
- Deduct slot, add casting time to initiative order
- Dispatch to BP_SpellEffectHandler

## GB-050 Create Spell Effect Handler — *(target: The Threshold System)*
- BP_SpellEffectHandler
- HandleEffect(SpellID, Caster, Target/Area):
  - Damage spells: roll damage dice, apply to all targets in AoE (including friendly!)
  - Binding Grasp: apply ECondition.Restrained to targets failing a Willpower save
  - Tangling Vines: mark tiles as difficult/impassable
  - Quicken/Sluggish: apply condition modifiers (Quickened/Slowed)
  - Veil of Shadows: apply ECondition.Blinded
  - Mend Wounds: restore HP
  - Rekindle: restore dead character (caster permanently loses 1 Vigor point)
  - Renewal: remove ECondition.Sapped, restore lost levels
- FriendlyFireCheck — AoE spells affect ALL combatants in area (no immunity for party)

## GB-051 Create AoE Grid Visualisation
- WBP_SpellAoEPreview — overlays AoE shape on combat grid before confirming cast
- Highlights tiles in AoE (red = enemies, yellow = allies, orange = mixed)
- Confirms or cancels cast

---

# EPIC 012 — Character Management

## GB-052 Create Character Screen
- WBP_CharacterScreen
- Full character sheet: all ability scores, derived stats, Defense Rating, Strike Number, saving throws, HP, XP, level
- Condition display
- Equipment display (all slots)
- Inventory display
- Spell memorisation display (slots available, memorised spells)
- Rogue skills display (if applicable)

## GB-053 Create Inventory System
- SItem struct: ItemID, Name, Type (Weapon/Armour/Misc/Consumable/QuestItem), Weight, Value (GP), MagicBonus, EquipSlot, EffectPayload
- DT_Items — master item list
- Party inventory: per-character array (max carry weight from STR table)
- PartySharedStash: optional pooled inventory

## GB-054 Create Equipment System
- BP_EquipmentManager
- EquipItem(CharacterIndex, ItemID, EquipSlot) — validates class/race restrictions
- UnequipItem(CharacterIndex, EquipSlot)
- OnEquip: recompute Defense Rating (armour), Strike Number bonus (magic weapon), saving throw bonus (magic rings), etc.
- AlignedItemCheck — if item is alignment-restricted, damage character on equip if misaligned

## GB-055 Create Loot System
- BP_LootManager
- GenerateLoot(MonsterGroupID) — rolls from DT_MonsterLoot tables
- ShowLootScreen(LootArray) — displays loot pile after combat
- DistributeLoot — player assigns items to party members

## GB-056 Create Shop / Merchant System
- WBP_ShopScreen
- DT_ShopInventory — keyed by ShopID, lists available ItemIDs with quantities and prices
- BuyItem(ShopID, ItemID, CharacterIndex)
- SellItem(CharacterIndex, ItemID) → GP value (50% base)
- Party gold tracked in BP_PartyManager

---

# EPIC 013 — Save System

## GB-057 Create Save Game
- USaveGame subclass: BP_GoldBoxSave
- Serialise: full party (all SCharacter data), current dungeon level + position, story flags map, session step counters, party gold
- SaveSlots: 3 manual slots + 1 autosave
- SaveToSlot(SlotIndex)

## GB-058 Create Load Game
- LoadFromSlot(SlotIndex) → bool
- Deserialise all party and world state
- Restore current dungeon level and player position
- Restore story flags

## GB-059 Create Auto Save
- TriggerAutoSave on: level transition, camp, post-combat
- AutoSave slot separate from manual slots

---

# EPIC 014 — Camp System

## GB-060 Create Camp Screen
- WBP_CampScreen
- Available from exploration command menu (not during combat)
- Options: Rest, Memorise Spells, Reorder Party, View Characters, Save Game, Quit

## GB-061 Create Rest & Spell Memorisation
- RestDuration = 4 hours base + 15 min per spell level per spell memorised (sequential)
- Player selects spells to fill each spell slot per caster
- OnRestComplete: restore memorised spell slots, restore HP (optional rule — full rest)
- Random encounter roll during rest (camp is not always safe)

## GB-062 Create Party Reordering
- WBP_PartyReorder — drag-and-drop or swap party member order
- Order affects: who speaks first in encounters, marching order (future)

## GB-063 Create Random Rest Encounter
- On rest begin: roll for random encounter (% chance per dungeon level)
- If triggered: interrupt rest, start combat
- If rest completes without interruption: apply rest benefits

---

# EPIC 015 — Alignment & Story Systems

## GB-064 Create Alignment Tracking
- Each SCharacter has EAlignment
- AlignedItemCheck on equip (see GB-054)
- StoryFlag outcomes can be influenced by party alignment (good/evil choices)
- NPC reaction modifier based on party leader alignment (future)

## GB-065 Create Story Flag System
- BP_GameManager.StoryFlags Map (string key → bool value)
- SetFlag(Name, Value), CheckFlag(Name) → bool
- Flags persist in save file
- Used by: trigger conditions, encounter outcomes, dialogue branches, door/gate unlocks

---

# EPIC 016 — Overland / Wilderness Map

## GB-066 Create Overland Map
- WBP_OverlandMap — top-down 2D map view (distinct from combat grid)
- Party represented as single icon moving between named locations
- DT_OverlandLocations: LocationID, Name, MapPosition, ConnectedTo (array), DungeonLevelID
- Supports up to 4 wilderness areas per campaign

## GB-067 Create Overland Movement
- BP_OverlandManager
- MoveToLocation(LocationID) — validates connection exists
- OverlandStepCounter — random encounters during travel
- ArriveAtLocation → load dungeon level or town

## GB-068 Create Overland Random Encounters
- DT_OverlandEncounters — keyed by WildernessAreaID
- Weather system modifier: weather affects encounter rate and movement (see GB-070)

---

# EPIC 017 — Extended World Systems

## GB-069 Create NPC Party Member System
- NPC allies use SCharacter struct (IsPlayerControlled = false)
- BP_NPCPartyManager — join, leave, manage NPC allies
- NPC combat: AI-controlled but using same rules as player characters
- NPC XP share: divide party XP among all living members including NPCs

## GB-070 Create Weather System
- EWeather (Clear, Overcast, Rain, Storm, Snow, Blizzard)
- BP_WeatherManager — changes weather per overland day
- Weather effects on overland: movement speed modifier, encounter rate modifier
- Weather display in overland HUD

## GB-071 Create Character Import / Export
- ExportCharacter(SCharacter) → save to standalone character file (.sav)
- ImportCharacter(FilePath) → SCharacter
- Import applies level caps for the receiving campaign (strip excess XP if over cap)
- Item filter on import (campaign-breaking items blocked)

---

# EPIC 018 — Audio System

## GB-072 Create Audio Manager
- BP_AudioManager
- PlayMusic(TrackID) — dungeon ambient, combat theme, victory fanfare, camp music
- PlaySFX(SFXID) — sword hit, spell cast, door open, footstep, monster death
- StopMusic, FadeMusic
- Audio state changes with EGameState

## GB-073 Create Audio Data Tables
- DT_MusicTracks — TrackID → sound asset + loop bool
- DT_SFX — SFXID → sound asset
- Audio categories: Music (looping), Ambience, Combat SFX, UI SFX

---

# MILESTONE 2 — PLAYABLE ALPHA

- Multiple connected dungeon levels with level transitions
- Full character creation flow
- Complete party of 6 with all ability scores, Strike Number, Defense Rating, saving throws
- Full combat: Strike Number/Defense Rating resolution, three-category saving throws, conditions, multiple attacks, Ambush Strike, morale, flee
- Working magic system: spell memorisation, casting, AoE with friendly fire
- Camp system: rest, memorise spells, save/load
- Loot and basic inventory
- Overland map connecting dungeon locations
- Random encounters (dungeon and overland)
- Audio: music and key SFX

---

# EPIC 019 — Content Pipeline

## GB-074 Create Map Editor Tools
- Level design tools for authoring SDungeonMap data tables
- Visual grid editor for tile layout, wall placement, door placement
- Tile property editor: event trigger ID, message ID, special flags
- Export to DT_DungeonMaps format

## GB-075 Create Encounter Authoring
- Encounter editor: create DT_Encounters rows
- Choice builder: add/remove SEncounterChoice entries
- Outcome preview

## GB-076 Create Dialogue Authoring
- Simple dialogue tree tool (branching text, flag conditions, flag outcomes)
- Export to DT_DialogueTrees

## GB-077 Create Dungeon Themes
- Theme asset bundles: mesh sets + materials per visual style (Stone, Crypt, Ice Cavern, Forest Ruin)
- BP_DungeonGenerator selects theme bundle on level load

## GB-078 Create Monster Group Templates
- DT_MonsterGroups — predefined combat setups (MonsterID, Count, Formation, GridLayout)
- Used by scripted encounters and random encounter tables

---

# MILESTONE 3 — FULL GAME

- Complete campaign with overland map, multiple dungeon regions
- Full character progression from level 1 to level cap (race/class dependent)
- All character classes playable with correct rules (multi-class, dual-class)
- Full spell lists (Arcane, Divine)
- Equipment economy: shops, loot, magic items
- Character import/export between connected campaigns
- NPC party members
- Weather system (overland)
- Alignment-tracked story consequences
- Full audio (music + SFX)
- Full content pipeline (map editor, encounter editor, dialogue editor)

---

# Summary: Systems Added vs Original Roadmap

| System | Original | Revised |
|---|---|---|
| Character struct (full AD&D fields) | Partial | ✅ Complete |
| Character creation flow | Missing | ✅ GB-019 |
| Race/class restrictions & level caps | Missing | ✅ GB-018 |
| Saving throws | Missing | ✅ GB-038 |
| Condition / status effect system | Missing | ✅ GB-039 |
| Multiple attacks per round | Missing | ✅ GB-037 |
| Sweep attacks (Fighters vs <1HD) | Missing | ✅ GB-037 |
| Backstab (Thief) | Missing | ✅ GB-044 |
| Morale system (monsters) | Missing | ✅ GB-040 |
| XP award & level-up | Missing | ✅ GB-041 |
| AoE spells with friendly fire | Missing | ✅ GB-050, GB-051 |
| Spell slots by level (1–7) | Partial | ✅ GB-048 |
| Full spell data table | Missing | ✅ GB-047 |
| Encounter portrait system | Missing | ✅ GB-016 |
| Step counter / random encounters | Missing | ✅ GB-023, GB-024 |
| Event trigger type framework | Missing | ✅ GB-025, GB-026 |
| Dungeon level transitions | Missing | ✅ GB-009 |
| Alignment system | Missing | ✅ GB-064 |
| Story flags | Partial | ✅ GB-065 |
| Overland / wilderness map | Missing | ✅ GB-066–068 |
| Camp: rest + memorisation | Partial | ✅ GB-061 |
| Character import/export | Missing | ✅ GB-071 |
| NPC party members | Missing | ✅ GB-069 |
| Weather system | Missing | ✅ GB-070 |
| Audio system | Missing | ✅ GB-072, GB-073 |
| Shop / merchant system | Missing | ✅ GB-056 |
| Monster struct (127 records) | Missing | ✅ GB-021, GB-022 |
| Enemy AI (full logic) | Partial | ✅ GB-045 |
| Combat retreat / flee | Missing | ✅ GB-046 |
| Original ruleset (replacing AD&D-derived placeholder math) | Missing | ✅ Threshold_Ruleset_v1.md, migration tracked as GB-079 |
