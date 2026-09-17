# Campaign 1-1 Continue transition

## Result

Build 103 has a complete client-side Continue contract even though no retained
retail packet capture proves the retired server's persistence policy. On the
1-1 chain-voting screen, selecting the next mission sends exactly:

```text
AC 01 01 <selected-squad-record-id:u32le>
```

The smallest response that actually starts loading the next mission is
`GamePrepareForStart (0xB0)`, not `0xAA`, `0xAB`, or a required `GameState
(0x8A)` transition. Its 16-byte body identifies the level and markerset. For
the first chain continuation those identities are `zelems_3.Level` and
`zelems_3_ai.Markerset`; the `0xB0` receiver resolves the level noun and then
enters `cPreDungeonState` itself. `GameStart (0xB1)` follows only after the
client's normal loading/ready handshake.

The strongest supported single-player sequence is therefore:

```text
S2C 8A state=11 ChainVoting       // already entered after 1-1 Beam Out
C2S AC 00                         // request result/vote data
S2C A9 00 <337-byte record>       // zelems_1 completed, zelems_3 offered
S2C A9 01 <seconds:f32le>         // reset visible countdown
C2S AC 01 01 <squad-id:u32le>     // Continue
S2C B0 <zelems_3 setup:16 bytes>  // first message that begins loading 1-2
C2S 88 status=2 progress=<f32>    // loading progress, possibly repeated
C2S 88 status=4 progress=1        // local load complete
C2S 88 status=8 progress=1        // player/world ready
S2C B1 <level/player-index:u32le> // enter Dungeon
```

The exact status ordering is client-proven. Server replies interleaved with
those statuses, including `LabsPlayerUpdate (0xA1)`, dungeon object setup, and
whether retail repeated `0xB0`, remain server-owned. Darkspin's existing entry
flow already reflects statuses and sends `B1` on status `8`; Continue should
reuse that flow with a fresh `zelems_3` session rather than create a second
handshake.

`ChainLevelResults (0xAA)` is **not locally consumed by this executable**. The
wire name exists in the 78-entry message vocabulary, but build 103 has no
registration, dispatcher case, reader, copy, state callback, or sender for its
logical message ID `43`. No `0xAA` body should be invented.

## Evidence boundary

The labels used below are:

- **client contract**: directly read, written, or state-selected by build 103;
- **content contract**: exact runtime `content.db` identity or adjacency;
- **compatibility evidence**: behavior already exercised by darkspin, but not
  proof of retired retail-server policy;
- **server authority**: a choice the client and packaged content cannot make;
- **fallback**: the conservative local rule recorded in `notes/help.md`.

There is no retained retail trace of the 1-1 Continue transaction. In
particular, `bin/server/darkspin/logs/traces/client.jsonl` does not supply an
`A9`-`AC` result exchange. Static analysis proves what build 103 sends and what
it can consume; it does not turn local compatibility ordering into retail
ordering.

## Continue request

`MaxisPlanetScreen.AcceptMission` is bound by `sub_527C80` at `0x00527C80` to
callback `sub_527BF0` at `0x00527BF0`. The callback receives the ActionScript
selection as a double, converts it to an integer list position, selects one
64-byte record from the account's squad/deck list, stores that record's first
dword as the active selection, and calls `sub_527970(1, selectedID)`.

`sub_527970` creates logical message `41`, which the build-103 message table
maps to wire `0xAC` (`kGmsChainPlayerMsgs`). It reserves six payload bytes and
writes them as one subtype byte followed by five bytes copied from adjacent
locals:

```text
application ID                         AC
payload +0x00  subtype:u8              01
payload +0x01  continue flag:u8        01
payload +0x02  selected record:u32le   <selectedID>
```

The application message is therefore seven bytes including its ID. The first
dword of the selected 64-byte UI record is exact. Its client role is the
selected squad/deck record identity; no evidence makes it a level hash. The
server must validate it against the authenticated player's offered squad
records and must choose `zelems_3` from the frozen result, not trust this dword
as a next-level selector.

Unlike `MaxisBeamOut.OnBeamOutClicked`, this native callback has no one-shot
latch and no local transition. A double click or retry can resend the same
request while the client remains in Chain Voting. Acceptance must therefore be
idempotent on the server.

Cash Out is separate. `PlanetScreen.OnCashOut` calls the same sender with
subtype `2`, producing exactly `AC 02`; it does not share the Continue body.

## Result-message requirements

| Message | Proven receiver behavior | Requirement for Continue |
| --- | --- | --- |
| `A9 00` | `sub_449750 -> sub_449090` copies exactly 337 bytes, constructs the Planet Screen, and exposes completed and offered missions. | Required to present the voting screen. The offered-level field at body `+0x0E9` must be `zelems_3.Level`. |
| `A9 01` | Reads one `f32` and calls `PlanetScreen.ResetTimer(seconds)` if the Planet Screen already exists. | Required only when a visible countdown is desired. Send after `A9 00`; an early update has no screen to reset. |
| `A9 02` | Reads one boolean, optionally applies its account/party side effect, then always selects state `12` (`ChainCashOut`). | Not a Continue acknowledgement. It routes to Cash Out regardless of the flag. |
| `B9` | The global dispatcher routes logical `57` / wire `0xB9` to `sub_536EE0`, which consumes the final objective-result snapshot. | Not needed to select or load 1-2. It may precede voting if authoritative objective values exist, but its retail order and 1-1 values remain unknown. |
| `AA` | No local receiver exists; see the negative proof below. | Never required and must not be fabricated. |
| `AB 00` | `cChainCashOutState` copies exactly 712 bytes and renders cashout XP/DNA/loot presentation. | Cash Out only. Sending it on Continue would present a grant and violate the at-risk reward boundary. |
| `8A` | Selects a game state from the standard 25-byte body. State `5` is PreDungeon and state `11` is Chain Voting. | A separate state-5 packet is permitted but not required: `B0` itself enters state `5`. Retail emission of a redundant `8A` is unproved. |
| `B0` | Reads exactly four dwords, resolves the first as a level noun, installs setup values, and selects state `5`. | Required load trigger and the smallest safe Continue response. |
| `B1` | Reads one dword, finalizes the level/player index, and selects Dungeon state `6` for ordinary chain play. | Required only after the status-8 ready boundary. |

### Why `0xAA` is not consumed

This is an exhaustive negative result for the canonical build-103 executable,
not an inference from a missing trace:

1. The table at `0x011841A0` contains 78 authored mappings. Entry `43` maps
   wire `0xAA` to logical ID `43` and the name
   `kGmsChainLevelResultsGMSMsgs`. Thus the opcode name is real.
2. Every incoming registration maps its logical ID through `sub_A8FDD0` at
   `0x00A8FDD0`. The complete decompiler output contains registrations for
   logical `42` (`A9`) in `cChainVotingState` and logical `44` (`AB`) in
   `cChainCashOutState`, but no call with logical `43`.
3. The common gameplay dispatcher `sub_53ADC0` at `0x0053ADC0` has cases for
   the globally consumed messages, including `48`/`49` (`B0`/`B1`), `57`
   (`B9`), and `58` (`B8`). It has no case `43`.
4. The A9 state callback `sub_449750` accepts only subtypes `0`, `1`, and `2`.
   The AB state callback `sub_4499D0` accepts only subtype `0`. There is no
   adjacent AA callback, fixed copy, reflection decode, or state field.
5. Every outbound message allocation is visible through `sub_A8FC00`; none
   allocates logical ID `43` either.

Therefore build 103 cannot derive a local transition, reward commit, or UI
body from `0xAA`. A retail server may have transmitted it for another build,
logging, or a removed listener, but this executable supplies no body contract
and does not need it to continue.

## Countdown interaction

`A9 01` reaches `sub_449290`, reads four bytes as a float, and calls
`sub_527950`. `sub_5278A0` stores the float in the Planet Screen and invokes
the exact ActionScript method `ResetTimer`. No native code decrements the
value, chooses a mission at zero, or emits a timeout request.

Consequences:

- `A9 00` must precede the first `A9 01` update;
- repeated `A9 01` packets are presentation-safe timer resets;
- an `AC 01` accepted before expiry cancels the server's voting deadline;
- receiving a late timer update after Continue must be suppressed by the
  result-phase/epoch check;
- the zero-time choice, party vote, and tie/disconnect policy are retail-server
  authority. The safe single-player fallback is Cash Out through the existing
  idempotent path, not an automatic Continue that risks an unrequested reward
  ledger.

The packaged Flash resource does not recover an authoritative network action
at timer zero. Native registration proves only that ActionScript can invoke
`AcceptMission` or `OnCashOut`; it does not prove which callback the retired
server expected on timeout.

## Exact 1-2 identity and first loading message

The authoritative runtime database is
`bin/darkspinner/darkspin/cache/content.db`, queried through
`bin/darkspinner/darkspin.toml`. `chain_level` rows for content resource `693`
begin:

| Ordinal | Row ID | Level ID | Reference |
| ---: | ---: | ---: | --- |
| `0` | `1` | `56` | `zelems_1.Level` |
| `1` | `2` | `60` | `zelems_3.Level` |

This proves the first-chain adjacency. The walkthrough independently names the
offered mission Gnarled Plateau 1-2. The case-insensitive FNV-1 identifiers
used by the executable are:

```text
zelems_3.Level          = 0xB23B6411
zelems_3_ai.Markerset   = 0x3D8377AE
```

`GamePrepareForStart` has this exact application layout:

```text
B0
11 64 3b b2             level = zelems_3.Level
ae 77 83 3d             markerset = zelems_3_ai.Markerset
01 00 00 00             single-player mask = bit 0
00 00 00 00             local level/player index = slot 0
```

The final two dwords are session values, not content-derived constants; the
shown values are the single-player slot-0 case. `sub_537E00` at `0x00537E00`
copies all 16 bytes, calls `sub_9C5FF0(levelHash)`, installs the resolved level,
markerset, and mask, stores the final index, and calls `sub_455A10(5)`. This is
the first packet whose receiver resolves and begins loading `zelems_3.Level`.

A preceding `GameState state=5` can display PreDungeon without supplying the
level identity. It cannot substitute for `B0`. Conversely, `B0` already
selects PreDungeon, so requiring both would be a server ordering invention.

## Player-ready handshake

The PreDungeon update `sub_52B490` drives the same `PlayerStatusUpdate (0x88)`
sender `sub_538C30` used by initial campaign entry. Each message is exact:

```text
88 <status:u32le> <progress:f32le>
```

Its relevant milestones are:

| Status | Client behavior | Server obligation |
| ---: | --- | --- |
| `2` | Publishes aggregate loading progress; the client can send intermediate fractions and finally `1.0`. | Treat as replayable progress. Reflect player state as needed; do not start gameplay. |
| `4` | Publishes the local load-complete boundary after level resources have been accepted. | Prepare/replicate the authoritative level and player setup. A repeated identical `B0` is tolerated by the current compatibility flow but is not newly proved as retail-required. |
| `8` | Publishes the final player/world-ready boundary after local construction. | Once per continuation epoch, send `B1` and enter Dungeon. Duplicate status `8` must replay no reward or setup mutation. |

`sub_537EC0` at `0x00537EC0` consumes `B1 <u32le>`, finalizes the index, and
selects state `6` for ordinary campaign play. `B1` is thus the start gate, not
the first loading message.

Darkspin's existing initial campaign handshake already follows the compatible
shape: it reflects status `2`, replies to status `4` with player/setup data,
and sends `B1` on status `8`. Continue only needs to replace the completed
session binding with `zelems_3` before reusing those handlers.

## Progression and reward ordering

Three ledgers must not be conflated:

| Ledger | Continue rule |
| --- | --- |
| Already accepted in-level pickups | Leave durable and do not recreate them from A9, B9, AA, or AB. |
| Permanent campaign progression | Monotonically record completed index `1` before exposing the irreversible `B0` load transition. Replays are no-ops. |
| At-risk chain reward offer | Carry under the same frozen result/chain epoch into 1-2; do not insert it into inventory, XP, or DNA on Continue. |

The safe authority transaction is:

1. authenticate the peer, result ID, generation, phase, Continue flag, selected
   squad ID, and exact offered level;
2. compare-and-swap the immutable result from `ChainVoting` to
   `ContinueReserved`; duplicate identical requests return the same outcome;
3. in one durable operation, advance `chain_progression` to at least `1` and
   record a continuation reservation keyed by the result ID, with next level
   `zelems_3` and the ungranted at-risk ledger;
4. create or replace the gameplay session for `zelems_3`, with a fresh runtime
   generation and no live 1-1 director, objects, timers, or scheduled work;
5. only after steps 3-4 succeed, send the deterministic `B0` response;
6. on status `8`, admit `B1` once for that generation.

This ordering prevents both bad outcomes: committing rewards before Continue
would duplicate a later cashout, while sending `B0` before durable reservation
could load 1-2 and then lose the only recoverable result state.

The client does not acknowledge a progression or reward commit. `B0`, status
`8`, and `B1` are loading/state boundaries only. Persistence success must be a
server precondition, not inferred from receipt of any of them.

## Failure, cancellation, and replay

- Invalid length, subtype, flag, squad ID, result ID, generation, or offered
  level: ignore without changing the frozen snapshot and keep serving the same
  A9 data on `AC 00`.
- Storage or new-session construction failure before `B0`: roll back the
  Continue reservation and remain in Chain Voting. Do not send a partial load
  response.
- Send loss after commit: retain the reservation and replay the same `B0`; do
  not create another result or advance progression again.
- Duplicate `AC 01`: return/replay the reserved transition only when its body
  matches; reject a conflicting squad/level choice.
- Duplicate statuses `2`, `4`, or `8`: update progress monotonically and make
  setup/start publication idempotent for the continuation generation.
- Disconnect before `B0`: the durable reservation may be resumed. Disconnect
  after `B0` must reconnect to the same `zelems_3` epoch or conservatively
  terminate the pending chain without granting its at-risk reward.
- Explicit Cash Out or timeout wins only while the snapshot is still in
  `ChainVoting`. Once Continue is reserved, late `AC 02`, `AC 04`, countdown,
  and old scheduled packets are stale and must be ignored.
- Build 103 exposes no Continue-specific cancel request and no native local
  rollback to the result screen. After `B0`, a server-side abort must use the
  normal game/session failure path; it cannot pretend the client never left.

The retired server's policy for 1-2 load failure, disconnect grace, and loss of
the at-risk reward is not recoverable. The conservative fallback is to preserve
durable 1-1 progression and pickups, never grant the pending chain reward, and
return the player to a recoverable ship/result state.

## Cash Out compatibility cross-check

The current Cash Out path provides the right safety pattern but commits at a
different boundary:

1. `AC 02` compare-and-swaps Chain Voting to Chain Cash Out and sends state
   `12`;
2. state entry automatically sends `AC 04`;
3. darkspin monotonically advances `chain_progression` before sending the
   proven `AF 00` return-to-ship message;
4. it emits no fabricated `AB`, item, DNA, or XP reward.

This path is replay-safe for progression and does not claim that the missing
reward was granted. Continue should share its immutable result ID, phase CAS,
monotonic progression, and storage-failure behavior. It must differ by carrying
the ungranted ledger into `zelems_3` and sending `B0` instead of entering Cash
Out or returning to ship.

The current result handler deliberately ignores the six-byte Continue payload
when `campaignResult` is active. The older compatibility branch outside that
result state already recognizes a six-byte ChainPlayer request and returns
`B0`; that is useful cross-check evidence for the receiver contract, but it
must not be reused without the transaction and new-session replacement above.

## Smallest safe server implementation

No new `B9`, `AA`, or `AB` encoder is needed for Continue. The minimum coherent
implementation is one result-phase transition plus reuse of existing entry
messages:

- decode exactly `AC 01 01 <u32le>`;
- validate the selected owned squad and frozen offer `zelems_3`;
- durably reserve `(resultID, userID, zelems_3, pendingLedger)` while advancing
  permanent progression monotonically;
- replace the peer's 1-1 runtime with a fresh 1-2 generation;
- return exact `B0(zelems_3, zelems_3_ai, mask, slot)`;
- reuse status `2/4/8` handling and send one `B1` at ready;
- make every boundary replayable and suppress all old epoch callbacks.

Until actual reward values exist, the pending ledger can be an explicit empty
reservation. It must still have a stable result ID and grant state so later
reward work cannot accidentally treat repeated Continue, Cash Out, or reconnect
as independent completions.

## Evidence references

- `bin/game/GameBin/Game.c`: `sub_449750`, `sub_449090`,
  `sub_449290`, `sub_5278A0`, `sub_527970`, `sub_527BF0`, `sub_527C80`,
  `sub_52B490`, `sub_537E00`, `sub_537EC0`, `sub_538C30`, `sub_53ADC0`,
  `sub_A8FAC0`, `sub_A8FC00`, and `sub_A8FDD0`.
- `bin/game/GameBin/Game.idb`: canonical database corresponding to
  those build-103 addresses; the decompiler output is sufficient for the
  contracts above.
- `notes/campaign/1-1/result-fields.md`: complete A9 and AB consumed-field maps.
- `notes/campaign/1-1/results.md`: Beam Out, snapshot, Cash Out, and durability
  boundary.
- `notes/campaign/1-1/completion.md`: mission completion and failure boundary.
- `notes/campaign/progression.md`: 72-entry chain and permanent map gate.
- runtime `content.db`, `chain_level.content_source_resource_id=693`: exact
  `zelems_1` to `zelems_3` adjacency.
- `server/campaign_result.go` and `server/gameplay_udp.go`: current compatibility
  path used only as a cross-check, not retail authority.
