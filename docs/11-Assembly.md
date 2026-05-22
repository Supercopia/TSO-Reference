# Trident Slime Ops Assembly

`Trident_Slime_Ops_Assembly/` is the integrated build — the merge of the proto-1 unit architecture into a single project, plus three Assembly-unique additions: the `UnitStatChanger` system, the Unit Package `.tscn` library, and the integration scaffold (PMSV + UnitSpawnSys + player deck UI).

Per Stream A's diff, **Assembly is ~95% verbatim from `tsop---unit-structure/`**. The `addons/Combat_system/` is byte-identical; `StateNodeClass`, `AttackNode`, `HitboxDamageArea2D`, `RangeDetectors`, `MoonwalkState` are byte-identical; `Ally`, `Enemy`, `UnitNodePackage`, `UnitDataClass` differ only by `@icon` path. The genuinely new architecture is:

1. `UnitStatChanger` — a post-UnitData stat-modification layer.
2. The `Unit_Parts/node-packages/` `.tscn` library — concrete Unit Package instances (Hurtbox, Movement, Attacking, HealthDisplay).
3. Integration scaffold — PMSV + UnitSpawnSys + player deck scaffold from proto-1's UI, plus the partition-by-role script layout under `functionality/Combat_Units_System/Scripts/`.

This chapter documents **only what's Assembly-unique**. Everything else is in the chapters it shares architecture with:

- Combat addon → [Combat System Addon](01-Combat-System-Addon.md)
- UnitClass, Ally, Enemy, UnitData, AttackNode, HitboxDamageArea2D, RangeDetectors, StateNode, MoonwalkState → [Unit Hierarchy](02-Unit-Hierarchy.md)
- UnitSpawnSys, queue mechanics → [Spawn and Wave System](03-Spawn-and-Wave-System.md)
- PMSV, player deck, card_slot_behavior, p_deck_controls → [Stage Loop](04-Stage-Loop.md)

The full per-file research transcript is in `_research/assembly-deep-dive.md`.

## project.godot

```ini
config_version=5

[application]
config/name="Trident: Slime Ops"
config/tags=PackedStringArray("full_game")
run/main_scene="uid://dq3pqflqcvhf6"      # → temporary_test_scene.tscn
config/features=PackedStringArray("4.6", "Mobile")
config/icon="res://icon.svg"

[autoload]
PMSV="*uid://ceq2bio7i0xxe"
UnitSpawnSys="*uid://m54y28cycvfq"

[physics]
3d/physics_engine="Jolt Physics"

[rendering]
rendering_device/driver.windows="d3d12"
renderer/rendering_method="mobile"
```

- **Godot 4.6**, Mobile renderer, D3D12, Jolt physics — same as proto-1.
- **Only two autoloads:** `PMSV` and `UnitSpawnSys`. Proto-1 also has `no_camera_solver` and `AutoAuto`; Assembly dropped both. `qgm` and `SaveHandler` (proto-2's autoloads) were never ported.
- **Main scene is `temporary_test_scene.tscn`** — see [§ Main scene](#main-scene).
- `config/tags=PackedStringArray("full_game")` — new tag, not present in earlier prototypes.

## Combat_DEPENDENCY_README.txt

Quoted verbatim from `Trident_Slime_Ops_Assembly/Combat_DEPENDENCY_README.txt:1-3`:

```
If you ever transfer the Combat_Units_system folder to other projects,
please note that this addon (Combat_system) is required for it to function.
```

The note is correct about the dependency direction but misses the bidirectional coupling: `addons/Combat_system/Scripts/damage_receiver_2d.gd:13` calls `UnitNodePackage.find_unit(self)` — `UnitNodePackage` lives outside the addon. See [Combat System Addon § damage_receiver_2d depends on code outside the addon](01-Combat-System-Addon.md#where-it-lives).

## Folder layout — partition by role

Where proto-1 keeps unit code under `Unit_System/scripts/class_define/` and `Unit_System/scripts/child_nodes/`, Assembly **partitions by role** under `functionality/Combat_Units_System/Scripts/`:

```
of-new-classes/   UnitClass, UnitDataClass, UnitNodePackage,
                  UnitStatChangerClass, StateNodeClass, Ally, Enemy
of-unit-parts/    AttackNode, HitboxDamageArea2D, RangeDetectors,
                  basic_attack_node.tscn
one-offs/         MoonwalkState, debug_attack_cooldown_display,
                  fade_unit_death_anim, health_label, stat-changers/
```

The partition makes intent more legible. New classes are central; unit parts are the leaf nodes the new classes orchestrate; one-offs are everything that didn't fit either bucket.

## UnitStatChanger

`Scripts/of-new-classes/UnitStatChangerClass.gd` (77 lines). The post-UnitData stat-modification layer.

### How it integrates

`UnitClass.get_children_things()` (`UnitClass.gd:417-426`) collects all child `UnitStatChanger` nodes into `usc_array`, then:

1. Runs `UnitData.parent_uses_data()` first — pushes baseline stats.
2. For each changer in `usc_array`, calls `changer.apply_changes()`.
3. Calls `set_up_next_KB()` last — KB thresholds reflect post-change `max_health`.

### apply_changes

Source `UnitStatChangerClass.gd:7-29` (the two stat dicts) and `:35-74` (the function), selected lines verbatim:

```gdscript
var flat_changes: Dictionary = {
    "health":        0, # also affects max health.
    "max_health":    0,
    "attack_damage": 0,
    "move_speed":    0, 
    "kb_count":      0, # knockback count - probably can't be evenly multiplied?
    "kb_dist":       0, # knockback distance
    "range":         0, # Raises attack range, attack hit distance, AND standing range.
    "attack_range":  0,
    "stand_range":   0,
    "atk_hit_dist":  0,
}
var mult_changes: Dictionary = {
    "health":        1.0,
    # ... same keys as flat_changes except no "kb_count", all defaults 1.0 ...
}

func apply_changes():
    if changes_target != null:
        var unit = changes_target
        ## Flat Additive/Subtractive changes - batch 1
        if changes_target:
            if flat_changes.has("health"):
                unit.max_health += flat_changes.get("health")
                unit.current_health += flat_changes.get("health")
            if flat_changes.has("max_health"): unit.max_health += flat_changes.get("max_health")
            # ... 6 more flat-stat branches ...
            if flat_changes.has("range"):
                unit.Attacking_Range += flat_changes.get("attack_range")
                unit.Standing_Range += flat_changes.get("stand_range")
                unit.Attack_Hit_Distance += flat_changes.get("atk_hit_dist")
            # ... per-stat flat branches for attack_range / stand_range / atk_hit_dist ...
        
        ## Multiplicative changes - batch 1 (lines 60-73)
        if changes_target:
            if mult_changes.has("health"):
                unit.max_health *= mult_changes.get("health")
                unit.current_health *= mult_changes.get("health")
            # ... mirrors the flat-changes structure with *= ...
```

The double `if changes_target != null: ... if changes_target:` is in source verbatim (`:36, 41, 59`). Whether the inner `if changes_target:` ever differs from the outer guard isn't clear from source — `changes_target` is set once in `_ready` and never reassigned.

### Bugs

!!! warning "`if dict.has(key)` is always true (`UnitStatChangerClass.gd:42-73`)"
    Both dictionaries are initialised with all keys at declaration. `.has()` returns true for every key. The intended branching never branches — every changer touches every stat, applying default 0 / 1.0 for unspecified keys. Functionally harmless (0 and 1.0 are no-ops), structurally dead control flow.

!!! warning "`range` master-key branch reads the wrong keys (`:50-53, 67-70`)"
    Inside `if flat_changes.has("range"):`, the code reads `attack_range`, `stand_range`, `atk_hit_dist` — not `range`. The `range` key is never actually used. Same bug duplicated in the `mult_changes` branch.

!!! warning "Integer * float assignment truncates (`:61-73`)"
    Lines like `unit.max_health *= mult_changes.get("health")` multiply `int` by `float` and assign back to int. Multipliers < 1.0 round down. The Carefulness changer's `move_speed × 0.84` becomes `move_speed * 0` for any `move_speed < 6`.

### Concrete subclasses

Three stat-changers in `Scripts/one-offs/stat-changers/`, each ≈10 lines:

| Class | Effect |
|---|---|
| `Carefulness` | `health × 1.42`, `move_speed × 0.84` |
| `DamageUp` | `attack_damage × 1.25` |
| `HeavyPacking` | `attack_damage × 1.42`, `move_speed × 0.84` |

Pattern: `extends UnitStatChanger`, override `_ready()` to call `super()` then `mult_changes.set("key", value)`. Each ships as a `.tscn` (`Unit_Parts/stat-changers/Carefulness.tscn`, etc.) so it can be instanced from the inspector without a script path reference.

These read as **early Strive prototypes** — they apply on Unit spawn and persist for the unit's lifetime. The vault's design specifies Strives should be triggered by events; that distinction isn't yet implemented.

## Unit Package .tscn library

`Unit_Parts/node-packages/`. The concrete `.tscn` instances of the Unit Package System. Each is a `Node2D` root with `UnitNodePackage.gd` attached.

### Unit-Node-Packages-README.txt

Quoted in full in `_research/assembly-deep-dive.md`. Key takeaways:

- **Needed** (essential): Hurtbox, Allow Movement, Allow Attacking.
- **Not Needed** (optional): Display Unit Health.
- **Not Working**: MoonwalkState as a package (and by extension all Special States) — "kinda suck, but is liveable."

Composition rules:

- Bases / Objects → just the Hurtbox.
- Non-Fighting Units → Hurtbox + Allow Movement.
- Full combat unit → Hurtbox + Allow Movement + Allow Attacking.

### The four packages

| Pack | Root | Contents |
|---|---|---|
| `give_hurtbox_pack.tscn` | Hurtbox | `DamageReceiver2D` + capsule shape + `FrontDetector` Area2D + segment shape |
| `allow_attacking_pack.tscn` | Allow Attacking | `AttackRay` (RangeDetector) + `HitboxDamageArea2D` + `CollisionShape2D` + `AttackNode` + `AttackCooldownDisplay` (rotated diamond UI) |
| `allow_movement_pack.tscn` | Allow Movement | `StandRay` (RangeDetector with `Stand_or_Range = false`) |
| `display_unit_health_pack.tscn` | Display Unit's Health | Label + two ColorRect bars + NameLabel |

`allow_attacking_pack.tscn` is the largest — bundles 7 distinct nodes including the debug cooldown display (a rotated `ColorRect` with sub-children for foreswing/backswing fillings).

Stat-changer scenes (`Unit_Parts/stat-changers/`) follow the same pattern — single-node `.tscn`s holding their respective script — so they can be instanced from the inspector.

## Main scene

`functionality/Gameplay_Characters/temporary_test_scene.tscn`, verbatim:

```
[gd_scene format=3 uid="uid://dq3pqflqcvhf6"]

[ext_resource type="PackedScene" uid="uid://d2tg0oxcbkvj2" path="res://functionality/Gameplay_Characters/fodderp.tscn" id="1_6hrfl"]
[ext_resource type="PackedScene" uid="uid://bimvwycrtrctd" path="res://functionality/Gameplay_Characters/second_slime_test.tscn" id="2_elw4i"]

[node name="TemporaryTestScene" type="Node2D" unique_id=359248475]

[node name="fodderp" parent="." unique_id=1466529190 instance=ExtResource("1_6hrfl")]
position = Vector2(309, 358)

[node name="basic_slime" parent="." unique_id=792248777 instance=ExtResource("2_elw4i")]
position = Vector2(824, 358)
```

Two pre-placed units, no camera, no background, no spawner. On launch, fodderp (Enemy with Carefulness + HeavyPacking) walks left, basic_slime (Ally with same stat-changers + `Default_Damage = 50`) walks right. They collide and fight.

**That is the entire gameplay loop on launch.** Despite PMSV and UnitSpawnSys being autoloads, the main scene contains neither the `player_deck.tscn` UI nor any spawner reference. `player_deck.tscn` is fully built (8 SLOT buttons, Strives menu, cooldown buttons, info box, position arrangement for all 8 slots) but **never instanced anywhere**.

## fodderp.tscn / second_slime_test.tscn

Two concrete unit scenes in `Gameplay_Characters/`.

`fodderp.gd`:
```gdscript
extends Enemy
```

`second_slime_test.gd`:
```gdscript
extends Ally
```

That's the entire script side for both. The scene structure carries the gameplay (`fodderp.tscn:1-49`, `second_slime_test.tscn:1-51`). Children, in order, for both:

1. `UnitData` (with per-scene stat overrides)
2. `Hurtbox` (instance of give_hurtbox_pack)
3. `StateNodeTest` (instance of MoonwalkState.gd — direct child, not in a package)
4. `UnitGraphics` (Node2D, with a `Sprite2D` child)
5. `AnimationPlayer` (empty)
6. `Allow Attacking` (instance of allow_attacking_pack)
7. `Display Unit's Health` (instance of display_unit_health_pack)
8. `Allow Movement` (instance of allow_movement_pack)
9. `Carefulness` (stat-changer)
10. `Heavy Packing` (stat-changer)

So both units have **two stacked stat-changers** that multiply: Carefulness × HeavyPacking → `move_speed × 0.84² ≈ 0.706`. Multiplicative stacking is the de-facto convention; the vault doesn't spell out whether this is intended.

## Gameplay Resources

`functionality/Gameplay_Resources/` carries the autoload scripts and the deck UI:

- `PlayerMidStageVault.gd` — see [Stage Loop § PMSV](04-Stage-Loop.md#playermidstagevault-pmsv). Assembly's version is the one-slot proto-1 variant, with `unit_in_slot_1` **never assigned**.
- `UnitSpawnSystem.gd` — see [Spawn and Wave System](03-Spawn-and-Wave-System.md).
- `card_slot_behavior.gd`, `p_deck_controls.gd` — see [Stage Loop](04-Stage-Loop.md). Both fully built; deck UI dead-loaded.

### Folder description (verbatim, `Gameplay_Resources/FOLDER_DESCRIPTION.txt`)

> Gameplay_Resources
>
> I mostly call this that because it involves resources the player can access.
> - Their Unit Slots and Player Deck
> - That's kinda it?
> - Oh and the cooldowns ofc.

## Differences from proto-1

Key code-level deltas (full list in `_research/assembly-deep-dive.md` § 4.1):

| Change | Where |
|---|---|
| Brightness restore loop in `_physics_process` | `UnitClass.gd:76-80` |
| Knockback rewrite — `velocity`-driven, exponential decay, `move_and_slide()` | `UnitClass.gd:308-324` |
| Knockback_Distance sign flipped (units slide backward instead of forward) | `UnitClass.gd:330` |
| `set_up_next_KB` reads `max_health` not `Default_Health` | `UnitClass.gd:352-364` |
| StatChanger integration in `get_children_things` | `UnitClass.gd:417-426` |
| Removed `if KEY_ENTER: _start_knockback()` debug | (proto-1 had it at `:180`) |
| Greyout `Color(0.537,...)` in `_attack_continues` commented out | `UnitClass.gd:270, 277` |

The knockback rewrite is the most recent commit (`52569d8 Smoother Unit Knockbacks`). It maps onto proto-1's `_start_knockback()` + `next_KB_health` tracking — same algorithm, applied to Assembly's copy.

## What hasn't been ported into Assembly yet

Confirmed by `find` in [Stream A](../_research/assembly-deep-dive.md) and [Stream B § Cross-reference drift report](../_research/vault-extraction.md#8-cross-reference-drift-report):

| System | Lives in | Status in Assembly |
|---|---|---|
| `QuickGameManager` (qgm) + win/loss | proto-2 | Not present. `find *game_manager* *qgm*` returns nothing. |
| `SaveHandler` + save addon | proto-2 | Not present. |
| Bases (`ally_base_go`, `enemy_base_go`) + `on_base_death_node` | proto-2 | Not present. |
| `waiting_base_for_menu.gd` (wave config kickoff) | proto-2 | Not present. |
| Pause menu (`ui/pause_menu.gd`) | proto-2 | Not present. |
| `EnemySpawnStartwatch` + `config_1.tscn` | proto-2 | Not present. |
| Overworld map / IOTile / OverworldPlayerPath | `tsop---overworld-map/` | Not present. |
| Wallet leveling (vault Iteration 1 Step 4) | Designed in vault, not coded anywhere | Not present. |
| `no_camera_solver` autoload | proto-1 | Dropped from autoloads. |
| `AutoAuto` autoload | proto-1 | Dropped from autoloads. |

So Assembly carries the **combat architecture** of proto-1 (UnitClass, UnitNodePackage, StateNode, knockback) but **none of the gameplay-loop polish** of proto-2 (qgm, save, pause, bases, win/loss, waves). The deck UI is built but un-instanced. PMSV's slot is never assigned.

Per the vault's Menu Navigation plan (Iteration 1 in [Team Vault — Production Notes](12-Vault-Production-Notes.md)), Assembly is roughly halfway through Step 1-6 — combat works, integration scaffold is in place, but the actual game loop has not been wired up.

## Assembly-specific bug catalog

Bugs unique to Assembly (proto-1 versions of these scripts behave differently or don't have these scripts at all):

| # | Location | Bug |
|---|---|---|
| 1 | `UnitStatChangerClass.gd:42-73` | `dict.has(key)` always true — dead control flow |
| 2 | `UnitStatChangerClass.gd:50-53, 67-70` | `range` master-key reads wrong dict keys |
| 3 | `UnitStatChangerClass.gd:61-73` | int *= float truncates |
| 4 | `Enemy.gd:8-15` | `Stat_Magnification` re-applies `max_health` after `set_up_next_KB` ran |
| 5 | `PlayerMidStageVault.gd:48` | Always queues `unit_in_slot_1`, which is `null` (never assigned) |
| 6 | `UnitSpawnSystem.gd:18-19` vs `Ally/Enemy.gd` | `which_side` sign convention inverted across modules |
| 7 | `UnitSpawnSystem.gd:68` | Units `add_child`ed to autoload, not to active scene |
| 8 | `card_slot_behavior.gd:32-58` | Greenboxing flag toggles but no auto-spawn loop consumes it |
| 9 | `p_deck_controls.gd:70` | `if type != "heavy" or "lite":` always true |
| 10 | `p_deck_controls.gd:59-60` | `_ready` resets `@export`ed booleans, discarding inspector values |
| 11 | `fade_unit_death_anim.gd:3-7` | `modulate_subtract = 100` off-by-one — fade never plays, unit `queue_free`s immediately |
| 12 | `health_label.gd:18` | Integer division on HP bar — `current/Default_Health` collapses to 0/1 |
| 13 | `temporary_test_scene.tscn` | Main scene has no spawner, no player deck, no win/loss — gameplay is two pre-placed units |
| 14 | `Unit-Node-Packages-README.txt` | References `res://Unit_System/test_2/unit_template_v2.tscn` which doesn't exist in Assembly |
| 15 | `UnitClass.gd:77, 285` | Hardcoded `$UnitGraphics` lookup crashes if node not named exactly that |
| 16 | `combat_body_2d.gd:18` | `if Damage_Receiver != DamageReceiver2D:` instance-vs-class comparison (inherited from addon) |
| 17 | `damage_receiver_2d.gd:20` | 100 ms hardcoded delay for `team_direction` (inherited from addon) |

See full catalog cross-linked back to chapters in [Known Bugs Catalog](15-Known-Bugs-Catalog.md).

## Carry-forward verdict

**Assembly architecture (script partitioning, Unit Package `.tscn` library) — Tier 1.** The `of-new-classes/` / `of-unit-parts/` / `one-offs/` split is more legible than proto-1's `class_define/` / `child_nodes/` and should stay. The Unit Package `.tscn` library is the *intended use* of the Unit Package System — these `.tscn`s are reusable across units and should be the model for any future Strive packs.

**`UnitStatChangerClass` — Tier 2 (rewrite, keep the idea).** The composable stat-modification idea is right; the implementation has three real bugs (dead control flow, wrong-keys, int truncation). Fix before further use.

**Concrete stat-changers (Carefulness / DamageUp / HeavyPacking) — Tier 3.** Early Strive prototypes. Once Strives are properly designed per vault, these should be re-implemented from the design spec rather than refactored from these.

**`temporary_test_scene.tscn` — discard.** Bare arena; replace with the battlefield scene described in the vault's Iteration 1 Step 6.

**Open structural work:** port qgm, SaveHandler, pause menu, bases, waves from proto-2 into Assembly. The combat half is done; the gameplay-loop half is not. The vault's Menu Navigation plan covers the steps in order.
