# GB-019: Character Creation Flow — Development Guide
## UE5.7 · Blueprints Only · Solo Dev

---

## Status: COMPLETE (2026-07-11)

All 7 panels wired. Character assembly functional. Save/Load persistence working via `SG_CharacterRoster`.

---

## Overview

**Widget:** `WBP_CharacterCreation`  
**Pattern:** Widget Switcher with 7 stacked panels  
**Output:** Populated `S_Character` struct added to party via `BP_PartyManager.AddCharacter`, persisted to disk via `SG_CharacterRoster` SaveGame

### Architecture Notes (Post-Implementation)
- `PartyManagerRef` is self-initialised in `Event Construct` via `GetGameInstance -> Cast BP_GameManager -> Get PartyManager` — no external `Initialise` call needed.
- `CharacterName` is stored as `Text` on the widget, converted to `FString` via `Conv_TextToString` when assembling `S_Character`.
- `S_Character` uses `FortitudeSave`/`ReflexSave`/`ResolveSave` (renamed from old `SaveVs*` fields).
- `DT_LevelProgression` has rows for only 4 of 9 classes (Warden, Devout, Adept, Rogue). Other 5 use hardcoded level-1 defaults as fallback.

---

## Step 8 — Confirm & Add to Party (IMPLEMENTED)

### Summary Panel
Summary TextBlocks populated by `PopulateConfirmPanel()`:
- `Value_Name` <- `CharacterName`
- `Value_Race` <- `GetRaceDisplayName(SelectedRace)`
- `Value_Class` <- `GetClassDisplayName(SelectedClass)`
- `Value_Alignment` <- `GetAlignmentDisplayName(SelectedAlignment)`
- `Value_ConfirmMight/Vigor/Reflex/Acuity/Resolve/Presence` <- Conv_IntToText of each score

### Btn_Next at Step 6
`Btn_Next` handler branches: if `CurrentStepIndex == 6`, calls `AssembleAndAddCharacter` instead of advancing.

### AssembleAndAddCharacter Flow
1. Switch on `SelectedClass` -> lookup `DT_LevelProgression` row `"{ClassName}_1"` (for Warden/Devout/Adept/Rogue) or hardcoded fallback (other 5 classes)
2. `ComputeMaxHP(HitDiceCount, HitDiceType, HitPointsFixed)` -> TotalMaxHP
3. `GetVigorBonus(VigorScore)` -> VigorBonus
4. `Add(TotalMaxHP, VigorBonus)` -> FinalHP (= MaxHP = CurrentHP)
5. `ComputeSavingThrows(SelectedClass, 1, VigorScore, ReflexScore, ResolveScore)` -> FortSave, ReflexSave, ResolveSave
6. Make `S_Character` struct with all fields
7. `PartyManagerRef.AddCharacter(assembled struct)`
8. `PartyManagerRef.SavePartyToDisk()`
9. `RemoveFromParent()`

---

## Save/Load System (IMPLEMENTED)

### SG_CharacterRoster
SaveGame blueprint (`/Game/Blueprints/Data/SG_CharacterRoster`) with one variable: `SavedCharacters: TArray<S_Character>`.

### BP_PartyManager Functions
- `LoadSavedParty()` — called from `BeginPlay`. Checks if "CharacterRoster" save exists, loads and ForEach adds all characters to `PartyMembers`.
- `SavePartyToDisk()` — appends last PartyMember to existing save (or creates new save on first character). Both branches end with `SaveGameToSlot("CharacterRoster")`.

### Persistence Flow
Save file location: `Saved/SaveGames/CharacterRoster.sav`

---

## Verification Checklist

- [x] Roll all 6 stats — values appear, rolls remaining decrements
- [x] Roll button disables at 0 remaining
- [x] Next blocked if any stat unrolled
- [x] Race panel highlights selected, defaults to Human
- [x] Class panel shows all 9 classes, defaults to Warden
- [x] Alignment grid, defaults to Righteous
- [x] Name entry with Random/Clear
- [ ] Portrait click highlights selection (deferred)
- [x] Confirm shows full summary (Name, Race, Class, Alignment, 6 scores)
- [x] Confirm -> character added -> widget closes
- [x] Character saved to disk and loaded on restart
- [x] Back hidden on step 0, visible on others
- [x] Next says "Confirm" on final step

---

## Files Created/Modified

| File | Changes |
|---|---|
| `/Game/Blueprints/UI/WBP_CharacterCreation` | Full creation flow widget |
| `/Game/Blueprints/Data/SG_CharacterRoster` | SaveGame for character persistence |
| `/Game/Blueprints/Managers/BP_PartyManager` | Added `LoadSavedParty`, `SavePartyToDisk`; wired `BeginPlay -> LoadSavedParty` |
| `/Game/Blueprints/Structs/S_Character` | Renamed saves to FortitudeSave/ReflexSave/ResolveSave |
| `/Game/Data/DataTables/DT_LevelProgression` | Row data for Warden/Devout/Adept/Rogue levels 1-5 |
