# Scene Loading Architecture

`tsop---scene-loading-structure/` prototypes the top-level scene-loading concept: a persistent root scene with three branches (`TONS_Combat`, `TONS_WorldMap`, `TONS_MenuTremendo`) that load and unload their content on demand. One branch is active at a time; switching unloads the old branch and instances the new.

!!! note "This prototype's status changed since the May 15 report"
    `PROTOTYPES_REPORT.md` called this a no-op because the defining bug — `activate_spawn()` instantiated a `PackedScene` but never `add_child`ed it — made the whole system never put anything on screen. **That bug is fixed.** Current source has `add_child(child_scene)` at `TreeOfNodes.gd:20`, all three branches wire up their scenes, mutual exclusion is implemented, and there's a substantial debug UI layer (~350 lines across three scripts) that wasn't mentioned in the report.

    The corrected framing: this is now a **working-but-rough mechanic spike** with a debug UI on top. Documented in detail in [verify-prototypes-3-8.md § A2](../_research/verify-prototypes-3-8.md#a2-tsop---scene-loading-structure--major-rewrite--18-claims-4-6-3-9).

## TONS — Tree of Nodes

The persistent-root concept. The architecture diagram from the source's `editor_description` strings:

```
SLIMEGAME_ROOT  (Node, persistent)
├── TONS_Combat        (activated by debug_key_1; loads pretend_combat.tscn)
├── TONS_WorldMap      (activated by debug_key_3; loads pretend_worldmap.tscn)
├── TONS_MenuTremendo  (activated by debug_key_2; loads pretend_menus.tscn)
├── DebugButtonLine    (Control — on-screen buttons that emulate debug_key_*)
└── MenuStructureVisual (Control — preview thumbnails of menu states)
```

Only one of the three TONS_* nodes holds an instantiated child at a time. The others are dormant. `debug_key_4` closes all three.

The acronym "TONS" expands to "Tree Of Nodes" (per the docstring); also "Tree of Nodes" in the script comments. Each branch's `editor_description` (set in `SLIME_GAME.tscn:16-37`) carries the architectural intent inline in the scene.

## TreeOfNodes.gd

`game-project/ROOT_SCENES/TreeOfNodes.gd`. `class_name TreeOfNodes, extends Node`. Each TONS_* node attaches this script.

### Activation

```gdscript
@export var ThisTreeSpawns: PackedScene
@export var reportsDoing: bool = false   # opt-in logging
var isActive: bool = false
var child_scene: Node

func activate_spawn() -> void:           # :13-22
    if isActive: return
    if not ThisTreeSpawns: return
    if not ThisTreeSpawns.can_instantiate(): return
    child_scene = ThisTreeSpawns.instantiate()
    add_child(child_scene)                # ← the fix
    isActive = true
    if reportsDoing: print(name, " activated")

func deactivate_spawn() -> void:         # :24-30
    if not isActive: return
    isActive = false
    if child_scene:
        child_scene.queue_free()
```

The fix is the `add_child` on line 20. Without it the instance was orphaned and never seen.

### Branch-name dispatch

`_physics_process` (`:32-60`) reads input and matches `self.name` against substrings. Verbatim source (`:46-57`):

```gdscript
if name.contains("Combat"):
    if (Input.is_action_just_pressed("debug_key_1")) && !isActive: activate_spawn()
    elif (Input.is_action_just_pressed("debug_key_2") or Input.is_action_just_pressed("debug_key_4")
    or Input.is_action_just_pressed("debug_key_3")) && isActive: deactivate_spawn()
if name.contains("Menu"):
    if (Input.is_action_just_pressed("debug_key_2")) && !isActive: activate_spawn()
    elif (Input.is_action_just_pressed("debug_key_1") or Input.is_action_just_pressed("debug_key_4")
    or Input.is_action_just_pressed("debug_key_3")) && isActive: deactivate_spawn()
if name.contains("World") and name.contains("Map"):
    if (Input.is_action_just_pressed("debug_key_3")) && !isActive: activate_spawn()
    elif (Input.is_action_just_pressed("debug_key_2") or Input.is_action_just_pressed("debug_key_4")
    or Input.is_action_just_pressed("debug_key_1")) && isActive: deactivate_spawn()
```

Key dispatch:

- `debug_key_1` → activates Combat (and deactivates Menu / WorldMap if they were active)
- `debug_key_2` → activates Menu
- `debug_key_3` → activates WorldMap
- `debug_key_4` → universal close-all (each branch treats it as a deactivate trigger)

Mutual exclusion arises from the **substring checks being mutually exclusive** — no node name contains both "Combat" and "Menu", so only one of the three `if` blocks runs per node. Each branch independently deactivates itself when any of the other branches' activator keys (or `debug_key_4`) fires. No central manager and no `elif` chaining between the three branches.

The `isActive` guards (`&& !isActive` on activate, `&& isActive` on deactivate) avoid redundant `instantiate()` calls and avoid `queue_free`-ing a child that doesn't exist.

### Self-removal guard

Lines 40-43: if `self.name` doesn't match `TONS_*` (none of the substrings), the node `queue_free`s itself. Defensive — prevents stray scripts from causing the dispatch to misbehave.

## debug_key actions

`project.godot:20-68` defines `debug_key_0` through `debug_key_9` action mappings — bound to the numpad and number row. These power both:

- Physical keypresses via `TreeOfNodes._physics_process`.
- On-screen buttons via the `DebugButtonLine` Control, which calls `Input.action_press()` / `action_release()` to simulate the same input.

## Debug UI layer

Three scripts the report didn't mention. They make the prototype usable as an interactive demonstration of the architecture.

### debug_button_line.gd

84 lines. `DebugButtonLine extends Control`. Registers `debug_key_N` actions into `InputMap` at `_ready()`, exposes `button_pressed(N)` that simulates a press + release of the corresponding action.

On-screen buttons in the scene (`button_0..button_9`) each call `button_pressed` with their number. Result: clicking the on-screen "1" button drives `TreeOfNodes._physics_process` through the same code path as physically pressing `1`.

### debug_menu_structure_visual.gd

251 lines. `MenuStructureVisual extends Control`. Auto-discovers `CombatTree` / `WorldMapTree` / `MenuTree` via `get_parent().get_child(N)` (lines 13-21). Per-tree input handlers (lines 40-141).

`condition` state-machine string (lines 80-128) tracks menu sub-states: `viewing_saves`, `selected_empty_save`, `delete_save_submenu`, `are_you_sure_delete_submenu`, **`are_you_really_sure_delete_submenu`**. Each transition is keyed off one of the debug_key actions.

Seven `selbox_config_N` presets (lines 202-251) define `SelectorBox` position/size for each menu state, animated between with tweens.

`report()` (lines 144-153) routes log messages to either `print` or an on-screen emulator-label, depending on a runtime toggle.

### toggle_button_line_info_panel.gd

17 lines. Toggles sibling label visibility on `button_0` press. Trivial but useful for clean-screen mode.

## Pretend scenes

Three placeholder scenes the TONS branches load:

- `pretend_combat.tscn`
- `pretend_menus.tscn`
- `pretend_worldmap.tscn`

Each is a `Node2D` with a labeled `BackgroundContainer`, a same-named `DebugButtonLine` carrying battle/menu/worldmap-suffixed actions, and an inline `labelGainsColor` GDScript subresource (paints a label different colors so you can tell at a glance which branch is loaded).

## HEADS_UP_DISPLAY.tscn

Exists in the project (`game-project/ROOT_SCENES/HEADS_UP_DISPLAY.tscn`) but **referenced by nothing**. Structure: `SLIMEGAME_HUDROOT` (Control) → `UserInterface` + `DebugConsole` (`ColorRect`-backed). Future stage HUD scaffolding.

## SLIME_GAME.tscn

The root scene. Lines 16-37 carry the `editor_description` strings that document the architecture inline. Lines 42-273 host the debug-UI Control nodes (`DebugButtonLine`, `MenuStructureVisual` with 5 preview sprites — MainMenu, CommandRoom, Map&Prep, OpenBase, Combat — plus a `SelectorBox` Node2D).

The 5 preview sprites are static screenshots; clicking through them with the SelectorBox is how the prototype "shows" what each menu mode would look like without actually building those menus.

## Where it lives

`tsop---scene-loading-structure/` only. The TONS concept has not been adopted by any other prototype. Neither `tsop---unit-structure/` nor `Trident_Slime_Ops_Assembly/` use a persistent-root architecture — they switch top-level scenes via `change_scene_to_file()`.

## Carry-forward verdict

**Architectural concept (TONS — persistent root + branch loaders) — Tier 1.** This is a legitimate pattern for a game that needs to keep autoloaded state alive across major mode switches (combat ↔ menus ↔ overworld) without re-creating singletons. Worth lifting into the unified TSO build.

**`TreeOfNodes.gd` — Tier 2.** The substring `name.contains("Combat")` dispatch is brittle (rename a branch and it stops working). A `@export enum BranchKind` would be cleaner. The dispatch should also live on a central scene manager rather than being duplicated on each TONS node.

**Debug UI layer — Tier 2 / discard.** Genuinely useful for demonstrating the architecture; not needed in a real build. Can serve as the scaffold for the final debug toolbar but isn't shippable as-is.

**Pretend scenes — discard.** Placeholders only.

Open work to extract the TONS pattern:

1. Generalise `name.contains(...)` dispatch.
2. Centralise the mutual-exclusion logic on a `SceneManager` autoload.
3. Define what state survives a branch swap (autoloads survive by definition; non-autoload state needs an explicit handoff protocol).
4. Decide how transitions between branches handle the `qgm` / `PMSV` autoloads — saved progress vs. transient combat state.
