# Quadra cancellation ownership audit

This note records the current worktree as of 2026-07-18. It covers the
per-gameplay-session Quadra presentation run only; it does not propose changes
to Quadra combat authority or to object `42`'s encounter lifetime.

## Result

Object `42` death, `HelloPlayerRequest` replacement, and accepted Beam Out each
cancel a live run as normal lifecycle operations. Construction/producer error
paths also stop a run, but they are failure cleanup rather than lifecycle
ownership. Peer teardown, tutorial restart, natural presentation/encounter
phase exit, Blaze game/player removal, and explicit server close have no
gameplay-session cancellation hook. Cancellation of the server `Run` context
stops pending scheduler goroutines, but it does not stop or remove their
`quadraPresentationRun` objects and is not a substitute for gameplay ownership.

| Required boundary | Current entry path(s) | Current result |
| --- | --- | --- |
| Object `42` death/deletion | `server.newGameplayHandler`, `ActionCommandMsgs`; `tutorialEncounter.applyDamage` | **Implemented, with an in-flight-send hazard.** Accepted defeat stops and clears the run under `sessionMutex` before the session commit and delete response. |
| `HelloPlayer` replacement | `server.newGameplayHandler`, `HelloPlayerRequest` | **Implemented, with an in-flight-send hazard.** Calls `stop()` under `sessionMutex`, then replaces the entire session value. |
| Disconnect/peer teardown | `raknet.Server.handleConnectedPayload`; `raknet.Server.legacyOpenReply`; `sharedUDPServer.endpointProtocol` | **Missing.** Disconnect payloads are ignored, peers are never explicitly torn down, peer replacement only makes old scheduled sends stale, and route expiry only removes routing metadata. |
| Tutorial restart | Blaze `resetDedicatedServerHandler`; RakNet `ReloadLevel`; repeated tutorial start/status traffic | **Missing.** No restart path reaches the gameplay session map. `ReloadLevel` is only a packet ID and is ignored by the handler. |
| Phase exit | Quadra `SequenceCompleteIntent`; teleporter/horde transition; dungeon/beam-out status | **Partial.** Accepted Beam Out stops and clears the run. Natural timeline completion and the concrete teleporter/horde transition do not. |
| Game/player removal | Blaze `removePlayerHandler`, `destroyGameHandler`, `advanceGameStateHandler`; `game.Instance.RemovePlayer`; `game.Manager.Remove` | **Missing.** Blaze/game locks and mutations are independent of the UDP gameplay closure. |
| Server shutdown | `server.Server.Run`, `server.Server.Close`, `server.Server.closeNetworkServices`, `sharedUDPServer.Close`, `raknet.Server.Close` | **Partial transport cancellation only.** A canceled `Run` context cancels scheduler groups; direct close only changes/closes the socket. Neither path stops runs or clears gameplay sessions. |

## Current owner and cancellation primitive

`server/gameplay_udp.go:23-52` defines `gameplayPeerSession`; its
`quadraPresentation *quadraPresentationRun` field is the sole live ownership
reference. `newGameplayHandler` (`server/gameplay_udp.go:60`) closes over:

- `session map[string]gameplayPeerSession`, keyed by `UDPAddr.String()`; and
- `sessionMutex sync.RWMutex`, which serializes the session map, the run's
  simulator/cursor, and the mutable tutorial encounter.

The returned type is only `raknet.Handler`, a function. It exposes no
`Close`, cancel-by-address, cancel-by-user, cancel-by-game, or cancel-all
operation. Consequently `raknet.Server`, `sharedUDPServer`, `game.Manager`, the
Blaze handlers, and `server.Server.Close` cannot reach the actual owner.

`quadraPresentationRun` is defined in `server/quadra_sim.go:23-29`. It owns the
raw `*sim.Simulator`, the `specialOne` `sim.CancelScope`, the RakNet encoder,
the event cursor, and a `raknet.CancelSchedule`. Its `stop` method
(`server/quadra_sim.go:106-117`) is nil-safe and idempotent enough for repeated
calls: it invokes and clears the schedule cancel handle, invalidates the
`specialOne` role, and stops the simulator. `stop` does **not** clear the
enclosing `gameplayPeerSession.quadraPresentation`; the session owner must do
that or replace/delete the session while holding `sessionMutex`.

The simulator itself is not synchronized. `sim.Simulator.InvalidateRole` and
`Stop` are in `server/sim/simulator.go:121-126` and `246-251`; advancement is
in `AdvanceTo` at `206-227`. Current run advancement is correctly performed
under `sessionMutex` by `quadraProducer` (`server/gameplay_udp.go:128-146`). Any
new normal cancellation path must use that same mutex before calling `stop`,
clearing the pointer, or deleting/replacing the session.

The transport group is registered by
`raknet.Server.scheduleConnectedPacketGroup`
(`server/raknet/server.go:344-413`). It captures the current `*peer` and socket,
derives a group context from the handler context, and returns its context
cancel function. `sendScheduledPayloads` (`server/raknet/server.go:416-444`)
then takes `scheduledSendMu`, checks peer identity and the current socket under
`raknet.Server.mu`, encodes, and writes the batch.

## Object `42` death and deletion: implemented, publication not drained

There is one authoritative current damage/death path. In the
`ActionCommandMsgs` ability branch of `newGameplayHandler`
(`server/gameplay_udp.go:778-1152`):

1. The handler takes `sessionMutex` at line 789 and resolves the session and
   target from `openingEncounter` at lines 790-803.
2. Ride or basic damage calls `openingEncounter.applyDamage` at lines 834 or
   852. `tutorialEncounter.applyDamage`
   (`server/tutorial_encounter.go:486-533`) deletes the enemy map entry at line
   499. For object `42`, it returns at lines 509-510 without spawning another
   encounter stage.
3. Still under `sessionMutex`, the handler can set
   `isTeleporterUnlocked = true`, then copies the mutated session back into the
   session map. Lines 864-868 detect defeated object `42`, call `stop`, and set
   `peerSession.quadraPresentation = nil`; line 869 commits that value.
4. The handler releases `sessionMutex` at line 871, can perform tutorial XP
   persistence at lines 873-890, and only later builds the death response.
5. On defeat, lines 1046-1066 build `ObjectDeleteMessage`. Object `42` is not
   an `isOpeningTutorialEnemy`, so its delete is appended immediately to the
   response at line 1065; it does not use the 1.5-second opening-enemy
   death schedule.

This implements the required owner ordering. Clearing the pointer under the
lock is essential because a producer that has started and is blocked on
`sessionMutex` then fails the pointer-identity check at
`server/gameplay_udp.go:132` and returns no packets.

Current ordering hazards:

- Cancellation and pointer clearing under `sessionMutex` cannot retract a
  batch whose producer already returned. There is a gap between
  `quadraProducer` releasing `sessionMutex` at line 144 and
  `sendScheduledPayloads` acquiring `scheduledSendMu` at line 417. Such a batch
  can be written after the object `42` delete response. Publication therefore
  needs a final session/run-generation validity check or an equivalent
  cancel-and-drain barrier, not only producer-side validation.

## `HelloPlayerRequest` replacement: implemented, publication not drained

The `HelloPlayerRequest` case in `newGameplayHandler`
(`server/gameplay_udp.go:150-162`) first completes
`helloPlayerResponses`/`GameplayJoin.Execute`. On success it takes
`sessionMutex`, stops the previous run if present, replaces the complete map
value with `gameplayPeerSession{binding: binding}`, and unlocks.

This correctly serializes `stop` against simulator advancement and removes the
old pointer by replacing the session. A producer blocked on `sessionMutex`
sees the new session value and fails its expected-pointer check.

Two ordering limits remain:

- The old run remains live while `helloPlayerResponses` executes. This is
  desirable on join failure (the valid old session is retained), but a
  deadline can publish before the successful replacement reaches the lock.
- As with object death, a producer can return its application batch, release
  `sessionMutex`, and then race with replacement. Because the underlying
  RakNet `*peer` did not change, the stale-peer check accepts that batch and
  the cancel handle does not drain it. The old batch can therefore be sent
  after the new Hello response.

## Disconnect and peer teardown: missing

There is no implemented RakNet disconnect-to-gameplay path:

- `raknet.Server.handleConnectedPayload` (`server/raknet/server.go:263-321`)
  handles transport payload `0x04`, transport payload `0x11`, and application
  IDs at or above `HelloPlayerRequest`. A RakNet disconnection notification
  below `0x7f` falls through the `payload[0] < HelloPlayerRequest` guard at
  lines 290-292 and is ignored. No code marks `peer.isConnected = false` or
  deletes `s.peer[address]`.
- A new offline open at the same address executes
  `raknet.Server.legacyOpenReply` (`server/raknet/server.go:169-198`) and
  overwrites `s.peer[address]` under `raknet.Server.mu` at lines 181-186. This
  makes an old scheduled send fail the pointer-identity check in
  `sendScheduledPayloads`, but the producer runs first, may advance the old
  Quadra simulator, and leaves the gameplay session/run installed.
- `sharedUDPServer.endpointProtocol` (`server/udp_mux.go:141-160`) removes an
  idle route under `sharedUDPServer.mu` at lines 147-155. It does not remove the
  RakNet peer or gameplay session. Route expiry is lazy: it occurs only when a
  later packet from that address calls `endpointProtocol`.
- Blaze TCP `Session.serve` removes only the Blaze session from the Blaze
  server indexes (`server/blaze/server.go:213-219`, `464-475`). It is not a
  RakNet peer teardown and has no gameplay-session callback.

A real peer teardown must first detach/mark stale the transport peer so no new
application work is accepted, then cancel/delete the matching gameplay session
under `sessionMutex`. It must not call into gameplay while holding
`raknet.Server.mu` or `sharedUDPServer.mu`; those locks are also used on send
and packet routing paths. A callback invoked after releasing the transport lock
avoids creating a `raknet.Server.mu -> sessionMutex` order opposite to packet
handling/scheduling.

## Tutorial restart: missing

There are three restart-shaped surfaces, none of which resets the gameplay
owner:

- Blaze GameManager command `0x19` is
  `resetDedicatedServerHandler`
  (`server/blaze/game_manager_component.go:69-110`). It creates and configures
  a new `game.Instance`, then attempts to add the user. It has no reference to
  the UDP address or gameplay handler. Its game mutations use `game.Manager.mu`
  in `Manager.Create` (`server/game/manager.go:212-228`) and `Instance.mu` in
  `AddPlayer` (`server/game/manager.go:86-107`), not `sessionMutex`.
- `raknet.ReloadLevel` exists as packet ID `0xbf`
  (`server/raknet/types.go:73`), but `newGameplayHandler` has no case for it; it
  reaches the default ignored-packet branch at
  `server/gameplay_udp.go:1233-1235`.
- `PlayerStatusUpdate` status `8` sets only `peerSession.isDungeon = true`
  under `sessionMutex` (`server/gameplay_udp.go:1217-1220`). The subsequent
  DebugPing initialization is gated by `!isDungeonSetupSent`
  (`server/gameplay_udp.go:168-219`), so repeating this traffic does not reset
  encounter fields or the Quadra run.

A restart must be a session replacement/reset operation: stop and clear the
old run under `sessionMutex` before resetting encounter flags/maps or accepting
a new Sage trigger. Reusing `sim.Simulator.Reset` alone would be unsafe because
`quadraPresentationRun` also owns the encoder cursor and scheduler group; the
whole run must be retired and a fresh run created for the new tutorial epoch.

## Phase exit: partial

The Quadra adapter is constructed with a raw `sim.New(quadraPhase)` in
`newQuadraPresentationRun` (`server/quadra_sim.go:49-65`). It is not attached to
the gameplay `sim.Director`, and the gameplay session has no phase field. The
generic `sim.Director.Transition` invalidates old simulator phase scopes
(`server/sim/director.go:190-215`), but no live Quadra path calls it.

Current phase-exit-shaped paths are:

- The final `4.291667s` producer advances through
  `SequenceCompleteIntent`. On success `quadraProducer` simply unlocks and
  returns (`server/gameplay_udp.go:136-145`). It neither calls `stop` nor clears
  `peerSession.quadraPresentation`, so the completed run remains installed
  indefinitely.
- Movement through the boss teleporter sets `isHordeArenaEntered`, creates a
  new horde, resets the health-obelisk map, and changes player position under
  `sessionMutex` (`server/gameplay_udp.go:469-474`). This is the concrete
  gameplay transition out of the Sage/Quadra encounter, but it does not retire
  the run.
- `PlayerStatusUpdate` Beam Out (`status == 0x20`) first completes persistence,
  then takes `sessionMutex`, re-resolves the current session, stops and clears
  its run, and commits the session before marshalling the completion snapshot
  (`server/gameplay_udp.go:1182-1204`). This path is implemented. If tutorial
  completion persistence fails, it returns before cancellation; no phase exit
  has been accepted in that case.

Natural sequence completion ends presentation only; it must not remove object
`42` from `openingEncounter` or cancel its combat authority. It should retire
presentation ownership (stop/clear the run) after the final deadline has been
successfully committed. An early logical phase exit must do the same before
committing the new phase's session mutations. Beam Out has the same
post-producer publication window described for death and Hello replacement.

## Game and player removal: missing

The current Blaze/game lifecycle has several removal paths:

- `removePlayerHandler` (`server/blaze/game_manager_component.go:173-208`)
  calls `Instance.RemovePlayer` after optional tutorial completion persistence.
- `destroyGameHandler` (`server/blaze/game_manager_component.go:113-130`)
  snapshots players, calls `gameManager.Remove`, then queues notifications.
- `advanceGameStateHandler` (`server/blaze/game_manager_component.go:240-258`)
  can assign `StatePostGame` or `StateDestructing`, but writes `instance.Info.State`
  directly without `Instance.mu` and has no gameplay hook.
- `game.Manager.Remove` (`server/game/manager.go:253-263`) deletes the instance
  under `Manager.mu`, releases that lock, then removes each player.
  `Instance.RemovePlayer` (`server/game/manager.go:148-157`) deletes player,
  slot, and tutorial-completion entries under `Instance.mu`, unlocks, then
  releases the user's game claim under the user's own lock.

None of these paths can identify or cancel the gameplay closure's session.
`GameplayBinding` (`server/game/gameplay_session.go:26-36`) contains `UserID`
but not `GameID`; the gameplay map is keyed only by UDP address and has no
secondary user/game index. After game removal, the gameplay session retains its
copied binding and tutorial state even though the authoritative game membership
has been released.

Cancellation should be requested after releasing `Manager.mu`/`Instance.mu`,
and before removal is reported complete to clients. Destroying a game requires
cancelling every session indexed to that game; removing one player requires
cancelling only that user's session. That needs stable game identity in the
binding/session plus an owner API or lifecycle callback. Calling a gameplay
callback while holding game locks would introduce avoidable cross-subsystem
lock ordering and notification latency.

## Server shutdown: partial transport behavior, missing owner cleanup

`server.Server.Run` creates `runCtx` at `server/server.go:531-532` and passes it
through the UDP listener into every packet scheduler group. Parent cancellation
therefore closes each group's derived context at
`server/raknet/server.go:377-391`; producers that have not begun return without
running. A service error also calls `cancel` before closing network services at
`server/server.go:559-564`.

The owner is still not cleaned up:

- `server.Server.Close` (`server/server.go:575-610`) is guarded by
  `closeOnce` and calls `closeNetworkServices`, storage flush/close, scheduler
  shutdown, and Lua/trace close. It cannot access the gameplay session closure.
- `server.Server.closeNetworkServices` (`server/server.go:700-710`) closes the
  shared UDP server but does not cancel `runCtx` itself.
- `sharedUDPServer.Close` (`server/udp_mux.go:188-202`) clears its route map
  under `sharedUDPServer.mu`, calls `raknet.AttachConnection(nil)`, and closes
  the socket. It neither clears RakNet peers nor gameplay sessions.
- `raknet.Server.Close` (`server/raknet/server.go:135-147`) only reads the
  socket under `raknet.Server.mu` and closes it.

Thus shutdown reached by canceling `Run` suppresses not-yet-started transport
producers, but run pointers/simulators remain live in memory until the handler
becomes unreachable. A direct `Server.Close` while `Run`'s context is still
live does not even cancel the group timers: a timer can wake, run
`quadraProducer`, mutate the simulator, and only then fail the changed-socket
check in `sendScheduledPayloads`.

Shutdown ownership should cancel all gameplay sessions under `sessionMutex`
before or as part of network shutdown, without holding RakNet socket or game
manager/instance locks during per-run work. The run context
remains a useful second transport signal, but the gameplay owner must still
stop each simulator and clear/delete every session entry.

## Implemented cleanup and test coverage

For completeness, lifecycle and failure cleanup currently exists in these
places:

- Authority marshal or group-registration failure calls `quadraRun.stop`
  before unlocking and before the run/session mutations are committed
  (`server/gameplay_udp.go:441-460`).
- Object `42` defeat calls `stop` and clears the session pointer under
  `sessionMutex` (`server/gameplay_udp.go:864-869`). Accepted Beam Out does the
  same after persistence (`server/gameplay_udp.go:1191-1198`).
- A delayed producer error calls `expected.stop`, clears
  `peerSession.quadraPresentation`, writes the session back, and unlocks
  (`server/gameplay_udp.go:136-142`).
- `TestQuadraPresentationRunStopCancelsFutureDeadline`
  (`server/quadra_sim_test.go:60-69`) proves that advancing a stopped simulator
  fails. RakNet scheduler tests prove cancellation suppresses a producer that
  has not started and that peer replacement suppresses the final send, but no
  test currently exercises death, Beam Out, Hello replacement, or any of the
  missing lifecycle boundaries through `newGameplayHandler`.

The local gameplay handler owns death, Beam Out, Hello replacement, and
producer-failure cleanup. The key missing cross-layer boundary is an
address/user/game-aware gameplay-session owner that can be invoked by restart,
natural/encounter phase, peer, game, and server lifecycle events, with a
publication validity check that closes the post-producer send window.
