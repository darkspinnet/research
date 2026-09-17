# Campaign ground-click movement admission

## Result

The server does not fail to decode ordinary campaign movement. Build 103 never
sends the type-`3` command in the failing 2026-07-22 12:18 run because the
client's locally selected basic-attack state is not retired by darkspin's
accepted-action response.

The decisive mismatch is `ActionCommandResponse` (`0xa8`) response type `0`.
Every currently accepted campaign ability uses it. Build 103's receiver
`sub_537D90 -> sub_4D64D0` has cases `1`, `2`, `4`, `8`, and `16`, but no case
`0`; a type-`0` response returns without changing action state. A locally
admitted targeted attack has already stored its definition and sync byte in the
combat input state at `+4/+8`. An out-of-range local `shouldPursue` result can
instead leave the same definition and sync byte in the queued-basic slot at
`+52/+56`, which has no timeout. Consequently the empty-ground resolver sees
an ability still active/queued, does not immediately dispatch its separate
movement descriptor through `sub_4DEF80`, and the queued-basic pump
`sub_4D5F50` can construct a type-`7` targetless request at the ground point.

Response type `1` is the native accepted-action branch. It first clears both
`+4/+8` and `+52/+56`, resolves the response's ability definition, and installs
the bounded response/cooldown window at `+24/+28/+32..+44`. Response type `2`
is the matching rejection/cancel branch and also clears both basic slots when
the sync byte matches. Darkspin already sends type `2` for rejected requests,
but sends type `0` for every accepted one. Neither the current initial-player
record, ability boundary, `CooldownUpdate`, nor `SetAnimationState` repairs the
unhandled success response.

This is distinct from held Shift+left-click. That input intentionally selects
the targetless basic path and sends type `7` with target `0`, equal cursor and
target positions, and request byte `1`; release sends type `10`. It is also
distinct from a genuine ground-fired ability, which has the same target-zero
type-`7` wire shape. Target ID zero is therefore never evidence of movement.

## 12:18 playtest evidence

The run began at `2026-07-22 12:18:50 PDT`, entered campaign 1-1 at
approximately `12:19:52`, and reported ordinary ground movement still broken.
The relevant chronology in `darkspinner.log` is:

- `12:20:06.463` through `12:20:07.073`: four targeted Blitz basic requests
  enter the server's `shouldPursue` path for target `9`.
- `12:20:07.173`: the targeted basic is accepted. More targeted requests are
  rejected while the action is busy, and another is accepted at
  `12:20:07.772`.
- `12:20:23.683`: Ride the Lightning is accepted.
- `12:20:52.761`: control switches to Sage.
- `12:20:58.452` through `12:21:06.308`: the server repeatedly receives and
  accepts/rejects Sage basic requests with target `0`, index `0`, and equal
  cursor/target positions. These are targetless ability requests, not moves.
- There is no `RakNet campaign movement` acceptance or rejection in the run.
  The handler logs either result for every decoded type-`3` campaign command,
  so this is a client-admission absence rather than silent server rejection.

The Fang capture independently shows the packet family causing the state
mismatch. Its `message_receive` entries decode accepted `0xa8` prefixes such as
`02 00 00 00 f7 b5 c6 43 ...`: sync byte `2`, response type `0`, followed by
the accepted ability hash. Rejections use prefixes such as
`09 02 00 00 01 00 00 00 ...`: the second byte is response type `2`.
Accepted actions are followed by ordinary `0xa5` animation and/or `0xc1`
cooldown messages, but no recognized type-`1` action response. For example an
accepted basic later in the same capture arrives as the contiguous sequence
`0xa8(type 0)`, `0x91`, `0x95`, `0xa5`, `0xc1`; the cooldown hook confirms the
`0xc1` was applied, not that action selection was cleared.

Fang records outgoing RakNet datagram sizes and digests rather than decoded
action bodies. The companion server log is therefore the authoritative source
for the absence of type `3`; the trace is the source for the exact received
response, animation, and cooldown sequence.

## Empty-ground input path

Addresses below are image virtual addresses in the canonical build-103
`Game.c`.

### Resolver choice

The ordinary click resolver around `sub_44CFA0` first distinguishes a valid
clicked combat target, a held basic/Shift input, interactables, and empty
terrain. Its empty-terrain movement branch calls `sub_44A870`, which calls
`sub_4DFA50`. The latter creates the persistent movement descriptor at
`dword_1438A0C..dword_1438A1C` only when `sub_4DF140` admits it. The per-frame
pump `sub_4DF800` dispatches it only when the admission result is `1`.

`sub_4DF140` consults `sub_4D5250` and `sub_4D52C0`, the active and timed
action-response states. A still-active ability changes admission from the
immediate movement result, leaving the click pending instead of calling the
movement builder. A held basic takes `sub_44C7F0 -> sub_4DFBE0` and deliberately
creates a type-`7` ability descriptor instead.

### Type-3 construction

`sub_4DEF80` builds the distinct 64-byte movement body:

| Body offset | Meaning |
| ---: | --- |
| `0` | type `3` (or type `4` when the descriptor is a stop) |
| `1..39` | common action snapshot from `sub_4DED90` |
| `40` | zero |
| `44..55` | goal XYZ |
| `56` | goal flags (`0` here; local application adds ordinary locomotion state) |
| `60` | trailing movement word from the descriptor |

`sub_4DF540` sends this descriptor through `sub_5370F0`; `sub_4DF6E0`
applies it locally through `sub_4E5B90`. In the latter, case `3` calls
`sub_A147D0` point locomotion and clears the queued-basic definition and sync
at `+52/+56`. The movement path does not inspect or encode an ability target.

### Type-7 construction

`sub_4D53C0` builds the 84-byte character-ability body. Its type is `7` for
bar indexes below `5` and `8` otherwise. The tail is target ID, cursor XYZ,
retained target XYZ, bar index, rank, one request byte plus padding, and user
data.

`sub_4D5F50` is particularly important here. It pumps the persistent queued
definition at combat-state `+52`, tests whether it can now execute, maps it
back to a bar index, and then calls `sub_4D53C0` with target ID **zero** and the
stored point copied to both position fields. It clears `+52/+56` only after
this construction succeeds. Thus a stale queued basic naturally produces the
exact target-zero ground-request shape in the playtest without ever reaching
`sub_4DEF80`.

## Persistent client fields

There are three related state groups. Only the first two select or retain a
basic; the third can temporarily block movement but cannot turn it into an
attack by itself.

### Combat input state at `sub_4E4E00()` (`dword_1438CB0 + 80`)

| Offset | Role | Writers and retirement |
| ---: | --- | --- |
| `+4` / `+8` | Locally active ability definition / action sync byte | Set by successful local `sub_4D5A90`; type-`1` response clears it, matching type `2` clears it, and `sub_4D5250` has a three-second safety expiry using `+16`. Current response type `0` leaves it unchanged. |
| `+16..+23` | Active-slot safety deadline | Set to client-now plus 3000 ms on local success. It bounds `+4`, but does not retire `+52`. |
| `+24` / `+28` | Accepted response definition / action sync byte | Set only by response type `1`; cleared by its `+40..+47` end deadline, response type `4`, reset, or replacement. This is a timed busy/admission state, not the selected-basic producer. Current type `0` never creates it. |
| `+32..+47` | Converted response start/commit/end timing | Populated by type `1`; consulted by cooldown/channel and input-admission helpers. Bad values can temporarily block movement, but cannot populate `+4` or `+52`. |
| `+48` | Current local action/animation handle | Cleared at the start of `sub_4D5A90` and reset paths. It participates in presentation/action ownership but is not used by `sub_4D5F50` to choose a basic. |
| `+52` / `+56` | Queued/pursuit basic definition / action sync byte | Set directly when local admission returns result `3`; response type `8` also transfers `+4` here before starting follow/point locomotion. Matching type `2`, response type `1`, normal type-`3` local application, reset, or successful `sub_4D5F50` construction clears it. It has **no time expiry** and is the durable stale-basic hazard. |
| `+60` | Retained target object ID | Stored by `sub_4D5A90`; used for local pursuit setup and geometry. It does not change the queued pump's target-zero type-`7` construction. |
| `+64..+75` | Retained target/ground point | Stored by `sub_4D5A90`; copied to both type-`7` position fields by `sub_4D5F50`. |
| `+76` | Retained rank/request operand | Used when resolving and retrying the queued definition. |
| `+80` | Retained range | Used by local object/point pursuit. |
| `+84...` and `+576...` | Per-ability and global cooldown tables/deadlines | Updated by `CooldownUpdate` through `sub_53A520 -> sub_4D6320`. They affect admission timing, but do not write `+4/+8` or `+52/+56`. |

### Input resolver globals

| Field group | Role and lifecycle |
| --- | --- |
| `byte_1438A4C..byte_1438A4F`, `dword_1438A50..dword_1438A74` | Persistent generic/ability descriptor: active, sent/held flags, sync byte, selected definition/target, cursor/target points, actor, and user data. `sub_4DFBE0` fills it; `sub_4DF800` sends/applies it and normally clears the active byte. It is the source descriptor, not the post-send combat slot. |
| `dword_1438A0C..dword_1438A1C` | Persistent ordinary movement descriptor. It reaches type `3` only when `sub_4DF140` returns immediate admission. |
| `byte_11BEF28`, `byte_11BEF3C`, `byte_11BEF44`, `byte_11BEF4C` (with adjacent timers/points) | Mouse/basic press, modifier/held-mode, and the two targetless-basic press/release trackers in the `sub_44CFA0` resolver family. They explain intentional Shift+left-click type `7` and its type-`10` release. They do not prove that an arbitrary target-zero request was movement. |
| `byte_1438A4E`, `byte_1438AC2`, `dword_1438B1C` | Action sync/user-data bookkeeping used to match response/cancel boundaries. A matching response type `2` clears the active or queued combat slot; an unrelated rejection does not. |

### Type-10 cancel

`sub_4DEF10` changes a held ability descriptor to type `10` when its cancel bit
is set. `sub_4E5B90` has no type-`10` case, so local application of the cancel
does not itself clear combat-state `+4` or `+52`. The resolver bookkeeping and
`sub_4DEE10/sub_4DEC70` retire the matching press descriptor; the server's
current type-`10` branch releases its retained `basicSequence`. It sends no
client action response. Therefore type `10` correctly stops authoritative held
repetition but cannot compensate for an earlier accepted type-`0` response or
an already stale queued-basic slot.

## Packet audit

### Initial player and ability boundary

`marshalCampaignInitialPlayer` sends `LabsPlayerStatus` with
`AbilityCount=5`, the three characters, resources, and controlled/deploy
identity. Build 103 consumes the ability boundary in its player/ability-bar
record (`player +4976` in the recovered path). It does not write the combat
input state returned by `sub_4E4E00`. Combat state is zeroed by
`sub_4D5EF0`, including `+4/+8`, `+24..+56`, the cooldown summary, and input
maps. No current initial-player or boundary field selects the basic.

### Action response

This is the faulty packet:

- current accepted campaign response: type `0`;
- build-103 receiver: no type-`0` case, so all active/queued fields survive;
- native accepted branch: type `1`, which clears `+4/+8` and `+52/+56` before
  installing the bounded response state;
- current rejected branch: type `2`, which clears only when its sync byte
  matches the stored action.

The response already carries the fields type `1` expects: ability hash,
ability index, source start/commit/end, global cooldown, and user data. The
wire layout does not need to change.

Current Fang does not alter this contract. `app/fang/fang.c` traces decoded
`message_receive` prefixes and hooks `sub_4D6320` for `cooldown_apply`, but it
does not hook `sub_4D64D0` or rewrite `0xa8`. The bad response byte therefore
reaches the stock build-103 switch unchanged; Fang is observational in this
run, not the source of the stale selection.

### Cooldown

`CooldownUpdate` (`0xc1`) is decoded by `sub_53A520` and applied by
`sub_4D6320`. It updates the per-definition cooldown map and global cooldown
deadline. The Fang `cooldown_apply` records in this run prove those packets are
accepted. That path never clears the active or queued basic slots, so adding,
removing, or retiming `0xc1` cannot fix the admission bug.

### Animation

`SetAnimationState` (`0xa5`) is decoded by `sub_53A1F0`; it resolves the object,
converts the timestamp, and applies animation state. It does not call
`sub_4E4E00`, `sub_4D64D0`, or either basic-slot clearer. The current animation
packets are therefore neutral to movement admission.

## Ordinary click versus ground attack

The server must preserve the wire discriminator:

| Intent | Required evidence | Never infer from |
| --- | --- | --- |
| Ordinary point movement | action type `3` and its movement tail | target ID zero, equal cursor/target positions, or lack of a clicked object |
| Stop | action type `4` | a stationary type-`7` request |
| Character ability | action type `7` plus the equipped definition's targeting rules | the assumption that every target-zero basic is a move |
| Squad ability | action type `8` plus its definition | position equality alone |
| Held-input release | action type `10` with the matching sync/action boundary | elapsed cooldown or animation end |

Legitimate cursor-area, toss, lob, point-blank, and ground-fired abilities can
all use type `7`, target `0`, and populated/equal positions. Converting those
requests to movement would corrupt combat and still leave the client's stale
selection state unresolved.

## Implementation-ready fix

Fix the server response, not movement decoding:

1. Change accepted campaign character-ability responses in
   `server/gameplay_udp.go` from `ResponseType: 0` to `ResponseType: 1`.
   Preserve the current sync byte, ability hash, index, start/commit/end,
   global cooldown, and user data. Apply this consistently to melee,
   projectile, cursor-area, point-blank, toss/lob, Ride the Lightning, and
   Tree of Life success paths (and the equivalent tutorial successes if the
   common client-state bug is to be removed there too).
2. Keep response type `2` for rejected/cancelled actions. Do not use target ID
   zero as a movement fallback.
3. For an out-of-range `shouldPursue` request retained by the server, cancel
   the client's stale local queue with a matching type-`2` response when the
   server takes pursuit ownership, then publish authoritative `0x91/0x95`
   pursuit. At authoritative arrival, commit the retained action and send the
   normal type-`1` accepted response using the original sync byte. This avoids
   both a client-generated targetless retry and dual movement ownership.
4. Continue treating type `10` solely as held-input release unless a matching
   response is explicitly sent; it is not an accepted-action completion.
5. Add packet tests that decode the accepted response's second byte as `1`,
   rejection/pursuit-transfer cancellation as `2`, and assert that target-zero
   type `7` is never routed to the type-`3` movement handler.

If changing all success paths at once is considered too broad, the smallest
Fang compatibility fix is to rewrite incoming campaign `0xa8` response type
`0` to `1` immediately before `sub_537D90` dispatch, leaving the remaining 55
payload bytes unchanged. The server fix is preferable because it restores the
native protocol contract and keeps unpatched build-103 clients correct.

## Minimal verification hook

The static cause and fix are proven, but one compact Fang hook will verify the
runtime state transition without decoding mouse intent from packets. Hook
`sub_4D64D0` on entry/exit and emit one `client_state` record containing:

```text
sync, responseType,
activeDefinition(+4), activeSync(+8), activeDeadline(+16),
responseDefinition(+24), responseSync(+28), responseEnd(+40),
queuedDefinition(+52), queuedSync(+56), retainedTarget(+60)
```

Optionally add one record at `sub_4DFA50` with `sub_4DF140`'s result and one at
`sub_4D5F50` only when `+52 != 0`. That is the smallest hook set that proves:

- current type `0` leaves `+4` or `+52` unchanged;
- fixed type `1` clears both and installs the bounded response window;
- the next genuine empty-ground click reaches movement admission and produces
  a 64-byte type-`3` action;
- any later target-zero type `7` came from the explicit held/ground-ability
  path rather than being relabelled as movement.

No implementation file was changed for this investigation.
