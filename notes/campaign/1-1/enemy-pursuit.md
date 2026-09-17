# 1-1 enemy pursuit authority

## Result

Build `5.3.0.103` makes the simulation authority responsible for NPC pursuit. The
native Lua continuation starts a target-follow locomotion job, sends its command
to clients, and then remains yielded until authoritative runtime objects satisfy
a live range predicate. It neither waits for a client completion notification nor
computes a one-shot arrival time from distance and speed.

For 1-1, the server should therefore simulate navigation and NPC transforms,
re-evaluate the moving target and path state during simulation steps, and resume
the pending ability only when the native-equivalent range test succeeds. The
server may replicate the movement goal with `ObjectPlayerMove` (`0x91`) and
subsequent goal corrections with `LocomotionUnreliable` (`0x95`), but those
messages are presentation output and provide no authority feedback.

The exact retail campaign phase scheduler, base speed operands, simulation tick
interval, repath cadence, and navigation implementation remain unrecovered. None
of those gaps changes which side owns arrival. Consequently, this investigation
does not add a fallback decision to `notes/help.md`.

## Evidence provenance

- Client build: `bin/game/GameBin/version_bin.txt` reports
  `5.3.0.103`.
- Executable: `bin/game/GameBin/Game.exe`, SHA-256
  `3c7248a4c6614450290a66c22ab7fcda86052b29a7b555e834d7c23f9b73ee5b`.
- Canonical decompiler output: `bin/game/GameBin/Game.c`, SHA-256
  `1fac9cda954e076446889376d5e3e89859ce058b75c2d50739d4aedfd662b1eb`.
- Runtime content database: `bin/darkspinner/darkspin/cache/content.db`,
  SHA-256
  `13fd5043b5aa5fffa703b6b3457568e3ecffcb0349720c2343ea23d3c471bb04`.
- Packaged behavior bytecode was enumerated through `darkrun db` and extracted
  beneath `bin/game/logs/1-1-enemy-pursuit/behaviors` for inspection.
- Relevant Lua resource: `lua_chunk` ID `826`, server-data resource ID
  `14403`, source name `behaviors/0xD3200CED.lua`, decoded size `1550`,
  SHA-256
  `a831735975eee01bce296bee2e1c67e532d56495fec674f51e1a386a7c845d6d`.

Addresses below are image virtual addresses from the canonical build-103
decompiler output.

## Packaged pursuit behavior

The requested symbol `nBehavior_Pursue` is not registered in the 31 packaged
behavior chunks, and the literal strings `nBehavior_Pursue` and `Pursue` do not
occur in them. The build-103 pursuit leaf is named
`nBehavior_MoveToRange`; it should not be documented as a recovered alias.

Chunk `826` registers `nBehavior_MoveToRange` and contains these contracts:

1. `GetAdjustedRange(range)` returns `range - 1` when `range > 10`, otherwise
   `range * 0.800000011920929`.
2. `Tick` obtains the acting and target object IDs from `nBehaviorTree`, reads
   its command state from `nThreadData`, rejects an absent target and an actor
   with nonzero `Immobilized`, and derives the adjusted range.
3. For a live target it yields through
   `nThread.MoveTowardObject(actor, target, adjustedRange, false, false, 0,
   false, true)`.
4. If the target object is no longer live, the alternate branch moves toward
   the retained target point with
   `nLocomotion.MoveToPointWithinRange(actor, x, y, z, adjustedRange)`.
5. Only after the locomotion call resumes does the behavior invoke
   `nAbility.RequestAbility` with the retained action-command fields and mark
   the request state with `nThreadData.SetInt(0, 1)`.
6. `Deactivate` sends an action-cancel response through
   `nGameSimulator.SendActionCancelMessage(playerID, actionID)`. This is an
   outbound cancellation of the originating action command, not an NPC arrival
   acknowledgement from a client.

Disassembly operands show float slots `0`, `3`, `4`, and `5` and integer slots
`0`, `1`, `2`, `6`, and `8`. Their complete authored field names are not
preserved in the bytecode and remain unresolved; only their uses above are
asserted. Existing behavior-tree evidence identifies `MoveToRange` as stimulus
flag `0x10` in the static creature tree. The missing ordinary campaign phase
owner must not be invented from this queued-action leaf.

## Native locomotion continuation

The native binding table at `sub_A0FE90` (`0x00A0FE90`) maps:

- `nThread.WaitForNearGoal` to `sub_A08D40` (`0x00A08D40`);
- `nThread.MoveTowardObject` to `sub_A08FD0` (`0x00A08FD0`).

The locomotion binding table at `sub_A0D0E0` (`0x00A0D0E0`) maps
`nLocomotion.MoveToPointWithinRange` to `sub_A09FD0` (`0x00A09FD0`).

### Moving target

`sub_A08FD0` resolves the actor and target, calls `sub_A149C0` to install a
target-follow locomotion operation, calls `sub_A20210(actor)` to replicate the
new locomotion command, and installs `sub_A08830` (`0x00A08830`) as the Lua
continuation predicate. It does not calculate or store a travel duration.

`sub_A149C0` (`0x00A149C0`) initializes the movement state as follows:

- goal position is the target's current position;
- target object ID is retained, so the movement remains target-relative;
- flags are `(optionA ? 0x40 : 0) | (optionB ? 0x80 : 0) | 0x01`;
- both allowed and desired stopping distances are actor footprint plus target
  footprint plus the supplied adjusted range;
- the separate target-position vector in the wire structure is cleared.

The packaged NPC call supplies both flag options as false, producing flags
`0x00000001` and a nonzero target object ID.

On every scheduler check, `sub_A08830` resolves the current actor and target,
terminates on invalid/dead/blocked state, and otherwise tests
`sub_A15170(actor, target, range)` before optionally testing facing. The positive
range predicate at `sub_A15170` (`0x00A15170`) is exactly:

```text
actorFootprint + targetFootprint + adjustedRange
    > EuclideanDistance(actor.position, target.position)
```

The comparison is strict `>`. Both positions and both footprints are read from
the live runtime objects when the continuation is checked. A target that moves
can therefore extend pursuit without any rescheduled deadline.

### Retained point and path completion

`sub_A09FD0` installs point-within-range movement and continuation
`sub_A08740` (`0x00A08740`). That continuation compares the authoritative actor
position and retained goal against the stopping threshold; it also has no
network acknowledgement or computed duration.

`nThread.WaitForNearGoal` (`sub_A08D40`) uses `sub_A08610` as another live mover
predicate. Its defaults include epsilon `0.000015258789`, factor `0.95`, and
timeout `-1`; those are continuation-test operands, not evidence of a native
travel estimate.

The main mover update advances locomotion using frame delta from `sub_9D2960`,
copies the mover position into the runtime object's XYZ fields, and dirties the
object for replication. `sub_9F97A0` (`0x009F97A0`) queries the mover's reached
state through `sub_A59540` and emits a replay event through `sub_A5D8C0`.

`sub_A5D8C0` (`0x00A5D8C0`) creates the replay event
`Mover::ReachedGoal`, containing a replay ID and `reachedGoal` boolean. Its
receiver, `sub_A5FA90` (`0x00A5FA90`), applies that state locally through
`sub_9F97A0`. This is replay/diagnostic plumbing: it is not a game-message
constructor and is not evidence of a client-to-server completion packet.

## Speed operands and cadence

`sub_9E49D0` (`0x009E49D0`) computes effective native movement speed:

```text
base = inCombat or simulatorState == 3
    ? CombatSpeed attribute 12
    : NonCombatSpeed attribute 11
effectiveSpeed = base * (1 + MovementSpeedBuff attribute 48)
```

The attribute IDs are corroborated by packaged global definitions:
`NonCombatSpeed = 11`, `CombatSpeed = 12`, and `MovementSpeedBuff = 48`.
The four low-band 1-1 enemy nouns use locomotion/profile `npcCreature` and
ordinary walking: `ZelemBasicRanged.Noun` (instance `0x8F291AF3`),
`ZelemSpecialHaster.Noun` (`0xA1FDCCD8`), `NomadSnipe.Noun`
(`0x1DDB0187`), and `NomadWithDrone.Noun` (`0xB366BC74`). They also author
`builtins!sphere` and `DefaultPhysics.prop`, but no initial target or movement
goal. Their imported non-player-class records do not currently project
attributes 11 and 12, so their retail base speeds remain unresolved. Tutorial
constants such as noncombat `3` and combat `4.5` are not evidence for these
campaign nouns and must not be substituted as retail values.

Behavior wrapper `sub_A68F10` (`0x00A68F10`) activates the Lua behavior and
prepares its `Tick` continuation. Native continuation predicates are revisited
by the simulation scheduler while mover advancement consumes actual frame
delta. No fixed build-103 AI tick period, actor visitation order, or repath
period was recovered. Authored combat-idle waits in the `0.1` to `0.6` second
range are behavior delays, not the simulation tick. A local `100 ms` cadence is
therefore an implementation fallback, not retail evidence.

## Network contract

### `ObjectPlayerMove` / `0x91`

The logical locomotion command is 80 bytes, or 81 bytes including the packet
opcode:

| Offset including opcode | Size | Field |
| ---: | ---: | --- |
| `0` | 1 | opcode `0x91` |
| `1` | 4 | object ID, little-endian `uint32` |
| `5` | 4 | flags, little-endian `uint32` |
| `9` | 12 | goal XYZ, three `float32` |
| `21` | 12 | facing XYZ |
| `33` | 12 | external velocity XYZ |
| `45` | 12 | external force XYZ |
| `57` | 4 | allowed stopping distance |
| `61` | 4 | desired stopping distance |
| `65` | 12 | target-position XYZ |
| `77` | 4 | target object ID |

For the packaged target-follow request, flags are `0x00000001`, the goal is the
target position when the job is installed, both stopping distances are the sum
of actor footprint, target footprint, and adjusted range, the target-position
vector is cleared, and target object ID is nonzero.

`sub_A20210` (`0x00A20210`) copies this 80-byte command into logical message
type `18`, which maps to wire opcode `0x91`. Its send path is guarded by
`!sub_9BCF80()`; other role checks establish `sub_9BCF80() == 1` as the client
role. Thus this movement command is created by the authority/server role and
sent outward. There is no corresponding client-role send in this function.

### `LocomotionUpdate` / `0x94`

`LocomotionUpdate` is reflected locomotion-component state sent to a client.
No inspected path treats it as a client-owned NPC position or completion
message.

### `LocomotionUnreliable` / `0x95`

The packet is 17 bytes including its opcode:

| Offset including opcode | Size | Field |
| ---: | ---: | --- |
| `0` | 1 | opcode `0x95` |
| `1` | 4 | object ID, little-endian `uint32` |
| `5` | 12 | goal/partial-goal XYZ, three `float32` |

Its receiver copies the vector into the locomotion goal and partial-goal state.
Local runtime captures pair it with an earlier `0x91` when an NPC begins
moving, consistent with authority-to-client goal correction. The canonical
client decompilation contains the logical type-18 constructor used for `0x91`,
but no constructor call for logical type `22`, the `0x95` application message.
Accordingly, the build contains a receiver for `0x95`, not evidence that a
client sends NPC positions or completion acknowledgements with it. The exact
dirty-flush sender and RakNet reliability mode remain unresolved.

### Actual client input

Client-controlled movement and ability intent use `ActionCommandMsgs`
(`0x9c`). Observed player command bodies include a 64-byte left-click form and
an 84-byte targeted form, carrying the controlled actor, current/goal
coordinates, action data, and optional target. These messages express player
input. They do not transfer ownership of spawned NPCs and contain no NPC path
completion acknowledgement.

RakNet ACK traffic acknowledges datagram delivery only. It does not acknowledge
that a locomotion goal was reached. No client-to-server handler or sender was
found for an NPC arrival, NPC authoritative transform, or pursuit-complete
message.

## Server contract for 1-1

The evidence supports this runtime sequence:

1. The authoritative behavior selects pursuit and installs a target-follow
   mover using the adjusted requested range.
2. The authority sends `0x91` and may send later goal corrections such as
   `0x95` so clients can render the NPC.
3. The authority advances the mover/path using simulation frame delta and owns
   the NPC transform.
4. The yielded behavior is polled against live actor, target, path, death,
   immobilization, and range state.
5. When the strict live range predicate succeeds, the continuation resumes and
   requests the ability. Invalid or blocked state follows the behavior's
   failure/cancellation path.

A server implementation should simulate the path or an equivalent authoritative
navigation state. It should not trust a client notification because no such
retail message was recovered, and it should not schedule attack admission from
a one-time `distance / speed` estimate because the native contract does not do
so and the target, modifiers, and path can change during pursuit.

## Unresolved retail details

- The ordinary campaign/horde phase scheduler that chooses pursuit and retries
  an out-of-range attack. The recovered `MoveToRange` leaf proves the movement
  continuation contract but not that top-level scheduler's field layout.
- Base `NonCombatSpeed` and `CombatSpeed` values for the specific 1-1 enemy
  classes, and the full authored modifier path that produces attribute 48.
- Exact simulation tick interval, actor visitation order, path-replan cadence,
  path-corner representation, obstacle avoidance, and stuck recovery.
- The sender/flush policy and RakNet reliability used for `0x95` after the
  initial `0x91` command.
- Semantic names for all retained `nThreadData` slots in chunk `826`.

These are implementation-fidelity gaps, not an unresolved authority policy.
