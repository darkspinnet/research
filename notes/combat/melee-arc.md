# Melee arc and turn-in-place wire contract

## Result

Build 103's shared melee template does **not** select from an arc frozen at
attack start. At the hit continuation it calls `MakeHitArc`, which reads the
attacker's **live** position, facing, and footprint. The template does retain
the attack-start attacker position and derived direction, but that direction
is passed to `TakeDamage`; it is not an input to `IsObjectInArc` or
`FindBestTargetInArc`. The retained initial target position has one separate
selection role: an owner-latched `TargetInRangeAtStart` target that moved
strictly less than `missMovementAmount` is accepted without an arc test.

`TailZap` and `TutorialPoisonMelee` share these selection values:

```text
hitArcLength      = 1.25
hitAngle          = 90 degrees (shared-template default)
missMovementAmount = 1.0 (shared-template default)
```

Their accepted activation also shares the same native turn boundary because
their AI definitions set `faceTarget=true`. Before Lua starts, native code
replaces translation with a turn-in-place goal and directly sends exactly one
`ObjectPlayerMove` (`0x91`) command. It is neither the ordinary stopped
`flags=0x20` shape nor the walking `0x91`/`0x95` pair:

```text
0x91 ObjectPlayerMove, flags = 0x42, 81 bytes including opcode
```

That `0x91` contains the stationary goal, normalized facing, and faced target
position. No `0x95 LocomotionDataUnreliableUpdate` and no reflected `0x94`
locomotion update are emitted by this activation path. Facing is therefore
serialized explicitly in the `0x91`, not left as a client-only inference.

## Evidence and identities

The build-103 content identities are:

| Role | Chunk / resource | Source | SHA-256 |
| --- | ---: | --- | --- |
| Shared melee template | `832 / 14409` | `Abilities/0x7BF2D7DD.lua` | `2951fef16876a72687c235cc93377ede9dcec31286a8237a9c6a8af978da778a` |
| TailZap | `411 / 13962` | `Abilities/0xFB57253F.lua` | `5724700129df632f31ecee861c0743d67b1451728420cd8e3507c3de7d2a894e` |
| TutorialPoisonMelee | `898 / 14476` | `Abilities/0x2B8B0FB2.lua` | `a4d5f23fd349091f1e1cb1deefec4d2dde913cf3b8c3735d36ac156a03a81033` |

The exact template instruction flow and constants are build-103
content-proven. The retained shared TargetUtils source in the server-reference
corpus establishes the bodies of `MakeArc`, `IsObjectInArc`,
`FindObjectsInArc`, and `FindBestTargetInArc`; build-103 bytecode independently
fixes every call and argument the melee template supplies to those helpers.
The turn mutation and direct sender are build-103 client-native-proven at:

| Address | Role |
| ---: | --- |
| `0x009E0AE0` | ability activation; conditionally calls the turn and immediately publishes it before starting the ability |
| `0x00A15610` | constructs the turn-in-place locomotion goal |
| `0x00A20210` | copies the fixed 80-byte command body and sends logical message `0x12`, wire `0x91` |

Focused IDA diagnostics are in
`bin/game/logs/melee-arc-locomotion-wire.log`. The primary executable is the
build-103 binary identified in the research ledger by SHA-256
`3c7248a4c6614450290a66c22ab7fcda86052b29a7b555e834d7c23f9b73ee5b`.

## What is retained at attack start

The shared `tick` closure captures, before animation and before its timed wait:

1. attacker ID and team;
2. attacker attribute snapshot;
3. animation sequence index;
4. owner-supplied `TargetInRangeAtStart`;
5. target object ID;
6. target object's position, if the ID is nonzero;
7. attacker position;
8. the ability target position;
9. `direction(attackerPosition, abilityTargetPosition)`;
10. release and hit deadlines.

The two position snapshots have different later uses:

- the retained target-object position is compared with that target's live
  continuation position for the one-unit movement exception;
- the retained attacker position is only used to derive the retained attack
  direction;
- the retained direction is passed as the last three arguments to
  `TakeDamage` for the selected target;
- neither retained attacker position nor retained direction constructs the
  selection arc.

This corrects the tempting but wrong model in which the wind-up freezes an arc
origin and ray. In the normal uncorrected case the distinction is visually
hidden: native activation has already stopped translation and set facing, so
the live continuation transform normally still equals the accepted-start
transform. External movement, teleport, correction, knockback, or another
facing mutation exposes the difference.

## Exact continuation selection

At the hit deadline the template waits first, then performs the following
selection flow. `none` below is `kObjIDNone`.

```text
PayCooldownAndMana()

arc = MakeHitArc()
selected = none

if originalTarget != none and
   ValidateHostileTarget(retainedAttackerTeam, originalTarget):
    liveTargetPosition = GetPosition(originalTarget)
    displacement = distance(retainedTargetPosition, liveTargetPosition)

    if TargetInRangeAtStart and displacement < 1.0:
        selected = originalTarget
    else if IsObjectInArc(arc, originalTarget):
        selected = originalTarget

if selected == none:
    selected = FindBestTargetInArc(arc, attackerID)
```

The comparisons matter:

- `displacement < 1.0` is strict; exactly one unit does not take the bypass;
- the bypass does not test current angle, arc contact, or current activation
  range;
- the original target is preferred whenever it survives either branch;
- fallback can redirect the swing to another currently hostile object;
- if no target is found, both abilities continue without damage because the
  shared default `abortOnNoTarget` is false.

The original-target validity and all fallback hostility tests use the retained
attacker team but continuation-time object validity/state.

### Arc construction and footprints

Both abilities override `hitArcLength` with ranked value `1.25`. Their
`MakeHitArc` calls:

```text
nTargetUtils.MakeArc(attackerID, 1.25, 90, true)
```

With the final argument true, `MakeArc` reads the attacker's live transform and
builds:

```text
origin    = live attacker center
facing    = live attacker facing
length    = 1.25 + attacker footprint radius
halfAngle = radians(90) / 2 = pi/4
```

The true branch is important. The other `MakeArc` mode offsets the origin
forward by the attacker footprint; melee does not use that mode. It leaves the
origin at center and adds the footprint to arc length instead.

For the tutorial nouns:

| Ability / attacker | Attacker footprint | Arc length passed to intersection |
| --- | ---: | ---: |
| TailZap / TutorialSloth | `1.30` | `2.55` |
| TutorialPoisonMelee / TutorialBasicPoison family | `1.40` | `2.65` |

`IsObjectInArc` then reads the candidate's **live** position and footprint. It
rejects only when `abs(candidateZ - arcZ) > 20`, so a difference of exactly 20
survives, and passes the candidate's footprint circle to
`CircleIntersectsArc` with the live center, the arc center/facing, the length
above, and full angle `90` degrees. The target is consequently selected by
footprint intersection, not by requiring its center to lie inside the wedge.

Against the tutorial hero's `0.80` footprint, the straight-ahead radial contact
envelope of the direct intersection call is based on:

```text
TailZap:            1.25 + 1.30 + 0.80 = 3.35 center units
TutorialPoisonMelee: 1.25 + 1.40 + 0.80 = 3.45 center units
```

Those are arc-contact extents, not activation ranges. Admission remains the
separate footprint-expanded range predicate:

```text
TailZap:             1.30 + 0.80 + 0.75 = 2.85
TutorialPoisonMelee: 1.40 + 0.80 + 0.75 = 2.95
```

The angular sides also use circle-versus-arc intersection, so a target center
slightly outside the nominal 45-degree half-angle can still qualify when its
footprint overlaps the wedge.

### Fallback target ranking

`FindBestTargetInArc` first asks `FindObjectsInArc` for the current objects in
the arc. That helper performs a radius broad phase around the live arc origin
using `arc.length`, then applies the same live-position/live-footprint
`IsObjectInArc` test. For each hostile survivor it computes:

```text
distanceScore = min(GetObjectDistance(attacker, candidate) / arc.length, 1)

angle = GetObjectAngleRad(attacker, candidate)
if angle > pi:
    angle = 2*pi - angle
angleScore = min(angle / arc.halfAngle, 1)

score = distanceScore + angleScore
```

The chosen object is the first one whose score is strictly lower than the
current best (initially `1000`). Equal scores do not replace the earlier
candidate. Candidate position, attacker transform, validity, distance, and
angle are all continuation-time reads. Footprints determine broad-phase/arc
contact; the ranking formula itself does not add either footprint explicitly.

## Exact turn-in-place command

For `faceTarget=true` with a valid external target, `sub_9E0AE0` calls
`sub_A15610(actor, targetPosition)` and immediately calls
`sub_A20210(actorID)`. The Lua ability callback and its attack animation begin
after that publication.

`sub_A15610` clears the previous goal state and constructs:

```text
goalFlags      = 0x42
goalPosition   = current actor position
facing         = normalize(targetPosition - current actor position)
targetPosition = the faced continuation/activation target point
targetObjectID = 0
```

`sub_A20210` copies the relevant locomotion fields into the fixed 80-byte
`ObjectPlayerMove` body and sends logical type `0x12`. With the one-byte GMS
opcode, the exact useful layout is:

| Packet bytes | Size | Meaning in this turn command |
| ---: | ---: | --- |
| `0` | 1 | `0x91` |
| `1..4` | 4 | actor object ID |
| `5..8` | 4 | flags `0x00000042` |
| `9..20` | 12 | goal XYZ = current actor position |
| `21..32` | 12 | normalized facing XYZ toward target |
| `33..44` | 12 | external linear velocity, reset to the locomotion default by the turn constructor |
| `45..56` | 12 | current/preserved external force |
| `57..60` | 4 | current/preserved allowed stop distance |
| `61..64` | 4 | current/preserved desired stop distance |
| `65..76` | 12 | target position XYZ |
| `77..80` | 4 | target object ID = zero |

Thus facing is present twice in the command's semantics: directly as the
normalized facing vector and indirectly through the target position from which
it was derived. This packet is an active turn goal (`0x42`), not the plain
stopped goal (`0x20`). The sender has one call to the GMS publication routine
and returns; it does not invoke the fixed unreliable-goal sender, so appending
`0x95` would invent a second transition absent from this native path.

The activation order for both abilities is therefore:

```text
admit and allocate ability
  -> replace chase/strafe with flags-0x42 turn-in-place state
  -> send one 0x91 containing position, facing, and target position
  -> start shared melee Lua
  -> send/set attack animation
  -> wait to t=0.430000007
  -> rebuild arc from live attacker transform
  -> select using retained-target exception plus live arc state
  -> TakeDamage with retained attack-start direction
```

TailZap releases at `1.200000048s`; TutorialPoisonMelee releases at `2s`.
Their damage and hit-effect recipes differ, but their `0.43s` selection and
activation locomotion boundary do not.

## Evidence limits

- `TargetInRangeAtStart` is an owner-supplied byte retained in the ability
  runtime. The build-103 client-shaped activation caller passes false because
  the authoritative NPC owner is absent. A parity server admitting an NPC
  melee attack from the footprint-expanded envelope should compute and latch
  the boolean; the template behavior for both boolean values is exact.
- The exact `nTargetUtils` call topology, arguments, strict comparisons, and
  ranking are fixed by the retained scripts. The build-103 client does not
  retain the authoritative server implementations of the spatial-query and
  circle/arc primitives. Their named geometry contract is recoverable; a
  build-103 server binary or retail boundary capture would be needed to turn
  primitive-internal floating-point edge behavior into a byte-for-byte test.
- No retail trace is required to decide the activation message count: the
  build-103 native activation path directly constructs and sends the single
  flags-`0x42` `0x91`, and contains no `0x95` or `0x94` publication call.
