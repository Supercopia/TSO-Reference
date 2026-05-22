# Progression System

`progression-system/` is an **empty stub**. Documented here for completeness so future engineers don't open it expecting working code.

## What's in the directory

Non-boilerplate files (Godot's standard `.editorconfig` / `.gitattributes` / `.gitignore` / `icon.svg.import` also present):

```
progression-system/
├── project.godot              # 24 lines: [application] + [physics] (Jolt) + [rendering] (D3D12, mobile)
├── all_interfaces_screen.tscn # one Control → one Label
└── icon.svg                   # default Godot icon
```

The single scene `all_interfaces_screen.tscn` (UID `uid://hxv8fbg67hjn`) is a `Control` with a child `Label`. The label text reads, verbatim across two lines (including the misspelling `waht`):

> "All Interfaces load from here
> help me I have very little idea waht I'm doing-"

`project.godot` has no `run/main_scene` set — opening the project in Godot launches into an empty editor with no entry point. The physics and rendering sections are full Godot defaults, identical to the other 4.6 prototypes (Jolt Physics, D3D12, Mobile renderer).

## What it was supposed to be

Per `MEMORY.md` and the team vault's Menu Navigation plan, this prototype was the placeholder for the **Macro Progression** UI cluster — the system that would have housed:

- Loadout Menu (8-slot deck editor)
- Upgrades Screen (per-Slime EXP spending, refund returns 75%)
- Strives & Forms Menu (Strive unlocks, EX/DX Form switching)

The full design for this lives in [Team Vault — Production Notes](12-Vault-Production-Notes.md), specifically the **Macro Progression Matters** doc and the **Menu Navigation Implementation Plan**. None of that design has been translated into code anywhere — neither in this prototype nor elsewhere.

## Where the actual work lives

| Concern | Where it lives today |
|---|---|
| Mid-stage slot UI | `Level_Functions/refurbished_player_deck/` in proto-2 and proto-1 (`p_deck_controls.gd`, `card_slot_behavior.gd`) — see [Stage Loop](04-Stage-Loop.md) |
| Save persistence | `addons/godot_save_load_manager/` + `qgm.save_progress` in proto-2 — see [Stage Loop](04-Stage-Loop.md) |
| Stat changers (early Strive prototypes) | `Trident_Slime_Ops_Assembly/.../UnitStatChangerClass.gd` + Carefulness / DamageUp / HeavyPacking — see [Assembly](11-Assembly.md) |
| Macro Progression design spec | `OBSIDIAN-TEAM-VAULT/Production Notes/Macro Progression Matters.md` |

## Where it lives

`progression-system/` only. Nothing references this prototype from elsewhere; it can be deleted at any time without breaking anything.

## Carry-forward verdict

**Discard.** Recreate fresh when building the real progression UI per the vault's design. The empty `Control` → `Label` scene contains no salvageable structure.

The vault's design work (Macro Progression + Menu Navigation) is the carry-forward, not this directory.
