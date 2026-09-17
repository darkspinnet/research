# Campaign player pursuit and movement admission

## Result

Build `5.3.0.103` has three different operations which must remain distinct:

1. A plain ground move is client input action type `3` in
   `ActionCommandMsgs` (`0x9c`). It carries a movement tail and is not an
   ability.
2. A ground-fired character ability is action type `7`, target object `0`,
   with its cursor and target-position fields still populated. It is a valid
   ability request, not an encoded move.
3. A targeted `shouldPursue` ability which fails admission only because it is
   out of range retains the original action in the authoritative simulation,
   starts `nBehavior_MoveToRange`, follows the live target to a footprint-
   expanded inset, and retries the ability once the authoritative mover is in
   range.

The latest 1-1 run reached case 2 for empty-ground clicks: the server log shows
target-zero index-zero basics with equal cursor and target positions, while the
client trace contains no corresponding player movement command. This is
possible because the client input resolver does not define every empty-ground
click as movement. When a pending/basic action is selected, the action path at
`0x004D5F50` constructs an ability request through `0x004D53C0`; the separate
movement builder at `0x004DEF80` is never reached. Repeated/held basic state can
therefore produce fresh targetless ability actions on empty ground.

That packet is not sufficient to reconstruct a movement request. Build 103
uses the same target-zero ability shape for real cursor-fired and ground-fired
attacks. Movement must be admitted only from action type `3` (and stopped by
type `4`), while a type-`7` request must be evaluated using the equipped
ability's target/cursor semantics. A zero target is not a movement discriminator.

For an out-of-range targeted basic, the client may predict controlled-object
locomotion, but the server-side simulation owns arrival and attack commitment.
The client sends no reached-goal acknowledgement. `ObjectPlayerMove` (`0x91`),
`LocomotionUpdate` (`0x94`), and `LocomotionUnreliable` (`0x95`) are
authority-to-client state/presentation in the inspected paths, not permission
for the server to commit the attack.

## Evidence provenance

- Client version: `bin/game/GameBin/version_bin.txt`, `5.3.0.103`.
- Canonical decompiler output:
  `bin/game/GameBin/Game.c`, SHA-256
  `1fac9cda954e076446889376d5e3e89859ce058b75c2d50739d4aedfd662b1eb`.
- Canonical IDB: `bin/game/GameBin/Game.idb`, SHA-256
  `3ba215056767e7cff2f7575738dae9f3ad731e8013ee2e1471f3939d6b97422d`.
- Runtime content database:
  `bin/darkspinner/darkspin/cache/content.db`, SHA-256
  `13a6c89f5642fa588d273ea872f4a3104ce5f13c948fb6d44c8cd8212f772423`.
- Packaged behavior chunk `826`, server-data resource `14403`, source
  `behaviors/0xD3200CED.lua`, retained at
  `bin/game/logs/1-1-enemy-pursuit/behaviors/chunk-826.luac`, SHA-256
  `a831735975eee01bce296bee2e1c67e532d56495fec674f51e1a386a7c845d6d`.
- Latest trace:
  `bin/darkspinner/darkspin/logs/traces/game.jsonl`, last modified
  `2026-07-22 10:31:01 PDT`, SHA-256
  `68010ae043da03440195b74cc76642a9b598c3cc5b38c41d42dad085e9dc48a4`.
  It contains 66 received `0x91`, 31 received `0x94`, and 46 received
  `0x95` application messages. The trace records received application payload
  prefixes but only socket digests for sends, so the companion
  `bin/darkspinner/darkspin/logs/darkspinner.log` is the evidence that no
  type-`3` command reached the server during the failed ground clicks.

Addresses below are image virtual addresses in the canonical build-103 image.

## Client action construction

### Common 40-byte action header

`sub_4DED90` (`0x004DED90`) builds the shared header. `sub_A17440`
(`0x00A17440`) fixes the type-`3`/`4` tail at 24 bytes and the type-`7`/`8`
tail at 44 bytes.

| Body offset | Size | Meaning | Confidence |
| ---: | ---: | --- | --- |
| `0` | 1 | action type (`3` move, `4` stop, `7` character ability, `8` squad ability, `10` cancel) | exact |
| `1` | 1 | action ID/sync byte, echoed by action response/cancel handling | strong; historical decoder calls it unknown |
| `2` | 2 | reserved/padding in observed constructors | strong |
| `4` | 4 | controlled object's input-sync field copied from object offset `+88` | exact copy; semantic name unresolved |
| `8` | 4 | controlled object ID | exact |
| `12` | 12 | controlled object's current XYZ | exact |
| `24` | 16 | controlled object's current quaternion XYZW | exact |

The header transform is a request snapshot. It is not authority for actor
ownership or position.

### Plain movement tail

`sub_4DEF80` (`0x004DEF80`) is the distinct movement/stop builder. A normal
movement action has a 64-byte body:

| Body offset | Size | Meaning |
| ---: | ---: | --- |
| `40` | 4 | reserved/unknown word, constructed as zero |
| `44` | 12 | goal XYZ |
| `56` | 4 | goal flags |
| `60` | 4 | trailing movement word; semantic name unresolved |

`ActionStopMovement` type `4` is also serialized as a 64-byte body by build
103: `sub_A17440` assigns it the same 24-byte tail as type `3`. The stop branch
of `sub_4DEF80` writes only the common header, however, so the native tail has
no established stop semantics and must be ignored. The current server decoder
also accepts a shortened 40-byte type-`4` command; that is implementation
compatibility, not the exact native send length. An ordinary live left click
has been observed as type `3`, 64 bytes, with goal flags `1`. These are
player-to-server `0x9c` actions; they are not `0x91` packets.

### Character-ability tail

`sub_4D53C0` (`0x004D53C0`) constructs the 84-byte type-`7` body. The complete
application packet is 85 bytes after adding opcode `0x9c`:

| Body offset | Size | Meaning | Constructor evidence |
| ---: | ---: | --- | --- |
| `40` | 4 | target object ID, zero for a ground/targetless cast | direct |
| `44` | 12 | cursor XYZ | direct |
| `56` | 12 | retained target XYZ | direct |
| `68` | 4 | ability-bar/runtime index | direct |
| `72` | 4 | rank | strong from definition/rank lookup |
| `76` | 1 + 3 pad | request boolean plus padding; targetless held capture has word value `1` | byte write exact; name unresolved |
| `80` | 4 | user/action data token | direct; complete semantics unresolved |

`sub_4D5F50` maps a pending basic definition back to its bar index, supplies
target `0`, and copies the selected ground point to both XYZ fields before
local dispatch and network send. That is the native explanation for the exact
shape seen in the failed 1-1 run. It says nothing about whether the selected
ability damages a target, an area, the caster's surroundings, or only terrain;
the ability definition decides that.

`sub_5370F0` (`0x005370F0`) serializes the 40-byte header plus the tail size
from `sub_A17440` into logical message `29` / wire `0x9c`. Its fixed send
arguments establish reliable ordered action traffic. It normalizes target-like
object IDs for action types `7`, `8`, `9`, and `11` before sending.

## Local prediction versus authoritative action

`sub_4E5B90` (`0x004E5B90`) applies locally generated input before it is sent:

- type `3` starts local point locomotion through `sub_A147D0`;
- type `4` stops local locomotion;
- type `7` resolves the equipped definition/rank and calls the generic action
  path `sub_4D5A90` (`0x004D5A90`).

When local generic admission returns result `3` (pursuit), `sub_4D5A90`
starts predictive target-follow locomotion when the target resolves, otherwise
predictive point locomotion toward the retained target point. The target-follow
call uses `sub_A149C0` with player-follow flags `0x81` and a `0.9` multiplier
on the locally retained range operand. This is client responsiveness, not an
authoritative success decision. The exact derivation of that retained local
range operand is less certain than the server behavior's explicit authored
range transformation below.

The action is still sent to the authority. Generic authoritative admission at
`sub_9E0660` validates definition/rank, actor and target state, conflicting
ability state, cooldown, mana, range, hit policy, and blocking modifiers.
`sub_9E1540` creates an ability only after acceptance. Pursuit result `3`
creates no ability and pays no cooldown or mana.

## Authoritative `MoveToRange` continuation

The executable's creature behavior tree contains leaf ID `0x08BC8619`,
callback `sub_A20CB0` (`0x00A20CB0`), and stimulus mask `0x10`. This callback
consumes the retained out-of-range action and activates the packaged behavior
named exactly `nBehavior_MoveToRange` through `sub_A69040` (`0x00A69040`). It
copies the following values into behavior thread data:

| Thread slot | Type | Retained meaning |
| ---: | --- | --- |
| `0` | float | authored ability range; later overwritten with completion marker `1` |
| `1` | float/Lua number | ability definition/GUID operand passed back to `RequestAbility` |
| `2` | int | rank/request operand passed back to `RequestAbility` |
| `3..5` | float | retained target point XYZ |
| `6` | int | originating action ID used by cancellation |
| `7` | int | retained native owner/modifier slot; not consumed by the Lua retry |
| `8` | int | user/action data passed back to `RequestAbility` |

Chunk `826` performs this exact sequence:

1. Read the actor and current behavior-tree target.
2. Read retained point XYZ and action ID.
3. Reject an immobilized actor.
4. Transform the authored range `R` with `GetAdjustedRange`:

   ```text
   adjustedRange = R > 10 ? R - 1 : R * 0.800000011920929
   ```

5. With a live nonzero target, yield in
   `nThread.MoveTowardObject(actor, target, adjustedRange)`. With target zero,
   yield in `nLocomotion.MoveToPointWithinRange(actor, x, y, z,
   adjustedRange)`.
6. After the locomotion continuation resumes, recover the ability operand,
   rank/request operand, and user data and call
   `nAbility.RequestAbility` once with the retained actor, target, XYZ, action
   ID, and user data.
7. Mark thread slot `0` with `1` and return. There is no Lua polling loop and
   no request from the client on arrival.

The inset is intentional. It places the actor safely inside the authored
admission envelope rather than stopping exactly on the strict native boundary.

### Target-follow stopping distance

`nThread.MoveTowardObject` binds to `sub_A08FD0` (`0x00A08FD0`). For a
controlled/player actor it forces the second locomotion option on, so
`sub_A149C0` (`0x00A149C0`) constructs goal flags `0x81`:

```text
flags = 0x01 | 0x80
allowedStopDistance = desiredStopDistance
                    = actorFootprint
                    + targetFootprint
                    + adjustedRange
```

The goal retains the target object ID and its current position. The separate
target-position vector in the locomotion command remains zero. Footprints come
from both live runtime nouns; they are not optional collision padding supplied
by the client.

The continuation `sub_A08830` (`0x00A08830`) repeatedly resolves both objects
and tests `sub_A15170` (`0x00A15170`) against live positions and footprints:

```text
actorFootprint + targetFootprint + adjustedRange
    > EuclideanDistance(actor.position, target.position)
```

The comparison is strict. A moving target can extend the chase. Invalid/dead,
blocked, immobilized, or failed-path state terminates through the behavior
failure/deactivation path rather than authorizing an attack.

For example, Blitz's recovered footprint is `0.8` and Voltic Slash has authored
range `1.75`. Its authoritative pursuit inset is `1.4`, so against a target
with footprint `Ft` the center stopping distance is `2.2 + Ft`. Ordinary
admission still uses the authored `0.8 + 1.75 + Ft` envelope; the `0.35` gap is
the deliberate retry margin.

### Point stopping distance

`nLocomotion.MoveToPointWithinRange` binds to `sub_A09FD0`
(`0x00A09FD0`). Core point-goal construction adds only the actor footprint:

```text
allowedStopDistance = desiredStopDistance
                    = actorFootprint + adjustedRange
targetObjectID = 0
```

`sub_A08740` (`0x00A08740`) compares the authoritative actor position with the
retained point and resumes on the same strict footprint-expanded threshold.
This branch supports a real targetless ability whose native admission requests
pursuit toward its point. It does not convert a type-`7` action into a normal
move and does not imply that every targetless ability should pursue.

## Movement replication packet layouts

### `ObjectPlayerMove` / `0x91`

`sub_A20210` (`0x00A20210`) snapshots the 80-byte locomotion command into
logical message `18`, wire opcode `0x91`. It is authority-role guarded and is
sent outward; the inspected client role does not use it as player input.

| Offset including opcode | Size | Field |
| ---: | ---: | --- |
| `0` | 1 | opcode `0x91` |
| `1` | 4 | object ID |
| `5` | 4 | goal flags |
| `9` | 12 | goal XYZ |
| `21` | 12 | facing XYZ |
| `33` | 12 | external velocity XYZ |
| `45` | 12 | external force XYZ |
| `57` | 4 | allowed stopping distance |
| `61` | 4 | desired stopping distance |
| `65` | 12 | target-position XYZ |
| `77` | 4 | target object ID |

A player target-follow pursuit is therefore an 81-byte packet with flags
`0x81`, current target XYZ in the goal, both footprint-expanded inset distances,
zero target-position XYZ, and the nonzero target object ID. A normal point
move uses target ID zero and its own goal flags; a stationary/facing command is
another shape and must not be confused with pursuit.

### `LocomotionUpdate` / `0x94`

`0x94` is reflected `cLocomotionData` state. Its size is field-mask dependent
(the latest trace includes 71-byte payload examples). It carries snapshots and
special locomotion initializers such as lobs; it is not a fixed reached-goal
message and no inspected handler treats it as client confirmation of arrival.

### `LocomotionUnreliable` / `0x95`

The fixed payload is 16 bytes, 17 including opcode:

| Offset including opcode | Size | Field |
| ---: | ---: | --- |
| `0` | 1 | opcode `0x95` |
| `1` | 4 | object ID |
| `5` | 12 | goal/partial-goal XYZ |

The receiver writes the vector into locomotion goal/partial-goal state. The
latest trace shows it accompanying authority-created `0x91` motion and then
continuing as corrections. No client sender or pursuit-complete meaning was
recovered. Exact original-server reliability and dirty-flush cadence remain
unresolved.

## Cancellation, replacement, and retry ownership

- The original client action has an action ID. `MoveToRange` retains that ID
  in thread slot `6`.
- If the behavior is deactivated before it completes, its packaged
  `Deactivate` callback resolves the player owning the actor and calls
  `nGameSimulator.SendActionCancelMessage(playerID, actionID)`. The native
  binding is `sub_A05CA0` (`0x00A05CA0`) and publishes cancellation through
  `sub_9BD880`.
- A replacement movement, stop, new selected action, target loss, actor death,
  immobilization, or behavior-tree priority change must retire/replace the
  active pursuit rather than leave two owners moving the same hero. The exact
  retail priority ordering among every creature-tree stimulus is not fully
  named, but the single leaf/stimulus and its explicit deactivation callback
  make concurrent retained pursuits incompatible with the recovered design.
- Arrival does not cause the client to resend the original action. The
  authoritative behavior owns the one post-arrival `RequestAbility` retry.
  Native admission remains the only component allowed to create/commit the
  ability. If that retry is no longer admissible, no attack instance or cost
  may be invented.
- Held/basic repetition is different: build 103 can emit fresh client action
  requests with fresh action IDs. Those are new attacks, not arrival retries
  for the retained pursuit.
- `0x91`, `0x94`, `0x95`, RakNet ACKs, elapsed `distance/speed`, and client
  transform estimates are not attack-admission signals.

The exact wire opcode/body for the behavior-originated action-cancel message,
and whether every possible replacement publishes it or some teardown paths are
silent, remain open. The native call and retained action-ID ownership are exact.

## Latest-capture interpretation

The failed 1-1 interval provides two useful negative/positive results:

- Empty-ground attempts produced target-zero index-zero ability activity and
  no type-`3` movement at the server. The trace's cooldown hooks and server log
  agree that the client remained on the ability path.
- The same run later produced targeted basics outside server-admitted range,
  for example Wraith source `3`, target `9`, index `0`, repeatedly rejected at
  an eight-unit reported separation. No player pursuit followed because the
  current server rejects that command instead of preserving result `3` as a
  `MoveToRange` continuation.

The trace's received player-object `0x91` packets use flags `0x42` around
accepted current-server abilities. Those packets describe the present
implementation's facing/stop publication and are not retail proof for
out-of-range pursuit. Native `sub_A08FD0 -> sub_A149C0` supplies the recovered
player-pursuit flags `0x81` and stopping distances.

## Implementation boundary

### Retarget and sync boundary

Build 103 proves that the action sync byte is an action identity, not a target
generation. `ActionCommandResponse` type `2` clears the active or queued client
slot only when its sync byte matches. The authoritative `MoveToRange` behavior
separately retains one originating action ID in thread slot 6; deactivation
calls `SendActionCancelMessage(playerID, actionID)`, whose native sender builds
the same type-2 response for that exact action ID. Arrival retries the retained
action once and does not ask the client to manufacture a new sync value.

Held basics can still create fresh client requests with fresh action IDs. They
are distinct attacks, but the recovered client and packaged behavior do not
establish the absent retail authority's priority when such a fresh request
names another target while `MoveToRange` is active. The behavior tree proves
there can be only one retained pursuit owner; it does not prove whether retail
replaced the old owner, rejected the new attack, or deferred it.

The local campaign fallback therefore retains the first live target for one
basic pursuit and sends matching cancellation responses for same-basic request
noise until arrival or explicit movement, a different ability, swap, reset,
target invalidation, or teardown cancels the owner. This prevents the observed
alternating-target restart storm. It is a documented playability policy and
must be replaced if a retail server capture recovers fresh-action retarget
priority.

An evidence-aligned server boundary is:

1. Decode action types without shape inference. Accept ordinary movement only
   from type `3`; type `4` stops it. Never synthesize movement from target-zero
   type `7`.
2. Authenticate the controlled actor and validate all client positions as
   request data. Resolve the equipped ability definition and server-owned rank.
3. For type `7`, preserve target ID, cursor XYZ, target XYZ, index, request
   boolean, action ID, and user data separately. Let the authored ability shape
   decide whether target zero is legal and which position it uses.
4. Run native-equivalent admission. On success, cancel/replace movement as the
   ability requires and commit the attack normally. On a non-pursuit rejection,
   reject without movement or cost.
5. Only when a `shouldPursue` request returns out-of-range result `3`, retain
   the original action and install one authoritative pursuit owner:
   live-object follow for a valid target, retained-point follow only when the
   targetless ability itself legitimately requests point pursuit.
6. Use `adjustedRange`, both noun footprints for object follow, and the actor
   footprint for point follow. Publish player target-follow `0x91` flags
   `0x81`; publish `0x95` only as locomotion correction, not arrival.
7. Advance the hero and path in authoritative simulation steps. Re-evaluate
   live target, actor, path, death, immobilization, and the strict range
   predicate. Do not schedule commitment from a one-shot travel estimate and
   do not trust a client transform as arrival.
8. At authoritative arrival, retry the retained ability once through the same
   admission operation. Commit only if accepted. On replacement/failure,
   retire the pursuit and preserve the original action-cancel semantics.

This boundary fixes targeted basics without requiring a target for all basics,
without treating legitimate ground attacks as moves, and without transferring
arrival authority to the client.

## Confidence and remaining gaps

| Finding | Confidence |
| --- | --- |
| Type-`3` movement versus type-`7` targetless ability distinction and body layouts | exact native plus live capture |
| Empty-ground clicks can take the pending/basic path and emit target-zero abilities | strong native plus latest-run evidence |
| `MoveToRange` retained fields, range transform, one retry, and cancel callback | exact bytecode/native |
| Object-follow stop = both footprints + adjusted range; player flags `0x81` | exact native |
| Point stop = actor footprint + adjusted range | exact native |
| Server simulation owns commit-time arrival; no client reached-goal packet | strong positive and negative native/network evidence |
| Client performs local predictive movement before/while sending | exact native |
| Predictive path's `0.9` operand is the final authored-range equivalent | medium; multiplier/call are exact, upstream operand naming is not |
| Every possible behavior replacement sends an action cancel | medium; explicit deactivation does, all tree priority paths are not enumerated |
| `0x95` original-server reliability/cadence and cancel packet body | unresolved |

No executive fallback is required to decide movement versus attack or arrival
authority. Navigation/pathfinding details, simulation cadence, stuck recovery,
the complete behavior-tree priority order, and the two remaining wire details
above still require a retail server capture or deeper scheduler recovery.
