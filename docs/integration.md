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
| **Lyra** | step by step in [Multiplayer → In a Lyra project](multiplayer.md#in-a-lyra-project) |

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

## Cost at the call site

- **Path A** is a component lookup and a function call. No allocation.
- **Path B** adds a walk over your profile rows once per hit. Tag comparison is
  name comparison, not string comparison.
- **Neither path allocates per shot.** The damage work is proportional to the
  cells the hit touches, not to the size of the wall.
