# Reverse-engineering research roadmap

This note tracks research intended to reduce guesswork while reconstructing the
Game server for client build `5.3.0.103`. It coordinates static analysis,
asset evidence, controlled traces, and surviving gameplay footage. It is a
research plan, not confirmed protocol evidence; verified results belong in
`../confirmed/` and should be linked back here.

## Research target and available binaries

| Label | Version | SHA-256 | Role |
| --- | --- | --- | --- |
| Primary client | `5.3.0.103` | `3C7248A4C6614450290A66C22AB7FCDA86052B29A7B555E834D7C23F9B73EE5B` | Authority for addresses and client behavior. |
| Retail homolog | `5.3.0.127` | `949D29CE4815473E9D144B6EF13BD4CC54B43704B51D1A7E9DD23E12F9FC7FF3` | Cross-build function and class matching. |
| Locally patched 127 | `5.3.0.127` | `667B5F389DA5AA1A0F84513578B4A9F1CF1DB8DE172C4D7F9D04014D04D3B4AF` | Local-service behavior only; do not treat as a development build. |

`bin/game/dev/Game - Dev.exe` differs from the retail 127 executable by
197 bytes in 12 regions. Most differences replace production hosts with
`127.0.0.1`. The two code changes redirect the call at 127 VA `0x00C407E2`
from `0x00E15270` to `0x00426660` and change the comparison at VA
`0x00E4DB97` from zero to one. Its Authenticode signature no longer matches.
The adjacent IDA database currently contains automatic analysis rather than a
substantial set of developer or analyst function names.

Both the 103 and 127 executables contain the same set of 141 unique MSVC RTTI
type strings. The shared types include the Blaze, RakNet, TDF, and `nSporeNet`
transport classes most relevant to server reconstruction. This makes 127 a
useful structural homolog even though its absolute addresses are different.

## Evidence rules

Every conclusion should carry one of these labels:

| Label | Meaning |
| --- | --- |
| Binary-confirmed | Verified in the named executable and recorded with its hash and address or reproducible signature. |
| Trace-confirmed | Observed in a paired client/server trace with the action and relevant event ordering recorded. |
| Data-confirmed | Present in extracted build-compatible assets; extractor field names may still need validation. |
| Video-observed | Visible in surviving footage with source URL and timestamp, but not yet tied to a wire contract. |
| Cross-build matched | Transferred from another build using structural evidence; requires confirmation in 103 before becoming binary-confirmed. |
| Inferred | Best current interpretation of several facts; must state what would disprove it. |

Absolute addresses must never be copied between builds without an independent
match. Generic RakNet strings such as `ID_DATABASE_*` and `ID_SQLite3_*` do
not establish the schema or technology of the original Game server.

### Database-name hypothesis guardrail

The current name searches find `spApp::DatabaseProperty`-style text at build
103 VA `0x01008470` and build 127 VA `0x0100E9C8`. They also find the generic
RakNet `ID_DATABASE_*` message-name block at `0x01034800` in 103 and
`0x0103AF70` in 127. These are useful cross-build anchors, but neither result is
a physical server database, table, or column name. Until an xref reaches an
HTTP/TDF/GMS serializer or a persisted gameplay mutation, classify database
names derived from these strings or nearby vtables as `reference-only`.

## Current evidence inventory

- Build-103 IDA database: `bin/game/GameBin/Game.idb`.
- Build-127 IDA database: `bin/game/dev/Game - Dev.i64`.
- IDA query helpers: `scripts/reverse/`.
- Generated IDA diagnostics: `bin/game/logs/`.
- Paired client/server traces: `bin/server/darkspin/logs/traces/`.
- Extracted asset corpus: 13,503 XML files and 87 Lua files beneath
  `bin/server/data/`.
- Confirmed 103 foundation: `../confirmed/game-5.3.0.103-first-pass.md`.
- Confirmed progression work: `../confirmed/game-5.3.0.103-progression.md`.
- Gameplay reconstruction: `../game-design/progression-and-campaign.md`.
- Tutorial reconstruction and video ledger: `../tutorial.md`.
- Build-103 RakNet and GMS exchange: `raknet-gameplay-exchange.md`.
- Tutorial footage: `bin/video/walkthrough/0-tutorial.mkv`, indexed by
  `bin/video/walkthrough/list.md`.
- Tutorial contact sheets and selected frames:
  `bin/video/walkthrough/tutorial/`.

## Progress checkpoint: 2026-07-16

- **VIDEO-01 observation pass complete.** The full local 6:36 recording has
  been reviewed at broad and focused intervals. The timestamped mission ledger,
  SHA-256, evidence limits, and asset/Lua correlations are in
  `../tutorial.md`; 36 reusable PNG references are preserved beside the video.
- **A candidate Blaze setup sequence is trace-observed.** In
  `server-cookie-fix.jsonl` sequences `3691-3698`, the client sends GameManager
  reset-dedicated-server command `0x19`; the local server replies with `GID`,
  then sends notifications `0x0F`, `0x14`, and `0x15`. The client immediately
  sends remove-player command `0x0B`, followed by local notification `0x28`.
  This proves that build 103 parses enough of the candidate setup to react, but
  the immediate removal means the ordering and field set are not yet accepted
  as the original contract.
- **The current tutorial-named trace is a failure baseline, not a completion
  capture.** `server-tutorial-complete.jsonl` sequence `39` contains a
  GameManager join request `0x09`; sequence `40` rejects it with error
  `0x0002`. No connected gameplay RakNet or GMS exchange follows. Use this pair
  to drive IDA-01; do not cite it as evidence for the `3000` persistence write.
- **Static anchors exist, but cross-build transfer has not started.** Build 103
  has concrete GMS strings, handler xrefs, opcode ranges, and network-class
  names recorded below. The repository still lacks an RTTI/vtable exporter and
  a machine-readable 127-to-103 function correspondence table. The existing
  `103-functions-vftable.log` and `103-names-vftable.log` contain no matches;
  the generic helpers do not recover MSVC RTTI or vtable structure.
- **Hello identity has a reference-only candidate.** The current codec in
  `server/raknet/application.go` reads little-endian `uint64 UserID` followed
  optionally by little-endian `uint64 PlaygroupID`. This came from the
  ReCap/reference implementation, not a build-103 request serializer or live
  gameplay trace, so IDA-03 is `candidate` rather than confirmed.
- **HTTP and tutorial asset research are partial foundations.** Confirmed HTTP
  parser work is already recorded in the build-103 first-pass note. Tutorial
  Lua, level, reward, and trigger correlations are recorded in
  `../tutorial.md`, but they are not yet a general asset relationship index.
- **The tutorial completion snapshot is binary-confirmed.** Gameplay opcode
  `0xC8` (`TutorialGameMsgs`), subtype `0`, carries a signed 32-bit cumulative
  XP total. A positive value derives the account level and moves in-memory
  onboarding progress to `3000`; nonpositive input clears XP/level fields and
  returns progress to `2000`. The handler makes no persistence request, so
  return-to-ship ordering and durable save behavior remain IDA-06 work.
- **The first tutorial asset slice is now structurally mapped.** Five callback
  zones have IDs, coordinates, radii, once/server-only flags, and callback
  names; the teleporter destination, arena director, two horde listeners,
  fixed enemies, pickups, and obelisks are inventoried in `../tutorial.md`.
  The supposedly separate boss-arena layer is empty and unreferenced.
- **The second-creature mission primitive is binary-confirmed.** Native script
  registration maps `UnlockSecondCreature` to `sub_A050C0`, which increments a
  simulation/player field only up to two creatures. It does not call the
  account creature-unlock API, preserving persistent Sage ownership as a
  separate unresolved grant.
- **The build-103 RakNet opening is binary-confirmed.** It uses a single-stage
  `0x09 -> 0x0a` offline exchange, not the later `0x05-0x08` exchange currently
  implemented by darkspin. The request carries protocol `0x0d`, an eight-byte
  GUID, offline magic, target address, and padding. Twelve 500 ms attempts span
  UDP payload sizes `1464`, `1172`, and `548`. A valid reply is padded to the
  successful request size, after which the client sends reliable internal
  message `0x04`. The server validates its password and answers `0x0e`; the
  client completes the internal exchange with `0x11`. Those payloads, address
  arrays, timestamps, reliability values, and static anchors are recorded in
  `raknet-gameplay-exchange.md`; the reliability datagram/ACK envelope remains
  unresolved.

## Workstreams

### A. Cross-build symbol and structure recovery

The goal is a reusable 127-to-103 correspondence database, not two independent
sets of one-off IDA notes.

For every useful match, record:

```text
semantic name:
103 address:
127 address:
match confidence:
supporting strings, callers, callees, constants, RTTI, or vtable:
prototype and owning class:
open questions:
```

Recover MSVC RTTI object locators, class hierarchy descriptors, and vtables in
both databases. Apply names first to network ownership boundaries such as
`cClientSession`, `cProtocolTransport`, `cTransportRakNet`,
`cBlazeGameManager`, listener classes, and message callbacks. Vtables are most
valuable for class identity, inheritance, object layout, virtual dispatch, and
packet ownership. They are not expected to reveal the physical server database
schema.

Use structural matching to transfer candidates from 127 into 103, then confirm
each candidate using 103-local xrefs or instruction signatures. Avoid spending
time naming library code unless it directly bounds a game protocol path.

### B. Protocol-first IDA analysis

Static-analysis work should begin from a current gameplay blocker and end with
an implementable contract or a narrower experiment. The initial backlog is:

| ID | Priority | Question | Initial anchors | Required deliverable | Stop condition |
| --- | --- | --- | --- | --- | --- |
| IDA-01 | P0 | What exact Blaze notifications and TDF fields move a game through create/join, endpoint allocation, pre-game, and in-game? | Paired Blaze traces; GameManager component/command IDs; `cBlazeGameManager`; `GameManagerAPIListener`; TDF label strings. | Ordered notification table with component, command, direction, required fields, and client handler. | A minimal server sequence moves 103 to the RakNet connection attempt twice from clean processes. |
| IDA-02 | P0 | What connected RakNet handshake follows the single-stage `0x09/0x0a` offline opening? | Confirmed offline parser/builders; RakPeer connect/update/send paths; `cTransportRakNet`; socket trace timestamps. | Byte/bit layout, direction, reliability, ordering channel, and state transition for every handshake message. | A local responder reaches the first application packet repeatably. Static framing is substantially complete; retransmit/windows remain. |
| IDA-03 | P0 | What is the single 64-bit identity in build-103 `HelloPlayerRequest`, and how is it tied to Blaze game membership? | Confirmed 8-byte serializer and send call; virtual identity provider; Blaze user/game IDs in traces. | Identity provenance, validation, and response behavior; the wire width/order and send parameters are already confirmed. | Two local profiles join the same game with distinct, stable identities. |
| IDA-04 | P0 | What is the required initial replication order? | Object-create packet IDs; level, party, player, and world-object constructors; first in-game trace window. | Ordered dependency graph and minimum packet fixture for an empty level. | Client enters a level without timeout, crash, or missing-owner errors on two fresh runs. |
| IDA-05 | P1 | Which movement, physics, cooldown, hit, loot, and completion values are server-authored? | Send/receive dispatch tables; comparison branches; packet constructors; controlled one-variable traces. | Authority matrix identifying accepted, rejected, echoed, clamped, and derived fields. | Each first-vertical-slice packet has an explicit ownership rule backed by a trace or 103 code path. |
| IDA-06 | P1 | What end-game/result sequence safely returns the client to menus? | Completion/cashout/result UI strings; progression endpoints; packet and Blaze handlers near level teardown. | Ordered end-game messages and persistence checkpoints, including failure/reconnect behavior. | Completion, cashout, menu return, and relogin preserve state twice from clean processes. |
| IDA-07 | P1 | Which HTTP response and mutation fields remain required by build 103? | Confirmed parser functions in the first-pass note; XML field strings; endpoint traces. | Per-endpoint request/response schema with required, optional, and ignored fields. | All endpoints exercised by the first vertical slice parse without compatibility fallbacks. |
| IDA-08 | P2 | Which asset properties drive the first playable level and its AI/objectives? | Resource IDs in packet constructors; noun/class/level/objective names; Lua native lookups. | Asset-to-code xref table and typed property subset needed by simulation. | Every value consumed by the first level is sourced from a named asset or an explicitly documented server rule. |

#### Concrete build-103 anchors

Use these as starting points, not as conclusions:

- Connected RakNet transport: `cTransportRakNet`, the surviving
  `sntransport_transport_raknet.cpp` source-path strings, and the connection
  name table installed at `0x00AADCFB`. Connection result strings run from
  `ID_CONNECTION_REQUEST_ACCEPTED` at `0x01034100` through the disconnect,
  connection-lost, and incompatible-version strings near `0x01034210`.
  Transport-specific strings also occur near `0x0102EBBC-0x0102EC24`.
- Game hello/join: investigate opcodes `0x7F-0x86`. The
  `OnGmsHelloPlayer` xref at `0x00A8EBE7` is in the function beginning at
  `0x00A8EAB0`; the `OnGmsPlayerJoined` xref at `0x00A8ED02` is in the
  function beginning at `0x00A8ECB0`. Relevant names include
  `kGmsHelloPlayer` at `0x0102FCB0`, `kGmsReconnectPlayer` at `0x0102FCC0`,
  `kGmsPlayerJoined` at `0x0102FCF0`, and `kGmsPartyMergeComplete` at
  `0x0102FD04`.
- Blaze GameManager: begin with component `0x04`; commands currently under
  test include create `0x01`, join `0x09`, finalize `0x0F`, and reset
  dedicated server `0x19`. Candidate notifications include `0x0F`, GameSetup
  `0x14`, PlayerJoining `0x15`, player removed `0x1E`, state `0x64`, and mesh
  status `0x74`. Correlate these against the TDF labels already handled by
  `server/blaze/game_manager_component.go` rather than assuming the current
  server ordering is correct.
- Initial replication: follow the contiguous retail opcode/name table from
  `0x7F-0xCC`, especially GameState `0x8A`, ObjectCreate `0x8C`, QuickGame
  `0xAF`, PrepareForStart `0xB0`, GameStart `0xB1`, and ObjectivesInit
  `0xB7`. String anchors include `kGmsObjectCreate` at `0x0102FD90` and
  `kGmsGameStart` at `0x010300E8`.
- Movement and combat: prioritize ObjectPlayerMove `0x91`,
  LocomotionUpdate/Unreliable `0x94/0x95`, AttributeDataUpdate `0x96`,
  CombatantDataUpdate `0x97`, ActionCommandMsgs `0x9C`, PlayerDamage `0x9D`,
  ActionCommandResponse `0xA8`, and CooldownUpdate `0xC1`.
  `kGmsActionCommandMsgs` is at `0x0102FF00`; the action-response name is at
  `0x01030004`.
- Completion and persistence: prioritize LootSpawned `0x9E`, LootAcquired
  `0x9F`, ChainLevelResults `0xAA`, ChainCashOut `0xAB`, ChainGameOver
  `0xAE`, objective messages `0xB7-0xB9`, and LootDrop `0xCB`, then correlate
  their state transitions with `api.game.exitGame` and the following account
  refresh.

The first bounded foundation task is to map only the network, Blaze, and first
gameplay slice between 127 and 103. Do not wait for a whole-executable match
before beginning IDA-01 through IDA-03. Finish the P0 transport, hello/join,
and Blaze setup contracts before broad object, ability, or asset reversing.

For each IDA task, prefer this search order:

1. failing or missing trace event;
2. packet ID, TDF label, HTTP method, UI text, or error string;
3. xrefs to the parser, constructor, or dispatch table;
4. owning RTTI class and vtable;
5. callers, state predicates, and adjacent messages;
6. a controlled server response that confirms the interpretation.

### IDA tooling backlog

The existing helpers efficiently find strings, names, functions, xrefs,
callers, and nearby context. Add batch exporters only where they eliminate
repeated manual work:

None of the exporters below exists yet. The seven reusable scripts currently in
`scripts/reverse/` are generic search/context helpers, including the bounded
multi-function/caller/call-target dumper `ida_dump_functions.py`. Several
917-byte vftable logs contain only IDA startup/shutdown output, and automated
decompilation also failed when Hex-Rays was not loaded; neither should be
counted as recovered structure.

1. RTTI type descriptor to complete-object-locator, vtable, and method slots;
2. GMS opcode registration to handler address and owning class;
3. constant arguments at RakPeer `Connect` and `Send` call sites;
4. cross-build function fingerprints and match confidence;
5. a machine-readable evidence export in CSV or JSON.

Each exported row must distinguish `confirmed-103`, `matched-from-127`,
`reference-only`, and `trace-confirmed`. A renamed IDA function is intermediate
work; the stop condition remains a client-derived byte fixture or a reproducible
live state transition.

### C. Asset relationship index

Package extraction is already functioning. The next gain comes from connecting
the extracted records instead of merely producing more files. Build a searchable
index that relates:

```text
resource type/group/instance and symbolic name
  -> noun
  -> player or non-player class
  -> class attributes and abilities
  -> loot rigblock/prefix/suffix
  -> level, objectives, phases, markers, and server events
  -> executable constants and trace events
```

Prioritize opaque or incomplete records required by IDA-04, IDA-05, and IDA-08.
Record raw property IDs alongside extractor-generated names so an incorrect
field name cannot silently become a server rule.

### D. Controlled dynamic experiments

Change one server response or account variable per capture. Every experiment
must record the executable hash, account fixture, server revision, action
timeline, and paired trace filenames. Useful early variations include:

- empty versus populated account;
- one versus three squad members;
- different `chain_progression` and account levels;
- omitted versus present TDF fields;
- accepted versus rejected create/join responses;
- packet reliability and ordering-channel variations;
- zero versus nonzero mission rewards;
- each candidate end-game packet enabled independently.

Add short timeline markers such as `selected 1-1`, `loading appeared`,
`first world object appeared`, `final enemy died`, and `cashout appeared`.
The trace diff should be centered on those markers.

### E. Campaign video evidence

Campaign footage can establish player-visible ordering and semantics that are
not obvious from strings or assets. Prefer uncut, high-resolution footage with
minimal overlays. Record each source before analysis:

The current research focus is the tutorial. Complete the tutorial observation
ledger and correlate its transitions before expanding video analysis to the
rest of the campaign.

| Source ID | URL | Coverage | Quality | Status | Notes |
| --- | --- | --- | --- | --- | --- |
| VIDEO-01 | [Game: Complete Story Playthrough - Part 0: Tutorial](https://www.youtube.com/watch?v=wy0nroTZaBo) | Tutorial, 6:36 | Local 1920x1080 MKV with audio; no captions | Observation pass complete | Timestamped lesson-order analysis and reference frames are recorded in `../tutorial.md`; the click on `RETURN TO SHIP` and subsequent persistence remain uncaptured. |

For every useful observation, record:

| Source ID | Timestamp | Visible action/state | Candidate asset or field | Correlated trace/IDA path | Confidence |
| --- | --- | --- | --- | --- | --- |

Target observations include:

- tutorial and onboarding state transitions;
- mission, threat, planet, and difficulty ordering;
- create/join/loading and return-to-menu sequences;
- objective activation and completion order;
- account level, XP, DNA, hero choices, and unlock timing;
- loot presentation, inventory changes, cashout, and risk/reward choices;
- squad and equipment mutations;
- disconnect, retry, defeat, and reconnect behavior;
- UI labels that give semantic names to otherwise opaque fields.

Video evidence proves visible behavior, not packet encoding. Promote a
video-observed claim only after assets, build-103 code, or a controlled trace
supplies the implementation evidence.

## Research tracker

Use these statuses: `queued`, `active`, `blocked`, `candidate`, `confirmed`, or
`superseded`.

| Work item | Status | Owner | Evidence/output | Next action |
| --- | --- | --- | --- | --- |
| Classify the patched 127 executable | confirmed | Codex | Binary diff summarized above | Preserve as local-patch evidence; do not call it a dev build. |
| Inventory RTTI across 103 and 127 | confirmed | Codex | 141 identical unique type strings | Recover object locators and vtables in both IDBs. |
| Recover RTTI object locators and vtables | queued | | Shared RTTI strings and existing string/name helpers | Implement the bounded exporter described in the tooling backlog; start with transport and GameManager types. |
| Build 127-to-103 function correspondence index | queued | | Shared RTTI inventory and build-103 anchors; no transferred rows yet | Match the IDA-01 through IDA-03 network functions in 127 and emit the first evidence rows. |
| Create asset relationship index | active | | Tutorial Lua/level/reward correlations in `../tutorial.md` | Turn the tutorial subset into machine-searchable rows retaining raw property IDs. |
| Analyze VIDEO-01 tutorial footage | confirmed | Codex | Local video, SHA-256, timestamp ledger, and 36 PNG references | Preserve as the visible-behavior baseline; reopen only if a better or longer source appears. |
| Correlate tutorial triggers with protocol and persistence | active | | Five exact callback zones, native `UnlockSecondCreature`, authored teleporter/horde topology, video ledger, and binary-confirmed `0xC8` subtype-0 completion snapshot in `../tutorial.md` | Recover the remaining callback bodies/consumers and `0xC8` framing; later capture Sage, loot, return, and account persistence when execution is available. |
| Resolve IDA-01 Blaze game-state sequence | active | | Candidate `0x19 -> 0x0F/0x14/0x15 -> 0x0B/0x28` trace; join `0x09` error `0x0002` failure baseline | Diff the candidate reset trace against a clean run, then trace the client predicate that chooses remove-player. |
| Resolve IDA-02 connected RakNet handshake | active | Codex | Trace-confirmed `0x09`; binary-confirmed `0x09/0x0a`, internal `0x04 -> 0x0e -> 0x11`, reliability enum, data/ACK/NACK headers, ranges, encapsulated fields, and first GMS send in `raknet-gameplay-exchange.md` | Recover retransmit/windows, split limits, optional ACK feedback, and exact ACK boundary around `0x11 -> 0x82 -> 0x7f`. |
| Resolve IDA-03 player hello identity | active | Codex | Build-103 serializer confirms one raw LE `uint64`, exact 8-byte payload, and priority `0`/reliability `3`/channel `1`; optional playgroup ID is reference-only | Map the virtual identity-provider result to a Blaze field, then validate it with two captured profiles. |
| Resolve IDA-04 initial replication order | queued | | GMS opcode/name table anchors below | Begin only after IDA-02 reaches the first application packet. |
| Resolve IDA-05 gameplay authority | queued | | Video semantics and movement/combat opcode anchors only | Defer packet ownership experiments until a minimal replicated level loads. |
| Resolve IDA-06 end-game/result sequence | active | | `TutorialGameMsgs` `0xC8` subtype `0` and its cumulative-XP/progress mutation are binary-confirmed; `LABS_TUTORIAL_COMPLETE` also selects survey classification `1` in `cSurveySystem`, not a result-controller state; visible `RETURN TO SHIP` prompt | Live-confirm whether another result/exit event follows `0xC8`, then recover the button/teardown/account-refresh order and RakNet reliability. |
| Resolve IDA-07 HTTP schemas | active | | Endpoint/parser tables in `../confirmed/game-5.3.0.103-first-pass.md` | Convert first-vertical-slice endpoints into required/optional/ignored field tables. |
| Resolve IDA-08 first-level asset inputs | active | | Exact callback markers, teleporter/horde graph, 27 fixed enemies, 10 pickups, obelisks, and video correlations in `../tutorial.md` | Materialize searchable rows with resource/raw property IDs and link callbacks and state changes to GMS consumers. |

## Immediate execution queue

1. Drive IDA-01 from the known failure boundary: explain why build 103 sends
   remove-player after the candidate `0x19` setup sequence and why the separate
   join attempt references a missing game. Extend redacted tracing to retain
   the selected non-sensitive GameManager TDF values needed for the diff.
2. Add the GMS opcode-registration exporter, anchored first at HelloPlayer,
   ObjectCreate, GameStart, and TutorialGameGMSMsgs.
3. Continue IDA-02 beneath the confirmed internal `0x04 -> 0x0e -> 0x11`
   exchange: recover retransmission/windows, split limits, optional ACK
   feedback, and the exact boundary around server `0x82` and client `0x7f`.
4. Complete IDA-03 by mapping the confirmed 64-bit build-103 hello value to
   its Blaze identity source and validation rule.
5. Trace the `0xC8` sender in the dev/server binary or generic broadcast path,
   then recover RakNet framing/reliability and its ordering around horde
   completion and `RETURN TO SHIP`; locate native consumers for the three
   server-only tutorial callbacks.
6. Materialize the tutorial asset subset as the first asset relationship-index
   fixture, retaining resource keys and raw property IDs.
7. When client execution is available again, capture before/after account
   state for Sage, both item grants, horde completion, and return-to-ship.

## Contributions that produce the largest gains

In approximate value order:

1. Original packet captures, HTTP proxy logs, Blaze logs, account snapshots, or
   crash dumps from the official service era.
2. PDB, MAP, symbol-server remnants, QA/beta/demo executables, or truly distinct
   development builds.
3. An annotated build-103 IDA database with semantic names, types, comments, or
   structures beyond automatic analysis.
4. Complete campaign and onboarding video URLs, preferably uncut and at the
   highest available resolution.
5. Reproducible controlled captures with one changed variable and timestamped
   user actions.
6. Screenshots or saves showing before/after account, creature, inventory,
   squad, campaign, and reward state.

Before adding an artifact, record its origin, build/version, SHA-256 when
applicable, and whether it may contain user credentials or other private data.
