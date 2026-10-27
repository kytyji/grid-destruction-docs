# Changelog

## 1.0.0

First release, for Unreal Engine 5.7 on Win64.

**Destruction**

- Grid destruction with cells that erode from both faces, adaptive subdivision,
  a procedural surface and a matching procedural collider.
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

- Server-authoritative. Only damaged cells replicate, sharded and budgeted to
  stay under the engine's limits.
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
