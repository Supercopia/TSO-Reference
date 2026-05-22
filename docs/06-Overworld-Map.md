# Overworld Map

`tsop---overworld-map/` prototypes the Battle Cats-style overworld where the player taps a stage tile and a cursor walks along a path to it. The prototype is a working *mechanic spike* — the cursor moves, the path supports dual-direction traversal, and stage tiles detect cursor proximity — but the scene-loading "now actually start the stage" handoff is a `print` stub.

This is the only TSO prototype still pinned to **Godot 4.5** (`project.godot:16`). The others are 4.6.

## Two scripts that matter

| Script | Class | Role |
|---|---|---|
| `Interactive_Overworld_Tile.gd` | `IOTile` | One stage tile. Holds the level name, scene UID, and a duck-typed handoff. |
| `OverworldPlayerPath.gd` | — extends `Path2D` | The path the cursor travels. Handles dual-direction input, auto-discovers child nodes, gates input on collision distance. |

## IOTile — the cleanest script in the repo

`test_scenes/Interactive_Overworld_Tile.gd`. 14 lines. `extends Marker2D`, `class_name IOTile`:

```gdscript
class_name IOTile
extends Marker2D

@export var stage_name: String
@export var stage_scene: PackedScene

func update_level_info(_stage_name, _stage_scene) -> void:
    print("stage_name is ", _stage_name)
    # Loading scenes is a whole headache, currently not implemented
```

That's the whole file. Pattern: the cursor's raycast hits an `IOTile`, walks up via `get_collider().get_parent()`, checks `has_method("update_level_info")`, and calls it. Pure duck-typed handoff — IOTile doesn't know anything about the cursor, the cursor doesn't know anything about IOTile.

The `update_level_info` body is a placeholder print. The real "load this stage" is unimplemented.

!!! note "Report inaccuracy"
    `PROTOTYPES_REPORT.md:458` says `update_level_info` prints `"Loading scenes is a whole headache..."`. Actually that string is a comment on the next line; the print is `"stage_name is ", _stage_name`. Caught in [verify-prototypes-3-8.md § A4](../_research/verify-prototypes-3-8.md#a4-tsop-overworld-map-19-claims-16-2-1-4).

## OverworldPlayerPath — the spaghetti

`test_scenes/OverworldPlayerPath.gd`. 198 lines. `extends Path2D`. Self-described as "Spaghetti" at line 76.

### Child discovery

`_ready()` (lines 21-34) walks children for a `PathFollow2D` and its `RemoteTransform2D`. If discovery fails, falls back to hardcoded `$PathFollow2D/RemoteTransform2D` and `$PathFollow2D`. The PathFollow2D drives the cursor along the curve via `progress_ratio`.

### Endpoint snapping

Lines 40-45 handle the 2-point and multi-point cases when computing the curve's start/end. The curve's first and last point determine where the cursor sits at progress 0.0 and 1.0.

### Dual-direction inputs

`OverworldPlayerPath.gd:8-17` exports two pairs of direction keys:

```gdscript
@export_enum("North", "East", "South", "West") var start_to_end_direction: int
@export_enum("north", "east", "south", "west") var end_to_start_direction: int
```

Uppercase = Start→End traversal, lowercase = End→Start. The input handler at lines 142-166 reads `get_parent().player_input_dir` (a String written by the cursor's input script) and increments/decrements `progress_ratio` accordingly.

!!! warning "Unguarded `get_parent().player_input_dir` access (4 sites)"
    Lines 142, 150, 158, 166. If the path is not a child of the expected cursor node, `player_input_dir` will not exist on the parent and the field read will crash. Documented in [verify-prototypes-3-8.md § A4](../_research/verify-prototypes-3-8.md#a4-tsop-overworld-map-19-claims-16-2-1-4).

### The 42.0 threshold

`input_suite()` (line 133) gates input on `collision_distance_away > 42.0`. The threshold is hardcoded with no symbolic name. Means: if the cursor is "close enough" to a tile, the path won't accept further movement input until the cursor walks away. Magic number; no comment.

### speed: 0.05 — declared and re-typed

Line 55 declares `var speed: float = 0.05`. Line 63 advances `progress_ratio += 0.05` using the **hardcoded literal**, not the variable. If `speed` is ever changed, line 63 won't see it. Latent bug — `PROTOTYPES_REPORT.md:457` flagged this; still present.

### Raycast collider lifetime

`ignore_weird_null()` (`:106-112`, with `null_object_straight_jail: Array` declared at `:105`) maintains an array of dead colliders. The hand-rolled defensive guard fixes a known issue where Godot's `RayCast2D.get_collider()` can return an instance whose underlying object is queue-freed mid-frame. The proper fix is `is_instance_valid(collider)` guards rather than maintaining a sidecar array. Lines 68 and 71 are the raycast usage sites.

### add_exception accumulation

`add_exception()` is called (lines 79-80) every time the cursor crosses a tile, but `clear_exceptions()` is never called. The exception list grows unbounded across a session. Minor leak; for a real game would need a reset point.

## Cursor

`overworld_player_cursor.gd` — movement stub. `overworld_cursor_animations.gd` — literally `extends AnimatedSprite2D` and nothing else (one line).

`first_contact_bool` + `first_contact()` (lines 91, 114-117) — one-shot debug logger that prints once when the cursor enters its first tile.

`cosmetic_arrow_setup()` (lines 176-197) — 22 lines positioning/rotating debug arrows that hint at the next valid direction. Visual polish only; doesn't gate logic.

## Showcase scene

`test_scenes/overworld_map_test_1.tscn` — the playable scene. Six `IOTile` instances and eight `OverworldPath2D` instances (`loopdeloop`, `pathA`, `pathB`, `overworld_path_2d`, `overworld_path_2d2..5`). One of the curves is the **9-point `loopdeloop`** (`Curve2D_6gkco.point_count = 9`) — a showcase curve demonstrating that paths can be non-monotonic.

## Where it lives

`tsop---overworld-map/` only. No part of the Overworld system has been ported into `Trident_Slime_Ops_Assembly/`. The vault's Menu Navigation plan puts overworld integration in **Iteration 2** (after the playable loop in Iteration 1 is done).

## Carry-forward verdict

**`Interactive_Overworld_Tile.gd` — Tier 1, lift as-is.** Cleanest script in the entire repo. Zero bugs. The duck-typed `update_level_info` pattern is good.

**`OverworldPlayerPath.gd` — Tier 2.** The dual-direction Path2D + PathFollow2D pattern is the right approach. The 42.0 magic number, the hardcoded 0.05 vs `speed` divergence, the unguarded `get_parent()` access, and the `add_exception` accumulation all need to be addressed before lifting. The "Spaghetti" self-comment is honest.

**Cursor scripts — Tier 3.** Movement stub, animation stub, `update_level_info` stub. Rewrite when integrating.

The fixes the vault calls out (vault `Menu Navigation - Implementation Plan.md`, Iteration 2):

1. Complete cursor pathfinding so it actually walks to a tapped tile.
2. Make `update_level_info` load the `stage_scene` into the Preparation Menu.
3. Distinguish Level tiles from Non-Level tiles (Open Base, B-Switch, Command Room).
4. Add a "go to Command Room" button always-visible on the map.
