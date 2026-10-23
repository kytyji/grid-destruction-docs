# Effects and sounds

Walls fire their own effects and sounds, **where material actually came off**,
on every client. The weapon does not fire them at the bullet's point, a shot
that opens nothing spawns nothing, and a player who joins mid-fight does not see
old damage replayed.

**It works out of the box.** The shipped `DA_GridSurfaceMaterials` has Niagara
systems and sounds assigned for every material, and every
`GridDestructibleWall` adds a `GridDestructionFXComponent` at runtime that plays
them. There is nothing to wire. Nothing here replicates, and nothing plays on a
dedicated server.

!!! tip "Keep your weapon's impact effect"
    It is instant for the shooter. The wall's effects arrive with replication,
    one network update later on a client. The weapon says "hit"; the wall says
    "this is what broke".

## The slots

Each row of the surface material asset (`Configs` → a material → **Effects**)
has four effect slots and three sound slots.

| slot | fires | once per | +X points | User parameters |
|---|---|---|---|---|
| `BreakFX` | material opens: a hole, an exit blowout, a cut through an end or a bore | **cluster** of fine cells that opened together | out of the face the material **broke out through**: the back for a shot from the front, the side itself for a cut through a cap or jamb | `CellCount`, `ExtentCm`, `Normal`, `ParticleBudget`, `ExitsBack`, `ExitsSide` |
| `CollapseFX` | a piece of wall breaks away | falling piece | out of the wall's front | `SizeCm`, `CellCount`, `Normal` |
| `LandingFX` | a falling piece hits something hard | piece, once | up | `SizeCm`, `CellCount`, `ImpactSpeed` |
| `ImpactDustFX` | a hit knocks material off a face or a side **without** opening anything | patch; at most `MaxDustPerBatch` per batch | out of the face that lost it | `CellCount`, `RemovedDepthCm`, `ExtentCm`, `Normal` |

| sound slot | plays | notes |
|---|---|---|
| `BreakSound` | with `BreakFX` | a short crunch with variations |
| `CollapseSound` | with `CollapseFX`; when set, `BreakSound` does not also play for those cells | a heavy crack |
| `LandingSound` | with `LandingFX` | the rubble must simulate physics |

- **Fine resolution.** Breaks and dust are detected per fine cell, so a bullet
  hole smaller than a cell fires them.
- **Drawn once.** Cells opened by a collapse fire `BreakFX` only if `CollapseFX`
  is empty. A hole fires `BreakFX`, not dust as well. A corner shared by two
  joined walls fires on one wall only.
- **Niagara or Cascade.** A slot accepts either.

## Parameters a system receives

The system spawns **rotated**: local **+X** points as in the table above, and
**+Z up the wall** (if +X itself points up or down, +Z points out of the wall's
face). Author everything that should fly "out" along local +X, and everything
that lies along the wall in local Y/Z. The effect then turns to fit any wall,
including a sloped one.

Declare the parameters in the system's **User Parameters**, with exactly these
names and types. A parameter the wall does not set keeps its default, and a typo
means the parameter always stays at its default.

| parameter | type | meaning |
|---|---|---|
| `User.ExtentCm` | float | half the size of the break or patch: about 5–15 cm for a bullet, 50–150 cm for a grenade. **The main scale parameter** |
| `User.ParticleBudget` | int | already scaled by `sg.EffectsQuality` (Low 3 / Medium 8 / High 15) and never above `CellCount`. Use it as the number of chunks |
| `User.CellCount` | int | how many fine cells opened: 1–4 for a bullet at subdivision 3, tens to hundreds for a grenade |
| `User.Normal` | vector | the wall's outward direction |
| `User.ExitsBack`, `User.ExitsSide` | bool | the break came out of the back, or out of a side |
| `User.SizeCm` | float | size of a falling piece |
| `User.ImpactSpeed` | float | landing speed, usually 250–1500 cm/s |
| `User.RemovedDepthCm` | float | how much material a dent removed in total: 1–5 cm for one bullet |

## Tuning on the wall

| knob | on | default | what it does |
|---|---|---|---|
| `MaxSpawnDistanceCm` | GridDestructionFX | 6000 | skips effects far from the local camera |
| `MinSoundIntervalSeconds` | GridDestructionFX | 0.08 | shortest gap between two sounds of one kind from one wall, so automatic fire does not become a wall of noise |
| `SoundVolumeScale` | GridDestructionFX | 1 | the wall's overall volume |
| `bEmitDust` | GridDebrisSpawner | on | turns the dent dust off |
| `MaxDustPerBatch` | GridDebrisSpawner | 4 | caps dust per batch |
| `DustMinErosionBytes` | GridDebrisSpawner | 3 | ignores the trickle the far-face coupling adds around a hit |
| `LandingMinSpeedCmS` | GridCollapseVisual | 250 | how fast a piece must be moving for a sudden stop to count as a landing |

**Per-wall override.** In a wall Blueprint, add `GridDestructionFXComponent`
yourself, tick `bOverrideMaterialEffects` and fill `OverrideEffects`. The wall
does not add a second one.

The shipped sounds share `SCon_GridDestruction` (a Sound Concurrency) and
`SA_GridDestruction` (attenuation) in `/GridDestruction/Audio/`. A grenade opens
many clusters at once, so keep a concurrency limit on any sounds you assign.

## Making your own systems

The shipped systems are a starting point. When you replace them, these are the
rules that make wall effects read as physical.

### Three layers

Real destruction is three things at once, and a fake-looking effect almost
always shows only one:

| layer | what it is | how it behaves |
|---|---|---|
| **chunks** | pieces of 1–8 cm | leave fast, fall on a parabola, bounce once and lie still |
| **grit** | sand, under 1 cm | fastest, short-lived, barely visible one by one |
| **dust** | a cloud | starts fast, **brakes hard**, grows 2–4×, settles and fades over 2–4 s |

- **Dust is the wall's colour, lighter and greyer.** Brick dusts reddish,
  concrete grey; wood gives splinters and sawdust rather than dust.
- **Timing.** Chunks and grit are done in the first 0.1 s; dust lives 20–40 times
  longer.
- **Size follows the hole.** Scale by `ExtentCm`. A bullet hole with a cloud the
  size of half the wall reads worse than no effect.
- **Lit particles.** Unlit dust glows in shadow. Use a lit translucent material
  (Volumetric PerVertex NonDirectional is cheap) with Depth Fade 20–50 cm.

### Chunks (GPU)

1. Sim target **GPU**, local space **off**, **fixed bounds on** (about ±300 cm).
2. Spawn burst: count = `User.ParticleBudget`.
3. Spawn position: a box `(2, ExtentCm × 2, ExtentCm × 2)` in local space, not a
   point: the YZ plane is the wall.
4. Velocity: a cone around local +X, 35–55°, 150–600 cm/s. Smaller pieces faster.
   Optionally 20–30% at half speed along −X: the entry side loses grit too.
5. Gravity and **drag 0.4–1.2**: without drag every piece flies the same arc.
6. Mesh renderer with 4–6 chunk meshes, random index and rotation, sizes weighted
   to the small end.
7. GPU depth-buffer collision, restitution 0.05–0.2, high friction.
8. **Freeze on first contact.** When the collision flag is set, zero the
   velocity, gravity and drag, **and the rotation rate** (or scale it by
   `saturate(length(Velocity) / 50)`), or a landed chip spins until its lifetime
   ends. A settled particle then costs rendering only, and depth-buffer collision,
   which only knows what is on screen, no longer matters to it.
9. Scale the mesh to 0 over the last 10% of its life.

### Dust

CPU is fine (few particles). 4–12 sprites from a disc of radius
`ExtentCm × 0.6` in the YZ plane; velocity along +X 100–300 cm/s in a 30–45° cone
with **drag 3–6**, so the cloud bursts out and hangs; size grows ×2–4 over its
life; alpha peaks at 0.25–0.5 quickly and fades slowly; life 1.5–4 s; slow
rotation and a weak curl noise.

### Per material

| | Concrete | Brick | Stone | Wood | Metal | Glass |
|---|---|---|---|---|---|---|
| chunks | angular rubble | brick fragments and grey mortar | larger angular rubble | splinters, 3–12 cm | torn strips, few | flat shards, many |
| cone half-angle | 45° | 40° | 35° | 55° | 30° | 70° |
| speed, cm/s | 200 | 200 | 150 | 260 | 120 | 350 |
| drag | 0.6 | 0.6 | 0.5 | 1.2 | 0.4 | 0.9 |
| restitution | 0.05 | 0.05 | 0.05 | 0.10 | 0.15 | 0.20 |
| dust colour (linear) | 0.45 0.44 0.42 | 0.50 0.34 0.26 | 0.42 0.40 0.37 | 0.55 0.45 0.32, little | none | none |
| special | — | two chunk colours | — | sawdust instead of dust | sparks: short, bright, velocity-aligned | high specular, roughness 0.05 |

### Recipes per slot

- **`BreakFX`** = chunks + grit + dust. Multiply count and speed by
  `ExitsBack ? 1.3 : 1.0` for a bigger exit than entry. For a bullet keep it short
  and modest (3–8 chunks, 4–6 puffs, gone in 2 s); a grenade is the same system
  scaled by its parameters. Check both.
- **`ImpactDustFX`** is the most frequent effect, so keep it the cheapest: 3–6
  puffs, alpha ≤ 0.3, life 1–2 s, scaled by `clamp(RemovedDepthCm / 4, 0.4, 1.5)`.
- **`CollapseFX`**: large, slow dust from the piece's volume (30–120 cm/s, grows
  ×3, 3–5 s, settles down) and 10–30 small pieces falling from the break with
  almost no initial speed.
- **`LandingFX`**: +X is up, so the YZ plane is the floor. A ring of dust of
  radius `SizeCm × 0.5`, moving radially outward at 80–250 cm/s with drag 3–5,
  strength `clamp((ImpactSpeed − 250) / 1000, 0.3, 1)`.

### Performance rules

The component takes systems from Niagara's pool and culls them by distance. For
that to work:

1. **Every system must finish.** All emitters *Loop Behavior* **Once**, the
   system's *Inactive Response* **Complete** or **Kill**. An endless emitter never
   returns to the pool: a leak you only see after an hour of play.
2. **Give the systems an Effect Type** with a max distance around 6000 cm, an
   instance cap of 16–32 per system, and scalability by `sg.EffectsQuality`.
   Its visibility culling must **Deactivate**, not Pause: a paused system off
   screen keeps its particles forever.
3. **GPU emitters need fixed bounds.**
4. **No per-pixel sorting, distortion or lights on dust.** Light renderers only
   for metal sparks, only on High.
5. **Check:** `stat Niagara` after a minute of shooting should level off, not
   climb. `fx.DumpNCPoolInfo` shows pooled components in use.

### Checklist per material

- [ ] dust matches the wall's colour, but lighter
- [ ] dust brakes and grows rather than flying evenly
- [ ] chunks vary in size, small ones fly further
- [ ] one system gives a small bullet effect and a large grenade effect
- [ ] a shot through from the front throws chunks out of the back
- [ ] a shot into an end or a reveal dusts out of that side
- [ ] particles are lit (check in shadow)
- [ ] the system finishes (pool instances do not grow)
- [ ] it looks decent on Low

## Driving your own effects from events

To spawn effects yourself instead of through the slots, leave the slots empty and
bind the wall's events. `GridDebrisSpawnerComponent` spawns nothing itself: it
watches cells open, groups them into clusters, applies a budget, primes a
baseline so a joining client does not replay every old hole, and broadcasts.

| event | on | fires |
|---|---|---|
| `OnDebrisBurstBP` | GridDebrisSpawner | once per cluster of opened cells, with `FGridDebrisBurst` |
| `OnDustBP` | GridDebrisSpawner | a dent that opened nothing |
| `OnChunkDetachedBP` | GridCollapseVisual | a piece breaks away |
| `OnChunkLandedBP` | GridCollapseVisual | a falling piece lands |

The collapse events fire after the component has finished its own work, so a
listener may damage the wall safely.

`FGridDebrisBurst`:

| field | use it for |
|---|---|
| `WorldLocation` | where to spawn: the cluster centre, on the wall's mid-surface |
| `WorldNormal` | the wall's outward direction, always the front |
| `ExitNormal`, `bExitsBack`, `bExitsSide` | which way to throw: out of the face the material broke out through |
| `CellCount` | how many cells opened, counted at the finest level. Scale the burst; never spawn per cell |
| `ExtentCm` | half-size of the cluster: how wide to spread the spawn |
| `Surface` | which system and material to pick |
| `bFromCollapse` | the cells were opened by a collapse |
| `SuggestedParticleCount` | already scaled by `sg.EffectsQuality` |

One system per surface type beats one system with a switch: the mesh, material
and size distribution all differ.

## What this deliberately does not do

- **Particles are never shootable and never block anything.** Settled geometry
  that blocked shots would stand in a different place on every machine, because
  rubble does not replicate. Nothing in gameplay depends on where particles land.
- **A doorway-sized plank is not a particle.** Particles work at fist scale and
  below. A large falling piece is real geometry for about a second, from
  `GridCollapseVisualComponent`; particles carry what is left on the floor.
- **Two clients may disagree about the dust.** That is fine, and it costs no
  bandwidth.
