# GoldBox RPG — UE5 Naming Conventions

## Asset Prefixes

| Asset Type           | Prefix  | Example                  |
|----------------------|---------|--------------------------|
| Blueprint Actor      | BP_     | BP_GameManager           |
| Widget Blueprint     | WBP_    | WBP_PartyPanel           |
| Struct               | S_      | S_Character (or SCharacter in BP) |
| Enum                 | E_      | E_GameState              |
| Data Table           | DT_     | DT_Monsters              |
| Static Mesh          | SM_     | SM_Wall_Stone            |
| Material             | M_      | M_WallStone              |
| Sound Cue            | SC_     | SC_SwordHit              |
| Texture              | T_      | T_WallStone              |
| Map / Level          | (none)  | TestDungeon              |

## Blueprint Variable Names
- Use PascalCase: CurrentHP, PartyMembers, IsPlayerControlled
- Booleans prefix with b: bIsAlive, bIsPlayerControlled
- Arrays suffix with s: PartyMembers, Conditions

## Function Names
- Use PascalCase verbs: ResolveAttack, ComputeTHAC0, GetLivingMembers

## Notes
- All Blueprint assets live in Content/Blueprints/
- All Data Tables live in Content/Data/DataTables/
- All Widget Blueprints live in Content/Blueprints/UI/