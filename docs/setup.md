# Setup

The full path from an installed plugin to walls that look and break right. If
you have not yet seen a wall break, start with the [Quick start](quick-start.md).

## Presets

A wall takes almost everything from a **`GridWallPreset`** data asset. Seven
ship in `/GridDestruction/Presets/`: `GP_Concrete`, `GP_Concrete_Thin`,
`GP_Brick`, `GP_Stone`, `GP_Wood`, `GP_Metal` and `GP_Glass`.

To make your own: **Add → Miscellaneous → Data Asset → `GridWallPreset`**, or
duplicate a shipped one. Four fields matter to get something on screen:

| field | what it does |
|---|---|
| `WallMesh` | the source mesh. Its bounds decide the wall's size, unless the wall overrides it. |
| `SurfaceType` | which material column of [Surface presets](surface-presets.md) this wall is |
| `ThicknessCm` | how deep the wall is. Everything about penetration scales off this. |
| `CellDensity` | cells per metre. 4.0 means 25 cm cells. |

Everything else has a working default. The numbers per material, and what each
one changes, are in [Surface presets](surface-presets.md).

!!! warning "Apply Preset after every edit"
    A preset is stamped onto the wall. Press **Apply Preset** on the wall after
    changing the preset, and do not type values on the wall's component that the
    preset also sets: the next press overwrites them. A feature set only on the
    component is simply off. `GridDestruction.CheckPresets` prints `STALE` for
    every wall that is out of date.

## Placing walls

1. Drag `GridDestructibleWall` into the level.
2. Assign the preset to its **Preset** field.
3. Click **Apply Preset**.

### Sizing a wall

By default the preset's `WallMesh` decides width and height. To size one wall on
its own, tick **Wall | Size → Override Wall Size** and type **Wall Size Cm**
(width, height). The wall is always a whole number of cells, rounded down, and
**Resolved Wall Size Cm** shows what was built. Editing these re-applies the
preset at once.

Keep the actor's scale at 1: scale stretches the cells.

**Cell density is a target.** The cell actually used is the closest size that
divides the wall's width and height exactly (shown as **Resolved Cell Size
Cm**), so changing the density never moves or resizes the wall. Round wall sizes,
in multiples of 10 cm, give it plenty to choose from.

### Pivot and quick building

`Wall | Placement → Pivot`: **Bottom Left** puts the pivot on the wall's
bottom-left corner as seen from the front, so **End** drops it on the floor and
grid snapping lines walls up edge to edge.

`Wall | Build` adds a flush copy to the right, to the left, above, or as an
outside corner. Press again to keep going from the wall just added.
`Override Cap Edges` closes this wall's own ends: a free-standing end or an
outside corner.

### Viewport handles

Select a wall in the level and handles appear in front of it:

- **Green arrows** add a wall to the right, to the left or above.
- **Cyan arrows** add an outside corner. The new wall is selected, so you can
  keep clicking.
- **Yellow points** on the right and top edges resize: click one, then drag the
  move widget (W). The size changes in whole cells and the opposite edge stays
  put.

Every click and drag is one Ctrl+Z. The handles exist only in the editor.

### Wall pieces and kits

Make your own pieces once, then build from them.

The plugin ships samples to start from in **GridDestruction Content →
Variants**: concrete and brick walls 400 and 200 cm wide, wood 300 cm, stone
400 cm and glass 200 cm, all 300 cm high, some with a centred door
(100 × 200 cm) or window (100 × 100 cm). Four kits group them:
`WK_Concrete`, `WK_Brick`, `WK_Wood` and `WK_Mixed` (every sample). Set one as
a wall's **Build Kit**, or right-click a build arrow with no kit set to pick
from every piece in the project. Duplicate a sample into your own folder
before you change it: plugin content is replaced when you update the plugin.

1. Content Browser → right-click → **Miscellaneous → Data Asset → Grid Wall
   Variant**. Set a preset, `Wall Size Cm`, and any doors and windows under
   `Openings`. Name it for what it is, for example `WV_Brick_400_Window`.
2. Optional: **Data Asset → Grid Wall Kit**, and list the variants you want
   together, in menu order: a kit per style or per level.
3. **Place:** drag a variant from the Content Browser into the level. It stands
   on the floor where it is dropped.
4. **Build on:** select a wall. A left click on a green or cyan arrow adds that
   wall's **Build Variant**, or a copy when none is set; the line above the wall
   says which. A **right click** on an arrow opens a menu: a copy, or any piece
   from the wall's **Build Kit** (from the whole project when it has none). The
   piece you pick becomes the new wall's Build Variant, so left clicks keep
   adding it.

Placing from a variant stamps its values once, like a preset: editing the
variant later does not change walls already placed. **Source Variant** on a
wall shows what it was placed from.

### Openings: doors and windows

Select the **GridData** component → `Grid | Openings` → **Openings** → **+**.
`Offset Cm` is measured from the wall's bottom-left corner (X along, Y up), and
`Size Cm` is the width and height. The cells are worked out for you and
re-worked whenever the density or the wall size changes. **Resolved Size Cm**
shows what was cut, in whole cells, so use multiples of the cell size for an
exact fit.

A shot on the reveal of an opening cuts the reveal, not the face.

### Placing the texture

Each wall has **Wall | UV**:

| field | what it does |
|---|---|
| `UV Tile Cm` | centimetres per texture repeat |
| `UV Tile Height Cm` | 0 = square; lets a square texture fit a wall that is not |
| `UV Offset Cm` | slides the texture along or up the wall, to line courses up with a neighbour or the floor |
| `UV Rotation Deg` | rotates it; 90° turns wood grain upright, and the break pattern turns with it |

Four buttons do the common cases: **UV Stretch To Wall** (one texture, corner to
corner), **UV Fill Wall Keep Aspect** (one square texture covering the wall,
centred), **UV Snap To Whole Tiles** (whole repeats, no half brick at the ends)
and **UV Reset**. The preview follows live while you drag.

### What the Details panel shows

The wall shows the preset, the size and the density overrides. The preset shows
mesh, material, surface, thickness, density and caps; its tuning is behind the
**Advanced** arrow. On the GridData component, `Grid | From Preset` is
read-only: those values come from the preset, so change them there and press
Apply Preset.

## Anchors: what holds a wall up

A placed wall is anchored at its **bottom edge only**: the floor holds it. The
walls it touches are found on their own and hold the edges they meet, and an
anchor on such an edge is ignored.

Set **GridData → Grid | Anchors → Anchor Edges** by hand only where an edge is
built into geometry the plugin does not see:

| the wall is | Anchor Edges |
|---|---|
| standing on the floor, sides free or joined to other walls | Bottom (the default) |
| set in a doorway of a static-mesh building | Bottom, Left, Right |
| a span between two static columns | Left, Right |
| a ceiling or floor slab resting on walls | nothing to do: the walls under it hold it |
| a slab with nothing under it (a balcony) | the edges that are built into something |

Anchors live on the placed wall, not on the preset, so Apply Preset never
changes them. **A piece that hangs in the air is almost always held by an anchor
on an edge nothing really holds.**

## Joined walls

Walls find their destructible neighbours on their own at BeginPlay. A link is
made only at a real joint: two walls end to end, one stacked on another, an L
corner, a T-junction, or a wall standing on a destructible floor. Walls that
merely stand near each other, cross, or lie face to face are never linked.

- **A joint holds what touches it, if the other side is held up.** Two walls
  that are both cut off do not hold each other up; both fall.
- **A wall stacked on another stands on that wall**, so it falls when the wall
  below is gone.
- **A shot at the joint damages both walls.** A blast that reaches across it
  bites the neighbour too.
- **A small gap is closed.** Walls up to `SeamToleranceCm` apart (2 cm by
  default) have their facing edges moved half the gap each, so no line of sky
  shows along the joint.
- **The texture lines up** across an end-to-end joint when the wall width is a
  whole multiple of `UV Tile Cm`.
- A **two-sided** wall material flickers at joints. Keep wall materials
  one-sided.

`bAutoLinkSeams` on the preset (on by default) keeps a wall in or out of every
joint. `GridDestruction.ShowSeams` in PIE draws each linked edge in green and
joins the two walls with a yellow line.

## Floors and ceilings

A floor or ceiling is an ordinary wall laid flat, rotated so that its face looks
up or down. Lay it **on top of** the walls that carry it.

- **Clear its anchor edges.** A slab resting on walls ignores its anchors anyway,
  but a slab resting on nothing logs a `SLABANCHOR` line at the first hit if it
  still has them.
- **One slab per room, not a carpet of small panels.** A single slab is one
  structural check, held by the walls under it through its own material. Every
  joint between panels adds a seam, judged more coarsely. Split a ceiling only
  where it really is several pieces.
- **Overhang** past the supporting walls is limited by the preset's
  `MaxOverhangCm`. A ceiling built wider than that still stands as built, and
  loses only what damage leaves overhanging. `BuiltOverhangs` on the preset
  picks a stricter rule; see [Surface presets](surface-presets.md#maxoverhangcm-and-builtoverhangs).

## Built-up walls (layers)

A wall can be several of these walls pressed face to face, each with its own
preset: plaster over brick, tiles over concrete. Place them parallel with the
faces touching, within `SeamToleranceCm`. Each layer breaks by its own material,
and a shot goes through them one at a time.

<video src="../assets/clips/layered-walls.mp4" poster="../assets/clips/layered-walls.jpg" autoplay loop muted playsinline style="width:100%;border-radius:6px"></video>

*Concrete over brick: each layer breaks by its own material.*

The far-face blowout skips a face that is pressed against another layer, so the
layers' holes line up and the wall reads as one. The outer faces keep it. To
switch the blowout off on one placement, clear **Wall | Damage → Exit Spall** on
that wall.

## Where the wall breaks

The break pattern decides which cells give way first. Three sources; the first
one that answers wins:

1. `FractureWeightMask` on the preset: an explicit texture. This wall breaks
   this way, full stop.
2. The height texture of the wall's own material, read automatically. On by
   default.
3. A per-cell hash scaled by `FractureNoise`: rough, but not patterned.

The middle one is the one you will use. Swapping a brick material for a
different bond changes how the wall cracks, with nothing to re-author.

**You do not have to change the texture's import settings.** The bake decodes
BC1 to BC5 directly and walks down to a resident mip if the texture streams.
BC7 is the one format it refuses, by name: re-import that one texture as BC1,
which loses nothing a height map needs.

**You do not have to name the height parameter either.** `Height` is the first
guess. When it misses, the bake searches the material's parameters for one whose
name reads as a height map (`HeightTexture`, `Displacement`, `ORDp`…) and logs
which it took. A material with no parameters at all is searched by the names of
the textures it samples (`T_Brick_H`, `T_Wall_Displacement`…). Anything named
like a normal map is never chosen. Set `FractureHeightParameter` only to pin a
different one.

!!! success "Worked when"
    The log shows one of

    ```
    FractureMask (material height map): 'T_Brick_ORDp' baked, 2048x2048 texels -> 75x75 fine cells, format=DXT1 mip=0 channel=Blue ...
    FractureMask (FractureWeightMask): 'T_Mortar_Mask' baked, ...
    ```

    **Read `range=` in that line.** `range=128..128` means the texture is flat:
    the bake discards it, warns, and the wall keeps the hash. The usual cause is
    a material whose height slot still holds Unreal's placeholder
    `T_DefaultDisplacement`.

Without a height map, an ambient-occlusion map works well for masonry, since the
joints are dark: put an ORM texture in `FractureWeightMask` with
`FractureMaskChannel` = Red. A base colour works in a pinch with `Luminance`,
plus `bInvertFractureMask` when the mortar is lighter than the bricks.

**Is the grid fine enough for the pattern?** When the mask repeats (brick
courses, planks), the bake checks the repeat against the fine cells: two fine
cells per repeat are the minimum, one for the joint and one for the brick. If
the wall has fewer, a `MASKSCALE` line names the `SubdivisionFactor` that fixes
it. Raise `SubdivisionFactor` first: fine cells exist only where the wall was
hit, so it costs nothing on an intact wall.

### Packed textures (Megascans, ORM, ORDp)

**You do not need to unpack anything into a black-and-white texture.** Many
materials, Megascans among them, keep the height in one channel of a packed
texture beside occlusion and roughness. The bake reads a single channel of any
texture, so you only tell it which one:

1. Open the wall's material and find the texture sample that feeds the height
   or displacement (in a Megascans material it is usually named `ORDp`, *Occlusion,
   Roughness, Displacement*).
2. Note which output pin of that sample is used for height: **R**, **G**, **B**
   or **A**.
3. On the preset, set `FractureMaskChannel` to that channel. **Auto reads red**,
   which is right only for a dedicated height texture.

For `ORDp` the height is the **Blue** channel. The bake's log line names the
texture and the channel it read, so one line confirms it:

```
FractureMask (material height map): 'T_Brick_ORDp' baked, ... channel=Blue ...
```

## The wall's material

The visible surface is a procedural mesh, and its material is the preset's
`WallMaterial`, or the first slot of `WallMesh` when that is unset.

The shipped presets already use the damage material instances
`MI_Grid_Damage_*`. For your own surface, make an instance of
`M_Grid_DamageWall` and set its textures; [Damage material](damage-material.md)
walks through it.

The plugin also has a debug lattice, `UGridVisualComponent`: one cube per cell,
hidden by default and allocating nothing until you turn it on. It is a
diagnostic overlay, not the wall's material.

## Rubble and physical materials

Set `PhysicalSurface` on the preset to a physical material. Falling pieces take
their density from it, so a concrete slab does not land like a crisp packet, and
the collider carries it too, so your existing impact decals and surface-type
logic answer correctly. Effects and sounds are covered in
[Effects and sounds](effects.md).

## AI navigation

The wall's collider is part of the navmesh. In the editor it is built with the
wall's preview, so a Nav Mesh Bounds Volume goes around the wall: press **P** in
the viewport to check.

For bots to path **through breaches**, set **Project Settings → Navigation Mesh
→ Runtime → Runtime Generation = Dynamic**. The wall then re-registers its
collider every time a hole opens or closes (never on a dent), and the engine
rebuilds the tiles under it in the background. With the default **Static**
navmesh the walls are still obstacles, but a hole is never walkable.

## Pre-damaged walls and saved damage (beta)

!!! note "Beta"
    Capturing and applying damage works, but the capture workflow is still
    rough. Follow the steps below exactly; expect the workflow to change in a
    later version.

A wall's damage can be saved as a **snapshot** and put back later: to build
ruined houses in the editor, or to carry a room's state between game sessions.

### What a snapshot is

A snapshot is the struct **`FGridDamageSnapshot`**: the wall's damaged cells,
nothing else. No mesh, no collider, no textures are stored; the wall rebuilds
all of those from the cells, exactly as it does after a shot.

A snapshot fits only a wall with the same grid size and `SubdivisionFactor`,
and it carries a format version, so a later plugin version can still read it.
Undamaged cells and authored openings are not stored: openings come from the
wall itself. So a wall with a few holes is a few kilobytes, and an intact wall
is an empty snapshot.

There are two places to keep one:

- **An asset**, `GridDamagePreset`. It holds one snapshot and is saved in your
  project like any data asset. A wall points at it through **Initial Damage**.
  This is for pre-damaged buildings.
- **Your save game.** The struct is Blueprint-ready, so it goes into your own
  `SaveGame` object and is written with **Save Game to Slot** like any other
  variable. This is for carrying damage between sessions.

### How it is applied

The **server** applies a snapshot by writing the stored cells into the wall
through the same path a shot uses. The wall's mesh and collider are rebuilt
from them, and clients receive the cells as ordinary replicated damage. So a
snapshot is never sent as a file: the network only ever sees damaged cells.

- On the server it is **silent**: no debris, no falling pieces, no effects.
- A client already watching the wall sees the change arrive like damage, with
  effects. Apply snapshots at the start of play or on a level load, before
  players look at the wall. Players who join later receive it silently.
- It **replaces** the wall's current damage; it does not add to it.
- Pieces that had collapsed come back as holes; nothing falls again.
- A snapshot that does not fit the wall's grid is refused, with a
  `DAMAGESNAPSHOT ... refused` line naming both grids.

A wall with an **Initial Damage** asset has it applied when the level starts,
and the editor viewport shows that damage while you edit.

### A pre-damaged wall, in the editor

One wall plus a small asset per damage state replaces a separate "damaged" prop.

1. Content Browser → **Add → Miscellaneous → Data Asset → `GridDamagePreset`**.
   Name it, for example `DP_House_WallNorth`.
2. Select the wall in the level → **Wall | Damage → Initial Damage** = that
   asset.
3. Press **Play** and shoot the wall into the shape you want.
4. Open the console (**~**) and type `GridDestruction.CaptureDamage`. Every wall
   with an Initial Damage asset writes its current damage into that asset; the
   Output Log says `DAMAGESNAPSHOT [...]: captured N cell(s) ...`.
5. Stop play and **save the asset** (File → Save All). The capture changed the
   asset in memory only; it is kept once it is saved. The wall now starts
   damaged in every game, and the editor viewport shows the damage.

Capture again at any time: the command replaces the asset's contents. One asset
can go on several walls of the same grid size, for example a row of identical
ruined walls.

### Carrying damage between sessions

In Blueprint, on the server:

1. In your `SaveGame` class, add a variable of type
   `Map<Name, GridDamageSnapshot>`.
2. **Saving:** for each `Grid Destructible Wall`, call **Grid Data → Capture
   Damage Snapshot** and add the result to the map under the wall's actor name.
   Then **Save Game to Slot**.
3. **Loading:** after the level is loaded and before players look at it,
   **Load Game from Slot**, and for each wall found in the map call **Grid Data
   → Apply Damage Snapshot** with its entry. It returns false when the snapshot
   does not fit the wall.

Actor names are stable for walls placed in the level. Walls you spawn at runtime
need a key of your own.

### Resetting

`GridDestruction.ResetWalls`, or **Reset Damage** on a wall, returns walls to the
damage they started with: their Initial Damage, else intact. The sample character's
**R** key does this.
