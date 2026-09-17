# Live Quadra timeline adapter design

## Decision

Replace only the Sage-boundary Quadra presentation assembled by
`marshalTutorialSageEncounterStart`. Keep Sage unlock packets, Quadra authority,
combat registration, object creation, and the stationary movement goal on the
existing path. Route Quadra's visibility and animation through the deterministic
`FirstAggro_SpecialOne` program recovered from Lua chunk `502`.

This is deliberately not a conversion of tutorial encounter handling to
`server/sim`. In particular, the generic opening-enemy branch in
`ActionCommandMsgs` remains byte-for-byte and control-flow unchanged: do not
change `isEncounterSpawnNeeded`, `marshalTutorialEnemyFirstAggro`, the immediate
pursuit packets, or `scheduleOpeningAttack`.

The first live slice consumes these chunk-502 intents as follows:

| Simulation offset | Intent | Live adapter result |
| ---: | --- | --- |
| `0` | cinematic focused on `specialOne`, duration `4.291667s`, radius `100` | Retain in the semantic trace, but emit no packet in this slice. Adding the cinematic wire changes more than Quadra visibility/animation and is a separate integration. |
| `0` | visibility false | One `ObjectUpdate` for object `42`, at the authored Quadra position, with `IsVisible=false`. |
| `0` | wait `2s` | No packet; register the next outbox deadline. |
| `2s` | visibility true | One visible `ObjectUpdate` for object `42`. |
| `2s` | visibility true | Emit the second visible `ObjectUpdate` too. It is authored, ordered, and intentionally idempotent. |
| `2s` | animation `character_teleport_in` | One `SetAnimationState` for object `42`, after both visibility packets. |
| `2s` | wait `1.291667s` | No packet; retain the continuation. |
| `3.291667s` | wait `1s` | No packet; retain the continuation. |
| `4.291667s` | sequence complete | Mark the adapter run complete; no packet and no object mutation. |

The existing immediate `character_teleport_in` must be removed only from the
Quadra construction at this boundary. Ordinary stationary enemies, ability-test
enemies, teleporter-test enemies, horde enemies, and the generic opening enemies
continue using their existing marshal helpers.

## Exact role resolution

Use the stable simulator role `sim.Role("specialOne")`. Resolve it from a
per-gameplay-session, immutable adapter binding created when the Sage encounter
trigger is accepted:

```text
specialOne
  object ID: 42 (tutorialSageEnemyObjectID)
  definition: tutorialSageEnemyDefinition()
  noun: TutorialSpecialOne_Intro.Noun
  position: tutorialSageUnlockPosition
  owner: the gameplay session keyed by packet.Address.String()
```

Do not resolve the role by scanning noun hashes, by proximity, by the active
hero, or by creature index. Sage is object `17` (`PC_LF_Mage.Noun`) and is not
`specialOne`; the triggering/active player is normally object `1` and is not the
role either. The binding must also carry a monotonically increasing adapter-run
generation so a reused UDP address cannot receive an earlier session's packets.

At trigger acceptance, verify under the gameplay session lock that the session
exists, is a dungeon, the loot/Sage prerequisite is met, Quadra has not already
started, and `openingEncounter` exists. Then bind `specialOne`, activate its
simulator role lifetime, and add object `42` to `openingEncounter` exactly once.
The existing `isSageUnlocked` and `isSageEnemySpawned` flags remain the command
and combat gates; do not create a parallel authority flag in the adapter.

## Immediate packet boundary

Split the current combined helper conceptually, without broadening the generic
spawn helper:

1. Marshal the existing five Sage unlock/deploy packets.
2. Marshal Quadra's existing authority packets: `ObjectCreate`,
   `CombatantDataUpdate`, and `AttributeDataUpdate`.
3. Marshal the existing stationary `ObjectPlayerMove` with goal flags `0x20`.
4. Run chunk `502` in the activated `specialOne` scope and translate its due
   visibility intent to the hidden `ObjectUpdate`.
5. Return the complete boundary batch in that order.

Quadra is therefore created and combat-authoritative immediately, but hidden
until the simulator reaches `2s`. Do not delay `openingEncounter.addEnemy`, the
three authority packets, or the stationary goal. Do not emit the old immediate
visible update or old immediate animation.

## Packet scheduling and simulator clock

Store one Quadra adapter run in `gameplayPeerSession`. It owns the simulator (or
a narrowly wrapped `sim.Session`), the `specialOne` cancellation scope, the
event cursor, the source-time epoch, the adapter-run generation, and its pending
outbox batches. `server/sim` starts no goroutines and is not concurrency-safe;
all calls to `RunProgram`, `AdvanceTo`, `Events`, `InvalidateRole`, `Stop`, and
cursor/outbox mutation must occur while holding `sessionMutex` for this session.

Use simulator event offsets as authority. Schedule wakeups for the three future
deadlines `2s`, `3.291667s`, and `4.291667s` relative to the accepted movement
packet. A wakeup must:

1. lock `sessionMutex`;
2. re-resolve the session key and compare the session and adapter-run
   generations;
3. confirm the `specialOne` scope is still active;
4. advance to the exact simulator deadline, never by observed wall-clock
   lateness;
5. collect all newly due events in sequence order;
6. translate and marshal the whole same-deadline packet batch;
7. atomically commit the event cursor and outbox batch;
8. unlock, then hand a copied batch to RakNet.

The `2s` callback returns exactly three application packets in order: visible,
visible, animation. The later callbacks normally return no application packets,
but they must still advance the simulator so completion and cancellation state
are deterministic. Equal-deadline ordering is simulator event sequence order,
not Go map order or goroutine arrival order.

Do not hold `sessionMutex` while encoding RakNet datagrams or writing the UDP
socket. The producer may build application payloads under the lock to preserve
the atomic commit, but it returns copied immutable byte slices after unlocking.

## Source timestamps

Capture `packet.SourceTime` from the accepted Sage-triggering movement packet as
`sourceTimeEpochMilliseconds`. That field is milliseconds since the RakNet
server's `startTime`; it is not client time and it must not be sampled again in
a delayed callback.

For every timestamped presentation message use:

```text
timestamp = sourceTimeEpochMilliseconds + floor(event.At / 1ms)
```

Thus the Quadra teleport-in animation is stamped exactly
`packet.SourceTime + 2000`. Visibility messages have no timestamp. Keep the
simulator's microsecond deadlines (`3.291667s` and `4.291667s`) intact for
scheduling and cancellation even though the current live packet slice emits no
timestamped message at those boundaries. Never use callback wall time,
`time.Now`, accumulated timer delays, or a fresh server source time; those make
late callbacks change authored timestamps.

## Cancellation

Cancellation is generation-checked and idempotent. Invalidate the
`specialOne` role and discard every unsent Quadra outbox entry when any of these
occurs:

- object `42` is defeated or otherwise deleted;
- the encounter leaves the phase that owns the Sage/Quadra introduction;
- the gameplay session is reset/replaced by a new `HelloPlayerRequest` for the
  same address;
- the peer/session is torn down;
- the simulator is reset or stopped.

Object death uses `Simulator.InvalidateRole("specialOne")`; session teardown
uses `Simulator.Stop()` and increments the enclosing session generation. A
scheduled producer first checks both generations and returns no packets for a
stale scope. `SequenceCompleteIntent` only ends the authored thread: it must not
despawn Quadra, remove it from `openingEncounter`, or cancel combat authority.

The existing RakNet scheduler's request context is an additional transport
cancellation signal, not the gameplay lifetime authority. It cannot replace the
session/role generation checks because the handler context and a reused peer
address need not have the same lifetime as object `42`.

## Atomic outbox requirement

The live cutover must not use three independent fire-and-forget calls to the
current `Packet.ScheduleFunc`. That API starts independent goroutines, returns
no cancellation handle, drops producer errors, and can partially write a
multi-packet application batch. It cannot guarantee the required commit:

```text
accepted Sage/Quadra state
+ activated simulator scope
+ every timeline wakeup
+ immediate application batch
```

Introduce the smallest RakNet scheduling extension that can atomically register
one ordered group of deadline producers and return a cancellation handle (or an
equivalent per-peer ordered outbox transaction). Registration must be all or
nothing. If registration fails, the movement handler must return an error with
no Sage/Quadra flags, encounter enemy, simulator run, cursor, or immediate
Quadra packets committed. Cancelling the group must prevent producers that have
not begun from publishing.

Within a producer, marshal every application payload for one simulator deadline
before advancing the committed cursor. If any intent is unsupported, role
resolution fails, or any payload fails to marshal, append nothing and leave the
cursor at the first undispatched event so the batch is retryable/diagnosable.
Never expose the first visible packet without the second visible packet and
animation from the same `2s` deadline. Copy payload bytes into the outbox so
later session mutation cannot alter them.

UDP cannot provide transactional delivery once writes start. Here "atomic"
means atomic state/outbox publication and ordered ReliableOrdered submission,
not impossible rollback of datagrams already accepted by the socket. The
outbox sender must submit a same-deadline batch through one peer-ordered send
critical section and report/log the first encoding or write failure with the
adapter-run generation and simulator event sequence. It must never mark a batch
sent before all its payloads have been encoded and accepted for ordered send.

## Failure policy

Compile/load the allowlisted chunk-502 program before accepting the live
trigger (preferably when constructing the gameplay service), including its
content ID/hash validation. There is no fallback to the old immediate visible
spawn: a missing/mismatched program, missing scheduling/outbox capability, role
mismatch, or initial marshal failure rejects the Sage/Quadra transition without
mutating session authority. A delayed adapter failure cancels the run, leaves
Quadra authority unchanged, and logs the exact event sequence and provenance;
it must not invoke the generic first-aggro helper.

## Verification contract

Add focused adapter tests when implementing:

- role `specialOne` resolves only to object `42`, never Sage `17` or player `1`;
- immediate order is Sage packets, Quadra authority/create state, stationary
  goal, hidden update, with no immediate visible update or animation;
- `2s` produces visible, visible, teleport-in and stamps the animation at
  `source epoch + 2000` even when the callback is late;
- `3.291667s` and `4.291667s` advance and complete without packets;
- death, phase exit, session replacement, teardown, reset, and stop suppress
  every later packet;
- duplicate Sage-boundary movement starts no second run and adds no second
  enemy;
- marshal or schedule registration failure commits neither flags, encounter
  state, simulator cursor, nor outbox bytes;
- a same-deadline failure publishes none of the three `2s` packets;
- two sessions have independent clocks, roles, generations, and outboxes;
- existing generic opening-enemy first-aggro tests and packet order remain
  unchanged.
