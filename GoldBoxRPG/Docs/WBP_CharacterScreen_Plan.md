# WBP_CharacterScreen — Build Plan
## User Story GB-052 / EPIC 012 — Character Management

---

## 1. What We're Building

A tabbed character sheet widget (`WBP_CharacterScreen`) with five tabs:

| Tab | Content |
|-----|---------|
| **Stats** | Ability scores, HP, AC, THAC0, XP, Level, Saving Throws, Active Conditions |
| **Equipment** | 8 equipment slots (Weapon, Shield, Armour, Helmet, Ring ×2, Cloak, Misc) |
| **Inventory** | Scrollable grid of carried item IDs |
| **Spells** | Spell slots per level (L1–L7), memorised spell lists |
| **Thief Skills** | 9 skill percentages — visible only for Rogue/Infiltrator |

---

## 2. Data Source

**Primary struct:** `/Game/Blueprints/Structs/S_Character` (64 fields)

**Key fields by tab:**

- **Ability scores:** Might, Acuity, Resolve, Reflex, Vigor, Presence
- **Combat:** CurrentHP, MaxHP, DefenseRating (=AC), StrikeNumber (=THAC0), XP, Level, Level2
- **Saves:** SaveVsPoison, SaveVsWands, SaveVsPetrification, SaveVsBreath, SaveVsSpells
- **Equipment:** EquipWeapon, EquipShield, EquipArmour, EquipHelmet, EquipRing1, EquipRing2, EquipCloak, EquipMisc (all int32 IDs)
- **Inventory:** Inventory (TArray\<int32\>)
- **Spells:** SpellSlotsCurrentL1–L7, SpellSlotsMaxL1–L7, MemorisedSpellsL1–L7
- **Thief skills:** SkillPickPockets, SkillOpenLocks, SkillFindTraps, SkillRemoveTraps, SkillMoveSilently, SkillHideInShadows, SkillHearNoise, SkillClimbWalls, SkillReadLanguages
- **Conditions:** Condition (TArray\<E_Condition\>), E_Condition enum: Normal, Restrained, Paralysed, Poisoned, Blinded, Quickened, Slowed, Diseased, Sapped, Petrified, Dead, Unconscious

**Runtime data source:** `BP_PartyManager` actor → `PartyMembers` (TArray\<S_Character\>) + `OnPartyUpdated` delegate

**Test data:** `/Game/Data/DataTables/DT_TestParty` — 4 rows (Fighter: Thorin the Warden, Cleric, MagicUser, Thief: Mira the Rogue)

---

## 3. Widget Hierarchy

```
CanvasPanel "RootCanvas"
├── Border "LeftPanel"                          [character list sidebar]
│   └── VerticalBox
│       ├── TextBlock "PartyHeader"              "PARTY"
│       ├── ScrollBox
│       │   └── VerticalBox "PartyListVBox"      [buttons added dynamically]
│       └── Button "Btn_Close"                   "CLOSE"
│
├── Border "RightPanel"                          [tabbed detail area]
│   └── VerticalBox
│       ├── TextBlock "CharacterNameHeader"
│       ├── HorizontalBox "TabButtonRow"
│       │   ├── Button "Btn_TabStats"             "STATS"
│       │   ├── Button "Btn_TabEquipment"         "EQUIPMENT"
│       │   ├── Button "Btn_TabInventory"         "INVENTORY"
│       │   ├── Button "Btn_TabSpells"            "SPELLS"
│       │   └── Button "Btn_TabThief"             "THIEF SKILLS"
│       └── WidgetSwitcher "TabSwitcher"
│           ├── [0] Border "StatsPanel"
│           │   └── VerticalBox
│           │       ├── "ATTRIBUTES" + GridPanel (Might/Acuity/Resolve/Reflex/Vigor/Presence)
│           │       ├── "COMBAT" + GridPanel (HP, AC, THAC0, XP, Level)
│           │       ├── "SAVING THROWS" + GridPanel (Poison/Wands/Petrify/Breath/Spells)
│           │       └── WrapBox "ConditionsBox"
│           ├── [1] Border "EquipmentPanel"
│           │   └── GridPanel (Weapon/Shield/Armour/Helmet/Ring1/Ring2/Cloak/Misc)
│           ├── [2] Border "InventoryPanel"
│           │   └── ScrollBox + UniformGridPanel "InventoryGrid"
│           ├── [3] Border "SpellsPanel"
│           │   └── ScrollBox + VerticalBox (L1–L7, hidden if MaxSlots==0)
│           └── [4] Border "ThiefSkillsPanel"
│               └── GridPanel (9 skill rows: PickPockets through ReadLanguages)
```

---

## 4. Blueprint Variables

| Variable | Type | BindWidget? | Purpose |
|----------|------|-------------|---------|
| `PartyManagerRef` | BP_PartyManager (Obj Ref) | No | Cached data source |
| `SelectedIndex` | int32 | No | Which party member is active |
| `PartyListVBox` | VerticalBox | Yes | Dynamic character list |
| `TabSwitcher` | WidgetSwitcher | Yes | Switches between 5 panels |
| `Btn_TabStats/Equipment/Inventory/Spells/Thief` | Button ×5 | Yes | Tab buttons |
| `Btn_Close` | Button | Yes | Close button |
| `CharacterNameHeader` | TextBlock | Yes | Selected character's name |
| `ConditionsBox` | WrapBox | Yes | Condition indicators |
| `InventoryGrid` | UniformGridPanel | Yes | Inventory item grid |
| `SpellsVBox` | VerticalBox | Yes | Spell slot container |
| All `Val_*` TextBlocks (~35 total) | TextBlock | Yes | Value display fields |

---

## 5. Blueprint Functions & Logic

### Event Graph

```
Event Construct
  → Initialise(PartyManagerRef)

PartyManagerRef.OnPartyUpdated (bind event)
  → RefreshPartyList
  → If SelectedIndex is valid → PopulateAllTabs
```

### Initialise(PartyManager)
- Store PartyManagerRef
- Bind OnPartyUpdated → RefreshPartyList + PopulateAllTabs
- Call RefreshPartyList

### RefreshPartyList
- Clear Children of PartyListVBox
- For Each PartyMembers → Create Widget (Button)
  - Set button text = CharacterName
  - Set button style = button sprite (existing)
  - Bind OnClicked → OnCharacterSelected(Array Index)
  - Add Child to PartyListVBox
- Auto-select first character if none selected

### OnCharacterSelected(Index)
- Set SelectedIndex = Index
- Call PopulateAllTabs

### PopulateAllTabs
- Get PartyMembers[SelectedIndex] → Break S_Character
- **Stats tab:** Format and set all Val_* TextBlocks
  - HP → `"52 / 52"`
  - Level → `"1"` or `"1 / 3"` if dual-class (Level2 > 0)
  - Equipment → `"—"` when ID == 0
- **Inventory tab:** Clear InventoryGrid, For Each Inventory → Create TextBlock ("Item #X") → Add to grid
- **Spells tab:** For each level L1–L7
  - If SpellSlotsMaxL[N] == 0 → hide entire level row
  - Show `"2 / 3"` format (Current / Max)
  - List memorised spell IDs
- **Thief tab:** Set all 9 skill percentages
- **Conditions:** Clear ConditionsBox → For Each Condition → Create TextBlock (abbreviated name, color from GetConditionColor)
- Call UpdateThiefTabVisibility

### OnTabClicked(TabIndex)
- TabSwitcher.SetActiveWidgetIndex(TabIndex)
- Loop all tab buttons: set active = highlighted (button_ready_on), inactive = dim (button)

### UpdateThiefTabVisibility
- Check Class1 or Class2 == Rogue or Infiltrator
- SetVisibility on Btn_TabThief and ThiefSkillsPanel

### Btn_Close.OnClicked
- Remove from Parent (self)

---

## 6. Integration: Hooking into Exploration HUD

**Modify WBP_ExplorationHUD → Btn_View ("[V] View Party") OnClicked:**

```
Btn_View.OnClicked
  → Create Widget (class = WBP_CharacterScreen)
    → Owning Player: Get Owning Player
  → Add to Viewport
  → CharacterScreenRef.Initialise(PartyManagerRef)
```

---

## 7. Theming (following existing GoldBox palette)

| Element | Color / Sprite |
|---------|---------------|
| Root Canvas | Transparent (modal overlay) |
| Panel backgrounds | `(R=0.04, G=0.03, B=0.02, A=1.0)` — dark brown-black |
| Inner stat panels | `(R=0.05, G=0.05, B=0.04, A=1.0)` — charcoal |
| Header text | `(R=0.55, G=0.48, B=0.30)` — parchment gold |
| Value text | `(R=0.85, G=0.82, B=0.70)` — light parchment |
| Font | Default engine font, Size 14–18 for body, Size 20–24 for headers |
| Active tab button | `/Game/Art/Sprites/button_ready_on` sprite |
| Inactive tab button | `/Game/Art/Sprites/button` sprite |
| Hovered tab button | `/Game/Art/Sprites/button_ready_off` sprite |
| HP bar fill | `(R=0.0, G=0.5, B=1.0, A=1.0)` — blue |
| Condition "Poisoned" | `(R=0.4, G=0.8, B=0.1)` — sickly green |
| Condition "Dead" | `(R=0.6, G=0.1, B=0.1)` — dark red |

---

## 8. Build Order

| Step | What | Dependencies |
|------|------|-------------|
| 1 | Create WBP_CharacterScreen, build all widget hierarchy | None |
| 2 | Add all BindWidget variables | Step 1 |
| 3 | Implement Initialise + RefreshPartyList + OnCharacterSelected | Step 2 |
| 4 | Implement PopulateAllTabs (Stats, Equipment, Inventory, Spells, Thief, Conditions) | Step 3 |
| 5 | Implement tab switching + button highlighting + Thief tab visibility | Step 4 |
| 6 | Wire WBP_ExplorationHUD Btn_View to spawn + initialise WBP_CharacterScreen | Step 5 |

---

## 9. Verification Checklist

- [ ] Press [V] / click View Party → WBP_CharacterScreen opens
- [ ] Thorin (Fighter) displays: Might 17, AC 2, THAC0 50, HP 52/52, Warden class, no Thief Skills tab
- [ ] Mira (Thief/Rogue) displays: Reflex 18, Pick Pockets 30, Climb Walls 85, Thief Skills tab visible
- [ ] Each tab button switches the WidgetSwitcher correctly
- [ ] Active tab button is highlighted, inactive tabs are dimmed
- [ ] Close button removes the widget from viewport
- [ ] Re-opening refreshes data correctly
- [ ] Empty equipment slots show "—"
- [ ] Empty spell levels are hidden
- [ ] Conditions display as colored text

---

## 10. Known Gaps (future work)

- Equipment/inventory/spell DataTables don't exist yet → display raw int32 IDs
- No portrait rendering (Portrait field exists but display not implemented)
- Re-roll stat generation is character-creation only (not in this screen)
- No drag-drop for inventory management
- No equipment comparison tooltips

---

*Document created 7/7/2026 — GoldBoxRPG UE 5.7.4*
