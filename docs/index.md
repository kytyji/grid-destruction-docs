# Grid Destruction

**Server-authoritative destructible walls for Unreal Engine 5 multiplayer.**

Shoot a wall and a hole appears in real geometry. Bullets, grenades and players
pass through it, and every client sees the identical hole.

This is not decals, a swapped-out "broken" mesh or client-side cosmetics. The
server owns a grid of cells, and each cell erodes from its front and back faces
independently. Everything else is rebuilt from that one model: the surface, the
collider, the structural collapse and the debris.

[Quick start](quick-start.md){ .md-button .md-button--primary }
[Wire your weapons](integration.md){ .md-button }

<video src="assets/clips/room-collapse.mp4" poster="assets/clips/room-collapse.jpg" autoplay loop muted playsinline style="width:100%;border-radius:6px"></video>

*The showcase level: a glazed wall blows out, then a room loses its walls and the roof comes down.*

---

## What you get

| Feature | What it means |
|---|---|
| **Real holes** | A procedural surface rebuilt from cell state. Craters go the full depth of the wall, rims show its real thickness, and edges chip at the subdivision's own resolution rather than smearing. |
| **Matching collision** | The collider carries the same holes as the mesh, so what you can see through, you can shoot and walk through. |
| **Replication you can verify** | Only damaged cells replicate, budgeted to stay under the engine's limits, and clients never author damage. Each client checks its walls against a hash the server publishes and heals itself on a mismatch. `GridDestruction.HashCells` prints its own MATCH / MISMATCH verdict. |
| **Adaptive subdivision** | A cell subdivides into fine children on its first bite. Detail exists only where the wall has been shot, so an untouched wall costs what a coarse grid costs. |
| **Per-material behaviour** | Concrete, brick, stone, wood, metal and glass differ in penetration, spall, edge crumble, debris weight and grazing response. Glass is whole or gone. Every number is a documented preset field, and all of them live in one CSV table. |
| **Every side breaks** | A shot on a top cap, an end or a window jamb cuts that side along its own normal, and the bore of a hole widens as it is shot through. |
| **Joined walls** | Walls that meet at a corner or end to end find each other. The seam disappears, a hit crosses the joint, and pieces hold each other up only through real ground. |
| **Structural collapse** | Detached pieces fall as rubble with the weight of their material. A ceiling rests on the walls below it, and an overhang longer than the material can carry gives way. |
| **Authored break patterns** | A mask, or the height map of the wall's own material. Brick cracks along its mortar joints. |
| **A ready damage material** | Dents read as chips and cracks with parallax depth, holes get cracks and dust, and broken rims show the inside of the material. An intact wall looks exactly like its textures. |
| **Effects and sounds** | Break, collapse, landing and dust effects and sounds fire per material, from the side the piece came off, with no Blueprint wiring. |
| **Level-building tools** | Walls sized in centimetres, built by clicks, resized with viewport handles, grouped into kits. Window and door openings are cut live in the editor. Walls are part of the navmesh. |
| **Saved damage** *(beta)* | Capture a wall's damage as an asset for pre-damaged buildings, or as a struct your save game carries. |
| **Physical materials on the collider** | Your existing impact decals and surface-type logic answer correctly against a destructible wall. |

## Requirements

- Unreal Engine **5.7**, **Win64**.
- A C++ project, or the Fab binaries for a Blueprint-only project.
- The `ProceduralMeshComponent` and `Niagara` engine plugins. Both are declared
  as dependencies, and the editor enables them for you.

The Core module has no dependency on Lyra, on the Gameplay Ability System or on
anything but the engine. The plugin has been tested in a blank project and in a
Lyra Starter Game copy, as a listen server, with clients on the same PC and on
a second PC.

## Not in this version

- Walls are **planar**. Curved and cylindrical sections are on the roadmap.
- Built and tested against **Unreal Engine 5.7 only**, on **Win64**.
- The shipped damage material is **one generic look** (cracks, chips, dust). The
  data under it is documented channel by channel, for a material of your own.
- **A late joiner receives each wall's damaged cells in one update**, which the
  engine caps at about 64 KB. A very large wall at the highest subdivision can
  exceed that when fully destroyed. Such walls get a warning at start, and
  [Multiplayer](multiplayer.md#late-join-size) lists the ways out.
- Saved damage is **beta**: it works, but the capture workflow is still rough.

## Modules

Five modules with a strict one-way dependency. That direction is what makes the
plugin usable outside the project it was built in.

```
GridDestructionSample    ← a worked example: a pawn that shoots the wall
        │
        ▼
GridDestructionBridge    ← hit → grid damage, keyed by GameplayTag
        │
        ▼
GridDestructionCore      ← zero Lyra, zero GAS, zero project dependencies
        ▲
        │
GridDestructionFX        ← effects and sounds from the wall's events (optional)
GridDestructionEditor    ← viewport handles, kits, click building (editor only)
```

| Component | Role |
|---|---|
| `UGridDataComponent` | Model: cell state, replication, structural queries |
| `UGridWallMeshComponent` | Render: the wall's procedural surface |
| `UGridCollisionComponent` | Physics: a procedural collider with the same holes |
| `UGridDebrisSpawnerComponent` | Groups opened cells into debris bursts |
| `UGridCollapseVisualComponent` | Detached pieces falling |
| `UGridDestructionBridgeComponent` | Bridge: a hit becomes grid damage |
| `UGridNetClientComponent` | Lets a client ask for a resend at once; the on-screen net panel |
| `UGridWallPreset` | The data asset you tune |

## Credits

Developed by **kytyji**. Published on Fab by **Anna Gonezhuk**.

## Support

**annybelogurova@gmail.com**

Two things save a round trip when you report something: whether you saw it in
the server window, the client window or both, and the output of
`GridDestruction.HashCells` typed once while it is happening.

Grid Destruction is distributed through Fab. Your rights to a purchased copy
come from the Fab Content License Agreement.
