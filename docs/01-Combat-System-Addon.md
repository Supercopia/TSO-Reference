# Combat System Addon

The `Combat_system` addon is the lowest-level invariant of TSO's combat. Everything else — Units, Hitboxes, Hurtboxes, the Unit Package System — assumes its three classes exist and behave a specific way.

The addon ships as `addons/Combat_system/` and is bundled with three projects: `tsop---unit-structure/`, `tso-rushed-stages-showcaes/`, and `Trident_Slime_Ops_Assembly/`. Citations below are against the `tsop---unit-structure/` copy unless otherwise noted; all three copies are kept in sync (see [Where it lives](#where-it-lives)).

## Plugin Manifest

`addons/Combat_system/plugin.cfg` registers the addon:

```ini
[plugin]
name="Goodai9's Combat system"
description="Easy to implement combat system for 2D godot games for dealing and receiving damage.

Minor changes by AcadeeAlkana."
author="Goodai9 & AcadeeAlkana"
version="1.0.1"
script="Combat_system.gd"
```

`Combat_system.gd` is an empty `EditorPlugin` stub — both `_enter_tree` and `_exit_tree` are `pass`. The plugin entry exists only so the addon shows as enabled in Project Settings.

## The Three Classes

Three independent classes — no shared parent class, not a scene-tree hierarchy. They cooperate at runtime via signals and an ancestor walk.

| Class | Extends | Role |
|---|---|---|
| `CombatBody2D` | `CharacterBody2D` | Owns HP. Entry point for damage via `take_damage(amount)`. |
| `DamageReceiver2D` | `Area2D` | Hurtbox. Routes incoming `DamageArea2D` overlaps to the host's `take_damage`. |
| `DamageArea2D` | `Area2D` | Hitbox. Carries `Damage`, emits `damage_dealt` on overlap. |

In a real scene a `DamageReceiver2D` is typically a grandchild of a `CombatBody2D` (inside a Unit Package), and a `DamageArea2D` is typically a child of the swing-side hitbox node. The classes don't *require* this layout — that's just how Unit Packages compose them.

`Unit extends CombatBody2D` (`Unit_System/scripts/class_define/UnitClass.gd:4`). `HitboxDamageArea2D extends DamageArea2D` (`Unit_System/scripts/child_nodes/HitboxDamageArea2D.gd:1`). Everything in the Unit hierarchy ultimately bottoms out at these three classes.

!!! note "Assembly paths"
    Citations in this chapter use the `tsop---unit-structure/` paths (per the opening). In `Trident_Slime_Ops_Assembly/` the equivalents live at `functionality/Combat_Units_System/Scripts/of-new-classes/UnitClass.gd` and `functionality/Combat_Units_System/Scripts/of-unit-parts/HitboxDamageArea2D.gd`. Line numbers match.

## CombatBody2D

`addons/Combat_system/Scripts/combat_body_2d.gd` — owns hit points and the damage entry-point.

Key exports and fields (lines 6–12):

```gdscript
class_name CombatBody2D
extends CharacterBody2D

@export var Overide_Damage_Receiving: bool = false
@export var Damage_Receiver: DamageReceiver2D
@export var Default_Health: int = 100
var max_health: int
var current_health: int
signal damage_token(amount)
```

On `_ready()` (lines 14–22):

- `max_health = Default_Health` and `current_health = Default_Health`.
- If `Damage_Receiver` is unset, walks children to find one and assigns it.

### Damage entry point

```gdscript
func register_damage(damage: int, ignore_overide: bool) -> void:  # :25
    if ignore_overide:
        current_health = clampi(current_health - damage, 0, max_health)
        damage_token.emit(damage)
        return
    if not Overide_Damage_Receiving:
        current_health = clampi(current_health - damage, 0, max_health)
        damage_token.emit(damage)

func take_damage(damage: int) -> void:  # :35
    register_damage(damage, true)
    flicker_when_hit()

func heal_damage(amount: int) -> void:  # :43
    current_health = clampi(current_health + amount, 0, max_health)
```

`take_damage` is the canonical entry point — subclasses (`Unit`, `Ally`, `Enemy`) override it and chain via `super(damage)`. `register_damage` is also exposed so a stat-changer or status node can bypass the visual flicker.

`flicker_when_hit()` (lines 46–50) brightens the body's `modulate` to `Color(1.15, 1.15, 1.15)` for 0.04 s, then restores it. Cheap visible damage feedback with no dependencies.

!!! warning "Spelling: `Overide_Damage_Receiving`"
    `combat_body_2d.gd:6, 31` misspells "Override". Because it's an exported var name, the typo is now baked into every `.tscn` that touches the property. Renaming requires a project-wide scene rewrite — leave it.

!!! warning "Damage_Receiver auto-detect gate is a tautology"
    `combat_body_2d.gd:18` reads `if Damage_Receiver != DamageReceiver2D:` — comparing an instance against the *class*. The intent was `== null`. The branch is always taken, so the child-scan runs every `_ready()` whether or not the inspector set `Damage_Receiver`. Behaviorally harmless (the assignment is the same either way) but reads as broken control flow.

## DamageArea2D

`addons/Combat_system/Scripts/damage_area_2d.gd` — the hitbox. Trivially small:

```gdscript
class_name DamageArea2D
extends Area2D

@export var Damage: int = 10
signal damage_dealt(amount: int)

func _ready() -> void:
    self.area_entered.connect(deal_damage)

func deal_damage(argument_1) -> void:
    damage_dealt.emit(Damage)
```

The author's inline comment (`damage_area_2d.gd:17-19`) acknowledges that `deal_damage` does not actually apply damage — it only emits a signal. The receiver is responsible. This split is intentional: the hitbox decides *Damage*, the receiver decides whether the hit is *valid* (pierce, `Overide_Damage_Receiving`). Team filtering happens earlier — at the engine's collision-layer level — so by the time a `DamageArea2D` reaches a `DamageReceiver2D`, the teams are already correct.

`HitboxDamageArea2D` (the Unit subclass) overrides this with much more elaborate pierce and area-attack logic (covered in [Unit Hierarchy](02-Unit-Hierarchy.md)).

## DamageReceiver2D

`addons/Combat_system/Scripts/damage_receiver_2d.gd` — the hurtbox. The most substantive class in the addon.

Three responsibilities:

1. Detect a `DamageArea2D` overlap and route the damage to the host Unit's `take_damage`.
2. Configure its own collision layer/mask based on the Unit's `team_direction`.
3. Configure the sibling `FrontDetector` Area2D (used by Range Detectors) on the same team basis.

`_ready()` (lines 7–34) is the integration point:

```gdscript
func _ready() -> void:
    self.area_entered.connect(take_damage)
    _unit_ref = UnitNodePackage.find_unit(self)
    if not _unit_ref:
        push_warning(name, ": no CharacterBody2D ancestor found.")
        return
    self.damage_received.connect(_unit_ref.take_damage)

    await get_tree().create_timer(0.1, true, true).timeout
    var which_team = _unit_ref.team_direction
    set_collision_layer_value(1, false)
    set_collision_mask_value(1, false)
    if which_team < 0:                       # Ally
        set_collision_layer_value(3, true)
        set_collision_mask_value(7, true)
    else:                                    # Enemy
        set_collision_layer_value(6, true)
        set_collision_mask_value(2, true)
    var front_detector = get_parent().get_node_or_null("FrontDetector")
    if front_detector:
        front_detector.set_collision_layer_value(1, which_team < 0)
        front_detector.set_collision_layer_value(5, which_team > 0)
    _unit_ref.nudge_hurtbox(self)
```

### The collision layer convention

| Layer | Decimal | Role |
|---|---:|---|
| 1 | 1 | Ally `FrontDetector` (target for ally StandRay/AttackRay) |
| 2 | 2 | Ally hitbox (`HitboxDamageArea2D`) |
| 3 | 4 | Ally hurtbox (`DamageReceiver2D`) |
| 5 | 16 | Enemy `FrontDetector` |
| 6 | 32 | Enemy hurtbox (`DamageReceiver2D`) |
| 7 | 64 | Enemy hitbox (`HitboxDamageArea2D`) |

Allies own layers 1–3, enemies own layers 5–7. Layers 4 and 8 are reserved for future per-team features (vision, healing range). The hitbox/hurtbox layers are deliberately **not symmetric** across teams (ally hitbox=2, hurtbox=3 vs. enemy hitbox=7, hurtbox=6) so that masks set in one direction can't accidentally double-resolve in the other.

This convention is documented in the team vault (`OBSIDIAN-TEAM-VAULT/Production Notes/Unit Package System - Collision Layers.md`) and codified in code in three places: this file, `HitboxDamageArea2D.gd:22-31`, and `RangeDetectors.gd:29-33`.

### Hit pipeline

```
DamageArea2D enters DamageReceiver2D
        │
        ▼
take_damage(area)              # damage_receiver_2d.gd:42
        │
        ├── area.tick_up_pierce_calls()   # cooperate with pierce-distribution
        ▼
confirm_damage(area)           # if area.using_pierce_on() returns true
        │
        ▼
damage_received.emit(area.Damage)
        │
        ▼
_unit_ref.take_damage(damage)  # connected at line 18
        │
        ▼
CombatBody2D.take_damage → register_damage → clampi/emit → flicker
```

`using_pierce_on()` is owned by `HitboxDamageArea2D` — the receiver asks the hitbox whether this particular contact should consume a pierce slot. This split keeps pierce accounting on the hitbox side (which knows how many targets it can hit) while keeping team-validity and override on the receiver side.

!!! warning "100 ms hardcoded delay"
    `damage_receiver_2d.gd:20` awaits `create_timer(0.1, true, true).timeout` before reading `_unit_ref.team_direction`. The wait is a band-aid: `team_direction` is set by `Ally._ready()` / `Enemy._ready()`, which may not have run yet when the receiver hits `_ready()`. Any hurtbox collision in the first 100 ms after spawn will miss because the layer/mask is still the default (layer 1, mask 1). For most units this is invisible — they haven't reached the enemy yet. For pre-placed units in `temporary_test_scene.tscn` it could matter on the first frame.

## Architectural choices

### Why split hit / hurt / body across three classes

A single "Combat Entity" class would be simpler but couples the visual root (CharacterBody2D), the damage-receiving Area2D, and the damage-sending Area2D into one node. The split lets:

- A `CombatBody2D` exist without a hurtbox (an invulnerable training dummy).
- A `CombatBody2D` exist without a hitbox (a base / destructible).
- A unit have *multiple* `DamageArea2D` children (multi-hit attacks, AoE auras).
- The hurtbox and the body to belong to different scenes — a unit's `DamageReceiver2D` lives inside the `give_hurtbox_pack.tscn` Unit Package, not on the unit's root scene.

This payoff is why `Unit` doesn't just inherit from `Area2D` + `CharacterBody2D` — Godot doesn't allow multiple inheritance, and the split makes the composition explicit.

### Why signals + ancestor walk, not exports

`DamageReceiver2D` finds its host Unit via `UnitNodePackage.find_unit(self)` (`damage_receiver_2d.gd:13`) — a static helper that walks `get_parent()` up until it finds a `CharacterBody2D`. Then it connects `damage_received` to `_unit_ref.take_damage`.

The alternative would be an `@export var owner_unit: Unit`. The walk-and-connect approach lets the receiver live inside a packaged `.tscn` (`give_hurtbox_pack.tscn`) that doesn't know which unit it's been dropped onto. This is the central enabling pattern of the Unit Package System — see [Unit Hierarchy](02-Unit-Hierarchy.md) and the vault's `Unit Package System - Architecture Fix.md`.

## Where it lives

| Project | Copy | Status |
|---|---|---|
| `tsop---unit-structure/addons/Combat_system/` | Reference copy | Live — used by `UnitClass`, `HitboxDamageArea2D`, etc. |
| `tso-rushed-stages-showcaes/addons/Combat_system/` | In sync | Live — used by older `Unit.gd` (still has bugs above the addon layer) |
| `Trident_Slime_Ops_Assembly/addons/Combat_system/` | In sync | Live — used by `UnitClass.gd` (same as proto-1) |

A `diff -r` across the three `addons/Combat_system/` directories returns no differences for `combat_body_2d.gd`, `damage_area_2d.gd`, `plugin.cfg`, or `Combat_system.gd`. `damage_receiver_2d.gd` is also in sync — the version that uses `UnitNodePackage.find_unit(self)` and the 100 ms team-direction wait.

!!! note "`damage_receiver_2d.gd` depends on code outside the addon"
    Despite living in `addons/`, `damage_receiver_2d.gd:13` references `UnitNodePackage` (which is defined in `Unit_System/node_packages/UnitNodePackage.gd`, not in the addon). The addon as-distributed is *bidirectionally coupled* to the project that ships it. `Combat_DEPENDENCY_README.txt` (in the Assembly build) only documents the one-way dependency. To extract the addon for a different project, you'd need to either bring `UnitNodePackage.gd` with it or refactor `find_unit()` into the addon.

## Carry-forward verdict

**Tier 1 — lift as-is.** The addon is the cleanest sub-system in the repo. Bugs above it (Unit-level movement, slot lookups, range detection) do not touch it. The two cosmetic smells (typo `Overide`, tautological `Damage_Receiver` gate) are inert in practice.

Issues to clean up before any extraction would be: the `UnitNodePackage` coupling in `damage_receiver_2d.gd`, the 100 ms `create_timer` wait, and the lack of a comment documenting the layer convention inside the addon itself (it's currently external in the vault).
