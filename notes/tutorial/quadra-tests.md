# Quadra movement-handler verification plan

## Scope and test level

Add a focused in-package test file, `server/gameplay_quadra_test.go`, which calls
the `raknet.Handler` returned by `newGameplayHandler` directly. Do not start a
UDP listener, a RakNet peer, timers, or goroutines. This is the narrowest level
that still exercises the live path in `server/gameplay_udp.go`: gameplay join,
session creation, dungeon setup, Sage-boundary movement detection, Quadra run
creation, immediate packet assembly, schedule-group registration, and the
generation check inside each delayed producer.

Keep the existing `server/quadra_sim_test.go` tests as adapter-level coverage.
The new tests complement them; they must not call `newQuadraPresentationRun`
instead of the gameplay handler for the main success and rollback cases.

The proposed tests require no proprietary content database. Use a literal
semantic program matching the already bytecode-checked oracle. The optional
`DARKSPIN_TEST_CONTENT_DB` evidence test in
`server/sim/lua_content_test.go` remains responsible for proving that retail
chunk 502 compiles to this oracle.

## Harness construction

### Gameplay binding and session

Create one `sporenet.User` and a small `ActiveUserFinder` fake in package
`server`:

```go
user := sporenet.NewUser("Quadra Test", "quadra-test", "")
user.Account.ID = 42
user.Account.Level = 2
user.Account.XP = 167

gameManager := game.NewManager()
gameManager.SetHostNetwork(game.NetworkPair{
	External: game.NetworkEndpoint{IP: 0x7f000001, Port: 42127},
})
instance := gameManager.Create()
instance.Info.Level = game.TutorialLevel
instance.Info.Mode = game.ModeTutorial
if !instance.AddPlayer(user) {
	t.Fatal("AddPlayer() = false")
}
gameplayJoin, err := game.NewGameplayJoin(testActiveUserFinder{user: user}, gameManager)
```

Use `log.New(io.Discard, "", 0)` and a nil or no-op
`tutorialProgression`; the Quadra trigger does not call progression. Construct
the handler with:

```go
handler := newGameplayHandler(
	gameplayJoin,
	progression,
	simulationPrograms{quadraFirstAggro: verifiedQuadraProgram(t)},
	logger,
)
```

Use one stable address for every packet in a test, for example
`&net.UDPAddr{IP: net.IPv4(127, 0, 0, 1), Port: 42000}`. The address string is
the private session key.

Drive the real session state in this order:

1. `HelloPlayerRequest`: use
   `raknet.HelloPlayerRequestMessage{UserID: 42}.EncodePayload()` as the packet
   payload. Assert the two normal responses so a failed join cannot make later
   assertions misleading.
2. `PlayerStatusUpdate`: provide an eight-byte payload containing status `8`
   followed by `float32(0)`. This sets `peerSession.isDungeon` and returns the
   player update, game-start, and ping packets.
3. `DebugPing`: provide any eight-byte payload. This creates
   `openingEncounter`, initializes the deployed player as object `1`, and fixes
   `playerPosition` at the selected checkpoint.
4. `ActionCommandMsgs`: send the movement described below with the fake
   schedule group attached.

Select the existing focused checkpoint by saving
`tutorialPlayerSpawnPosition`, assigning `tutorialSageUnlockTestPosition`, and
restoring it with `t.Cleanup`. Do not call `t.Parallel` in these tests because
the checkpoint is a package global. This is existing replay configuration, not
a new production seam. At `DebugPing` it creates the intended stage-3 empty
encounter and sets `isLootObeliskUsed`, satisfying the real Sage prerequisite.

### Movement payload

Add a test helper that returns the 64-byte packed movement command accepted by
`raknet.DecodeActionCommand`. Populate only the fields relevant to this path:

- byte `0`: `byte(raknet.ActionMovement)`;
- bytes `8:12`: little-endian object ID `tutorialPlayerObjectID` (`1`);
- bytes `44:56`: little-endian float bits for the X/Y/Z components of
  `tutorialSageUnlockPosition`;
- bytes `56:60`: goal flags, preferably `1`;
- every other byte: zero.

The offsets are `40` bytes of `ActionCommon`, then the movement payload's
four-byte unknown field, 12-byte goal, flags, and final unknown field. Moving
from `tutorialSageUnlockTestPosition` to `tutorialSageUnlockPosition` intersects
the authored Sage sphere without intersecting the ability, second-ability, orb,
teleporter, or opening-enemy triggers.

Set the accepted movement packet's `SourceTime` to `7000`. Immediate application
packets are the handler return value; no network capture is necessary.

### Verified chunk-502 program

`verifiedQuadraProgram(t)` should construct the exact nine-step oracle and fail
the test immediately if its provenance drifts from:

```text
LuaChunkID:     502
BytecodeSHA256: da18208fc597c67ce3251de30d12ad694ae4d644af0e1cb79b5383cc12ec6d8d
FunctionName:   FirstAggro_SpecialOne
Confidence:     sim.ConfidenceBytecode
```

The steps are, in order:

1. cinematic on `specialOne`, duration `4.291667s`, radius `100`;
2. visibility false for `specialOne`;
3. wait `2s`;
4. visibility true;
5. visibility true;
6. animation `character_teleport_in`;
7. wait `1.291667s`;
8. wait `1s`;
9. sequence complete.

Use the production constants `quadraRole`, `quadraFirstAggroChunkID`, and
`quadraFirstAggroSHA256` in the fixture. Keep the semantic list visibly aligned
with `TestRetailFirstAggroBytecodeMatchesGoOracle` and
`quadraOracleProgram`. The live unit test injects this already-verified
program through the existing `simulationPrograms` argument; it must not open
`content.db`.

### Fake ScheduleGroup

Use a recording fake with mutable `err`, `groups`, `isCancelled`, and
`cancelCount` fields. Its function must copy the producer slice before storing
it, return `err` without a cancel function when configured to fail, and on
success return an idempotent cancel function that records cancellation.

The fake starts no goroutines. Tests invoke each stored `Produce` function
directly. Before invoking anything, assert one group containing exactly these
ordered delays:

```text
2s
3.291667s
4.291667s
```

Manual invocation deliberately decouples callback wall time from simulator
time. Calling the `2s` producer immediately still must stamp its animation at
`7000 + 2000`, demonstrating that `packet.SourceTime` and the semantic deadline,
not `time.Now`, are authoritative.

## Exact proposed tests

### `TestGameplayQuadraMovementPublishesImmediateAndScheduledTimeline`

Build the complete session, capture a successful schedule group, and invoke the
Sage-triggering movement once.

Assert the immediate handler result has exactly 12 packets in this order:

| Index | Packet ID | Essential assertion |
| ---: | --- | --- |
| 0 | `ObjectPlayerMove` | accepted player object `1` and the authored goal |
| 1 | `LabsPlayerUpdate` | Sage reveal snapshot |
| 2 | `ObjectCreate` | Sage object `17` |
| 3 | `CombatantDataUpdate` | object `17` |
| 4 | `AttributeDataUpdate` | object `17` |
| 5 | `PlayerCharacterDeploy` | slot `0`, creature `0`, active object `1` |
| 6 | `ObjectCreate` | Quadra object `42` |
| 7 | `CombatantDataUpdate` | object `42` |
| 8 | `AttributeDataUpdate` | object `42` |
| 9 | `ObjectPlayerMove` | object `42`, flags `0x20`, authored Quadra position |
| 10 | `CinematicMsgs` | rounded duration `4292ms`, authored focus, radius `100` |
| 11 | `ObjectUpdate` | object `42`, authored position, visibility false |

Use packet IDs plus the relevant little-endian fields, not whole-packet golden
byte slices. Explicitly assert that indices `0:10` contain no
`SetAnimationState`, and that the only immediate Quadra `ObjectUpdate` is hidden.

Invoke the recorded producers manually:

- `2s` returns exactly `ObjectUpdate`, `ObjectUpdate`, `SetAnimationState`.
  Both updates target `42`, preserve the authored position, and set visibility
  true. The animation targets `42`, hashes `character_teleport_in`, has timestamp
  `9000`, overlay false, and scale `1`.
- `3.291667s` returns a non-nil or nil empty batch with no error.
- `4.291667s` returns an empty batch with no error.

Finally send the same boundary movement again. It must return only the normal
player movement packet and must not register a second schedule group. This is
the public proof that `isSageEnemySpawned` prevents a duplicate run and a
duplicate encounter enemy.

### `TestGameplayQuadraScheduleFailureRollsBackTransition`

Configure the fake `ScheduleGroup` to return a sentinel error. Invoke the
boundary movement and assert:

- the error matches the sentinel and contains `moveSageSchedule`;
- the returned immediate batch is nil/empty;
- registration was attempted once with all three correct producers;
- no cancel function was returned or called;
- none of the captured producers is invoked by the test.

Then clear the fake error and send the same movement again to the same handler
and address. The retry must return the complete 12-packet immediate batch and
register a new successful group. Invoke its three producers and assert the
normal timeline. A successful retry proves that the failed attempt did not
commit `isSageUnlocked`, `isSageEnemySpawned`, `quadraPresentation`, the
encounter enemy, or a consumed simulator cursor. If any of those gates had
leaked, the retry would return only movement or would suppress the producer
timeline.

### `TestGameplayQuadraAuthorityMarshalFailureRollsBackTransition`

Use the single dependency seam below with a test closure that reads a mutable
`marshalErr`: while non-nil it returns that sentinel; after the test clears it,
the same closure delegates to `marshalTutorialSageEncounterAuthority`. Invoke
the boundary movement and assert:

- the error matches the sentinel and contains `moveSageAuthority`;
- no immediate packets escape;
- `ScheduleGroup` is never called;
- its cancel function is never called.

Clear `marshalErr` and retry the same movement through the same handler. Require
the complete 12-packet batch and all three producers. This
behavioral retry assertion covers the same session rollback fields as the
schedule-failure test and additionally proves that the failed temporary Quadra
run did not advance a cursor visible to the retry.

The concrete messages passed to `raknet.MarshalApplication` on this path cannot
naturally fail: their `EncodePayload` methods do not return errors and
`MarshalApplication` fails only for a nil interface. A dependency seam is
therefore necessary to exercise the live handler's authority-marshal rollback
branch rather than merely testing an unrelated malformed simulation program.

### `TestGameplayQuadraSessionReplacementCancelsCapturedProducers`

Start a successful Quadra transition but do not invoke its producers. Send a
second valid `HelloPlayerRequest` from the same UDP address. Assert the fake
group's cancel function is called exactly once. Manually call every producer
captured from the old group; each must return an empty batch without error
because `quadraProducer` no longer finds the expected run in the replacement
session. This directly covers the session-key/expected-run guard and makes the
fake cancellation observable without relying on a real timer.

### `TestGameplayQuadraInitialEncodeFailureDoesNotCommit`

This is a small negative companion that needs no seam. Construct a nonempty
test program whose offset-zero batch contains an unsupported intent after one
valid intent. The handler must return an error containing
`moveSageSimulation`, `immediateEncode`, and `batchEncode`, publish no prefix of
the immediate simulation batch, and never call `ScheduleGroup`. Send the same
movement again and require the same error, proving the Sage/Quadra committed
gate was not set by the first attempt. This verifies same-deadline encoder
atomicity; it does not replace the authority-marshal seam test above.

## Required small test seam

Only authority marshalling needs a new seam. Scheduling, future producer
capture, program injection, session replacement, and immediate packet capture
are already injectable through existing arguments and `raknet.Packet` fields.

Add an unexported dependency form without changing the existing constructor's
callers or production behavior:

```go
type gameplayHandlerDependencies struct {
	marshalSageEncounterAuthority func(game.GameplayBinding, raknet.Vector3) ([][]byte, error)
}

func defaultGameplayHandlerDependencies() gameplayHandlerDependencies {
	return gameplayHandlerDependencies{
		marshalSageEncounterAuthority: marshalTutorialSageEncounterAuthority,
	}
}
```

`newGameplayHandler` should delegate to an unexported
`newGameplayHandlerWithDependencies` using the defaults. The live Sage branch
calls the dependency field at the existing authority-marshal line. Validate a
nil dependency at construction or replace it with the default; do not use a
mutable package-level function variable, because that would make tests
order-dependent and race-prone.

Do not expose the handler's session map, simulator, or cursor for assertions.
Retry behavior, duplicate-trigger behavior, cancellation, and captured producer
output provide stronger black-box evidence and avoid a broad session-inspection
API. Do not add clock or scheduler interfaces: producer timing is already fully
controlled by the fake `ScheduleGroup`.

## Assertion helpers

Keep helpers test-only and decode just the stable wire fields needed by these
tests:

- `assertPacketIDs` for ordered packet IDs;
- `assertObjectID` for payload bytes `1:5` on object-keyed application packets;
- `assertObjectUpdate` for object ID, position, and trailing visibility field;
- `assertAnimation` for object ID, state hash, timestamp, overlay, and scale;
- `assertCinematic` for signed duration, focus vector, and radius;
- `movementActionPayload` for the packed client command;
- `startQuadraGameplaySession` for Hello/status-8/ping setup;
- `recordingScheduleGroup` for registration, cancellation, and manual production.

Each helper should call `t.Helper()`. Copy producer slices and returned packet
bytes in the fake where ownership could otherwise be ambiguous. Follow the
repository's separate error-assignment style and do not run checkpoint-mutating
tests in parallel.

## Verification command and completion criteria

Run the focused tests first:

```text
go test ./server -run 'TestGameplayQuadra' -count=1
```

Then run the affected packages:

```text
go test ./server ./server/raknet ./server/sim/... -count=1
```

The harness is complete when the live handler success path, all three manually
invoked deadlines, duplicate suppression, scheduling rollback, authority
marshal rollback, initial batch atomicity, and session-replacement cancellation
all pass without sockets, sleeps, production database access, or exported test
APIs.
