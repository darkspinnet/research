# 1-1 Security Teleporter contact geometry

## Result

Build `5.3.0.103` does not use a three-unit point test for the five routed
`SecurityTeleporter` pads. The packaged generic teleporter passive creates a
PhysX sphere of authored radius `2` at the teleporter agent position. PhysX
reports overlap against the entrant's actual `NxShape`; for the observed Blitz
entrant, the packaged `PC_EL_Rogue.Noun` supplies a radius-`0.8` sphere. Its
center-to-center contact boundary is therefore `2 + 0.8 = 2.8` units, or
`distanceSquared <= 7.84`.

This is an instantaneous physics-overlap predicate evaluated at simulated
poses. Retail does not sweep the straight segment between two movement-command
endpoints. A command-endpoint segment test is a server-adapter approximation
that can catch crossings which retail would only see if locomotion actually
occupied an overlapping intermediate pose.

The separate security-clear query is authored at radius `20`, not `35`.
Native range lookup broad-phase-expands that sphere by each candidate object's
spatial radius before chunk `144` applies its live/team/attribute filter. The
current radius-`35` center-point test is therefore neither the authored query
radius nor the recovered native geometry.

## Five route pads and their packaged noun

The current five source pads are authored rows in
`zelems_1_design.Markerset` (`level 56`, marker set `1791`, resource `7327`).
The numeric suffixes are marker display-name disambiguators; all five deploy
the same packaged base noun, `SecurityTeleporter.Noun`.

| Current route index | Marker row / authored ID | Displayed noun | Position |
| ---: | --- | --- | --- |
| `0` | `125486 / 1714532343` | `SecurityTeleporter.Noun-1` | `(-141.3413,83.7317,0.0340)` |
| `1` | `125487 / 1802649208` | `SecurityTeleporter.Noun-2` | `(556.1375,-38.2726,0.0175)` |
| `2` | `125488 / 952042169` | `SecurityTeleporter.Noun` | `(601.7911,42.2988,33.0519)` |
| `3` | `125461 / 4047011481` | `SecurityTeleporter.Noun-4` | `(-543.5760,546.1642,0.1581)` |
| `4` | `125473 / 1915578781` | `SecurityTeleporter.Noun-3` | `(-625.0099,655.0704,0.1358)` |

The sixth design marker, row `125462` / ID `1432124960`, is
`SecurityTeleporter.Noun-5` at `(191.8558,742.7648,0.0523)`. It is not a source
in the current five-hop fallback; its nearby spawn at
`(191.1970,736.2937,0.0443)` is the fallback's final arrival.

`AssetData_Binary.package` proves the shared base asset identity:

| Resource | Package ordinal | Type / group / instance | Decoded identity |
| --- | ---: | --- | --- |
| `SecurityTeleporter.Noun` | `7959` | `0x76A8F7D8 / 0 / 0x30A7C477` | `654` bytes; SHA-256 `1dbd53f715248f8ca8447207d5617f3cd220850c2cd0392c1234a8c397377ea1` |
| `SecurityTeleporterPassive` companion | `7958` | `0xEEEB0E31 / 0 / 0x30A7C477` | `666` bytes; SHA-256 `d5af5366a15369aacd01ea989f120609037d7a60dbe0b3f341ec1953cd113a25` |

The same-instance noun names `SecurityTeleporter.AIDefinition`; its companion
names `SecurityTeleporterPassive`. Chunk `144` registers that generic passive
and `BossSecurityTeleporterPassive` through the same security-teleporter
template. The generic pads therefore use the same radius-`2` trigger and
radius-`20` security scan already recovered for the boss form; this is a
content link, not a name-based analogy.

## Authored teleporter noun footprint versus trigger volume

The decoded generic noun authors:

- `graphicsScale = 1`;
- local noun bounds from `(-5.93,-5.93,0.01)` to `(5.93,5.93,0.65)`;
- visual model `Shared!teleporter_level.bmdl`;
- physics properties `DefaultPhysics.prop`;
- no packaged `builtins!sphere`, capsule, inline box, or inline trigger shape.

Those bounds describe the circular platform/model envelope. In the native
no-geometry fallback, `sub_9D70E0` derives the XY footprint as the diagonal of
the largest absolute X/Y extents times runtime scale:

```text
sqrt(5.93^2 + 5.93^2) * 1 = 8.386286...
```

That approximately `8.386`-unit noun footprint is not the entry volume. Chunk
`144` independently calls `CreateTriggerVolume(x,y,z,2,...)`; the resulting
dynamic PhysX sphere is centered on the agent and has no authored height or
orientation. Neither the `11.86`-unit platform width nor its `8.386`-unit
fallback footprint is added to the trigger radius.

## Player noun footprint

Contact is shape-versus-shape, so there is no universal "player radius."
The deployed creature's packaged noun and runtime scale determine its PhysX
shape. An actor with multiple accepted shapes can report contact when any one
overlaps the trigger.

For the live route observation, Blitz was active. Packaged
`PC_EL_Rogue.Noun` (type `0x76A8F7D8`, group `0`, instance `0xA823E2E9`,
ordinal `4724`, decoded SHA-256
`7aa6ab3a11d78d8ee830d4a2a7bd85e1ea49bf1d3c926d1710c68b67ff0bd4fe`)
authors `builtins!sphere`, base `boundingRadius=0.5`, and
`graphicsScale=1.6`. At runtime scale one, Blitz's shape radius is `0.8`.

For this entrant only, the recovered boundary is:

```text
distance(Blitz center, trigger center) <= 2 + (0.5 * 1.6)
distanceSquared <= 2.8^2
distanceSquared <= 7.84
```

Another deployed creature must use its own authored shape. For example, the
already projected Sage noun has a radius of `0.825`; substituting Blitz's
`0.8` globally would be another fallback, not build-103 parity.

## Native consumers in `Game.c`

The canonical decompiler establishes the geometry and notification boundary:

1. `sub_A0AC20` at `0x00A0AC20` (`Game.c:1430928`) reads `(x,y,z)` from
   Lua arguments `1..3`, radius from argument `4`, and three callbacks from
   `5..7`, then calls `sub_A16D00`. The separate box wrapper is
   `sub_A0AD10`; chunk `144` does not call it.
2. `sub_A16D00` at `0x00A16D00` (`Game.c:1440617`) forwards the center,
   one radius, callback mask, and filter mode to the simulator's dynamic
   trigger factory. It supplies `0.0` for the two other shape dimensions.
   There is no capsule height, box extent, path segment, or added gameplay
   tolerance.
3. `sub_A24F90` at `0x00A24F90` (`Game.c:1452108`) receives the trigger
   shape, the other `NxShape`, and a PhysX status mask. It obtains the actor
   attached to the overlapping other shape and dispatches bit `1` to callback
   one, bit `2` to callback two, and bit `4` to callback three.
4. `sub_A15F00` at `0x00A15F00` (`Game.c:1439761`) invokes the retained
   Lua callback with the trigger ID and entrant object ID. It does not replace
   the overlapping shape with a center-distance or render-bounds test.

Chunk `144` maps the mask roles as follows:

| PhysX state bit | Meaning | Retail behavior |
| ---: | --- | --- |
| `1` | enter | If active and player-controlled/owned, resolve the linked destination and request `TeleporterModifier` (`0x502F1932`). |
| `2` | leave | Empty callback. |
| `4` | stay | Retry the enter callback only when that entrant no longer has the teleporter modifier. |

The overlap state is retained by the physics scene: outside-to-inside is
enter, inside-to-inside is stay, and inside-to-outside is leave. Stopping or
sending another movement command while still inside does not create another
enter. Teleporting the actor away can produce a later leave, but the authored
leave callback is empty.

## Retail security authority

Contact and security clearance are independent predicates. Chunk `144` polls
the latter every `0.5` seconds in a stable state by calling
`GetObjectsInRadius(teleporterPosition,20,organicDamageableObjectTypes)`.
`sub_A10260` (`Game.c:1434912`) passes that authored center and radius to
`sub_9D4300`, which reaches broad-phase query `sub_9F72C0`.

`sub_9F72C0` (`Game.c:1415825`) admits a spatial leaf when:

```text
distanceSquared(queryCenter, candidateCenter)
    < (queryRadius + candidateSpatialRadius)^2
```

Chunk `144` then counts only candidates which are alive, have team `0`, and
have `InvisibleToSecurityTeleporters == 0`. Zero qualifying candidates powers
the teleporter up; a nonzero count powers it down. Entry additionally requires
the teleporter's private active state and a player-controlled entrant (or an
entrant whose owner is player-controlled). These filters and state transitions
are retail server authority; the visible active/inactive effects are only its
client presentation.

The native broad-phase comparison is strict at exact tangency, whereas the
PhysX trigger API reports actual shape overlap. No chunk-`144` constant or
dynamic-trigger construction argument adds a server epsilon, skin width, or
lag allowance. No explicit skin-width override was found in this creation
path. Any tolerance added by darkspin must therefore be labeled a compatibility
policy rather than authored geometry.

## Comparison with the current fallback

| Concern | Build-103 predicate | Current fallback | Consequence |
| --- | --- | --- | --- |
| Entry volume | Radius-`2` PhysX sphere versus entrant `NxShape`; Blitz combined radius `2.8` | Swept movement-command segment versus radius-`3` point sphere | `3` gives Blitz an unsupported `0.2`-unit radial allowance, ignores per-noun shapes, and a segment can trigger without retail occupying an overlapping simulated pose. |
| Teleporter model | Local bounds `11.86 x 11.86 x 0.64`; fallback noun footprint about `8.386` | Pad treated as a point | The model footprint is real authored data but is not entry geometry. |
| Threat query | Authored radius `20`, broad-phase-expanded by each candidate radius, then alive/team/attribute filters | Radius-`35` center-point test over current enemy snapshots | For an ordinary roughly one-unit candidate, retail's center reach is about `21`, so `35` is roughly `14` units broader and lacks the exact retail filter set. |
| Motion sampling | Shape overlap at authoritative physics poses with enter/stay/leave state | One straight segment between accepted packet endpoints | The fallback prevents tunneling but is not the retail predicate or notification cadence. |
| Tolerance | No recovered gameplay epsilon | Entry radius rounded up from Blitz `2.8` to `3` | The extra `0.2` is server policy only. |

This recovery invalidates the geometry portions of the existing
`campaign-1-1-first-security` help fallback. It does not prove the current
five source-to-arrival edges: the normalized design markers still have
`target_marker_id=0`, so their nearby spawn pairing remains a separate retail
server-authority gap.

The asynchronous Security Teleporter transfer now relocates every living
companion owned by the entering hero at the same deadline that commits the
hero destination. The direct contact-transfer path does the same. This closes
the disconnected-navigation failure captured with Zrin's Pain Hounds, where
the hero arrived on the next island but ordinary follow movement left the
hounds reserved on the source island.

## Reproducible evidence

- Generic noun: `bin/game/logs/security-teleporter-noun.bin`, decoded directly
  from package ordinal `7959`.
- Generic passive companion:
  `bin/game/logs/security-teleporter-passive.bin`, decoded from ordinal `7958`.
- Chunk `144`: content DB row `144`, server-data resource `13674`, bytecode
  SHA-256
  `c2e727c6cf4c13708fa315321fb51f0402a3a4a9763eaf2f8d782c1eca3267f0`.
- Canonical native source: `bin/game/GameBin/Game.c` and the address
  references listed above.
- Marker identities and positions: `notes/campaign/1-1/route.md`.
- Prior shape/contact proof for the boss registration and Blitz entrant:
  `notes/tutorial/teleporter-contact.md`.
