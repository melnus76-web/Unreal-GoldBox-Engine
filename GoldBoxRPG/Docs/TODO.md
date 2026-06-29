# Gold Box RPG â€” TODO / Deferred Items
## UE5.7 Â· Blueprints Only Â· Solo Dev

---

## Recent Fixes

| Item | Notes |
|---|---|
| BuildInitiativeOrder Loop Body disconnected | ForEach Loop Body was not wired to DiceRoll after refactor. All combatants had Initiative=0. Reconnected. |

---

## Conditions (GB-039) â€” 3 Remaining

| Condition | Blocker | Notes |
|---|---|---|
| **Diseased** | Game-time/calendar system | Vigor reduced by 1 per day until cured. Needs a day/night or rest system to trigger |
| **Sapped** | Level-down logic | Permanent loss of 1 character level. Needs a reverse-level-up procedure (recompute SN, MaxHP, remove spell slots, etc.) |
| **Petrified** | Damage-type system | Treat as Paralysed, immune to damage except force/sonic. Needs a damage-type enum and resistance checks |

---

## Combat â€” Deferred from Phase 4

| Item | Ticket | Notes |
|---|---|---|
| Live HP sync during combat | New (GB-039a) | `SCharacter` in `BP_PartyManager` only updates on death, not on hits. Party panel shows stale HP for living damaged characters |
| Full defeat/game-over screen | GB-046 | `CheckDefeat` stub calls `EndCombat(false)` â€” needs proper game-over UI |
| Multi-condition display | GB-039 | Party panel only checks `Contains(Dead)`. Needs multi-icon or scrolling condition display |
| Downed marker visual polish | GB-039 | Currently 0.5 scale only â€” needs material tint, icon, or animation |
| Monster AI multi-attack | GB-037 | `GetAttacksThisRound` works for player but `ExecuteEnemyAttack` doesn't use it yet. Current monsters all have `NumAttacks=1` (harmless) |
| Adjacency check for attacks | GB-042 | Complete. S_Monster/S_Combatant.AttackRange fields, GetCombatDistance in BPL_RulesLibrary, range gate in ExecutePlayerAttack and ExecuteEnemyAttack. Monsters default melee (range=1), players hardcoded range=1 until weapon tables exist |
| Movement range redesign | GB-042 | Currently hardcoded to 3 for VS. Needs rules-based system (class/race/armour-based) |
| Attack range / diagonal movement | GB-042 | Chebyshev allows 8-directional movement. Consider whether diagonal should cost extra |

---

## Rules Engine â€” Deferred

| Item | Ticket | Notes |
|---|---|---|
| ResolveSave consumer wiring | GB-038 | `ResolveSave` built but nothing calls it yet. GB-039 poison ticks call Fortitude. Still needs: Ambush Strike (GB-044) Reflex, retreat (GB-046) Reflex |
| Morale system | GB-040 | ✅ Core complete: E_MoraleState enum, S_Combatant.MoraleState + GroupID, ResolveMorale(GroupID) in BP_CombatManager, Normal→Shaken→Fleeing transitions, Fleeing monsters skip turn. Remaining: flee-to-map-edge + removal from combat. **Polish:** base morale check on total morale rating of the monster group (sum all monsters' MoraleRating) rather than per-monster |
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
| ESpellSchool 4â†’2 shrink | Currently MagicUser/Illusionist/Cleric/Druid. Threshold target: Arcane/Divine. Deferred until magic system built (Phase 8) |

---

## Deferred Polish Items

| Item | Notes |
|---|---|
| Player camera centring in viewport | BP_ExplorerPawn â€” polish task |
| Standalone mode dungeon rendering | DungeonGenerator None error, works in PIE Selected Viewport |
| CommandMenu visible during Encounter | WBP_ExplorationHUD â†’ RefreshCommandMenu |
| Portrait area white placeholder | WBP_EncounterScreen â€” deferred to art pass |
| Monster spawn zone logic | BP_CombatManager â†’ BuildCombatants â€” FindNearestTraversableTile starts at (14,8), may land wrong side for some maps |
| HUD font sizes | Too small in exploration mode |
| Viewport rendering overhaul | Switch to Render Target 2D approach instead of full-screen behind HUD |
| BP_CharacterRules audit | Two functions found living there instead of BPL_RulesLibrary (GetStrengthBonus, ComputeSavingThrows). Worth checking what else remains there |

---

## Named CharacterID Constraint

`SCharacter.Name` is not guaranteed unique â€” any character-matching logic must use `CharacterID`, never `Name`. CharacterID is auto-assigned sequentially by `BP_PartyManager.AddCharacter`.

---

*Generated after completing conditions 8/12 (Blinded, Quickened, Slowed, Paralysed, Unconscious).*
