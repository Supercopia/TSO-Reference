# External Study — GAY-DLE

`td-like-game-study/dafans-main/` is a complete third-party Godot tower-defense game (a Battle Cats clone by Benjamin Thio). It is bundled in the TSO repo as a **reference implementation** — to study how someone else solved the same problems.

Internal nickname: **GAY-DLE**. It ships as a fully playable standalone project on Godot 4.5, **GL Compatibility** renderer (every other TSO project is Mobile renderer). **35 GDScript files**, a **797-line `DOCUMENTATION.md`**, and a **41 MB `.godot/` cache** that's checked in across 348 files (the repo's `.gitignore` predates Godot 4 — see [§ Hygiene](#hygiene-notes)).

This chapter focuses on **patterns worth porting**, with file:line citations against the GAY-DLE source. All code excerpts below are verbatim from source unless noted.

## The patterns

### 1. pauseable_sleep — highest-value port

`Scripts/time.gd:10-23`, verbatim:

```gdscript
func pauseable_sleep(owner_node: Node, sec: float, timeout_function: Callable = Callable()) -> void:
    var timer: Timer = Timer.new()
    
    owner_node.add_child.call_deferred(timer)
    timer.wait_time = sec
    timer.autostart = true
    timer.timeout.connect(
        func() -> void:
            if not timeout_function.is_null():
                timeout_function.call()
            owner_node.remove_child(timer)
            timer.queue_free()
    )
    await timer.timeout
```

Parents a real `Timer` to `owner_node` (deferred — avoids "can't add child while parent is busy" errors). Sets `autostart`, connects `timeout` to a lambda that optionally invokes the callback then cleans up. The outer `await timer.timeout` is what callers see.

The non-pausable variant (`time.gd:3-8`) uses a `SceneTreeTimer` instead — for cases where you genuinely want unpausable delays (UI animations):

```gdscript
func sleep(sec: float, timeout_function: Callable = Callable()) -> void:
    var timer: SceneTreeTimer = get_tree().create_timer(sec)
    
    if not timeout_function.is_null():
        timer.timeout.connect(timeout_function)
    await timer.timeout
```

Why this is high-value: TSO's combat code uses `await get_tree().create_timer(...)` in many places (`UnitClass`, `HitboxDamageArea2D`, `AttackNode`). Those `SceneTreeTimer`s **do not pause** when `get_tree().paused = true`. The proto-2 pause menu doesn't actually freeze combat properly; units mid-attack will continue their swing through the pause.

Drop `pauseable_sleep` into an autoload (like GAY-DLE's `time` autoload), replace `create_timer` calls with it, and pause becomes correct for free.

### 2. Async await-loops instead of state machines

`Scripts/battlefield.gd:32-39`, verbatim:

```gdscript
func keep_adding_money() -> void:
    var money_increase_pause_gap_duration_at_specific_level: float = get_value_at_specific_cut(min_money_increase_pause_gap_duration, max_money_increase_pause_gap_duration, wallet_level, max_wallet_level)
    
    add_money(money_increase_each_time)
    
    await time.pauseable_sleep(self, money_increase_pause_gap_duration_at_specific_level)
    
    keep_adding_money()
```

Three things in order: compute the current tick duration (depends on `wallet_level`, see [§ 6](#6-wallet-leveling)), add money, await the pause, tail-recurse. Equivalent state-machine in TSO would be a Timer node + signal connection + `_on_timeout` handler.

`enemy_base.gd:24-42` uses the same pattern for a scripted boss intro: `idle_duration` wait → `knockback_shockwave()` → spawn `enemy_god` → 5 s → 2× tank spawn → 10 s → infinite `keep_spawning_enemy` loop.

### 3. UID-based targeting (custom counter, not Godot instance IDs)

!!! note "Correction vs. May 15 report"
    `PROTOTYPES_REPORT.md:498` says GAY-DLE uses Godot's `instance_from_id()` for UID targeting. Actual implementation uses a custom monotonic counter, not Godot instance IDs. Documented in [verify-prototypes-3-8.md § A6](../_research/verify-prototypes-3-8.md#a6-td-like-game-studydafans-main-22-claims-17-4-5).

`battlefield.gd:71-73`, verbatim (sets the spawned node's `.uid` field from the global counter):

```gdscript
func register_uid(node: Node) -> void:
    node.uid = uid_registered
    uid_registered += 1
```

Each base tracks who's currently in contact by UID. `enemy_base.gd:11`:

```gdscript
var opponents_hitbox_uid: Array[int] = []
```

Body-entered / body-exited handlers add / remove UIDs (`enemy_base.gd:89-95`):

```gdscript
func _on_opponent_body_entered(body: Node2D) -> void:
    if body.is_in_group("ally"):
        opponents_hitbox_uid.append(body.uid)

func _on_opponent_body_exited(body: Node2D) -> void:
    if body.is_in_group("ally"):
        opponents_hitbox_uid.erase(body.uid)
```

The damage loop continues recursively **while the opponent is still in contact** (`enemy_base.gd:78-83`):

```gdscript
func keep_dealing_damage_to_self(opponent_damage: int, opponent_attack_gap_duration: float, opponent_uid: int) -> void:
    deal_damage_to_self(opponent_damage)
    
    await time.pauseable_sleep(owner, opponent_attack_gap_duration)
    
    if opponent_uid in opponents_hitbox_uid:
        keep_dealing_damage_to_self(opponent_damage, opponent_attack_gap_duration, opponent_uid)
```

The pattern: opponents are tracked by their integer UID (not by `Node` reference). The damage-recursion bottoms out automatically when the opponent leaves contact (their UID is `erase`d from the list) — no manual disconnect needed, no risk of dereferencing a queue_freed Node. This is "continuous damage while in contact", not "remember who we've already hit".

Caveat: the counter resets per stage (it's a field on `battlefield.gd`, not an autoload). If you need IDs that survive a stage transition, move the counter to an autoload.

### 4. Dictionary-driven slot configuration

`Scripts/character_slots_container.gd:1-66`, verbatim:

```gdscript
extends VBoxContainer

enum CHARACTER {
    PRODUCTIVE_MINION,
    MINION,
    TANK,
    FIGHTER,
    MARKSMAN,
    BLASTER,
    MAGE
}
var pro_minion_packed_scene: PackedScene = preload("res://Instances/productive_minion.tscn")
var minion_packed_scene: PackedScene = preload("res://Instances/minion.tscn")
var tank_packed_scene: PackedScene = preload("res://Instances/tank.tscn")
var fighter_packed_scene: PackedScene = preload("res://Instances/fighter.tscn")
var marksman_packed_scene: PackedScene = preload("res://Instances/marksman.tscn")
var blaster_packed_scene: PackedScene = preload("res://Instances/blaster.tscn")
var mage_packed_scene: PackedScene = preload("res://Instances/mage.tscn")
var character: Dictionary = {
    CHARACTER.PRODUCTIVE_MINION: {
        cost = 300,
        cooldown_duration = 2,
        packed_scene = pro_minion_packed_scene,
        cooldown_animation = null
    },
    CHARACTER.MINION: {
        cost = 300,
        cooldown_duration = 3,
        packed_scene = minion_packed_scene,
        cooldown_animation = null
    },
    # ... TANK (cost 500, cd 5), FIGHTER (1500, 6), MARKSMAN (2000, 7),
    #     BLASTER (3000, 10), MAGE (7000, 60) follow the same shape ...
}
var slots_configuration: Array[Array] = [
    [CHARACTER.PRODUCTIVE_MINION, CHARACTER.MINION, CHARACTER.TANK, CHARACTER.FIGHTER, CHARACTER.MARKSMAN],
    [CHARACTER.BLASTER, CHARACTER.MAGE, null, null, null],
]
```

Per-character entries carry **four** keys: `cost`, `cooldown_duration` (seconds), `packed_scene`, and `cooldown_animation` (`Tween` reference, used to gate re-spawning while a cooldown is in flight). Note Dictionary literals use identifier-style keys (`cost = 300`) rather than string keys (`"cost": 300`) — both are equivalent in GDScript 2.

`spawn(row_index, slot_index)` (`character_slots_container.gd:89-108`), verbatim:

```gdscript
func spawn(row_index: int, slot_index: int) -> void:
    var configured_character: CHARACTER = slots_configuration[row_index][slot_index]
    var global_configured_character: Dictionary = character[configured_character]
    
    if owner.money_owned_in_wallet >= global_configured_character.cost and global_configured_character.cooldown_animation == null:
        var spawned_character: CharacterBody2D = global_configured_character.packed_scene.instantiate()
        var base_spawnpoint: Marker2D = base.get_node("Spawnpoint")
        var slot: Control = get_child(row_index).get_child(slot_index)
        var slot_cooldown_progress_bar: TextureProgressBar = slot.get_child(0)
        
        owner.register_uid(spawned_character)
        owner.get_node("AllyLayer").add_child(spawned_character)
        owner.money_owned_in_wallet -= global_configured_character.cost
        spawned_character.global_position = base_spawnpoint.global_position
        slot_cooldown_progress_bar.value = 0
        global_configured_character.cooldown_animation = create_tween().tween_property(slot_cooldown_progress_bar, "value", slot_cooldown_progress_bar.max_value, global_configured_character.cooldown_duration).finished.connect(
            func() -> void:
                global_configured_character.cooldown_animation = null
        )
```

Gate: `money_owned_in_wallet >= cost AND cooldown_animation == null`. So a slot is unavailable both when you can't afford it and while its cooldown tween is in flight. The tween fills a `TextureProgressBar` over `cooldown_duration` seconds; the `.finished` callback nulls out `cooldown_animation`, re-enabling the gate.

This is two patterns TSO would benefit from:

- **Single grid lookup** for which character a slot represents (`slots_configuration[row][slot]`) + a single Dictionary of per-character config (`character[which]`) — direct fix for TSO's broken `PMSV.request_ally()` which hardcodes 8 separate `unit_in_slot_N` vars and ignores the slot parameter.
- **Per-slot cooldown via tween + progress bar** — direct model for TSO's Lite/Heavy/Income cooldown buttons in `p_deck_controls.gd`, which today only have stub on-click handlers and no actual cooldown gating.

Caveat: `character_slots_container.gd:84-85` computes the slot button label using `CHARACTER.find_key((row_index * 5) + slot_index).capitalize()` — hardcodes a 5-column grid. Silently breaks on resize. The line is just for the display label; the grid math elsewhere uses the `slots_configuration` 2D array correctly.

### 5. velocity + move_and_slide (referenced, not re-verified)

`PROTOTYPES_REPORT.md:502` claims GAY-DLE uses `velocity = speed * direction` + `move_and_slide()` for movement, contrasted against TSO's `MAOT() position.x +=`. This claim was not directly re-verified in [Stream D](../_research/verify-prototypes-3-8.md#a6-td-like-game-studydafans-main-22-claims-17-4-5) — the relevant CharacterBody2D unit scripts weren't read in that pass. Take the claim as plausible but un-reconfirmed.

If true, lift the pattern when fixing TSO's `MAOT()`.

### 6. Wallet leveling

`battlefield.gd:68-69`, verbatim:

```gdscript
func get_value_at_specific_cut(min_value: float, max_value: float, cut: float, max_cuts: int) -> float:
    return min_value + ((max_value - min_value) * (cut - 1) / (max_cuts - 1))
```

A linear lerp between `min_value` and `max_value`, picking the `cut`-th value out of `max_cuts` evenly-spaced points (1-indexed). Used to scale both the **income amount per tick** and the **pause duration between ticks** as the wallet levels up.

10 wallet levels with `1,500 → 15,000` capacity and a scaled income tick rate. Vault's Menu Navigation plan calls this out as Iteration 1 Step 4 — the planned replacement for TSO's broken `CPC += 1` per-frame income.

Capacity reference (from the vault plan; not literal source values):

| Level | Max SP | Income/tick |
|---|---:|---:|
| 1 | 1,500 | 7 |
| 5 | ~8,250 | 15 |
| 10 | 15,000 | 25 |

`level_up_wallet()` costs 50 % of current capacity; gates at max.

### 7. Win/loss flow via a `victory: bool` global

`enemy_base.gd:74-76`, verbatim:

```gdscript
if hitpoints <= 0:
    Global.victory = true
    get_tree().call_deferred("change_scene_to_file", "res://Scenes/game_over.tscn")
```

`game_over.gd:10-16` reads `Global.victory` to render Victory vs Defeat.

A subtler version of TSO's `qgm.declare_winner(who: int)` — uses a bool flag in a Global autoload instead of an integer team_direction. Either approach works; TSO already does this its own way.

### 8. Async tween → AoE pattern

`cannon_charger.gd:15-44`. Not quoted here — the function is 30 lines of tween setup. Behavioural summary:

1. Tween fills a charge bar over N seconds.
2. On `tween.finished`, enable `fireable = true` + start `blink()` loop.
3. `blink()` is a separate await loop that flashes the bar until the player taps to fire.
4. Tap → spawn AoE projectile + reset.

Three independent async tasks coordinated by callbacks. Clean for a "charge attack" mechanic.

## Architecture notes

### Two `Global.gd` claims to verify

The May 15 report describes `Global.gd` as "bare `victory` bool + scene swap." Actual content (`Global.gd:1-12`), verbatim:

```gdscript
extends Node

var victory: bool

func _process(_delta):
    if Input.is_action_just_pressed("fullscreen"):
        if DisplayServer.window_get_mode() != DisplayServer.WINDOW_MODE_FULLSCREEN:
            DisplayServer.window_set_mode(DisplayServer.WINDOW_MODE_FULLSCREEN)
        else:
            DisplayServer.window_set_mode(DisplayServer.WINDOW_MODE_WINDOWED)
```

Just the bool + a fullscreen toggle (bound to the `"fullscreen"` input action). **No scene swap** in `Global.gd`. The scene swaps happen in `enemy_base.gd:76` and `game_over.gd:19, 22`. Cited in [verify-prototypes-3-8.md § A6](../_research/verify-prototypes-3-8.md#a6-td-like-game-studydafans-main-22-claims-17-4-5).

### Scripted boss intros

`enemy_base.gd:24-42` is the boss-intro pattern: an await chain that orchestrates a boss-cinematic sequence. `idle_duration` wait → shockwave → spawn boss → 5s → tanks → 10s → infinite trash loop. Then `enemy_base.gd:66-70` triggers a half-HP "ulti" phase: bumps spawn rate to 5, fires another shockwave, spawns an elite enemy.

Pattern is portable — TSO has no boss code today; this is the model for one.

### Hygiene notes

- **`.godot/` cache committed.** 41 MB across 348 files. `dafans-main`'s own `.gitignore` predates Godot 4 — it lists `.import/`, `.mono/`, `data_*/` (Godot 3 conventions) but not `.godot/`. (Every TSO prototype's `.gitignore`, by contrast, correctly lists `.godot/`.) Optional fix: `git rm -r --cached td-like-game-study/dafans-main/.godot` + add `.godot/` and `*.tmp` to `dafans-main/.gitignore`. Or do nothing — the cache is in a vendored sub-project; not a hot path.
- **8 `*.tmp` editor scratch files** across `Instances/` and `Scenes/`. Same fix.

## Where it lives

`td-like-game-study/dafans-main/`. The repo's standalone reference. Not integrated into any TSO prototype.

## Carry-forward verdict

| Pattern | Tier | Notes |
|---|---|---|
| `pauseable_sleep` autoload | 1 | Drop into TSO's autoloads; fixes proto-2 pause-during-combat. |
| Async await-loops | 1 | Replaces Timer-node-plus-signal boilerplate for periodic background work. |
| UID-based targeting (custom counter) | 1 | Replaces `Array[Unit]` references in TSO with `Array[int]` UIDs — fixes stale-reference crashes. |
| Dictionary-driven slot config | 1 | Direct fix for TSO's broken `request_ally` "always slot 1" bug. |
| Per-slot cooldown via tween + progress bar | 1 | Direct model for TSO's Lite/Heavy/Income cooldown buttons (currently stubbed). |
| Wallet leveling | 1 | Iteration 1 Step 4 in the vault's plan. |
| `velocity + move_and_slide` | 2 (unverified) | Confirm against GAY-DLE source, then port to TSO's `MAOT()`. |
| Async tween → AoE | 2 | Useful when implementing charge attacks. |
| Boss intro / ulti pattern | 2 | Model for first TSO boss stage. |
| `base.gd`/`enemy_base.gd` files | 3 | The rushed-stages versions are better; don't lift these directly. |
| `Global.gd` fullscreen toggle | 3 | Project-specific; not portable. |

Don't port: the scripts named above as Tier 3, plus `time.gd`'s `sleep` (non-pausable variant — TSO can write the equivalent in one line), plus `random.gd` (TSO can write equivalents), plus `character_slots_container.gd` directly as a file (TSO's `p_deck_controls.gd` is more polished — but its broken slot lookup AND missing cooldown gating should both learn from this file).
