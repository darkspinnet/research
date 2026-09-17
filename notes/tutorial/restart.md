# Tutorial game over and restart protocol

## Result

Build 103 does **not** implement the tutorial death HUD as a
request/response restart protocol. The apparent restart surface splits into
two different paths:

1. `HUD_Death.swf` exposes one native callback,
   `MaxisGameOver.OnExitClicked`. The callback plays the HUD outro and changes
   the client state to `GameSpaceship` (`2`). It does not construct or send a
   gameplay message.
2. `ReloadLevel` (`0xBF`) is a server-to-client, zero-payload command. It
   reloads the current level in place and leaves the connected RakNet peer
   intact. There is no client-side `ReloadLevel` request builder and no
   acknowledgement payload.

Consequently there is no exact "restart-button request/response packet pair"
to implement for the tutorial. Treating `0xBF` as a client request, or treating
Blaze GameManager reset-dedicated-server command `0x19` as the death-button
request, would combine unrelated mechanisms.

## Exact gameplay messages

All application bytes below begin at the Game GMS boundary, after RakNet
reassembly and before the one-byte application ID is stripped by the client
dispatcher.

| Direction | Bytes | Layout | Native effect |
| --- | --- | --- | --- |
| S2C | `AD 01` | `ChainGameMsgs (0xAD)` + subtype `1` | In the active gameplay state, selects pending client state `13` (`GameOver`). This is the two-byte game-over transition message. |
| S2C | `AE 00` or `AE 01` | `ChainGameOverMsgs (0xAE)` + one-byte subtype | Once `cGameOverState` is active, its handler consumes subtypes `0` and `1` but performs no mutation. These are not client acknowledgements and carry no trailing fields. |
| S2C | `AF 00` | `QuickGameMsgs (0xAF)` + subtype `0` | In the active gameplay state, selects pending state `2` (`Spaceship`). It is a server-authored return transition, not a response emitted by the native death-HUD button. |
| S2C | `BF` | `ReloadLevel (0xBF)` and no payload | Calls the in-place level teardown/reload path, reacquires the current level tuple, installs the rebuilt level, and clears the old level state. |

There is no C2S application-level game-over acknowledgement. Receipt is
acknowledged only by RakNet when the enclosing datagram uses a reliable mode.
That transport ACK identifies the datagram sequence, not `0xAD`, and has no
game-over-specific payload.

The repository's `GameOverMessage` encodes the active gameplay state's
game-over selector as logical message `45`, wire `0xAD`, subtype `1`:
`sub_44DC60 -> sub_44B0A0` records pending state `13`. Logical message `46`,
wire `0xAE`, is the handler installed by `cGameOverState` itself and merely
consumes its one-byte subtype. This distinction matters if these messages are
implemented later.

### Payload widths

The layouts are fixed:

```text
ChainGame game-over
  uint8 application_id = 0xAD
  uint8 subtype        = 0x01

ChainGameOver while game-over
  uint8 application_id = 0xAE
  uint8 subtype        = 0x00 or 0x01

QuickGame return
  uint8 application_id = 0xAF
  uint8 subtype        = 0x00

ReloadLevel
  uint8 application_id = 0xBF
  // no payload
```

No player ID, game ID, object ID, timestamp, state scalar, or reflection
terminator follows these bytes.

## Ordering

The client-side order supported by the executable is:

```text
connected gameplay / tutorial world
  S2C AD 01
  client enters cGameOverState (13)
  client opens HUD_Death.swf
  player clicks its return button
  local MaxisGameOver.OnExitClicked callback plays "outro"
  client transitions locally to GameSpaceship (2)
```

An authoritative server may instead send `AF 00` to move an active gameplay
state to Spaceship. That is a separate server-driven transition. It is not
causally paired by the executable with the HUD callback.

The only in-place reload order proven by the client is:

```text
connected gameplay / current level
  S2C BF
  client tears down and rebuilds the level in place
  subsequent S2C GameState/world/object initialization
```

`0xBF` itself contains no epoch, level identifier, or correlation ID. The
server must therefore serialize it before the replacement level's GameState
and replication traffic. No application-level acknowledgement follows it.

RakNet transport ACKs remain ordinary ACK records for the enclosing reliable
datagrams. Their exact bytes necessarily contain the live datagram sequence
number and cannot be recovered from an application message definition.

## UDP peer and session lifetime

For `BF`, the UDP peer survives. The `ReloadLevel` dispatcher
(`sub_53ADC0`, logical message `64`) calls `sub_537F60`; that handler performs
level cleanup and reconstruction but does not call RakPeer disconnect,
transport connect, `HelloPlayerRequest`, or any gameplay handshake routine.
The reload therefore occurs inside the existing connected RakNet peer and its
reliability session.

The HUD return-button path is not an in-place restart. It leaves gameplay for
Spaceship. A later tutorial launch is a new game-entry lifecycle, so server
gameplay state and object-scoped jobs must not be carried through it as though
`BF` had been received. The retained material does not contain the live
disconnect/reconnect datagrams needed to state the precise transport teardown
sequence for that return path.

Darkspin implements that local-return boundary when the same connected peer
later submits a new dungeon-entry player status after its squad terminal latch.
The server increments the gameplay epoch and preserves only the resolved player
binding. It stops the old session's schedules, then lets the ordinary
GameStart/DebugPing path reconstruct squad HP, world objects, encounters,
cooldowns, pickups, effects, and route facts. It does not infer a `BF` request
or send `AF 00`; those remain optional server-owned policies with no recovered
client request trigger.

Returning to Spaceship does not skip onboarding or unlock normal squad
management. Death does not deliver a successful `TutorialGameMsgs` (`0xC8`)
subtype-`0` snapshot, so `new_player_progress` remains an incomplete value
(`0`, `1000`, or `2000`) rather than advancing to `3000`. At those incomplete
values the Spaceship navigation path disables ordinary room selection, and
`MapRoomUI.StartGame` selects game mode `1` (`Tutorial`) again. The expected
player-level flow is therefore failure screen -> Spaceship/tutorial entry -> a
fresh tutorial game lifecycle, not failure screen -> unlocked squad
management. The precise intermediate screens still require a clean retained
death trace.

## Why the other candidates are not the button request

### `ChainGameOverMsgs (0xAE)`

The client registers `0xAE` as an inbound message while gameplay is active.
Subtype `0` sets the next state to Spaceship. No direct client message builder
constructs logical message `47` / wire `0xAE`.

### `ReloadLevel (0xBF)`

The global inbound dispatcher registers logical message `64` and routes it to
the zero-payload reload handler. The executable's direct outgoing-message
constructors contain no logical-`64` builder.

### Blaze GameManager reset dedicated server (`0x0004/0x0019`)

This RPC creates/configures game membership. The local handler replies with a
`GID` and queues GameManager notifications `0x0F`, `0x14`, and `0x15`. Nothing
in `MaxisGameOver.OnExitClicked` calls this RPC. The historical candidate trace
also records the client immediately removing itself after the local server's
candidate setup, so that sequence is a failed game-setup experiment rather
than tutorial-restart evidence.

## Content and trace check

`content.db` contains the known compiled tutorial unlock/lesson chunks in
resource group `0x24F78AA1`; it contains no separately identified game-over,
death-HUD, or restart chunk. This agrees with the native ownership above: squad
death presentation and reload are engine/UI state-machine behavior, not a
recovered tutorial Lua primitive.

The mandated retained trace directory,
`bin/server/darkspin/logs/traces`, is absent/empty in this workspace. Notes
refer to historical `server-cookie-fix.jsonl` and
`server-tutorial-complete.jsonl` observations, but those files are not present
and neither observation was a clean death/restart capture. Therefore this note
does not invent reliability mode, ordering channel, datagram sequence numbers,
or a client request absent from the executable.

## Native evidence ledger

- Application vocabulary: `server/raknet/types.go` and the executable's
  `kGms*` table at `0x0118419C` onward establish wire IDs `0xAC`, `0xAD`,
  `0xAE`, `0xAF`, and `0xBF`.
- Active gameplay handlers: `sub_44DC60` reads the one-byte `0xAD` subtype;
  subtype `1` calls `sub_44B0A0`, which records state `13`.
- Game-over handler: `cGameOverState` is constructed by `sub_44A3F0`; its
  protocol callback `sub_44A470` reads exactly one byte from logical message
  `46` (`0xAE`) and accepts `0`/`1` without a state mutation.
- Server return transition: `sub_44DD60` reads the one-byte logical-message
  `47` (`0xAF`) subtype; subtype `0` calls `sub_44B150`, recording state `2`.
- Failure UI: `sub_44A2C0` enters game-over presentation and creates
  `HUD_Death.swf`; `sub_41B620` registers only
  `MaxisGameOver.OnExitClicked`; `sub_41B510` plays `outro`, and
  `sub_41B540` completes the local transition to state `2`.
- In-place reload: global dispatcher `sub_53ADC0` maps logical message `64`
  (`0xBF`) to `sub_537F60`. That function reads no payload and performs no
  transport operation.
- Outgoing-message negative check: direct `sub_A8FC00` constructors in
  `Game.c` include PlayerStatusUpdate and other client commands but no
  logical message `47` or `64`.

## Implementation implication (for later work)

No implementation is made here. A future restart feature must first choose an
explicit product contract:

- reproduce the native failure button and return to Spaceship, retiring the
  gameplay session; or
- add a server-driven `BF` in-place reload, preserving the RakNet peer while
  replacing every gameplay epoch, encounter, object map, and scheduled job.

Those are different lifecycle operations and must not share stale session
state.
