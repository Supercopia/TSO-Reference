# Animation Relay

The animation-relay layer decouples gameplay code from `AnimationPlayer` resources. Gameplay says "play `attack`" with a priority number; the relayer translates `attack` to whatever the artist named the animation (`basic_attack`, `attack`, `swing_v3`), filters out re-trigger spam, and respects priority gating.

This is one of the cleanest patterns in the repo. It was prototyped in `tsop---character-animations/` and is the model the [Unit Hierarchy](02-Unit-Hierarchy.md) `_play_anim()` gate is built around.

## Setup

A unit's scene tree at the relay layer:

```
Unit (Node2D / CharacterBody2D)
├── animation_handler  (Node2D + AnimationRelayClass.gd, class_name AnimationRelayer)
│   └── AnimationPlayer (Godot built-in, holds the named animations)
└── ...
```

The Unit holds a reference to `animation_handler`. Gameplay calls `_play_anim(name, priority)` on the Unit. The Unit forwards to `animation_handler.play_anim(name)` after its own priority check (see [Unit Hierarchy § Animation priority gate](02-Unit-Hierarchy.md#animation-priority-gate)). The relayer does the dictionary lookup and tells the `AnimationPlayer` what to actually play.

## AnimationRelayClass.gd

`tsop---character-animations/old_shipment/unit_animations/AnimationRelayClass.gd`. `class_name AnimationRelayer extends Node2D`. 80 lines.

```gdscript
@icon("res://slime_icon.svg")
extends Node2D
class_name AnimationRelayer

signal animation_ended
var animation_player: AnimationPlayer
var PlayableAnims: Dictionary = {
    "RESET":          "RESET",         # always first
    "forward_move":   "moving",
    "basic_attack":   "attack",
    "moment_staggered": "knockback",
    "moment_ded":     "death",
    # plus alternate canonical names → same anim
}
```

`_ready()` (`AnimationRelayClass.gd:30-45`):

1. Find child `AnimationPlayer` via `is_class("AnimationPlayer")` walk.
2. For each canonical key in `PlayableAnims`, scan `animation_player.get_animation_list()` for a track whose name ends with the dictionary value (`forward_move` matches `walk_forward_move`, etc.).

This **suffix-match** lookup is the relay's main trick. Two animation files with different naming conventions can both be played by the same gameplay code as long as the relayer's dictionary value appears as a suffix of the actual track name.

`play_anim(name_string)` (`:48-59`): looks up `PlayableAnims[name_string]`, finds the matching track, calls `animation_player.play(track)`. Two cases (`death`, `knockback`) are commented out at this layer — disabled at the play path even though they're present in the dictionary.

`has_animation(name_string)` (`:69-76`):

```gdscript
func has_animation(name_string: String) -> bool:
    var afk_value = PlayableAnims.get(name_string)
    if afk_value == null:
        return                          # bare return — yields null
    if not animation_player.has_animation(afk_value):
        return false
    return true
```

`on_animation_ended()` (`:61-63`): instead of emitting `animation_ended`, pokes `get_parent().last_anim_priority = 0` directly. Resets the priority gate on the Unit, allowing the next animation to play.

## Bugs

!!! warning "Null-deref on missing AnimationPlayer (`AnimationRelayClass.gd:37`)"
    `_ready` does `if not animation_player.is_class("AnimationPlayer")` at line 32 — but `animation_player` is `null` if no AnimationPlayer child exists. The check at line 37 will deref null. Should be `if animation_player == null or not animation_player.is_class(...)`.

!!! warning "`has_animation()` returns `null` instead of `false` (`:73`)"
    The bare `return` on line 73 (in the `if afk_value == null` branch) yields `null`, not `false`. Callers expecting a bool comparison (`if not relay.has_animation(...)`) will misbehave because `null != false` in some GDScript contexts (and falsy in others). Fix: `return false`.

!!! warning "Signal `animation_ended` declared but never emitted (`:13`)"
    The class declares `signal animation_ended` but only ever uses `on_animation_ended` (a method) to poke the parent's `last_anim_priority`. The signal is decorative — anything connected to it will never fire.

!!! warning "`death` and `knockback` are disabled at the play path (`:48-59`)"
    Two of the six canonical names that the dictionary registers are commented out in `play_anim()`. They appear in `has_animation` checks but can't actually be played.

## Other files in the prototype

`tsop---character-animations/` is a sandbox built around the relay pattern. Key files:

- `practice_unit.gd` — Node2D test unit. Discovers its animation handler via `has_method("play_anim")` (`:11-14`), then maps arrow keys to anim names in `_physics_process` (`:17-30`): `ui_up` → idle, `ui_down` → attack, `ui_left`/`right` → moving. Defines its own `_play_anim` (`:37-50`) mirroring `UnitClass._play_anim`'s priority gate, so the relay pattern can be exercised without the full Unit class.
- `practice_unit_anims.tscn` — depends on `old_shipment/AnimationRelayClass.gd` — meaning the live test scene **depends on a file in an `old_shipment/` (archive) folder**. Promoting the script out of `old_shipment/` is the structural fix; until then, "cleaning up" the archive risks breaking the scene.
- `akds_StatesAnimPlayer.gd` — 70 lines of design comments only. Not active.
- `cuteProjectorAnimV0.gd` — orphaned signal (`signal show_hud` declared and emitted, never connected).

### old_shipment/

Five generations of slime anim `.tscn` files: `proto_basicslime`, `proto2_basicslime`, `proto_newspaperslime`, `proto_fodderp`, `unit_v1_forced_basicslime`. Plus a folder candidly named **"i regret trying puppets triangle_smol"** for an abandoned puppet-rig approach. Three backdrop scenes under `level_backdrops/`.

The archive is named "old_shipment" — but the live test scene `practice_unit_anims.tscn` actively imports `AnimationRelayClass.gd` from inside it. Treat it as half-archive, half-live.

## Where it lives

| Component | Where |
|---|---|
| `AnimationRelayer` (the class) | `tsop---character-animations/old_shipment/unit_animations/AnimationRelayClass.gd` |
| `_play_anim` priority gate (the caller side) | `tsop---unit-structure/.../UnitClass.gd:236-248` and Assembly's equivalent |
| `AnimationPlayer` (empty) | `fodderp.tscn:38`, `second_slime_test.tscn:40` in Assembly — present but no tracks |

The relay pattern is referenced in `UnitClass.gd` via the `_play_anim()` method, but the actual `AnimationRelayClass.gd` script doesn't ship in `tsop---unit-structure/` or `Trident_Slime_Ops_Assembly/`. Those projects have empty `AnimationPlayer` nodes ready to host tracks; the relay layer hasn't been ported.

## Carry-forward verdict

**Tier 1 — the relay pattern.** Promote `AnimationRelayClass.gd` out of `old_shipment/` into a permanent location, fix the four bugs above, then port into `tsop---unit-structure/` and Assembly. The canonical names (`forward_move`, `basic_attack`, `moment_staggered`, `moment_ded`) are good. The suffix-match flexibility is good.

**Tier 3 — `old_shipment/` archive.** Keep the slime `.tscn` generations as reference; do not lift. The "i regret trying puppets" folder is a textbook negative result.

The relay layer is the missing piece that would make TSO's animations work without subclassing Unit per character — once it's properly integrated, a new slime can be a `.tscn` with a sprite, an `AnimationPlayer`, and a relayer child that maps the canonical names to whatever the artist authored.
