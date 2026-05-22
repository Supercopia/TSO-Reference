# Quickstart

This chapter answers **"where do I start?"** for four common kinds of reader. Each path is a short ordered list of chapters plus the source files that earn the most from a first read.

## If you are joining the team

You want a working mental model of what's been built, what works, and what's planned.

1. **[Overview](index.md)** — engine, repo layout, the maturity matrix.
2. **Trident Slime Ops Assembly** — the architecture target. Carries the newest combat code (UnitClass, Unit Package System, StateNode, UnitStatChanger) but boots into a bare two-unit arena — no waves, no bases, no win/loss.
3. **Stage Loop** — the playable experience today lives in `tso-rushed-stages-showcaes/`. Pause menu, save/load, bases, win/loss. The gameplay-loop work that hasn't yet been ported into Assembly.
4. **Team Vault — Production Notes** — the team's design vocabulary (Unit Package System, Unit Effects, Macro Progression) lives in the vault.
5. **Glossary** — skim once so `PMSV`, `TONS`, `MAOT`, `IOTile`, *Strives*, *Foreswing/Backswing*, *Pierce* stop being opaque.
6. **Known Bugs Catalog** — the shape of the rough edges across prototypes.

Hands-on: open `Trident_Slime_Ops_Assembly/project.godot` in Godot 4.6 and run the main scene to see the current combat architecture. Open `tso-rushed-stages-showcaes/project.godot` to see the playable gameplay loop.

## If you are extending the combat system

You are adding a unit, a weapon, a damage type, a status effect, or a stat changer.

1. **Combat System Addon** — the `CombatBody2D` / `DamageArea2D` / `DamageReceiver2D` triangle is the lowest-level invariant. Read this first; everything above depends on it.
2. **Unit Hierarchy** — `Unit` → `Ally` / `Enemy`, `AttackNode`, `RangeDetectors`, `HitboxDamageArea2D`, `UnitData`.
3. **Trident Slime Ops Assembly** — three composition layers introduced in Assembly: the **Unit Package System** (`UnitNodePackage` + the four `.tscn` packs: Hurtbox / Movement / Attacking / HealthDisplay) replaces hard-coded subclassing; the **state-machine layer** (`StateNode` + `MoonwalkState`) lets Special States drop in as child nodes; the **stat-changer layer** (`UnitStatChangerClass` + Carefulness / DamageUp / HeavyPacking) modifies stats post-UnitData. New units should be built with packages, not by extending `Ally`.
4. **Team Vault — Production Notes** — the four "Unit Package System — *" docs (Architecture Fix, Collision Layers, Developer Guide, Removed Comments Reference) plus *Unit Effects*.
5. **Known Bugs Catalog** — search for the system you are touching before you commit.

!!! note "Combat System ships in all three projects, kept in sync"
    The `addons/Combat_system/` directory exists under `tsop---unit-structure/`, `tso-rushed-stages-showcaes/`, and `Trident_Slime_Ops_Assembly/`. A `diff -r` finds the three copies byte-identical. Edits should land in all three at once.

## If you are building stages, waves, or a world map

You care about content tooling: spawning enemies on a timeline, laying out a stage, or moving the player across an overworld.

1. **Spawn and Wave System** — `UnitSpawnSystem`, `EnemySpawnStartwatch`, `config_set_off_all_timers`, `config_1`. The runtime side.
2. **Stage Data Format** — `cleanerExampleEnemy` schema, `StartCondition`, `HeadsUpBeforeSpawn`, `SpawningTimes`. The design-time side. Note the dispatcher is not yet implemented.
3. **Stage Loop** — `QuickGameManager`, `PMSV`, deck UI, camera nav. How a stage actually plays.
4. **Overworld Map** — `IOTile`, `OverworldPlayerPath`, dual-direction `Path2D` traversal.
5. **Team Vault — World Map Layouts** — the Tutorial Beach canvas.

!!! note "`config_1` is currently dead"
    Every `EnemySpawnStartwatch` in `config_1.tscn` defaults `Unready = true` and none override it, so all timers `queue_free` themselves on `_ready()` and spawn nothing. See **Spawn and Wave System** and the **Known Bugs Catalog**.

## If you are porting patterns from GAY-DLE

You read about `dafans-main` and want to lift specific ideas.

1. **External Study: GAY-DLE** — the chapter walks through `time.pauseable_sleep`, async `await` loops, UID-based targeting, dictionary-driven slot config, and physics-correct movement.
2. **Carry-Forward Tiers** — which patterns are Tier 1 (port now), Tier 2 (port when ready), Tier 3 (study only).
3. **Unit Hierarchy** — the target for most ports (the two-timer attack dance, `position.x +=` movement, broken `request_ally` slot lookup).

## Where the source-of-truth files are

- `PROTOTYPES_REPORT.md` (repo root, 529 lines) — the 2026-05-15 audit. This reference re-verifies every claim against current source rather than copying it.
- `NEXT-STEPS.md` (repo root) — team next-steps.
- `OBSIDIAN-TEAM-VAULT/` — production notes and `.canvas` diagrams. Open with Obsidian for the canvases; the **Team Vault** chapters reconstruct them in prose.
- `Trident_Slime_Ops_Assembly/Combat_DEPENDENCY_README.txt` — quoted verbatim in the Assembly chapter.
- `Trident_Slime_Ops_Assembly/functionality/Combat_Units_System/Unit_Parts/node-packages/Unit-Node-Packages-README.txt` — quoted verbatim in the Assembly chapter.

## Building this site locally

```bash
pip install mkdocs-material
cd TSO-Reference
mkdocs serve   # http://127.0.0.1:8000
mkdocs build   # outputs to site/
```
