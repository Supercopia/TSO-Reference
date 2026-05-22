# Glossary

Vocabulary used throughout the TSO codebase and design notes. Where a term has a specific code definition, the `file:line` citation is given. Where it lives only in vault / design space, the source doc is cited.

## Autoloads (short names)

| Short | Full | Role | Where |
|---|---|---|---|
| `PMSV` | `PlayerMidStageVault` | Mid-stage player state: unit slots, currency, `request_ally(slot)` | All combat projects |
| `qgm` | `QuickGameManager` | Level state, `levels_beaten` dict, `declare_winner(who)`, `Menu_Camera` bool | proto-2 only |
| `UnitSpawnSys` | `UnitSpawnSystem` | Dual-side spawn queue (ally + enemy), drained each physics frame | All combat projects |
| `SaveHandler` | (from `godot_save_load_manager` addon) | Disk persistence for `qgm.levels_beaten` | proto-2 only |
| `no_camera_solver` | (same name) | Creates a fallback Camera2D at zoom 1.5 if none exists | proto-1 only |
| `AutoAuto` | `LayeredAutoAutoload` | Stub for consolidating future pseudo-autoloads | proto-1 only |
| `time` | (GAY-DLE only) | `pauseable_sleep(owner, sec, cb)` and `sleep(sec)` | `td-like-game-study/dafans-main/Scripts/time.gd` |
| `Global` | (GAY-DLE only) | `victory: bool` + fullscreen toggle | `td-like-game-study/dafans-main/Scripts/Global.gd` |
| `Audio` | (GAY-DLE only) | Fire-and-forget SFX | `td-like-game-study/dafans-main/Scripts/audio.gd` |

## Currency and stats

- **`CPC`** — *Current Player Currency*. `int` on `PMSV`. Incremented by 1 each physics frame (60/s at 60fps). Vault calls for replacing this with wallet-leveling. `PlayerMidStageVault.gd:30` (Assembly).
- **`current_player_currency`** — Sibling variable in PMSV. Set once on `_ready` from CPC, never updated. Diverges from CPC immediately. `PlayerMidStageVault.gd:23` (Assembly; proto-2 has it at `:25`). See bug #30 in [Known Bugs Catalog](15-Known-Bugs-Catalog.md).
- **`SP`** — The in-game label for currency. Rendered as `"SP: " + str(PMSV.CPC)` in `currency_display_label.gd`.
- **`Stat_Magnification`** — Percent multiplier on `max_health` and `current_damage` in `Enemy.gd`. 100 = baseline, 200 = doubled, 50 = halved. `Enemy.gd:4`.
- **Sapphires** — Vault currency. Used to buy new Units from the World Map. Duplicate purchase yields a Training Clock. Source: [Tutorial Beach canvas](13-Vault-World-Map-Layouts.md), Notes block.
- **Training Clock** — First-time stage reward. Spent on a Unit, grants 1 Training Point. Vault: `Macro Progression Matters.md`.
- **Training Point (TP)** — Spent on a Trainable Strive to make it free during stages. Or per Tutorial Beach Notes: spent to keep one Trainable Strive permanently active.

## Combat design vocabulary

- **Foreswing** — Wind-up portion of an attack, before damage hits. Returned from `Unit._begin_attack()`. `UnitClass.gd:264`.
- **Backswing** — Recovery portion after damage hits. Part of the `[backswing, cooldown]` array returned from `Unit._attack_continues()`. `UnitClass.gd:268`.
- **Midswing** — (Vault concept) The hitbox-active window between foreswing and backswing. Typically instant; called "Hit Duration" in code as `Attack_Hit_Duration` (`UnitDataClass.gd:28`).
- **`Attack_Hit_Kilter`** — Exported in `UnitData` (`UnitDataClass.gd:24`). NOT WIRED YET per the source comment.
- **`Attack_Hit_Distance`** — Hitbox forward distance from the unit. Exported, conditionally wired via `Use_Hit_Distance` toggle. `UnitDataClass.gd:23`.
- **Pierce** — `DamageArea2D` carries a pierce counter; `DamageReceiver2D` asks `using_pierce_on()` to decide whether to consume one and let the hit through. `HitboxDamageArea2D.gd:53-105`.
- **`MAOT()`** — Acronym never expanded in source; reads as "Move At Other Team". The movement step on `Unit.gd` / `UnitClass.gd`. Currently uses `position.x +=` instead of `velocity` + `move_and_slide()`. `UnitClass.gd:208-211`. See bug #5 in [Known Bugs Catalog](15-Known-Bugs-Catalog.md).
- **`Overide_Damage_Receiving`** — Boolean on `CombatBody2D` that blocks damage when `true`. Spelled `Overide` (one r) in source; baked into every `.tscn`. `combat_body_2d.gd:6, 31`.

## Strives system

- **Strive** — Per-Unit ability upgrade. Optional, persists globally for that Unit. Bought with stage currency, refundable at 50%. Each purchased Strive makes remaining ones 25% more expensive. Source: vault `Menu Navigation - Implementation Plan.md`, Macro Progression spec.
- **Strive Slot / Strive Capacity** — Number of Strives a Unit can equip simultaneously. Starts at 2 at Level 1; +1 at Levels 6, 11, 16, 21, 26 (per Macro Progression per-level entries). Endgame total: 7 slots.
- **Trainable** — Strive keyword. Spending a Training Point makes the Strive free of SP cost during stages.
- **Priceless** — Strive keyword. Cannot be activated with SP. Only usable through other means (TP, status effects).
- **Overtime** — Strive keyword. TP spent yields a free copy instead of reducing price.
- **Hardworking** — Strive keyword. Cannot equip unless TP spent. More TP → free.
- **X Form Only** — Strive keyword. Form-locked. Equip on other Forms gives +1% to all stats consolation.
- **The Bottom 1%** — Universal "Strive available to everyone" giving +1% to all stats. Available always. ("the original effects was a 0% increase to every stat. Lucky you.")
- **`Strive_N_Activated`** — Three boolean exports on `Ally.gd` in proto-2 (`Ally.gd`, since dropped in proto-1). Stubs.
- **`StrivesMenuOpen`** — Boolean on `p_deck_controls.gd` that gates whether the deck routes slot-click input to the Strives menu instead of `request_ally`. `p_deck_controls.gd`.
- **`MenuOpenable`** — Sibling boolean controlling whether the strives-menu-trigger button toggles the strives menu open at all. Both flags are reset to `false` in `_ready()` of proto-1, overriding inspector values — see bug #33 in [Known Bugs Catalog](15-Known-Bugs-Catalog.md).

## Overworld

- **`IOTile`** — `class_name` of `Interactive_Overworld_Tile.gd`. Marker2D-derived. Holds `@export var stage_name` and `@export var stage_scene`. Duck-typed `update_level_info(name, scene)` is called when a cursor raycast hits it. `Interactive_Overworld_Tile.gd:2`.
- **`OverworldPath2D`** — Conceptual name for `OverworldPlayerPath.gd extends Path2D`. Carries a `PathFollow2D` cursor along a curve via `progress_ratio`. Supports dual-direction traversal: uppercase axis names (`North/East/South/West`) for Start→End, lowercase for End→Start. `OverworldPlayerPath.gd:8-17`.
- **loopdeloop** — Name of the 9-point showcase curve in `tsop---overworld-map/test_scenes/overworld_map_test_1.tscn` demonstrating non-monotonic Path2D traversal.
- **Spot** — Vault term for a tile on the Overworld Map. Two kinds: Level spots (load into Preparation Menu) and Non-Level spots (Open Base, B-Switch???, Command Room/Menu Base). Source: `Menu Navigation - Implementation Plan.md`.

## Scene loading

- **`TONS`** — *Tree Of Nodes*. The persistent-root architecture concept: a `SLIMEGAME_ROOT` Node with three top-level branches (`TONS_Combat` / `TONS_WorldMap` / `TONS_MenuTremendo`), one active at a time. Vault canvas legend uses the same names.
- **`TreeOfNodes`** — `class_name` of `TreeOfNodes.gd`. The lazy scene-branch loader attached to each TONS_* node. `tsop---scene-loading-structure/.../TreeOfNodes.gd:2`.
- **`ThisTreeSpawns`** — Exported `PackedScene` on each `TreeOfNodes` node. The scene this branch will instantiate when activated.
- **`isActive`** — Boolean on `TreeOfNodes` tracking whether the branch's `child_scene` is currently spawned.
- **`debug_key_N`** — Named input actions (`debug_key_0` through `debug_key_9`) defined in `tsop---scene-loading-structure/project.godot:20-68`. Bound to numpad and number-row keys. Used both by `TreeOfNodes` and `debug_button_line.gd`.

## Stage data format (`from-data-to-stage` prototype)

- **`StartCondition`** — `[kind, args...]` array. Source examples: `["type", 100, 0]` (template in `cleanerExampleEnemy:83`) and `["OnBaseHealth", 100]` (in `OneEnemyExample`). Three condition kinds exist as `pass`-stub functions: `OnBaseHealth`, `OnObjectHealth`, `OnBossHealth`. No dispatcher implemented. See [Stage Data Format](07-Stage-Data-Format.md).
- **`HeadsUpBeforeSpawn`** — Per the function signature and source comment: `[count, time_before]` array. In the current `cleanerExampleEnemy` schema it's the scalar `0`. Inconsistency flagged in [Stage Data Format](07-Stage-Data-Format.md).
- **`SpawningTimes`** — `[delay, interval]` array per the source comment: `[0]` = delay before the first spawn, `[1]` = interval between spawns.
- **`Magnification`** — Per-enemy stat magnification. `OneEnemyExample` treats it as an int percent (like `Stat_Magnification`); `cleanerExampleEnemy` uses the float `1.00`. Schema drift between the two.
- **`UseNiche`** — Boolean in `cleanerExampleEnemy`. Gates the "niche" fields that follow it; concept not further documented in source.
- **`OnVisualLayer`** — Int, Z-index sorting layer for the spawn relative to the Spawn Node.
- **`SpecialSpawnAnim`** — Int (`-1` = none). Names an animation played before the unit enters play; the handler function is a `pass`/early-return stub.
- **`SchematicForLevel`** — `@export`, declared, never read. Dead.
- **`testDictionaryDouble`** — `@export`, declared, never read. Dead.

## Unit Package System (Assembly-era)

These class names appear in `Trident_Slime_Ops_Assembly/.../Scripts/of-new-classes/` and are documented in the vault.

- **Unit Package** — A `.tscn` you instantiate as a child of a `Unit` to grant it a capability. Examples: `give_hurtbox_pack.tscn`, `allow_attacking_pack.tscn`, `allow_movement_pack.tscn`, `display_unit_health_pack.tscn`. See [Assembly](11-Assembly.md).
- **`UnitNodePackage`** — `class_name` for the base script of those `.tscn` files. `Node2D`-derived. Owns the static helper `find_unit()` that child scripts use to locate their host Unit by walking parents. `UnitNodePackage.gd:3`.
- **`find_unit(from: Node) -> Node`** — Static method on `UnitNodePackage`. Walks `from.get_parent()` upward until a `CharacterBody2D` ancestor is found. Returns that node or `null`. The central enabling pattern of the Unit Package System.
- **`UnitStatChangerClass`** — Modifies unit stats (damage, range, speed) via flat additions and multiplicative scaling. Three concrete subclasses ship: `Carefulness`, `DamageUp`, `HeavyPacking`. `UnitStatChangerClass.gd:1`.
- **`StateNode`** — `class_name` for base of state-machine state nodes. Connects to a host Unit's `parent_transition_to` via `call_new_state` signal. `StateNodeClass.gd:1`.
- **`call_new_state`** — Signal emitted by a `StateNode` carrying the target state's name (a StringName matching the target node's `name`). `"none"` is the sentinel for "exit the state machine".
- **`MoonwalkState`** — Only concrete `StateNode` in the repo; present in both proto-1 (`Unit_System/scripts/child_nodes/MoonwalkState.gd`) and Assembly (`functionality/Combat_Units_System/Scripts/one-offs/MoonwalkState.gd`). Triggered by Backspace; slides the unit backward for 60 ticks; emits `call_new_state.emit("none")`.

## Forms and progression

- **Form** — A Unit's visual / mechanical variant. Three categories per Unit (where applicable): **Base**, **Extra (EX)** (unlocks at Level 10), **Deluxe (DX)** (unlocks at Level 20). Higher Forms keep previous ones unlocked. Source: Macro Progression Matters + Coding Chunks.
- **Save File** — Per-user game state. Stores EXP, unlocked Forms, unlocked Strives per Unit, unlocked Overworld Paths. Vault: `Coding Chunks.md` → Save System.
- **NAGO** — Unknown external project the team had a save system to port from. Mentioned in `Coding Chunks.md`; no other documentation. Open question in [Team Vault — Production Notes](12-Vault-Production-Notes.md).
- **Campaign Log** — Vault concept: post-Victory upload of winning-run statistics. Implementation TBD.
- **Sleeping Spot** — Vault concept: a save-anchor in Open Base. Exit-to-Main-Menu saves the current Sleeping Spot as the return point.

## Test unit naming

- **test_1** — Original combat sandbox units: `first_slime_test`, `first_enemy_test`. Used as the canonical "two units fight" demo across proto-1, proto-2, Assembly.
- **test_2** — Multi-slime placeholder set in proto-2: `second_slime_test`, `big'un-slime_test`, `popper-slime_test`, `scooter-slime_test`, `tantrum-slime_test`. Not present in proto-1.
- **`fodderp`** — Assembly Enemy test unit. Likely "fodder player" or "fodder placeholder" per repo conventions. `Trident_Slime_Ops_Assembly/functionality/Gameplay_Characters/fodderp.gd:1`.
- **`HeroUnit`** — Player-controlled platformer test in proto-1. Uses `velocity` + `move_and_slide()` correctly (the model for fixing `MAOT()`). `tsop---unit-structure/Unit_System/hero_unit_test_1/HeroUnitScript.gd:1`.
- **acadee** — Artist on the TSO team. Produced the `old_shipment/` archive in `tsop---character-animations/` (five generations of slime anim rigs + the "i regret trying puppets triangle_smol" folder).
- **Supercopia** — User of this reference; primary engineer.

## In-game canonical animations (`AnimationRelayer`)

Six canonical names the relayer's `PlayableAnims` dictionary maps. Suffix-matched against the AnimationPlayer's track list:

| Canonical | Aliases | Meaning |
|---|---|---|
| `RESET` | (always first) | Idle |
| `forward_move` | `moving` | Walk toward the enemy |
| `basic_attack` | `attack` | Attack swing |
| `moment_staggered` | `knockback` | Knockback recoil (commented out in `play_anim`) |
| `moment_ded` | `death` | Death (commented out in `play_anim`) |

## Random named things in source

- **`go` / `wait`** — Debug color indicator children of `Ally.gd` in proto-2, toggled to show attack-cycle state.
- **`spawny`** — Local variable in `tso-rushed-stages-showcaes/Unit_System/go_now/waiting_base_for_menu.gd:begin_spawn_enemies()`. Set but never instantiated; function does nothing. See bug #35 in [Known Bugs Catalog](15-Known-Bugs-Catalog.md).
- **`null_object_straight_jail`** — Array (declared `OverworldPlayerPath.gd:105`) used by `ignore_weird_null()` (`:106-112`). Band-aid for a raycast collider-lifetime bug; fix is `is_instance_valid(collider)` guards.
- **`Unready`** — Boolean on `EnemySpawnStartwatch`; if `true`, the timer `queue_free`s itself on `_ready` instead of starting. Defaults `true`. All `config_1.tscn` timers leave it at default → none fire.
- **`greenboxing`** — Hold-the-slot-button-60-ticks toggle on `card_slot_behavior.gd` (proto-1 / Assembly). Spawns a debug border sprite. Has no consumer code; inert.
- **`Greenbox` / `greenbox_active`** — Same feature, the boolean flag.
- **GAY-DLE** — User's internal nickname for `td-like-game-study/dafans-main/` (Benjamin Thio's standalone Battle Cats clone, bundled as a reference).
- **`labelGainsColor`** — Inline GDScript subresource in the three `pretend_*.tscn` scenes of `tsop---scene-loading-structure/`. Paints a label different colors so you can tell which TONS branch is loaded.

## Coding chunks vocabulary

From `Coding Chunks.md`:

- **Defeat Cycle** — Stage Timer + Defeat Process + Defeat Menu, built together.
- **Basic Quick Menu** — The placeholder Level Select with button stubs to other menus.
- **Leveling Management** — Upgrade Menu + Refund button, built together.
- **Saving Equipped Stuff** — Loadout system + Preset Loadouts, built together.

## Misc

- **TBC** — *The Battle Cats*, the game TSO models its combat structure on. Referenced in `Unit Effects.md`.
- **PvZ** — *Plants vs Zombies*. Cited in `Macro Progression Matters.md` as the source of the Training Clock concept.
- **Mao-Mao** — TBC character reference. Used in `Unit Effects.md` as an example of "When Gathering" animation usage.
- **Aku Shield**, **Barrier Breaker**, **Shield Piercing** — TBC mechanics enumerated in `Unit Effects.md` as effects to recreate.
- **B-Switch???** — Vault concept. Small training-course / platformer level on the Overworld. Three question marks because design is uncertain. Source: `Menu Navigation - Implementation Plan.md`.
- **Sleeping Slime** — Vault implies (but doesn't fully spec) a Slime variant tied to the Sleeping Spot save-anchor.
- **PC Slime** — Vault: NPC Slime in Open Base; talking to it routes to the Command Room.
