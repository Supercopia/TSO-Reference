# Stage Data Format

`from-data-to-stage/` is a **design-only** prototype. It defines a per-enemy wave schema, demonstrates the `get_or_add` Dictionary method for safely populating nested dictionaries with defaults, and documents — in comments — what each field should mean. **The runtime side is not implemented.** Every condition / niche-spawn function is a `pass` stub.

It's worth a chapter because the schema is the eventual successor to the `EnemySpawnStartwatch` Timer-config pattern from [Spawn and Wave System](03-Spawn-and-Wave-System.md) — Timers can't express conditional spawning, this can.

## StageDataClass

`StageDataClass.gd` (194 lines). `extends Node`, `class_name StageInfoClass` (`:1-2`). Note the file / class name disagreement — the file is `StageDataClass.gd`, the class is `StageInfoClass`.

### The active schema: cleanerExampleEnemy

Source `:78-91`, quoted verbatim (whitespace normalized):

```gdscript
@export var cleanerExampleEnemy: Dictionary = {
    "DebugName":         "string",                # string; What their name is in the debug tools.
    "EnemyResource":     "res://string.tscn",     # res://string; What Enemy is spawned
    "Count":             1,                       # int; How many of them are spawned
    "Magnification":     1.00,                    # float; Stat magnification
    "StartCondition":    ["type", 100, 0],        # array; Function feeds ["type", values] array to whatever function can use it, to set Spawn Condition
    "SpawningTimes":     [1.0, 10.0],             # array; [0] = Delay for the first spawn, [1] = interval between spawns
    "UseNiche":          false,                   # bool; If set to false, does not use the following:
    "OnVisualLayer":     0,                       # int; set integer to what Z-index it has, relative to the Spawn Node.
    "IsABoss":           false,                   # bool; If true, then this enemy is a boss.
    "SpawnLocation":     0,                       # int; Which spawn point they spawn from. (0 for default)
    "SpecialSpawnAnim":  -1,                      # int; Which animation from their library they do before entering play. play_string("spawn_"+SpecialSpawnAnim)
    "HeadsUpBeforeSpawn":0,                       # array; [0] How many of them warn the player ahead of their spawn time, and [1] how long in advance the warning appears.
}
```

12 keys. Source values are template placeholders (`"string"`, `"res://string.tscn"`, `1`, `1.00`, `["type", 100, 0]`, etc.) — designers replace each with concrete per-enemy values.

!!! warning "`HeadsUpBeforeSpawn` schema/comment mismatch"
    The comment on `:90` says "array; [0] how many … [1] how long in advance", but the value supplied is the scalar `0`. The `HeadsUpBeforeSpawn(how_many_times, time_before)` function at `:189` accepts two arguments — neither pulls from this scalar usefully. See [HeadsUpBeforeSpawn schema inconsistency](#headsupbeforespawn--schema-inconsistency) below.

### The legacy schema: OneEnemyExample

Source `:22-46`. A plain `var` (not `@export`ed). Earlier draft of the schema with fewer keys (`Name`, `Count`, `StartCondition`, `Magnification`, `SpawningTimes`, `SpawnLocation`, `SpawnAnim`, `HeadsUpBeforeSpawn`) and different conventions:

- `"Name"` instead of separate `DebugName` + `EnemyResource`.
- `"SpawnAnim": [0, 0]` (array) instead of `"SpecialSpawnAnim": -1` (scalar).
- `"HeadsUpBeforeSpawn": [0, 0]` (array) instead of scalar.
- No `UseNiche`, `OnVisualLayer`, or `IsABoss`.

Both schemas coexist in the file; the chapter docs `OneEnemyExample` as history, `cleanerExampleEnemy` as current.

### Dead `@export`s

- `@export var SchematicForLevel` (`:11-19`) — declared, never read.
- `@export var testDictionaryDouble` (`:99-104`) — declared, only used as a target by the dead test functions below.

### Dead functions

Five functions defined but never called from `_ready()` (`:111-115`). The `_ready` body has the four other calls commented out, only `get_or_add_flex()` runs:

```gdscript
func _ready() -> void:
    #dictionary_assign_test()
    #get_or_add_test_3galoo()
    get_or_add_flex()
    #_test_this_stuff() # Does a test print
```

Dead functions:

- `_test_this_stuff()` (`:105-109`) — prints `testDictionaryDouble`.
- `dictionary_assign_test()` (`:117-123`) — demonstrates that `Dictionary.assign()` replaces all keys of the target with values from the source.
- `get_or_add_test_1()` (`:125-128`) — adds a `"nonexistent_key"` mapping to `testDictionaryDouble`.
- `get_or_add_test_2()` (`:131-136`) — adds to a separate `empty_dictionary_1`.
- `get_or_add_test_3galoo()` (`:138-142`) — adds two enemies to `empty_dictionary_1`.

## get_or_add pattern

`get_or_add()` is a built-in `Dictionary` method. From source comments:

> "If the first variable isn't returned, add the second one, and return that value."

Equivalent to `dict[key] if key in dict else (dict[key] = default; default)`. Avoids accidentally clobbering an existing value.

`get_or_add_flex()` (`:144-154`) is the only function `_ready` actually calls. It hardcodes eight entries into `empty_dictionary_1`:

```gdscript
func get_or_add_flex():
    empty_dictionary_1.get_or_add("firstEnemy",  cleanerExampleEnemy)
    empty_dictionary_1.get_or_add("secondEnemy", cleanerExampleEnemy)
    empty_dictionary_1.get_or_add("thirdEnemy",  cleanerExampleEnemy)
    ...  # fourthEnemy through eighthEnemy
    print(empty_dictionary_1)
```

The function is a demonstration — it loads the same template into eight slots. A real dispatcher would have one entry per wave with distinct values.

`dictionary_assign_test()` (`:117-123`) documents a separate footgun — `Dictionary.assign()` replaces all keys of the target dict with values from the source. Useful to know if you ever reach for `assign()` thinking it merges. The function itself does nothing useful at runtime since it's never called.

## StartCondition kinds

The schema's `StartCondition` field is an `[kind, args...]` array. Inferred from the condition predicates defined at the bottom of the file (`:157-178`):

| Kind | Function | Args | Meaning |
|---|---|---|---|
| (none / default) | `lazy_constant()` (`:164-165`) | — | "Just do the default one" — pass stub; intended no-op for waves that need no trigger. |
| `OnBaseHealth` | `WhenEB_Health(percentage, levelBase)` (`:168-171`) | `pct, base_index` | Trigger when an Enemy Base's HP falls below `pct`%. `levelBase` defaults to `0` (the first base). |
| `OnObjectHealth` | `WhenObjectHealth(percentage, levelObject)` (`:173-174`) | `pct, object_index` | Same, against a pre-placed Object's HP. If no Object exists yet, the spawner should hold the wave in a "special waiting queue". |
| `OnBossHealth` | `WhenBossHealth(percentage, levelBoss)` (`:176-177`) | `pct, boss_index` | Same, against a Boss's HP. Same waiting-queue note as above. |

All four are `pass` stubs. The header comment at `:160-161` says: "the 'WhenHEALTH' trio all reference an array that the level keeps track of. (There's an array for EnemyBases, Objects, and Boss Enemies.)" — so the implementation plan is for the level to maintain three parallel arrays the condition functions index into.

The dispatcher (which would walk every wave entry each tick, evaluate `StartCondition`, and fire `UnitSpawnSys.queue_unit(...)` on transition) does not exist.

## Niche spawn functions

Two more stub functions live under "NICHE SPAWNING STUFF" at `:181-194`:

- `SpecialSpawnAnim(which_one: int = -1)` (`:186-187`) — early-returns if `which_one < 0`. The schema's `SpecialSpawnAnim` field (default `-1`) feeds this; intended to play `"spawn_" + str(which_one)` from the unit's animation library before it enters play.
- `HeadsUpBeforeSpawn(how_many_times, time_before)` (`:189-194`) — `pass`. Intended to display a warning indicator at the spawn point `time_before` seconds before the spawn fires, optionally only the first `how_many_times` times.

Neither runs today.

## HeadsUpBeforeSpawn — schema inconsistency

The function signature is `HeadsUpBeforeSpawn(how_many_times: int, time_before: float)` — two scalar args. Source comments describe the wave-entry field as `array; [0] how many of them warn the player ahead of their spawn time, and [1] how long in advance the warning appears.` — i.e. a 2-element array.

But:

- `cleanerExampleEnemy["HeadsUpBeforeSpawn"]` (`:90`) is the scalar `0`.
- `OneEnemyExample["HeadsUpBeforeSpawn"]` (`:44`) is `[0, 0]` (matches the comment).

Either the new schema needs to revert to the array form, or the function needs to accept a scalar and be rewritten.

## What this prototype is for

It's a paper schema. The deliverable is:

1. The schema-evolution comment block at `:49-70` (lists which fields evolved from `OneEnemyExample` to `cleanerExampleEnemy`).
2. The `cleanerExampleEnemy` example at `:78-91`.
3. The condition / niche-spawn function stubs at `:164-194` as placeholders for the dispatcher to fill in.

A future engineer can:

1. Copy `cleanerExampleEnemy` for each wave entry.
2. Tune fields in a spreadsheet, paste back.
3. Implement the dispatcher: a Node that walks an array of these dicts every tick, evaluates each `StartCondition` (via the relevant `WhenX_Health` function once they're real), and calls `UnitSpawnSys.queue_unit(...)` for any wave whose condition just became true (with edge-detection to avoid re-triggering).

The dispatcher is the missing piece. Without it, the schema is documentation.

## Where it lives

`from-data-to-stage/` only. The schema design has not been ported anywhere else; the active wave system is still `EnemySpawnStartwatch` + `config_1.tscn` Timers (see [Spawn and Wave System](03-Spawn-and-Wave-System.md)).

## Carry-forward verdict

**Schema design — Tier 2 (reference for future runtime).** Keep as the design source-of-truth for stage data when the dispatcher is built. Before authoring real waves against it, reconcile the two schemas (drop `OneEnemyExample`) and decide whether `HeadsUpBeforeSpawn` is scalar or array (and fix whichever side is wrong).

**`get_or_add` pattern — Tier 1.** Built-in Dictionary method, drop-in safe-default merge. Use it.

**`StageDataClass.gd` runtime — Tier 3 (do not lift).** Pure stub. The five-function `_test_this_stuff` / `dictionary_assign_test` / `get_or_add_test_1/2/3galoo` family is exploratory test code. The condition predicates are placeholders. Implement fresh against the schema.
