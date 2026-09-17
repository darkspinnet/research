# Chunk 144 teleporter contact boundary

This note resolves the contact boundary created by retail Lua chunk `144`
(`BossSecurityTeleporterPassive`) in build `5.3.0.103`. It distinguishes the
retail physics predicate from darkspin's current radius-`6.67` compatibility
shortcut.

## Result

The trigger is a PhysX sphere of radius `2` centered at the teleporter object's
world position. Contact is not a point test against the entrant center, a
rendering AABB, or a capsule. PhysX tests that trigger sphere against each
entrant `NxShape`. The tutorial player Blitz has one authored spherical physics
shape with unscaled `boundingRadius=0.5` and `graphicsScale=1.6`, giving an
entrant radius of `0.8`.

The sphere/sphere contact boundary is therefore

```text
distance(playerCenter, teleporterCenter) <= 2 + (0.5 * 1.6)
distanceSquared <= 2.8 * 2.8
distanceSquared <= 7.84
```

Radius `6.67` has no retail contact authority. It is the existing darkspin
compatibility harness and must not be described as chunk `144`'s boundary.

The native contact path does not perform a segment/sphere sweep. It receives a
PhysX trigger shape, the overlapping other shape, and a trigger-status bitmask
after scene simulation. A movement command supplies or changes a locomotion
goal; it does not itself contain or generate a contact mask. Retail locomotion
moves the player through intermediate physics poses, and those simulation poses
produce enter/stay/leave notifications. Treating the previous command goal and
the new command goal as one swept segment is an adapter approximation, not the
recovered retail predicate.

## Evidence

### Chunk 144

Chunk `144`, resource `13674`, SHA-256
`c2e727c6cf4c13708fa315321fb51f0402a3a4a9763eaf2f8d782c1eca3267f0`, calls

```text
CreateTriggerVolume(x, y, z, 2, callback1, callback2, callback3)
```

at the teleporter agent position. The native Lua registration at `0x00A11423`
maps `CreateTriggerVolume` to `sub_A0AC20`. That wrapper reads three coordinates,
one radius, and three callbacks and calls `sub_A16D00`. The corresponding box
API maps to a different wrapper and constructor (`sub_A0AD10 -> sub_A16BE0`).
The sphere constructor passes its radius and center to the simulator's dynamic
trigger factory; it does not pass bounds, a height, or segment endpoints.

The trigger's native filter mode is `0`, so the native layer does not restrict
contact to player objects. Chunk `144` performs the gameplay filter: accept an
entrant only when it is player-controlled or its owner is player-controlled.
The private active/inactive teleporter state is a second, independent condition.

### Entrant shape

`AssetData_Binary.package` resource
`004724_76a8f7d8_00000000_00000000a823e2e9.bin` decodes to the base
`PC_EL_Rogue` noun (`771` bytes). It identifies `builtins!sphere`, references
`DefaultPhysics.prop`, stores `graphicsScale=1.6`, and stores the authored
`boundingRadius=0.5`. The same radius and scale were independently exercised by
the tutorial marker boundary. The deployed Blitz collision radius is therefore
`0.8`, not the visual platform radius and not a capsule radius.

The retail trigger report at `sub_A24F90` has the PhysX 2.x
`NxUserTriggerReport` shape: trigger shape, other shape, status mask. It invokes
the other shape's virtual `getActor` path and reads the actor's game-object data.
Consequently, the native callback receives the object owning the shape that
actually overlapped. It never replaces that shape with an object-center or
render-bounds query. For an actor with several physics shapes, any accepted
shape overlap could report the actor; Blitz's authored contact shape here is the
sphere above.

This agrees with NVIDIA's description of PhysX shapes as the spatial extents
used for intersection tests and trigger volumes, and of trigger reports as
overlap notifications rather than contacts resolved by the solver:

- [PhysX rigid-body collision and trigger shapes](https://docs.nvidia.com/gameworks/content/gameworkslibrary/physx/guide/Manual/RigidBodyCollision.html)
- [PhysX 2.x `NxUserTriggerReport` migration mapping](https://docs.nvidia.com/gameworks/content/gameworkslibrary/physx/guide/Manual/MigrationFrom28.html)

No explicit skin-width override is present in the dynamic sphere creation path.
The recovered gameplay boundary is the two authored sphere radii above.

## Notification masks and movement

`sub_A24F90` tests all supplied bits independently and in this order:

| Mask | PhysX meaning | Chunk 144 callback | Effect |
| ---: | --- | --- | --- |
| `1` | `NX_TRIGGER_ON_ENTER` | callback 1 | If the teleporter is active and the entrant passes player ownership, request `TeleporterModifier` GUID `0x502f1932`. |
| `2` | `NX_TRIGGER_ON_LEAVE` | callback 2 | Empty; no modifier request and no state mutation. |
| `4` | `NX_TRIGGER_ON_STAY` | callback 3 | If the modifier is absent, invoke callback 1 again; otherwise suppress the duplicate request. |

NVIDIA's PhysX 2-era flag descriptions likewise define enter as a shape entering
the volume, leave as leaving it, and stay as continuing to intersect it:
[APEX schema descriptions of the PhysX 2 trigger flags](https://docs.nvidia.com/gameworks/content/gameworkslibrary/physx/apexsdk/1.3.1/_static/build_params/structDestructibleActorParam.html).

For one actor/trigger pair, the conceptual overlap-state mapping is:

| Previous overlap | Current overlap after a physics step | Mask/notification |
| --- | --- | --- |
| false | false | none |
| false | true | enter, bit `1` |
| true | true | stay, bit `4` |
| true | false | leave, bit `2` |

The native dispatcher would call multiple callbacks in `1, 2, 4` order if
PhysX supplied combined bits. Normal movement should be modeled as state
transitions, not as “one mask per movement packet.” In particular:

- an `ActionMovement` command changes the goal; subsequent authoritative
  locomotion/physics steps decide overlap;
- another movement command while already overlapping must not synthesize a new
  enter; continuing overlap is stay;
- `ActionStopMovement` while overlapping does not imply leave and can continue
  to receive stay notifications;
- the modifier's eventual teleport changes the entrant pose and can cause a
  later leave notification, but chunk `144`'s leave callback is empty;
- destroying the trigger unreferences it without synthesizing leave.

Chunk `144` creates the trigger with delay `0` and repeat/latch flag `0`.
Callback 3 can therefore run for every PhysX stay notification. Its cadence is
the physics notification cadence, not a recovered duration or a movement-packet
count.

## Correct Go predicate

A parity physics/director boundary needs retained overlap state and the current
authoritative player pose:

```go
const teleporterRadius = float32(2)
const blitzRadius = float32(0.5 * 1.6)
const contactRadius = teleporterRadius + blitzRadius // 2.8

deltaX := playerPosition.X - teleporterPosition.X
deltaY := playerPosition.Y - teleporterPosition.Y
deltaZ := playerPosition.Z - teleporterPosition.Z
isOverlapping := deltaX*deltaX+deltaY*deltaY+deltaZ*deltaZ <= contactRadius*contactRadius
isEntering := !wasOverlapping && isOverlapping
```

Before acting on `isEntering`, validate finite coordinates, the live tutorial
phase/trigger instance, the authoritative actor binding, player control or
player-controlled ownership, and chunk `144`'s active private state. Commit the
new overlap state even when the Lua gameplay filter rejects the entrant, because
the contact pair belongs to physics while acceptance belongs to the script.

Do not use `doesTutorialSegmentIntersectSphere(previousGoal, newGoal, center,
6.67)` as the parity predicate. If darkspin temporarily keeps a
movement-command-only shortcut without authoritative locomotion integration, a
segment test padded to `2.8` can reduce tunneling, but it remains explicitly a
compatibility approximation. It cannot reproduce enter/stay/leave cadence from
command endpoints alone.
