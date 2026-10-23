# Surface presets

What every preset field does, and the order to tune them in.

**The shipped presets already carry these values.** `GP_Concrete`, `GP_Brick`
and the others in `/GridDestruction/Presets/` are ready to use, so you do not
type anything from this page to get a working wall. The value tables are folded
away: open them when you make a preset of your own, want to compare materials,
or need to get back to the shipped numbers after experimenting.

Three assets carry them, and the split matters:

| asset | answers |
|---|---|
| `GridSurfaceMaterialDataAsset` (shipped: `DA_GridSurfaceMaterials`) | how this **material** fails: shared by every wall made of it |
| `GridWallPreset` (shipped: `GP_*`) | what this **wall** is: thickness, density, how much of it may go |
| `GridCollapseVisualComponent` on the wall | what its **rubble** does after it falls |

Use one project-wide surface material asset and reference it from every preset.
A surface type missing from its `Configs` map falls back to neutral values, so a
partial map is valid.

## Editing the numbers: `GridMaterials.csv`

Every number in tables 1 and 2 lives in one spreadsheet, `Config/GridMaterials.csv`
in the plugin's folder: one column per preset, one row per field, and a Notes
column saying what each does. The plugin's folder is `Plugins/GridDestruction/`
in your project, or `Engine/Plugins/Marketplace/GridDestruction/` for a Fab
install. Edit the CSV in Excel, Google Sheets or a text editor, then in the
editor choose **Tools → Execute Python Script** and pick
`scripts/ue_apply_material_table.py` from the same folder. The **Python Editor
Script Plugin** must be enabled (it is by default).

It creates or updates each column's preset (row `Asset`) and its entry in the
surface material asset (`DA.*` rows), then presses Apply Preset on every wall in
the open level that uses one of them. **Save the level afterwards.** The log
prints each field it changed as `old -> new`.

- An empty cell leaves that value alone.
- Point `Asset` at your own preset to tune it from the table; add a column for a
  second kind of brick. Two columns of one `SurfaceType` must agree on the
  `DA.*` rows, and the script stops if they do not.
- `WallMesh` and `WallMaterial` take asset paths. A preset the script created
  has neither, so fill them before a wall uses it.
- Masks, effects and `CapEdges` stay on the assets.
- **The CSV wins.** A value you changed by hand in the editor is overwritten on
  the next run unless its cell is empty.
- It reaches only walls in the **open level** whose preset is a column. In
  another level using the same presets, run it there too or press Apply Preset;
  a wall that missed it logs `GRIDSTAMP` at BeginPlay.

!!! tip "Work on copies"
    Copy the shipped `GP_*` presets and the surface material asset into your
    project, and point the CSV's `Asset` and `SurfaceMaterialData` rows at the
    copies. A plugin update then never overwrites your tuning.

## Table 1: the surface material asset

`DA_GridSurfaceMaterials` → `Configs` → one row per surface type.

??? info "Shipped values: open the table"

    | | Concrete | Brick | Stone | Wood | Metal | Glass |
    |---|---|---|---|---|---|---|
    | `DamageResistance` | 1.0 | 0.9 | 1.4 | 0.6 | 2.0 | 0.3 |
    | `FalloffScaleU` | 1.0 | 1.6 | 0.9 | 3.0 | 0.8 | 1.2 |
    | `FalloffScaleV` | 1.0 | 0.7 | 0.9 | 1.0 | 0.8 | 1.2 |
    | `SmoothingAngleDeg` | 30 | 25 | 20 | 35 | 60 | 15 |
    | `RicochetAngleDeg` | 15 | 15 | 18 | 10 | 30 | 0 |
    | `GrazingDamageScale` | 0.35 | 0.35 | 0.30 | 0.50 | 0.10 | 1.0 |
    | `EdgeProfile.Style` | Rounded | Square | Rounded | Shard | Rounded | Shard |
    | `EdgeProfile.AmplitudeU` | 0.30 | 0.30 | 0.25 | 0.30 | 0.08 | 0.35 |
    | `EdgeProfile.AmplitudeV` | 0.30 | 0.04 | 0.25 | 0.06 | 0.08 | 0.35 |
    | `EdgeProfile.QuantiseU` | 0 | 0.15 | 0 | 0 | 0 | 0 |
    | `EdgeProfile.QuantiseV` | 0 | 0 | 0 | 0 | 0 | 0 |
    | `EdgeProfile.DepthNoise` | 0.25 | 0.15 | 0.30 | 0.20 | 0.05 | 0.10 |
    | `bBrittle` | false | false | false | false | false | true |
    | `BrittleBreakFraction` | 0.35 | 0.35 | 0.35 | 0.35 | 0.35 | 0.35 |
    | `BrittleShardSizeCm` | 12 | 12 | 12 | 12 | 12 | 12 |

### Why each column looks the way it does

- **Concrete** is the reference: resistance 1.0, chaotic spall on both axes, a
  circular blast. Every other column is a departure from it.
- **Brick** fails along its courses. `QuantiseU 0.15` against `AmplitudeU 0.30`
  gives discrete levels that read as mortar joints rather than noise, and
  `AmplitudeV 0.04` keeps the courses straight. `FalloffScaleU 1.6` against
  `0.7` spreads a hit sideways along a course instead of down through one.
- **Stone** is the hardest masonry: high resistance, the highest `DepthNoise` so
  the crater surface is rough, and a low smoothing angle so chips stay faceted.
- **Wood** splits along the grain: `FalloffScaleU 3.0` against `1.0` makes a
  hit reach three times as far along the grain as across it. U and V are the
  visible texture's axes, so rotating `Wall | UV` by 90° for upright boards turns
  the grain with them.
- **Metal** deforms rather than breaks: the highest resistance, the smallest edge
  amplitude and a 60° smoothing angle, so it reads as bent sheet.
- **Glass** shatters: the lowest resistance, the largest amplitude and a hard 15°
  smoothing angle.

### The fields

`DamageResistance`
:   How much of each bite this material takes. It changes how many shots a wall
    takes and almost nothing about how it looks. **Thickness is the real
    durability knob**; resistance saturates once a blast is deep enough to open
    a cell.

`FalloffScaleU` / `FalloffScaleV`
:   The blast's shape along and across the texture. The single most
    recognisable knob, and it costs nothing.

`SmoothingAngleDeg`
:   Normals are smoothed across folds flatter than this. `-1` means unset (the
    wall mesh component's own value), `0` is flat shading. An angle above
    `atan(halfThickness / cellSize)` never engages: at 10 cm cells and 15 cm
    thickness that limit is 37°.

`RicochetAngleDeg` / `GrazingDamageScale`
:   What a shallow hit takes off. The angle is measured **off the surface**: 90
    is straight on, 0 is along the wall. Below it the bite ramps down to
    `GrazingDamageScale` at 0°. A ricochet angle of 0 means the material never
    grazes. Keep `GrazingDamageScale` above 0 for anything that spalls: a
    skipping round still ploughs a furrow, and a shot that leaves no mark reads
    as a broken wall. The plugin decides how much material came off; whether the
    projectile deflects, sparks or stops is your weapon's call, using
    `Obliquity` and `bGrazing` in the damage result.

`EdgeProfile.Style`
:   How a hole's outline is drawn. `Square` follows the cells' sides, honest for
    brick. `Shard` cuts outward corners on the diagonal, so a staircase becomes a
    straight cut. `Rounded` relaxes the contour so stairs become straight runs
    and corners round off. Render only: the collider stays on the cells.

`EdgeProfile.AmplitudeU/V`, `QuantiseU/V`, `DepthNoise`
:   The rim's jitter in the wall's plane, its banding, and the roughness of the
    crater's floor. Amplitude above **0.40 is refused**: two neighbouring
    vertices would swap places. Keep `Quantise` around half the amplitude; at
    twice the amplitude or more every vertex rounds to zero and the axis has no
    jitter at all. Apply Preset warns about both.

`bBrittle`, `BrittleBreakFraction`, `BrittleShardSizeCm`
:   A brittle cell is whole or gone, so a pane never shows a dish or a dent.
    `BrittleBreakFraction` is the share of a cell's thickness one hit must take to
    break it for certain; smaller hits break it with a proportional chance, so a
    blast's edge cracks unevenly and repeated hits always get through. Lower it
    for thin panes, raise it for laminated glass. The pane is divided into
    irregular shards about `BrittleShardSizeCm` across, and a hit breaks out
    whole shards connected to the impact. The roll is made on the server, so
    every machine sees the same holes.

### Two kinds of brick

`SurfaceType` has six values and no `Custom`. That is deliberate: the enum is a
lookup key for "what is this physically", for effects and sounds. Everything
that differs between a red brick and a grey one lives on the preset.

| You want | Do this |
|---|---|
| Two brick walls that break differently | Two presets, both `SurfaceType = Brick` |
| ...that also resist differently | Point their `SurfaceMaterialData` at two different data assets |
| Glass that shatters vs glass that spiders | Two presets, different `FractureWeightMask` |

## Table 2: the wall preset

??? info "Shipped values: open the table"

    | | Concrete | Brick | Stone | Wood | Metal | Glass |
    |---|---|---|---|---|---|---|
    | `ThicknessCm` | 25 | 20 | 40 | 8 | 4 | 2 |
    | `CellDensity` | 4 | 4 | 3 | 5 | 5 | 6 |
    | `SubdivisionFactor` | 2 | 4 | 2 | 2 | 1 | 1 |
    | `StructuralThreshold` | 0.25 | 0.30 | 0.20 | 0.40 | 0.15 | 0.60 |
    | `MaxOverhangCm` | 200 | 150 | 120 | 150 | 300 | 60 |
    | `MaxDestructionPercent` | 1.0 | 1.0 | 1.0 | 1.0 | 1.0 | 1.0 |
    | `MinSpallRadiusCells` | 1.5 | 1.4 | 1.2 | 1.6 | 0.8 | 1.5 |
    | `ExitSpallDepthScale` | 0.35 | 0.40 | 0.30 | 0.45 | 0.15 | 0.40 |
    | `ExitSpallRadiusScale` | 1.6 | 1.7 | 1.5 | 1.8 | 1.2 | 1.2 |
    | `ObliquityStretch` | 1.0 | 1.2 | 0.8 | 1.2 | 0.6 | 0.4 |
    | `ObliquitySkew` | 1.0 | 1.2 | 0.8 | 1.2 | 0.6 | 0.4 |
    | `BreachFalloffCells` | 3.0 | 2.5 | 3.5 | 2.5 | 1.5 | 4.0 |
    | `FractureNoise` | 0.40 | 0.30 | 0.45 | 0.30 | 0.10 | 0.20 |
    | `FractureNoiseScaleCm` | 20 | 20 | 25 | 10 | 15 | 15 |
    | `FractureMaskContrast` | 2.0 | 2.0 | 2.0 | 2.0 | 2.0 | 2.0 |
    | `bDeriveFractureMaskFromMaterial` | true | true | true | true | false | false |
    | `SpikeSuppressionChance` | 0.7 | 0.7 | 0.7 | 0.7 | 0.7 | 0.7 |
    | `MinSupportingNeighbours` | 2 | 2 | 2 | 2 | 2 | 2 |
    | `HoleRimThicknessFraction` | 0.10 | 0.12 | 0.10 | 0.18 | 0.30 | 0.45 |
    | `EdgeCrumbleScale` | 1.0 | 1.0 | 1.0 | 0.8 | 0.25 | 0.9 |
    | `RevealCrushScale` | 1.5 | 1.8 | 1.3 | 2.0 | 1.0 | 2.5 |
    | `CornerChipScale` | 4 | 4 | 4 | 4 | 0 | 0 |
    | `EdgeFirst` | 1 | 1 | 1 | 1 | 0 | 0 |
    | `DentStepCm` | 3 | 4 | 4 | 2 | 0 | 0 |
    | `CapEdges` | per placement | | | | | |

### Size and structure

`ThicknessCm`, `CellDensity`
:   How deep the wall is, and cells per metre. Bigger cells hold more depth, so a
    wall that opens with no dent first wants a lower density or more thickness.

`SubdivisionFactor`
:   A cell splits into `factor × factor` fine children on its first bite. Detail,
    and its cost on the wire, exist only where the wall was hit. Raise it when a
    `MASKSCALE` line asks for it, or when an edge on show looks too smooth.

`StructuralThreshold`
:   The remaining fraction below which a cell stops carrying load. Wood gives up
    its load path long before it opens; reinforced concrete carries down to a
    shell; glass sheds whole panes at the first serious crack.

`MaxDestructionPercent`
:   The share of the wall that shooting may remove: `0` for a level boundary,
    `0.3` for a load-bearing element. Per wall, not per material, which is why
    the row is uniform. It gates shooting only: a severed piece still falls, and
    a collapse counts against the cap.

### `MaxOverhangCm` and `BuiltOverhangs`

A cell is held only if it is within `MaxOverhangCm` **sideways** of a column that
stands on, or hangs from, a support. Up and down are free on an upright wall, so
an intact wall always stands; a cantilever longer than the limit loses its tip,
and a beam held at both ends spans twice the limit. On a ceiling or floor neither
axis is free.

- Keep it **above half the widest opening with no wall under it**, or the lintel
  over it falls on the first hit anywhere on the wall.
- `0` turns it off. The limit is rounded down to whole cells.

What was built stands: the limit counts from how far a wall reached when damage
first came to it, so it drops only what damage leaves overhanging. A wall built
past the limit logs an `OVERHANG` line once. `BuiltOverhangs` on the preset
(Wall | Durability, advanced) picks a stricter rule:

| value | a span built past the limit |
|---|---|
| `Keep` (default) | stands; the limit judges only what damage leaves |
| `SettleSilently` | is gone when the level starts, with no debris or sound |
| `Collapse` | comes down when the level starts, with debris and falling pieces |

### Shape of a hit

`MinSpallRadiusCells`
:   Every hit is spread over at least this radius. If an edge looks too smooth,
    lower it to 0.8–1.0 on brick and concrete for a more local bite, and raise
    `FractureNoise` to keep small holes from reading as squares. Do not take it
    to 0.

`ExitSpallDepthScale`, `ExitSpallRadiusScale`
:   The blowout of the far face, as a fraction of the wall's thickness and a
    multiple of the hit's radius. It starts once the deepest cell in a hit is
    eaten past `ThroughCouplingThreshold` of the thickness (0.8 by default, on
    the wall's GridData component). Brittle materials throw more off the back;
    metal petals instead. `0` switches it off for the material.

`ObliquityStretch`, `ObliquitySkew`
:   A glancing hit becomes a furrow instead of a dish. The stretch lengthens the
    crater along the shot's direction; the skew slides it forward. **Keep the
    two equal**: that pins the trailing edge where a square-on crater's would be
    and sends all the added length forward. Below it the crater grows backwards,
    above it the crater leaves the impact point.

`FractureNoise`, `FractureNoiseScaleCm`
:   Varies each cell's resistance by a fixed amount, so a small hole does not
    read as a square. Above about 0.6 the rim turns noisy and specks survive
    inside holes. Clamped at 0.9.

`FractureMaskContrast`, `bDeriveFractureMaskFromMaterial`
:   Strength of the break pattern, and whether it comes from the wall material's
    height map. See [Setup → Where the wall breaks](setup.md#where-the-wall-breaks).

`SpikeSuppressionChance`, `MinSupportingNeighbours`
:   Clean slivers off a hole's rim. A cell with fewer standing neighbours than
    the minimum is pruned with this chance. The chance is 0.7 on purpose: a rim
    with every spike pruned reads as machined. **Keep the minimum at 2**: higher
    strips the rim of every hole. By default the chance follows the break
    pattern, so brick cleans up along its joints and keeps its teeth on the
    bricks. `SpikeSuppressionMask` paints it per wall; `SpikeMaskStrength` sets
    how far a mask moves it.

### Edges and rims

`HoleRimThicknessFraction`
:   The band of wall left standing at a hole's edge, as a fraction of the
    thickness: what makes a hole read as a break in a solid rather than a cut in
    paper. Thin materials need more to show anything. It is also depth the
    crater cannot reach, so too much flattens craters; above about 0.5 a hole
    reads as a bevelled port.

`EdgeCrumbleScale`
:   How much an exposed edge (a capped side, a reveal) wanders as it erodes:
    crisp when intact, the material's own jitter when eaten. It needs a real
    `EdgeProfile` amplitude to show.

`RevealCrushScale`
:   A shot **into a side** (a reveal, a capped end, the bore of a hole) cuts that
    side along its own normal by `PeakDepthCm × RevealCrushScale`. `0` makes such
    a shot an ordinary face hit.

`CornerChipScale`
:   A shot at the **face** near an edge also chips the edge. Values near 1 make an
    edge give way about as fast as a shot straight into it; the shipped 4 makes
    corners crumble visibly faster than the face. Lower it toward 1–1.5 for a
    sturdier look. `0` switches it off.

`EdgeFirst`
:   In a cell at an edge, a face hit dents the face by `1 − EdgeFirst` of its
    bite. At 1 the corner crumbles away before the face behind it gives. Does
    nothing while `CornerChipScale` is 0.

`DentStepCm`
:   Dents are drawn in whole steps of this depth, so a hit drops a chip instead of
    crumpling the face. A hit shallower than one step leaves the geometry flat
    and is shown by the damage material. The collider uses the same steps. `0`
    draws the exact depth.

`BreachFalloffCells`
:   Purely visual: how far, in cells, the "near a hole" channel of the damage
    material reaches. Glass at 4.0 is the deliberate outlier: its cracks radiate
    far past what was removed.

### `CapEdges`: closing the wall's own sides

Per placement, not per material. A wall's outer perimeter is open by default,
because most walls are seated in a frame or a facade. Tick the sides a player can
see:

| the wall is | tick |
|---|---|
| seated in a frame, or flush between two others | nothing |
| protruding from another wall at 90° | the exposed end only |
| a free-standing barricade or a pillar of wall | all four |
| a panel whose top the player can see over | Top |

A cap breaks with the wall: it is built from the same cells as a hole's rim. If
a capped edge looks like a knife edge, raise `HoleRimThicknessFraction`.

## Table 3: rubble

On the wall's **GridCollapseVisual** component, not on the preset.

??? info "Shipped values: open the table"

    | | Concrete | Brick | Stone | Wood | Metal | Glass |
    |---|---|---|---|---|---|---|
    | `DebrisMassScale` | 1.0 | 0.9 | 1.3 | 0.35 | 0.8 | 0.5 |
    | `ChunkLifetimeSeconds` | 1.6 | 1.6 | 2.0 | 1.4 | 1.4 | 1.0 |
    | `KeepSettledAboveSizeCm` | 0 | 0 | 0 | 0 | 0 | 0 |

- **Mass comes from the physical material first.** Set `PhysicalSurface` on the
  preset; `DebrisMassScale` is taste on top. Leave it at 1.0 until the physical
  materials are in place.
- **Keep `KeepSettledAboveSizeCm` at 0 in multiplayer** (it is read as 0 in any
  networked game). Rubble never replicates, so a piece left standing would be
  cover in a different place on every client that nothing can remove.

| knob | default | raise it when | lower it when |
|---|---|---|---|
| `SettleLinearVelocityThresholdCmS` | 2.0 | rubble twitches instead of resting | a chunk freezes in the air |
| `SettleHoldSeconds` | 0.5 | a chunk freezes in the air | settled rubble simulates longer than needed |
| `EjectImpulse` | 400 | chunks fall back into the hole they made | chunks fly like litter |
| `TumbleDegreesPerSecond` | 45 | rubble reads as rigid | rubble spins like a prop |

The first two are clamped to a safe range when read. A wall carrying an
out-of-range value says so once at BeginPlay in a `HANGRISK` line.

## Where each knob lives

| knob | lives on | survives Apply Preset? |
|---|---|---|
| table 1 | the surface material asset → `Configs[SurfaceType]` | read live |
| table 2 | `GridWallPreset` | stamped, so the preset wins |
| `bAutoLinkSeams`, `SeamToleranceCm` | `GridWallPreset` | stamped |
| **`AnchorEdges`** | the placed wall's GridData, Grid \| Anchors | **not stamped: edit it on the wall** |
| **`ThroughCouplingThreshold`, `ThroughCouplingScale`** | the placed wall's GridData, Grid \| Config | **not stamped: edit them on the wall** |
| **`bEmitDamageAttributes`** | the placed wall's GridWallMesh, Grid \| Visual | **not stamped: edit it on the wall** |
| table 3 | the placed wall's GridCollapseVisual | not stamped |

### Checking that a wall runs what the preset says

```
GridDestruction.CheckPresets
```

```
GRIDSTAMP [BP_Wall_Brick_7]: MATCH against preset 'GP_Brick' — ...
GRIDSTAMP [BP_Wall_Brick_9]: STALE against preset 'GP_Brick' — 1 field(s) differ:
    MinSupportingNeighbours live=2 preset=3. ...
GRIDSTAMP SUMMARY: 11 wall(s) — 1 stale, 0 without a preset, 10 current.
```

`STALE` means your edit never reached that wall: press Apply Preset on it and
save the level. `MATCH` means the wall really runs the asset's values, so a knob
that still looks inert is not a stamping problem. The same check runs on its own
at BeginPlay on the server, and **Log Preset Stamp Drift** beside Apply Preset
checks one wall. It reports and never corrects: a value set on the component
alone is a legitimate per-wall override.

A Blueprint's own Class Defaults override the C++ defaults. If a knob does
nothing, look at the placed actor's Details panel before anywhere else.

## Tuning order

The tables are meant to be right on the first build. When a wall does not look
like its material, work **in this order**, because each step changes what the
next one is looking at. Change one number at a time.

### 1. Silhouette: does the hole have the right shape?

Look at one hole from the front.

| what you see | do |
|---|---|
| the hole is a square | raise `FractureNoise`, or give it a break pattern |
| the rim is noisy rather than rough | lower `FractureNoise` below 0.6 |
| specks of wall survive inside the hole | lower `FractureNoise`, or raise `SpikeSuppressionChance` |
| the rim has no teeth | raise `EdgeProfile.AmplitudeU/V`, or lower `SpikeSuppressionChance` |
| brick cracks through the bricks, not the joints | give it the material's height map, or raise `FractureMaskContrast` |

### 2. Thickness: does the hole read as a break in a solid?

Look at the same hole from an angle.

| what you see | do |
|---|---|
| the edge is a knife edge | raise `HoleRimThicknessFraction` |
| a visible band, but the crater looks shallow | lower it |
| the wall's own outer edge is open | tick the right `CapEdges` |
| a shot at a doorway's reveal dents the face | raise `RevealCrushScale` |
| a shot at the face next to an edge never chips it | raise `CornerChipScale` |
| the rim looks like the face, not like broken material | use the [damage material](damage-material.md) |

### 3. Depth: does a hit dent before it opens?

Shoot the same spot repeatedly.

| what you see | do |
|---|---|
| the wall opens with no dent first | lower `CellDensity`, or raise `ThicknessCm` |
| the crater is a flat disc | raise `EdgeProfile.DepthNoise` |
| the crater bottoms out early | lower `HoleRimThicknessFraction` |
| the far face stays flat until it opens | lower `ThroughCouplingThreshold` on the wall, or raise `ExitSpallDepthScale` |
| holes are crisp but cost too much | lower `SubdivisionFactor`; see [Multiplayer](multiplayer.md) |

### 4. Identity: is it the right material?

The three knobs that carry most of a material's identity, in order:
`FalloffScaleU/V`, `SmoothingAngleDeg`, `EdgeProfile.QuantiseU`.
`DamageResistance` comes last: it changes how many shots a wall takes, not how
it looks.

Then fire along the wall:

| what you see | do |
|---|---|
| a glancing hit leaves the same disc as a square one | `ObliquityStretch` is 0: type table 2's row |
| the furrow reaches behind where the round touched | raise `ObliquitySkew` to equal `ObliquityStretch` |
| the crater has left the impact point | lower `ObliquitySkew` below `ObliquityStretch` |
| a glancing hit on steel digs as deep as a square one | `RicochetAngleDeg` is 0: type table 1's row |

### 5. Weight: does the rubble belong to the wall?

Author `PhysicalSurface` physical materials before touching `DebrisMassScale`.

### 6. The overlay: does the damage read as damage?

See [Damage material](damage-material.md).
