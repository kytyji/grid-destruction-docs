# Troubleshooting

Most problems in a destruction setup are silent: they look like "nothing
happened". The plugin logs a reason for almost every refusal, so start with the
Output Log and the console commands below.

## Console commands

These need a running PIE session or game: the geometry and the grid exist only
after BeginPlay.

| command | what it does |
|---|---|
| `GridDestruction.HashCells` | hashes every wall on every live world and prints a MATCH / MISMATCH verdict, naming the first differing cell |
| `GridDestruction.CheckPresets` | one line per wall: `MATCH` or `STALE` against its preset, plus walls over the late-join size |
| `GridDestruction.ShowSeams` | draws which walls are joined (green edges, yellow links) and logs each `GRIDSEAM` |
| `GridDestruction.RebuildWalls` | rebuilds every wall's visible mesh now |
| `GridDestruction.DumpGeometry` | audits winding, normals, counts and bounds on every wall |
| `GridDestruction.ProfileWall` | per-wall mesh statistics, including `degen=` (should be 0) and `dropped=` |
| `GridDestruction.ResetWalls` | every wall back to the damage it started with; on the server or standalone. The sample pawn's **R** key asks the server for you |
| `GridDestruction.CaptureDamage` | writes every wall's damage into its Initial Damage asset ([Setup](setup.md#pre-damaged-walls-and-saved-damage-beta)) |
| `GridNet.Measure` | (development builds, server) per wall: damaged cells per network quarter, bytes now, worst case |
| `GridNet.ToggleDebug` | (development builds) shows or hides this machine's net panel |
| `GridNet.CorruptCell <Wall> <Cell>` | (development builds) changes one damaged cell on this machine only, to watch desync detection heal it |

## What you see → what to check

| what you see | look at |
|---|---|
| A setting does nothing | Press **Apply Preset** on the wall. The preset overwrites the component, and a value set only on the component is off. `CheckPresets` lists stale walls. |
| A setting is on the preset and still does nothing | A Blueprint's Class Defaults override the C++ defaults. Check the placed actor's Details, not the source. |
| I call the damage function and nothing happens | It ran on a client. Every entry point is server-only and silent elsewhere. Check **Has Authority**. |
| Client and server look different | `GridDestruction.HashCells`. MISMATCH names the first differing cell. |
| A hole reads as a hard square | `FractureNoise` is 0, usually because the wall has no preset applied. |
| Breaks ignore the texture's pattern | The bake found no height map. The `FractureMask` log line names what it tried and what to set. |
| `pixel format BC7` and the mask is refused | Re-import that texture as BC1 (Default). |
| `has no texture parameter 'Height'` | The same line lists every parameter the material has. Put one in `FractureHeightParameter`. |
| `none of them carry any bytes` | The texture streams and is not resident. Set `NeverStream` on it, or `MipGenSettings = NoMipmaps`. |
| `the texture is flat ... discarded` | The material's height slot holds an engine placeholder, usually `T_DefaultDisplacement`. The wall falls back to the hash meanwhile, which is correct. |
| The pattern lands in the wrong places | `FractureMaskChannel`: the height may be in green or blue. |
| Cracks follow brick bodies instead of joints | `bInvertFractureMask`. |
| The pattern does not line up with the visible joints | `FractureMaskTiling` must match what the material does with the same texture; leave it at 1 unless the material tiles again inside itself. |
| The pattern is too faint | Raise `FractureMaskContrast` (ships at 2.0). |
| Thin slivers hang off a hole's rim | See [below](#slivers-on-a-holes-rim). |
| A piece of wall hangs in the air | It is held by an anchor: check the wall's **Anchor Edges** ([Setup](setup.md#anchors-what-holds-a-wall-up)). |
| A falling chunk freezes in mid-air | A `HANGRISK` line names the value; see [Surface presets → rubble](surface-presets.md#table-3-rubble). |
| A chunk never moves at all | `GridDestruction.ProfileWall`: a non-zero `degen=` is a bug; send the log line. |
| A lintel falls on the first hit anywhere | `MaxOverhangCm` is below half the opening's width. |
| A shot at the edge of a window dents the face | Raise `CornerChipScale` and `RevealCrushScale` on the preset. |
| The wall's outer edge is open when seen edge-on | Tick the visible sides in `CapEdges`. |
| Joints between walls flicker | The wall material is two-sided; make it one-sided. |
| A line of sky along a joint | The walls are further apart than `SeamToleranceCm` (2 cm). `ShowSeams` shows what was linked. |
| A slab's edge pieces hang in the air | Clear the slab's Anchor Edges; a `SLABANCHOR` line says so at the first hit. |
| Holes lag under very heavy fire | The byte budget; see [Multiplayer](multiplayer.md#bandwidth-and-budgets). |
| A late joiner gets no damage on one wall | The wall is over the late-join size; see [Multiplayer](multiplayer.md#late-join-size). |
| Debris pieces spin on the floor forever | A custom Niagara system without the freeze-on-contact step; see [Effects](effects.md#chunks-gpu). `show Particles` hides all particles, to tell particles from falling wall pieces. |
| Effect count keeps growing | A system that never finishes; see [Effects → performance](effects.md#performance-rules). |
| Frame time is bad on walls nobody shot | The debug lattice is on (`bShowDebugCellsInGame` on GridVisual). It is off by default. |
| Bots do not path through holes | Set the navmesh's Runtime Generation to **Dynamic** ([Setup](setup.md#ai-navigation)). |
| The editor froze when reaching a wall's Details during play | Do not eject with F8; use the console commands. |

## Slivers on a hole's rim

**Why they appear.** Every cell resists a little differently, by design: that is
what stops small holes reading as squares. At the rim of a blast the bite is close
to what a cell has left, so neighbouring cells flip independently and a few tough
ones are left standing in the opening.

**What removes them:** `MinSupportingNeighbours` and `SpikeSuppressionChance` on
the preset, shipping at 2 and 0.7. Two standing neighbours remove single spikes
and pieces hanging by one edge, and keep anything with a shoulder on two sides.
The chance is 0.7 rather than 1.0 on purpose: a rim with every spike pruned reads
as machined.

- By default the chance follows the break pattern, so brick cleans up along its
  joints and keeps its teeth on the bricks. `SpikeMaskStrength` turns that down.
- **Do not raise `MinSupportingNeighbours` above 2** to chase floating pieces:
  at 3 or 4 it strips the rim of every hole. Pieces nothing holds fall on their
  own; `IslandCollapse` in the log counts them.
- Lowering `FractureNoise` and `FractureMaskContrast` makes rim cells behave more
  alike, at the cost of some raggedness.

The pass costs nothing on the network: it looks only at cells the hit already
touched, and the decision is a hash of the cell's position. The log says how many
pieces it pruned on every hit that removes something:

```
SpikeSuppression [Wall_0]: pruned 3 piece(s) ... minNeighbours=2 chance=0.70 mask=fracture weights strength=1.00
```

The wall's own border and the cells beside an authored opening are never pruned.

## Reporting a problem

Write to **annybelogurova@gmail.com** with:

- whether you saw it in the server window, the client window or both;
- the output of `GridDestruction.HashCells` typed once while it is happening;
- the log lines the plugin printed around it (they start with the wall's name in
  square brackets);
- the engine version and whether the project is Blueprint-only.
