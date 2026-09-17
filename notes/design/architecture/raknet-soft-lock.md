# RakNet application-handler soft lock

## Executive finding

The 2026-07-25 failure is a server-side packet-processing stall, not a RakNet
sequence rejection and not an application action lock that first occurred in
the build-103 client.

The client received and dispatched every server application message through
outbound datagram `0x2fe`, acknowledged the complete `0x2f3..0x2fe` range, and
then continued making successful UDP `sendto` calls to
`127.0.0.1:42127` for 19.5 seconds. The server logged none of those later
datagrams. Its shared UDP loop processes packets synchronously, and the RakNet
entry point holds `outboundMu` while it runs arbitrary application handlers.
Scheduled producers also run arbitrary application code while holding the same
mutex. Schedule cancellation synchronously waits for the scheduler goroutine
to finish. These paths permit a lock cycle in which the UDP reader holds
`outboundMu` and waits for a scheduled goroutine which itself needs
`outboundMu`, or simply stalls inside an application handler/producer while
holding the transport lock.

The existing capture does not identify the exact blocking callback or the
first post-boundary client datagram because server ingress is logged only
after synchronous processing completes and the Fang socket trace records a
digest rather than packet bytes. It does, however, locate the failure boundary
with high confidence:

## Implementation status

The immediate deadlock cycle is corrected: inbound application handlers and
scheduled producers no longer execute under `outboundMu`, and ordinary
schedule cancellation signals and returns without joining the producer
goroutine. A focused regression test blocks a scheduled producer and verifies
that cancellation and inbound ACK processing continue.

Per-peer bounded dispatch, watchdog diagnostics, and controlled campaign
rejoin remain follow-up resilience work. They are not required to break the
proven cancellation/outbound-lock cycle, but remain valuable protection
against an independently wedged gameplay handler.

- the operating system continued accepting client UDP sends;
- the build-103 RakNet receive and scene-dispatch paths were alive;
- the final server datagrams were fully received and acknowledged;
- the server stopped completing its serial `processPacket` path;
- shutdown later timed out while attempting to close the server.

The primary correction is therefore to remove application work and blocking
schedule cancellation from the RakNet outbound critical section. A Fang
client-state recovery hook cannot release this server stall. It should be
considered only after server-side isolation/reconnect exists, and only if a
controlled server resynchronization still cannot clear the client's local
combat state.

## Evidence and scope

This analysis used:

- `bin/darkspinner/darkspin/logs/darkspinner.log`, including the newest failed
  exchange and the nearest successful campaign exchanges;
- `bin/darkspinner/darkspin/logs/traces/game.jsonl`, including raw socket,
  client scene-message, action, reset, and shutdown events;
- the complete current `server/raknet` implementation and its shared UDP
  caller in `server/udp/server.go`;
- the canonical decompiler output
  `bin/game/GameBin/Game.c`;
- the current Fang observational hooks in `app/fang/fang.c`;
- read-only Git history/blame for the relevant RakNet locking and scheduling
  paths.

No production code, game binary, game asset, runtime database, or Git state
was modified.

## Exact terminal exchange

There are two clocks in the evidence. Server log times are wall-clock local
times. Fang `time_ms` is the client's monotonic process clock.

### Server-observed boundary

| Server time | Direction | Datagram/application event |
|---|---|---|
| `09:07:08.5949829` | client -> server | Movement accepted from `(-152.72073,45.319874,-0.012229472)` toward `(-145.71637,47.816154,-0.012230163)`. |
| immediately after | server -> client | ACK of inbound client datagram sequence `0x44`: `c0000101440000`. |
| immediately after | server -> client | `seq=0x2f3`, reliable index `0x2f3`, ordered index `0x2ee`, channel 1, application `0x91`, hero object 1 movement. Prefix/full logged packet begins `84f30200600288f30200ee02000191010000000100000064b711c3be433f42066148bc`. |
| immediately after | server -> client | `seq=0x2f4`, reliable index `0x2f4`, ordered index `0x2ef`, channel 1, application `0x95`, hero object 1. Packet begins `84f40200600088f40200ef020001950100000064b711c3be433f42066148bc`. |
| immediately after | server -> client | `seq=0x2f5`, reliable index `0x2f5`, ordered index `0x2f0`, channel 1, application `0x91`, object 35 stop/state update. Packet begins `84f50200600288f50200f0020001912300000020000000e04916c379e831429a9bd93e`. |
| before `09:07:08.609` | server -> client | Scheduled `seq=0x2f6..0x2fb`, decoded below from the client scene trace. |
| `09:07:08.609` | client -> server | ACK range `0x2f3..0x2fb`: `c0000100f30200fb0200`. |
| before `09:07:08.631` | server -> client | Scheduled `seq=0x2fc..0x2fe`, decoded below. |
| `09:07:08.631` | client -> server | ACK range `0x2fc..0x2fe`: `c0000100fc0200fe0200`. |
| afterward | client -> server | No later datagram completed the server's synchronous processing/logging path. |

The client trace supplies the otherwise unlogged application sequence. Each
application message occupies one reliable ordered datagram and the receive
sizes match RakNet framing (`31`, `40`, or `95` bytes):

| Datagram | Client dispatch time | ID | Object | Meaning |
|---|---:|---:|---:|---|
| `0x2f3` | `135503531` | `0x91` | 1 | accepted hero movement |
| `0x2f4` | `135503531` | `0x95` | 1 | hero position/state companion |
| `0x2f5` | `135503531` | `0x91` | 35 | flags `0x20`, stop/state update |
| `0x2f6` | `135503531` | `0x95` | 36 | position/state companion |
| `0x2f7` | `135503531` | `0x91` | 36 | flags `0x42`, attack update |
| `0x2f8` | `135503531` | `0xA5` | 36 | state hash `0x34452778` (`zlm_minn_tc_2_attack1`) |
| `0x2f9` | `135503531` | `0x95` | 34 | position/state companion |
| `0x2fa` | `135503531` | `0x91` | 34 | flags `0x42`, attack update |
| `0x2fb` | `135503531` | `0xA5` | 34 | state hash `0x34452778` (`zlm_minn_tc_2_attack1`) |
| `0x2fc` | `135503546` | `0x95` | 32 | position/state companion |
| `0x2fd` | `135503546` | `0x91` | 32 | flags `0x42`, attack update |
| `0x2fe` | `135503546` | `0xA5` | 32 | state hash `0x90c6842f` (`nomad_lieu_tc_3_attack`) |

The raw client socket trace confirms the wire sequence:

1. At trace line 18750 (`time_ms=135503515`) the final accepted movement was a
   successful 79-byte `sendto`.
2. Lines 18752-18761 show the server ACK plus the first nine data datagrams
   arriving on socket 3540.
3. Lines 18763-18771 show all nine application messages entering the client
   scene dispatcher.
4. Line 18776 is the successful 10-byte ACK for `0x2f3..0x2fb`.
5. Lines 18780-18782 show the remaining three datagrams arriving.
6. Lines 18784-18786 show those three messages entering the scene dispatcher.
7. Line 18789 is the successful 10-byte ACK for `0x2fc..0x2fe`.
8. At line 18925 (`time_ms=135504031`), 485 ms after the final ACK, the client
   resumed successful gameplay-sized sends. They continue through line 24067
   (`time_ms=135523531`).

Across that 19.5-second interval the client made 968 successful OS-level
`sendto` calls on the same socket to the same loopback endpoint:

| UDP size | Count |
|---:|---:|
| 16 | 4 |
| 79 | 49 |
| 99 | 600 |
| 154 | 145 |
| 174 | 170 |

Every recorded result equals the requested byte count and every socket error
is zero. Although successful `sendto` alone cannot prove application receipt,
loss of all 968 loopback datagrams is not a credible packet-loss explanation.
The evidence instead means that the server either read the first later
datagram and blocked before its post-processing log, or was already blocked by
a scheduled publisher before returning to `ReadFromUDP`.

## Successful campaign comparisons

The nearest successful campaign 1-1 trace crosses the same encounter boundary
without a protocol discontinuity. On 2026-07-24:

- a movement around log line 115640 produced/retired outgoing sequences
  `0x297..0x2b1`;
- movement from approximately `(-152.52045,22.445206)` toward
  `(-148.31163,39.622772)` produced ACKs over `0x2b2..0x2be`;
- a subsequent movement was accepted at line 115686 and play continued.

The packet burst shape is the same: an accepted move followed by position,
movement, and enemy activation/attack messages. Therefore the mere presence
of the three `0x91`, three `0x95`, and three `0xA5` enemy messages is not an
abnormal client-wait condition.

A 2026-07-23 successful trace is also decisive for the apparent low-byte
boundary. Around log lines 109990-110025, the client acknowledged:

- ranges through `0x2f3`, then `0x2f4..0x2f6`;
- the exact range `0x2fc..0x2fe`;
- singleton `0x2ff`;
- range `0x300..0x302`;
- further datagrams followed by another accepted movement.

Thus `0xfe -> 0xff -> 0x00` in the low byte is normal. RakNet sequence numbers
are 24-bit triads; `0x2fe` is nowhere near the actual
`0xffffff -> 0x000000` wrap.

## Transport audit

### ACK/NACK range decoding

`server/raknet/datagram.go:195-225` reads ACK/NACK record counts in big endian,
then reads each single/range endpoint as a little-endian triad. It rejects a
descending range and caps expansion at 65,535 entries
(`server/raknet/datagram.go:218`). Both terminal ACKs are ordinary one-record
ranges:

- `c0 0001 00 f30200 fb0200` = `0x2f3..0x2fb`;
- `c0 0001 00 fc0200 fe0200` = `0x2fc..0x2fe`.

The decoder does not represent one range that crosses the full 24-bit wrap,
but the sender can encode that as separate records and no such wrap occurs
here. There is no NACK at the boundary.

### Datagram sequence progression and receive window

`server/raknet/receive.go:54-150` uses 24-bit modular forward distance, a
half-range comparison, and a maximum accepted gap of 4,096. The failed
session's inbound datagrams were sequential through client datagram `0x44`.
There is no evidence of a large jump, stale half-range ambiguity, or receive
window exhaustion. The current session had also advanced and acknowledged
server sequences continuously from approximately `0x291` through `0x2fe`.

The first post-boundary raw bytes were not captured, so it is impossible to
prove its client datagram sequence directly. It would nevertheless have been
near `0x45`, not a wrap or receive-window edge. If the receive window had
rejected a duplicate or stale datagram normally, `HandleAndWriteDatagrams`
would have returned and the shared server would have logged later packets.
That is not what occurred.

### Reliable indexes and ordering channels

The final server datagrams use reliable message indexes matching their
datagram sequence (`0x2f3..0x2fe`) and monotonically increasing ordered
indexes on channel 1. Reliable duplicate filtering is implemented by
`receiveSequenceWindow.accept` at `server/raknet/receive.go:211-240`;
ordered delivery by `receiveState.acceptOrdered` at
`server/raknet/receive.go:243-288`; sequenced delivery by
`receiveState.acceptSequenced` at `server/raknet/receive.go:290-310`.

The client scene trace proves these packets cleared its RakNet ordering layer
and reached application dispatch in order. A missing reliable index or held
ordered message therefore did not leave build 103 waiting at `0x2fe`.

### Split reassembly

Split validation/reassembly is at `server/raknet/receive.go:312-390`. None of
the terminal inbound or outbound packets is split; all are far below MTU.
There is no incomplete split assembly at the boundary.

### Retransmission retirement

Outbound retention is at `server/raknet/retransmit.go:63-85`; ACK retirement
at `server/raknet/retransmit.go:87-104`; timeout collection at
`server/raknet/retransmit.go:137-187`; and retransmission writing at
`server/raknet/retransmit.go:189-218`.

The two client ACKs name every sequence `0x2f3..0x2fe`. Retirement deletes
exact sequence keys from the peer's pending map. There is neither a gap nor a
NACK that could cause the server to withhold an expected datagram. The client
received all twelve messages before it sent the ACKs, independently proving
that premature retirement did not suppress delivery.

### Connected-state enforcement

Peer attachment/state ownership is at `server/raknet/server.go:109-192`;
connected datagram dispatch at `server/raknet/server.go:469-505`; and
connected payload enforcement at `server/raknet/server.go:508-585`. Campaign
traffic, movement, Lightning Rogue Basic, and the terminal movement were all
accepted on the established peer. There is no disconnect, invalid-state
response, or address change at the boundary.

If the next packet had been rejected by the normal connected-state guards,
the handler would have returned an error or response and the shared loop would
have continued. Total loss of subsequent server logging is inconsistent with
a clean connected-state rejection.

## Responsible concurrency paths

### Shared UDP serialization

`server/udp/server.go:90-100` performs `ReadFromUDP` and immediately processes
that packet on the same goroutine. `processPacket` at
`server/udp/server.go:141-184` synchronously invokes RakNet handling. Packet
logging occurs only after this call returns. One blocked packet therefore
stops reads and logs for this endpoint and, if the socket is shared, delays
unrelated traffic too.

`SharedServer.Close` at `server/udp/server.go:243-257` detaches the RakNet
connection. The failed run logged a close timeout five seconds into shutdown
and a server shutdown timeout five seconds later. This is consistent with a
goroutine that never released or could never acquire the RakNet outbound
mutex. Earlier runs contain some shutdown timeouts, so this corroborates the
live boundary but cannot identify its exact owner by itself.

### Inbound processing under the outbound mutex

`Server.HandleAndWriteDatagrams` at `server/raknet/server.go:395-442` locks
`outboundMu` before it calls the full `HandleDatagrams` path. That path
decodes, updates receive state, dispatches connected payloads, and invokes
application handlers. The lock is therefore not merely a packet-encoding or
socket-write lock; arbitrary gameplay behavior executes inside it.

`AttachConnection` also needs the same mutex
(`server/raknet/server.go:170-192`), explaining why a wedged owner or waiter
can prevent clean shutdown.

### Scheduled application producers under the same mutex

Schedule setup is at `server/raknet/server.go:656-743`.
`produceAndSendScheduledDeadline` at `server/raknet/server.go:766-827`
acquires `outboundMu` near line 771, then invokes `producer.Produce()` at line
800 while still holding it. Any producer that blocks, re-enters RakNet,
acquires a gameplay lock in the opposite order, or waits on cancellation
stops inbound UDP progress.

The current retransmission goroutine also legitimately uses the outbound
critical section (`server/raknet/retransmit.go:189-218`). That is not evidence
of a retransmission bug, but it means the recently hardened transport has
another concurrent contender whose progress depends on this mutex remaining a
short wire-state lock.

### Synchronous schedule cancellation

The `CancelSchedule` closure at `server/raknet/server.go:740-743` calls the
context cancellation function and then blocks on `<-done`. The scheduled loop
can need `outboundMu` before it observes cancellation and exits
(`lockScheduledOutbound`, `server/raknet/server.go:830-847`).

This permits the concrete cycle:

```text
UDP reader/application handler
  holds outboundMu
  -> invokes gameplay Stop/CancelSchedule
     -> waits for scheduler done

scheduler
  -> waits for outboundMu
  -> cannot finish and close done
```

The same design also permits an application producer to stall directly while
holding `outboundMu`. For example, scheduled melee behavior retains and
cancels schedules in `server/raknet/melee.go:187-200`; multiple gameplay run
types follow this ownership pattern. The trace lacks the phase/owner markers
needed to name which run was involved.

Read-only history shows the synchronous cancellation join and
producer-under-`outboundMu` pattern originated in the recent RakNet
scheduling/refactor work (commit `fcee2f4`, 2026-07-19). Subsequent hardening
added sound receive/retransmission machinery but retained the over-broad
critical section. The hardening is relevant as context, not evidence that ACK
decoding or sequence arithmetic regressed.

## Client-side determination

The build-103 client did not enter an initiating application wait after
receiving a malformed terminal sequence:

- `Game.c:1543036-1543101` (`sub_A8A1F0`) repeatedly drains RakPeer
  receive results, identifies application IDs, invokes `sub_A8A400`, and
  releases each packet.
- `Game.c:1543105-1543167` (`sub_A8A400`) constructs the scene message
  and invokes its handler.
- Fang's `message_receive` events prove all twelve final messages crossed this
  boundary.
- `Game.c:1549528-1549555` (`sub_A95530`) is the socket receive wrapper.
- `Game.c:1549564-1549590` (`sub_A956D0`) is the socket send wrapper.
  Every later send in the Fang trace comes from RVA `0x695725` inside this
  wrapper and succeeds.

The action gate at `Game.c:305267-305337` (`sub_4D5980`) can report a
blocked response state until a deadline, and the action response path at
`Game.c:305939-306121` can install or clear accepted/rejected/released
state by sync token. That is not the initiating failure here:

- the preceding Lightning Rogue Basic release received `0xA8` response type 4
  for sync 10;
- Fang's following action snapshot showed active, response, and queued state
  all clear;
- movement after that action was accepted;
- the client continued transmitting gameplay-sized traffic after server
  responses stopped.

The later `/reset` trace at `time_ms=135556906` shows the UI and local command
path were still alive. Fang cleared local pending/action state, but the reset
could not affect a server whose only UDP reader was wedged. The visible
client-side limbo is consequently a downstream failure mode: local actions
continue or queue while no authoritative response can arrive.

## Root-cause verdict

| Hypothesis | Verdict | Confidence | Basis |
|---|---|---:|---|
| Server stopped completing the read/process loop | Yes | Very high | 968 later successful loopback sends, no completed server packet logs, synchronous shared read/process loop |
| Server incorrectly retired a needed datagram | No | Very high | client received and dispatched all `0x2f3..0x2fe` before ACKing them |
| Server withheld a datagram because of NACK/retransmission state | No | High | complete ACK coverage, no NACK/gap, all terminal messages delivered |
| ACK/NACK range or low-byte wrap was rejected | No | Very high | exact `0x2fc..0x2fe` range and `0x2ff -> 0x300` succeed in comparison trace |
| Reliable/order/split receive state rejected the next valid sequence | No evidence; unlikely | High | ordinary sequential state; a clean rejection would return and permit later logs |
| Server deadlocked or stalled in/behind `outboundMu` | Yes, structural root cause | High | application handlers and scheduled producers run under mutex; cancellation waits for a scheduler that can need same mutex; shutdown corroborates |
| Exact callback/lock owner is known | No | — | no pre-handler ingress log, phase markers, or goroutine dump |
| Server sent a terminal sequence that left build 103 waiting | No | Very high | build 103 dispatched all messages and continued raw gameplay sends |
| Client application action lock initiated the failure | No | High | prior action state cleared; movement accepted; transport sends continued |
| Client entered brittle limbo after loss of authority | Yes, consequence | High | local reset/UI alive while no authoritative responses returned |

The highest-confidence root cause is therefore **an over-broad RakNet
outbound critical section combined with synchronous schedule cancellation,
which allows the server's sole UDP processing goroutine to stall or
deadlock**. The most plausible immediate trigger is the collision between an
incoming ability/cancel path and one of the enemy attack producers scheduled
at the terminal movement boundary. The first later client send occurs 485 ms
after the final ACK and is 99 bytes, consistent with ability-sized/retry
traffic, but the capture does not contain its bytes and this trigger remains a
hypothesis rather than a proven callback identity.

## Minimal corrective design

### 1. Restrict `outboundMu` to outbound wire state

No application callback may execute while `outboundMu` is held.

For scheduled sends:

1. validate/copy the peer and producer deadline state;
2. call `producer.Produce()` outside `outboundMu`;
3. acquire `outboundMu`;
4. revalidate the peer generation/address;
5. allocate datagram, reliable, and ordered indexes; encode;
6. retain retransmission state and perform the contiguous UDP writes;
7. release the mutex before logging, callbacks, or failure notification.

For inbound sends, split the present combined path:

1. decode, validate receive state, and serialize application handling outside
   the outbound wire mutex;
2. have handlers return raw application payload intents, not mutate RakNet
   counters or write the socket;
3. acquire `outboundMu` only to encode those payloads against current peer
   counters, retain them, and write them.

Inbound application ordering still needs serialization. Use a distinct
per-peer processing mutex or bounded per-peer worker queue for that purpose.
Do not overload the outbound counter/socket mutex with gameplay ownership.

### 2. Make ordinary schedule cancellation non-blocking

`CancelSchedule` should be idempotent and should signal cancellation, then
return immediately. If callers genuinely need a join, expose a separate
`Done`/`Wait` operation used only by shutdown and focused tests. Never wait for
it while holding RakNet, gameplay-run, entity, or scheduler locks.

The current scheduled loop already observes group cancellation and
`lockScheduledOutbound` already aborts when its context is canceled. A
synchronous join is unnecessary for normal `Stop` semantics.

### 3. Isolate a wedged peer from the socket reader

Copy each received datagram into a bounded per-peer queue and let a per-peer
worker serialize receive-window and application processing. The shared socket
reader must remain able to receive, timestamp, and route later datagrams even
if one peer's application handler stalls.

This also makes recovery observable:

- record `lastIngress`, `handlerStarted`, `handlerCompleted`, and peer
  generation;
- if a handler exceeds a short watchdog threshold while ingress continues,
  cancel that peer's gameplay generation and schedules without waiting on the
  stuck worker;
- start a fresh generation only after atomically detaching the old worker so
  stale results cannot encode against new counters.

This queue should be bounded. Overflow should disconnect/rejoin one peer with
a clear diagnostic rather than exhaust memory or stall the shared reader.

### 4. Add a controlled server recovery transaction

Once ingress is isolated, recovery should be authoritative rather than a
blind sequence/token sweep:

1. cancel the affected gameplay generation and schedules non-blockingly;
2. invalidate stale handler/producer output with a peer generation number;
3. emit stop/state snapshots for server-known active entities;
4. send a matching `0xA8` rejection/cancel only for server-known outstanding
   action syncs;
5. if in-place resync is not protocol-safe, explicitly disconnect and permit
   campaign rejoin while preserving the durable campaign checkpoint.

The normal chat `/reset` path is insufficient as an emergency boundary
because it is itself gameplay traffic behind the wedged worker. A transport
recovery control, if added, must be recognized before gameplay dispatch and
must only request the controlled transaction above. It must not reset RakNet
indexes in place or guess client action tokens.

### 5. Fang only as a last-resort client escape

Fang cannot repair a server-wide `outboundMu` stall. A compatibility hook is
useful only after the server can isolate a peer and accept reconnect/rejoin.
If testing then proves that a controlled server disconnect/resync still leaves
build 103's local combat gate wedged, a narrow Fang fallback may:

- detect a sustained interval in which UDP sends succeed but neither UDP
  receives nor scene messages arrive;
- clear the same local combat/action descriptors used by the existing manual
  reset;
- request an explicit reconnect/rejoin rather than continuing to issue
  gameplay actions into limbo;
- display a recoverable connection-state indication.

It should not fabricate accepted actions, suppress native gameplay decisions,
or reset on ordinary packet jitter. Triggering should require server recovery
support or a long no-authority threshold and should be rate-limited.

## Focused instrumentation required to prove the immediate trigger

The current capture proves the architectural failure class but cannot name
the exact callback. Add the following diagnostic instrumentation before
changing behavior:

1. **Pre-processing UDP ingress:** immediately after `ReadFromUDP`, before
   `processPacket`, record monotonic ingress ID, endpoint, length, first byte,
   and a capped local-debug hex prefix. This distinguishes “not read” from
   “read and blocked.”
2. **RakNet phases:** record ingress ID at `outboundMu` wait, acquisition,
   handler begin/end, encode begin/end, write begin/end, and release. Include
   peer generation and application ID.
3. **Scheduled producer phases:** assign schedule-group and producer IDs;
   record deadline, producer begin/end, lock wait/acquire, encode/write, cancel
   requested, cancel returned, and group done.
4. **ACK retirement:** record parsed ranges and pending-retransmission counts
   before/after retirement, including sequence keys not found.
5. **Lock watchdog:** if an outbound wait/hold or application handler exceeds
   250 ms, report the operation/ingress/group owner IDs and capture all
   goroutine stacks with `runtime.Stack` into
   `bin/server/darkspin/logs/traces`. Repeat sparingly with rate limiting.
6. **Shutdown watchdog:** capture stacks before the existing five-second close
   timeout, not after the evidence has been discarded.
7. **Fang wire diagnostics:** for local diagnostic builds, record a capped raw
   RakNet prefix rather than only the digest, parse datagram/reliable/order
   indexes and application ID, and add dispatcher exit events around the call
   from `sub_A8A1F0` to `sub_A8A400`. Current hooks in
   `app/fang/fang.c:1213-1252` observe construction/entry but not successful
   return.
8. **Client authority watchdog:** record RakPeer queue/statistics plus action
   active/response/queued state when successful sends continue without a
   receive. This is diagnostic first; it should not mutate state until the
   server recovery contract is implemented.

A loopback packet capture can independently demonstrate kernel arrival, but
it is lower value than pre-processing ingress plus lock/goroutine phase
markers. The latter will identify both the first trapped datagram and the
blocking server owner in one reproduction.

## Acceptance criteria for the correction

- Reproduce the same campaign 1-1 movement/enemy activation boundary under
  repeated Lightning Rogue Basic input without a UDP read stall.
- Force a scheduled producer to block in a test and prove that socket ingress
  and another peer continue.
- Cancel a schedule while its worker is waiting for outbound encoding and
  prove cancellation returns without a lock cycle.
- Verify ordered/reliable counters remain gap-free when producers complete
  outside `outboundMu` and a peer generation changes before encoding.
- Verify stale producer/handler results are discarded after peer recovery.
- Verify all ACK ranges, including `0x2fc..0x2fe`, `0x2ff`, `0x300..0x302`,
  and an actual `0xffffff -> 0x000000` wrap fixture.
- Trigger the watchdog and confirm the trace identifies ingress, lock owner,
  schedule group/producer, and goroutine stacks.
- Verify controlled recovery either restores authoritative play or performs a
  clean campaign rejoin; it must never leave the client accepting local input
  indefinitely without an authority-state transition.
