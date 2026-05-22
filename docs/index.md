# Overview

This site is a source-backed reference for the **Trident Slime Ops (TSO)** repo — a collection of Godot 4 prototypes for a Battle Cats–style 2D tower-defense game, plus an integrated build (`Trident_Slime_Ops_Assembly/`) and a team design vault (`OBSIDIAN-TEAM-VAULT/`).

Every claim in this reference cites a `file:line` against current source. Where source has drifted from older planning notes, the drift is flagged in `!!! warning` boxes rather than silently corrected.

!!! note "Where the docs live"
    `TSO-Reference/` is a separate git repo nested inside the prototypes repo, with the parent's `.gitignore` excluding it. The two coexist on disk so that prose references like `Trident_Slime_Ops_Assembly/project.godot:12` resolve for any reader with the prototype repo checked out alongside.

## Engine

- **Godot 4.6** (4.5 for the overworld prototype only), Mobile renderer (D3D12 on Windows), **Jolt** physics.
- GDScript throughout. No C#, no GDExtension.
- Exception: the vendored external study `td-like-game-study/dafans-main/` runs on the **GL Compatibility** renderer.

## Repo Layout

```
tso-prototyping-starting-4-3-26/
├── PROTOTYPES_REPORT.md          # 2026-05-15 source-backed audit (529 lines)
├── NEXT-STEPS.md                 # team next-steps doc
├── tsop---unit-structure/        # Prototype 1 — Unit architecture foundation
├── tso-rushed-stages-showcaes/   # Prototype 2 — Playable stage loop
├── tsop---character-animations/  # Prototype 3 — Animation-relay decoupling
├── tsop---overworld-map/         # Prototype 4 — Path2D world-map navigation
├── from-data-to-stage/           # Prototype 5 — Enemy-wave data format
├── tsop---scene-loading-structure/ # Prototype 6 — Top-level scene manager
├── progression-system/           # Prototype 7 — Macro-progression UI (stub)
├── td-like-game-study/dafans-main/ # Prototype 8 — External Battle Cats clone
├── Trident_Slime_Ops_Assembly/   # Integrated build — Unit Package System
├── OBSIDIAN-TEAM-VAULT/          # Team design / production notes
└── TSO-Reference/                # this site (nested own-repo)
```

## Prototype Maturity Matrix

| Folder | Subsystem | Maturity |
|---|---|---|
| `tsop---unit-structure` | Unit architecture foundation | Working base |
| `tso-rushed-stages-showcaes` | Playable stage loop | Playable, buggy |
| `tsop---character-animations` | Animation-relay decoupling layer | Delivered an artifact |
| `tsop---overworld-map` | Path2D world-map navigation | Working mechanic spike |
| `from-data-to-stage` | Enemy-wave data format | Design-only, no runtime |
| `tsop---scene-loading-structure` | Top-level scene manager | Working spike + debug UI |
| `progression-system` | Macro-progression UI | Empty stub |
| `td-like-game-study/dafans-main` | External Battle Cats clone study | Reference only |
| `Trident_Slime_Ops_Assembly` | **Integrated build** | Inherits proto-1 architecture; adds UnitStatChanger + .tscn package library |
| `OBSIDIAN-TEAM-VAULT` | Team design notes | Production notes + canvases |

Maturity labels are seeded from `PROTOTYPES_REPORT.md` (2026-05-15) and updated where re-verification found drift. The chapter for each prototype re-verifies every concrete claim against current source.

## Reading Conventions

- **Path references** are written as plain `path/to/file.gd:NN` text, not markdown links. This keeps citations layout-independent and editor-friendly (jump-to-file).
- **Code excerpts** are quoted inline in fenced blocks, kept short, and prefixed with the source citation.
- **Bugs and smells** appear in `!!! warning` admonitions next to the code they describe.
- **Drift from planning docs** (vault or `PROTOTYPES_REPORT.md`) is called out in `!!! warning "Drift"` boxes rather than rewritten silently.

## Chapter Map

| # | Chapter | Covers |
|---|---|---|
| 1 | This page | Engine, repo layout, maturity matrix |
| 2 | [Quickstart](00-Quickstart.md) | Reading paths for newcomers |
| 3 | Combat System Addon | `CombatBody2D`, `DamageArea2D`, `DamageReceiver2D` |
| 4 | Unit Hierarchy | `Unit` → `Ally` / `Enemy`, attack & range nodes, `UnitData` |
| 5 | Spawn and Wave System | `UnitSpawnSystem`, `EnemySpawnStartwatch`, `config_1` |
| 6 | Stage Loop | `QuickGameManager`, `PMSV`, deck UI, camera, test units |
| 7 | Animation Relay | `AnimationRelayer`, `_play_anim()` priority gate, `old_shipment/` |
| 8 | Overworld Map | Dual-direction `Path2D`, `IOTile`, cursor stubs |
| 9 | Stage Data Format | `cleanerExampleEnemy` schema, `get_or_add` pattern |
| 10 | Scene Loading Architecture | `TreeOfNodes`, persistent-root concept |
| 11 | Progression System | Stub status, intended scope |
| 12 | External Study GAY-DLE | `pauseable_sleep`, async loops, UID targeting |
| 13 | Trident Slime Ops Assembly | `UnitNodePackage`, `UnitStatChangerClass`, `StateNodeClass` |
| 14 | Team Vault — Production Notes | Unit Package, Menu Nav, Unit Effects, Macro Progression, Coding Chunks |
| 15 | Team Vault — World Map Layouts | Tutorial Beach canvas |
| 16 | Carry-Forward Tiers | Tier 1/2/3 extraction list |
| 17 | Known Bugs Catalog | Every bug cross-linked back to chapters |
| 18 | Glossary | PMSV, TONS, MAOT, IOTile, Strives, Pierce, etc. |
