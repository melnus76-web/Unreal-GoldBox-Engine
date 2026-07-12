# Gold Box RPG — TODO / Deferred Items
## UE5.7 · Blueprints Only · Solo Dev

---

## Recent Fixes (2026-07-12)

| Item | Notes |
|---|---|
| **Character Creation — Confirm Panel** | Wired PopulateConfirmPanel, AssembleAndAddCharacter, Btn_Next step-6 branch. Full S_Character assembly with DT_LevelProgression lookup, ComputeMaxHP, GetVigorBonus, ComputeSavingThrows |
| **Save/Load System** | Created SG_CharacterRoster SaveGame. Added LoadSavedParty (BeginPlay) and SavePartyToDisk (appends) to BP_PartyManager. Characters persist to Saved/SaveGames/CharacterRoster.sav |
| **WBP_CharacterCreation Self-Init** | PartyManagerRef set from Event Construct via GetGameInstance -> Cast BP_GameManager -> PartyManager. No external Initialise call needed |
| **S_Character Save Fields** | Renamed SaveVsPoison/Wands/Petrification/Breath/Spells to FortitudeSave/ReflexSave/ResolveSave |
| GetAdjacentTiles DX/DY swap | NX was X+DY, NY was Y+DX (swapped). Fixed to NX=X+DX, NY=Y+DY. BFS now produces correct paths |
| FindPathBFS target tile exemption | IsTraversable blocked ALL occupied tiles including targets. Added NOT IsOccupied OR (Neighbour==End) gate so BFS can step onto target's tile |
| BuildInitiativeOrder Loop Body disconnected | ForEach Loop Body was not wired to DiceRoll after refactor. All combatants had Initiative=0. Reconnected. |
| **Font Size Adjustments** | All UI widgets adjusted for 1080p displays. WBP_CharacterCreation: headers 27, labels 21, buttons 18-19.5. WBP_CharacterScreen: headers 12, labels 11.25, buttons 10.5 |
| **WBP_CharacterScreen (GB-052)** | Widget tree built with 5 tabs. Functions implemented: Initialise, RefreshPartyList, OnPartySlotClicked, OnTabClicked, RefreshCharacterDetails, GetEquipName. Stats and Equipment tabs populated. Integrated with WBP_ExplorationHUD |

---

## Character Creation — Deferred

| Item | Notes |
|---|---|
| Portrait selection | Empty panel — 3D asset portraits deferred to polish phase |
| MainMenu integration | WBP_MainMenu does not exist yet. Creation tested via TestDungeon Level Blueprint. Will be rebuilt when all components are stable |
| Party Roster UI | DT_CharacterRoster is intended as a catalog the player selects from. SaveGame stores the full roster; party composition UI still needs to be built |
| DT_LevelProgression for missing classes | Only Warden/Devout/Adept/Rogue have rows. Skirmisher/Templar/Sylvan/Shadowpriest/Infiltrator use hardcoded fallback (d8 or d6, StrikeNumber=50) |

---

## Character Screen (GB-052) — In Progress

| Item | Notes |
|---|---|
| **Stats tab** | ✅ Populated — ability scores, HP (Current/Max), Defense Rating, saving throws (Fortitude/Reflex/Will) all set by RefreshCharacterDetails |
| **Equipment tab** | ✅ Populated — 8 slots (Weapon/Shield/Armour/Helmet/Ring1/Ring2/Cloak/Misc) via GetEquipName |
| **Inventory tab** | ❌ Widget tree built but no population logic. Needs ForEach Inventory → Create TextBlock grid |
| **Spells tab** | ❌ Widget tree built but no population logic. Needs spell slots display + memorised spell lists |
| **Thief Skills tab** | ❌ Widget tree built but no population logic. Needs 9 skill percentages. Class visibility gate not yet implemented |
| **Tab switching** | ✅ OnTabClicked → SetActiveWidgetIndex wired. Tab button highlighting not yet implemented |
| **Party list** | ✅ RefreshPartyList creates WBP_PartySlot buttons dynamically. OnPartySlotClicked → RefreshCharacterDetails |

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
| Monster AI multi-attack | GB-037 | `GetAttacksThisRound` works for player but `ExecuteEnemyAttack` doesn't use it yet |
| Dynamic combat camera | GB-047 | Partially built. Blocked: CombatHUD overlays viewport behind UI panels |
| Full Enemy AI | GB-045 | Movement complete. TODO: Incapacitated target priority, intelligent spellcasting |

---

*Last updated: 2026-07-12 — Added font size fixes and WBP_CharacterScreen (GB-052) progress*
