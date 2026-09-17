# Chunk-215 Sage/Quadra live-cutover verification matrix

## Scope and oracle

Verify the live `ActionCommandMsgs` movement path at the application-payload
boundary. The test must call the gameplay handler directly and manually invoke
the captured ordered deadline producers; it must not use sockets, sleeps,
goroutines, or callback wall time.

Use the focused Sage checkpoint and a movement to
`tutorialSageUnlockPosition` (`259.346, 81.391, 25.088`). The accepted player is
object `1`, Sage is object `17`, and Quadra (`specialOne`) is object `42`. Let
`S` be the triggering movement packet's `SourceTime`; use `S = 7000ms` in the
fixture.

Pin the two authored inputs so a semantically similar substitute cannot pass:

| Input | Required identity |
| --- | --- |
| Sage unlock job | chunk `215`; SHA-256 `e98b1e855e64904689451bd79635c3487a7a20e29f1b943885c9648b61f8e30a`; `nTutorial_IntroSecondCreatureUnlock.main` |
| Quadra presentation | chunk `502`; SHA-256 `da18208fc597c67ce3251de30d12ad694ae4d644af0e1cb79b5383cc12ec6d8d`; `FirstAggro_SpecialOne` |
| Marker policy | marker `3912233898`; server-only; repeatable; authored radius and level identity |

Chunk 215 owns the first `0.5s`. Quadra begins only after that continuation
successfully unlocks Sage. Consequently all Quadra offsets below are relative
to `S + 500ms`, while scheduler deadlines are relative to the accepted
movement.

## Timeline and order matrix

All packet lists are application packets in one ReliableOrdered batch. Packet
IDs and list positions are mandatory; a set comparison is insufficient.

| Case / deadline from movement | Exact ordered result | Byte-level assertions | State and scheduling assertions |
| --- | --- | --- | --- |
| Accepted movement, immediate | `91 95` (`ObjectPlayerMove`, `LocomotionUnreliable`) | `91` is 81 bytes: object `[1:5] = 1`, flags `[5:9] = 1`, goal XYZ `[9:21]`, then 60 zero bytes. `95` is 17 bytes: object `[1:5] = 1`, the same XYZ `[5:17]`. | Register exactly one ordered group before returning the movement batch. Its delays are `0.5s`, `2.5s`, `3.791667s`, `4.791667s`. No Sage or Quadra authority/presentation packet is immediate. |
| Chunk-215 continuation, `+0.5s` | `a1 8c 97 96 a7 9b 8c 97 96 91 c9 8d` | Apply the per-packet matrix below. In particular, the minimal `9b` is index 5; Quadra authority follows it; cinematic precedes hidden visibility. | The whole 12-packet batch is one producer result. Only after successful production set Sage unlocked/spawned, add object `42` once, and advance the boundary to Quadra. |
| Quadra visibility/animation, `+2.5s` | `8d 8d a5` | The two `8d` payloads are byte-identical visible updates for object `42`. `a5` targets `42`, hashes `character_teleport_in`, and has timestamp `S + 2500 = 9500`. | Publish all three or none, preserving the authored duplicate visibility. Callback lateness must not change bytes or timestamp. |
| Quadra wait, `+3.791667s` | empty | Zero application bytes and no error. | Advance the simulator to Quadra offset `3.291667s`; do not complete or clear combat authority. |
| Quadra completion, `+4.791667s` | empty | Zero application bytes and no error. | Advance to Quadra offset `4.291667s`, consume sequence completion, and clear the pending run. Do not despawn object `42`. |
| Duplicate boundary movement | `91 95` only | The bytes obey the immediate movement contract above. No `a1`, Sage/Quadra `8c`, `9b`, `c9`, `8d`, or `a5` appears. | Schedule-group count remains one; no second run and no second encounter enemy. Run this assertion after the `+0.5s` producer has committed the spawn gate. |
| Schedule registration failure | empty result plus an error matching the injected sentinel and containing `moveSageSchedule` | No movement prefix and no Sage/Quadra application bytes escape. | Registration was attempted once with all four ordered producers. No cancel handle is returned/called; no Sage/Quadra flag, encounter enemy, run, cursor, or outbox is committed. Clear the error and retry the same movement: it must return `91 95`, register a fresh group, and produce the complete normal timeline. |

## The `+0.5s` packet matrix

This batch is the critical order proof: chunk 215's squad mutation and client
notification complete before Quadra becomes authoritative or cinematic.

| Index | ID | Owner / purpose | Required byte checks |
| ---: | ---: | --- | --- |
| 0 | `a1` | Sage roster reveal | `LabsPlayerUpdate` uses the fixture binding; creature count is `2`, locked-deck minimum is `2`, and the second hero is `PC_LF_Mage.Noun` / asset `0x55d1408f`. |
| 1 | `8c` | Sage create | Object `[1:5] = 17`; noun is `PC_LF_Mage.Noun`; asset is `0x55d1408f`; XYZ is the authored Sage position. |
| 2 | `97` | Sage combatant authority | Object `[1:5] = 17`; compare the remaining payload to `TutorialHeroStateMessages(17)` rather than accepting only the ID. |
| 3 | `96` | Sage attribute authority | Object `[1:5] = 17`; compare the complete attribute payload to the typed fixture. |
| 4 | `a7` | Keep Blitz deployed while revealing Sage | Player index byte `[1]` equals the binding slot, creature index `[2:6] = 0`, active object `[6:10] = 1`. |
| 5 | `9b` | `PlayerUnlockedSecondCreature` | Exact seven bytes: `9b 0f 36 bc e2 71 ff`. No default asset, object, position, or zero-valued reflected fields may be inserted. |
| 6 | `8c` | Quadra create authority | Object `[1:5] = 42`; noun is `TutorialSpecialOne_Intro.Noun`; XYZ is the authored Sage/Quadra position. |
| 7 | `97` | Quadra combatant authority | Object `[1:5] = 42`; full payload matches `TutorialEnemyStateMessages(42, 20)`'s combatant message. |
| 8 | `96` | Quadra attribute authority | Object `[1:5] = 42`; full payload matches the corresponding typed enemy attribute message. |
| 9 | `91` | Quadra stationary goal | 81 bytes; object `[1:5] = 42`, flags `[5:9] = 0x20`, XYZ `[9:21]` is authored position, trailing 60 bytes are zero. |
| 10 | `c9` | Quadra cinematic | Exact length 25; signed LE duration `[1:9] = 4292ms`; focus XYZ `[9:21]` is authored position; radius `[21:25] = float32(100)`. There is no subtype byte. |
| 11 | `8d` | Hide Quadra | Exact length 21; object `[1:5] = 42`, field tag `[5] = 6`, XYZ `[6:18]`, visibility tag `[18] = 16`, value `[19] = 0`, terminator `[20] = ff`. |

The stable authored-position float bytes are `4a ac 81 43`, `31 c8 a2 42`, and
`39 b4 c8 41` for X, Y, and Z. The cinematic radius bytes are
`00 00 c8 42`.

## Later presentation byte contract

Each visible `ObjectUpdate` is exactly 21 bytes:

```text
8d | 2a 00 00 00 | 06 | 4a ac 81 43 31 c8 a2 42 39 b4 c8 41 | 10 01 ff
```

The two copies must remain adjacent and identical. The animation follows them
and is exactly 26 bytes:

```text
a5 | object=42:u32le | hash("character_teleport_in"):u32le |
timestamp=(S+2500):u64le | overlay=00 | scale=1.0:f32le | source=0:u32le
```

For `S = 7000`, the timestamp bytes are `1c 25 00 00 00 00 00 00` (`9500`).
The timestamp is derived from the captured movement epoch plus the combined
`0.5s + 2s` simulation offset, never from producer invocation time.

## Atomicity and negative assertions

Use explicit prefix-failure checks in addition to the schedule rollback row:

- If chunk 215 has the wrong provenance, wrong event ID, or wrong step order,
  run creation fails before schedule registration and returns no packets.
- If Quadra authority marshalling or the offset-zero cinematic/hidden batch
  fails, return no packets and never register a group. A retry after removing
  the injected failure must traverse the full timeline.
- If any intent in the `+2.5s` same-deadline batch fails to encode or resolve
  `specialOne`, publish none of `8d 8d a5`; leave the cursor at the first
  undispatched event for diagnosis.
- Invoking a producer captured before `HelloPlayerRequest` replacement,
  teardown, accepted Beam Out, or object-42 defeat returns an empty batch. The
  cancel handle is idempotent and the stale run cannot mutate the new session.
- Producer invocation out of chronological order must fail rather than skip a
  deadline or publish a later suffix.

The success test is complete only when it proves byte fields and order together:
packet-ID order alone would miss the expanded `9b`, the two intentionally
identical visible updates, the cinematic's subtype-free layout, and the
source-time error that would stamp the animation `9000` instead of `9500`.
