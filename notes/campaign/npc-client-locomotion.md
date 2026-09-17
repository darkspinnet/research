# Build-103 NPC client locomotion contract

## Result

`ObjectPlayerMove` (`0x91`) is valid for non-player actors. Its build-103
receiver is not player-class gated: it resolves any object, requires that
object's `cLocomotionData` component, and applies the complete locomotion
command unless the object is the locally controlled actor and the local-input
override is inactive. Native NPC `nThread.MoveTowardObject` installs the same
component goal and publishes it through the authority-only `0x91` sender.
Ordinary NPC pursuit does not require `LocomotionUpdate` (`0x94`) or another
hidden AI/goal opcode.

The client-visible walking transition is:

```text
0x8c ObjectCreate and dependent component/target state
  -> once per new/replaced walking goal:
       0x91 complete command, flags nonzero
       0x95 matching goal/partial-goal
  -> while the same goal remains active:
       optional 0x95 corrections, without replaying 0x91 every simulation step
  -> arrival/attack:
       0x91 stop or target-facing command (normally flags 0x20 or 0x42)
       then animation/ability consequences
```

The v0.7.17 scheduled pursuit path violated that contract after a valid
generation start. Each material target movement replayed a complete `0x91`
followed by `0x95`, and a one-second refresh additionally inserted `0x8d` at
the server-simulated current position before another complete command. Each
`0x91` replay overwrites the full command fields and clears the receiver's
transient state. The refresh also introduced an intermediate authoritative
position that was neither the active target goal nor a recovered retail
stuck-correction policy.

The SS-000003 v0.7.17 capture confirms the failure in the real client: NPC
object `78` received 70 complete `0x91` commands, 66 `0x95` updates, and 25
`0x8d` writes in 30 seconds. During the final ten seconds its rendered root
remained fixed for five seconds and then moved about 5.5 units per second while
the authoritative server root finished 28.244 units ahead. No packet loss,
malformed input, pending output, or projectile drift explains that result.

The implementation fix is to give every observing subscriber, including the
source, exactly one initial/replacement `0x91` followed by `0x95`; send only
coalesced corrections while that goal generation remains active; and send one
explicit stop/turn command at arrival. `0x94` must remain reserved for reflected
component snapshots and special initializers such as lob/projectile motion.

## Evidence and confidence labels

- **Proven native** means a build-103 receiver, constructor, call site, or
  component registration in `bin/game/GameBin/Game.c`.
- **Proven content** means the build-103 noun/profile or Lua record in the
  read-only runtime `content.db`.
- **Live-proven** means an unmodified build-103 client visibly accepted the
  transition, as recorded by existing tutorial notes and retained traces.
- **Current implementation** means the inspected Go source or the current
  server's transport wrapper, not an assertion about the missing retail server.
- **Conservative fallback** means safe policy where the original retail sender
  or cadence is absent.

Artifacts inspected:

- client version: `5.3.0.103`;
- canonical decompiler:
  `bin/game/GameBin/Game.c`, SHA-256
  `1fac9cda954e076446889376d5e3e89859ce058b75c2d50739d4aedfd662b1eb`;
- runtime database:
  `bin/darkspinner/darkspin/cache/content.db`, SHA-256
  `2b3be07b2f259eb0a533f4a09a02f6018957c8f92aa4863846904af3b0352ef8`
  at inspection time;
- latest read-only client trace:
  `bin/darkspinner/darkspin/logs/traces/game.jsonl`, last modified
  `2026-07-26 03:26:05 PDT`, SHA-256
  `04d900f9345565338cf0c9b7ba074b333dbd7807d7f92c46b0a067a9b45e6eb3`;
- companion server log:
  `bin/darkspinner/darkspin/logs/darkspinner.log`.
- v0.7.17 diagnostic set:
  `bin/game/logs/ss-20260831T173552`, especially SS-000003 captured at
  `2026-08-31T17:30:54Z`.

Existing focused evidence was cross-checked in:

- `notes/tutorial/horde-ai.md`;
- `notes/tutorial/history.md`;
- `notes/tutorial/overview.md`;
- `notes/campaign/1-1/enemy-pursuit.md`;
- `notes/campaign/1-1/enemy-runtime.md`;
- `notes/campaign/1-1/enemy-speed.md`;
- `notes/campaign/player-pursuit.md`;
- `notes/client/packet.md`.

All addresses below are image virtual addresses in the canonical build-103
decompiler.

## Object and component prerequisites

### Creation

`ObjectCreate` receiver `sub_53A760` (`0x0053A760`) first reads the object ID,
decodes `cGameObjectCreateData`, constructs the noun-backed runtime object, and
only then decodes the base `sporelabsObject` reflection. Later component
receivers resolve that existing object by ID. Therefore `0x8c` must precede
`0x91`, `0x94`, `0x95`, `0x99`, animation, modifier, and combat messages for
the actor.

`ObjectCreate` cannot carry `cLocomotionData`. Its optional prefix has ten
fields and its terminated tail is the 23-field base-object reflection. The
locomotion component is attached by noun construction. Build-103 campaign
creature nouns author locomotion/profile `npcCreature`, `builtins!sphere`, and
`DefaultPhysics.prop`; native construction therefore gives them a
`cLocomotionData` pointer at object offset `+0x298` (`+664`). Construction
starts stopped, without a target or walking goal.

The current `EnemyObjectCreateMessage` is an 81-byte application packet
including opcode:

```text
8c
<objectID:u32>
ff 03                         create fields 0..9 present
<noun:u32>
<position:vec3>
<rotationEuler:vec3>
00 00 00 00 00 00 00 00     asset ID
00 00 80 3f                  generic scale 1.0
00 <collidable:bool> 00      team, collision, playerControlled
06 <position:vec3>           base-object dirty field 6
07 00..00 00 00 80 3f       field 7 quaternion (0,0,0,1)
ff
```

This is receiver-valid and content-compatible for the locomotion component.
It is not proof of the retail hostile team, owner, movement-type, source-marker,
or complete spawn dirty set. A separate `0x94` is not required merely to make
an `npcCreature` noun walk.

### Target and AI state

The native target-follow constructor retains a nonzero target object ID in the
locomotion component. Client locomotion can resolve that target only after it
exists. For darkspin's explicitly assigned hostile target, publish the
`0x99 AgentBlackboardUpdate` before the target-bearing `0x91`; this is also the
established target-consumer ordering for first-aggro and attack presentation.
There is no separate "enable AI" application message.

## `0x91 ObjectPlayerMove`: exact contract

### Receiver and non-player validity

Receiver `sub_5393D0` (`0x005393D0`) reads exactly 80 payload bytes, resolves
the object, and checks its `+664` locomotion component. The only ownership gate
is:

```c
object && object->locomotion &&
    (localControlledObjectID != object->id || localInputOverride())
```

It does not test player class, player-controlled state, combatant type, or
blackboard presence. A non-player noun with `cLocomotionData` takes the normal
branch.

Native `nThread.MoveTowardObject` binding `sub_A08FD0` (`0x00A08FD0`) installs
target-follow state through `sub_A149C0` (`0x00A149C0`) and calls
`sub_A20210` (`0x00A20210`). That sender is guarded by `!sub_9BCF80()`, the
authority/server role, snapshots the same 80 bytes into logical message 18,
and sends it outward. This is direct native proof that the so-called
`ObjectPlayerMove` family is the NPC locomotion command too.

### Wire layout and component writes

All multibyte fields are little-endian. Offsets include the opcode:

| Packet offset | Size | Field | Receiver component destination |
| ---: | ---: | --- | --- |
| `0` | 1 | `0x91` | |
| `1` | 4 | object ID | object lookup |
| `5` | 4 | goal flags | `+0x144` |
| `9` | 12 | goal XYZ | `+0x148` |
| `21` | 12 | facing XYZ | `+0x178` |
| `33` | 12 | external linear velocity | `+0x184` |
| `45` | 12 | external force | `+0x190` |
| `57` | 4 | allowed stop distance | `+0x19c` |
| `61` | 4 | desired stop distance | `+0x1a0` |
| `65` | 12 | target-position XYZ | `+0x1ac` |
| `77` | 4 | target object ID | `+0x090` |

The receiver also clears transient component float/byte `+0x1a4/+0x1a8`.

### Goal flags

Proven native constructors establish:

- `0x00000001`: ordinary active point/target-follow goal;
- `0x00000020`: stopped locomotion (`sub_A14D70`, `0x00A14D70`);
- `0x00000040`: target/facing option, combined with another active bit;
- `0x00000042`: live-proven target-facing attack turn;
- `0x00000080`: the second target-follow option; player pursuit uses it, while
  packaged NPC `MoveToRange` passes false and stays at `0x01`;
- `0x00000100`, `0x00000400`, `0x00000800`: other native operation modes,
  not ordinary NPC pursuit.

For packaged NPC `nBehavior_MoveToRange`, `sub_A149C0` sets flags `0x01`,
copies the target's current XYZ into goal position, retains its object ID,
clears the separate target-position vector, and sets both stop distances to:

```text
actor footprint + target footprint + adjusted ability range
```

`GetAdjustedRange` is `range - 1` above 10, otherwise `range * 0.8`. Darkspin's
`PlanActionWithProfile` already replaces `profile.Range` with the equivalent
footprint-expanded `NPCStopDistance`, so `npcraknet.Pursuit`'s use of
`plan.Profile.Range` is intentional.

## `0x95 LocomotionUnreliable`: exact contract

Receiver `sub_539810` (`0x00539810`) reads 16 payload bytes, resolves any
object and its `+664` locomotion component, then writes the one vector twice:

```text
component +0x148 = vector   mGoalPosition
component +0x154 = vector   mPartialGoalPosition
```

The exact application bytes are:

```text
95 <objectID:u32> <goal-or-partial-goal:vec3>
```

It does not write flags, facing, speed, stop distance, target ID, target
position, current object transform, reached state, or component type. Therefore:

- `0x95` alone cannot change a freshly constructed/stopped NPC into an active
  walker;
- `0x95` is not an authoritative transform snapshot despite darkspin currently
  advancing an authoritative position on the same cadence;
- `0x95` is valid after `0x91` to refine a goal/partial goal;
- `0x95` is not an arrival acknowledgement and the client has no recovered
  NPC-arrival sender.

The method name says unreliable because that is the logical component-update
family. The retained client receiver and current trace do not recover the
missing retail server's outer RakNet reliability policy.

## `0x94 LocomotionUpdate`: exact boundary

Receiver `sub_539900` (`0x00539900`) reads the object ID, resolves
`object + 664`, clears `+0x1a4/+0x1a8`, and reflection-decodes type hash
`0xB6F447EF` (`cLocomotionData`). Each sparse field is encoded as one byte
field ID followed by its value; `0xff` terminates the reflection.

| Dirty ID | Field | Wire value |
| ---: | --- | --- |
| `0` | `lobStartTime` | `u64` |
| `1` | `lobPrevSpeedModifier` | `f32` |
| `2` | `lobParams` | 84-byte `cLobParams` |
| `3` | `mProjectileParams` | 60-byte `cProjectileParams` |
| `4` | `mGoalFlags` | `u32` |
| `5` | `mGoalPosition` | `vec3` |
| `6` | `mPartialGoalPosition` | `vec3` |
| `7` | `mFacing` | `vec3` |
| `8` | `mExternalLinearVelocity` | `vec3` |
| `9` | `mExternalForce` | `vec3` |
| `10` | `mAllowedStopDistance` | `f32` |
| `11` | `mDesiredStopDistance` | `f32` |
| `12` | `mTargetObjectId` | `u32` |
| `13` | `mTargetPosition` | `vec3` |
| `14` | `mExpectedGeoCollision` | `vec3` |
| `15` | `mInitialDirection` | `vec3` |
| `16` | `mOffset` | `vec3` |
| `17` | `reflectedLastUpdate` | `i32` |

`0x94` can represent a complete navigation snapshot if an authoritative sender
selects fields 4-13, but receiver capability is not sender policy. The
build-103 native path has a direct fixed-message constructor for `0x91`; no
equivalent ordinary-walk constructor for logical 21 was found. Live tutorial
NPC walking succeeds with `0x91` followed by `0x95`. Therefore adding an
invented `0x94` startup packet is neither necessary nor the right fix for the
current frozen model.

Use `0x94` where separate evidence proves a reflected initializer: fields
0/1/2 for lob motion and field 3 plus its applicable collision/clock fields for
projectiles. `0x95` cannot substitute for those modes.

## Creation, update, and arrival ordering

### Proven dependency order

For a newly visible hostile NPC:

```text
0x8c create
-> optional 0x97 current HP/mana and 0x96 attributes
-> base object visibility/collision updates if needed
-> 0x99 nonzero target state
-> 0x91 active locomotion command
-> 0x95 same initial goal/partial goal
-> first-aggro/ordinary action presentation as authored
```

`0x97` versus `0x96` relative ordering and equal-default publication are not
client prerequisites. Create-before-component, target-before-target-consumer,
and `0x91`-before-`0x95` are prerequisites.

### Cadence

The native client mover consumes variable frame delta from `sub_9D2960`
(`0x009D2960`) and updates object pose in `sub_9EA1A0` (`0x009EA1A0`).
Movement/rotation changes invoke a same-pass callback. No fixed retail-server
simulation or replication interval is recoverable from the client.

Live tutorial evidence proves one `0x91`/`0x95` pair is sufficient to start an
NPC walk. Later `0x95` corrections are valid when the partial goal changes.
The current 50 ms server integrator is explicitly a local fallback. The
conservative publication policy is:

- send `0x91` only on locomotion generation start or semantic replacement;
- coalesce `0x95` to at most the simulation cadence, and only when the
  goal/partial-goal changed materially;
- do not emit both target goal and authoritative current position as competing
  `0x95` meanings at one deadline;
- if exact authoritative pose correction is required, use a proven transform
  family (`0x8d`, `0x90`, or `0x92`) under a separately recovered correction
  policy rather than relabeling current XYZ as `0x95`.

### Stop, turn, and arrival

Native `nLocomotion.Stop` installs flags `0x20`; a target-facing attack installs
flags `0x42`, current actor position as goal, normalized facing, target
position, and the applicable target identity. The target-facing native sender
does not append `0x95`.

On server-authoritative arrival:

1. commit the final NPC position in the NPC session;
2. send one flags-`0x20` stop at that position, or the attack's flags-`0x42`
   turn command;
3. do not send another active flags-`0x01` command after it for the retired
   pursuit generation;
4. start animation/ability timing only after the stop/turn packet in the same
   ordered stream.

Arrival remains server-owned. Native `MoveToRange` yields until the strict live
range predicate succeeds; `Mover::ReachedGoal` is replay/diagnostic state, not
a GMS client-to-server completion message.

## Current transport versus retail transport

All current darkspin handler responses, poll output, and scheduled output call
`encodeConnectedPackets(..., ReliableOrdered, payload)`. Non-split gameplay
packets use ordering channel `1`; split packets also use channel `1`. Thus the
current effective transport for `0x8c`, `0x91`, `0x94`, and `0x95` is
**RakNet reliable-ordered, channel 1**, one application payload per connected
encapsulation.

This is current implementation fact, not retail proof. The build-103 client
does prove reliable-ordered channel 1 for its own gameplay request sender, but
the original authoritative NPC dirty-flush owner is absent. The exact retail
outer reliability for `0x95`, batching, resend policy, and channel remain
unresolved. Preserve current reliable-ordered channel 1 for the first
implementation fix because it guarantees create/target/goal dependency order.
Changing only `0x95` to unreliable-sequenced should be a later trace-backed
transport change, not part of the locomotion correctness fix.

## Tutorial/retail comparison with the current zone path

### Visibly moving tutorial path

Existing clean build-103 runs established:

- enemy `0x8c` plus state was already present;
- after the authored first-aggro pause, the server sent one flags-`0x01`
  `0x91`, immediately followed by fixed 16-byte-body `0x95` with the same goal;
- both tutorial infectors visibly walked from their AI markers and settled
  around the hero;
- `0x91` alone and repeated base-object position updates had previously failed.

This proves component admission plus initial partial-goal initialization, not a
need for a special player relationship.

### Native retail-shaped NPC path

`nBehavior_MoveToRange` content (`lua_chunk` 826, resource 14403) calls
`nThread.MoveTowardObject`. Native `sub_A08FD0 -> sub_A149C0 -> sub_A20210`
installs and sends one target-follow command, then leaves its Lua continuation
yielded. The simulation mover advances each frame and the continuation tests
live actor/target positions. It does not resend `0x91` on every predicate poll.

### Current zone path

The source action path returns `npcraknet.Pursuit` directly, while the semantic
`ActionEventPursuit` projects the equivalent pair to other subscribers. Each
subscriber therefore receives the generation-start `0x91` followed by `0x95`.
`campaignNPCPursuitRuntime.produceStep` advances the authoritative server
position every 50 ms and coalesces a moving target change to at most one update
per 250 ms after the target has moved 0.75 units.

Before the SS-000003 correction, each coalesced change was incorrectly encoded
as a replacement `0x91` plus `0x95`, and every second also prepended an `0x8d`
server-position write. The source received that batch directly and remote
subscribers received the same replacement through
`ActionEventPursuitRedirect`. The corrected path emits one goal-only `0x95` to
both recipient classes and emits nothing when the goal has not materially
changed.

The 0.6.15 Cannonator capture also exposed a separate hybrid-profile boundary:
`ZelemBasicHybrid` selected its melee action whenever centers were within eight
units, even though its footprint-adjusted melee range was only 2.3 units. The
actor consequently restarted an unreachable melee pursuit every 100 ms from an
unchanged position instead of using its ranged action. Hybrid melee selection
now uses the same surface-distance calculation as its action planner, keeping
Cannonators ranged until they are genuinely inside melee reach.

### Latest trace evidence

The seven v0.7.17 snapshots in `bin/game/logs/ss-20260831T173552` contain no
Orcus instance, but they reproduce the same server/client position split across
ordinary Scaldron NPCs. SS-000003 is the cleanest active-pursuit sample:

- object `78` (`ZelemSpecialThree`) ends at server position
  `(-429.746, 316.409, 40.655)` while the client renders it at
  `(-450.058, 296.793, 40.086)`, a 28.244-unit separation;
- the final ten seconds contain 15 `0x91`, 14 `0x95`, and four `0x8d` packets
  for that object, while after-apply locomotion flags are predominantly active
  flags `0x01`;
- the rendered root is stationary at approximately
  `(-478.430, 296.460, 40.090)` for five seconds, then advances at the authored
  5.5-unit speed without closing the already-created server lead;
- the snapshot reports no dropped or malformed client lines, no pending server
  output, and aligned projectile positions.

This directly corroborates the complete-command replay risk identified from
the receiver writes. It also disproves the v0.7.16 assumption that a periodic
`0x8d` root refresh bounds drift: four such refreshes occur during the final
ten seconds and the separation remains 28.244 units.

The older retained trace creates campaign NPC object IDs `32..38` at
`time_ms=200941625` as 80-byte `0x8c` bodies. It shows two materially different
movement streams:

- object `33` at `time_ms=201279015` receives one flags-`1` `0x91` and matching
  `0x95`, then receives changing `0x95` positions at roughly 94-109 ms until
  flags-`0x42` attack turns begin at `201282046`. This is the expected
  start/correct/turn lifecycle.
- object `32` from `time_ms=200995625` receives the same flags-`1`, same-goal
  `0x91` repeatedly at roughly 100 ms, frequently duplicated at the same
  deadline. Multiple `0x95` values for the same object are interleaved. This
  is not the native one-command continuation and is direct evidence of
  overlapping or repeated publishers in the runtime binary used for that run.

The trace records application payloads after client receipt, not the outer
RakNet encapsulation, and records only a 16-byte prefix for the 80-byte bodies.
It is sufficient to prove opcode, object ID, flags, goal prefix, body size,
ordering, duplication, and cadence; it cannot prove every trailing `0x91`
field or retail transport policy.

## Concrete Go changes

The shared scheduled-pursuit correction now makes these scoped changes:

1. Keep `npcraknet.Pursuit` and the existing direct/projection split as the
   generation-start encoder, preserving `0x91` followed by `0x95` for every
   subscriber.
2. Encode a material moving-target change as one `0x95` goal update. Projectile
   updates preserve the existing range-adjusted pursuit goal.
3. Remove the one-second active-pursuit `0x8d` refresh and unchanged-goal
   command replay. Arrival and action families retain their established
   authoritative stop/attack transition.
4. Apply the same goal-only projection to remote co-op subscribers so source
   and observers cannot diverge by adapter path.
5. Leave the client, shipped assets, `content.db`, ordinary-walk `0x94`, and
   reliable-ordered transport unchanged.

Suggested focused tests:

- a solo source subscriber receives exactly `[0x91, 0x95]` on first pursuit;
- a second subscriber receives the same pair in the same semantic order;
- the source is not excluded unless equivalent start packets are returned
  directly in that transaction;
- ten 100 ms authority steps under an unchanged target emit no additional
  `0x91`;
- a changed partial goal emits one `0x95`;
- arrival emits one `0x42`/`0x20` command after the last correction and no
  later active command from the old generation;
- `0x95`-only startup is rejected by the publication state;
- packet goldens assert `0x91` is 81 bytes and `0x95` is 17 bytes including
  opcode, with object ID and goal equality;
- a trace-style semantic test covers create, blackboard target, pursuit start,
  corrections, arrival turn, and attack in order.

## Conservative fallbacks and unresolved retail details

Safe fallbacks:

- retain the current 50 ms authority integrator until a retail simulation
  cadence is recovered;
- use reliable-ordered channel 1 for all locomotion packets during the fix;
- send an initial `0x95` immediately after every start/replacement `0x91`;
- coalesce later unchanged corrections rather than flooding.

Still unresolved:

- original retail server simulation, repath, dirty-flush, and correction
  cadence;
- retail `0x95` RakNet reliability, ordering channel, and batching;
- whether retail periodically sent sparse navigation `0x94` snapshots as
  recovery checkpoints;
- late-join locomotion snapshot policy;
- exact stuck recovery, path-corner smoothing, and pose correction threshold;
- retail hostile team/owner/movement-type/source-marker fields in `0x8c`.

These gaps do not block the fix. The client contract, non-player validity of
`0x91`, source-exclusion bug, and danger of replaying the complete command each
step are proven independently of the missing retail policies.
