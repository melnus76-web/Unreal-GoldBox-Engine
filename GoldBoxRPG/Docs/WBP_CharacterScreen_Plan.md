# WBP_CharacterScreen — Build Plan
## User Story GB-052 / EPIC 012 — Character Management

> **Build Progress (2026-07-12):** Widget tree fully built with all 5 tabs. Stats and Equipment tabs populated via RefreshCharacterDetails. Party list with dynamic WBP_PartySlot creation working. Tab switching wired. Remaining: Inventory/Spells/ThiefSkills detail population, tab button highlighting, Thief tab class visibility gate.

---

## 1. What We're Building

A tabbed character sheet widget (`WBP_CharacterScreen`) with five tabs:

| Tab | Content | Status |
|-----|---------|--------|
| **Stats** | Ability scores, HP, AC, THAC0, XP, Level, Saving Throws, Active Conditions | ✅ Populated |
| **Equipment** | 8 equipment slots (Weapon, Shield, Armour, Helmet, Ring ×2, Cloak, Misc) | ✅ Populated |
| **Inventory** | Scrollable grid of carried item IDs | ❌ Widget tree only |
| **Spells** | Spell slots per level (L1–L7), memorised spell lists | ❌ Widget tree only |
| **Thief Skills** | 9 skill percentages — visible only for Rogue/Infiltrator | ❌ Widget tree only |

---

## 2. Data Source

**Primary struct:** `/Game/Blueprints/Structs/S_Character` (64 fields)

**Key fields by tab:**

- **Ability scores:** Might, Acuity, Resolve, Reflex, Vigor, Presence
- **Combat:** CurrentHP, MaxHP, DefenseRating (=AC), StrikeNumber (=THAC0), XP, Level, Level2
- **Saves:** FortitudeSave, ReflexSave, ResolveSave
- **Equipment:** EquipWeapon, EquipShield, EquipArmour, EquipHelmet, EquipRing1, EquipRing2, EquipCloak, EquipMisc (all int32 IDs)
- **Inventory:** Inventory (TArray<int32>)
- **Spells:** SpellSlotsCurrentL1–L7, SpellSlotsMaxL1–L7, MemorisedSpellsL1–L7
- **Thief skills:** SkillPickPockets, SkillOpenLocks, SkillFindTraps, SkillRemoveTraps, SkillMoveSilently, SkillHideInShadows, SkillHearNoise, SkillClimbWalls, SkillReadLanguages
- **Conditions:** Condition (TArray<E_Condition>), E_Condition enum: Normal, Restrained, Paralysed, Poisoned, Blinded, Quickened, Slowed, Diseased, Sapped, Petrified, Dead, Unconscious

**Runtime data source:** `BP_PartyManager` actor → `PartyMembers` (TArray<S_Character>) + `OnPartyUpdated` delegate

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
