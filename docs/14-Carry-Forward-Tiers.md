# Carry-Forward Tiers

Aggregated extraction list. Three tiers:

- **Tier 1** — lift as-is (or near-as-is). Works, well-shaped, can be a foundation for whichever project ends up being the unified TSO build.
- **Tier 2** — lift after a small rewrite. Right pattern, has fixable issues.
- **Tier 3** — don't lift; rebuild fresh. Either superseded, broken, or design-only.

The "where" column points to the canonical home today. The "fix-before-port" column flags issues that block Tier 1 status.

## Tier 1 — lift as-is

| Item | Where | Notes |
|---|---|---|
| `Combat_system` addon (CombatBody2D, DamageArea2D, DamageReceiver2D) | `addons/Combat_system/` in any project | Cleanest sub-system in the repo. See [Combat System Addon](01-Combat-System-Addon.md). |
| `UnitNodePackage.find_unit()` + the permanent-package pattern | `tsop---unit-structure/.../UnitNodePackage.gd` and Assembly | The central enabling pattern for composable units. See [Unit Hierarchy](02-Unit-Hierarchy.md), [Team Vault — Production Notes](12-Vault-Production-Notes.md). |
| `UnitClass.gd` (the rewrite — proto-1 / Assembly) | `tsop---unit-structure/.../UnitClass.gd`; Assembly's variant | Replaces proto-2's `Unit.gd`. State machine, attack pipeline, knockback skeleton, animation priority gate. |
| `StateNode` + `MoonwalkState` base pattern | proto-1 and Assembly | Composable Special States via `call_new_state.emit("name")`. |
| Unit Package `.tscn` library (Hurtbox / Movement / Attacking / HealthDisplay) | `Trident_Slime_Ops_Assembly/.../Unit_Parts/node-packages/` | Concrete reusable composition. See [Assembly](11-Assembly.md). |
| `IOTile` (Interactive_Overworld_Tile.gd) | `tsop---overworld-map/.../Interactive_Overworld_Tile.gd` | Cleanest single script in the repo. 14 lines. Zero bugs. See [Overworld Map](06-Overworld-Map.md). |
| `UnitSpawnSystem.gd` (queue + per-frame drain) | All three combat projects | Queue logic is sound. Caveat: units added as children of autoload — fix when porting. See [Spawn and Wave System](03-Spawn-and-Wave-System.md). |
| `qgm.save_progress()` + `SaveHandler` autoload + `godot_save_load_manager` addon | `tso-rushed-stages-showcaes/` | Recently added; not yet in Assembly. See [Stage Loop](04-Stage-Loop.md). |
| Pause menu (`ui/pause_menu.gd`) | `tso-rushed-stages-showcaes/ui/` | Recently added; drop-in. See [Stage Loop](04-Stage-Loop.md). |
| `pauseable_sleep(owner, sec, cb)` autoload pattern | `td-like-game-study/dafans-main/Scripts/time.gd` | Highest-value GAY-DLE port — makes async logic pause-aware. See [External Study — GAY-DLE](10-External-Study-GAYDLE.md). |
| Async `await` loops for periodic background work | GAY-DLE `battlefield.gd`, `enemy_base.gd` | Replaces Timer-node-plus-signal boilerplate. See [External Study — GAY-DLE](10-External-Study-GAYDLE.md). |
| UID-based targeting (custom counter pattern) | GAY-DLE `battlefield.gd:71-73`, `enemy_base.gd:11, 83` | Replaces stale-reference `Array[Unit]` patterns. See [External Study — GAY-DLE](10-External-Study-GAYDLE.md). |
| Dictionary-driven slot configuration | GAY-DLE `character_slots_container.gd:1-66` | Direct fix for TSO's broken `request_ally` slot lookup. See [External Study — GAY-DLE](10-External-Study-GAYDLE.md). |
| Per-slot cooldown via tween + progress bar | GAY-DLE `character_slots_container.gd:89-108` (spawn function) | Direct model for TSO's Lite/Heavy/Income cooldown buttons (currently stubbed in `p_deck_controls.gd`). See [External Study — GAY-DLE](10-External-Study-GAYDLE.md). |
| Wallet leveling system | GAY-DLE `battlefield.gd:68-69` (`get_value_at_specific_cut`) | Iteration 1 Step 4 of vault plan. Replaces `CPC += 1`. See [External Study — GAY-DLE](10-External-Study-GAYDLE.md). |
| Collision layer convention | Documented in vault, codified in 3 code locations | Layers 1-3 ally, 5-7 enemy; 4 and 8 reserved. See [Team Vault — Production Notes](12-Vault-Production-Notes.md). |
| `get_or_add` pattern (built-in `Dictionary` method) | Built into Godot 4; demoed by `get_or_add_flex()` in `from-data-to-stage/StageDataClass.gd:144` | Safe-default merge for nested dicts. Drop-in. See [Stage Data Format](07-Stage-Data-Format.md). |
| All four Unit Package System vault docs | `OBSIDIAN-TEAM-VAULT/Production Notes/Unit Package System - *.md` | Architecture Fix + Developer Guide + Collision Layers + Removed Comments Reference. |
| Menu Navigation Implementation Plan | `OBSIDIAN-TEAM-VAULT/Production Notes/Menu Navigation - Implementation Plan.md` | The merge roadmap. See [Team Vault — Production Notes](12-Vault-Production-Notes.md). |
| Unit Effects design spec | `OBSIDIAN-TEAM-VAULT/Production Notes/Unit Effects.md` + canvas | Design for the eventual Status Node system. |
| Tutorial Beach canvas | `OBSIDIAN-TEAM-VAULT/World Map Layouts/Tutorial Beach.canvas` | Spec for the first chapter overworld. See [Team Vault — World Map Layouts](13-Vault-World-Map-Layouts.md). |
| `AnimationRelayer` pattern | `tsop---character-animations/old_shipment/unit_animations/AnimationRelayClass.gd` | The decoupling layer the Unit code expects. **Fix-before-port:** 4 bugs (null-deref, bare-return, dead signal, disabled play paths). See [Animation Relay](05-Animation-Relay.md). |
| `Carry-forward Tiers` list itself | This chapter | Update when porting tasks complete. |

## Tier 2 — lift after rewrite

| Item | Where | Why not Tier 1 |
|---|---|---|
| `OverworldPlayerPath.gd` (dual-direction Path2D) | `tsop---overworld-map/` | 42.0 magic threshold, hardcoded 0.05 literal vs `speed` var, unguarded `get_parent()`, unbounded `add_exception` accumulation. The pattern is right; the implementation needs tightening. |
| `EnemySpawnStartwatch` + `config_set_off_all_timers` | `tso-rushed-stages-showcaes/` | Works, but `Unready = true` default is a footgun — `config_1.tscn` ships broken. Flip default to `false` before porting. |
| `UnitStatChangerClass` | Assembly | Three bugs (dead-control-flow `has()`, wrong-keys `range` branch, int-truncate). Right idea, wrong implementation. See [Assembly](11-Assembly.md). |
| `TreeOfNodes` / TONS pattern | `tsop---scene-loading-structure/` | `name.contains()` dispatch is brittle. Need a central scene-manager autoload to replace the per-node dispatch logic. See [Scene Loading Architecture](08-Scene-Loading-Architecture.md). |
| Debug UI layer (`DebugButtonLine`, `MenuStructureVisual`) | `tsop---scene-loading-structure/` | Useful as scaffold for a real debug toolbar; not shippable as-is. |
| Camera nav (proto-2's 4 scripts) | `tso-rushed-stages-showcaes/Gameplay_Display/` | Tightly coupled to `qgm.Menu_Camera` bool. Lift only with the menu-to-stage transition. |
| `StageDataClass` schema design | `from-data-to-stage/` | Reconcile the two schemas (`OneEnemyExample` vs `cleanerExampleEnemy`), decide `HeadsUpBeforeSpawn` array vs scalar, then implement the dispatcher. See [Stage Data Format](07-Stage-Data-Format.md). |
| Boss intro / "ulti" pattern (await chain in `enemy_base.gd`) | GAY-DLE | Pattern model for TSO's first boss; doesn't apply directly. See [External Study — GAY-DLE](10-External-Study-GAYDLE.md). |
| `velocity = speed * direction + move_and_slide()` (GAY-DLE claim) | GAY-DLE (unverified) | Needed to fix TSO's `MAOT()` `position.x +=`. Verify against GAY-DLE source first. |
| `Async tween → AoE` pattern (`cannon_charger.gd`) | GAY-DLE | Useful for charge attacks once that mechanic is being built. |

## Tier 3 — discard / rebuild fresh

| Item | Where | Why |
|---|---|---|
| `tso-rushed-stages-showcaes/.../Unit.gd` (the older 163-line version) | proto-2 | Superseded by `UnitClass.gd`. Still has every "What's broken" bug from the May 15 report. Don't lift; document its bugs in [Known Bugs Catalog](15-Known-Bugs-Catalog.md). |
| All proto-2 child-node scripts (`AttackNode.gd`, `HitboxDamageArea2D.gd`, `RangeDetectors.gd`) | proto-2 | Same — still have the `get_class() != "CharacterBody2D"` self-destruct bugs. The proto-1 / Assembly versions fix them via `UnitNodePackage.find_unit()`. |
| `progression-system/` entire prototype | as named | Empty stub. Rebuild fresh against the vault's Macro Progression spec. See [Progression System](09-Progression-System.md). |
| `Concrete stat-changers (Carefulness, DamageUp, HeavyPacking)` | Assembly | Early Strive prototypes. Re-implement from the vault's Strive design when Strives are properly specced. |
| `temporary_test_scene.tscn` (Assembly main scene) | Assembly | Bare arena, two pre-placed units. Replace with the `battlefield.tscn` described in Iteration 1 Step 6. |
| `config_1.tscn` specifically | proto-2 | All six timers have `Unready = true`; the scene spawns nothing. Pattern is keep-able, the file isn't. |
| `cuteProjectorAnimV0.gd` orphaned signal | `tsop---character-animations/old_shipment/` | Orphaned signal, never connected. |
| `akds_StatesAnimPlayer.gd` | `tsop---character-animations/` | 70 lines of design comments, no code. |
| `old_shipment/` archived slime `.tscn`s (5 generations) | `tsop---character-animations/` | Reference / archaeology. Don't lift. But `AnimationRelayClass.gd` inside this archive **is** a Tier 1 item that needs to be promoted out. |
| `dafans-main/.godot/` cache (41 MB) | `td-like-game-study/dafans-main/` | Committed binary cache. Optional `.gitignore` cleanup. |
| `dafans-main/Global.gd` fullscreen toggle | GAY-DLE | Project-specific input handling; not portable. |
| `dafans-main/character_slots_container.gd:84-85` 5-column-grid assumption | GAY-DLE | Silently breaks on resize. Pattern (dictionary slot config) is Tier 1; the specific code isn't. |
| `time.gd` / `random.gd` (the rest of GAY-DLE's utility scripts) | GAY-DLE | TSO can implement equivalents; no need to drag in unrelated GAY-DLE utils. |

## Items not yet in any project

The vault designs these but no code exists yet:

- Stage data format dispatcher (the [Stage Data Format](07-Stage-Data-Format.md) schema → runtime)
- Status Node / Effect Node runtime (Unit Effects spec)
- Overworld Map scene-loading handoff (post-`update_level_info` printing)
- Loadout save system (named slots persisted to `user://`)
- Macro Progression UI (Loadout / Equip / Upgrades / Strives & Forms screens)
- Wallet leveling on PMSV
- Pause menu in Assembly (built in proto-2, not yet ported)
- `qgm` autoload in Assembly (built in proto-2, not yet ported)
- Bases in Assembly (built in proto-2, not yet ported)
- Wave Timers in Assembly (built in proto-2, not yet ported)
- Overworld integration in Assembly (built in `tsop---overworld-map/`, not yet ported)
- Animation Relay layer in proto-1 or Assembly (lives only in `tsop---character-animations/old_shipment/`; needs to be promoted out of the archive and integrated into Assembly's empty `AnimationPlayer` slots)
- Score system and S-Rank requirements (vault Iteration 3 — explicitly deferred)
- Unit Swap with Swap Batteries (vault Iteration 3 — explicitly deferred)
- Global Shop currency system (TBD per vault)
- Campaign Log upload (TBD per vault)

## Priority of porting work

Following the vault's Menu Navigation plan, the order should be:

**Iteration 1 (playable loop)** — port Tier-1 items into Assembly:

1. `EnemySpawnStartwatch` + `config_set_off_all_timers` (Tier 2 — fix `Unready` default during port)
2. Bases (`ally_base_go`, `enemy_base_go`, `on_base_death_node`, `waiting_base_for_menu`)
3. `qgm` autoload + save/load + pause menu
4. Wallet leveling (replaces `CPC += 1`)
5. Main menu, game-over, pause panel scenes
6. New `battlefield.tscn` replacing `temporary_test_scene.tscn`

**Iteration 2 (navigation)** — Tier 1 + Tier 2 items:

7. Finish Overworld cursor pathfinding (Tier 2)
8. Build Control Room scene (new)
9. Build Preparation Menu scene (new)
10. Loadout save system

**Iteration 3 (polish — deferred)** — only after the game's first content chapter is built:

11. S-Rank, Score, Unit Swap, Campaign Log, full Strive pricing, Global Shop

Concurrent with Iteration 1: promote `AnimationRelayClass.gd` out of `old_shipment/` and integrate into Assembly's empty `AnimationPlayer` slots. Concurrent with Iteration 2: pull `pauseable_sleep`, UID targeting, and dictionary slot config from GAY-DLE.
