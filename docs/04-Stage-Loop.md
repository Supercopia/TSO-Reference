# Stage Loop

The Stage Loop is everything you see during a playable match: the player deck UI, the per-frame currency tick, the win/loss declaration, the camera between menu and stage, the bases, the pause menu, and the save/load wrapping.

This chapter primarily documents the **`tso-rushed-stages-showcaes/` prototype** — that's where the playable game loop lives. `tsop---unit-structure/` carries a stripped-down version of the same UI (one slot, no bases, no win/loss) and `Trident_Slime_Ops_Assembly/` carries the deck UI as a dead-loaded scene with no instance in the main scene tree. See [Where it lives](#where-it-lives).

## Autoloads

| Autoload | Script | Role |
|---|---|---|
| `qgm` | `QuickGameManager.gd` | Level state, win/loss, save/load wrapping |
| `PMSV` | `PlayerMidStageVault.gd` | Mid-stage state: slots, currency, `request_ally()` |
| `UnitSpawnSys` | `UnitSpawnSystem.gd` | See [Spawn and Wave System](03-Spawn-and-Wave-System.md) |
| `SaveHandler` | `addons/godot_save_load_manager/SaveLoadData.gd` | Disk persistence |

## QuickGameManager (qgm)

`QuickGameManager.gd` (≈90 lines). Owns level progress and the win/loss path.

### State

```gdscript
@export var Chosen_Level: int = 0              # 0 = menu, 1..8 = stages
var Menu_Camera: bool = true                   # camera toggle hook
var levels_beaten: Dictionary = {
    "level_1": 0, "level_2": 0, "level_3": 0, "level_4": 0,
    "level_5": 0, "level_6": 0, "level_7": 0, "level_8": 0,
}
var allyBaseNode: CombatBody2D                 # declared, unused
var enemyBaseNode: CombatBody2D                # declared, unused
```

`levels_beaten` values are 0 (not beaten) or higher (beaten, with star count).

### declare_winner

```gdscript
func declare_winner(who: int) -> void:         # :28
    if who > 0:
        _record_level_beaten(Chosen_Level)
        save_progress()
    elif who < 0:
        # no penalty
        pass
```

Called by `on_base_death_node.gd` from either base. `who > 0` = player wins, `who < 0` = enemy wins.

`_record_level_beaten(level_number)` (`:35-41`) bumps the relevant `levels_beaten[key]` using `maxi(value, 1)` so a clear can't downgrade a higher star rating.

### load_stage and update_loadout

`load_stage(input_number)` (`:43-47`) sets `Chosen_Level = input_number` and `Menu_Camera = false`. `update_loadout()` (`:48-57`) clamps level flags and conditionally enables a stars system (`if levels_beaten["level_8"] > 0: pass # Enable Stars system`).

`first_playthru_win()` (`:59-61`) bumps `levels_beaten["level_8"]` and saves.

### Save/load

```gdscript
func save_progress() -> void:                  # :69
    var data = { "levels_beaten": levels_beaten }
    SaveHandler.save_data(data)

func load_progress() -> void:                  # :74
    var data = SaveHandler.load_data()
    if data and data.has("levels_beaten"):
        for key in levels_beaten.keys():
            levels_beaten[key] = data["levels_beaten"].get(key, 0)
```

Backed by `addons/godot_save_load_manager/SaveLoadData.gd` — an autoload providing `save_data` and `load_data` to a known `user://` path.

!!! note "qgm save/load is new since the May 15 audit"
    The `PROTOTYPES_REPORT.md` (line 322) notes "Save/load — `levels_beaten` dict exists in qgm but nothing persists it." That claim is now obsolete; `save_progress()` is wired in `declare_winner` (`:32`) and `first_playthru_win` (`:61`). See [verify-prototypes-1-2.md § Save/load](../_research/verify-prototypes-1-2.md#level-stage-systems--present-and-matching).

## PlayerMidStageVault (PMSV)

`PlayerMidStageVault.gd`. The mid-stage scoreboard.

### Slots

```gdscript
var unit_in_slot_1: PackedScene = preload("res://Unit_System/test_2/second_slime_test.tscn")
var unit_in_slot_2: PackedScene = preload("...")
... slots 3–8, all preload "second_slime_test.tscn" ...
```

In proto-2 all eight slots point to `second_slime_test.tscn`. In proto-1 (the rewrite), the slot count is **reduced to one** — `unit_in_slot_1` only, pointing to `first_slime_test.tscn`. Assembly's version follows proto-1's one-slot design but `unit_in_slot_1` is **never assigned** in the script. See [Assembly](11-Assembly.md).

### Currency: CPC and current_player_currency

```gdscript
var current_player_currency: int = 0
var CPC                  # untyped in proto-2, int in proto-1

func _ready() -> void:
    CPC = current_player_currency

func _physics_process(_delta: float) -> void:
    CPC += 1                            # one SP per frame, 60/sec at 60fps
```

CPC is the live counter. `current_player_currency` is read once on ready and **never updated** — the two values diverge from frame 2 onward.

!!! warning "`CPC` vs `current_player_currency` divergence"
    `PlayerMidStageVault.gd:25` does `CPC = current_player_currency` exactly once. Every later read of `current_player_currency` returns 0. Documented in `PROTOTYPES_REPORT.md:215`; confirmed unchanged.

`current_unit_prices: Array = [100, 200, 300, 400]` — four prices for eight slots. In proto-2, `request_ally()` at `PlayerMidStageVault.gd:42-44` has a bounds check that prints `"Unit values error: Out of bounds price tag."` and early-returns when `unit_slot >= 4`, so slots 4–7 silently fail to spawn rather than crashing. Proto-1 only has slot 0, so the out-of-bounds case doesn't arise.

### request_ally

```gdscript
func request_ally(unit_slot: int):
    # bounds check + price check + decrement CPC
    UnitSpawnSys.queue_unit(unit_in_slot_1, 1)   # always queues slot 1
```

!!! warning "Always queues slot 1 regardless of `unit_slot`"
    `PlayerMidStageVault.gd:56` (proto-2) / `:48` (proto-1). The parameter `unit_slot` is used for the bounds-check and price index, but the actual queue call always passes `unit_in_slot_1`. Documented in `PROTOTYPES_REPORT.md:214, 334`.

    In proto-1 this is now *structural* — there is only one slot — so functionally the bug doesn't manifest there. In proto-2 it does: every deck click spawns the slot-1 unit.

## Player Deck UI

Lives under `Level_Functions/refurbished_player_deck/`.

### card_slot_behavior.gd

Attached to each `SLOT1..SLOT8` Button. Strips the "SLOT" prefix to get its number (`name.erase(0, 4)`), classifies side (1/2/5/6 = top, 3/4/7/8 = bottom).

`_process` (proto-2 only): shows/hides slots based on parent's `showButtons` field and level-locking via `qgm.levels_beaten[mystring]`. Teleports unavailable slots to `position.y = 2000`.

`_on_button_down`: calls `get_parent()._on_slot_int_button_down(whichNum, sideOfDeck)` AND `PMSV.request_ally(int(whichNum - 1))`.

### Greenboxing (proto-1 only)

Hold the slot button for 60 ticks: `greenbox_active = true`, spawns a `polite_debug_sprite.svg` child to show the state. The flag toggles, but no auto-spawn loop reads it — UX is half-built.

### p_deck_controls.gd

Attached to the `PlayerDeck` Control root. ≈170-200 lines depending on prototype.

Owns:

- `MenuOpenable: bool`, `StrivesMenuOpen: bool` (proto-1 resets these to `false` in `_ready()` — overriding inspector values)
- `showButtons: int` (0=all, 1=left half, 2=right half)
- `currentlyStriving: int` (selected slot 0-7, -1 = none)
- Cached button refs for 8 slots, 3 strive popups, 3 cooldown buttons (Heavy/Lite/Income).

`arrange_manga_strives(slot_number)` (~65 lines of hardcoded pixel positions: proto-2 at `:96-158`, proto-1 at `:116-187`) hard-codes pixel positions for each of the 8 slots. The comment `## PLEASE RECODE THIS THING LATER` survives at `p_deck_controls.gd:55` in proto-2.

!!! warning "`_on_cooldown_button_press` early-return condition is always true"
    Proto-1 line 70: `if type != "heavy" or "lite":` — evaluates as `(type != "heavy") or "lite"`, where `"lite"` is a truthy string. Always true → the function early-returns on every call. The intended logic is `type != "heavy" and type != "lite"`.

## Bases (proto-2 only)

Two scenes under `Unit_System/go_now/`:

- `ally_base_go.tscn` / `enemy_base_go.tscn` — immobile CharacterBody2D (`Move_Speed = 0`), `DamageReceiver2D`, `AnimationPlayer`, `HealthLabel`.
- `on_base_death_node.gd` — calls `qgm.declare_winner(team_direction)` at 0 HP.
- `waiting_base_for_menu.gd` — waits for `qgm.Chosen_Level != 0`, then instantiates and starts a wave config.

Both bases have entries in `first_slime_test.gd:14-21` / `first_enemy_test.gd:13-18` that hook `_process_death` to declare the winner:

```gdscript
func _process_death() -> void:
    hide()
    position.y = 3000
    if name.contains("Base"):
        qgm.declare_winner(-1)   # ally base died = enemy wins
```

## Camera and menu

Proto-2 has a primitive level-select menu wired to `qgm.Chosen_Level`. Four scripts:

- `camera_2d_fast.gd` — owns `enabled = qgm.Menu_Camera`; toggles between menu camera (panning over level cards) and in-stage camera.
- `camera_2d_in_stage_fast.gd` — inverse: enables when `Menu_Camera == false`.
- `button_fast_menu.gd` — left/right arrow buttons that cycle `Chosen_Level`, scrolling the menu camera by 1152 px per press (`set_position(Vector2(position.x + 1152.0 * direction, 0))`).
- `marker_2d_fast_set_spawn.gd` — sets `UnitSpawnSys.ally_spawn_position` or `enemy_spawn_position` from a Marker2D's global position.

Proto-1 dropped the camera nav entirely. Instead, the autoload `no_camera_solver.gd` creates a fallback `Camera2D` at zoom 1.5 if no Camera2D exists among root children.

## Pause menu and save (proto-2)

`ui/pause_menu.gd` (101 lines) wired to Esc. Toggles `get_tree().paused`. Confirmed at `ui/pause_menu.gd:1-101`.

`SaveHandler` autoload — see [§ qgm save/load](#saveload) above. The save addon `godot_save_load_manager` ships under `addons/`.

These are the **most recent additions** to proto-2 — the chapter author of this reference was told they live in `tso-rushed-stages-showcaes` and confirmed via current source.

## Test units

`Unit_System/test_1/`:

- `first_slime_test.gd` (proto-2) — extends Unit. Hides + moves to y=3000 on death; declares winner if name contains "Base". In proto-1, this is a thin Ally subclass with `Move_Speed *= 3` and a randomized Standing_Range, no death override.
- `first_enemy_test.gd` — mirror.

`Unit_System/test_2/` (proto-2 only):

- `big'un-slime_test.tscn`, `popper-slime_test.tscn`, `scooter-slime_test.tscn`, `tantrum-slime_test.tscn` — four variants.
- `second_slime_test_template.gd` — Sprite swap between four `Sprite2D` children. Minor bug: hides D, D2, D3 on ready (no hide of D4); MAOT() hides D4. `PROTOTYPES_REPORT.md:282-284` flags this.
- `second_first_enemy_spiiiiiiiiiin.gd` — 5-line oddity that spins faster when `get_parent().in_attacking_range`.

## End-to-end frame

For a single stage frame:

```
PMSV._physics_process:     CPC += 1
UnitSpawnSystem._physics_process:
    for n in 1..4:  _spawn_next_unit(1)  # ally
    _spawn_next_unit(-1)                 # enemy

(player presses SLOT2 button)
    card_slot_behavior._on_button_down:
        PMSV.request_ally(1)             # always queues slot 1 unit
            (price check, CPC -=, UnitSpawnSys.queue_unit)

(wave timer fires - proto-2 only)
    EnemySpawnStartwatch._on_timeout:
        UnitSpawnSys.queue_unit(Enemy_To_Spawn, -1)

(enemy hits ally base)
    DamageReceiver2D.take_damage flow → CombatBody2D.current_health -= …
    if current_health <= Minimum_Health:
        Unit._process_death() → adds MapleLeafDisappear child

(ally base hits 0 HP)
    first_slime_test._process_death():
        if "Base" in name: qgm.declare_winner(-1)
            qgm.save_progress() if winner > 0 else no-op
```

## Where it lives

| Component | proto-1 (`tsop---unit-structure`) | proto-2 (`tso-rushed-stages-showcaes`) | Assembly |
|---|---|---|---|
| `PMSV` | 1 slot, never assigned | 8 slots, all hardcoded | 1 slot, never assigned |
| `qgm` | Not present | Present + save/load + pause | Not ported |
| `UnitSpawnSys` | Present (4-ally/frame) | Present (1/frame) | Present (4-ally/frame) |
| `SaveHandler` | Not present | Present (addon) | Not ported |
| Bases (ally/enemy) | Not present | Present | Not ported |
| Camera nav | Replaced by `no_camera_solver` | Present (4 scripts) | Not present |
| Player deck UI | Present (1 slot active, greenboxing) | Present (full 8 slots, level-locking, manga strives) | **Present but never instanced** in main scene |
| Pause menu | Not present | Present (`ui/pause_menu.gd`) | Not ported |
| Wave Timers (`config_1`) | Not present | Present (but `Unready = true` bug) | Not present |

## Carry-forward verdict

**`qgm` save/load wrapping — Tier 1** (proto-2's recent work). Ready to port into Assembly.

**Pause menu — Tier 1** (proto-2's recent work). Drop-in.

**Camera nav (proto-2) — Tier 2.** Functional but tightly coupled to qgm's `Menu_Camera` boolean. Worth lifting once the menu-to-stage transition replaces it.

**`PMSV.request_ally()` slot logic — Tier 3 (must rewrite).** The "always slot 1" bug is structural in proto-1, logical in proto-2. The fix is the dafans-main pattern: dictionary-keyed slot config + a row/slot grid (see [External Study — GAY-DLE](10-External-Study-GAYDLE.md)).

**Income (`CPC += 1`) — Tier 3 (must rewrite).** The vault's Menu Navigation plan calls for a wallet-leveling system ported from dafans-main: levels 1–10, capacity 1,500 → 15,000, scaled tick rate. Iteration 1 Step 4 of the plan. Not yet done in any project.

**`Stat_Magnification` ordering bug on Enemy** — fix is to recompute `set_up_next_KB` after magnification. One-line patch.
