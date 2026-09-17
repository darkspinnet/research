# Tutorial Poison Cloud projectile wire

This note closes the packet-facing gap for the ordinary, non-homing
`TutorialPoisonCloud` projectile in build 103. The required application-message
sequence at the authored `0.17s` shot point is:

1. generic `ObjectCreate` (`0x8c`), 96 bytes;
2. attached-trail `ServerEvent` (`0x9b`), 16 bytes;
3. `LocomotionDataUpdate` (`0x94`), normally 72 bytes, or 85 bytes when a
   predicted terrain collision is present.

The create is generic: it contains no combatant, attribute, agent-blackboard,
owner, or homing state. The locomotion update is the reflected reliable
application-message family, not the fixed `0x95` goal correction used for
walking actors.

## Evidence and confidence

The result combines three mutually checking sources:

- build-103 `cGameObjectCreateData`, `sporelabsObject`, `cLocomotionData`, and
  `cProjectileParams` reflection registrations and the `0x8c`/`0x94`
  receivers in `bin/darkspinner/GameBin/Game.c`;
- build-103 `WaitForProjectile` at `sub_A0FBE0`, ordinary launch initialization
  at `sub_A2E320`, and the post-construction reflection baselines seeded by
  `sub_9D1810`/`sub_9D1790`;
- the Resurrection Capsule sender/serializer implementation at commit
  `2851559d098a447ccdabd00353f57bab7fd00997`, especially
  [`SendObjectCreate` and `SendLocomotionDataUpdate`](https://github.com/vitor251093/recap_server/blob/2851559d098a447ccdabd00353f57bab7fd00997/game_server/source/raknet/server.cpp),
  [`Object::WriteReflection`](https://github.com/vitor251093/recap_server/blob/2851559d098a447ccdabd00353f57bab7fd00997/game_server/source/game/object.cpp),
  and [`Locomotion::WriteReflection`](https://github.com/vitor251093/recap_server/blob/2851559d098a447ccdabd00353f57bab7fd00997/game_server/source/game/locomotion.cpp).

The field layout and byte recipes below are exact client-consumable build-103
application packets. No retained EA-server capture independently proves the
outer RakNet reliability enum or whether EA batched these application messages
in the same datagram. Do not turn the reference server's common
`UNRELIABLE_WITH_ACK_RECEIPT` send policy into a retail claim.

## Values used by TutorialPoisonCloud

| Name | Wire value |
| --- | --- |
| Projectile noun | `Ability_Fireball.Noun` = `0xf17a951f` |
| Asset ID | `0` |
| Scale | `1.0` |
| Team | caster's team, repeated in both create structures |
| Collision flag | `false`; collision volumes remain authoritative simulator data |
| Player-controlled | `false` |
| Owner ID | absent for the enemy-target Poison Cloud branch |
| Speed | `6.0` |
| Acceleration | `0.0` |
| Range | `12.0` for the ordinary tutorial case |
| Spin rate | `0.0` |
| Projectile flags | `0`; therefore ordinary rather than homing motion |
| Remaining projectile parameters | zero/default |
| Reflected last update | `1000` (`0x000003e8`) |

`RangeIncrease` is the only fixture-dependent change to the base parameter
block. The recovered Lua and native boundary applies it twice, so field
`mRange` is `12 * (1 + RangeIncrease)^2`. The tutorial fixture uses zero and
therefore sends `12.0`.

## Generic projectile ObjectCreate

Let:

- `I` be the little-endian `uint32` projectile object ID;
- `P` be the launch position as three little-endian `float32` values;
- `R` be the three create rotations as little-endian `float32` values;
- `T` be the one-byte caster team;
- `D` be the normalized launch direction;
- `V = D * 6`, as three little-endian `float32` values;
- `Q` be the projectile orientation quaternion, four little-endian `float32`
  values in the registered build-103 order.

The complete application packet is:

```text
8c
I
ff 03
1f 95 7a f1
P
R
00 00 00 00 00 00 00 00
00 00 80 3f
T
00
00
00 T
04 V
06 P
07 Q
ff
```

The first `ff 03` is the ten-field `cGameObjectCreateData` mask `0x03ff`, not a
reflection terminator. Its 43 bytes are all fixed and ordered as noun,
position, rotation X/Y/Z, asset ID, scale, team, collision, and
player-controlled.

The following `sporelabsObject` reflection has more than sixteen fields and is
therefore an indexed list terminated by `0xff`:

| Index | Size | Value | Why present |
| --- | ---: | --- | --- |
| `0` | 1 | `T` | The projectile copies the caster's team after construction. |
| `4` | 12 | `D * 6` | `WaitForProjectile` changes linear velocity from its zero baseline. |
| `6` | 12 | `P` | Object creation establishes the launch position. |
| `7` | 16 | `Q` | `WaitForProjectile` recomputes orientation from direction and the local surface frame. |

The packet is exactly 96 bytes: one opcode, 49 bytes through the fixed create
prefix, and a 46-byte indexed object tail.

Do **not** add the following fields:

- field `5` angular velocity: the native wrapper writes
  `direction * acceleration`, but Poison Cloud acceleration is zero and the
  reflection comparer sees no change from the construction baseline;
- field `18` owner: the enemy-target creation branch does not set it;
- field `19` movement type: only the unused homing branch changes it to `3`;
- fields `16`/`17` visibility/collision: their runtime values remain the noun
  defaults already represented by creation data.

The create rotations and quaternion intentionally remain transform inputs in
the recipe. They are not interchangeable encodings: the fixed prefix carries
three floats while object field `7` carries four. An encoder must derive both
from the same authoritative launch orientation, not copy quaternion bytes into
the rotation slots.

## Reliable locomotion snapshot

`LocomotionDataUpdate` is application opcode `0x94`. Its reflected type has 18
fields, so it also uses ascending one-byte field indices followed by `0xff`.
For unobstructed ordinary Poison Cloud travel the exact packet is:

```text
94
I
03
00 00 c0 40                         # +0x00 speed = 6.0
00 00 00 00                         # +0x04 acceleration = 0.0
00 00 00 00                         # +0x08 jinkInfo = 0
00 00 40 41                         # +0x0c range = 12.0
00 00 00 00                         # +0x10 spinRate = 0.0
D                                     # +0x14 direction, 3 float32
00                                    # +0x20 projectileFlags = 0
00 00 00                              # +0x21 alignment/padding
00 00 00 00                         # +0x24 homingDelay = 0.0
00 00 00 00                         # +0x28 turnRate = 0.0
00 00 00 00                         # +0x2c turnAcceleration = 0.0
00                                    # +0x30 piercing = false
00                                    # +0x31 ignoreGroundCollide = false
00                                    # +0x32 ignoreCreatureCollide = false
00                                    # +0x33 alignment/padding
00 00 00 00                         # +0x34 eccentricity = 0.0
00 00 00 00                         # +0x38 combatantSweepHeight = 0.0
11 e8 03 00 00
ff
```

Top-level field `3` is not a nested reflection mask. It is the fixed 60-byte
in-memory-layout serialization of `cProjectileParams`; the explicit padding at
offsets `0x21-0x23` and `0x33` is part of the wire value. Top-level field `17`
then carries signed `int32(1000)`. Including the opcode, this unobstructed form
is exactly 72 bytes.

Ordinary initializer `sub_A2E320` ray-tests `D * range`. When that test predicts
a terrain collision, insert field `14` before field `17`:

```text
0e <predicted collision x, y, z as 3 float32>
11 e8 03 00 00
ff
```

That form is 85 bytes. Creature collision does not populate field `14`; it is
the predicted geometry-collision point. A deterministic direct-hit fixture
with an unobstructed launch omits field `14`. A wall/terrain fixture must send
it and use the same point in authoritative flight termination.

Fields `12` target ID, `13` target position, and `15` initial direction are
absent in ordinary Poison Cloud. They belong to other locomotion modes or the
homing branch. Direction is already inside top-level field `3` at projectile
parameter offset `0x14`.

## Ordering and lifecycle

The projectile is constructed and its reflection baseline is seeded before
Lua calls `WaitForProjectile`. The Lua/native call then changes team,
velocity, orientation, projectile parameters, and reflected-last-update before
yielding. The next replication pass consequently folds the object-side changes
into the first `ObjectCreate`, then emits the locomotion component snapshot.

The trail event may be serialized between those two messages because Lua calls
`AddEffect` before `WaitForProjectile`:

```text
9b 01 <one-based slot> 04 01 06 f6 c5 65 c6 07 I ff
```

That packet is 16 bytes. Poison Cloud later deletes the projectile directly;
it does not need a separate trail-stop packet because the object deletion owns
effect-slot cleanup.

The implementation golden should therefore assert this application ordering:

```text
0x8c ObjectCreate
0x9b attached trail
0x94 reliable locomotion snapshot
... impact/damage ...
ObjectDelete
```

Do not substitute `0x95`. Its receiver copies only one goal vector into full
and partial navigation goals and cannot initialize projectile parameters.

## Encoder acceptance checklist

- `ObjectCreate` is 96 bytes and ends with object field `7` plus `0xff`.
- The noun bytes are `1f 95 7a f1`.
- Team appears in fixed create field `7` and object field `0`.
- Linear velocity appears once in object field `4`; zero angular velocity is
  omitted.
- Position appears in fixed create field `1` and object field `6`.
- `0x94` field `3` occupies exactly 60 bytes, including four padding bytes.
- `0x94` field `17` is `11 e8 03 00 00`.
- An unobstructed shot is 72 bytes; a predicted-terrain shot with field `14`
  is 85 bytes.
- No owner, target ID, target position, initial-direction top-level field,
  homing movement type, combatant update, or `0x95` goal packet is present at
  launch.
