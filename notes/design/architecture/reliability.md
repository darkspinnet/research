# Gameplay reliability and soft-lock recovery

## Executive finding

The main reliability problem is not one malformed gameplay packet. The server
currently permits a transport acknowledgement, gameplay mutation, semantic
response, delayed publication, and projection dequeue to succeed or fail at
different boundaries. A failure between those boundaries can leave the client
waiting for an action result while the server believes the action was handled.
That is the recurring soft-lock shape.

The highest-priority correction is to make every admitted client action and
every recovery command reach one explicit terminal outcome. Transport work
must remain responsive even when gameplay work stalls, and reconnect must
replace a transport binding without destroying the retained zone member.

## Implementation status

The first reliability batch is implemented:

- polled commands receive the same timing and scheduling capabilities as
  ordinary inbound application packets;
- `/reset` preserves and publishes its player recovery when NPC restart fails
  and republishes the controlled hero baseline;
- projection events are encoded, transport-preflighted, and retained for
  reliable delivery before their cursor advances, and projection faults no
  longer swallow direct gameplay responses; each subscriber is bounded to
  4,096 pending events and overflow requests a current baseline instead of
  dropping an arbitrary transition prefix;
- delayed producer guards bind to the gameplay generation captured when they
  are scheduled;
- valid action failures and missing terminal responses are converted to safe
  client rejection, with authoritative correction for failed movement;
- accepted and pursuing actions are tracked by transport-generation leases and
  receive a safe rejection plus an authoritative movement, resource, and hero
  control baseline if their promised release never arrives; delayed terminal
  responses close those leases only after successful RakNet publication, not
  when a scheduled producer merely creates packet bytes;
- RakNet datagrams are dispatched through bounded per-peer executors rather
  than running gameplay on the shared UDP reader;
- whole-peer snapshot rollback is removed; and
- a peer-specific UDP write failure no longer closes the shared socket for
  every connected player;
- ordinary replies, scheduled publications, and retransmissions are serialized
  by a peer-local outbound owner instead of one socket-wide writer lock; and
- transport loss now marks a zone member disconnected, excludes that member's
  actors from NPC targeting, retains the semantic membership for ten minutes,
  and permits the next authenticated gameplay connection to publish a current
  hero, NPC, objective, and security baseline into the same zone; connected
  co-op peers also reacquire and restart NPC actions after another member
  disconnects. The departing member's live motion is stopped and synchronized,
  and canceled connection-local action gates, status effects, and presentation
  identities are cleared before retaining the session.
- result votes expose their resolved decision without destructively
  acknowledging it before Continue or Cash Out succeeds. The pending poller can
  replay next-level setup, Cash Out routing, or an already committed reward
  receipt from authoritative result state, and Continue preserves the active
  transport generation while allocating a fresh zone generation.

The recovery baseline now reconstructs live pickups, companions, consumed
interactables, teleporter progression, active horde barriers, and a pending
dead-hero selection instead of replaying the defeated hero as alive. Durable
checkpoints, launcher Continue/Abandon UX, immutable aggregate snapshots, and
the removal of packet-coupled feature scheduling remain active work.

## Reviewed high-risk findings

### P0: one gameplay handler can stop the shared UDP reader

Previously, `server/udp/server.go` read one datagram and synchronously called
`processPacket`, which invoked gameplay on the same call path. RakNet traffic
now enters bounded per-peer executors, preserving order within one peer while
keeping the shared reader and other peers responsive.

Each peer dispatch now warns after 250 milliseconds and has a five-second hard
boundary. At that boundary its request context is canceled, the exact monotonic
RakNet generation is retired, queued datagrams for that worker are abandoned
for client retransmission, all goroutine stacks are captured, and a fresh
worker may accept reconnect traffic. Lifecycle notification is asynchronous so
a member lock held by the stalled handler cannot block worker detachment.

The warning and timeout records include the exact generation's connection
state, reliable-window depth and oldest age, future and missing receive counts,
ordered backlog, split-assembly count and bytes, plus ingress queue depth. This
keeps diagnostics bounded while distinguishing a feature stall from receive
ordering, fragmentation, or outbound-window pressure.

Packet-carried delayed schedules deliberately use a connection-owned context
without the request's cancellation. Timing out one ingress handler therefore
cancels that handler but does not erase delayed releases that were already
accepted; explicit schedule cancellation, peer replacement, and connection
shutdown still terminate that work.

Gameplay dispatch checks request cancellation before entering feature logic,
after feature dispatch, after NPC recovery, and after waiting for the member
replacement gate. A retired handler that eventually unblocks therefore stops
before it can drain projections or continue reconnect replacement work. It
also reapplies exact-generation lifecycle cleanup after returning. If copied
session code wrote stale state back after the watchdog's first callback, that
second cleanup removes the resurrected endpoint and retains membership again;
a replacement generation remains untouched. Go still cannot terminate the
arbitrary blocked goroutine before it returns.
Duplicate cleanup is idempotent at the zone boundary: if membership is already
disconnected or belongs to a newer semantic generation, only stale
connection-local runs and presentation allocations are quiesced. It does not
call `Zone.Leave` and cannot delete the retained membership needed by rejoin.
The registry also tracks one authoritative RakNet transport generation per
game member independently of endpoint session copies. A late old endpoint is
discarded before lifecycle teardown when that owner has advanced, preventing
it from disconnecting a successfully rejoined member whose semantic zone
generation intentionally remained stable.
Tutorial restart now carries that active transport generation into its new
semantic gameplay epoch instead of resetting it to zero, so producer guards
and disconnect matching remain effective after a retry.
The same authority record includes the current endpoint. If late work writes an
old transport copy onto a different endpoint, that record is discarded. If it
overwrites a newly negotiated transport at the same endpoint, the adapter
retains the current transport generation, disconnects only zone targetability,
and marks the session for a fresh authoritative baseline on the next control
poll instead of leaving an unbound connected client.

Authenticated endpoint replacement acquires its member gate with the request
context rather than an unbounded mutex wait. Rejoin commit callbacks never wait
on that gate after baseline packets have been written; a busy gate leaves the
session pending so the next client poll can replay and commit the authoritative
baseline.

Moving application work outside `outboundMu` removed the previously proven
deadlock cycle. Gameplay dispatch now completes before the peer's outbound
critical section begins; only response preflight, RakNet index allocation,
retransmission retention, socket publication, and response commits remain
serialized. Scheduled work from an earlier action can therefore publish while
a feature handler is still working. Work registered by the current handler is
held behind a response-local barrier until its immediate accepted response has
been written and committed, preventing a zero-delay release from overtaking
admission. Per-peer dispatch also isolates the socket reader. A context passed
to a handler is still not a timeout unless all called operations observe it; a
Go goroutine cannot otherwise be killed.

Explicit RakNet disconnect packets now invoke the same gameplay lifecycle as
idle expiry, write failure, and watchdog retirement. Removing transport no
longer leaves an address-owned ghost session active until a later login happens
to supersede it.

Retransmission expiry now waits for the exact peer's outbound publication lock
and revalidates idle time or exhausted attempts before removing transport. A
concurrent valid response can no longer write and commit after the sweeper has
retired its peer, and a retransmission collected just before an ACK is skipped
once its pending sequence disappears.

The global retransmission sweep never waits for a busy peer publication lock.
It skips that peer until the next sweep, then revalidates that the datagram is
still pending and due before incrementing its attempt count. Lock contention or
a concurrent ACK therefore cannot consume retry attempts for bytes that were
never sent, and one wedged peer cannot pause retransmissions for all others.

Socket lifetime uses a connection-wide read/write gate distinct from peer
publication ownership. Immediate responses, scheduled batches, and
retransmissions hold a shared read gate only around final connection validation
and writes, so peers remain concurrent; attach and close take the exclusive
gate and cannot replace or close a socket beneath an in-flight publication.

Replacing or closing that socket atomically retires every peer generation and
its reliable window from the old connection. Generation-fenced gameplay
notifications run after connection locks are released, preventing stale peers
from being treated as connected on a new socket without making attach or close
wait for zone cleanup.

Transport lifecycle events carry the retired RakNet generation through the
runtime boundary. Gameplay disconnect ignores a notification unless both its
endpoint and transport generation still match, so asynchronous watchdog,
explicit-disconnect, write-failure, and retransmission cleanup cannot remove a
replacement session that reused the endpoint. Retransmission cleanup is then
safe to run outside the global sweep, preventing one slow member cleanup from
pausing reliable delivery for every peer.

RakNet offline-open replacement also publishes its handshake without waiting
for the old generation's gameplay cleanup. The exact-generation callback runs
asynchronously, so a slow retained-zone transition cannot prevent the new
transport from completing negotiation and the callback cannot remove it later.
Managed publication deliberately transfers ownership from the pre-request peer
to the generation created by offline open before validating and writing the
reply; this is the sole request that is expected to replace its own peer.
Replacement first acquires the previous generation's publication lock without
holding the registry mutex. An old scheduled packet that already entered its
final write therefore finishes before the generation swap, while every later
old-generation publisher fails exact-peer validation and cannot interleave
gameplay bytes with the new handshake.
The request's address, header, protocol, magic, and minimum reply MTU are all
validated before that lock and generation swap. A malformed or undersized
renegotiation cannot retire an otherwise healthy peer before discovering that
no valid reply can be emitted.

The shared UDP multiplexer treats the legacy `0x09` open request as an
unambiguous route transition even when that endpoint was previously cached as
QoS. Reusing a client source port can therefore begin RakNet recovery
immediately rather than remaining pinned to the QoS decoder until its
five-minute route expiry.

Process shutdown first closes network admission, then stops the shared
scheduler, then clears gameplay sessions before flushing users or closing
user/content storage. Pending zone and retained-session callbacks therefore
cannot begin after gameplay teardown or race handles that have already been
closed. Network and gameplay teardown have independent process-owned execution
guards, so concurrent service failure and explicit close paths cannot invoke
either lifecycle twice.
Scheduler shutdown still discards pending work immediately, but now tracks
callbacks that were already running or detached from its serial worker. It
waits up to five seconds for those callbacks before gameplay and persistence
teardown, retaining the non-hanging guarantee for a genuinely stuck callback.
The shared UDP listener owns a cancellable serve context used by every
endpoint worker. `Close` cancels those workers as it detaches the RakNet socket,
and runtime teardown closes listeners before emptying gameplay registries, so
an in-flight handler observes cancellation instead of rebuilding state after
cleanup.
Gameplay registry cleanup also closes request admission and waits up to five
seconds for already-admitted handlers to return. The wait is bounded so a
pathological handler cannot prevent process exit; a handler returning after the
bound still runs exact-generation cancellation cleanup at its outer boundary.
Blaze owns the same kind of serve-generation cancellation. Closing the service
cancels active RPC contexts as well as their sockets, and an accept completing
across that boundary is closed before it can enter the live session directory.
Listener generations also fence deferred close calls, so an older serve loop
cannot close a replacement listener installed on the same server instance.
After cancellation and socket close, Blaze waits up to five seconds for its
active session handlers to leave the live directory. Runtime persistence
therefore normally cannot close beneath an RPC that is still unwinding, while
a pathological handler still cannot prevent shutdown forever.
HTTP applies the same boundary around its router: shutdown closes admission,
closes active connections to cancel request contexts, and waits up to five
seconds for admitted handlers before persistence teardown continues.
The shared TCP protocol multiplexer checks its closed signal while holding the
same lock that admits accepted sockets to the pending router set. A connection
accepted across shutdown is closed immediately instead of escaping ownership
before either the Blaze or HTTP server receives it.

ACK and NACK mutation now shares peer publication ownership with ordinary
responses and retransmission. A NACK cannot increment or select a retained
datagram concurrently with the periodic retransmitter, and an ACK cannot race
a resend after removing its sequence. Connected handshake replies are queued
as application payloads and encoded only in the final publication section, so
a datagram that batches control and gameplay never executes the later gameplay
handler while holding the peer's outbound lock.
Decoded acknowledgement ranges are capped at the same 4,096-datagram reliable
window the peer can own. A compact malformed range can no longer expand into
an unbounded sequence slice or leave a watchdog-detached goroutine consuming
CPU after transport retirement. Connected datagrams use the same cap for
encapsulated packet records, preventing a maximum-size UDP datagram made of
empty packet headers from amplifying dispatch work without bound.

Explicit disconnect is terminal for the remaining encapsulated payloads in its
datagram, but peer retirement is a response commit. The server writes the
datagram ACK first and only then removes the exact generation and invokes the
retained-zone lifecycle; a replacement that wins before commit is preserved.

Gameplay Hello treats a request as a retransmission only when game, user, and
RakNet transport generation all match. If a new generation reaches Hello before
the asynchronous old-generation callback, admission synchronously retains that
exact endpoint under its original member gate, then consumes the retained
membership for the new transport. Endpoint reuse can no longer masquerade as a
duplicate Hello or leak the replaced zone membership.
Admission repeats the duplicate test after acquiring that member gate. Two
concurrent retransmissions that both observed an empty endpoint can no longer
let the second overwrite and stop the session installed by the first.
The serialized check also orders transport generations. An older Hello that
finishes authentication late is rejected, while a newer generation retains an
older same-endpoint session before consuming it for baseline rejoin.

Gameplay disconnect also never waits on a busy member gate. It schedules a
short generation-fenced retry and returns; the retry becomes a no-op if another
path already retained or replaced that transport. A stuck member operation can
therefore no longer accumulate blocked lifecycle goroutines or hold up shared
transport cleanup.

Action recovery leases use a server-owned action generation as well as the
transport generation and client sync stamp. The client stamp is only eight
bits and can be reused during a long session; a delayed terminal response now
completes only the exact action that scheduled it and cannot delete a newer
lease with the same stamp.
Each lease retains its own cancellation handle. A committed release or reject
removes the lease and cancels its recovery timer immediately, so completed
combat does not accumulate dormant watchdog goroutines until their deadlines.

Required direction:

- copy and route datagrams immediately from the socket reader;
- serialize RakNet receive processing in one bounded executor per peer;
- keep peers independent so one full or wedged executor cannot stop another;
- record ingress before dispatch and handler completion after dispatch;
- detach a timed-out peer generation and offer rejoin instead of blocking the
  shared reader indefinitely.

Queue overflow must be an explicit peer failure with diagnostics. Silently
dropping a reliable ordered application datagram would create another lock.

Queue overflow now cancels and removes that endpoint's worker, discards its
remaining queue, retires only the exact RakNet generation observed at ingress,
and invokes the ordinary retained-zone disconnect path. A replacement
generation is compare-and-delete protected and remains untouched.

### P0: `/reset` can partially commit and then lose its response

The current poll path creates `raknet.Packet{Address: address}` without any
scheduler functions. A queued `/reset` can mutate the peer session, cancel
actions, stop movement, reset cooldowns, synchronize the zone hero, and restart
NPC targeting before `scheduleFirstActions` discovers that scheduling is
unavailable. The runtime then returns `eventResetEnemySchedule`, and the reset
packets are not published.

This exact path occurred in the latest runtime log:

```text
eventResetEnemySchedule: enemyPursuitFallback[3]:
enemyZelemSchedule: scheduler unavailable
```

Scheduling is a server capability and must not be borrowed from the inbound
RakNet packet. Zone/gameplay operations should receive a feature-owned timer or
scheduler and produce semantic events. The RakNet adapter should only encode
and publish those events.

Recovery commands must follow plan/commit semantics:

1. validate that every required recovery operation is available;
2. build the recovery plan without changing live state;
3. commit cancellations and authoritative state changes;
4. enqueue a complete recovery baseline;
5. return a success/failure result that cannot be mistaken for an unhandled
   command.

### P0: projection drain is lossy and can discard an action response

`gameplayProjectionRuntime.drain` removes every queued zone event before the
RakNet adapter marshals it. If one event cannot be encoded, the queue has
already been cleared. The outer gameplay handler also returns the projection
error in place of the otherwise valid direct action response. This can both
lose world events and strand the client's current action token.

Replace destructive `Drain` with `Peek` plus `Commit(sequence)` or an
equivalent cursor. Encode first and advance the cursor only after the batch is
accepted by the peer outbound queue. A malformed semantic event should be
quarantined with its sequence and context; it must not erase later events or
discard an unrelated action rejection/release.

Projection queues are currently unbounded per subscriber. Add a bounded limit,
queue age, and a baseline-resync fallback. Do not recover overflow by dropping
an arbitrary prefix because many packets describe state transitions rather
than replaceable snapshots.

### P0: action errors have no enforced terminal semantic response

RakNet ACK confirms receipt of a transport datagram; it does not release the
client's action state. The gameplay handler result is only `packets, error`, so
it cannot distinguish these cases:

- intentionally ignored control traffic;
- an action rejected with a terminal response;
- an accepted action awaiting a registered delayed release;
- an action handler that accidentally returned no response;
- a handler that mutated state and then failed.

Introduce a typed application outcome. For an action command it should include
the sync stamp, actor, disposition (`completed`, `deferred`, `rejected`, or
`fatal`), emitted semantic events, and any registered lease. Middleware at the
ActionCommand boundary must guarantee either a terminal response now or a
tracked deferred lease with a deadline. Handler errors should normally map to
a safe rejection; transport ACK alone is never sufficient.

Unknown non-action packets may still be observed and ignored. Terminal zone
state should explicitly reject or release an action rather than return no
packets merely because the zone is terminal.

### P0: whole-peer rollback races concurrent transport work

`HandleAndWriteDatagrams` clones the entire mutable RakNet peer and restores it
when handling or connection validation fails. Scheduled publication and
retransmission can mutate that peer concurrently. Restoring the snapshot can
rewind message/datagram indices, receive ordering state, last-receive time, or
the pending retransmission window, including work owned by another operation.

Do not transactionally roll back a shared transport object by assignment. A
per-peer outbound executor should be the sole owner of index allocation,
encoding, enqueue, write, and retransmission retention. Allocate ordered and
reliable indices only when a validated payload is committed to that queue.
Indices remain monotonic; failures transition the peer rather than rewinding
history.

### P1: delayed producer identity

The generic `gameplayProducerGuard` now captures game ID, user ID, semantic
zone generation, and transport generation. A callback from an older tutorial
restart, campaign epoch, member, or socket becomes a no-op even when the same
network endpoint remains in use. Ability-owned operation generations still
protect replacement runs within one otherwise-current zone session.
The ActionCommand scheduling adapter enforces that identity for every producer
registered through an admitted command, including legacy abilities that do not
explicitly decorate their producer list. A stale producer emits no packets and
its feature commit callback is not invoked. Action lease creation and expiry use
the same identity, so an old timeout cannot reject or reset a replacement zone
that happens to retain the same socket.

Action admission takes the member lifecycle gate and revalidates the member,
zone generation, and transport generation before dispatch. Reconnect and
endpoint replacement therefore cannot swap the session between command decode
and business mutation.

The shared producer decorator guards its commit callback as well as packet
creation. A producer suppressed because its member, zone, transport, or
terminal state is stale cannot execute feature mutation merely because an
empty scheduled publication completed successfully.

Every delayed operation must capture `(game ID, user ID, zone generation,
peer generation, operation generation)`. The shared scheduler validates that
identity before invoking it. Feature code should not need to reproduce this
check in every ability.

### P1: transport address currently owns gameplay lifetime

Gameplay sessions are indexed by UDP address. RakNet peer replacement or
expiry calls gameplay cancellation, which removes the member from the zone and
stops connection-local work. A reconnect on a new port cannot replace the peer
while preserving the zone state.

Use `(game ID, user ID)` as stable membership identity. Treat address and
RakNet peer generation as replaceable bindings. Transport loss should mark the
member disconnected for a configurable grace period, cancel only
connection-owned action leases/presentations, and retain zone-owned hero,
companions, NPCs, loot, objectives, and results. Explicit abort/leave remains a
separate operation.

The detailed reserve/project/commit contract is in `notes/zone/rejoin.md`.

### P1: scheduled work is coupled to RakNet packet closures

Scheduling helpers occur across many gameplay files and are carried on
`raknet.Packet`. This makes a server-owned action depend on which transport
entry point happened to trigger it; the reset poll failure is the direct
example. It also encourages callbacks that retain copied peer-session state.

Move scheduling to zone/action operations. Scheduled steps should return
transport-neutral semantic events into an outbox. A peer publisher wakes when
the outbox changes and maps events to build-103 packets. Inbound client traffic
must not be required to flush server-owned state.

### P2: observability cannot identify the first stalled operation

Current shared-UDP logging occurs after synchronous processing. The existing
`produced no response` message is also noisy for client ACK datagrams, where no
response is normal. This obscures the useful boundary.

Add structured records for:

- socket ingress time, endpoint, peer generation, datagram sequence, packet
  IDs, action sync stamp, and trace ID before dispatch;
- receive decision (`accepted`, `duplicate`, `ordered-wait`, or `rejected`);
- handler start/end, duration, outcome, mutation revision, and packet/event
  counts;
- peer inbound/outbound queue depth and oldest age;
- pending reliable datagrams and oldest retransmission;
- active action leases and oldest deadline;
- projection cursor, queue depth, and oldest event;
- recovery/rejoin phase and baseline revision.

Warn on handler latency before failure (for example at 250 ms and 1 s). On a
hard threshold, capture goroutine stacks and state ownership. A watchdog is
diagnostic until the underlying operations honor cancellation or the peer
executor can be detached safely.

Socket ingress now assigns a monotonic trace ID and carries it through RakNet
application packets, slow-action records, action lease expiry, queue pressure,
and peer watchdog diagnostics. Managed RakNet publication no longer emits the
misleading `produced no response` record merely because its bytes were written
inside the transport owner; standalone ACK/NACK silence is likewise omitted.

## Recovery model

Recovery should be a ladder, with mission abort last:

1. **Automatic action expiry**: every deferred action lease sends a safe
   release/rejection and clears held/pursuit state by its deadline.
2. **Peer resnapshot**: cancel connection-owned leases and publish a revisioned
   baseline for the current hero, squad/death state, resources, cooldowns,
   movement, NPCs/targets, companions, pickups, interactables, objectives, and
   teleporter/security state. Preserve shared zone progress.
3. **Transport replacement**: detach the unhealthy peer executor, create a new
   peer generation, and project the same baseline after reconnect. Do not call
   `Zone.Leave` merely because transport was lost.
4. **Zone rejoin**: launcher/client Continue attaches to retained membership by
   `(game ID, user ID)` within the grace period.
5. **Abort**: use only when the zone aggregate itself cannot produce a valid
   snapshot or the player explicitly leaves.

The existing Fang `/reset` hook may clear proven client-local descriptors and
held input, but it cannot repair a blocked server reader or reconstruct missing
authoritative packets. It should accompany, not replace, the server resnapshot
transaction.

## Recommended implementation order

1. Fix reset to use a server-owned scheduler and make its state transition
   atomic; always return a visible recovery result.
2. Make projection delivery non-destructive until encode/enqueue succeeds, and
   prevent projection failures from swallowing direct action responses.
3. Add generation to the shared delayed-producer guard and cancel all leases
   by generation on switch, death, reset, disconnect, and terminal transition.
4. Introduce typed ActionCommand outcomes and enforce terminal response or a
   deadline-bound deferred lease.
5. Replace whole-peer rollback and mixed writers with one outbound owner per
   peer.
6. Put RakNet work behind bounded per-peer executors so the shared UDP reader
   never runs gameplay synchronously.
7. Separate transport presence from zone membership and implement the retained
   rejoin transaction and full baseline projection.
8. Consolidate the duplicated packet-carried scheduling and local generation
   checks into zone scheduling and action-lease services.

The first three steps are narrow enough to reduce current soft locks before
the larger transport and rejoin changes land. Steps four through seven provide
the durable guarantee that a single failed action or connection no longer
destroys a run.

## Implemented recovery boundary

The production path now has bounded per-peer ingress, peer-local outbound
serialization, generation-scoped delayed work, terminal action admission with
expiry, reliable-commit projection delivery, bounded projection queues, and a
ten-minute retained-membership reconnect. Reconnect and projection-overflow
baselines reconstruct current heroes, NPCs, companions, pickups, objectives,
used interactables, teleporter progression, and active horde barriers. A
pending death selection republishes the defeated hero hidden and at zero health
without assigning control or beaming it back in. A captured projection revision
fences each baseline so events published during encoding remain queued rather
than being erased by a late cursor reset.

Rejoin activation is committed only after the complete baseline enters the
peer's reliable transport window. The retained member remains disconnected and
excluded from NPC targeting until that commit; NPC target acquisition and
action restart then use the ordinary recovery poll. Until then, ordinary
projections remain paused and premature action commands receive a safe rejection. RakNet
preflights every immediate or scheduled multi-packet application response
against MTU fragmentation, sequence reuse, and retransmission capacity so a
large baseline or delayed terminal sequence cannot be partially retained before
a predictable capacity failure.

Managed immediate responses now defer projection and rejoin commits until all
returned UDP datagrams have been written successfully. Scheduled response
callbacks likewise run only after their same-deadline packet batch is encoded,
retained, and written; registering a schedule no longer falsely completes an
action before its terminal packet exists. A failed write therefore leaves the
semantic transition recoverable instead of acknowledging state the client did
not receive.

Reconnect and projection-overflow baselines use an eight-attempt optimistic
revision fence. If client-visible zone state advances while the adapter reads
the component snapshots, it discards that mixed baseline and rebuilds against
the new revision. The projection queue still preserves events that arrive
after the accepted revision.

If all eight reconnect attempts race active zone mutations, baseline assembly
now remains pending instead of failing the gameplay handler and retiring the
new transport. The next ordinary connected-control poll retries from a fresh
revision; deterministic encoding and state errors remain terminal transport
faults rather than being hidden as churn.

Reconnect publication is no longer dependent on the client's `DebugPing`.
Any valid connected control traffic can drive the pending baseline, and zone
membership becomes connected atomically with advancing its projection cursor
through the delivered revision. A missing incremental projection cursor is
recreated in baseline-required state rather than silently losing future world
events.

Panics from scheduled producers, response commits, publication observers,
peer lifecycle callbacks, shared UDP dispatch, and process scheduler tasks are
contained at their asynchronous boundary and logged with stacks. One malformed
callback can fail its own operation, but can no longer terminate the process or
silently kill the only delayed-task worker for every zone.

The process scheduler preserves ordinary serial execution, warns when a task
runs for 250 milliseconds, and detaches it after one second so subsequent zone
timelines and reconnect expiries continue. Detachment does not make an
arbitrary stuck callback cancellable; feature callbacks must still use
generation guards and bounded I/O, but shutdown and unrelated zones no longer
wait behind it indefinitely.

An immediate response failure after transport acknowledgement, or any delayed
producer, preflight, encoding, write, or commit failure, removes only the exact
RakNet peer generation that owned the work. The gameplay lifecycle then retains
its semantic zone membership for baseline rejoin. Continuing on an
acknowledged-but-incomplete stream is forbidden because the client cannot
retransmit that consumed application command safely.

Delayed failure cleanup runs after releasing the peer publication lock. A
post-write commit callback failure forces the same clean reconnect but does not
invoke the feature's publication-failure rollback, because its packet is
already visible to the client. Address reuse is compare-and-delete guarded in
immediate, scheduled, and retransmission paths so stale failures cannot remove
a newer peer at the same endpoint.

Authenticated gameplay endpoints are now single-owner by `(game ID, user ID)`.
Hello transitions are serialized per member, supersede every older endpoint
through the retained-zone path, and discard stale-generation disconnects
instead of letting them replace valid retained state. Registry shutdown closes
admission before draining active and retained sessions, preventing an in-flight
Hello or disconnect from recreating ghost membership after cleanup.

The next boundary is an immutable zone snapshot acquired under one semantic
revision rather than the current bounded optimistic retry. Launcher Continue
and Abandon discovery and two-client fault-injection verification also remain.

## Evidence boundaries

This review is based on the current server implementation, the prior confirmed
soft-lock and rejoin audits, and the latest runtime log. It does not claim that
every observed gameplay freeze has the same cause: client-local action gates,
semantic server/client divergence, and a blocked server transport require
different recovery stages. The proposed trace IDs, terminal outcomes, leases,
and baseline revisions are intended to distinguish those cases on the first
failure rather than infer them after mission abort.
