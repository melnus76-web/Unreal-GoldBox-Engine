# Gold Box RPG — Blueprint Reference Document
## UE5.7 · Blueprints Only · Solo Dev

---

> **Ruleset Notice:** GB-079 (the migration to The Threshold System) is now **essentially complete**, with one deliberately deferred item. **Done and tested:** ability scores, the entire attack resolution system (percentage-based, replacing THAC0/AC d20 math), XP thresholds (formula-based), saving throws (3 categories, formula-based, replacing the 5-category table), and the `ECharacterClass`/`ECondition` enum renames (including `DT_LevelProgression`'s row names, which had to follow the class rename since they're built via `Enum to String`). **Deliberately deferred:** `ESpellSchool`'s 4→2 shrink, since the magic system itself isn't built yet — no urgency there. Note that renaming an enum is not the same as building new mechanics: the class enum now reads `Warden`/`Devout`/etc, conditions are now 8/12 complete (Dead, Restrained, Poisoned, Blinded, Quickened, Slowed, Paralysed, Unconscious complete; Diseased, Sapped, Petrified deferred post-VS). Per-class mechanics (Hit Dice, Extra Attacks, Ambush Strike) remain separate tickets (GB-037, GB-040, GB-044). See `Threshold_Ruleset_v1.md` §11 for the full status table.

---

## BP_GameManager
**Parent Class:** Game Instance  
**Location:** Blueprints/Core/

### Variables
| Variable | Type | Notes |
|---|---|---|
| CurrentGameState | EGameState | Current game state |
| StoryFlags | Map String → Boolean | Persistent story flags |
| CurrentDungeonLevelID | Integer | Active dungeon level |
| CurrentPlayerTileX | Integer | Player X position |
| CurrentPlayerTileY | Integer | Player Y position |
| PartyManager | BP_PartyManager (object ref) | Set in SpawnManagers |
| MapManager | BP_MapManager (object ref) | Set in SpawnManagers |
| TextManager | BP_TextManager (object ref) | Set in SpawnManagers |
| EventManager | BP_EventManager (object ref) | Set in SpawnManagers |
| DungeonGenerator | BP_DungeonGenerator (object ref) | Set in SpawnManagers |
| EncounterManager | BP_EncounterManager (object ref) | Set in SpawnManagers |
| EnemyAI | BP_EnemyAI (object ref) | Set in SpawnManagers |
| XPManagerRef | BP_XPManager (object ref) | Set in SpawnManagers |
| TempCharacter | SCharacter | Temporary character working variable |

### Event Dispatchers
| Dispatcher | Inputs | Notes |
|---|---|---|
| OnGameStateUpdated | NewState (EGameState) | Fires when game state changes |

### Functions
| Function | Inputs | Outputs | Notes |
|---|---|---|---|
| InitialiseGameState | — | — | Sets initial EGameState |
| SpawnManagers | — | — | Spawns all manager actors, sets their refs |
| InitialiseParty | — | — | Calls BP_PartyManager.InitialiseTestParty |
| InitialiseWorld | — | — | Calls BP_MapManager.LoadMap(1) |
| ChangeGameState | NewState (EGameState) | — | Sets state, calls OnGameStateUpdated |
| SetStoryFlag | FlagName (String), Value (Boolean) | — | Sets entry in StoryFlags map |
| CheckStoryFlag | FlagName (String) | Value (Boolean) | Reads StoryFlags map |
| LoadDungeonLevel | LevelID (Integer) | — | Loads dungeon, sets state to Exploration |
| StartCombat | — | — | Calls BP_CombatManager.StartCombat — wired in GB-033 |
  | EndCombat | Victory (Boolean) | — | Delegates to BP_CombatManager.EndCombat |
  | AwardXP | Amount (Integer) | — | Delegates to BP_XPManager.AwardCombatXP + DistributeXP |

> Note: EndCombat and AwardXP were originally implemented on BP_CombatManager/BP_XPManager. Later GB-003 tasks added `EndCombat(Victory)` and `AwardXP(Amount)` directly on BP_GameManager as convenience wrappers — they delegate to CombatManager/XPManager internally.

### Event Init Flow
```
Event Init
  → Get Current Level Name == "L_TestSuite"? → Branch
      True  → (exit — skip all initialisation)
      False
        → InitialiseGameState
        → SpawnManagers
        → InitialiseParty
        → InitialiseWorld
        → Print "GameManager Ready"
```

### SpawnManagers Flow
```
SpawnManagers
  → Spawn BP_PartyManager → SET PartyManager
  → Spawn BP_MapManager   → SET MapManager
  → Spawn BP_TextManager  → SET TextManager
  → Spawn BP_EventManager → SET EventManager
        → SET EventManager.MapManagerRef = MapManager
        → SET EventManager.TextManagerRef = TextManager
  → Spawn BP_DungeonGenerator → SET DungeonGenerator
  → Spawn BP_EncounterManager → SET EncounterManager
        → SET EncounterManager.TextManagerRef = TextManager
        → SET EncounterManager.GameManagerRef = Self
  → Spawn BP_CombatManager → SET CombatManager
        → SET CombatManager.GameManagerRef = Self
        → SET CombatManager.TextManagerRef = TextManager
        → SET CombatManager.PartyManagerRef = PartyManager
  → Spawn BP_EnemyAI → SET EnemyAI
  → Spawn BP_XPManager → SET XPManagerRef
  → Print "Managers Spawned"
```

---

## BP_PartyManager
**Parent Class:** Actor  
**Location:** Blueprints/Managers/

### Variables
| Variable | Type | Notes |
|---|---|---|
| PartyMembers | SCharacter Array | Up to 6 characters |
| PartyGold | Integer | Shared party gold |

> **SCharacter.CharacterID** (Integer) — added GB-041 to support reliable character lookups. Auto-assigned by AddCharacter (= array length at time of insertion); do not set manually elsewhere. Name field is NOT guaranteed unique — always match by CharacterID, never Name.

> **SCharacter ability scores** ✅ GB-079 — renamed Strength→Might, Intelligence→Acuity, Wisdom→Resolve, Dexterity→Reflex, Constitution→Vigor, Charisma→Presence. `StrengthExceptional` deleted entirely (no exceptional-score subsystem in The Threshold System). `AC`→`DefenseRating`, `THAC0`→`StrikeNumber` renamed on SCharacter, SCombatant, SMonster, and the SLevelProgression column. DT_TestParty's four characters all updated to StrikeNumber=50 (new level-1 baseline for every class).

> **SCharacter.Conditions** — **GB-039: changed from single `ECondition` value to `Array of ECondition`** to support multiple simultaneous conditions. `UpdateCharSlot` in `WBP_ExplorationHUD` updated to check for `Dead` via `Contains` rather than a direct enum match.

### Event Dispatchers
| Dispatcher | Inputs | Notes |
|---|---|---|
| OnPartyUpdated | — | Fires when party data changes. `WBP_ExplorationHUD` binds to this and calls `RefreshPartyPanel` in response. **GB-039: also fired by `ApplyDeathToCharacter` to sync party panel on player death mid-combat** |

### Functions
| Function | Inputs | Outputs | Notes |
|---|---|---|---|
| InitialiseTestParty | — | — | Reads DT_TestParty, adds 4 characters |
| AddCharacter | NewCharacter (SCharacter) | — | Auto-assigns CharacterID = current PartyMembers array length before appending. Guarantees unique sequential IDs (0, 1, 2, 3...) regardless of caller |
| GetLivingMembers | — | Array of SCharacter | Filters out Dead condition. Was briefly mis-named GetPartyMembers — renamed back during GB-041 since the dead-filtering logic was still present under the wrong name |
| IsPartyWiped | — | Boolean | True if all members dead |
| AddXPToCharacter | CharacterID (Integer), XPAmount (Integer) | — | Loops PartyMembers, matches by CharacterID (NOT Name), adds XPAmount to XP field, writes updated SCharacter back via Set Array Elem. Built for GB-041, called by BP_XPManager.DistributeXP |
| ApplyDeathToCharacter | CharacterID (Integer) | — | **GB-039 — built.** Loops PartyMembers, matches by CharacterID, promotes Conditions to local variable, adds Dead, sets CurrentHP=0, writes back via Make SCharacter + Set Array Elem, then fires OnPartyUpdated to trigger panel refresh. Called from BP_EnemyAI.ExecuteEnemyAttack via PartyManagerRef when a player character's HP drops to 0 |

#### AddXPToCharacter
```
AddXPToCharacter(CharacterID, XPAmount)
  → For Each PartyMembers
       → Break SCharacter → CharacterID, XP
       → Branch: CharacterID == input CharacterID?
            False → skip
            True
              → NewXP = XP + XPAmount
              → Make SCharacter (all fields from Break, XP = NewXP)
              → Set Array Elem (PartyMembers, Array Index, new SCharacter)
```

#### ApplyDeathToCharacter *(GB-039)*
```
ApplyDeathToCharacter(CharacterID)
  → For Each PartyMembers
       → Break SCharacter → CharacterID, Conditions
       → Branch: CharacterID == input CharacterID?
            False → Continue
            True
              → Promote Conditions to local UpdatedConditions
              → Get UpdatedConditions → Add (Dead)
              → Make SCharacter (all fields from Break, CurrentHP=0, Conditions=fresh Get UpdatedConditions)
              → Set Array Elem (PartyMembers, Array Index, new SCharacter)
              → Call OnPartyUpdated (fires RefreshPartyPanel in WBP_ExplorationHUD)
```

---

## BP_MapManager
**Parent Class:** Actor  
**Location:** Blueprints/Managers/

### Variables
| Variable | Type | Notes |
|---|---|---|
| CurrentMap | SDungeonMap | Active dungeon map |
| CurrentLevelID | Integer | Active level ID |
| IsMapLoaded | Boolean | True when map ready |
| TileGrid | SDungeonTile Array | Flat array of all tiles |
| PlayerX | Integer | Current player X |
| PlayerY | Integer | Current player Y |

### Functions
| Function | Inputs | Outputs | Notes |
|---|---|---|---|
| LoadMap | LevelID (Integer) | — | Tries DT_DungeonMaps, falls back to GenerateTestMap |
| GenerateTestMap | — | — | Hardcoded 8×8 test dungeon |
| AddTileToMap | TileX, TileY, Passable, TileType, MessageID, EventTriggerID, IsExit, WallN/E/S/W | — | Sets tile in TileGrid |
| FillMapWithEmptyTiles | — | — | Initialises TileGrid with empty tiles |
| GetTile | TileX, TileY (Integer) | SDungeonTile | Returns tile at coordinates. Pure. |
| CheckWall | TileX, TileY (Integer), Direction (EDirection) | Boolean | Returns wall state. Pure. |
| CheckDoor | TileX, TileY (Integer), Direction (EDirection) | Boolean | Returns door state. Pure. |
| UpdatePlayerPosition | NewX, NewY (Integer) | — | Updates PlayerX/Y |
| GetExitTile | — | Vector2D | Returns exit tile coordinates. Pure. |
| IncrementStepCounter | EncounterType (Integer) | — | Increments step count, stub random encounter check |
| HandleLevelTransition | — | — | Stub |

### Test Dungeon Layout (GenerateTestMap)
```
8×8 grid. All unlisted tiles = wall (impassable).

Tile  X  Y   Special
─────────────────────────────────────────
      0  3   Floor, WallN, WallS, WallW
      1  3   Floor, WallN, WallS
      2  3   Floor, WallN, WallS
      3  3   Floor, WallN, WallS, MessageID=2, EventTriggerID=2
      4  3   Floor, WallN, WallS, EventTriggerID=1
      5  3   Floor, WallN
      5  4   Floor, WallE, WallW  (turn south)
      5  5   Floor, WallE, WallW
      5  6   Floor, WallE, WallS, WallW, IsExit=true

Player start: X=0, Y=3, facing North
```

---

## BP_TextManager
**Parent Class:** Actor  
**Location:** Blueprints/Managers/

### Event Dispatchers
| Dispatcher | Inputs | Notes |
|---|---|---|
| OnMessagePosted | Message (Text), MessageType (EMessageType) | Fires when PostMessage called |
| OnClearLog | — | Fires when ClearLog called |

### Functions
| Function | Inputs | Outputs | Notes |
|---|---|---|---|
| PostMessage | Message (Text), MessageType (EMessageType) | — | Calls OnMessagePosted dispatcher |
| ClearLog | — | — | Calls OnClearLog dispatcher |

---

## BP_EventManager
**Parent Class:** Actor  
**Location:** Blueprints/Managers/

### Variables
| Variable | Type | Notes |
|---|---|---|
| MapManagerRef | BP_MapManager (object ref) | Set by BP_GameManager on spawn |
| TextManagerRef | BP_TextManager (object ref) | Set by BP_GameManager on spawn |
| StoryFlags | Map String → Boolean | Local story flag cache |

### Functions

#### EvaluateTriggers
**Inputs:** TileX (Integer), TileY (Integer), TriggerType (ETriggerType)

```
EvaluateTriggers(TileX, TileY, TriggerType)
  → GetTile(TileX, TileY) [pure — no exec]
  → Break SDungeonTile → EventTriggerID
  → Branch: EventTriggerID > 0?
      False → (exit)
      True
        → Get Data Table Row Names (DT_TriggerEvents)
        → For Each Loop
            Loop Body
              → Get Data Table Row (DT_TriggerEvents, ArrayElement)
              → Out Row pins exposed directly (no Break needed)
              → Branch: Out Row Trigger ID == EventTriggerID?
                  False → (next row)
                  True
                    → Switch on ETriggerType (Out Row Trigger Type)
                        OnEnterTile
                          → Branch: Out Row Is Active?
                              False → (next row)
                              True
                                → Make STriggerEvent (all Out Row pins)
                                → ExecuteTrigger(STriggerEvent)
                        (other types — unconnected for now)
```

#### ExecuteTrigger
**Inputs:** TriggerEvent (STriggerEvent)

```
ExecuteTrigger(TriggerEvent)
  → Break STriggerEvent
  → Switch on ETriggerActionType (Action Type)
      ShowMessage
        → TextManagerRef.PostMessage(MessageText, Exploration)
      StartEncounter
        → Get Game Instance → Cast to BP_GameManager
          → Get EncounterManager
            → EncounterManager.StartEncounter(EncounterID)
      StartCombat     → Print "Not implemented yet"
      TransitionLevel → Print "Not implemented yet"
      SetStoryFlag    → Print "Not implemented yet"
      SpawnLoot       → Print "Not implemented yet"
      Default         → Print "Unknown action type"
```

---

## BP_EncounterManager
**Parent Class:** Actor  
**Location:** Blueprints/Managers/

### Variables
| Variable | Type | Notes |
|---|---|---|
| GameManagerRef | BP_GameManager (object ref) | Set by BP_GameManager on spawn |
| TextManagerRef | BP_TextManager (object ref) | Set by BP_GameManager on spawn |
| CurrentEncounter | S_Encounter | Stores the active encounter data |
| CurrentEncounterID | Integer | Tracks which encounter is active |
| EncounterScreenRef | WBP_EncounterScreen (object ref) | Set from Level Blueprint on BeginPlay |

### Functions

#### StartEncounter
**Inputs:** EncounterID (Integer)

```
StartEncounter(EncounterID)
  → SET CurrentEncounterID = EncounterID
  → Get Data Table Row (DT_Encounters, Conv_IntToString(EncounterID) → Conv_StringToName)
  → Branch: Row Found?
      False → Print "Encounter not found: " + EncounterID (exit)
      True
        → SET CurrentEncounter = Out Row
        → Break S_Encounter → IntroText, Choices, PortraitID
        → GameManagerRef.ChangeGameState(Encounter)
        → TextManagerRef.PostMessage(IntroText, Encounter)
        → EncounterScreenRef.ShowPortrait(PortraitID)
        → EncounterScreenRef.SetEncounterManagerRef = Self
        → EncounterScreenRef.SetEncounterData(IntroText, Choices)
        → EncounterScreenRef.Add to Viewport (Z-Order 1)
        → Get Player Pawn → Cast to BP_ExplorerPawn → SET CanMove = false
```

#### ResolveChoice
**Inputs:** ChoiceIndex (Integer)

```
ResolveChoice(ChoiceIndex)
  → GET CurrentEncounter → Break S_Encounter → Choices array
  → Array Get (Choices[ChoiceIndex])
  → Break S_EncounterChoice → ChoiceLabel, OutcomeType, OutcomePayload
  → Switch on EOutcomeType (OutcomeType)
      Flee
        → TextManagerRef.PostMessage(Conv_StringToText(OutcomePayload), Encounter)
        → GameManagerRef.ChangeGameState(Exploration)
        → EncounterScreenRef.HidePortrait
        → EncounterScreenRef.Remove from Parent
        → Get Player Pawn → Cast to BP_ExplorerPawn → SET CanMove = true
      TextOnly
        → TextManagerRef.PostMessage(Conv_StringToText(OutcomePayload), Encounter)
      Reward
        → GrantReward(ItemID=0, XPAmount=Conv_StringToInt(OutcomePayload))
        → TextManagerRef.PostMessage("Reward granted.", Encounter)
        → GameManagerRef.ChangeGameState(Exploration)
        → EncounterScreenRef.HidePortrait
        → EncounterScreenRef.Remove from Parent
        → Get Player Pawn → Cast to BP_ExplorerPawn → SET CanMove = true
      Combat
        → TriggerCombatFromEncounter(Conv_StringToInt(OutcomePayload))
        → EncounterScreenRef.HidePortrait
```

#### TriggerCombatFromEncounter
**Inputs:** MonsterGroupID (Integer)

```
TriggerCombatFromEncounter(MonsterGroupID)
  → Get Game Instance → Cast to BP_GameManager → Get CombatManager
  → CombatManager.StartCombat(MonsterGroupID)
  → Print "TriggerCombatFromEncounter called: " + MonsterGroupID
```

#### GrantReward
**Inputs:** ItemID (Integer), XPAmount (Integer)

```
GrantReward(ItemID, XPAmount)
  → [TODO GB-041: BP_XPManager.AwardCombatXP]
  → [TODO GB-055: BP_LootManager item grant when inventory built]
  → Print "GrantReward called — XP: " + XPAmount + " ItemID: " + ItemID
```

---

## BP_ExplorerPawn
**Parent Class:** Pawn  
**Location:** Blueprints/Exploration/

### Variables
| Variable | Type | Notes |
|---|---|---|
| GridX | Integer | Current tile X |
| GridY | Integer | Current tile Y |
| IsMovementLocked | Boolean | Prevents movement during timeline transition |
| CanMove | Boolean | Default true. False during Encounter, Combat, Camp etc |
| MapManager | BP_MapManager (object ref) | Set on BeginPlay |
| CurrentFacing | EDirection | North/East/South/West |

### Key Functions

#### CanInitiateMove
```
CanInitiateMove(MoveDirection)
  → Branch: CanMove?
      False → Return false immediately
      True
        → Branch: NOT IsMovementLocked?
            False → Return false
            True
              → CheckWall(GridX, GridY, Direction) → Branch: HasWall?
                  True  → GetTargetTile → CanMoveTo → Branch
                              True  → SET LocalCanMove = true  → Return CanMove
                              False → SET LocalCanMove = false → Return CanMove
```

#### CompleteMovement
```
CompleteMovement(TargetTileX, TargetTileY)
  → SET GridX = TargetTileX
  → SET GridY = TargetTileY
  → MapManager.UpdatePlayerPosition(GridX, GridY)
  → Get Game Instance → Cast to BP_GameManager → Get Event Manager
  → EventManager.EvaluateTriggers(GridX, GridY, OnEnterTile)
  → MapManager.IncrementStepCounter(0)
  → SET IsMovementLocked = false
```

#### BeginPlay
```
BeginPlay
  → Get Player Controller (Index 0)
    → Set Show Mouse Cursor (true)
    → Set Input Mode Game and UI
```

---

## BP_CombatGrid
**Parent Class:** Actor  
**Location:** Blueprints/Combat/  
**Status:** ✅ GB-030 — complete (all functions built and tested, corridor-shaped arena verified)

### Design Decisions
- **Arena = vicinity:** 4×4 dungeon-tile window around the player, each dungeon tile subdivided into 4×4 combat sub-tiles → 16×16 combat grid
- **Fixed world placement:** grid actor spawns at a constant arena location. Layout (blocked tiles) is generated from dungeon data — *content* reflects the vicinity, *position* does not. Keeps GB-031 camera target constant and avoids dungeon mesh intersection
- **Single actor + Instanced Static Mesh component:** one ISM instance per non-blocked tile, added in row-major order. No per-tile actors (256 actors avoided)
- **Distance metric: Chebyshev (8-directional)** — GetAdjacentTiles, GetTilesInRange, and all future pathing (GB-042, GB-045) use this metric
- **OccupancyMap keyed by Integer TileIndex** — Vector2D is float-based and not hashable as a Blueprint Map key. Vector2D remains fine as an array *output* type
- **Blocking from Passable flag only for VS** — dungeon edge walls between two passable tiles not mapped to sub-tiles. Post-VS refinement
- **Window centring:** player at local dungeon tile (1,1) of the 4×4 window. Facing-biased window deferred post-VS
- **RebuildTileInstances** is a separate function called by both SpawnGrid and GenerateGridFromDungeon — branches on IsBlocked per tile: False → adds floor instance (TileMeshISM) + records TileIndexToInstanceIndex; True → adds wall instance (WallMeshISM) at the same TileToWorld position, no index recorded
- **Both ISM components must be cleared at the start of RebuildTileInstances** — Clear Instances called once for TileMeshISM and once for WallMeshISM. Missing the WallMeshISM clear causes wall instances to accumulate across rebuilds, visually blocking tiles that should be walkable

### Variables
| Variable | Type | Default | Notes |
|---|---|---|---|
| GridWidth | Integer | 16 | Set by SpawnGrid |
| GridHeight | Integer | 16 | Set by SpawnGrid |
| TileSize | Float | 200.0 | World units per combat sub-tile |
| GridOrigin | Vector | actor location | Set from GetActorLocation in BeginPlay |
| OccupancyMap | Map Integer → Integer | empty | TileIndex → CombatantID |
| BlockedTiles | Set of Integer | empty | TileIndices of impassable sub-tiles |
| TileMeshISM | Instanced Static Mesh Component | M_CombatTile material | Per-instance highlight via EmissiveColour parameter |
| WallMeshISM | Instanced Static Mesh Component | Brick cube placeholder mesh | Renders blocked tiles as visible wall blocks — added during wall visibility polish pass |
| TileIndexToInstanceIndex | Map Integer → Integer | empty | TileIndex → ISM instance index. Populated in RebuildTileInstances (floor tiles only — walls do not need lookup) |
| SpawnedHighlights | Array of BP_TileHighlight | empty | Active highlight actors — cleared in ClearHighlights |

### Enums Added
| Enum | Values | Notes |
|---|---|---|
| EAoEShape (or E_AoEShape) | None, Radius, Cone, Line | Used by GetTilesInSpellAoE; reused by GB-047/GB-051 |

### Functions
| Function | Pure | Inputs | Outputs | Status |
|---|---|---|---|---|
| TileToIndex | ✅ | X, Y (int) | Index (int) | ✅ Built |
| IndexToTile | ✅ | Index (int) | X, Y (int) | ✅ Built |
| TileToWorld | ✅ | X, Y (int) | WorldPos (Vector) | ✅ Built |
| IsTileInBounds | ✅ | X, Y (int) | bInBounds (bool) | ✅ Built |
| IsBlocked | ✅ | X, Y (int) | bBlocked (bool) | ✅ Built |
| SetBlocked | ❌ | X, Y (int) | — | ✅ Built |
| IsTraversable | ✅ | X, Y (int) | bTraversable (bool) | ✅ Built |
| IsOccupied | ✅ | X, Y (int) | bOccupied (bool) | ✅ Built |
| GetOccupant | ✅ | X, Y (int) | CombatantID (int), bFound (bool) | ✅ Built |
| SetOccupant | ❌ | X, Y, CombatantID (int) | — | ✅ Built |
| ClearOccupant | ❌ | X, Y (int) | — | ✅ Built |
| GetAdjacentTiles | ✅ | X, Y (int) | Tiles (Vector2D[]) | ✅ Built |
| GetTilesInRange | ✅ | OriginX, OriginY, Range (int) | Tiles (Vector2D[]) | ✅ Built |
| GetTilesInSpellAoE | ✅ | OriginX, OriginY (int), Shape (EAoEShape), Size (int) | Tiles (Vector2D[]) | ✅ Built (Radius only; Cone/Line stubbed) |
| SpawnGrid | ❌ | Width, Height (int) | — | ✅ Built |
| RebuildTileInstances | ❌ | — | — | ✅ Built |
| GenerateGridFromDungeon | ❌ | PlayerTileX, PlayerTileY (int) | — | ✅ Built |
| FindNearestTraversableTile | ❌ | PreferredX, PreferredY (int) | FoundX, FoundY (int), bFound (bool) | ✅ Built |
| GetValidMoveTiles | ✅ | OriginX, OriginY, Range (int) | ValidTiles (Vector2D[]) | ✅ Built — filters IsTraversable AND IsOccupied |
| HighlightTiles | ❌ | Tiles (Vector2D[]), Colour (LinearColor) | — | ✅ Built — spawns BP_TileHighlight actors |
| ClearHighlights | ❌ | — | — | ✅ Built — destroys SpawnedHighlights, clears array |

### Key Function Notes

**TileToIndex / IndexToTile:** row-major. `Index = (Y × GridWidth) + X`. Integer divide in IndexToTile (not float). All grid logic routes through these two.

**TileToWorld:** `GridOrigin + (X × TileSize + TileSize/2, Y × TileSize + TileSize/2, 0)`. Half-tile offset centres position in tile. Used by RebuildTileInstances and (later) combatant placement.

**IsTraversable:** `IsTileInBounds AND NOT IsBlocked`. The canonical movement/AI check — GB-042 and GB-045 must use this, not raw IsBlocked.

**GetTilesInRange:** geometric only — does not filter blocked or occupied tiles. Caller's responsibility (via IsTraversable). Chebyshev square = the nested ±Range loop itself, no distance formula needed.

**GetTilesInSpellAoE:** Radius = GetTilesInRange equivalent. Cone/Line print "not implemented (Phase 8)" and return empty array. TODO comment block in graph references GB-047/GB-051.

**SpawnGrid:** sets GridWidth/GridHeight, clears OccupancyMap and BlockedTiles, calls RebuildTileInstances.

**RebuildTileInstances:** clears ISM instances, nested Y/X loops (0..GridHeight−1, 0..GridWidth−1), Branch on IsBlocked — False → Add Instance at TileToWorld position with scale TileSize÷100×0.95. Outer Y loop Completed dead-ends (no Return node in void function). Called by both SpawnGrid and GenerateGridFromDungeon.

### GenerateGridFromDungeon Flow
```
GenerateGridFromDungeon(PlayerTileX, PlayerTileY)
  → SpawnGrid(16, 16)
  → WindowOriginX = PlayerTileX − 1
  → WindowOriginY = PlayerTileY − 1        // player at local (1,1) of 4×4 window
  → For DY = 0..3
      For DX = 0..3
        → DungeonX = WindowOriginX + DX
        → DungeonY = WindowOriginY + DY
        → Bounds check (inline — BP_MapManager has no IsTileInBounds):
             DungeonX >= 0 AND DungeonX < CurrentMap.Width
             AND DungeonY >= 0 AND DungeonY < CurrentMap.Height → bInBounds
        → GetTile(DungeonX, DungeonY) → Break SDungeonTile → Passable
        → Branch: (NOT bInBounds) OR (NOT Passable)?
            True  → For SY = 0..3, For SX = 0..3 → SetBlocked(DX×4+SX, DY×4+SY)
            False → (sub-block stays open)
  → RebuildTileInstances
```

**Notes:**
- Inline bounds check used — BP_MapManager has no IsTileInBounds function
- Width/Height read from MapManagerRef → Get CurrentMap → Break SDungeonMap
- WindowOrigin Subtract bottom pin must be `1` not `-1` (−(−1) = +1 — causes wrong coordinates)
- Verified at player (0,3): 40 instances — corridor shape correct, west edge clipping expected

### GB-030 Test Checklist
- [x] TileToIndex round-trips: (5,3)→53, (0,0)→0, (15,15)→255
- [x] IndexToTile round-trips: 53→(5,3), 255→(15,15), 0→(0,0)
- [x] IsTileInBounds: (16,0) false, (15,15) true, (−1,5) false
- [x] SetBlocked/IsBlocked/IsTraversable all verified
- [x] Occupancy: SetOccupant, IsOccupied, GetOccupant, ClearOccupant, overwrite all verified
- [x] GetAdjacentTiles: (0,0)→3, (8,8)→8, (15,15)→3, (0,8)→5
- [x] GetTilesInRange: (8,8,2)→25, (0,0,2)→9, (8,8,0)→1, (15,8,1)→6
- [x] GetTilesInSpellAoE: Radius(8,8,2)→25, Cone→0+print, None→0
- [x] SpawnGrid(16,16) → 256 instances
- [x] SetBlocked(0,0) + RebuildTileInstances → 255 instances, corner tile missing
- [x] GenerateGridFromDungeon(0,3) → 40 instances, corridor shape verified

### Deferred (post-VS)
- Edge-wall → sub-tile blocking (wall between two passable dungeon tiles)
- TileIndex → InstanceIndex highlight map — build in GB-042
- Facing-biased window placement
- Cone/Line AoE shapes — Phase 8 (GB-047/GB-051)

---

## CombatCameraActor
**Type:** Camera Actor (placed in level — not a Blueprint)  
**Location:** World Outliner → CombatCameraActor  
**Status:** ✅ GB-031 — complete

### Design Decisions
- **Camera Actor not Camera Component** — UE5 does not expose active camera selection for components on non-pawn actors. A dedicated Camera Actor is the correct UE-native approach for Set View Target With Blend
- **Perspective not Orthographic** — perspective at −90 pitch is visually identical to orthographic top-down with none of the rendering quirks. Can be swapped to true orthographic post-VS if needed
- **Instant cut (Blend Time=0)** — smooth blend tested and rejected in favour of clean instant cut matching Gold Box's original feel
- **C key test trigger** — removed after GB-033 wired. GB-033 StartCombat now owns the camera switch directly

### Transform
| Property | Value |
|---|---|
| Location X | −8400 |
| Location Y | 1600 |
| Location Z | 6000 |
| Rotation Pitch | −90 |
| Rotation Yaw | 0 |
| Rotation Roll | 0 |

### Level Blueprint Wiring (test trigger)
```
C Pressed
  → Get Player Controller (Index 0)
  → Set View Target With Blend
       New View Target ← CombatCameraActor (Outliner ref)
       Blend Time = 0.0
       Blend Func = EaseInOut
       Blend Exp = 2.0

C Released
  → Get Player Controller (Index 0)
  → Set View Target With Blend
       New View Target ← Get Player Pawn (Index 0)
       Blend Time = 0.0
       Blend Func = EaseInOut
       Blend Exp = 2.0
```

### GB-031 Test Checklist
- [x] Cut to combat view — full arena visible in frame
- [x] Cut back to exploration — returns to correct first-person eye height
- [x] Exploration camera functions normally on startup (no auto-activation conflict)

### Deferred (post-VS)
- Pan/zoom — GB-042
- Tile highlight — GB-042/GB-051
- Smooth blend option — trivial (change Blend Time from 0 to desired seconds)
- Camera centring on combatant positions — post GB-033 PlaceCombatants refinement

---

## WBP_ExplorationHUD
**Parent Class:** User Widget  
**Location:** Blueprints/UI/

### Layout
```
Canvas Panel (root, transparent)
  ├── AutomapPanel (Border, dark green)
  │     └── AutomapCanvas (Canvas Panel — stub, empty)
  │           └── Text_AutomapLabel ("AUTOMAP")
  ├── PartyStatusPanel (Border, dark warm black)
  │     └── PartyMemberList (Vertical Box)
  │           ├── CharSlot_0 (Horizontal Box)
  │           │     ├── CharName_0 (Text, Fill)
  │           │     ├── SizeBox_HPBar_0 → HPBar_0 (Progress Bar, 60px wide, 12px tall)
  │           │     ├── SizeBox_HPText_0 → HPText_0 (Text, 50px wide)
  │           │     └── SizeBox_Cond_0 → CondText_0 (Text, 36px wide)
  │           ├── CharSlot_1 ... CharSlot_5 (same structure)
  ├── TextLogPanel (Border, dark)
  │     └── MessageScrollBox (Scroll Box, hidden scrollbar)
  ├── CommandMenuPanel (Border, dark warm brown)
  │     └── CommandButtonRow (Horizontal Box)
  │           ├── Btn_Move, Btn_Search, Btn_Encamp
  │           ├── Btn_View, Btn_Items, Btn_Cast, Btn_Open
  ├── Divider_Vertical (Image, 2px wide)
  ├── Divider_AutomapBase (Image, 2px tall)
  ├── Divider_LogTop (Image, 2px tall)
  └── Divider_CmdTop (Image, 2px tall)
```

### Anchor/Position Reference (1920×1080)
| Widget | Anchor | Pos X | Pos Y | Size X | Size Y |
|---|---|---|---|---|---|
| AutomapPanel | Top-Right | -576 | 0 | 576 | 405 |
| PartyStatusPanel | Top-Right | -576 | 405 | 576 | 405 |
| TextLogPanel | Top-Left | 0 | 810 | 1920 | 162 |
| CommandMenuPanel | Bottom-Left | 0 | -108 | 1920 | 108 |
| Divider_Vertical | Top-Right | -576 | 0 | 2 | 810 |
| Divider_AutomapBase | Top-Right | -576 | 405 | 576 | 2 |
| Divider_LogTop | Top-Left | 0 | 810 | 1920 | 2 |
| Divider_CmdTop | Bottom-Left | 0 | -108 | 1920 | 2 |

### Variables
| Variable | Type | Notes |
|---|---|---|
| GameManagerRef | BP_GameManager (object ref) | Set in Event Construct |
| PartyManagerRef | BP_PartyManager (object ref) | Set in Event Construct |
| TextManagerRef | BP_TextManager (object ref) | Set in Event Construct |
| ConditionAbbreviations | Map ECondition → Text | Populated in InitialiseConditionAbbreviations |
| bAutomapVisible | Boolean | Default false, deferred to Phase 11 |
| MaxLogLines | Integer | Default 40 |
| CharNames | Text widget Array | [CharName_0 .. CharName_5] |
| HPBars | Progress Bar widget Array | [HPBar_0 .. HPBar_5] |
| HPTexts | Text widget Array | [HPText_0 .. HPText_5] |
| CondTexts | Text widget Array | [CondText_0 .. CondText_5] |

### Functions
| Function | Notes |
|---|---|
| InitialiseConditionAbbreviations | Populates ConditionAbbreviations map |
| InitWidgetArrays | Fills CharNames, HPBars, HPTexts, CondTexts arrays |
| RefreshCommandMenu(EGameState) | Shows/hides CommandMenuPanel based on state |
| RefreshPartyPanel | Loops 0–5, calls UpdateCharSlot or ClearCharSlot |
| UpdateCharSlot(Index, SCharacter) | Updates one party slot with character data. **GB-039: Conditions is now Array of ECondition — status display updated to check `Contains(Dead)` rather than a direct enum match. Multi-condition display (showing Poisoned, Restrained, etc simultaneously) deferred post-VS** |
| ClearCharSlot(Index) | Shows "---" and empty bars for empty slot |
| OnMessageReceived(Text, EMessageType) | Creates WBP_LogLine, adds to scroll box |
| ClearMessageLog | Clears MessageScrollBox children |

> **GB-039 bug fixed:** `OnMessagePosted_Event` previously had an erroneous `RefreshPartyPanel` call (added in a prior session) that caused an infinite loop whenever a message was posted while a dead character was in the party. Removed — party panel should only refresh via `OnPartyUpdated`, not on every message post.

### RefreshCommandMenu State Visibility
| EGameState | CommandMenuPanel |
|---|---|
| Exploration | Visible |
| Encounter | Hidden (deferred — currently still showing, fix in polish pass) |
| Combat | Hidden |
| Camp | Hidden |
| CharacterScreen | Hidden |
| MainMenu | Hidden |

> Note: All other panels (TextLog, PartyStatus, Automap) remain visible in all states.
> WBP_EncounterScreen draws on top of the viewport area at Z-Order 1 during Encounter state.

### Event Construct Flow
```
Event Construct
  → Cast to BP_GameManager → SET GameManagerRef
  → Get Party Manager from GameManagerRef → SET PartyManagerRef
  → Get Text Manager from GameManagerRef → SET TextManagerRef
  → InitialiseConditionAbbreviations
  → InitWidgetArrays
  → Assign OnGameStateUpdated (GameManagerRef) → OnGameStateUpdated_Event
        → OnGameStateUpdated_Event exec → RefreshCommandMenu(NewState)
  → Assign OnPartyUpdated (PartyManagerRef) → OnPartyUpdated_Event
        → OnPartyUpdated_Event exec → RefreshPartyPanel
  → Assign OnMessagePosted (TextManagerRef) → OnMessagePosted_Event
        → OnMessagePosted_Event → OnMessageReceived(Message, MessageType)
  → Assign OnClearLog (TextManagerRef) → OnClearLog_Event
        → OnClearLog_Event → ClearMessageLog
  → RefreshPartyPanel
  → RefreshCommandMenu(EGameState::Exploration)
```

---

## WBP_CombatHUD
**Parent Class:** User Widget
**Location:** Blueprints/UI/
**Status:** ✅ GB-004a — VS stub complete

### Layout
```
Canvas Panel (root)
  └── Horizontal Box (full-width bottom bar, Offset Left=0, Right=0, Y=-108, H=108)
        ├── Text_RoundCounter (Fill=1.0, Centre, Font 14, White) [Is Variable=true]
        ├── Text_TurnIndicator (Fill=2.0, Centre, Font 14, White) [Is Variable=true]
        ├── Button_Move (Fill=1.0) [Is Variable=true]
        │     └── Text "Move (M)"
        ├── Button_Attack (Fill=1.0) [Is Variable=true]
        │     └── Text "Attack (A)"
        ├── Button_EndTurn (Fill=1.0) [Is Variable=true]
        │     └── Text "End Turn (T)"
        └── Button_Flee (Fill=1.0) [Is Variable=true]
              └── Text "Flee (F)"
```

### Variables
| Variable | Type | Notes |
|---|---|---|
| GameManagerRef | BP_GameManager (object ref) | Set in Initialise function |

### Functions

#### Initialise
```
Initialise (regular function)
  → Get Game Instance → Cast to BP_GameManager → SET GameManagerRef
```

### Event Graph
```
OnClicked (Button_Move)
  → Get GameManagerRef → Get CombatManagerRef → EnterMoveMode

OnClicked (Button_Attack)
  → Get GameManagerRef → Get CombatManagerRef → EnterTargetSelectMode

OnClicked (Button_EndTurn)
  → Get GameManagerRef → Get CombatManagerRef → EndPlayerTurn

OnClicked (Button_Flee)
  → Get GameManagerRef → Get TextManagerRef → PostMessage("Flee not yet implemented", Combat)
```

### Show/Hide Pattern
Shown and hidden directly by `BP_CombatManager`:
- `StartCombat` → `CombatHUDRef.Add to Viewport (Z=1)`
- `EndCombat` → `CombatHUDRef.Remove from Parent`

### Notes
- Sits at same position as ExplorationHUD CommandMenuPanel (which hides during Combat state)
- `Text_RoundCounter` and `Text_TurnIndicator` are placeholders — wiring to BP_CombatManager deferred post-VS
- OnGameStateUpdated binding pattern does NOT work for widgets not yet in viewport — direct manager ref pattern used instead
- `CombatHUDRef` stored on BP_CombatManager, set from Level Blueprint after widget creation

### Deferred (post-VS)
- Wire Text_RoundCounter to CurrentRound from BP_CombatManager
- Wire Text_TurnIndicator to current combatant name from BP_CombatManager
- Cast spell button
- Use item button
- Guard button

---
**Parent Class:** User Widget  
**Location:** Blueprints/UI/

### Layout
```
Canvas Panel (root)
  ├── Border_Background (dark warm colour, full stretch anchor)
  └── VerticalBox (full stretch anchor, Offset Left=0, Top=0, Right=576, Bottom=270)
        ├── Image_Portrait (Fill=2.0, Is Variable=true — portrait display)
        ├── Text_EncounterText (Fill=1.0, Is Variable=true, Auto Wrap, Centre, Font 16)
        └── VerticalBox_Choices (Fill=1.0, Is Variable=true — dynamic buttons added here)
```

### Anchor Notes
- VerticalBox and Border_Background both use full-stretch anchor (all four corners)
- Offset Left=0, Offset Top=0, Offset Right=576, Offset Bottom=270
- Right=576 leaves sidebar (automap + party panel) visible
- Bottom=270 leaves text log and command menu visible
- Scales proportionally with all resolution changes

### Variables
| Variable | Type | Notes |
|---|---|---|
| EncounterManagerRef | BP_EncounterManager (object ref) | Set in Initialise |
| GameManagerRef | BP_GameManager (object ref) | Set in Initialise |
| CurrentChoices | Array of S_EncounterChoice | Stores choices for current encounter |

### Functions

#### Initialise
```
Initialise
  → Get Game Instance → Cast to BP_GameManager → SET GameManagerRef
  → GameManagerRef → Get Encounter Manager → SET EncounterManagerRef
  → [OnGameStateUpdated_Event wired in Event Graph — fires on state change]
```

#### SetEncounterData
**Inputs:** IntroText (Text), Choices (Array of S_EncounterChoice)
```
SetEncounterData(IntroText, Choices)
  → Set Text (Text_EncounterText, IntroText)
  → SET CurrentChoices = Choices
  → PopulateChoices(Choices)
```

#### ClearChoices
```
ClearChoices
  → Clear Children (VerticalBox_Choices)
```

#### PopulateChoices
**Inputs:** Choices (Array of S_EncounterChoice)
```
PopulateChoices(Choices)
  → ClearChoices
  → For Each Loop (Choices)
      Loop Body
        → Create Widget (WBP_ChoiceButton)
        → SET ChoiceIndex = Array Index (on button)
        → SET EncounterManagerRef (on button — from this widget's EncounterManagerRef)
        → SetLabel(ChoiceLabel from Break S_EncounterChoice)
        → Add Child (VerticalBox_Choices, button widget)
```

#### ShowPortrait
**Inputs:** MonsterID (Integer)
```
ShowPortrait(MonsterID)
  → Get Data Table Row (DT_MonsterPortraits, Conv_IntToString(MonsterID) → Conv_StringToName)
      Row Found
        → Break S_MonsterPortrait → PortraitTexture
        → Set Brush from Texture (Image_Portrait, PortraitTexture, MatchSize=false)
      Row Not Found → (exit — placeholder remains)
```

#### HidePortrait
```
HidePortrait
  → Set Color and Opacity (Image_Portrait)
       └── InColorAndOpacity = (R:0, G:0, B:0, A:0) — fully transparent
```

### Event Graph
```
OnGameStateUpdated_Event (Custom Event, Input: NewState EGameState)
  → Branch: NewState == Encounter?
      True  → Add to Viewport (Self, Z-Order 1)
      False → Remove from Parent
```

> Note: Event Construct does NOT fire until widget is added to viewport.
> Initialise is called explicitly from the Level Blueprint after widget creation.
> Do NOT rely on Event Construct for setup in this widget.

---

## WBP_ChoiceButton
**Parent Class:** User Widget  
**Location:** Blueprints/UI/

### Layout
```
Button_Choice (Is Variable=true)
  └── Text_ChoiceLabel (Is Variable=true, Centre, Font 14)
```

### Variables
| Variable | Type | Notes |
|---|---|---|
| ChoiceIndex | Integer | Which choice this button represents — set from PopulateChoices |

> Note: EncounterManagerRef is NOT stored as a variable on this widget.
> It is retrieved fresh from Game Instance at click time to avoid stale references.

### Functions

#### SetLabel
**Inputs:** LabelText (Text)
```
SetLabel(LabelText)
  → Set Text (Text_ChoiceLabel, LabelText)
```

### Event Graph
```
OnClicked (Button_Choice)
  → Get Game Instance → Cast to BP_GameManager → Get Encounter Manager
  → ResolveChoice(ChoiceIndex)
```

---

## WBP_LogLine
**Parent Class:** User Widget  
**Location:** Blueprints/UI/

### Layout
```
Size Box (root, auto height, fill width)
  └── MessageText (Text, Is Variable=true, Auto Wrap Text=On, Font 12)
```

---

## Level Blueprint (TestDungeon)
### Event Begin Play Flow
```
Event Begin Play
  → Create Widget (WBP_ExplorationHUD, OwningPlayer = GetPlayerController(0))
  → Add to Viewport (Z-Order 0)
  → Print "Viewport added"
  → Create Widget (WBP_EncounterScreen, OwningPlayer = GetPlayerController(0))
      [NOT added to viewport here — widget adds itself via OnGameStateUpdated]
  → Get Game Instance → Cast to BP_GameManager → Get Encounter Manager
  → SET EncounterManager.EncounterScreenRef = new WBP_EncounterScreen widget
  → WBP_EncounterScreen.Initialise
  → [existing Delay → Cast → TextManager → PostMessage chain continues]
```

---

## BP_CombatManager
**Parent Class:** Actor  
**Location:** Blueprints/Combat/  
**Status:** ✅ GB-033 — complete (VS scope). ✅ GB-039 — Dead and Restrained conditions wired in. ✅ GB-040 — morale system complete (ResolveMorale, Shaken/Fleeing transitions).

### Variables
| Variable | Type | Notes |
|---|---|---|
| GameManagerRef | BP_GameManager (object ref) | Set by BP_GameManager on spawn |
| TextManagerRef | BP_TextManager (object ref) | Set by BP_GameManager on spawn |
| PartyManagerRef | BP_PartyManager (object ref) | Set by BP_GameManager on spawn |
| CombatGridRef | BP_CombatGrid (object ref) | Set from Level Blueprint on BeginPlay |
| CombatCameraRef | Camera Actor (object ref) | Set from Level Blueprint on BeginPlay |
| EnemyAIRef | BP_EnemyAI (object ref) | Set from Level Blueprint on BeginPlay |
| XPManagerRef | BP_XPManager (object ref) | Set from Level Blueprint on BeginPlay |
| CurrentMonsterGroupID | Integer | Set in StartCombat from its MonsterGroupID input. Read by EndCombat to award XP after combat state begins clearing |
| Combatants | Array of SCombatant | All active combatants this combat |
| InitiativeOrder | Array of Integer | CombatantIDs sorted by initiative roll |
| CurrentTurnIndex | Integer | Which combatant is currently acting |
| CurrentRound | Integer | Round counter |
| bCombatActive | Boolean | True while combat is running |

### Functions

#### StartCombat
**Inputs:** MonsterGroupID (Integer)

```
StartCombat(MonsterGroupID)
  → Branch: bCombatActive? True → Print "already active" (exit)
  → SET bCombatActive = true
  → GameManagerRef.ChangeGameState(Combat)
  → TextManagerRef.PostMessage("Combat begins!", Combat)
  → Get Data Table Row (DT_Monsters, MonsterGroupID as Name)
      Row Not Found → Print "Monster not found" (exit)
      Row Found
        → GenerateGridFromDungeon(CurrentPlayerTileX, CurrentPlayerTileY)
        → BuildCombatants(SourceMonster)
        → BuildInitiativeOrder
        → FindStartCombatant
        → SpawnCombatantMarkers
        → TransitionToCombat
```

#### BuildCombatants
**Inputs:** SourceMonster (SMonster)
```
BuildCombatants(SourceMonster)
  → FindNearestTraversableTile(PreferredX=14, PreferredY=8) → MonsterSpawnPos
       (includes IsOccupied check — won't place on occupied tile)
  → SetOccupant(FoundX, FoundY, CombatantID=1)
  → Make SCombatant (monster):
       CombatantID=1, IsMonster=true, IsPlayerControlled=false
       Stats from SourceMonster, GridPosition=MonsterSpawnPos
  → Combatants ADD monster combatant
  → PartyManagerRef.GetLivingMembers → For Each Loop
       → FindNearestTraversableTile(PreferredX=0, PreferredY=4+ArrayIndex)
            (back row bottom of arena, spread horizontally in party order)
       → SetOccupant(FoundX, FoundY, CombatantID=1000+ArrayIndex)
       → Make SCombatant (party member):
            CombatantID = 1000 + Array Index
            CharacterID = SCharacter.CharacterID  ← **GB-039: new field, copied from source SCharacter to enable party-panel death sync**
            IsMonster=false, IsPlayerControlled=true
            Stats from SCharacter, GridPosition=PartySpawnPos
       → Combatants ADD party combatant
```

### BuildCombatants Attack Wiring *(GB-042)*
  ```
  BuildCombatants(SourceMonster, MonsterCount)
    -> [Monster ForLoop]
         -> Break S_Monster -> AttackRange -> S_Combatant.AttackRange (on MakeStruct)
         -> MovementRate -> MovementRange (existing)
    -> [Player ForEachLoop]
         -> MakeStruct S_Combatant: AttackRange=1 (hardcoded, same as MovementRange=6)
  ```

  ### GB-033/034 Notes
- MonsterGroupID maps directly to DT_Monsters row for VS — full MonsterGroup table lookup comes post-VS
- Party CombatantIDs start at 1000 to avoid collision with monster IDs
- CombatGridRef and CombatCameraRef set from Level Blueprint after SpawnManagers completes
- CurrentPlayerTileX/Y read from GameManagerRef — updated by BP_ExplorerPawn.CompleteMovement
- C key test trigger removed from Level Blueprint — StartCombat owns camera switch directly
- ResolveChoice Combat branch updated: HidePortrait + Remove from Parent before StartCombat fires
- GenerateGridFromDungeon runs BEFORE BuildCombatants — ensures traversable check uses correct corridor shape
- FindNearestTraversableTile includes IsOccupied check — prevents two combatants on same tile
- SetOccupant called after each placement — marks tile as taken for subsequent FindNearestTraversableTile calls
- Party spawns at bottom of arena (PreferredX=0, PreferredY=4+Index) in party order left to right
- Monster spawns at top of arena (PreferredX=14, PreferredY=8), FindNearestTraversableTile finds closest valid tile
- Return Node added to OnActionComplete after StartEnemyTurn — prevents double OnActionComplete execution
- IsMovementLocked used for combat input lock (CanMove is used for tile traversability in movement checks)
- MovementRange=3 for VS (rules redesign post-VS — placeholder value)

#### EnterMoveMode
```
EnterMoveMode
  → Branch: bPlayerTurnActive? False → dead-end
  → SET bMoveMode = true
  → For Each Combatants → find CurrentCombatantID
       → ClearHighlights
       → GetValidMoveTiles(CurrentTileX, CurrentTileY, RemainingMovement)
       → HighlightTiles(ValidTiles, Cyan)
       → PostMessage("Select a direction to move (WASD)", Combat)
```

#### ExecuteCombatantMove
**Inputs:** DeltaX (Integer), DeltaY (Integer)
```
ExecuteCombatantMove(DeltaX, DeltaY)
  → Branch: bPlayerTurnActive AND bMoveMode? False → dead-end
  → Find current combatant → get CurrentTileX, CurrentTileY
  → TargetX = CurrentTileX + DeltaX, TargetY = CurrentTileY + DeltaY
  → Branch: IsTraversable(TargetX, TargetY)? False → PostMessage "Can't move there"
  → Branch: IsOccupied(TargetX, TargetY)? True → PostMessage "Tile occupied"
  → Branch: TilesMovedThisTurn < MovementRange? False → PostMessage "No movement remaining"
  → ClearOccupant(CurrentTileX, CurrentTileY)
  → SetOccupant(TargetX, TargetY, CurrentCombatantID)
  → UpdateCombatantGridPosition(CurrentCombatantID, Vector2D(TargetX, TargetY))
  → MoveMarkerToTile(CurrentCombatantID, Vector2D(TargetX, TargetY))
  → RefreshMoveHighlights
```

#### UpdateCombatantGridPosition *(shared — used by player and enemy movement)*
**Inputs:** CombatantID (Integer), NewPosition (Vector2D)
```
UpdateCombatantGridPosition(CombatantID, NewPosition)
  → For Each Combatants
       → Break SCombatant → CombatantID
       → Branch: CombatantID == input CombatantID?
            False → skip
            True
              → Make SCombatant (all fields from Break, GridPosition = NewPosition)
              → Set Array Elem (Combatants, Array Index, new SCombatant)
```

#### MoveMarkerToTile *(shared — used by player and enemy movement)*
**Inputs:** CombatantID (Integer), NewPosition (Vector2D)
```
MoveMarkerToTile(CombatantID, NewPosition)
  → For Each SpawnedMarkers
       → Cast to BP_CombatantMarker
       → Get CombatantID → == input CombatantID?
            False → skip
            True
              → CombatGridRef.TileToWorld(NewPosition.X, NewPosition.Y) → Break Vector
              → Make Vector (X, Y, Z=50)
              → Set Actor Location (As BP_CombatantMarker, new Vector)
```

#### ApplyDamage *(shared — used by player and enemy attacks)*
**Inputs:** CombatantID (Integer), Damage (Integer)
```
ApplyDamage(CombatantID, Damage)
  → For Each Combatants
       → Break SCombatant → CombatantID, CurrentHP
       → Branch: CombatantID == input CombatantID?
            False → skip
            True
              → NewHP = CurrentHP - Damage
              → Make SCombatant (all fields from Break, CurrentHP = NewHP)
              → Set Array Elem (Combatants, Array Index, new SCombatant)
```

#### ApplyCondition *(shared — GB-039)*
**Inputs:** CombatantID (Integer), NewCondition (ECondition)
```
ApplyCondition(CombatantID, NewCondition)
  → For Each Combatants
       → Break SCombatant → CombatantID, Conditions
       → Branch: CombatantID == input CombatantID?
            False → Continue
            True
              → Branch: Conditions Contains NewCondition?
                   True → Continue (already has it, no duplicates)
                   False
                     → Promote Conditions to local UpdatedConditions
                     → Get UpdatedConditions → Add (NewCondition)
                     → Make SCombatant (all fields from Break, Conditions = fresh Get UpdatedConditions)
                     → Set Array Elem (Combatants, Array Index, new SCombatant)
```

#### RemoveCondition *(shared — GB-039)*
**Inputs:** CombatantID (Integer), ConditionToRemove (ECondition)
```
RemoveCondition(CombatantID, ConditionToRemove)
  → Same shape as ApplyCondition but calls Remove Item on UpdatedConditions instead of Add
```

#### HasCondition *(shared — GB-039)*
**Inputs:** CombatantID (Integer), ConditionToCheck (ECondition)
**Outputs:** bHasCondition (Boolean)
```
HasCondition(CombatantID, ConditionToCheck)
  → For Each Combatants
       → Break SCombatant → CombatantID, Conditions
       → Branch: CombatantID == input CombatantID?
            False → Continue
            True → Return Conditions.Contains(ConditionToCheck)
  → For Each Completed → Return false (fallback — no combatant matched)
```

#### SetMarkerDowned *(GB-039)*
**Inputs:** CombatantID (Integer)
```
SetMarkerDowned(CombatantID)
  → Same lookup shape as RemoveMarkerForCombatant but instead of Destroy Actor:
  → Set Actor Scale 3D (0.5, 0.5, 0.5) on the found marker
  → Visual placeholder — proper downed treatment (material tint, icon) deferred to polish pass
```


```
RefreshMoveHighlights
  → ClearHighlights
  → Find current combatant → get position and remaining movement
  → GetValidMoveTiles(CurrentTileX, CurrentTileY, RemainingMovement)
  → HighlightTiles(ValidTiles, Cyan)
```

#### EndPlayerTurn
```
EndPlayerTurn
  → Branch: bPlayerTurnActive? False → dead-end
  → SET bPlayerTurnActive = false
  → SET bMoveMode = false
  → SET bSelectingTarget = false
  → ClearHighlights
  → OnActionComplete
```

#### CheckVictory
**Outputs:** bVictory (Boolean)
```
CheckVictory
  → For Each Combatants
       → Break SCombatant → IsMonster, CurrentHP
       → Branch: IsMonster AND CurrentHP > 0?
            True → SET bAllEnemiesDead = false
  → Return bAllEnemiesDead
```

#### CheckDefeat *(stub — GB-039, proper defeat handling deferred to GB-046)*
```
CheckDefeat (called from OnActionComplete before turn dispatch)
  → bAllPartyDead = true (default assumption)
  → For Each Combatants
       → Break SCombatant → IsMonster, CombatantID
       → Branch: IsMonster == false?
            False → Continue (skip monsters)
            True → HasCondition(CombatantID, Dead)?
                   False → bAllPartyDead = false (found a living party member)
  → Branch: bAllPartyDead?
       True → PostMessage("Defeat — all party members have fallen.", Combat) → EndCombat
            ⚠ STUB — EndCombat used as placeholder. Replace with proper game-over/defeat screen in GB-046
  ```

  #### ExecutePlayerAttack Range Gate *(GB-042)*
  ```
  ExecutePlayerAttack(TargetCombatantID)
    -> [attacker/defender lookup -- unchanged]
    -> Break AttackerCombatant -> GridPosition, AttackRange
    -> Break DefenderCombatant -> GridPosition
    -> GetCombatDistance -> Distance
    -> Branch: Distance <= AttackRange?
         True -> [existing HasCondition / ResolveAttack flow]
         False -> PostMessage("[name] is out of range", Combat) -> OnActionComplete
  ```
       False → continue into normal turn dispatch
```

  #### ResolveMorale *(GB-040)*
  **Inputs:** GroupID (Integer)  Triggers when a monster dies. Filters all monsters in the given GroupID that are alive and not already Fleeing, finds the highest MoraleRating, rolls a d100 (PercentileRoll), and transitions state on failure (Normal->Shaken, Shaken->Fleeing). Posts a message to the combat log.  **Local variables:** AffectedCombatants (TArray<S_Combatant>), HighestMorale (int), RollResult (int), NewState (EMoraleState), bAnyShaken (bool).  **Call sites:**
  - ExecutePlayerAttack: after RemoveMarkerForCombatant -> ResolveMorale(DefenderCombatant.GroupID) -> CheckVictory  - ExecuteEnemyAttack: after ApplyCondition(Dead) -> ResolveMorale(LocalTarget.GroupID)

  #### EndCombat
```
EndCombat
  → XPManagerRef.AwardCombatXP(CurrentMonsterGroupID) → TotalXP
  → XPManagerRef.DistributeXP(TotalXP)
  → PostMessage("Victory! All enemies defeated.", Combat)
  → SET bCombatActive, bSelectingTarget, bMoveMode = false
  → SET CurrentRound, CurrentTurnIndex = 0
  → Clear Combatants, InitiativeOrder, SpawnedMarkers arrays
  → Get Player Pawn → Get Controller → Cast to PlayerController
       → Set View Target With Blend (Get Player Pawn, Blend=0)
  → Cast to BP_ExplorerPawn → SET IsMovementLocked = false
  → GameManagerRef.ChangeGameState(Exploration)
```

#### OnActionComplete *(updated GB-039)*
```
OnActionComplete
  → SET CurrentTurnIndex = CurrentTurnIndex + 1
  → CheckDefeat (stub) — if all party dead, EndCombat and return
  → Branch: CurrentTurnIndex >= InitiativeOrder.Length?
       True → SET CurrentTurnIndex = 0, SET CurrentRound = CurrentRound + 1
              → PostMessage (new round)
       False → (continue)
  → FIND NEXT COMBATANT: InitiativeOrder[CurrentTurnIndex] → SET CombatantID
  → DETERMINE PLAYER OR MONSTER: For Each Combatants
       → Break SCombatant → CombatantID, IsPlayerControlled
       → Branch: CombatantID == target CombatantID? (ID-match gate — GB-039 bug fix)
            False → Continue
            True → SET PlayerOrMonsterID, SET IsPlayerControlled
  → CHECK CONDITIONS: HasCondition(CombatantID, Dead) OR HasCondition(CombatantID, Restrained)
       True (should skip) → OnActionComplete (recurse to next combatant)
       False (normal turn) → Branch: IsPlayerControlled?
            True → StartPlayerTurn(PlayerOrMonsterID)
            False → StartEnemyTurn(PlayerOrMonsterID)
```

### Deferred (post-VS)
- Monster spawn zone — currently finds nearest to (14,8). Proper zone logic post-VS
- Full defeat handling / game over screen — GB-046 (CheckDefeat stub currently just calls EndCombat)
- Loot drop — GB-055
- Full MonsterGroup table lookup — Phase 10
- Adjacency check for attacks — player and enemy can currently attack from any distance
- Attack range system (melee adjacency, ranged line-of-sight + max range) — Phase 6
- Movement range rules redesign — currently hardcoded to 3
- Party HP not updating in HUD when enemy attacks mid-combat — ApplyDamage updates SCombatant only, SCharacter not synced until death. Full live-HP-sync is a new ticket (GB-039a or similar)
- Downed marker visual polish (material tint, icon) — SetMarkerDowned currently uses 0.5 scale only

---

## BP_EnemyAI
**Parent Class:** Actor
**Location:** Blueprints/Combat/
**Status:** ✅ GB-045 — VS scope complete. ✅ GB-039 — death check and Restrained auto-hit added. ✅ GB-040 — fleeing branch added to RunEnemyTurn.

### Variables
| Variable | Type | Notes |
|---|---|---|
| CombatManagerRef | BP_CombatManager (object ref) | Set by StartEnemyTurn before RunEnemyTurn is called |
| PartyManagerRef | BP_PartyManager (object ref) | **GB-039 — added.** Set by StartEnemyTurn alongside CombatManagerRef, used by ExecuteEnemyAttack to call ApplyDeathToCharacter on player death |

### Functions

#### RunEnemyTurn
**Inputs:** ActiveCombatant (SCombatant)
```
RunEnemyTurn(ActiveCombatant)
    -> Break SCombatant -> MoraleState
    -> Branch: MoraleState == Fleeing?
         True -> CombatManagerRef.OnActionComplete -> Return (GB-040: skip turn entirely)
         False
  → FindNearestPartyMember(ActiveCombatant) → NearestTarget, bFound
  → Branch: bFound?
       False → CombatManagerRef.OnActionComplete (no targets, skip turn)
       True
         → MoveOneStepToward(ActiveCombatant, NearestTarget)
         → ExecuteEnemyAttack(ActiveCombatant, NearestTarget)
         → CombatManagerRef.OnActionComplete
```

#### FindNearestPartyMember
**Inputs:** ActiveCombatant (SCombatant)
**Outputs:** NearestTarget (SCombatant), bFound (Boolean)
```
FindNearestPartyMember(ActiveCombatant)
  → Local: BestDist=999, BestTarget (SCombatant), bFound=false
  → For Each CombatManagerRef.Combatants
       → Break SCombatant → IsMonster, CurrentHP, GridX, GridY
       → Branch: IsMonster? True → skip
       → Branch: CurrentHP <= 0? True → skip
       → Dist = Max(Abs(GridX - ActiveCombatant.GridX), Abs(GridY - ActiveCombatant.GridY))
       → Branch: Dist < BestDist?
            True → SET BestDist=Dist, BestTarget=Element, bFound=true
  → Return BestTarget, bFound
```

#### MoveOneStepToward
**Inputs:** ActiveCombatant (SCombatant), Target (SCombatant)
```
MoveOneStepToward(ActiveCombatant, Target)
  → Local: BestDist=999, BestTile (Vector2D)
  → CombatManagerRef.CombatGridRef.GetAdjacentTiles(ActiveCombatant.GridX, ActiveCombatant.GridY)
  → For Each adjacent tiles
       → IsTraversable(Tile.X, Tile.Y)? False → skip
       → IsOccupied(Tile.X, Tile.Y)? True → skip
       → Dist = Max(Abs(TileX - Target.GridX), Abs(TileY - Target.GridY))
       → Branch: Dist < BestDist? True → SET BestDist=Dist, BestTile=Tile
  → On Completed:
       → ClearOccupant(ActiveCombatant.GridX, ActiveCombatant.GridY)
       → SetOccupant(BestTile.X, BestTile.Y, ActiveCombatant.CombatantID)
       → CombatManagerRef.UpdateCombatantGridPosition(ActiveCombatant.CombatantID, BestTile)
       → CombatManagerRef.MoveMarkerToTile(ActiveCombatant.CombatantID, BestTile)
```

#### ExecuteEnemyAttack
**Inputs:** ActiveCombatant (SCombatant), Target (SCombatant)
```
ExecuteEnemyAttack(ActiveCombatant, Target)
  → BPL_RulesLibrary.ResolveAttack(ActiveCombatant, Target) → SAttackResult
  → Break SAttackResult → bHit, Damage
  → Branch: bHit?
       False → PostMessage("Goblin misses!", Combat)
       True
         → CombatManagerRef.ApplyDamage(Target.CombatantID, Damage)
         → PostMessage("Goblin hits [Target.CombatantName]!", Combat)
         → CombatManagerRef.CheckVictory → Branch
              True → CombatManagerRef.EndCombat
              False → (return — OnActionComplete called from RunEnemyTurn)
```

### Notes
- CombatManagerRef set fresh each turn by StartEnemyTurn — not set at spawn time
- Chebyshev distance used throughout: Max(Abs(ΔX), Abs(ΔY))
- MoveOneStepToward checks both IsTraversable AND IsOccupied before selecting a tile
- Enemy moves one tile per turn then attacks — no action budget for VS
- Hit messages use hardcoded "Goblin" name for VS — post-VS replace with ActiveCombatant.CombatantName
- Attack range not enforced for VS — adjacency check deferred to Phase 6

### Deferred (post-VS)
- Incapacitated/held target priority
- AoE opportunity check
- Morale flee logic — ✅ GB-040 core complete (ResolveMorale, state transitions, turn-skip on Fleeing). Remaining: flee-to-map-edge + removal from combat
- Intelligent monster spellcasting
- ECombatAction_AI enum dispatch (ChooseAction function) — full priority switch in Phase 6 GB-045
- Dynamic hit/miss message using monster name from SCombatant

---

## BP_XPManager
**Parent Class:** Actor
**Location:** Blueprints/Managers/
**Status:** ✅ GB-041 — VS scope complete. ✅ **GB-079 — fully migrated.** Vigor/GetVigorBonus, StrikeNumber, and XP thresholds (via GetXPThreshold formula) all on the new system. Only the saving throws system (separate from this Blueprint) remains legacy project-wide.

### Variables
None — pure logic actor, same pattern as BP_EnemyAI. All required refs (PartyManagerRef) are fetched on demand via Get Game Instance → Cast to BP_GameManager rather than cached.

### Functions

#### AwardCombatXP
**Inputs:** MonsterGroupID (Integer)
**Outputs:** TotalXP (Integer)
```
AwardCombatXP(MonsterGroupID)
  → Get Data Table Row (DT_Monsters, MonsterGroupID as Name)
       Row Not Found → Return TotalXP=0
       Row Found
         → Break SMonster → XPValue
         → Return TotalXP=XPValue
```
VS scope is a single-monster lookup — no group multiplier. Extend here when MonsterGroup tables arrive (Phase 10).

#### DistributeXP
**Inputs:** TotalXP (Integer)
```
DistributeXP(TotalXP)
  → Get Game Instance → Cast to BP_GameManager → Get PartyManagerRef → GetLivingMembers
  → Array Length → LivingCount
  → Branch: LivingCount > 0?
       False → (no living members, exit)
       True
         → XPPerCharacter = TotalXP / LivingCount   (integer division — remainder dropped for VS)
         → For Each (GetLivingMembers result)
              → Break SCharacter → CharacterID
              → PartyManagerRef.AddXPToCharacter(CharacterID, XPPerCharacter)
              → XPManagerRef.CheckLevelUp(CharacterID)
```

#### CheckLevelUp
**Inputs:** CharacterID (Integer)
```
CheckLevelUp(CharacterID)
  → Get Game Instance → Cast to BP_GameManager → Get PartyManagerRef → Get PartyMembers (full array, not GetLivingMembers — checking one specific ID)
  → For Each PartyMembers
       → Break SCharacter → CharacterID, Class, Level, XP, Vigor
       → Branch: CharacterID == input CharacterID?
            False → skip
            True
              → NextLevel = Level + 1
              → ClassString = Enum to String(Class)
              → RowName = Append(ClassString, "_", ToString(NextLevel))  → "Warden_2" etc (class enum renamed under GB-079; DT_LevelProgression's 20 rows renamed to match)
              → Get Data Table Row (DT_LevelProgression, RowName as Name)
                   Row Not Found → (already max level, exit)
                   Row Found
                     → Break SLevelProgression → StrikeNumber (✅ migrated to percentage), HitDiceType (XPRequired output now unused — see below)
                     → XPThreshold = GetXPThreshold(Class, NextLevel)  ✅ GB-079 — formula-based, replaces reading XPRequired from the table
                     → Branch: XP >= XPThreshold?
                          False → (not enough XP yet, exit)
                          True
                            → HPRoll = RollDice(1, HitDiceType)
                            → HPBonus = GetVigorBonus(Vigor)
                            → HPGained = HPRoll + HPBonus
                            → XPManagerRef.ApplyLevelUp(CharacterID, NextLevel, StrikeNumber, HPGained)
```

#### ApplyLevelUp
**Inputs:** CharacterID (Integer), NewLevel (Integer), NewStrikeNumber (Integer), HPGained (Integer)
```
ApplyLevelUp(CharacterID, NewLevel, NewStrikeNumber, HPGained)
  → Get Game Instance → Cast to BP_GameManager → Get PartyManagerRef → Get PartyMembers
  → For Each PartyMembers
       → Break SCharacter → CharacterID, MaxHP, CurrentHP
       → Branch: CharacterID == input CharacterID?
            False → skip
            True
              → NewMaxHP = MaxHP + HPGained
              → NewCurrentHP = CurrentHP + HPGained   (level-up heals by the same amount)
              → Make SCharacter (all fields from Break, Level=NewLevel, StrikeNumber=NewStrikeNumber, MaxHP=NewMaxHP, CurrentHP=NewCurrentHP)
              → Set Array Elem (PartyMembers, Array Index, new SCharacter)
              → PostMessage("[Name] has reached level [NewLevel]!", Combat/System)
```

### Notes
- Matching always uses CharacterID, never Name (Name is not guaranteed unique)
- Row Name format for DT_LevelProgression lookups: "ClassName_Level" (e.g. "Fighter_2") — still uses the original class enum names since ECharacterClass hasn't been renamed yet under GB-079
- Integer division in DistributeXP drops any remainder — acceptable for VS scope, revisit if exact XP accounting matters later
- Level-up increases both MaxHP and CurrentHP by the same roll — character is effectively healed by the amount gained
- BP_CombatManager.CurrentMonsterGroupID (set in StartCombat) is what EndCombat passes into AwardCombatXP — without this, MonsterGroupID is unavailable by the time EndCombat runs since Combatants begins clearing
- Verified: single goblin (XPValue=20) → 5 XP per character across 4 living members; forced high-XP test (temporary DT_Monsters XPValue=10000) confirmed all four characters correctly leveled up with updated Level/THAC0/MaxHP/CurrentHP and posted messages

### Deferred (post-VS)
- MonsterGroup XP multiplier (currently single monster only)
- Saving throws and spell slots do not recalculate on level-up (THAC0 and HP only, per build order)
- Exact XP remainder handling (currently dropped via integer division)
- ComputeTHAC0 standalone library wrapper (currently inlined as a direct DT_LevelProgression lookup inside CheckLevelUp)

---

## BP_TileHighlight
**Parent Class:** Actor  
**Location:** Blueprints/Combat/  
**Status:** ✅ GB-042 — complete

### Components
| Component | Name | Notes |
|---|---|---|
| Static Mesh | HighlightMesh | Engine Plane, Location Z=5 (above tile), Scale 1.8×1.8 |

### Variables
| Variable | Type | Notes |
|---|---|---|
| DynamicMaterial | Material Instance Dynamic | Created at runtime in Initialise |

### Functions

#### Initialise
**Inputs:** Colour (Linear Color)
```
Initialise(Colour)
  → Get HighlightMesh → Create Dynamic Material Instance (M_CombatTile)
  → SET DynamicMaterial
  → Get DynamicMaterial → Set Vector Parameter Value
       Parameter Name ← "EmissiveColour"
       Value ← Colour
```

### Notes
- Uses **M_CombatTile** material with **EmissiveColour** Vector Parameter
- Spawned by BP_CombatGrid.HighlightTiles, destroyed by ClearHighlights
- Cyan (0, 0.8, 1, 1) for valid move tiles
- Deferred: different colours for attack range, AoE preview (GB-051)

---

## BP_RulesTestSuite ⚠ *(stale against the migrated GB-079 math — decided to delete and recreate the affected test tables fresh rather than patch them, at a later date. DT_THAC0Tests, DT_StrengthBonusTests, DT_AttackResolutionTests, DT_XPThresholdTests, DT_SavingThrowTests, and DT_LevelCapTests are all candidates given how much has changed — not yet actioned)*
**Parent Class:** Actor  
**Location:** Blueprints/Testing/

### Variables
| Variable | Type | Default |
|---|---|---|
| PassCount | Integer | 0 |
| FailCount | Integer | 0 |

### Functions
| Function | Notes |
|---|---|
| AssertEqual_Float(TestName, Expected, Actual, Tolerance) | Increments PassCount or FailCount, prints result |
| AssertEqual_Integer(TestName, Expected, Actual) | Integer comparison |
| AssertEqual_Bool(TestName, Expected, Actual) | Boolean comparison |
| RunDataTableTests(TestTable, TestCategory) | Generic data table test runner — loops rows, calls appropriate assert |
| RunHPFractionTests | Calls RunDataTableTests(DT_HPFractionTests, "HP Fraction") |
| RunTHAC0Tests | Calls RunDataTableTests(DT_THAC0Tests, "THAC0") |
| PrintSummary | Prints "Tests Complete — Passed: X Failed: Y" |

### BeginPlay Flow
```
Event BeginPlay
  → RunHPFractionTests   (8 tests)
  → RunTHAC0Tests        (14 tests)
  → PrintSummary
```

**Current result: 30 tests passing**

---

## Phase 1-3 Refactor: ForEach to FindByID (Complete)

All ForEach-loop-for-ID-match patterns across BP_CombatManager have been replaced with dedicated lookup functions:

**FindCombatantByID** (Built)
Inputs: CombatantID (Integer)
Outputs: OutCombatant (SCombatant), OutIndex (Integer), bFound (Boolean)
Location: BP_CombatManager function

Replaces ForEach-Combatants-Break-Compare patterns in: ExecutePlayerAttack, ApplyDamage, ApplyCondition, RemoveCondition, HasCondition, UpdateCombatantGridPosition, ExecuteCombatantMove, EnterMoveMode, EndPlayerTurn, CheckVictory, CheckDefeat, FindStartCombatant, FindActiveCombatant, OnActionComplete, StartPlayerTurn, StartEnemyTurn.

**FindMarkerByID** (Built)
Inputs: CombatantID (Integer)
Outputs: OutMarker (BP_CombatantMarker ref), bFound (Boolean)
Location: BP_CombatManager function

ForEach over SpawnedMarkers, Cast to BP_CombatantMarker, Compare CombatantID, return marker + bFound. Replaces ForEach-SpawnedMarkers-Cast-Compare patterns in: RemoveMarkerForCombatant, MoveMarkerToTile, SetMarkerDowned.

Note: Flow diagrams below for MoveMarkerToTile, ApplyDamage, ApplyCondition, RemoveCondition, HasCondition, SetMarkerDowned, RemoveMarkerForCombatant, UpdateCombatantGridPosition, EnterMoveMode, ExecuteCombatantMove, CheckVictory, CheckDefeat, OnActionComplete, ExecutePlayerAttack still show the old ForEach patterns. In the current code, all of these call FindCombatantByID or FindMarkerByID instead. The logic is identical; the diagrams are just structurally outdated.

 — duplicated logic patterns found in multiple Blueprints. Build these when next touching the relevant area, or as a dedicated refactor pass before Phase 6.


FindCombatantByID(CombatantID)
  → For Each Combatants
       → Break SCombatant → CombatantID
       → Branch: CombatantID == input CombatantID?
            True → Return Element, bFound=true
  → Completed (no match) → Return default SCombatant, bFound=false
```
DELETED_ORPHANED, break out CombatantID, compare" pattern duplicated across `ApplyDamage`, `UpdateCombatantGridPosition`, attacker/defender lookups in `ExecutePlayerAttack`/`ExecuteEnemyAttack`, and similar lookups anywhere a single combatant needs to be found by ID rather than all combatants needing modification.

### New library: BPL_GridMathLibrary (separate from BPL_RulesLibrary — spatial math, not AD&D rules)
| Function | Pure | Inputs | Outputs | Replaces |
|---|---|---|---|---|
| GetChebyshevDistance | ✅ | X1, Y1, X2, Y2 (Integer) | Distance (Integer) | Inline `Max(Abs(ΔX), Abs(ΔY))` duplicated in BP_EnemyAI.FindNearestPartyMember and BP_EnemyAI.MoveOneStepToward |

### Additions to BPL_RulesLibrary (ability-score-to-bonus family, alongside existing GetStrengthBonus)
| Function | Status | Pure | Inputs | Outputs | Notes |
|---|---|---|---|---|---|
| GetConstitutionHPBonus | ✅ Built (GB-041) | ✅ | Constitution (Integer) | HPBonus (Integer) | AD&D CON table: ≤6=-1, 7-14=0, 15-16=+1, 17=+2, 18=+3. Used by character creation and level-up HP rolls |
| GetDexterityACBonus | Not built | ✅ | Dexterity (Integer) | ACBonus (Integer) | Same table shape. Needed when ComputeAC uses real DEX instead of stub |
| GetWisdomSaveBonus | Not built | ✅ | Wisdom (Integer) | SaveBonus (Integer) | Needed for Cleric/Druid saving throws (GB-038) |

### Deferred candidate — not yet worth building
**FindNearestInArray (generic)** — the "loop collection, compute distance, track running best" shape appears in `FindNearestPartyMember` and `FindNearestTraversableTile`. Blueprint can't easily genericize this across different struct types (SCombatant vs grid coordinates) without a wrapper interface, so not a clean extraction yet. Revisit only if a third near-identical case appears (e.g. a "find nearest monster" spell effect post-VS).

### Blueprint editor tip — reducing Break/Make struct node clutter
For "take a struct, change one field, pass it on" patterns: right-click directly on a struct pin (Make, Break, or variable Get node) → **Split Struct Pin** — expands the pin into individual field pins on the same node, avoiding a separate Break/Make node pair. Right-click again → **Recombine Struct Pin** to collapse back. Faster to read and wire than full Break→Make chains for small edits. Note: Blueprint cannot genericize "copy struct, change field X" into a single reusable function since each call site changes a different field — this is a real engine limitation, not a missed opportunity.

---

## BPL_RulesLibrary ✅ *(GB-079 migration complete — all combat/leveling/save math now on The Threshold System)*
**Parent Class:** Blueprint Function Library  
**Location:** Blueprints/Libraries/

All functions are globally accessible. Most are Pure except dice-rolling functions (not Pure to prevent double-evaluation).

| Function | Pure | Inputs | Outputs | Notes |
|---|---|---|---|---|
| GetHPFraction | ✅ | CurrentHP (int), MaxHP (int) | HPFraction (float) | Clamps 0–1, handles divide by zero |
| GetHPBarColor | ✅ | HPFraction (float) | LinearColor | >0.6=green, >0.3=yellow, >0.0=orange, else=grey |
| GetConditionColor | ✅ | ECondition | SlateColor | One colour per condition |
| GetMessageColor | ✅ | EMessageType | LinearColor | One colour per message type |
| GetMightBonus | ✅ | Might (int) | ToHitBonus, DamageBonus (int) | **GB-079 — replaces GetStrengthBonus.** Chained Branch table, no exceptional-score tier. See table below |
| GetVigorBonus | ✅ | Vigor (int) | HPBonus (int) | **GB-079 — replaces GetConstitutionHPBonus.** Chained Branch table, max +4 not +3. See table below |
| GetAbilityModifier | ✅ | Score (int) | Modifier (int) | **GB-079 — built.** Generic small-bonus table, same shape as GetMightBonus's ToHit column. Reused by ComputeSavingThrows for whichever ability score is relevant per save |
| ComputeSavingThrows | ✅ | Class (ECharacterClass), Level (int), Vigor, Reflex, Resolve (int) | FortitudeSave, ReflexSave, WillpowerSave (int) | **GB-079 — fully migrated, and relocated.** Was found living in BP_CharacterRules rather than BPL_RulesLibrary (second occurrence of this doc/reality mismatch, after GetStrengthBonus — worth a full audit of BP_CharacterRules's actual remaining contents). Deleted from BP_CharacterRules, rebuilt here. Formula: `16 − floor(Level/2) − GetAbilityModifier(relevant score) − ClassBonus`. Class bonus +2 to one save via three separate Select nodes keyed on Class: Fighter→Fortitude, Cleric→Willpower, MagicUser→Willpower, Thief→Reflex. Not yet called from any gameplay trigger — ready for GB-038 |
| GetCombatDistance | yes | GridPositionA (FVector2D), GridPositionB (FVector2D) | Distance (int) | **GB-042 -- built.** Chebyshev distance: Max(Abs(dx), Abs(dy)). Used by ExecutePlayerAttack and ExecuteEnemyAttack for range gating |
  | RollDice | ❌ | NumDice (int), DieType (int) | Total (int) | For Loop — supports multi-dice rolls (2d6 etc) |
| DiceRollWithModifier | ❌ | NumDice (int), DieType (int), Modifier (int) | Total (int) | RollDice + Modifier |
| PercentileRoll | ❌ | — | Result (int) | 1–100. Originally built for Rogue skill checks — now also the core die roll for ResolveAttack |
| ResolveAttack | ✅ | Attacker (SCombatant), Defender (SCombatant) | AttackResult (SAttackResult) | **GB-079 — fully migrated.** Percentage-based resolution — see below |
| GetXPThreshold | ✅ | Class (ECharacterClass), Level (Integer) | XPRequired (Integer) | **GB-079 — built.** Formula: `500 × Level × (Level−1) × ClassMultiplier`. Multiplier via Select node on Class: Fighter 1.0, Cleric 1.1, MagicUser 1.2, Thief 0.9 (others default 1.0, unused). Replaces reading XPRequired from DT_LevelProgression |

`GetStrengthBonus`, `GetConstitutionHPBonus`, and `ComputeAC` have been deleted — fully replaced, not kept as wrappers.

### ResolveAttack Flow ✅ *(migrated under GB-079 — percentage-based, no more d20/THAC0/AC)*
```
ResolveAttack(Attacker, Defender)
  → Break SCombatant (Attacker) → StrikeNumber
  → Break SCombatant (Defender) → DefenseRating
  → RawHitChance = StrikeNumber + MightBonus(stub 0) + WeaponBonus(stub 0) + SituationalModifier(stub 0) − DefenseRating
  → FinalHitChance = Clamp(RawHitChance, 5, 95)
  → PercentileRoll() → SET D100Roll
  → Branch: D100Roll <= FinalHitChance?
       True  → SET bHit=true → RollDice(1,6) → SET Damage
       False → SET bHit=false → SET Damage=0
  → Make SAttackResult(bHit, D100Roll, FinalHitChance, Damage) → Return
```
No more natural-20/natural-1 special-casing — the `Clamp(5, 95)` does that job now, as a side effect of bounding the percentage rather than a separate rule.

### ResolveAttack VS Stubs (still outstanding, unrelated to GB-079)
- **MightBonus** — still hardcoded 0. Replace with `GetMightBonus` once equipment/inventory exists (GB-053)
- **WeaponBonus** — returns 0. Replace with equipped weapon magic bonus (GB-053)
- **SituationalModifier** — returns 0. Replace with condition checks (GB-039)
- **Damage** — 1d6 default. Replace with weapon damage dice from DT_Items (GB-053)
- **Monster Might/Reflex** — SMonster has no ability scores; bonuses are pre-baked directly into each monster's StrikeNumber/DefenseRating in DT_Monsters instead (the recommended approach, now actually applied — see DT_Monsters below)

### GetMightBonus Table ✅
| Might | ToHit | Damage |
|---|---|---|
| ≤ 4 | −2 | −2 |
| 5–7 | −1 | −1 |
| 8–12 | 0 | 0 |
| 13–15 | +1 | 0 |
| 16–17 | +1 | +1 |
| 18 | +2 | +2 |

### GetVigorBonus Table ✅
| Vigor | HP Bonus |
|---|---|
| ≤ 4 | −2 |
| 5–7 | −1 |
| 8–11 | 0 |
| 12–14 | +1 |
| 15–16 | +2 |
| 17 | +3 |
| 18 | +4 |

### GetAbilityModifier Table ✅
| Score | Modifier |
|---|---|
| ≤ 4 | −2 |
| 5–7 | −1 |
| 8–12 | 0 |
| 13–15 | +1 |
| 16–17 | +1 |
| 18 | +2 |

---

## BP_CharacterRules
**Parent Class:** Actor  
**Location:** Blueprints/Core/

> Note: Core calculation functions have been moved to BPL_RulesLibrary.
> This actor may retain wrapper functions or be deprecated in future.
> ⚠ **Two functions found here unexpectedly during GB-079** (GetStrengthBonus, then ComputeSavingThrows) that the docs claimed were already in BPL_RulesLibrary. Both have now been deleted from here and rebuilt correctly in BPL_RulesLibrary. Given this happened twice, a full audit of whatever's left in BP_CharacterRules is worth doing — there may be more functions sitting here that should be in the shared library instead. Not yet done.

---

## Structs

### SAttackResult
| Field | Type | Notes |
|---|---|---|
| bHit | Boolean | True if attack connected |
| RollResult | Integer | ✅ GB-079: now the d100 roll (was d20) |
| RollNeeded | Integer | ✅ GB-079: now Final Hit Chance, a percentage (was a d20 target number) — field name kept as-is, meaning changed |
| Damage | Integer | Damage dealt (0 if miss) |

### S_Monster *(added AttackRange -- GB-042)*
  | Field | Type | Notes |
  |---|---|---|
  | AttackRange | Integer | **GB-042 -- added.** Default 1 (melee). Copied to S_Combatant during BuildCombatants. Set per-monster in DT_Monsters |

  ### S_Combatant *(added AttackRange -- GB-042)*
  | Field | Type | Notes |
  |---|---|---|
  | AttackRange | Integer | **GB-042 -- added.** Set during BuildCombatants: monsters from S_Monster.AttackRange, players hardcoded 1 (melee until weapon tables exist). Checked against GetCombatDistance in attack functions |

  ### S_MonsterPortrait
| Field | Type | Notes |
|---|---|---|
| MonsterID | Integer | Matches SMonster.MonsterID |
| PortraitTexture | Texture2D (object ref) | Portrait image asset — null until art is added |

### S_Encounter
| Field | Type | Notes |
|---|---|---|
| EncounterID | Integer | Primary key — matches STriggerEvent.EncounterID |
| PortraitID | Integer | Index into DT_MonsterPortraits — stub until GB-016 |
| IntroText | Text | Narrative text displayed before choices appear |
| Choices | Array of S_EncounterChoice | Inline array, 2–5 entries |

### S_EncounterChoice
| Field | Type | Notes |
|---|---|---|
| ChoiceLabel | Text | Button text shown to player |
| OutcomeType | EOutcomeType | Drives ResolveChoice branching |
| OutcomePayload | String | Combat: MonsterGroupID as string. Flee/TextOnly: result text. Reward: XP amount as string. |

---

## Enums

| Enum | Values | Notes |
|---|---|---|
| EGameState | MainMenu, Exploration, Encounter, Combat, Camp, CharacterScreen | |
| EDirection | North, East, South, West | |
| ECharacterClass | Warden, Skirmisher, Templar, Devout, Sylvan, Adept, Shadowpriest, Rogue, Infiltrator | ✅ GB-079 — renamed from Fighter/Ranger/Paladin/Cleric/Druid/MagicUser/Illusionist/Thief/Assassin. Mechanics (Hit Dice, Extra Attacks, Ambush Strike) not yet built — only the names exist |
| ERace | Human, Elf, Dwarf, Gnome, HalfElf, Halfling, HalfOrc | |
| EAlignment | LawfulGood through ChaoticEvil (9 values) | |
| ECombatAction | Move, Attack, Cast, UseItem, Guard, Flee | |
| ECombatAction_AI | MoveToTarget, AttackNearest, AttackIncapacitated, CastSpell, Flee | |
| ECondition | Normal, Restrained, Paralysed, Poisoned, Blinded, Quickened, Slowed, Diseased, Sapped, Petrified, Dead, Unconscious | ✅ GB-079 — renamed from Held/Hasted/LevelDrained. Tick/effect logic not yet built — VS scope only wires up Dead + Restrained |
  | EMoraleState | Normal, Shaken, Fleeing | ✅ GB-040 — drives monster morale checks. Auto-valued (0/1/2). BP_CombatManager.ResolveMorale reads this to determine Shaken vs Fleeing transition |
| ESpellSchool | MagicUser, Illusionist, Cleric, Druid | ⚠ legacy — target 2 values, Arcane/Divine, GB-079. Deliberately deferred — magic system not built yet |
| ESpellEffect | Damage, Heal, AoEDamage, Entangle, Hold, Haste, Slow, Blind, LevelDrain, Summon, Utility | |
| ETriggerType | OnEnterTile, OnSearchTile, OnStoryFlag, OnTimedCondition, OnCombatEnd, OnItemUsed, OnDayNight, OnMoonPhase | |
| ETriggerActionType | ShowMessage, StartEncounter, StartCombat, TransitionLevel, SetStoryFlag, SpawnLoot | |
| EMessageType | Exploration, Combat, System, Loot, Encounter | |
| EOutcomeType | Combat, Reward, Flee, TextOnly | |
| ETileType | Floor, Pit, Water, Teleport, StairsUp, StairsDown | |
| ESaveType | Fortitude, Reflex, Willpower | ✅ GB-079 — restructured from VsPoison/VsWands/VsPetrification/VsBreathWeapon/VsSpells (no 1:1 mapping, clean replace) |
| ETestType | Float, Integer, Bool | |

---

## Data Tables

| Table | Struct | Purpose | Status |
|---|---|---|---|
| DT_LevelProgression | SLevelProgression | StrikeNumber, hit dice by class/level | ✅ Populated to L5. **GB-079: StrikeNumber column migrated to new ascending percentage values. XPRequired column now unused — CheckLevelUp reads GetXPThreshold's formula result instead, kept in the table only for reference. All 20 row names renamed (Fighter_1→Warden_1, Cleric_1→Devout_1, MagicUser_1→Adept_1, Thief_1→Rogue_1, etc) to match the renamed ECharacterClass enum.** |
| DT_SavingThrows | SSavingThrowRow | Save values by class/level | ✅ Populated to L5 — ✅ **GB-079 complete: superseded by ComputeSavingThrows's 3-category formula, table now entirely unused, kept for reference only.** |
| DT_Monsters | SMonster | 5 monsters seeded for VS | ✅ Complete. **GB-079: all 5 monsters' DefenseRating/StrikeNumber migrated to percentage values (Goblin 10/50, Orc 10/55, Skeleton 8/55, Zombie 5/50, Giant Rat 3/50)** |
| DT_TriggerEvents | STriggerEvent | Scripted dungeon triggers | ✅ 2 rows |
| DT_Encounters | S_Encounter | Scripted encounters | ✅ 1 row (Goblin patrol, 3 choices) |
| DT_MonsterPortraits | S_MonsterPortrait | Monster portrait textures | ✅ 1 stub row (no art yet) |
| DT_DungeonMaps | SDungeonMap | Dungeon level data | ✅ Empty (GenerateTestMap used) |
| DT_HPFractionTests | SRulesTestCase | Unit tests | ✅ 8 tests passing |
| DT_THAC0Tests | SThAC0TestCase | Unit tests | ⚠ **Tests the deleted d20 THAC0/AC math — stale, needs rebuilding against the new percentage ResolveAttack or removing entirely** |
| DT_SavingThrowTests | SSavingThrowTestCase | Unit tests | ⚠ Tests the old 5-category table-driven saves — stale now that saves are migrated. Candidate for delete-and-recreate |
| DT_StrengthBonusTests | SStrengthBonusTestCase | Unit tests | ⚠ **Tests the deleted GetStrengthBonus function — stale, needs rebuilding against GetMightBonus or removing entirely** |
| DT_AttackResolutionTests | SAttackTestCase | Unit tests | ⚠ **Tests the old d20 ResolveAttack — stale, needs rebuilding against the new percentage version or removing entirely** |
| DT_MultiAttackTests | SMultiAttackTestCase | Unit tests | ✅ Rows ready (unaffected — multi-attack not built yet) |
| DT_XPThresholdTests | SXPThresholdTestCase | Unit tests | ⚠ **Tests the deleted table-lookup XP threshold — stale, needs rebuilding against GetXPThreshold's formula or removing entirely** |
| DT_LevelCapTests | SLevelCapTestCase | Unit tests | ⚠ Possibly keyed by old class names — unverified, candidate for delete-and-recreate alongside the others |

---

## Condition Abbreviations Reference
| ECondition | Display | Color |
|---|---|---|
| Normal | _(blank)_ | Transparent |
| Restrained | RST | Purple (0.7, 0.4, 0.9) |
| Paralysed | PAR | Purple (0.7, 0.4, 0.9) |
| Poisoned | PSN | Acid green (0.6, 0.9, 0.1) |
| Blinded | BLD | Amber (0.8, 0.6, 0.1) |
| Quickened | QCK | Cyan (0.1, 0.9, 0.9) |
| Slowed | SLW | Muted blue (0.5, 0.5, 0.7) |
| Diseased | DIS | Dark red (0.4, 0.1, 0.1) |
| Sapped | SAP | Dark red (0.4, 0.1, 0.1) |
| Petrified | STN | Dark red (0.4, 0.1, 0.1) |
| Unconscious | UNC | Dark red (0.4, 0.1, 0.1) |
| Dead | DED | Dark red (0.4, 0.1, 0.1) |

## Message Type Colors
| EMessageType | Color (RGBA) | Description |
|---|---|---|
| Exploration | (0.78, 0.72, 0.48, 1.0) | Parchment gold |
| Combat | (0.9, 0.4, 0.3, 1.0) | Warm red |
| System | (0.3, 0.7, 0.3, 1.0) | Green |
| Loot | (0.5, 0.8, 0.9, 1.0) | Light blue |
| Encounter | (0.7, 0.5, 0.9, 1.0) | Purple |

---

## HP Bar Colors
| Condition | Threshold | Color (RGBA) |
|---|---|---|
| Healthy | HPFraction > 0.6 | (0.2, 0.8, 0.2, 1.0) Green |
| Wounded | HPFraction > 0.3 | (0.9, 0.75, 0.1, 1.0) Yellow |
| Critical | HPFraction > 0.0 | (0.9, 0.35, 0.1, 1.0) Orange |
| Dead | HPFraction == 0.0 | (0.3, 0.3, 0.3, 1.0) Grey |

---

## UE5 Lessons Learned

### Anchors in UMG
- Anchor presets determine what point widgets are pinned to
- Top-Right anchor: Position X is negative (measures leftward from right edge)
- Bottom-Left anchor: Position Y is negative (measures upward from bottom)
- Full-stretch anchor (all four corners): use Offset values to control margins
- Use **Alt+drag** to create SET nodes for widget variables
- Widget variables created from Designer are read-only in Graph — access via variable list

### Event Dispatchers
- Game Instance has inherited `OnGameStateChanged` — read only, cannot use
- Created `OnGameStateUpdated` as replacement on BP_GameManager
- Use **Assign On X** (drag off object ref) rather than Bind Event — generates correctly typed custom event automatically
- Delegate (red) pins cannot be manually connected — use Assign node approach
- Assign On X only appears when dragging off a correctly typed object ref with Context Sensitive ON

### Widget Lifecycle
- **Event Construct does NOT fire until the widget is added to the viewport**
- For widgets that add themselves conditionally (via state binding), use an explicit `Initialise` function called from Level Blueprint instead of Event Construct
- Create Widget + Initialise + Add to Viewport (conditional) is the correct pattern
- **OnGameStateUpdated binding does NOT work for widgets not yet in viewport** — the Assign node requires the widget to already be active. Use direct manager ref pattern instead: have BP_CombatManager (or BP_EncounterManager) call Add to Viewport / Remove from Parent directly via a stored widget ref
- EncounterManagerRef on child widgets (WBP_ChoiceButton) should be retrieved from Game Instance at click time — stored refs go stale

### Blueprints General
- Pure functions have no exec pins — data wires only
- Switch on Enum nodes: search with Context Sensitive OFF if not appearing
- Get Data Table Row: struct type is inferred from selected Data Table — no manual selection needed
- Widget array variables: use Make Array → Promote to Variable to get correct widget reference types
- Blueprint Function Libraries: globally accessible, no target needed — best home for rules functions
- Git reverts of .uasset files break references — after revert, expect missing structs/enums/data tables
- To call a function on another Blueprint, drag from its typed object ref with Context Sensitive ON
- String to Text conversion required when feeding String data into PostMessage or any UMG Text widget
- Manager refs must be explicitly SET after spawning — never auto-populated
- Breakpoints show as red dots on nodes — remove before testing to avoid false "not firing" diagnosis
- Subtract node bottom pin must be `1` not `-1` — entering −1 means subtracting negative one (= adding), a recurring source of off-by-one errors

### Combat System
- **CanMove vs IsMovementLocked** — CanMove is used for tile traversability checks in the movement system, NOT for blocking input during combat. Use IsMovementLocked to block all movement input during combat/encounter states
- **ISM per-instance material parameters** — ISM does not support per-instance Dynamic Material Instances. Use separate actor-based approach (BP_TileHighlight) for per-tile visual feedback instead
- **Blueprint recursion via latent functions** — calling a function that calls itself (even indirectly via OnActionComplete → StartEnemyTurn → OnActionComplete) causes infinite loop errors. Use Return Node to exit cleanly before the secondary call fires
- **For Each Clear Array bug** — placing Clear Array inside a For Each Loop Body fires after the first iteration, leaving stale references. Always place Clear Array on the For Each Completed pin
- **Destroy Actor target defaults to Self** — always explicitly wire the cast result to Destroy Actor's Target pin. Leaving it at default destroys the calling actor
- **Multiple ISM components on the same actor each need their own Clear Instances call** — adding a second ISM (e.g. WallMeshISM alongside TileMeshISM) for a new visual layer is easy to miss clearing on rebuild. Stale instances from a previous rebuild silently accumulate and overlap new ones, which can look like a logic/coordinate bug when the actual cause is simply an un-cleared ISM
- **Verify a Set node's Target type before wiring, not after** — a "Set [Variable]" node's title bar always states which Blueprint it targets (e.g. "Target is BP_CombatManager"). If a variable of the same name exists on multiple Blueprints, it's easy to accidentally place a Set node for the wrong one. Check the title bar first; if the Target pin refuses to accept the expected reference, that refusal IS the diagnostic — delete and recreate the node rather than trying to force the wrong connection
- **Renamed functions can silently drift from their actual behaviour** — a function renamed without updating its internal logic (or vice versa) still works, but misleads anyone reading the Blueprint later. If a function's name and behaviour disagree, fix the name to match the behaviour rather than leaving a comment to explain the mismatch

### Camera
- Camera Components on non-pawn actors cannot be set as the active camera in UE5 — no "Set Active Camera" option exists. Use a dedicated **Camera Actor** placed in the level instead
- Set View Target With Blend with Blend Time=0 produces an instant cut — use this for Gold Box-style camera switches
- Return view target after combat should be **Get Player Pawn** (the ExplorerPawn), not a raw actor location — returns to correct eye height

### Game Instance Timing
- Game Instance Event Init fires before viewport exists — cannot create widgets here
- Create widget from Level Blueprint Event Begin Play instead
- BP_GameManager (Game Instance) runs on ALL levels — use Get Current Level Name gate to skip initialisation on test levels
- Manager refs set from Level Blueprint may not be ready immediately — add a short Delay (0.1s) before reading them if getting None errors

### Viewport Rendering Overhaul (Deferred — post-VS)
Currently both exploration and combat cameras render full-screen behind the HUD overlay. This is functional for VS but not visually authentic to the Gold Box layout. The correct approach:
- Create a **Render Target 2D** asset
- Assign the active camera to render into the RT via a **Scene Capture Component** or camera assignment
- Display the RT as an **Image widget** in WBP_ExplorationHUD, anchored to the left viewport area (top-left panel)
- HUD panels (party status, text log, command menu) sit alongside the RT image rather than overlaying a full-screen camera
- Affects both BP_ExplorerPawn camera (exploration) and CombatCameraActor (combat)
- Must be coordinated with the combat camera switch — RT source swaps between exploration and combat camera on state change
- Deferred to art/polish pass after VS is complete

---

*Blueprint Reference Document — Gold Box RPG UE5.7*
*Updated: GB-042/GB-043/GB-046 session — full VS combat loop complete*
*Phase 4a-4c complete, Phase 4d next*
