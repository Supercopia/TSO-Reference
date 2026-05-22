# Spawn and Wave System

How units get into a stage. Two layers:

- **`UnitSpawnSystem`** — the runtime queue. Receives spawn requests from both sides and pops them onto the field one (or four) per physics frame.
- **`EnemySpawnStartwatch`** + **`config_set_off_all_timers`** + **`config_1.tscn`** — the wave-config layer. A `.tscn` of Timers that fires `queue_unit` calls into `UnitSpawnSystem` at scripted intervals.

Allies are spawned via a different path: [Stage Loop](04-Stage-Loop.md) covers `PMSV.request_ally()` and the player deck.

## UnitSpawnSystem

`Level_Functions/UnitSpawnSystem.gd` (108 lines). Autoload `UnitSpawnSys`. Owns two queues and two spawn positions.

!!! note "Line numbers in this section"
    Citations are against Assembly's version (the most-featureful copy; identical to proto-1's). Proto-2's `UnitSpawnSystem.gd` is a strict subset (74 lines) — see [Where it lives](#where-it-lives) for the deltas.

```gdscript
var AllySpawnQueue: Array = []
var EnemySpawnQueue: Array = []
var enemy_spawn_position: Vector2 = Vector2(-300, 0)
var ally_spawn_position:  Vector2 = Vector2( 300, 0)
```

### queue_unit

```gdscript
func queue_unit(unit_resource, which_side: int) -> void:
    if which_side == 1:        # ally
        AllySpawnQueue.append(unit_resource)
    elif which_side == -1:     # enemy
        EnemySpawnQueue.append(unit_resource)
```

`which_side` convention: **`1` = ally, `-1` = enemy**. This is the spawner's internal sign — **not** the same as `Unit.team_direction`, which uses the inverse (-1 for ally, +1 for enemy). The conventions are inverted across modules; `PlayerMidStageVault.request_ally()` correctly passes `1`, but reading the source you must remember which sign is in play.

### Spawn step

`_spawn_next_unit(for_side)` (`:23-69`) pops the queue head. Branches on resource type:

- `PackedScene` → `instantiate()`.
- `Dictionary` → `handle_intricate_spawn(unitDict)` (stubbed, see below).
- Otherwise → treat as `"res://"` path string, `load()` and instantiate.

Then `set_global_position(spawn_position)` and `add_child(next_spawn)`.

### Per-frame loop

`_physics_process` → `process_spawn_queues()` (`:74-81`):

```gdscript
for n in range(1, player_units_per_frame + 1):  # 4 ally spawns per frame
    _spawn_next_unit(1)
_spawn_next_unit(-1)                            # 1 enemy spawn per frame
```

The 4-per-frame ally burst is in proto-1 and Assembly. `tso-rushed-stages-showcaes/` uses 1-per-frame on both sides.

!!! warning "Units are added as children of the autoload"
    `UnitSpawnSystem.gd:68` calls `add_child(next_spawn)` on itself — `self` being the autoload Node. Units live under the autoload's node, not under the active scene. Two consequences:
    1. They persist across scene changes — would be a bug for level transitions, but the current build only has one scene.
    2. They aren't visible in the Remote scene tree under the playing stage — debugging needs to descend into the autoload.

!!! warning "`Vector2(Vector2)` cast is a no-op"
    `UnitSpawnSystem.gd:64` reads `next_spawn.set_global_position(Vector2(spawn_position))`. Wrapping a Vector2 in `Vector2()` does nothing. Cosmetic.

### handle_intricate_spawn (stub)

`:85-104`. Branches on dict keys — comments describe the intent: stat magnification for enemies, `activated_strives: Array` for allies. None implemented. The dictionary spawn branch in `_spawn_next_unit` accepts a Dictionary today but the receiver does nothing useful with it.

## EnemySpawnStartwatch

`Level_Functions/enemy_timers/EnemySpawnStartwatch.gd`. Per-wave Timer subclass.

```gdscript
extends Timer
@export var Unready: bool = true
@export var Enemy_To_Spawn: PackedScene
@export var Count: int = 1
@export var Start_Delay: float = 0.0

func _ready() -> void:
    if Unready:
        queue_free()      # disabled timers self-destruct on load
        return

func _on_timeout() -> void:
    Count -= 1
    UnitSpawnSys.queue_unit(Enemy_To_Spawn, -1)
    if Count <= 0:
        queue_free()
```

Each wave is one Timer with `wait_time`, `Count`, and an enemy PackedScene. `_on_timeout` queues one enemy per tick until `Count` runs out, then self-frees.

!!! warning "Default `Unready = true` means timers self-destruct unless explicitly turned on"
    `EnemySpawnStartwatch.gd:9-12`. Every Timer in a wave config must have `Unready = false` in its inspector slot — otherwise it `queue_free`s on first `_ready()` and spawns nothing.

!!! warning "`Start_Delay` is exported but never used"
    `EnemySpawnStartwatch.gd:7`. It's not passed to `wait_time`, not used as a one-shot delay. Decorative.

## config_set_off_all_timers

`Level_Functions/enemy_timers/config_set_off_all_timers.gd`:

```gdscript
func _ready() -> void:
    for child in get_children():
        if child is Timer:
            child.start()
```

Pure dispatcher. Attached to the root Node of a wave-config scene so all its child Timers start ticking when the scene loads.

## config_1.tscn

`Level_Functions/enemy_timers/config_1.tscn`. The only wave config in the repo. A Node with `config_set_off_all_timers.gd` and six `EnemySpawnStartwatch` Timer children.

```
config_1 (Node + config_set_off_all_timers.gd)
├── Timer  (wait_time=0.009, Count=…, Enemy_To_Spawn=…, Unready=true)
├── Timer
├── Timer
├── Timer
├── Timer
└── Timer
```

Each Timer's `timeout` signal is wired to its own `_on_timeout()` method (`config_1.tscn:47-52`).

!!! warning "All six timers in config_1.tscn ship with `Unready = true`"
    None of the six Timer node blocks override the default. On `_ready()` every timer `queue_free`s itself. **The config spawns nothing as shipped.** Confirmed at `config_1.tscn:47-52` and in [verify-prototypes-1-2.md § Level/stage systems](../_research/verify-prototypes-1-2.md#levelstage-systems---present-and-matching).

    To use config_1 as anything other than scaffolding, every timer needs `Unready = false` set explicitly in the inspector.

## Wave-config flow at runtime

```
Stage scene boots
        │
        ▼
config_1.tscn instanced as child of (typically) a Base node
        │
        ▼
config_set_off_all_timers._ready()
        │
        ├── for each Timer child:
        │     ├── if Unready: queue_free (the current bug)
        │     ├── else: start()
        │     │
        │     └── on each timeout:
        │           UnitSpawnSys.queue_unit(Enemy_To_Spawn, -1)
        │           Count -= 1; queue_free when Count <= 0
        │
        ▼
UnitSpawnSystem queues drain at 1 enemy/frame
        │
        ▼
Enemy unit instanced, positioned at enemy_spawn_position
        │
        ▼
Enemy.gd._ready() applies Stat_Magnification, then super().
```

## Where it lives

| Project | UnitSpawnSystem | EnemySpawnStartwatch + config_1 |
|---|---|---|
| `tsop---unit-structure/` | Present, 4 ally / 1 enemy per frame | **Not present.** No enemy-timer architecture in proto-1 |
| `tso-rushed-stages-showcaes/` | Present, 1 ally / 1 enemy per frame, diagnostic prints active. **74 lines** | Present. config_1.tscn here. Bases use it via `waiting_base_for_menu.gd` |
| `Trident_Slime_Ops_Assembly/` | Present (≈ proto-1 form, ally multi-spawn). **108 lines** | **Not present** — `QuickGameManager` / bases / waves not yet ported |

So the **wave system only runs in proto-2 today**. Proto-1 ships the spawner but has no scripted spawning UI; Assembly inherits the same gap.

### Proto-2 deltas vs the Assembly UnitSpawnSystem documented above

Proto-2's `Level_Functions/UnitSpawnSystem.gd` is a strict subset of Assembly's. Specific differences:

| Delta | Proto-2 | Assembly |
|---|---|---|
| File length | 74 lines | 108 lines |
| `handle_intricate_spawn` function | not present | `:85-104` (stubbed) |
| `process_spawn_queues()` body | proto-2 `:68-72`: two `if size() > 0: _spawn_next_unit(N)` calls — single spawn each side per tick | Assembly `:74-81`: `for n in range(1, player_units_per_frame + 1)` loop on ally side (4 per tick), single enemy spawn |
| Diagnostic prints | active at proto-2 `:59` | commented out in Assembly |
| `add_child(next_spawn)` | proto-2 `:62` | Assembly `:68` |
| `Vector2(spawn_position)` no-op cast | proto-2 `:58` | Assembly `:64` |
| `_spawn_next_unit` body | proto-2 `:23-63` | Assembly `:23-69` |
| `queue_unit` | proto-2 `:13-21` | Assembly `:13-21` (identical) |

So if you're reading proto-2's source, ignore the `handle_intricate_spawn` discussion above, the burst-loop excerpt, and shift the cited line numbers down by ~6.

## Architecture notes

The split into "queue + per-frame drain" (UnitSpawnSystem) vs. "scripted timers" (config_N.tscn) is intentional:

- The queue absorbs simultaneous spawn requests from multiple sources (multiple wave Timers, multiple ally button presses) without colliding.
- The per-frame cap prevents a single huge wave from spiking the frame.
- Timers are the *content authoring layer* — designers can build a wave in `.tscn` form by dropping Timers into a Node and tuning their `wait_time`/`Count`/`Enemy_To_Spawn` in the inspector.

The wave-config approach is shallow. It can't express:

- Conditional spawns ("when ally base at 50% HP, fire wave 2")
- Sequence dependencies ("after the boss intro, start the trash waves")
- Per-stage magnification multipliers

[Stage Data Format](07-Stage-Data-Format.md) covers the `from-data-to-stage` prototype, which is the design-stage successor to this system. It specifies a richer schema (`StartCondition`, `HeadsUpBeforeSpawn`, etc.) but ships no runtime — the dispatcher functions are all `pass` stubs.

## Carry-forward verdict

**`UnitSpawnSystem.gd` — Tier 1.** The queue mechanic is sound and used by both ally and enemy spawn paths. The autoload-as-parent quirk should be fixed (parent to current scene root instead), and the empty `handle_intricate_spawn` should either be implemented or removed.

**`EnemySpawnStartwatch.gd` + `config_set_off_all_timers.gd` — Tier 2.** Works, but the `Unready = true` default is a footgun (and bites config_1.tscn). Flipping the default to `Unready = false` would fix every existing config-style scene.

**`config_1.tscn` — discard.** The specific scene is broken (no timers run). The pattern is keep-able.

Long-term: replace with the `StageDataClass` schema once a dispatcher exists.
