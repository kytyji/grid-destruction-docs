# Multiplayer

Damage is decided on the server, and every client renders the same cells. This
page covers testing it, the bandwidth budgets and keeping clients in sync.

## Testing in the editor

Play dropdown → **Number of Players = 2**, **Net Mode = Play As Listen Server**.
Damage a wall from the server window; the hole appears on the client within
about one network update.

Once per session, or after changing anything about replication, type in
**either** window while PIE is running:

```
GridDestruction.HashCells
```

It hashes every live world and prints its own verdict:

```
GRIDHASH [Server/PIE] WORLD: hash=0x4B10D8F2 over 3 wall(s).
GRIDHASH [Client/PIE] WORLD: hash=0x4B10D8F2 over 3 wall(s).
GRIDHASH VERDICT: MATCH across 2 worlds.
```

**MATCH** means both machines hold the same cell data, so anything that still
looks wrong is in the mesh, not the network. **MISMATCH** prints the first cell
where they disagree, and how.

Run it **during play**: after PIE ends only the editor world is left, and the
command refuses to give a verdict over one world. Each wall's line also carries
`net=… epoch=… match`, the server's published hash for that wall against this
machine's own, so one window's log already says whether it agrees. Between two
PCs, compare the `world hash` on each machine's net panel (below).

## What replicates

Only damaged cells. An undamaged wall costs nothing after its first update, and
a late joiner receives only the cells that were actually hit.

Rubble, effects, sounds and the damage material's look do not replicate: every
machine derives them from the cells.

## Bandwidth and budgets

**Project Settings → Plugins → Grid Destruction — Network** holds every budget:

| setting | default | what it trades |
|---|---|---|
| `ItemsPerShardPerFlush` | 128 | Cells one wall sends per update. Lower: holes appear over a few more frames under heavy fire, with flatter peaks. Keep it well under 512: the engine refuses an update over 2048 changes, and a lost packet merges updates. |
| `MinFlushIntervalSeconds` | 0.1 | Shortest time between two updates of one wall. |
| `MaxBytesPerSecond` | 65536 | All walls together. Keep it well under your project's `MaxClientRate` (engine default 100000, Lyra 200000): damage shares the connection with movement. |
| `BurstBytes` | 32768 | How much may go out at once after a quiet spell, such as the first grenade. |
| `bKeyframeAfterQuiet` | on | After a wall has been quiet, resend the cells the last burst changed, once, from what new damage leaves of the budget. Heals a client that missed an update. |
| `KeyframeQuietSeconds` | 3 | How long "quiet" is. |
| `DesyncCheckDelaySeconds` | 0.5 | How long a client waits for data to settle before it compares its wall with the server's. |
| `ResyncRequestCooldownSeconds` | 2 | Shortest time between two resends of one wall on request, and between two requests from one player. |
| `LateJoinWarnBytes` | 49152 | Walls whose worst case is over this get a warning at start (below). |

**If holes lag under very heavy fire** (cutters, grenade spam), the byte budget
is what limits it. A brick cell at subdivision 4 costs up to about 62 bytes, so
the default budget carries roughly 1000 cells per second across all walls. Raise
`MaxBytesPerSecond` if your `MaxClientRate` has room, or lower
`SubdivisionFactor`.

`GridNet.Measure` (on the server, development builds) prints what each wall
costs on the network now, in bytes, and its worst case.

## Desync detection and self-heal

The server publishes a hash per wall. A client compares each wall with it; when
they still differ on a second look, it logs a `DESYNC` warning and asks the
server to resend the wall, and the resent cells overwrite the wrong ones.
`GridDestruction.HashCells` shows the result.

### Fast resync

Add **`GridNetClientComponent`** to your PlayerController, or to the pawn, so a
client can ask for that resend at once. Without it the wall still heals, at the
server's next resend after `KeyframeQuietSeconds`. The component needs no code.
In a GameFeature-based project such as Lyra, add it to the PlayerController
class with an **Add Components** action.

### The net panel

With `bShowNetDebug` on the component (the sample character has it on; **H** or
`GridNet.ToggleDebug` toggles it), each machine draws a panel: role, ping,
connection rates, how many walls match the server's published hash, the world
hash (`HashCells`' number: equal on two PCs means equal walls), items sent or
received, `DESYNC`s, and the busiest walls. A wall reads **settling** while its
data is still arriving, and **MISMATCH** only once the detector confirms it.
Every 5 seconds it also logs a `GRIDNET STAT` line, so two machines' logs can be
compared afterwards. The panel is compiled out of Shipping builds.

To watch the heal: `GridNet.CorruptCell <Wall> <Cell>` (development builds)
changes one damaged cell on this machine only.

## Late-join size

A joining client receives each wall's damaged cells in one update, and the
engine refuses one over about 64 KB. A wall that could exceed `LateJoinWarnBytes`
when fully damaged logs a warning at start, and `GridDestruction.CheckPresets`
lists it in the editor. A 48×48 wall is about 32 KB at subdivision 1 and about
140 KB at subdivision 4. Three ways out:

1. lower `SubdivisionFactor` or the grid size;
2. split the wall into two walls (joined walls behave as one);
3. raise the limit on **both** server and client, in `DefaultEngine.ini`:

    ```ini
    [ConsoleVariables]
    net.MaxConstructedPartialBunchSizeBytes=262144
    ```

A rejoining player gets every wall back with its damage, not instantly: the
damaged cells arrive at the connection's rate, then the meshes rebuild.

## Replication system and cheating

**Replication system.** Built and tested on the default (legacy) replication
system. Iris is untested; the server logs a warning if it runs with Iris.

**Dedicated servers.** Walls simulate damage and structure there, and spawn no
effects or sounds. A dedicated server cannot read textures, so a wall's break
pattern (the fracture mask or its material's height map) is baked when you save
the level, and the server uses that. **Save the level after changing a wall's
material, mask or size.** A wall spawned at runtime on a dedicated server has no
saved pattern: it breaks by the default pattern, with a warning in the log.

**Cheating.** Damage is decided on the server, and shots and movement are
checked against the server's collider. Detecting memory edits or wallhacks
belongs to your game's anti-cheat (EAC, BattlEye), not to this plugin.
