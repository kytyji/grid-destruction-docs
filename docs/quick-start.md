# Quick start

From an empty project to a wall that breaks, in about ten minutes. Each step
says what you should see when it worked, because most failures in a destruction
setup are silent and look like "nothing happened".

## 1. Install

**From Fab.** In the Epic Games Launcher, open **Library → Fab Library**, find
Grid Destruction and click **Install to Engine** for 5.7. Then, in your project,
open **Edit → Plugins**, search for *Grid Destruction*, tick it and restart the
editor.

**Into one project.** Create a `Plugins/` folder in the project root, and put
the plugin at `Plugins/GridDestruction/`, so that this folder directly contains
`GridDestruction.uplugin`. For a C++ project, right-click the `.uproject` →
**Generate Visual Studio project files**, open the `.sln`, pick
**Development Editor** and build.

!!! success "Worked when"
    The editor opens and `GridDestructibleWall` appears in the **Place Actors**
    panel.

## 2. Play the showcase

The plugin ships a showcase level, so you can see every material break before
you set anything up.

1. In the Content Browser, open **Settings → Show Plugin Content**.
2. Open **Plugins → GridDestruction Content → Demo →
   `L_GridDestruction_Showcase`** and press **Play**.
3. You fly a camera that shoots:

    | key | action |
    |---|---|
    | WASD | move |
    | E or Space / Q or Ctrl | up / down |
    | left mouse button | fire; hold for the cutter; throw the grenade |
    | **1** / **2** / **3** / **4** | rifle / breaching charge / cutter / grenade |
    | mouse wheel | cycle weapons |
    | **R** | reset every wall |
    | **H** | show or hide the network panel |

The level has every shipped material, built-up walls, doors and windows with
glass, piers carrying a beam, a room with a roof, a cantilever, stacked walls
and a corner of two materials, each one labelled. Every wall in it is an
ordinary `GridDestructibleWall` with a `GP_*` preset, so select one to see how
it is set up.

## 3. Your first wall

1. Drag **`GridDestructibleWall`** into your level.
2. In its Details panel, set **Preset** to one of the shipped presets, for
   example `GP_Brick` (in `/GridDestruction/Presets/`).
3. Click **Apply Preset**.

!!! success "Worked when"
    The Output Log shows two lines like these, with your wall's own numbers:

    ```
    ApplyPresetToGridData [GridDestructibleWall_0]: preset='GP_Brick' grid 16x12, cell 25.0cm, ...
    Grid initialized: 16x12 cells, 25cm each. ... HasAuthority=1, cells=192
    ```

!!! warning "Apply Preset is not optional"
    A preset is *stamped* onto the wall. A value edited on the preset reaches a
    placed wall only when you press **Apply Preset** on that wall, and a value
    typed on the wall's own component is overwritten the next time you press
    it. This is the most common reason a setting "does nothing".
    `GridDestruction.CheckPresets` lists every wall that is out of date.

## 4. Break it without a weapon

Press **Play**, select the wall in the World Outliner, and use the buttons in
its Details panel:

| button | what it is for |
|---|---|
| `DealTestDamage(Location, PeakDepthCm, RadiusCm, Face)` | one hit, precisely placed |
| `DealTestDamageRandom(PeakDepthCm)` | a hit somewhere, to prove the pipe works |
| `DealTestBlast(Location, Depth, Radius, Lobes, Raggedness, Seed, Face)` | a ragged crater instead of a circle |

`RadiusCm = 0` is a bullet; above 0 is a blast. Pass a non-zero `Seed` to
`DealTestBlast` and the same crater comes back every time.

!!! success "Worked when"
    A dent appears, then a hole, and debris falls from it.

<video src="../assets/clips/concrete-breach.mp4" poster="../assets/clips/concrete-breach.jpg" autoplay loop muted playsinline style="width:100%;border-radius:6px"></video>

## 5. Check it in multiplayer

In the Play dropdown set **Number of Players = 2** and **Net Mode = Play As
Listen Server**. Damage the wall from the server window: the hole appears on the
client too. Then type in either window's console:

```
GridDestruction.HashCells
```

```
GRIDHASH [Server/PIE] WORLD: hash=0x4B10D8F2 over 3 wall(s).
GRIDHASH [Client/PIE] WORLD: hash=0x4B10D8F2 over 3 wall(s).
GRIDHASH VERDICT: MATCH across 2 worlds.
```

MATCH means both machines hold the same cell data.

## 6. Connect your weapons

One call where your game already knows "this shot hit that surface": **Apply
Grid Damage** on the wall's `GridDataComponent`. It works from Blueprint, needs
no GameplayTags and no GAS, and must run on the server. The
[Blueprint API](integration.md) page has the details and the tag-table
alternative; it also has step-by-step wiring for Lyra Starter Game.

## Where next

- [Setup](setup.md): openings, building tools, floors and ceilings, layered
  walls, saved damage.
- [Surface presets](surface-presets.md): every number per material, and the
  order to tune them in.
- [Damage material](damage-material.md): make the shipped material use your
  textures.
- [Effects and sounds](effects.md): what the wall fires and how to replace it.
- [Troubleshooting](troubleshooting.md): console commands and the symptoms
  people actually run into.
