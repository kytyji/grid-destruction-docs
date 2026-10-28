# Damage material

The plugin ships a wall material that draws damage in the shader:

- chips with parallax depth inside dents;
- cracks spreading from dents and holes;
- the material's interior on broken rims;
- dust around holes.

It reads everything from vertex data the wall mesh already carries, so:

- **nothing travels over the network**, and a player who joins mid-match sees
  the same wall;
- **nothing is computed on the CPU** when a wall is hit;
- **an intact wall looks exactly like its textures.**

Cost: three texture reads and about 40 ALU instructions per pixel of wall. No
extra draw calls.

## What ships

In `/GridDestruction/Materials/`:

| asset | what it is |
|---|---|
| `M_Grid_DamageWall` | a complete wall material; make instances of it |
| `MF_GridDamageUV` | function: UV → `FaceUV` with the parallax offset, and `DetailUV` |
| `MF_GridDamage` | function: your BaseColor, Roughness and Normal → the same with damage, plus AO |
| `T_Grid_DamageDetail`, `T_Grid_DamageDetail_N` | the crack and chip texture and its normal map |
| `MI_Grid_Damage_Brick`, `_Concrete`, `_Stone`, `_Wood`, `_Metal`, `_Glass` | demo instances with tuned values and placeholder textures in `Materials/Surfaces/` |

The shipped `GP_*` presets already use the demo instances, and the CSV's
`FractureWeightMask` row gives brick, stone and wood their placeholder height
maps as the break pattern.

## Make an instance for your surface

1. Right-click `M_Grid_DamageWall` → **Create Material Instance**. Save it **in
   your own project**, for example `/Game/Materials/MI_Brick_Damage`, not in the
   plugin's Content, so a plugin update never overwrites it.
2. Open the instance. Under **Parameter Groups** you see **Surface** and
   **Grid Damage**. A parameter stays grey until you **tick the box to the left
   of its name**; unticked, the instance takes the parent's value.
3. Set the Surface group (below), save with **Ctrl+S**. The preview and the
   walls in the level update at once.

You need **one instance per wall material**: `MI_Brick_Damage`,
`MI_Concrete_Damage` and so on.

### Surface group

| parameter | what to set |
|---|---|
| `BaseColorMap` | the albedo, for example of your brick. sRGB on, like any colour texture |
| `NormalMap` | the normal map. Its texture must have **Compression = Normalmap**, or the parameter reports a sampler type error |
| `BaseTint` | a colour multiplier. White by default: the texture as it is |
| `Roughness` | one number: brick 0.85–0.9, concrete 0.9. For a roughness map, use the functions instead (below) |
| `UVTiling` | the number your old material multiplied TexCoord by. If it used TexCoord directly, leave 1 |
| `DamagedColorMap` | **optional.** A texture of the broken material: bare brick without patina, concrete with exposed aggregate, fresh wood. Shown in the damaged area and on rims when `DamagedMapAmount` > 0 |
| `DamagedNormalMap` | **optional.** The normal map for `DamagedColorMap`, Compression = Normalmap, so the relief changes too |

!!! success "Check"
    On an **intact** wall the instance must look exactly like your old material.
    If it does not, the Surface group is set wrong; the damage part has nothing
    to do with it.

## Put it on the walls

**Through the CSV**, so the value is not lost: write the instance's path in the
`WallMaterial` row of the right column of `GridMaterials.csv` and run the table
script ([Surface presets](surface-presets.md#editing-the-numbers-gridmaterialscsv)).

**Without the CSV:** put the instance in the preset's `WallMaterial` and press
**Apply Preset**.

!!! warning "Not on the actor"
    A material assigned on the wall actor is overwritten by the preset the next
    time it is applied. `GridDestruction.CheckPresets` should print `MATCH`.

`bEmitDamageAttributes` on the wall's **GridWallMesh** component is on by
default. If nothing reacts to hits, check it on the placed actor, not the class.

## Tuning without PIE

1. Shoot a wall in PIE and run `GridDestruction.CaptureDamage` to save its damage
   into a damage preset ([Setup → saved damage](setup.md#pre-damaged-walls-and-saved-damage-beta)).
2. Place a test wall with that preset as its **Initial Damage**. The editor
   viewport shows it damaged.
3. Open the instance next to the wall and turn the parameters: changes show at
   once.

It helps to have light dents, a deep crater and a hole through on one preset.

## Grid Damage parameters, in tuning order

### a) Crack scale: `DetailTiling` (default 2)

Crack tile size = `UV Tile Cm / (UVTiling × DetailTiling)`. With a 100 cm UV tile
and `UVTiling` 1 the tile is 50 cm: a crack network cell of about 11 cm and chips
of 2–4 cm.

- For brick use 2–3, so cracks are in scale with the bricks.
- If you see a repeating pattern, use a non-integer such as 2.37, so it does not
  line up with the brick tile.

### b) When chips appear: `DentGain` (default 4)

Chips show fully once `1 / DentGain` of the thickness is gone. The rule:

`DentGain ≈ ThicknessCm / the dent depth in cm at which chips should show fully`

For full chips at a 4 cm dent:

| material | thickness | DentGain |
|---|---|---|
| brick | 20 | 5 |
| concrete | 25 | 6 |
| stone | 40 | 10 |
| wood | 8 | 2 |

A good target: one bullet leaves visible but not solid chipping.

### c) Cracks from holes: `BreachCracks` (default 1), with `BreachFalloffCells`

- `BreachFalloffCells` on the preset (brick: 2.5 cells) sets how far the zone
  around a hole reaches.
- `BreachCracks` sets how densely cracks fill it: 1.5–2 for more cracks at the
  edge, 0 for cracks only inside dents.
- To make cracks reach further from a hole, change `BreachFalloffCells` in the
  CSV, not the material.

### d) The damaged area: `DamageTint`, `AreaSoftness`, `DamagedMapAmount`

A dent is coloured as one patch, and the patch's edge is:

- **ragged and chipped**, like knocked-off plaster, not the shape of the cells;
- **sharp, a pixel wide at any distance**: neither blurry up close nor stepped
  far away;
- sometimes **stopped by cracks**, where the breakup is lower.

| parameter | default | what it does |
|---|---|---|
| `DamageTint` | 0.8, 0.75, 0.72 | colour multiplier inside the patch. Lower is darker and more visible; 1, 1, 1 turns the patch off |
| `AreaSoftness` | 1.5 | edge width in pixels: 1 is sharpest, 3–4 softer. Below 1 it may shimmer far away |
| `DamagedMapAmount` | 0 | 0: the patch and rims use the face texture, tinted. 1: they show the broken material (`DamagedColorMap`, `DamagedNormalMap`, `DamagedRoughness`). The demo instances use 1 |
| `DamagedRoughness` | 0.95 | roughness of the broken material |

**Rims** (caps and opening reveals) look exactly like the face material on an
intact wall. The broken material appears on a rim only as it is damaged: at the
edge of a hole through, and where a side hit chipped the rim.

**Intact and broken in one material.** The patch grows with the cell's damage:
the deeper the damage, the more of the surface becomes the broken material. It
works like two instances, intact and broken, blended by the damage mask, but in
one material and one pass. Hits shallower than `DentStepCm` leave the geometry
flat and show only here.

If you cannot see the patch, light dents are too shallow for `DentGain`: raise
`DentGain` or darken `DamageTint`.

### e) Colours

| parameter | what it does |
|---|---|
| `CrackColor` | crack colour, near black; dark brown for brick |
| `ChipTint` | multiplies the surface colour inside a chip. Below 1 gives shadow and a sense of depth |
| `InteriorTint` | multiplies the surface colour on a hole's rims. Can go above 1: a fresh break is lighter than the surface |
| `DustAmount`, `DustTint` | dust around holes. Its colour is the **average colour of the texture** × `DustTint` (slightly lighter by default: 1.15, 1.12, 1.08). The master material reads the average from the smallest mip of `BaseColorMap`, so dust always matches the material |

### f) Depth: `ParallaxDepth` (default 0.02), tune last

An offset in fractions of the face tile: 0.02 on a 100 cm tile is about 2 cm at
grazing angles.

- Check it by walking along the wall at a sharp angle. If the texture inside
  chips swims, lower it.
- Working range 0.01–0.04; 0 turns parallax off.

### g) Relief: `NormalStrength` (default 1)

Concrete and stone 1.2–1.5, brick 1.

### Starting values

| | Brick | Concrete | Stone | Wood |
|---|---|---|---|---|
| DentGain | 5 | 6 | 10 | 2 |
| DetailTiling | 2.4 | 2 | 1.6 | 3 |
| BreachCracks | 1.2 | 1.5 | 1.2 | 0.5 |
| ParallaxDepth | 0.02 | 0.025 | 0.03 | 0.01 |
| CrackColor | 0.06, 0.035, 0.03 | 0.05, 0.05, 0.05 | 0.04, 0.04, 0.04 | 0.05, 0.035, 0.02 |
| ChipTint | 0.85, 0.75, 0.7 | 0.9, 0.9, 0.9 | 0.85, 0.85, 0.85 | 0.8, 0.7, 0.6 |
| InteriorTint | 1.1, 0.9, 0.8 | 1.15, 1.15, 1.15 | 1.05, 1.05, 1.05 | 1.2, 1.05, 0.85 |
| DustTint | 1.15, 1.1, 1.05 | 1.15, 1.15, 1.15 | 1.1, 1.1, 1.1 | 1.2, 1.15, 1.05 |
| DustAmount | 0.3 | 0.4 | 0.3 | 0.2 |
| NormalStrength | 1 | 1.3 | 1.5 | 0.8 |
| DamageTint | 0.8, 0.7, 0.65 | 0.85, 0.85, 0.85 | 0.8, 0.8, 0.8 | 0.75, 0.65, 0.55 |
| AreaSoftness | 1.5 | 2 | 1.5 | 1 |

The crack textures (`DamageDetail`, `DamageDetailNormal`) do not need
overriding: one pair suits brick, concrete, stone and plaster. Swap them only for
a different kind of break, such as wood splinters or torn metal. The crack
texture is made for mineral materials, so metal and glass look best with their
own detail textures.

## Keep your own material: the two functions

If you already have a good material, for example with roughness and AO maps,
keep it and wire in the two functions instead:

```
TexCoord[0] (× your tiling)  --> MF_GridDamageUV.UV
MF_GridDamageUV.FaceUV       --> the UVs input of EVERY surface Texture Sample
                                 (colour, normal, roughness, AO)
MF_GridDamageUV.DetailUV     --> MF_GridDamage.DetailUV
your albedo at its smallest mip --> MF_GridDamage.SurfaceAverage   (optional)
  (a Texture Sample of your albedo: MipValueMode = MipLevel, MipValue = 14,
   UVs = Constant2 0.5, 0.5. Unwired, dust is neutral grey)
your broken-material texture --> MF_GridDamage.DamagedColor   (optional,
  with UVs = FaceUV; used when DamagedMapAmount > 0)
your BaseColor / Roughness / Normal --> the inputs of MF_GridDamage
MF_GridDamage: BaseColor / Roughness / Normal --> the material's pins
MF_GridDamage.AO             --> Ambient Occlusion (multiply with your own AO)
```

The Grid Damage parameters appear in your material's instances, and everything
above applies to them.

**Without `FaceUV` on the textures there is no parallax**: the surface textures
must take their UVs from `FaceUV`, or the offset has nothing to move.

## Parallax: where it is and how to see it

The parallax is kept cheap, so it is **subtle by nature**:

- **It works only where there is depth**: in dents and open cracks. On an intact
  wall the offset is zero.
- **It shows only at an angle.** Straight on the offset is close to zero, which
  is physically right. Walk along the wall and look at a dent from the side.
- **To check it is wired:** set `ParallaxDepth` to 0.1 for a moment and look at a
  dent at an angle. The texture inside the chips should sink and swim. Then set
  it back to 0.02–0.03.
- **If nothing changes** with your own material, the textures are not taking
  `FaceUV`.

Most of the depth still comes from the dent's geometry, the chip normal and the
AO; parallax adds the sense of "inward".

## The data under the material

Use this to write your own damage material, or to check the data is arriving.
Every vertex of a wall's visual mesh carries four damage channels and a second UV
set, filled on every rebuild, at 20 bytes per vertex. The collider never carries
them.

### Vertex colour

| channel | carries | range | use it for |
|---|---|---|---|
| **R** | remaining material at this vertex: the thickness left, or what side cuts have left of a cell's width, whichever is lower | 1 intact → 0 eaten through | crack masks, roughness, depth, tint toward the interior colour |
| **G** | breach proximity | 1 at an opened cell → 0 at `BreachFalloffCells` | dust, soot, crack density, aggregate around a hole |
| **B** | interior-surface mask | 1 on rim and cap quads, 0 on the two faces | **the material switch**: `Lerp(FaceFinish, InteriorFinish, B)` |
| **A** | which face | 1 front, 0 back | a different finish per side. Read it only where B is 0: `Lerp(Back, Front, A * (1 - B))` |

**B is the one that earns the feature.** A hole's rim and a capped edge are the
broken thickness a player looks into. Without B a rim inherits the face's brick
or plaster, and the hole reads as a cut in paper. Wire B first, then R, then G.

R always matches the geometry the player sees, and every machine computes the
same values.

### UV1

`TexCoord` index **1**: 0..1 across the whole wall on both axes, whatever its
size, and independent of `UV Tile Cm`. It does not move when the wall is damaged.
Use it for anything that should stay put on the wall: a grime gradient from the
floor up, a decal atlas, a per-wall variation mask.

### Seeing a channel with your own eyes

Make a scratch material with **Shading Model: Unlit** and plug `VertexColor`
into **Emissive Color**. Put it in the **preset's** `WallMaterial` (not on the
actor) and press Apply Preset. Shoot the wall: red fades toward black in craters
(R), green blooms around openings (G), and every rim and capped edge is blue (B).
For UV1, plug `TexCoord` index 1 into Emissive: a clean red-green gradient across
the wall, unchanged by shooting it.

- B flat: the wall has no rim to show. Check `HoleRimThicknessFraction` and
  `CapEdges`.
- G flat: `BreachFalloffCells` is 0, or nothing has gone through yet.
- Everything flat: `bEmitDamageAttributes` is off on the placed actor.

## What you see → what to turn

| symptom | cause and fix |
|---|---|
| a light or white patch around damage | that is dust. It takes the texture's average colour, so a light patch means `DustTint` is too high or, with your own material, `SurfaceAverage` is not wired. Set `DustAmount` to 0 to check |
| the whole wall is white or light grey | `BaseColorMap` is not set: the default is a white texture |
| the whole wall is darker than before | `BaseTint` is not white |
| a checkerboard or an error on `NormalMap` | the normal texture's Compression is not Normalmap |
| nothing changes on hits | the material did not reach the wall (run `CheckPresets`), `bEmitDamageAttributes` is off, or `DentGain` is too low |
| one bullet chips the whole dent | lower `DentGain` |
| cracks are huge or tiny | `DetailTiling` |
| a repeating pattern shows | give `DetailTiling` a fractional part |
| the texture swims as the camera moves | lower `ParallaxDepth` |
| no parallax | see [Parallax](#parallax-where-it-is-and-how-to-see-it) |
| a hole's rims look like the face | channel B: check `HoleRimThicknessFraction` |
| chips on the back opposite a dent | expected: R is the total thickness, so the back also "sees" the dent. It reads as spalling from behind |
| the chip edge follows the cell grid | the mask is interpolated between vertices. Finer cells (`SubdivisionFactor`) or a higher `DetailTiling` help |
| the patch edge is blurry | `AreaSoftness` is too high |
| the patch edge shimmers far away | raise `AreaSoftness` to 2 |

If your symptom is not here, send a close and a distant screenshot of the wall
with the instance's Details panel.
