# Known Bugs Catalog

Every confirmed bug across the repo, cross-linked back to its chapter. Each entry: location, severity, brief description, and where the chapter discusses it.

Severities:

- **Bug** — observable misbehavior.
- **Risk** — likely to misbehave in a not-yet-exercised case (timing, ordering, scale).
- **Smell** — wrong-looking code that happens to be functionally inert.

## Combat addon

| # | Location | Sev | Description | Chapter |
|---|---|---|---|---|
| 1 | `combat_body_2d.gd:6, 31` | Smell | Exported var name misspelled `Overide_Damage_Receiving`. Baked into every `.tscn`. | [Combat System Addon](01-Combat-System-Addon.md) |
| 2 | `combat_body_2d.gd:18` | Smell | `if Damage_Receiver != DamageReceiver2D:` compares instance to class. Always true. Intended `== null`. | [Combat System Addon](01-Combat-System-Addon.md) |
| 3 | `damage_receiver_2d.gd:20` | Risk | Hardcoded 100 ms `create_timer` to wait for `team_direction`. First-100ms hurtbox collisions misbehave. | [Combat System Addon](01-Combat-System-Addon.md) |
| 4 | `damage_receiver_2d.gd:13` | Smell | Addon depends on `UnitNodePackage` (outside the addon). Bidirectional coupling not documented in `Combat_DEPENDENCY_README.txt`. | [Combat System Addon](01-Combat-System-Addon.md) |

## Unit hierarchy (proto-1 / Assembly canonical)

| # | Location | Sev | Description | Chapter |
|---|---|---|---|---|
| 5 | `UnitClass.gd:208-211` (MAOT) | Bug | `position.x +=` bypasses `CharacterBody2D` physics. No `velocity`, no `move_and_slide()`. | [Unit Hierarchy](02-Unit-Hierarchy.md) |
| 6 | `UnitClass.gd:77, 285` | Risk | Hardcoded `$UnitGraphics` lookup crashes if a Unit lacks a child literally named "UnitGraphics". Convention not enforced. | [Unit Hierarchy](02-Unit-Hierarchy.md), [Assembly](11-Assembly.md) |
| 7 | `UnitClass.gd:303` | Risk | `_process_knockback` has an early-return guard `return ## FIX THIS ONCE UNITS ARE FIXED` — actual KB code below is unreachable. | [Unit Hierarchy](02-Unit-Hierarchy.md) |
| 8 | `UnitClass.gd:236` | Smell | `_play_anim` parameter `name_string` shadows `Node.name`. Cosmetic. | [Unit Hierarchy](02-Unit-Hierarchy.md) |
| 9 | `UnitDataClass.gd:23, 24` | Bug | `Attack_Hit_Kilter` and (legacy) `Attack_Hit_Distance` exports are declared but never wired. | [Unit Hierarchy](02-Unit-Hierarchy.md) |
| 10 | `UnitDataClass.gd:48` | Smell | Synchronous `free()` on parent-type mismatch. `queue_free()` is safer. | [Unit Hierarchy](02-Unit-Hierarchy.md) |
| 11 | `Ally.gd:13` | Smell | Copy-paste comment: `team_direction = -1 # Enemies move to the right by default`. | [Unit Hierarchy](02-Unit-Hierarchy.md) |
| 12 | `Enemy.gd:8-15` | Bug | `Stat_Magnification` mutates `max_health` AFTER `set_up_next_KB` already ran in `super()`. KB thresholds calibrated to pre-magnification HP. | [Unit Hierarchy](02-Unit-Hierarchy.md), [Assembly](11-Assembly.md) |
| 13 | `AttackNode.gd:21, 38` | Risk | `AttackCooldown` starts during backswing → chain-attack race if cooldown < backswing. | [Unit Hierarchy](02-Unit-Hierarchy.md) |
| 14 | `HitboxDamageArea2D.gd:91, 130` | Smell | `times_checked` incremented but never read. Dead variable. | [Unit Hierarchy](02-Unit-Hierarchy.md) |
| 15 | `HitboxDamageArea2D.gd:64` | Risk | Pierce-distribution `for I in range(0, expected_pierce_calls): check_for_closest()` re-scans the same hurtboxes redundantly when two are detected per tick. | [Unit Hierarchy](02-Unit-Hierarchy.md) |
| 16 | `RangeDetectors.gd:50-51` | Smell | Bypasses dedicated `_standing_range_toggle()` setter on Unit; writes `in_attacking_range` / `in_standing_range` directly. Author noted at line 49. | [Unit Hierarchy](02-Unit-Hierarchy.md) |
| 17 | `MoonwalkState.gd:12` | Risk | State key string `"StateNodeTest"` must match the StateNode child's node-name in every scene. Silent break on rename. | [Unit Hierarchy](02-Unit-Hierarchy.md), [Assembly](11-Assembly.md) |

## Unit hierarchy (proto-2 — `tso-rushed-stages-showcaes`)

These persist in proto-2 only — proto-1's rewrite fixed them, but they're still live in the rushed-stages prototype.

| # | Location | Sev | Description | Chapter |
|---|---|---|---|---|
| 18 | `tso-rushed-stages-showcaes/.../Unit.gd:55-59` | Bug | `for child in get_children(): ... if child.is_class("DamageArea2D"): child = attack_hitbox`. Backwards assignment — sets the local var, not the field. `attack_hitbox` only gets set because `HitboxDamageArea2D.gd:8` self-registers via `get_parent().attack_hitbox = self`. | [Unit Hierarchy](02-Unit-Hierarchy.md) |
| 19 | `tso-rushed-stages-showcaes/.../Unit.gd:101-103` | Bug | `MAOT() position.x +=` bypasses physics (same as #5; proto-2 doesn't even have the `* 0.15` zoomies multiplier). | [Unit Hierarchy](02-Unit-Hierarchy.md) |
| 20 | `tso-rushed-stages-showcaes/.../Unit.gd:115` | Bug | `_fighting()` literally calls `_idle()`. Fighting and idle look identical. | [Unit Hierarchy](02-Unit-Hierarchy.md) |
| 21 | `tso-rushed-stages-showcaes/.../Unit.gd:159-160` | Bug | `_process_knockback()` is `pass`. Knockback never moves the unit. | [Unit Hierarchy](02-Unit-Hierarchy.md) |
| 22 | `tso-rushed-stages-showcaes/.../AttackNode.gd:13` | Bug | `if get_parent().get_class() != "CharacterBody2D": queue_free()`. `get_class()` returns "Ally"/"Enemy", not "CharacterBody2D" — always true — every AttackNode self-destructs on every Unit. | [Unit Hierarchy](02-Unit-Hierarchy.md) |
| 23 | `tso-rushed-stages-showcaes/.../HitboxDamageArea2D.gd:7` | Bug | Same `get_class()` self-destruct bug as #22. | [Unit Hierarchy](02-Unit-Hierarchy.md) |

## Spawn and wave system

| # | Location | Sev | Description | Chapter |
|---|---|---|---|---|
| 24 | `UnitSpawnSystem.gd:18-19` vs `Ally/Enemy.gd` | Risk | `which_side = 1 = ally` in the spawner, but `team_direction = -1 = ally` on the Unit. Conventions inverted across modules. | [Spawn and Wave System](03-Spawn-and-Wave-System.md), [Assembly](11-Assembly.md) |
| 25 | `UnitSpawnSystem.gd:68` | Risk | Units `add_child`ed to the autoload, not to the active scene. Would break level transitions. | [Spawn and Wave System](03-Spawn-and-Wave-System.md) |
| 26 | `UnitSpawnSystem.gd:64` | Smell | `Vector2(spawn_position)` wraps a Vector2 in `Vector2()`. No-op cast. | [Spawn and Wave System](03-Spawn-and-Wave-System.md) |
| 27 | `EnemySpawnStartwatch.gd:7` | Smell | `Start_Delay` exported but never used. Decorative. | [Spawn and Wave System](03-Spawn-and-Wave-System.md) |
| 28 | `config_1.tscn:47-52` | Bug | All six Timer children ship with `Unready = true` (the default). Every timer `queue_free`s itself on `_ready`. **The config spawns nothing.** | [Spawn and Wave System](03-Spawn-and-Wave-System.md) |

## Stage loop

| # | Location | Sev | Description | Chapter |
|---|---|---|---|---|
| 29 | `PlayerMidStageVault.gd:48` (Assembly), `:56` (proto-2) | Bug | `request_ally()` always passes `unit_in_slot_1`, ignoring the `unit_slot` parameter. In proto-2, every deck click spawns slot-1's unit. In Assembly, `unit_in_slot_1` is never assigned, so every click queues `null`. | [Stage Loop](04-Stage-Loop.md), [Assembly](11-Assembly.md) |
| 30 | `PlayerMidStageVault.gd:23` | Smell | `CPC = current_player_currency` is a one-shot copy in `_ready`. The two values diverge from frame 2. | [Stage Loop](04-Stage-Loop.md) |
| 31 | `proto-2 PlayerMidStageVault.gd:21` | Smell | `current_unit_prices: Array = [100, 200, 300, 400]` has 4 prices for 8 slots. The bounds check at `PMSV.gd:42-44` catches `unit_slot >= 4` with an error print and early return — so slots 4–7 silently fail to spawn rather than crashing. | [Stage Loop](04-Stage-Loop.md) |
| 32 | `p_deck_controls.gd:70` | Bug | `if type != "heavy" or "lite":` — evaluates as `(type != "heavy") or "lite"`. `"lite"` is truthy. Always true. Cooldown early-returns on every call. | [Stage Loop](04-Stage-Loop.md), [Assembly](11-Assembly.md) |
| 33 | `p_deck_controls.gd:59-60` | Smell | `_ready` resets `@export`ed `MenuOpenable` / `StrivesMenuOpen` to false, discarding inspector values. Author noted "And why does THIS control the menu being open or not????" | [Stage Loop](04-Stage-Loop.md), [Assembly](11-Assembly.md) |
| 34 | `card_slot_behavior.gd:32-58` (proto-1, Assembly) | Smell | "Greenboxing" hold-60-ticks toggles `greenbox_active` and spawns a debug sprite, but no auto-spawn loop reads the flag. Inert feature. | [Stage Loop](04-Stage-Loop.md), [Assembly](11-Assembly.md) |
| 35 | `proto-2 waiting_base_for_menu.gd:23-36` | Bug | `begin_spawn_enemies()` declares `var spawny` and runs `str(spawny)` on the last line. Function does nothing. | [Stage Loop](04-Stage-Loop.md) |

## Assembly-specific

| # | Location | Sev | Description | Chapter |
|---|---|---|---|---|
| 36 | `UnitStatChangerClass.gd:42-73` | Smell | `if dict.has(key)` is always true — dictionaries pre-seeded with all keys at declaration. Dead control flow. | [Assembly](11-Assembly.md) |
| 37 | `UnitStatChangerClass.gd:50-53, 67-70` | Bug | `range` master-key branch reads `attack_range` / `stand_range` / `atk_hit_dist` dict entries, not the `range` entry. `range` is non-functional. | [Assembly](11-Assembly.md) |
| 38 | `UnitStatChangerClass.gd:61-73` | Bug | `int *= float` assignment truncates. Multipliers < 1.0 round down. | [Assembly](11-Assembly.md) |
| 39 | `fade_unit_death_anim.gd:3-7` | Bug | `var modulate_subtract: int = 100` and condition `if modulate_subtract + 1 <= 100` is `101 <= 100 == false` from frame 1. **The fade never plays. Unit is `queue_freed` instantly.** | [Assembly](11-Assembly.md) |
| 40 | `health_label.gd:18` | Bug | `current_health / _unit_ref.Default_Health` — both `int`. `50 / 100 == 0`. HP bar collapses to 0/1, doesn't shrink smoothly. | [Assembly](11-Assembly.md) |
| 41 | `debug_attack_cooldown_display.gd:43` | Smell | `var in_backswing: bool = swing_left > 0.0` — same condition as `in_foreswing` minus the cooldown gate. Branching at `:66` is redundant. | [Assembly](11-Assembly.md) |
| 42 | `temporary_test_scene.tscn` | Bug | Main scene has no `player_deck.tscn` instance and no spawner reference. PMSV's `unit_in_slot_1` is never assigned. The full deck UI is dead-loaded. | [Assembly](11-Assembly.md) |
| 43 | `Unit-Node-Packages-README.txt` | Doc | References `res://Unit_System/test_2/unit_template_v2.tscn` — path doesn't exist in Assembly (only `fodderp.tscn` and `second_slime_test.tscn` exist as concrete units). | [Assembly](11-Assembly.md) |

## Animation Relay

| # | Location | Sev | Description | Chapter |
|---|---|---|---|---|
| 44 | `AnimationRelayClass.gd:37` | Bug | Null-deref if no `AnimationPlayer` child exists. `is_class("AnimationPlayer")` check at `:32` doesn't guard the unconditional access at `:37`. | [Animation Relay](05-Animation-Relay.md) |
| 45 | `AnimationRelayClass.gd:73` | Bug | `has_animation()` bare `return` in the `if afk_value == null` branch yields `null`, not `false`. Callers comparing to bool may misbehave. | [Animation Relay](05-Animation-Relay.md) |
| 46 | `AnimationRelayClass.gd:13` | Bug | `signal animation_ended` declared but never emitted. Decorative. | [Animation Relay](05-Animation-Relay.md) |
| 47 | `AnimationRelayClass.gd:48-59` | Bug | `death` and `knockback` cases commented out in `play_anim()`. Two of six canonical names are disabled at the play path. | [Animation Relay](05-Animation-Relay.md) |
| 48 | `practice_unit_anims.tscn` | Risk | Live test scene depends on `old_shipment/AnimationRelayClass.gd`. Anyone "archiving" the `old_shipment/` folder will break the scene. | [Animation Relay](05-Animation-Relay.md) |

## Overworld map

| # | Location | Sev | Description | Chapter |
|---|---|---|---|---|
| 49 | `OverworldPlayerPath.gd:142, 150, 158, 166` | Risk | Unguarded `get_parent().player_input_dir` access. If the path is not a child of the expected cursor node, this crashes. | [Overworld Map](06-Overworld-Map.md) |
| 50 | `OverworldPlayerPath.gd:55, 63` | Risk | `var speed: float = 0.05` declared but `progress_ratio += 0.05` uses the hardcoded literal at `:63`. If `speed` ever changes, line 63 won't see it. | [Overworld Map](06-Overworld-Map.md) |
| 51 | `OverworldPlayerPath.gd:133` | Smell | `42.0` hardcoded threshold in `input_suite()` — no symbolic name, no comment. | [Overworld Map](06-Overworld-Map.md) |
| 52 | `OverworldPlayerPath.gd:79-80, 105-117` | Smell | `add_exception()` accumulation never cleared. `null_object_straight_jail` array is a band-aid for raycast collider-lifetime; proper fix is `is_instance_valid()` guards. | [Overworld Map](06-Overworld-Map.md) |
| 53 | `Interactive_Overworld_Tile.gd:4` | Doc | `update_level_info()` prints `"stage_name is ", stage_name` — the May 15 report misquoted this as `"Loading scenes is a whole headache..."` (which is actually a comment on the next line). | [Overworld Map](06-Overworld-Map.md) |
| 54 | `tsop---overworld-map/project.godot:16` | Doc | Only prototype still pinned to Godot 4.5. Others are 4.6. | [Overworld Map](06-Overworld-Map.md) |

## Stage data format

| # | Location | Sev | Description | Chapter |
|---|---|---|---|---|
| 55 | `StageDataClass.gd:164, 168, 173, 176, 186, 189` | Bug | All six spawn-condition functions (`WhenEB_Health`, `WhenObjectHealth`, etc.) are `pass` stubs. The schema has no runtime. | [Stage Data Format](07-Stage-Data-Format.md) |
| 56 | `StageDataClass.gd:44` vs `:90` | Bug | `HeadsUpBeforeSpawn` schema is `[count, time_before]` (array) in `OneEnemyExample` and scalar `0` in `cleanerExampleEnemy`. Inconsistent. | [Stage Data Format](07-Stage-Data-Format.md) |
| 57 | `StageDataClass.gd:11, 99` | Smell | `SchematicForLevel` and `testDictionaryDouble` `@export`s are declared but never read. | [Stage Data Format](07-Stage-Data-Format.md) |
| 58 | `StageDataClass.gd` | Smell | Class name disagreement: file is `StageDataClass.gd`, `class_name` is `StageInfoClass`. | [Stage Data Format](07-Stage-Data-Format.md) |
| 59 | `StageDataClass.gd:105-142` | Smell | Five dead test functions (`_test_this_stuff` `:105-109`, `dictionary_assign_test` `:117-123`, `get_or_add_test_1` `:125-128`, `get_or_add_test_2` `:131-136`, `get_or_add_test_3galoo` `:138-142`). Their calls in `_ready()` (`:111-115`) are all commented out except `get_or_add_flex()`. | [Stage Data Format](07-Stage-Data-Format.md) |

## Scene loading architecture

| # | Location | Sev | Description | Chapter |
|---|---|---|---|---|
| 60 | `TreeOfNodes.gd:29` | Smell | `deactivate_spawn` calls `queue_free` but doesn't null `child_scene`. Reactivation is `isActive`-gated, so impact is bounded. | [Scene Loading Architecture](08-Scene-Loading-Architecture.md) |
| 61 | `TreeOfNodes.gd:46, 50, 54` | Risk | Branch dispatch uses `name.contains("Combat")` / `"Menu"` / `"World"+"Map"`. Brittle to renames. | [Scene Loading Architecture](08-Scene-Loading-Architecture.md) |

## External study (GAY-DLE)

| # | Location | Sev | Description | Chapter |
|---|---|---|---|---|
| 62 | `dafans-main/character_slots_container.gd:84-85` | Bug | Slot-button label uses `CHARACTER.find_key((row_index * 5) + slot_index).capitalize()` — hardcodes a 5-column grid. Silently breaks on resize. | [External Study — GAY-DLE](10-External-Study-GAYDLE.md) |
| 63 | `dafans-main/.godot/` | Hygiene | 41 MB / 348 files of editor cache committed. Repo's `.gitignore` predates Godot 4 (lists `.import/`, not `.godot/`). | [External Study — GAY-DLE](10-External-Study-GAYDLE.md) |

## Cross-cutting / documentation

| # | Location | Sev | Description | Chapter |
|---|---|---|---|---|
| 64 | `PROTOTYPES_REPORT.md:41` | Doc | "Most code is identical between both — unit-structure is the foundation, rushed-stages-showcaes builds on top of it." Inverted since report date. Proto-1 has been rewritten; proto-2 is the time-capsule. | [Unit Hierarchy](02-Unit-Hierarchy.md), [Assembly](11-Assembly.md) |
| 65 | `PROTOTYPES_REPORT.md:498` | Doc | "UID targeting via `Array[int]` of instance IDs + `instance_from_id()`". GAY-DLE actually uses a custom monotonic counter, not Godot instance IDs. | [External Study — GAY-DLE](10-External-Study-GAYDLE.md) |
| 66 | `PROTOTYPES_REPORT.md:502` | Doc (unverified) | "Real `velocity = speed * direction` + `move_and_slide()`" — claim is plausible for GAY-DLE but not directly re-verified. | [External Study — GAY-DLE](10-External-Study-GAYDLE.md) |
| 67 | `PROTOTYPES_REPORT.md:526` | Doc | "Single highest-priority bug across the new folders" no longer applies — `TreeOfNodes.gd:19`'s missing `add_child` is fixed. | [Scene Loading Architecture](08-Scene-Loading-Architecture.md) |
| 68 | `Combat_DEPENDENCY_README.txt` | Doc | Says the Combat addon is required by `Combat_Units_system`. Doesn't mention the reverse dependency: the addon's `damage_receiver_2d.gd:13` references `UnitNodePackage` (outside the addon). | [Assembly](11-Assembly.md) |
| 69 | Macro Progression Leveling List | Doc | Three editing slips in the per-level table (27/28 "x%", 29 "464%" inconsistent), plus Strive Slot rule conflict between per-level entries (3, 6, 11, …) and Strive Unlocks section (3, 8, 13, …), plus TP-gain Level 5/15 vs 8/18/28 conflict. | [Team Vault — Production Notes](12-Vault-Production-Notes.md) |

## Summary

69 entries: 25 Bugs, 12 Risks, 22 Smells, 9 Doc-only, 1 Hygiene.

**Highest-priority Bugs to fix in the current canonical (proto-1 / Assembly):**

- #29 PMSV `request_ally` always-slot-1 (or always-null in Assembly)
- #39 `fade_unit_death_anim` off-by-one (fade never plays)
- #40 `health_label` integer division
- #5 / #7 MAOT and `_process_knockback` physics bypass
- #12 Enemy `Stat_Magnification` ordering vs `set_up_next_KB`
- #32 `_on_cooldown_button_press` always-true conditional
- #37 / #38 `UnitStatChangerClass` `range` key + int-truncate
- #44 / #45 `AnimationRelayClass` null-deref and bare-return

**Proto-2-only bugs that don't need fixing if proto-2 is the time-capsule** (#18–#23). But should be fixed if proto-2 work continues — it carries the pause menu and save system.
