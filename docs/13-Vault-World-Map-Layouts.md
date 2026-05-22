# Team Vault — World Map Layouts

The vault's `World Map Layouts/` subdirectory contains one file: `Tutorial Beach.canvas`. It's the spatial design for the game's first-chapter overworld map — node-and-edges sketch of the stage progression on Tutorial Beach.

No `.md` companion exists. The canvas is the spec.

## The map

The canvas reads west-to-east as a tree of stage spots connected by paths. Starting point is the **Entrance Point** (west), terminal point is the **Stage 5 Boss** (northeast). One central hub: the **Command Room**.

```
                                              ┌─ Stage 4 ─── Stage 5 (Boss)
                                              │       \\        /
Entrance ─┬─ Stage 1B ─┐                      │        \\      /
          │            └─ Command Room ─ Stage 3        \\    /
          └─ Stage 1A ─┘                      │          \\  /
                  │                           └─ Stage B ─ Stage C ─ Stage D
                  └─ Stage A
```

## Stage list

| Stage | Reward |
|---|---|
| Entrance Point | Start the game with: **Basic Slime, Mightlight Slime, Tantrum Slime** |
| Stage 1A | Sapphires, EXP |
| Stage 1B | EXP |
| Command Room | Introduction to using the Command Room |
| Stage 3 | **Rocket Pilot Slime**, some EXP |
| Stage 4 | A large amount of Sapphires |
| Stage 5 (Boss) | **Transfer EXP and Sapphires gained from the Tutorial to the main Save File** |
| Stage A | **Hot Token Slime** |
| Stage B | A big chunk of EXP |
| Stage C | **Tremendo Slime**, Sapphires |
| Stage D | A lot of EXP |

The Boss stage's reward — transferring tutorial-earned EXP and Sapphires to the main save — is the design that lets the tutorial be safely skippable for veterans without losing the rewards. The boss is colored differently on the canvas (color 2 = critical / special).

## Branch markers

The canvas has six small circular nodes labeled `# A` or `# B`. They appear to mark fork points — one A and one B per fork — but the canvas doesn't legend them. Best read: visual aids for the designer to talk about specific branches ("the A side after Stage 1A" → Stage A).

## Notes block — meta-currency rules

At the far west of the canvas, a `# Notes` block summarises the meta-currency system that ties stages to progression. Verbatim:

> \# Notes:
>
> - Sapphires are the currency used to buy new Units
>     - If you buy a Unit you already have from the World Map, you get a Training Clock.
> - If a Unit has a **Trainable** Strive, a **Training Clock** can be used to make them always start combat with that Strive active. (Up to 1 per Unit)

This links to the [Macro Progression Matters](12-Vault-Production-Notes.md#macro-progression-matters) spec — Training Clocks grant Training Points, which can be spent to make Trainable Strives free during stages. The Notes block adds one mechanic the Macro Progression doc didn't: a Training Clock can also be spent to **always start a Trainable Strive active** (a permanent +1 active-Strive slot for one Unit).

## Two paths through the tutorial

The map structure encourages two playthrough styles:

1. **North path:** Entrance → 1B → Command Room → 3 → 4 → 5 (Boss). Direct, all the required stages.
2. **South path:** Entrance → 1A → Stage A (Hot Token Slime) → ... → Stage 3 → Stage B → C (Tremendo Slime) → D → 5. Longer, picks up the secret Slimes (Hot Token + Tremendo).

The North path teaches the basics. The South path is the completion run — only by going south do you unlock the side-rewards before fighting the boss. The Boss → Save File transfer means a player who beats the boss without exploring the south path leaves Hot Token and Tremendo locked permanently for that save (since the tutorial is a one-shot).

## Canvas nodes and edges

Full table of text nodes (id, x/y, text content) and the structural edges between them in `_research/vault-extraction.md` § 7.

The canvas has 11 stage nodes (Entrance Point + Stages 1A, 1B, 3, 4, 5, A, B, C, D + Command Room) plus 6 branch markers plus 3 empty positional connectors plus 1 Notes block — 21 nodes total, with 17 unlabeled structural edges between them.

## Where it lives

`OBSIDIAN-TEAM-VAULT/World Map Layouts/Tutorial Beach.canvas`. The only map layout in the vault.

No equivalent layout exists for any other chapter — Tutorial Beach is the only canvas-designed map so far. Future maps would presumably follow the same pattern: tree of stage spots, branch markers, Notes block for chapter-specific meta-currency rules.

## Carry-forward verdict

**The Tutorial Beach canvas is Tier 1 — the spec for the first chapter's overworld content.** The Slime unlock sequence (Basic + Mightlight + Tantrum at start; Rocket Pilot at Stage 3; Hot Token at Stage A; Tremendo at Stage C; Boss → transfer-to-main-save) is the content plan.

To implement, the team needs:

1. Overworld Map runtime ([Overworld Map](06-Overworld-Map.md) — current state is mechanic spike with cursor pathfinding incomplete).
2. Stage-to-scene routing — `IOTile.update_level_info` needs to actually load a stage.
3. Reward dispatch — the Sapphires/Training Clock/Slime-unlock pipeline lives nowhere in code today.
4. Save File coupling — transfer-to-main-save needs the save system from [Stage Loop § qgm save/load](04-Stage-Loop.md#saveload), expanded.

The canvas is the destination; the code is far from there yet.
