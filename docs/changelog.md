# Changelog

## 1.2.0

**Collapses**

- A building that loses its support comes down from the break outward over a few frames instead of in one long
  frame. The hole, its rubble and the wall's collision appear together, so what is seen is what blocks.
- Rubble no longer starts inside the collider of the wall it fell from, and a wall that falls whole drops its
  collision at once.
- Rubble, wall meshes and rebar are built on worker threads. On a test level with multi-storey buildings, frames
  over 100 ms during collapses fell from 114 to 42.
- A background check, after a structure settles, that no piece is left standing in the air. It only reports: a
  `HANGCHECK` warning in the log names the wall.
- New console variables: `grid.Rubble.FrameBudgetMs` (8, milliseconds a frame for showing what fell),
  `grid.Rubble.ParallelWalls` (4), `grid.Mesh.ParallelBuild` (1), `grid.HangCheck` (1).

**Fixes**

- Falling rubble keeps the damage paint of the wall it came from.
- Rebar falls with concrete that collapses after being hit; bars no longer stay in the air.
- Rubble no longer makes the navmesh rebuild while it flies.

!!! tip "Dynamic navmesh"
    With **Runtime Generation = Dynamic**, also tick **Do Fully Async Nav Data Gathering** on your RecastNavMesh
    (see [Setup → AI navigation](setup.md#ai-navigation)). Without it the engine gathers the colliders of collapsing
    walls on the game thread.

!!! note "After updating from 1.1.0"
    Rebar now appears a frame or two after the concrete around it changes.

## 1.1.0

**Rebar**

- Concrete can carry steel. Bars show in holes, in craters deeper than the cover and in chipped sides, and stay
  only while steel joins them to standing concrete.
- A grate of bars stops players and lets bullets through, on every machine including a dedicated server.
- Damage marked **Cuts Rebar** removes it and leaves short ends at the cut; a collapse takes the steel of what fell.
  In the sample the cutter and the breaching charge cut.
- A sample ribbed bar, laid in pieces so its ribs never stretch.
- One bit per cell in a byte that already replicates: no extra bandwidth.
- **Line Trace For Rebar Cut**, for weapons that have to hit the grate.

**Damage material**

- `InteriorColorMap`, `InteriorAmount` and `InteriorDepth`: the inside of a wall can have its own, darker texture.
  Off by default; on in the concrete demo instance.

!!! warning "After updating from 1.0.0"
    Walls using `GP_Concrete` now have rebar, and a hole through them blocks players until it is cut. To keep the old
    behaviour, untick `bRebar` on the preset.

## 1.0.0

First release, for Unreal Engine 5.7 and 5.8 on Win64.

**Destruction**

- Grid destruction with adaptive detail, a procedural surface and a matching
  procedural collider.
- Side erosion of caps, ends, window jambs and the bore of a hole; corners chip
  before faces dent; dents drawn in whole chips.
- Exit blowout on the far face; glancing hits leave furrows; grazing and
  ricochet response per material.
- Brittle glass that breaks in whole shards.
- Break patterns from a mask or from the wall material's own height map.

**Structure**

- Structural collapse with falling rubble weighted by the physical material.
- Joined walls: shared seams, hits that cross a joint, support only through real
  ground. Floors and ceilings that rest on walls; an overhang limit, with a
  choice of how overhangs built into a level behave.
- Layered walls whose inner faces take no blowout.

**Multiplayer**

- Server-authoritative. Only damaged cells replicate, budgeted to stay under
  the engine's limits.
- Published per-wall hash, client desync detection and automatic resend;
  `GridNetClientComponent` for an immediate resend and an on-screen net panel.
- `GridDestruction.HashCells` with its own MATCH / MISMATCH verdict;
  `GridNet.Measure` for bandwidth.
- Late-join size warning.

**Look**

- Six material presets (concrete, brick, stone, wood, metal, glass) and a thin
  concrete variant, all driven by one CSV table.
- Damage vertex channels and a wall-normalised UV set on every wall mesh.
- `M_Grid_DamageWall` damage material and two material functions for your own
  materials.
- Niagara effects and sounds per material, fired by the wall, with no Blueprint
  wiring.

**Tools**

- Walls sized in centimetres, click building, viewport handles, wall variants
  and kits, live openings, texture placement controls.
- Walls in the navmesh, with breaches walkable under a dynamic navmesh.
- Damage snapshots for pre-damaged levels and save games (beta).
- Sample wall pieces and four kits in `Variants/`, ready for the build
  arrows' right-click menu.
- A showcase level and a third-person sample character with a rifle, a breaching charge, a
  cutter and a thrown grenade.
- Blueprint integration with no dependencies (Apply Grid Damage) or through a
  GameplayTag weapon table (Notify Hit By Tag).
