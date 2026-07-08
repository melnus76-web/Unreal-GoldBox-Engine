# GB-019: Character Creation Flow — Development Guide
## UE5.7 · Blueprints Only · Solo Dev

---

## Overview

**Widget:** `WBP_CharacterCreation`  
**Pattern:** Widget Switcher with 7 stacked panels  
**Output:** Populated `S_Character` struct added to party via `BP_PartyManager.AddCharacter`

### Simplified from original GB-019 spec

| Change | Reason |
|---|---|
| Exceptional Strength dropped | Threshold System removed percentile subsystem |
| No multiclass | Hybrid classes come later as separate classes |
| No race/class restrictions | Deferred until needed |
| Portraits stored as int32 | 3D assets during polish phase |
| Dice from BPL_RulesLibrary | No separate BP_DiceSystem exists |
| ComputeMaxHP / ComputeSavingThrows | Already built on BPL_RulesLibrary |

---

## Step 1 — Widget Shell & Navigation

### 1a. Create the widget

Right-click in Content Browser → **User Interface** → **Widget Blueprint**. Name `WBP_CharacterCreation`. Save to `/Game/Blueprints/UI/`.

### 1b. Root structure

```
CanvasPanel (root, dark background R=0.02,G=0.02,B=0.03)
  Border "HeaderBar" — anchored top, full width, H=60
    TextBlock "StepLabel" — centered, Cinzel 18, gold, Is Variable
  WidgetSwitcher "StepSwitcher" — anchored below header, fills remaining area
    [panel 0] CanvasPanel "RollStatsPanel"
    [panel 1] CanvasPanel "ChooseRacePanel"
    [panel 2] CanvasPanel "ChooseClassPanel"
    [panel 3] CanvasPanel "ChooseAlignmentPanel"
    [panel 4] CanvasPanel "NameEntryPanel"
    [panel 5] CanvasPanel "PortraitPanel"
    [panel 6] CanvasPanel "ConfirmPanel"
  Border "NavBar" — anchored bottom, full width, H=56
    HorizontalBox "NavHBox"
      Button "Btn_Back" — Text "[Back]", Cinzel 14, dim gold
      Spacer (Fill)
      Button "Btn_Next" — Text "[Next]", Cinzel 14, gold
```

### 1c. Navigation widget variables

Add these to the widget (all **Is Variable**):

| Name | Type | Purpose |
|---|---|---|
| `CurrentStepIndex` | Integer (default 0) | Tracks active panel |
| `StepNames` | Text Array | `["Roll Stats","Choose Race","Choose Class","Choose Alignment","Name","Portrait","Confirm"]` |

### 1d. Navigation function

Create a function `UpdateNavigation` (no inputs, no outputs):

1. **SetText** on `StepLabel` = `StepNames[CurrentStepIndex]`
2. **SetVisibility** on `Btn_Back`: **Hidden** if `CurrentStepIndex == 0`, else **Visible**
3. **SetText** on `Btn_Next`: text `"Confirm"` if `CurrentStepIndex == 6`, else `"Next"`
4. **SetActiveWidgetIndex** on `StepSwitcher` = `CurrentStepIndex`

### 1e. Button event bindings

**Btn_Back OnClicked:** `CurrentStepIndex -= 1` → `UpdateNavigation`

**Btn_Next OnClicked** — starts as a simple advance. Replace with per-step validation as panels are built.

### 1f. Event Construct

Call `UpdateNavigation` to initialise the first step.

---

## Step 2 — Ability Score Roller

### 2a. Widget variables

| Name | Type | Default |
|---|---|---|
| `MightScore` | Integer | 0 |
| `VigorScore` | Integer | 0 |
| `ReflexScore` | Integer | 0 |
| `AcuityScore` | Integer | 0 |
| `ResolveScore` | Integer | 0 |
| `PresenceScore` | Integer | 0 |
| `MightRollsRemaining` | Integer | 3 |
| `VigorRollsRemaining` | Integer | 3 |
| `ReflexRollsRemaining` | Integer | 3 |
| `AcuityRollsRemaining` | Integer | 3 |
| `ResolveRollsRemaining` | Integer | 3 |
| `PresenceRollsRemaining` | Integer | 3 |

### 2b. Designer layout — `RollStatsPanel`

```
Border "StatsRollContainer" — centered, W=500, padding 12
  VerticalBox "StatsRollVBox"
    TextBlock "Header_RollStats" — "ROLL ABILITY SCORES", gold, Cinzel 16
    Spacer H=12
    [Six rows, each pattern:]
    HorizontalBox "Row_Might"
      SizeBox W=130 → TextBlock "Label_Might" — "Might", gold, Cinzel 14
      SizeBox W=60 → TextBlock "Value_Might" — "0", cream, right, Is Variable
      Spacer W=12
      Button "Btn_RollMight" — Is Variable, Text "Roll", Cinzel 12
      SizeBox W=80 → TextBlock "RollsLeft_Might" — "3 left", dim, Is Variable
    [Repeat for Vigor, Reflex, Acuity, Resolve, Presence]
```

### 2c. Roll handler per button

Example for **Btn_RollMight OnClicked**:

```
If MightRollsRemaining > 0:
  DiceRoll(3, 6) → set MightScore
  SetText Value_Might = MightScore (Format {0} → ToText)
  MightRollsRemaining -= 1
  SetText RollsLeft_Might = "{MightRollsRemaining} left" (Format Text)
  If MightRollsRemaining == 0:
    SetEnabled Btn_RollMight = false (or disable color)
```

Repeat for all six. Copy/paste and rename variables.

### 2d. Validation

Create `ValidateRollStats` → Boolean:

Check all six scores > 0. If any zero, return False.

In Btn_Next OnClicked, for step 0: Branch on `ValidateRollStats`. True → advance, False → do nothing.

---

## Step 3 — Race Selection

### 3a. Widget variable

| Name | Type | Default |
|---|---|---|
| `SelectedRace` | E_Race | Human |

### 3b. Designer layout — `ChooseRacePanel`

```
Border "RaceContainer" — centered, W=500, padding 12
  VerticalBox "RaceVBox"
    TextBlock "Header_ChooseRace" — "CHOOSE RACE", gold, Cinzel 16
    Spacer H=12
    UniformGridPanel "RaceGrid" — 3 columns, cell padding 8
      Button "Btn_Race_Human" → Text "Human"
      Button "Btn_Race_Elf" → Text "Elf"
      Button "Btn_Race_Dwarf" → Text "Dwarf"
      Button "Btn_Race_Gnome" → Text "Gnome"
      Button "Btn_Race_HalfElf" → Text "Half-Elf"
      Button "Btn_Race_Halfling" → Text "Halfling"
      Button "Btn_Race_HalfOrc" → Text "Half-Orc"
```

### 3c. Selection logic

Function `SelectRace(InRace E_Race)`:
1. Set `SelectedRace` = InRace
2. Loop all race buttons → set Brush Color to dark default
3. Set selected button Brush Color to gold highlight

Each OnClicked calls `SelectRace(Human)`, `SelectRace(Elf)`, etc.

On entering step (in UpdateNavigation, when step == 1): call `SelectRace(Human)`.

---

## Step 4 — Class Selection

### 4a. Widget variable

| Name | Type | Default |
|---|---|---|
| `SelectedClass` | E_CharacterClass | Warden |

### 4b. Designer layout — `ChooseClassPanel`

Same pattern as Race: UniformGridPanel 3 columns, 9 buttons:

Warden | Skirmisher | Templar  
Devout | Sylvan | Adept  
Shadowpriest | Rogue | Infiltrator

### 4c. Selection logic

Function `SelectClass(InClass E_CharacterClass)` — highlight pattern. No race filtering for now.

---

## Step 5 — Alignment Selection

### 5a. Widget variable

| Name | Type | Default |
|---|---|---|
| `SelectedAlignment` | E_Alignment | TrueNeutral |

### 5b. Designer layout — `ChooseAlignmentPanel`

UniformGridPanel 3×3:

| Good | Neutral | Evil |
|---|---|---|
| LawfulGood | LawfulNeutral | LawfulEvil |
| NeutralGood | TrueNeutral | NeutralEvil |
| ChaoticGood | ChaoticNeutral | ChaoticEvil |

Use short labels: "LG", "NG", "CG" / "LN", "TN", "CN" / "LE", "NE", "CE".

### 5c. Selection logic

Function `SelectAlignment(InAlign E_Alignment)` — highlight pattern.

---

## Step 6 — Name Entry

### 6a. Widget variable

| Name | Type | Default |
|---|---|---|
| `CharacterName` | String | "" |

### 6b. Designer layout — `NameEntryPanel`

```
Border "NameContainer" — centered, W=400, padding 16
  VerticalBox "NameVBox"
    TextBlock "Header_Name" — "CHARACTER NAME", gold, Cinzel 16
    Spacer H=16
    EditableTextBox "Input_Name" — Hint "Enter name...", Cinzel 14, Is Variable
    Spacer H=8
    TextBlock "Error_NameEmpty" — "Name cannot be empty", red, Is Variable
```

Error_NameEmpty starts Hidden.

### 6c. Validation

Function `ValidateName` → Boolean:

```
CharacterName = GetText(Input_Name) → ToString
If IsEmpty(CharacterName):
  SetVisibility Error_NameEmpty = Visible
  Return False
Else:
  SetVisibility Error_NameEmpty = Hidden
  Return True
```

Btn_Next for step 4 calls ValidateName before advancing.

---

## Step 7 — Portrait Selection (Placeholder)

### 7a. Widget variable

| Name | Type | Default |
|---|---|---|
| `SelectedPortraitID` | Integer | 1 |

### 7b. Designer layout — `PortraitPanel`

```
Border "PortraitContainer" — centered, W=500, padding 12
  VerticalBox "PortraitVBox"
    TextBlock "Header_Portrait" — "CHOOSE PORTRAIT", gold, Cinzel 16
    Spacer H=12
    HorizontalBox "PortraitHBox"
      [Six buttons side by side, each W=72 H=72:]
      Button "Btn_Portrait1" — number 1
      Button "Btn_Portrait2" — number 2
      ... through 6
```

Function `SelectPortrait(ID)` → highlight.

---

## Step 8 — Confirm & Add to Party

### 8a. Widget variable

| Name | Type |
|---|---|
| `PartyManagerRef` | BP_PartyManager (Object Ref) |

Set in Construct: `GetGameInstance → Cast BP_GameManager → get Party Manager`.

### 8b. Designer layout — `ConfirmPanel`

```
Border "ConfirmContainer" — centered, W=500, padding 16
  VerticalBox "ConfirmVBox"
    TextBlock "Header_Confirm" — "CONFIRM CHARACTER", gold, Cinzel 16
    Spacer H=12
    TextBlock "SummaryText" — empty, cream, Cinzel 14, Is Variable, Auto Wrap Text
    Spacer H=20
    Button "Btn_Confirm" — "CREATE AND JOIN PARTY", gold, Cinzel 16
```

### 8c. Build summary text

Function `BuildSummaryText` → Text. Called when entering Confirm step:

```
Name: {CharacterName}
Race: {SelectedRace via EnumToName → NameToString}
Class: {SelectedClass via EnumToName → NameToString}
Alignment: {SelectedAlignment via EnumToName → NameToString}

--- Ability Scores ---
Might: {MightScore}  Vigor: {VigorScore}
Reflex: {ReflexScore}  Acuity: {AcuityScore}
Resolve: {ResolveScore}  Presence: {PresenceScore}

Portrait: {SelectedPortraitID}
```

### 8d. Btn_Confirm OnClicked — AssembleAndAddCharacter

```
// 1. Create empty S_Character
Make S_Character with defaults

// 2. Fill identity
Set CharacterName, Race, Class1, Class2(==Class1), Alignment
Set Gender = "" (not used yet)

// 3. Fill ability scores
Set Might/Vigor/Reflex/Acuity/Resolve/Presence

// 4. Level & XP
Set Level = 1, Level2 = 0, XP = 0

// 5. Compute HP
GetDataTableRow(DT_LevelProgression, "{SelectedClass}_1")
→ HitDiceCount, HitDiceType, HitPointsFixed
ComputeMaxHP(HitDiceCount, HitDiceType, HitPointsFixed) + GetVigorBonus(VigorScore)
Set MaxHP, CurrentHP = result

// 6. Compute StrikeNumber & DefenseRating
Set StrikeNumber = from DT_LevelProgression row
Set DefenseRating = 0

// 7. Compute saving throws
ComputeSavingThrows(Class1, 1, VigorScore, ReflexScore, ResolveScore)
Store FortitudeSave, ReflexSave, WillpowerSave (map to old SaveVs fields for compat)

// 8. Empty spells, equipment, skills, inventory
Set all to 0 or empty arrays

// 9. Set portrait
Set Protrait = SelectedPortraitID
Set IsPlayerControlled = true
Set CharacterID = 0 (auto-assigned)

// 10. Add to party
PartyManagerRef.AddCharacter(AssembledCharacter)

// 11. Close
RemoveFromParent
```

---

## Step 9 — Opening the Widget

In `WBP_ExplorationHUD` (or similar), add a button or keybind:

```
OnClicked:
  CreateWidget(WBP_CharacterCreation, GetPlayerController(0))
  AddToViewport(ZOrder=10)
```

---

## Verification Checklist

- [ ] Roll all 6 stats — values appear, rolls remaining decrements
- [ ] Roll button disables at 0 remaining
- [ ] Next blocked if any stat unrolled
- [ ] Race panel highlights selected, defaults to Human
- [ ] Class panel shows all 9 classes, defaults to Warden
- [ ] Alignment 3×3 grid, defaults to TrueNeutral
- [ ] Name empty → error shown, Next blocked
- [ ] Portrait click highlights selection
- [ ] Confirm shows full summary
- [ ] Confirm → character added → party panel updates
- [ ] Character screen opens correctly for new character
- [ ] Back hidden on step 0, visible on others
- [ ] Next says "Confirm" on final step

---

## Files Created/Modified

| File | Changes |
|---|---|
| `/Game/Blueprints/UI/WBP_CharacterCreation` | New widget (full creation flow) |
| `/Game/Blueprints/UI/WBP_ExplorationHUD` | Add button to open creation widget |
