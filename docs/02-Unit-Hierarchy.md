# Unit Hierarchy

The Unit hierarchy is the middle layer between the [Combat System addon](01-Combat-System-Addon.md) and concrete unit scenes. It owns movement, the attack-state pipeline, knockback, the attack-priority animation gate, and the state machine.

This chapter documents the **current canonical version** — the rewritten `UnitClass.gd` that lives in `tsop---unit-structure/` and was re-exported into `Trident_Slime_Ops_Assembly/`. The older `Unit.gd` still living in `tso-rushed-stages-showcaes/` is the time-capsule; its differences are flagged in [Where it lives](#where-it-lives) below.

```
CombatBody2D                    (addon)
└── Unit  (class_name Unit, file UnitClass.gd)
    ├── Ally   (team_direction = -1)
    └── Enemy  (team_direction = +1, Stat_Magnification %)
```

## Class diagram

```
                     ┌────────────────────────────────┐
   UnitData (Node)   │           Unit                 │
   pushes stats ────►│  • Move, MAOT()                │
   on _ready         │  • Attack pipeline             │
                     │  • Knockback                   │
                     │  • State machine (StateNode)   │
                     │  • Animation priority gate     │
                     └────────────┬───────────────────┘
                                  │
                ┌─────────────────┼──────────────────┐
                ▼                 ▼                  ▼
           HitboxDamageArea2D  RangeDetectors    StateNode children
           (DamageArea2D)      (RayCast2D)       (MoonwalkState, …)
           self-registers      self-register
           as .attack_hitbox   as .attack_range_detector
                               / .stand_range_detector
```

Children **self-register** on the Unit during their own `_ready()`. The Unit no longer scans for them. This inversion of control is the central enabling pattern that lets Unit Packages drop in without the Unit knowing where each part is in the tree. See [§ Unit Package contract](#unit-package-contract).

## UnitClass.gd — the spine

`Unit_System/scripts/class_define/UnitClass.gd` (404 lines). `class_name Unit, extends CombatBody2D`.

### Stats it carries

Pushed in from `UnitData` (see [§ UnitData](#unitdata)), then modified by any child `UnitStatChanger`:

- **Survivability:** `Default_Health`, `max_health`, `current_health`, `Minimum_Health`, `Knockback_Count`, `Knockback_Distance`, `next_KB_health`.
- **Attack:** `Default_Damage`, `current_damage`, `Attacking_Range`, `Attack_Foreswing`, `Attack_Backswing`, `Attack_Cooldown`, `Attack_Hit_Distance`, `Attack_Hit_Duration`, `Attack_Hits_Area`.
- **Movement:** `Move_Speed`, `velocity` (inherited from CharacterBody2D).
- **Team:** `team_direction` (set by `Ally`/`Enemy`; -1 = ally, +1 = enemy).

### Combat state — two parallel state systems

Confusingly, Unit has **two** state representations:

1. **`CoreUnitStates` enum** (`UnitClass.gd:159-168`): IDLE / MOVING / FIGHTING / ATTACKING / KNOCKBACK / ACTING / DYING / OTHER. Stored in `CurrentCUS`. Read only by the knockback guard (`:188`). The author's own comment at `:177` admits: "As of now it's NOT CODED to be used in anything tbh."

2. **`StateNode` state machine** (`UnitClass.gd:98-159`): dictionary of named states, each represented by a `StateNode` child. The active machine (Backspace → MoonwalkState etc.). See [§ State machine](#state-machine).

The two systems do not interact directly — `CoreUnitStates` is bookkeeping the Unit sets on itself, while the `StateNode` machine is composition-driven from child nodes.

### Combat process loop

`_combat_process()` (`UnitClass.gd:176-200`) runs every physics tick when no state-machine state is active:

```gdscript
_process_knockback()
if CurrentCUS == CoreUnitStates.KNOCKBACK: return

if attack_range_detector:
    if attack_range_detector.is_colliding() && (started_attacking or in_standing_range):
        _fighting()
        skip_below_code = true
if skip_below_code: return

if stand_range_detector:
    if !in_standing_range && !started_attacking: MAOT()
    elif !started_attacking: _idle()
else: _idle()
```

Priority: **knockback** > **fighting** (attack-in-progress or in-attacking-range) > **moving** > **idle**. `MAOT()` is the movement step (named "Move At Other Team" per the source — never expanded).

`_fighting()` (`:218-222`) only sets `CurrentCUS = FIGHTING` — it doesn't play any animation. The current attack animation continues uninterrupted, which fixes the proto-2 bug where `_fighting()` called `_idle()` and reset the anim every frame.

### Movement — MAOT()

`UnitClass.gd:208-211`:

```gdscript
func MAOT() -> void:
    position.x += (team_direction * Move_Speed * 0.15)  ## zoomies factor
```

!!! warning "MAOT bypasses CharacterBody2D physics"
    `position.x +=` skips `velocity` and `move_and_slide()`. Collision masks set by other systems (`StaticBody2D` walls, other units) have no effect — units pass through each other and through solid bodies. The `0.15` multiplier was added recently to tame what the source calls "zoomies" — Move_Speed values authored in `UnitData` had become enormous when interpreted as pixels-per-frame at 60fps.
    `HeroUnitScript.gd` (the playable-test unit at `Unit_System/hero_unit_test_1/`) does it correctly with `velocity` + `move_and_slide()` — that pattern is the model for the eventual fix.

### Attack pipeline contract

The Unit doesn't drive its own attacks — an external `AttackNode` does (see [§ AttackNode](#attacknode)). Unit exposes a contract of three methods (`UnitClass.gd:260-277`):

| Method | Returns | What it does |
|---|---|---|
| `_begin_attack()` | `Attack_Foreswing` (float, seconds) | Sets `started_attacking = true`; subclasses can override to play a wind-up anim. |
| `_attack_continues()` | `[Attack_Backswing, Attack_Cooldown]` | Activates `attack_hitbox.damage_period(Attack_Hit_Duration)`. Hitbox does its damage during this window. |
| `_attack_ends()` | `void` | Clears `started_attacking`. |

AttackNode awaits the foreswing timer, calls `_attack_continues()`, awaits the backswing while the cooldown ticks in parallel, then calls `_attack_ends()`. The split lets the AttackNode own all timing (and the cooldown visualisation) while the Unit owns what happens during each phase.

### Damage and death

`UnitClass.gd:278-281`:

```gdscript
func take_damage(damage: int) -> void:
    super(damage)
    if current_health <= Minimum_Health: _process_death()
    if current_health <= next_KB_health: _start_knockback()
```

`super(damage)` chains to `CombatBody2D.take_damage` (the addon's flicker + clamp). The Unit-level override adds the two threshold gates that fire `_process_death()` and `_start_knockback()`.

`_process_death()` (`:293-297`) attaches a `MapleLeafDisappear` (`fade_unit_death_anim.gd`) child node and sets `is_dying = true`. The fade-out node should make the unit fade to alpha 0 over 100 ticks then `queue_free` itself. In Assembly's copy of that script, an off-by-one bug fires the `queue_free` on the first tick instead of after the fade — see the [Known Bugs Catalog](15-Known-Bugs-Catalog.md).

### Knockback

`_start_knockback()` (`UnitClass.gd:318`) sets `CurrentCUS = KNOCKBACK` and sets `velocity.x` to a launch value. `_process_knockback()` (`:302-314`) is gated behind an early `return ## FIX THIS ONCE UNITS ARE FIXED.` (line 303) — the actual knockback movement code is unreachable. Below the return, the implementation uses `move_toward` + `move_and_slide()` + `_play_anim("knockback", 3)` — i.e. physics-correct knockback. The early-return is presumably waiting on MAOT() to also use `move_and_slide()` so the two don't fight. Assembly's variant refines this further (see [Where it lives](#where-it-lives)).

`set_up_next_KB()` (`:338-364`) computes the next health threshold at which a knockback will fire: `next_KB_health = max_health * ((Knockback_Count - knockbacks_taken) / Knockback_Count)`. Recomputed by `get_children_things()` after UnitData and all UnitStatChangers have applied — so KB thresholds reflect post-buff `max_health`.

!!! warning "Enemy Stat_Magnification re-applies max_health AFTER set_up_next_KB"
    `Enemy._ready()` calls `super()` (which runs `get_children_things` and `set_up_next_KB`), THEN multiplies `max_health` by `Stat_Magnification / 100`. The KB thresholds end up calibrated to pre-magnification HP — a 200%-magnified enemy hits its first KB threshold "late" relative to its actual max health. See `Enemy.gd:8-15`.

### Animation priority gate

`_play_anim(name_string, priority)` (`UnitClass.gd:236-248`) is the only way to start an animation on a Unit's `AnimationPlayer`. Logic:

```gdscript
if not animation_handler: return
if not animation_handler.has_method("play_anim"): return
if not animation_handler.has_animation(name_string): return
if animation_handler.current_animation == name_string: return
if priority < last_anim_priority: return
animation_handler.play_anim(name_string)
last_anim_priority = priority
```

Higher-priority animations preempt lower-priority ones. `last_anim_priority` is reset to 0 by `on_animation_ended` (driven by an `AnimationRelayer`, see [Animation Relay](05-Animation-Relay.md)). This decoupling layer means Unit doesn't talk to `AnimationPlayer` directly — it talks to a relayer child whose `play_anim()` translates canonical names ("attack", "moving", "death") to whatever animation track names the artist happened to use.

## UnitData

`Unit_System/scripts/UnitDataClass.gd` (77 lines). `class_name UnitData, extends Node`.

A bag of exported stats with a single push-to-parent function. The pattern lets stat tuning happen in the inspector (or in a per-unit `.tscn`) without subclassing the Unit class for every variant.

Categories (`UnitDataClass.gd:5-31`):

- **Survivability:** `Default_Health` (100), `Move_Speed` (13), `Knockback_Count` (20), `Knockback_Distance` (300).
- **Attack:** `Default_Damage` (10), `Attacking_Range` (70), `Attack_Foreswing` / `_Backswing` / `_Cooldown` (1.0 each), `Attack_Hits_Area` (false).
- **Misc:** `Use_Standing_Range`, `Standing_Range`, `Use_Hit_Distance`, `Attack_Hit_Distance`, `Attack_Hit_Kilter` (unwired), `Attack_Hit_Duration`, `Unit_Type_Name`.

`parent_uses_data()` (`:40-72`) pushes every value onto `get_parent()`. Guard: `if not get_parent() is Unit: free()` (`:44`).

!!! warning "Attack_Hit_Kilter is exported but never read"
    Documented as NOT WIRED at `UnitDataClass.gd:24`. Same for the legacy `Attack_Hit_Distance` use when `Use_Hit_Distance = false`.

## Ally and Enemy

Both subclasses are minimal:

`Ally.gd` (13 lines):

```gdscript
extends Unit
class_name Ally
@export var Unit_Level: int = 1   # not yet used
func _ready() -> void:
    super()
    team_direction = -1
```

`Enemy.gd` (16 lines):

```gdscript
extends Unit
class_name Enemy
@export var Stat_Magnification: int = 100   # percent
func _ready() -> void:
    super()
    team_direction = 1
    current_damage = int(Default_Damage * Stat_Magnification * 0.01)
    max_health = int(max_health * Stat_Magnification * 0.01)
    current_health = int(current_health * Stat_Magnification * 0.01)
```

That's it for the team-direction split — combat behaviour is unified in `Unit`, plus the magnification multiplier on Enemy.

## Child node scripts

These live in `Unit_System/scripts/child_nodes/` and are documented per-file in the Assembly research; the canonical contracts are below.

### AttackNode

`AttackNode.gd` (63 lines). Plain `Node` (not a `UnitNodePackage`). Owns two child Timers (`SwingTimer`, `AttackCooldown`) inside `basic_attack_node.tscn`.

Drives the attack pipeline using the Unit's `_begin_attack` / `_attack_continues` / `_attack_ends` contract. Sequence in `do_entire_attack_sequence_v0()`:

1. Start `SwingTimer` with `parent._begin_attack()` (foreswing seconds).
2. Await `SwingTimer.timeout`. Call `parent._attack_continues()` → `[backswing, cooldown]`.
3. Start `AttackCooldown` with `cooldown`, start `SwingTimer` with `backswing`.
4. Await `SwingTimer.timeout`. Call `parent._attack_ends()`.

Gated on `check_if_parent_okay()` (`:48-62`): `in_attacking_range`, `!started_attacking`, `!running_a_state`, both timers idle.

### HitboxDamageArea2D

`HitboxDamageArea2D.gd` (132 lines). `extends DamageArea2D`. The attack-side hitbox.

On `_ready()` (`:5-31`): finds host via `UnitNodePackage.find_unit(self)`, sets `_unit_ref.attack_hitbox = self` (this is how Unit gets its hitbox reference), creates a child `CollisionShape2D` + `SegmentShape2D`, configures collision layer based on `team_direction` (ally → layer 2 / mask 6, enemy → layer 7 / mask 3).

Per-tick (`_physics_process`, `:33-42`): sets `Damage = _unit_ref.current_damage`; stretches `SegmentShape2D.b` to `Vector2(team_direction * length, 0)` where length is `Attack_Hit_Distance` if `Use_Hit_Distance` else `Attacking_Range`.

`damage_period(duration)` (`:44-51`) activates → `await create_timer(duration).timeout` → deactivates. Called from `Unit._attack_continues()`.

Pierce: `attack_pierce`, `current_pierce`, distribution chain `append_hurtbox` → `check_for_closest` → `tick_up_pierce_calls` → `using_pierce_on`. Two modes:

- `Attack_Hits_Area = true`: `attack_pierce = 99`, AoE.
- Otherwise: collect all hurtboxes in range this tick, pick the nearest, consume one pierce. Repeat for `current_pierce` slots.

### RangeDetectors

`RangeDetectors.gd` (52 lines). `extends RayCast2D`. Toggleable by `@export var Stand_or_Range: bool` — `true` = AttackingRange, `false` = StandingRange.

On `_ready` via `refresh_ranges()` (`:17-44`): finds Unit, writes `_unit_ref.attack_range_detector = self` or `.stand_range_detector = self`, awaits one frame so UnitData has finished pushing stats, sets `collision_mask` (ally → 16 to detect layer 5, enemy → 1 to detect layer 1), sets `target_position = Vector2(0, attack_range)` or `(0, stand_range)`.

Per-tick (`_physics_process`, `:46-51`): rotates the ray toward the other team, writes `_unit_ref.in_attacking_range` / `.in_standing_range` based on `is_colliding()`.

## Unit Package contract

A Unit Package is a `Node2D` (extending `UnitNodePackage`) that lives as a child of a Unit and contains a bundle of related child nodes. Examples ship as `.tscn` files in `Assembly/functionality/Combat_Units_System/Unit_Parts/node-packages/`:

- `fundamental/give_hurtbox_pack.tscn` — `DamageReceiver2D` + capsule shape + `FrontDetector` Area2D.
- `fundamental/allow_attacking_pack.tscn` — `AttackRay` + `HitboxDamageArea2D` + `AttackNode` + cooldown debug display.
- `fundamental/allow_movement_pack.tscn` — `StandRay`.
- `display_unit_health_pack.tscn` — HP label and bar.

The `fundamental/` subfolder groups the packs needed for a baseline combat-capable unit (Hurtbox + Movement + Attacking); the health display lives one level up because it's optional.

The key contract is in `UnitNodePackage.gd`:

```gdscript
static func find_unit(from: Node) -> Node:
    var n = from.get_parent()
    while n and not n is CharacterBody2D:
        n = n.get_parent()
    return n
```

Every package child calls `UnitNodePackage.find_unit(self)` to get a reference to the host Unit, regardless of how many package layers wrap it. The Unit recursively scans for relevant children (`UnitClass._scan_for_states`) when it needs to register state nodes.

The full development pattern, the history of how this came to be, and a list of bugs fixed along the way is in the vault (`OBSIDIAN-TEAM-VAULT/Production Notes/Unit Package System - *.md`) — extracted into [Team Vault — Production Notes](12-Vault-Production-Notes.md). The teaching version is the *Developer Guide*; the post-mortem version is the *Architecture Fix*.

## State machine

`StateNode` (`Unit_System/scripts/class_define/StateNodeClass.gd`, `class_name StateNode extends Node`) is the base class for special states. On `_ready` it walks up to its host Unit via `find_unit()` and triggers the Unit's `_refresh_special_states()` if it isn't already registered. The Unit's `node_states` dictionary maps state names (the node's name, as a StringName) to StateNode instances.

A state transitions by emitting `call_new_state.emit("OtherStateName")` — connected to `Unit.parent_transition_to()` (`UnitClass.gd:135-153`). The string `"none"` is the sentinel for "exit the state machine and return to normal combat".

The only concrete state in the repo is `MoonwalkState`, present in both proto-1 (`Unit_System/scripts/child_nodes/MoonwalkState.gd`) and Assembly (`functionality/Combat_Units_System/Scripts/one-offs/MoonwalkState.gd`). Triggered by Backspace, slides backward for 60 ticks, then emits `call_new_state.emit("none")`.

## Where it lives

| Project | Version | Notes |
|---|---|---|
| `tsop---unit-structure/` | **Canonical** — this chapter | `UnitClass.gd` (404 lines), `StateNode`, `UnitNodePackage`, `HeroUnit`, `MoonwalkState`, segment-shape hitbox, layer-routed collision, knockback skeleton |
| `Trident_Slime_Ops_Assembly/` | Proto-1 + refinements + `UnitStatChanger` | 427 lines. See [Assembly deltas](#assembly-deltas) below and the full Assembly chapter ([Assembly](11-Assembly.md)) |
| `tso-rushed-stages-showcaes/` | **Time capsule** — older `Unit.gd` | 163 lines. Still has: `child = attack_hitbox` swap-bug (`:55-59`), `_fighting()` calls `_idle()`, AttackNode/HitboxDamageArea2D `get_class()` self-destruct bugs (`AttackNode.gd:13`, `HitboxDamageArea2D.gd:7`), empty `_process_knockback()`. No StateNode, no UnitNodePackage, no HeroUnit. |

Bugs above are documented per location in the [Known Bugs Catalog](15-Known-Bugs-Catalog.md).

### Assembly deltas

Specific differences from the proto-1 canonical version documented above. Citations are against `Trident_Slime_Ops_Assembly/functionality/Combat_Units_System/Scripts/of-new-classes/UnitClass.gd`.

| Delta | Assembly citation | Effect |
|---|---|---|
| Brightness flash on hit | `_physics_process:76-80` + `take_damage:285` | Each `take_damage` sets `strange_brightness_value = 1.35` and recolors `$UnitGraphics` (`take_damage:282-291`); `_physics_process` decays the brightness toward 1.0 by 0.1 per frame. **Not present in proto-1.** |
| `take_damage` extended | `:282-291` (vs proto-1 `:278-281`) | Same 4 logical checks, plus the brightness flash above. |
| `_process_death()` moved | `:300` (vs proto-1 `:293-297`) | Identical behavior — line shift only. |
| Knockback rewritten | `:308-324` (proto-1 `:302-314`) | Replaces proto-1's `move_toward` with `velocity`-driven sliding: explicit `kb_destination_pos_x` target, exponential decay `velocity.x *= 0.95`, plus `move_and_slide()`. From the most recent commit `52569d8 Smoother Unit Knockbacks`. |
| Knockback direction flipped | `:330` | `Knockback_Distance * -1` — units slide backward instead of forward. |
| `set_up_next_KB` rebased | `:352-364` (proto-1 `:338-364`) | Now reads `max_health` instead of `Default_Health` so KB thresholds track post-buff HP. |
| StatChanger integration | `get_children_things():417-426` | Collects all child `UnitStatChanger` nodes into `usc_array`, applies them after UnitData, then re-runs `set_up_next_KB()` so thresholds reflect post-change `max_health`. |
| `_attack_continues` greyout removed | `:270, 277` (commented out) | Proto-1 had `set_modulate(Color(0.537,...))` to dim during the attack window; Assembly comments it out. |
| Debug KB trigger removed | proto-1 `:180-181` had `if Input.is_key_pressed(KEY_ENTER): _start_knockback()`; Assembly drops it. |
| Path moved | `functionality/Combat_Units_System/Scripts/of-new-classes/UnitClass.gd` (vs proto-1 `Unit_System/scripts/class_define/UnitClass.gd`) | Assembly partitions scripts by role; the file is the same logical script. |
| `UnitStatChanger` added | `of-new-classes/UnitStatChangerClass.gd` + three subclasses | New class layer not in proto-1 — see [Assembly](11-Assembly.md) for the full treatment. |

## Carry-forward verdict

**The proto-1 Unit architecture (UnitClass + StateNode + UnitNodePackage + RangeDetectors with `find_unit()`) is Tier 1** — it should be the canonical version in any future TSO build. Assembly already does this.

**Proto-2's Unit code is Tier 3 (reference / history)** — superseded by proto-1's rewrite. Its bug list serves as a "what we fixed" record but the code itself should not be lifted.

Open issues blocking a full Tier-1 lift:

1. `MAOT()` still uses `position.x +=` — needs `velocity` + `move_and_slide()` per the HeroUnit pattern.
2. Enemy `Stat_Magnification` re-applies `max_health` after `set_up_next_KB` — recompute KB thresholds after magnification.
3. `_process_knockback()` has working code below an early-return guard — remove the guard once MAOT is physics-based.
4. `Attack_Hit_Kilter` exported but unwired.
