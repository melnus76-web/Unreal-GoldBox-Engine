# Gold Box RPG — UE5.7 Blueprint Project Progress Summary
## Session Reference Document

---

## Project Overview

A Gold Box-inspired RPG built in **Unreal Engine 5.7**, Blueprints only. Tactical grid combat, six-character party management, and an original ruleset called **The Threshold System** — see Ruleset Notice below.

> **Ruleset Notice:** GB-079 (the migration to The Threshold System, `Threshold_Ruleset_v1.md`) is now **essentially complete**, with one item deliberately deferred. Done and tested: ability scores, the entire attack resolution system, XP thresholds, saving throws (3 categories, formula-based), and the `ECharacterClass`/`ECondition` enum renames. Deliberately deferred: `ESpellSchool`'s 4→2 shrink, since the magic system isn't built yet.

**Source documents in project:**
- `SSI_Gold_Box_Engine.md` — technical reference
- `Threshold_Ruleset_v1.md` — the original ruleset (target design)
- `GoldBox_UE56_Blueprint_Roadmap_1.md` — full feature roadmap
- `GoldBox_Solo_Dev_Build_Order.md` — phased build order
- `GoldBox_Blueprint_Reference.md` — live Blueprint architecture reference

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
- ✅ **GB-018** ⚠ partial — BP_CharacterRules
- ✅ **GB-020** — BP_PartyManager (AddCharacter, GetLivingMembers, IsPartyWiped, LoadSavedParty, SavePartyToDisk)
- ✅ **GB-020a** — Hardcoded test party

### Phase 2 — Dungeon Exploration
- ✅ **GB-007** — SDungeonTile, SDungeonMap structs, DT_DungeonMaps
- ✅ **GB-008** — BP_MapManager
- ✅ **GB-005** — BP_ExplorerPawn
- ✅ **GB-006** — Movement system
- ✅ **GB-010** — Placeholder dungeon meshes
- ✅ **GB-011** — BP_DungeonGenerator

### Phase 2 — Exploration UI (EPIC 003)
- ✅ **GB-012** — WBP_ExplorationHUD
- ✅ **GB-013** — Party Status Panel
- ✅ **GB-014** — WBP_TextLog + BP_TextManager
- ✅ **GB-015** — WBP_CommandMenu

### Phase 3 — Scripted Encounter (EPIC 007)
- ✅ **GB-025** — ETriggerType framework, STriggerEvent struct, DT_TriggerEvents
- 🔄 **GB-026** — BP_EventManager (partial)
- ✅ **GB-027** — S_Encounter struct, DT_Encounters
- ✅ **GB-028** — BP_EncounterManager
- ✅ **GB-029** — WBP_EncounterScreen, WBP_ChoiceButton
- ✅ **GB-016** — WBP_EncounterPortrait stub

### Phase 4a — Combat Grid and Camera (EPIC 008)
- ✅ **GB-030** — BP_CombatGrid
- ✅ **GB-031** — CombatCameraActor

### Phase 4b — Combat Data and Turn Flow (EPIC 008)
- ✅ **GB-022** — DT_Monsters seeded
- ✅ **GB-033** — BP_CombatManager
- ✅ **GB-034** — Combat Turn Flow

### Phase 4c — Attack Resolution (EPIC 008/009)
- ✅ **GB-036** — ResolveAttack (migrated under GB-079)
- ✅ **GB-043** — Attack Action (VS scope)
- ✅ **GB-042** — Move Action (VS scope)
- ✅ **GB-046 (partial)** — CheckVictory + EndCombat

### Phase 4d — Enemy AI (EPIC 008)
- ✅ **GB-045** — BP_EnemyAI full movement AI complete

### Phase 4e — Combat HUD and XP (EPIC 008/EPIC 004)
- ✅ **GB-004a** — WBP_CombatHUD stub complete
- ✅ **GB-041** — BP_XPManager complete

### Phase 4f — Minimum Conditions (GB-039) ✅
### Phase 4h — Multiple Monster Spawning + Initiative Fix ✅
### Phase 4i — Ambush Strike (GB-044) ✅

---

### Phase 7 — Character Creation (GB-019) ✅ COMPLETE

**WBP_CharacterCreation** — 7-step wizard widget. All panels wired and verified.

- ✅ **Step 0 — RollStatsPanel:** 6 ability score rows with DiceRoll(3,6), 3 re-rolls each
- ✅ **Step 1 — ChooseRacePanel:** 7 race buttons with highlight logic
- ✅ **Step 2 — ChooseClassPanel:** 9 class buttons with highlight logic
- ✅ **Step 3 — ChooseAlignmentPanel:** 7 morality buttons with highlight logic
- ✅ **Step 4 — NameEntryPanel:** EditableTextBox with Random/Clear buttons
- ✅ **Step 5 — PortraitPanel:** Placeholder (deferred to polish)
- ✅ **Step 6 — ConfirmPanel:** PopulateConfirmPanel wired, full summary display
- ✅ **Btn_Next step-6 branch:** Calls AssembleAndAddCharacter
- ✅ **AssembleAndAddCharacter:** DT_LevelProgression lookup, ComputeMaxHP, GetVigorBonus, ComputeSavingThrows, Make S_Character, AddCharacter, SavePartyToDisk
- ✅ **Save/Load:** SG_CharacterRoster SaveGame. LoadSavedParty (BeginPlay) + SavePartyToDisk in BP_PartyManager
- ✅ **Self-Init:** PartyManagerRef from Event Construct via GameInstance
- ⚠ **Portrait selection:** Deferred to polish

---

### Phase 7 — Character Sheet (GB-052) 🚧 IN PROGRESS

**WBP_CharacterScreen** — Tabbed character sheet widget with 5 tabs.

- ✅ Widget tree fully built: LeftPanel (party list), RightPanel (Stats/Equipment/Inventory/Spells/ThiefSkills tabs)
- ✅ Functions: Initialise, RefreshPartyList, OnPartySlotClicked, OnTabClicked, RefreshCharacterDetails, GetEquipName
- ✅ RefreshCharacterDetails: Breaks S_Character, populates all stat/save/equip values, computes saving throws, XP display via DT_LevelProgression, condition iteration
- ✅ RefreshPartyList: Dynamically creates WBP_PartySlot buttons from PartyMembers array
- ✅ Integrated with WBP_ExplorationHUD (listed as dependency)
- ⚠ Inventory/Spells/ThiefSkills tabs: Widget tree built but detail population logic not yet wired
- ⚠ Thief tab visibility gating (show only for Rogue/Infiltrator) not yet implemented

---

### GB-079 — Ruleset Migration to The Threshold System (Complete)
- (see original document for full GB-079 details)

---

### Phase 3 Test Checkpoint — PASSING ✅

---

## Current State (End of Session)

- ✅ Character creation (GB-019): COMPLETE — all 7 steps wired, save/load persistence working
- 🚧 Character screen (GB-052): Widget tree built, stats/equipment tabs populated. Inventory/Spells/Thief tabs need wiring
- Encounter system fully working end-to-end
- Full VS combat loop verified
- Dungeon renders correctly in PIE
- Ambush Strike (GB-044) complete
- GB-079 ruleset migration complete (ESpellSchool deferred)
- HUD font sizes adjusted for 1080p: WBP_CharacterCreation (headers 27, labels 21, buttons 18-19.5), WBP_CharacterScreen (headers 12, labels 11.25, buttons 10.5)

---

## Known Issues / Outstanding Bugs

| Issue | Location | Notes |
|---|---|---|
| PortraitPanel | WBP_CharacterCreation index 5 | Empty placeholder |
| Inventory/Spells/ThiefSkills tab population | WBP_CharacterScreen | Widget tree exists; detail logic not yet wired |
| Thief tab visibility gating | WBP_CharacterScreen | Should only show for Rogue/Infiltrator classes |
| MainMenu missing | WBP_MainMenu | Does not exist yet. Creation tested via TestDungeon Level BP |
| DT_LevelProgression gaps | DT_LevelProgression | Only Warden/Devout/Adept/Rogue have rows (20 total). Missing: Skirmisher/Templar/Sylvan/Shadowpriest/Infiltrator |
| Player camera not centred in viewport | BP_ExplorerPawn | Deferred — polish task |
| Standalone mode dungeon not rendering | BP_GameManager | DungeonGenerator None error — works in PIE |
| CommandMenu visible during Encounter | WBP_ExplorationHUD | Deferred to polish pass |
| Portrait area white placeholder | WBP_EncounterScreen | Deferred to art pass |
| Monster spawn zone | BP_CombatManager | May land on wrong side for some maps |

### Resolved
| Issue | Resolution |
|---|---|
| Party showing 8 members instead of 4 | Fixed — duplicate InitialiseTestParty call removed |
| Character creation save logic | Fixed — PopulateConfirmPanel + AssembleAndAddCharacter + SavePartyToDisk wired |
| HUD font sizes too small in exploration mode | Adjusted across all UI widgets for 1080p |

---

*Document updated: 2026-07-12 — GB-019 complete, GB-052 WBP_CharacterScreen in progress (widget tree + stats/equipment tabs populated), font sizes adjusted for 1080p.*
*UE5.7 · Blueprints Only · Solo Dev*
