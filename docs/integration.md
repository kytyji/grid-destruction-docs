# Blueprint API: wiring your weapons

What to call, from where, and what the plugin never asks of you.

## The one thing to know

**The plugin never looks for your weapons. Your weapon calls the plugin.**

There is no damage pipeline to opt into, no interface your projectile has to
implement, no component your character needs, and no dependency on the Gameplay
Ability System. Wherever your game already decides "this shot hit that surface",
you add one function call.

## Two entry points. Pick one.

| | Path A: direct | Path B: tag table |
|---|---|---|
| **Call** | **Apply Grid Damage** on `GridDataComponent` | **Notify Hit By Tag** on `GridDestructionBridgeComponent` |
| **You supply** | every number: depth, radius, face | where the hit landed and which way it was going |
| **Numbers come from** | your call site | a data asset, keyed by GameplayTag |
| **Needs** | nothing | one data asset |
| **Needs GameplayTags** | no | yes |
| **Good for** | one weapon, a prototype, a system that already has per-weapon numbers | a shipping game with a weapon table |

Both are `BlueprintCallable`, both are server-only, and both do the same thing
once the numbers are resolved: **Path B is Path A with a lookup in front of it.**
Start on A; move to B when you have more than a couple of weapons.

## Path A: no Bridge, no tags, no GAS

Everything you need is on the `GridDataComponent` that every wall already has.
After your trace, in Blueprint or C++:

1. From the hit actor, **Get Component by Class** → `GridDataComponent`. None
   means it is not a destructible wall; that is your branch.
2. Fill an **`FGridDamageParams`**:

    | field | value |
    |---|---|
    | `HitLocation` | the impact point, world space |
    | `HitDirection` | the direction the round was travelling, normalised |
    | `PeakDepthCm` | how deep it bites into reference material; a rifle bullet is 3–5 |
    | `RadiusCm` | 0 for a bullet, above 0 for a blast |
    | `ImpactFace` | leave at `Auto`: it is derived from `HitDirection` |

3. Call **Apply Grid Damage**.

That is the whole integration, and it works from a Blueprint-only project.

!!! tip "Use the shot's travel direction, not the surface normal"
    The crater is shaped by the angle the round arrived at, so a glancing hit
    leaves a furrow along its path. The impact normal is the *surface's* normal
    and throws exactly that away. In a trace, the direction is
    `Normalize(TraceEnd − TraceStart)`.

Apply Grid Damage returns an **`FGridDamageResult`**: which cells opened, whether
the wall was penetrated, how much penetration depth is left for whatever is
behind it, `Obliquity` and `bGrazing`. Read it if you want the round to carry on
through, or to spark off a ricochet.

## Path B: the tag table

Worth the setup as soon as "how deep does a 7.62 go into brick" stops being a
number you want at a call site.

**Every `GridDestructibleWall` adds a `GridDestructionBridgeComponent` to itself**
at BeginPlay. To opt a wall out, clear that entry from its `CompanionComponents`
list. For an actor with a `GridDataComponent` that is not a
`GridDestructibleWall`, call **Get Or Add Grid Bridge**: it returns the one there,
or adds one.

Create the weapon table, in this order:

1. One or more **`GridImpactProfile`** data assets, each holding the numbers:
   `PeakDepthCm`, `RadiusCm`, `bIsPenetrating`.
2. One **`GridImpactProfileSet`**. Each row is a GameplayTag and a profile asset.
   Set **`DefaultProfile`** so an unmapped weapon does something visible instead
   of nothing. The picker lists only saved profiles, so it is empty until step 1
   is done.

At the call site: **Notify Hit By Tag**, passing the set, the weapon's tag, the
impact point and the direction.

**Resolution is hierarchical.** A hit tagged `Damage.Type.Rifle.556` uses a row
for `Damage.Type.Rifle.556` if there is one, otherwise `Damage.Type.Rifle`, then
`Damage.Type`. Ship one row for a whole family and override a single calibre when
it earns it.

It **returns false** when no row matched and there is no default, so "this weapon
is not in the table" is distinguishable from "the wall ignored it".

## GAS

There is nothing GAS-specific to do. Call **Notify Hit By Tag** from wherever your
damage lands: a Gameplay Ability, a `GameplayEffectExecutionCalculation` reading
the hit from the effect's context, or a target-data callback. The tag you pass is
usually one the Gameplay Effect already carries.

**Walls have no Ability System Component and do not need one.** If your damage
pipeline only applies effects to actors with an ASC, it skips walls silently.
Call the plugin where you have the hit result, before the ASC check.

## Any weapon system

The plugin does not care what fired.

| weapon | call |
|---|---|
| **Hitscan** | once per hit, `RadiusCm = 0` |
| **Projectile** | from your impact handler, with the projectile's velocity as the direction |
| **Explosion** | one call at the blast centre with `RadiusCm` set. You do not find the cells yourself |
| **Melee, thrown, vehicle impact** | the same call |
| **Lyra Starter Game** | step by step in [Example: Lyra Starter Game](#example-lyra-starter-game) |

## Authority

Every entry point is **server-only and does nothing on a client.** Cell state
replicates from the server; clients render what they are told. That makes the
calls safe in code that runs on both machines, such as a weapon ability or a
shared damage function, without guarding them yourself.

!!! warning "Called only on a client, nothing happens and nothing is logged"
    "I call it and the wall does not break" is almost always this. Check
    **Has Authority** on the calling actor first.

For multiplayer, add **`GridNetClientComponent`** to your PlayerController or
pawn. It needs no code: it lets a client ask the server to resend a wall that went
out of sync, so it heals at once. See [Multiplayer](multiplayer.md#fast-resync).

## Other calls worth knowing

| call | on | what it does |
|---|---|---|
| **Capture Damage Snapshot** / **Apply Damage Snapshot** | GridDataComponent | save and restore a wall's damage ([Setup](setup.md#carrying-damage-between-sessions)) |
| **Reset Damage** | the wall | back to its Initial Damage, else intact (server) |

## Example: Lyra Starter Game

The plugin was also tested in a copy of Epic's **Lyra Starter Game**, a common
starting point for shooters. Lyra needs no adapter and no code: the wiring below
is a few Blueprint nodes and two data assets in your project, and nothing in
Lyra's source changes. It is also a worked example of [Path B](#path-b-the-tag-table)
for any GAS-based weapon system.

### 1. Walls

Place `GridDestructibleWall`s and apply presets as usual. Lyra's weapons trace on
`Lyra_TraceChannel_Weapon`, and Lyra's `DefaultEngine.ini` makes the `BlockAll`
profile block it; the wall's collider uses `BlockAll`. If you give the collider
your own profile, make it block that channel.

### 2. Impact numbers

Lyra never calls `TakeDamage`, so a weapon hit is passed to the wall through the
tag table ([Path B](#path-b-the-tag-table)):

1. A **`GridImpactProfile`** data asset, for example `DA_Impact_Bullet`:
   `Peak Depth Cm` 3–5, `Radius Cm` 0.
2. A **`GridImpactProfileSet`**, for example `DA_GridImpacts_Lyra`:
   `Default Profile` = `DA_Impact_Bullet`. Add rows (`Tag` → `Profile`) per
   weapon later.

### 3. Hitscan weapons

Content Browser → Settings → **Show Plugin Content**, then open `GA_Weapon_Fire`
from ShooterCore (the parent of `GA_Weapon_Fire_Pistol` and the others; if the
children's parent is the C++ class `LyraGameplayAbility_RangedWeapon`, edit each
child instead). In its Event Graph, find
**Event On Ranged Weapon Target Data Ready**:

1. Put a **Sequence** after the event: **Then 0** → Lyra's existing damage logic
   (Ctrl-drag the existing wire, do not delete it); **Then 1** → the nodes below.
2. **For Loop** from 0 to `Get Data Count From Target Data` (Target Data) − 1.
3. In the loop body: `Get Hit Result From Target Data` (Target Data, Index) →
   **Break Hit Result**.
4. `Hit Actor` → **Get Or Add Grid Bridge** → **Is Valid** (not a wall: skip).
5. On the Bridge: **Notify Hit By Tag** with
    - `Profile Set` = `DA_GridImpacts_Lyra`, `Impact Tag` empty (the set's
      Default Profile) or the weapon's tag;
    - `Origin` = `Impact Point`;
    - `Hit Direction` = `Normalize(Trace End − Trace Start)`;
    - `Hit Instigator` = `Get Avatar Actor From Actor Info`.
6. Compile and save. Children without their own
   `On Ranged Weapon Target Data Ready` inherit this; a child that has one
   overrides it, so add the same nodes there or **Add Call to Parent Function**.

The event also runs on the client (shot prediction). That is fine:
Notify Hit By Tag does nothing off the server and returns false there.

!!! success "Worked when"
    A shot in a listen-server PIE session makes a hole in both windows, and
    Notify Hit By Tag returns true on the server. False on the server: the set
    has no profile for the tag and no Default Profile. No hole and the node never
    runs: the shot went through a child ability with its own event (step 6).

### 4. Grenades and projectiles

In the explosion's or projectile's impact Blueprint, on the server: the same
**Get Or Add Grid Bridge** → **Notify Hit By Tag**, at the blast centre, with a
profile whose `Radius Cm` is set. One call per blast; the wall finds the cells.

### 5. Fast resync

Put `GridNetClientComponent` on the controller: open
the **ShooterCore** GameFeatureData (or your Experience's) → **Actions → Add
Components** → Actor Class **`LyraPlayerController`**, Component Class
**`GridNetClientComponent`**, Client ✓ Server ✓. Lyra sets `MaxClientRate` to
200000, so the default `MaxBytesPerSecond` leaves room for everything else.

## Cost at the call site

- **Path A** is a component lookup and a function call. No allocation.
- **Path B** adds a walk over your profile rows once per hit. Tag comparison is
  name comparison, not string comparison.
- **Neither path allocates per shot.** The damage work is proportional to the
  cells the hit touches, not to the size of the wall.
