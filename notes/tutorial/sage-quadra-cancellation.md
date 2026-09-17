# Sage/Quadra composite-run cancellation audit

This audit covers the current worktree's composite `sageEncounterRun`, the
gameplay-session owner around it, and the RakNet scheduled-publication group.
It is limited to cancellation and lifecycle behavior; it does not propose an
implementation.

## Result

The composite run fixes two old ownership problems: Sage unlock and Quadra now
share one schedule/cancel handle, and a successful final deadline clears the
session's run pointer. `HelloPlayerRequest` replacement, accepted object `42`
defeat, and accepted Beam Out also detach the pointer before calling `stop()`,
so a producer which has not begun observes an identity mismatch and returns no
packets.

The run is not lifecycle-complete, however. The normal cancellation paths can
deadlock against a deadline which has fired but is waiting for RakNet's outbound
lock. Transport failure after a producer succeeds is not reported back to the
gameplay owner, leaving a live but unscheduled run. A later producer failure
clears the run but leaves committed Sage/Quadra state with no presentation
recovery. Peer teardown and explicit tutorial restart still have no gameplay
owner hook. Same-deadline preparation is all-or-none through encoding, but UDP
writes are a sequential prefix and gameplay state commits before publication.

| Boundary | Current behavior | Audit result |
| --- | --- | --- |
| Hello replacement | Replaces the session, then stops the old run | Correct stale-producer identity guard; cancellation can deadlock at the outbound-lock boundary |
| Object `42` defeat | Clears the pointer under `sessionMutex`, then stops the run | Correct owner ordering; same outbound-lock deadlock remains |
| Accepted Beam Out | Persists completion, clears pending run pointers, then stops them | Correct only after persistence succeeds; same outbound-lock deadlock remains |
| Peer teardown/replacement | Old RakNet peer becomes stale; scheduled producer is suppressed | Gameplay session/run is neither stopped nor removed |
| Tutorial restart | `ReloadLevel` is ignored; repeated status `8` does not reset the session | Missing lifecycle boundary |
| Producer failure | Producer aborts and clears the run | First-deadline failure can retry; later failure strands committed encounter state |
| Same-deadline publication | Produce all, encode all, then write each datagram | No producer/encode prefix, but state is already committed and a write can publish a prefix |

## Findings

### F1 — Lifecycle cancellation can deadlock with a fired deadline

Severity: high.

`raknet.Server.HandleAndWriteDatagrams` holds `outboundMu` while it calls the
gameplay handler (`server/raknet/server.go:305-339`). A scheduled group calls
`produceAndSendScheduledDeadline`, which also takes `outboundMu` before checking
cancellation (`server/raknet/server.go:566-590`). The returned cancel function
cancels the group and synchronously waits for its goroutine's `done` channel
(`server/raknet/server.go:540-543`).

That creates this wait cycle when a timer has fired but the scheduled goroutine
has not yet acquired `outboundMu`:

1. An inbound Hello replacement, object `42` defeat, or Beam Out owns
   `outboundMu` and enters the gameplay handler.
2. The scheduled goroutine is blocked acquiring `outboundMu`.
3. The handler detaches the run and calls `sageEncounterRun.stop()`
   (`server/sage_encounter_sim.go:115-124`).
4. `stop()` calls the schedule cancel handle, which waits for the scheduled
   goroutine to exit.
5. The scheduled goroutine cannot observe cancellation or exit until the
   handler releases `outboundMu`; the handler cannot return until cancel exits.

The gameplay-only Hello test uses `recordingQuadraSchedule`, whose cancel
function merely flips fields. It cannot expose this transport lock cycle.
`TestScheduleConnectedPacketGroupCancelIsPublicationBarrier` proves that cancel
waits for an active producer, but calls cancel without holding `outboundMu`, so
it also misses the cycle.

Concrete tests:

- `TestGameplaySageEncounterHelloCancellationDoesNotDeadlockOutboundDeadline`:
  install a real scheduled group, arrange for its deadline goroutine to be
  queued on `outboundMu`, invoke a Hello replacement through
  `HandleAndWriteDatagrams`, and require both the handler and group to finish
  within a bounded timeout with no old-run payload.
- Repeat the same harness as
  `TestGameplaySageEncounterDefeatCancellationDoesNotDeadlockOutboundDeadline`
  and `TestGameplaySageEncounterBeamOutCancellationDoesNotDeadlockOutboundDeadline`.
  The defeat fixture must first publish the `500ms` deadline so object `42`
  exists; the Beam Out fixture must mark the horde complete and use a successful
  progression stub.

The test needs a deterministic latch immediately before the scheduled goroutine
takes `outboundMu`; relying on a sleep would make this race test flaky.

### F2 — Transport publication failure orphans the gameplay run

Severity: high.

`sageEncounterProducer` advances the simulation and commits gameplay state
under `sessionMutex` before returning packets (`server/gameplay_udp.go:153-180`).
At the `500ms` deadline it commits `isSageUnlocked`,
`isSageEnemySpawned`, and object `42` in `openingEncounter`. Only after the
producer returns does RakNet encode and write the batch
(`server/raknet/server.go:592-623`).

If `scheduledEncode` or `scheduledWrite` fails, the schedule goroutine reports
the error and exits (`server/raknet/server.go:527-535`), but it does not notify
the gameplay owner. Therefore `peerSession.sageEncounter` still points to a run
whose schedule has terminated. No later deadline will reveal/animate/complete
Quadra, and movement cannot create a replacement because the pointer and
committed spawn gate remain set. A write failure additionally detaches the
connection, but still does not retire the gameplay session.

Concrete tests:

- `TestGameplaySageEncounterFirstDeadlineEncodeFailureRetiresRun`: inject a
  transport encoder failure after the `500ms` producer succeeds. Assert that no
  datagram is published, the peer counters roll back, and the gameplay owner is
  explicitly retired or marked failed rather than retaining an unscheduled
  run. The test should also assert the chosen compensation policy for the
  already-mutated Sage/object-42 state.
- `TestGameplaySageEncounterDeadlineWriteFailureRetiresRun`: fail the first
  write and assert transport detachment plus the same owner cleanup.
- `TestGameplaySageEncounterTransportFailureAllowsDefinedRecovery`: reconnect
  and exercise the selected policy—fresh tutorial epoch, retry of the composite
  run, or explicit terminal failure. The current code provides none of those
  paths.

### F3 — A later producer failure strands a committed, possibly hidden Quadra

Severity: high for recoverability, medium for resource ownership.

When `expected.advance` fails, `sageEncounterProducer` aborts both child runs,
clears `peerSession.sageEncounter`, and returns the error
(`server/gameplay_udp.go:161-168`). This is correct cleanup before the first
deadline: the Sage/Quadra gates have not committed, so another crossing can
schedule a fresh run.

After the `500ms` producer succeeds, the same failure policy is incomplete.
The committed `isSageUnlocked` and `isSageEnemySpawned` flags and object `42`
remain. A failure encoding the `2.5s` visibility/animation deadline aborts and
clears the only run, while Quadra can remain hidden and the movement gate
prevents rescheduling. The error is only asynchronously logged by RakNet.

Existing tests cover construction-time immediate encoding failure and schedule
registration failure, not an asynchronous composite producer failure.

Concrete tests:

- `TestGameplaySageEncounterFirstProducerFailureRollsBackAndRetries`: use a
  fail-once creature-unlock/dispatch seam, invoke producer `0`, require no Sage
  flags, no object `42`, no retained run, then cross again and require a new
  schedule.
- `TestGameplaySageEncounterLaterProducerFailureHasDefinedCompensation`: let
  producer `0` commit, make producer `1` fail during Quadra batch encoding, and
  assert an explicit policy. A safe assertion must cover visibility/combat
  state as well as pointer cleanup; merely checking that the producer returned
  an error is insufficient.
- `TestGameplaySageEncounterFailureCancelsRemainingDeadlines`: with a real
  scheduler, fail producer `1` and prove producers `2` and `3` never run and no
  later packets are published.

### F4 — Peer teardown suppresses transport work but does not cancel gameplay ownership

Severity: medium.

An offline open from the same address replaces `s.peer[address]` with a new
pointer (`server/raknet/server.go:234-244`). Before running a scheduled producer,
RakNet compares the captured peer pointer with the current peer and rejects a
stale peer (`server/raknet/server.go:573-587`). This correctly prevents old
payloads from reaching the reused address.

It does not call into the gameplay handler, stop `sageEncounter`, or delete the
address-keyed gameplay session. Disconnect payloads below
`HelloPlayerRequest` are ignored by `handleConnectedPayload`, and connection
detach/close cancels transport contexts without clearing gameplay ownership.
If the replacement connection later sends Hello, the Hello path eventually
cleans up. Teardown without a new Hello leaves the run/session resident; a
reconnect path which sends gameplay traffic before Hello can also encounter the
old session state.

Concrete tests:

- `TestGameplaySageEncounterPeerReplacementCancelsOwner`: start the composite
  run, replace the RakNet peer at the same UDP address without sending Hello,
  and require both transport suppression and owner cancellation/removal.
- `TestGameplaySageEncounterDisconnectCancelsOwner`: send the supported RakNet
  disconnect notification and require the peer to become disconnected and the
  gameplay run to stop. This test first requires a real disconnect lifecycle
  surface; none exists now.
- `TestGameplaySageEncounterServerConnectionDetachCancelsAllOwners`: attach
  `nil`/close the listener while deadlines are pending and require every run to
  be stopped, not merely every timer to be canceled.

### F5 — Tutorial restart has no composite-run epoch boundary

Severity: medium.

`raknet.ReloadLevel` has no gameplay-handler case, so it is ignored. Repeating
`PlayerStatusUpdate` status `8` only marks the existing session as a dungeon;
the following setup remains gated by `isDungeonSetupSent`. Neither path stops
the composite run, resets encounter maps/flags, or permits a fresh Sage marker.
Hello replacement happens to create a fresh session, but it is not an explicit
restart contract and cannot cover a restart which retains the gameplay peer.

Concrete tests:

- `TestGameplayTutorialRestartCancelsSageEncounter`: start a run, issue the
  accepted restart command, and require cancellation, removal of stale
  producers, reset of Sage/Quadra/encounter flags, and a fresh dungeon setup.
- `TestGameplayTutorialRestartStartsFreshSageEncounterEpoch`: after restart,
  cross the marker again and require exactly one new schedule whose old
  producers all return empty and whose source timestamps come from the new
  movement packet.
- `TestGameplayRepeatedDungeonStatusIsNotRestart`: preserve a regression test
  making the current distinction explicit if repeated status `8` is not chosen
  as the restart command.

### F6 — Same-deadline publication is only atomic before UDP writes

Severity: medium.

At `500ms`, one composite producer returns the complete ordered batch: Sage
unlock packets, Quadra authority, cinematic, and hidden presentation. At
`2.5s`, one producer returns both visibility updates and the teleport animation.
RakNet gathers every producer for a deadline, gathers all payloads, and encodes
all payloads before beginning writes (`server/raknet/server.go:523-623`). Thus a
producer or encoding failure publishes none of that deadline's packets. The
existing generic RakNet tests verify same-deadline producer failure and order.

Two boundaries remain non-atomic:

- Gameplay state is committed inside the producer before transport encoding or
  publication; F2 describes the resulting state/wire split.
- Encoded datagrams are written sequentially. If write `N` fails, writes
  `0..N-1` may already be visible. UDP cannot provide transactional multi-write
  publication, and the current write-failure test closes the socket before the
  first write, so it does not exercise a partial prefix.

Concrete tests:

- `TestGameplaySageEncounterHalfSecondBatchOrder`: run the real composite
  producer and assert the exact twelve-packet order already used by the
  gameplay fixture, plus one shared deadline and no immediate Sage/Quadra
  authority before `500ms`.
- `TestScheduleConnectedPacketGroupEncodeFailurePublishesNoDeadlinePrefix`:
  fail encoding after at least one payload was successfully encoded, require no
  socket writes, and require restoration of all peer counters.
- `TestScheduleConnectedPacketGroupNthWriteFailureReportsPublishedPrefix`:
  use an injectable writer that succeeds for `N-1` datagrams then fails. Assert
  the exact unavoidable prefix, connection detachment, committed counter
  policy, and gameplay-owner failure notification. This prevents tests from
  overstating all-or-none wire publication.

## Coverage disposition

Keep the current tests for schedule registration rollback, construction-time
authority/immediate-encode failure, stale producer identity after Hello, exact
deadline ordering, generic cancel draining, and same-deadline producer
rejection. They are useful but do not cover the owner/transport integration
gaps above.

The minimum blocking regression set is:

1. one deterministic outbound-lock cancellation test covering the F1 wait
   cycle;
2. object `42` defeat and Beam Out owner-cancellation tests using the composite
   run rather than a standalone Quadra run;
3. one post-`500ms` producer failure test;
4. one transport encode/write failure test which observes gameplay ownership;
5. peer teardown and explicit restart tests; and
6. one same-deadline test which distinguishes preparation atomicity from a
   possible UDP write prefix.

