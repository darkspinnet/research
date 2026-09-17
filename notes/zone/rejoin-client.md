# Build-103 running-game rejoin

## Conclusion

The build-103 client explicitly supports rejoining an already-running game.
This is a live-session reconnect path, not a local save-game resume. The client
remembers the current game identity, asks Blaze to join that game again, binds
a new gameplay connection, accepts a server-selected client game state, and
then depends on the server to reconstruct the authoritative world.

For a campaign, the server can select game state `6` (`GameDungeon`) through
`S2C 0x81 ReconnectPlayer`. Reaching that state is only the state-machine
transition. A correct rejoin also requires a snapshot of the existing campaign
instance. Replaying the ordinary fresh-dungeon setup is not sufficient because
it can reset the hero, director, encounter, objective, and pickup presentation.

The client capability is high-confidence. The exact retail server policy for
rejoin grace time, packet ordering, and encounter snapshot contents remains
unrecovered.

## Recovered client evidence

### Current game identity

The account parser reads `current_game_id` and stores its numeric result:

- `bin/game/GameBin/Game.c:270600`
- parser function `sub_4AB910`, address `0x004AB910`

This field is separate from `current_playgroup_id`. Rejoining the game and
recovering party membership are therefore related but distinct operations.

### Reconnect state

The `SP_SporeLabs/cLoginState` path contains an explicit
`LABS_GAME_RECONNECT` transition:

- `bin/game/GameBin/Game.c:353333`

Its update routine, `sub_512060` at address `0x00512060`, performs the following
observable sequence:

1. Wait for the online platform connection.
2. Wait for the gameplay client session to become connected.
3. Register a callback for logical gameplay message `2`.
4. Require a nonzero current-game field.
5. Invoke the online-platform virtual operation with that game ID.
6. Continue through the client state transition once the operation succeeds.

The relevant body is at
`bin/game/GameBin/Game.c:353790-353886`.

The online-platform implementation logs:

```text
Client calls JoinGame with id [%u]
```

and invokes the Blaze GameManager join request:

- wrapper `sub_C25B00`, address `0x00C25B00`
- `bin/game/GameBin/Game.c:1882766`
- request implementation `sub_C3A3C0`, address `0x00C3A3C0`
- `bin/game/GameBin/Game.c:1894555`
- result callback `cBlazeGameManager::JoinGameResult`
- `bin/game/GameBin/Game.c:1894586`

This proves the reconnect state attempts to rejoin an existing Blaze game. It
is not merely a reconnect dialog or a return to the ship.

### Server-selected destination state

The callback registered for logical gameplay message `2` reads exactly four
bytes. A payload of `0xffffffff` takes the failure path; any other value is
passed to the global client game-state manager:

- callback `sub_511B40`, address `0x00511B40`
- `bin/game/GameBin/Game.c:353536`

The recovered application packet map identifies the message as:

```c
struct S2C_81_ReconnectPlayer {
    u32 game_state;
}; // 4 bytes
```

See `notes/client/packet.md:292`. State `6` is the dungeon state. Consequently,
the retail server could direct a reconnecting client back into a running
campaign instead of always returning it to the spaceship.

### Connection-loss awareness

The gameplay client session has explicit `OnConnectionLost` and
`OnConnectionClosed` handlers:

- `bin/game/GameBin/Game.c:1546423`

The complete route from every low-level disconnect reason into
`cLoginState` has not been recovered. Do not infer that every network failure
automatically retries forever. The supported claim is narrower: after the
client reaches the reconnect/login flow, it knows how to rejoin the recorded
game and accept a server-selected dungeon state.

## Intended protocol flow

The evidence supports this logical flow:

```text
gameplay connection is lost
    -> client authenticates again
    -> account response supplies current_game_id
    -> client sends Blaze JoinGame(current_game_id)
    -> client establishes a new gameplay connection
    -> normal gameplay identity/slot handshake completes
    -> server sends ReconnectPlayer(GameDungeon)
    -> server projects the existing campaign snapshot
    -> live campaign traffic resumes on the new peer generation
```

The exact ordering of `ReconnectPlayer` relative to every roster, merge, game
state, and snapshot packet is not yet proven by a retail capture. Preserve the
logical dependencies above without presenting a guessed packet sequence as an
exact retail contract.

## Current Dark Spin support

Several required pieces already exist:

- The account response exposes the runtime `current_game_id` in
  `server/game/api.go:670`.
- Blaze `JoinGame` resolves an existing game and admits the user in
  `server/blaze/game_manager_component.go:163`.
- `game.Instance.AddPlayer` treats the same existing user as an idempotent
  success in `server/game/manager.go:127`.
- Gameplay identity binding resolves `user.CurrentGameID()` and the existing
  player slot in `server/game/gameplay_session.go:116`.
- `server/zone/zone.go:131` accepts a newer peer generation for the same user,
  removes the superseded hero binding, and releases actions owned by the old
  connection generation.
- `raknet.StateMessage` encodes `ReconnectPlayer` in
  `server/raknet/application.go:137`.

These pieces establish game identity, membership continuity, generation
replacement, and the client state transition. A live-client transport rejoin
appends `ReconnectPlayer(GameDungeon)` after its baseline. Durable launcher
Continue instead reloads the level through `GamePrepareForStart` and
`GameStart` before publishing restored state.

### Known gaps

1. Exact retail ordering and reliability for `ReconnectPlayer` relative to a
   live reconnect baseline remain unproven. A `16:17` launcher-restart trace
   proved this opcode cannot bootstrap an unloaded process: build 103 accepted
   state 6 and crashed in native world lookup at `0x009c8e75` because no dungeon
   scene existed. Durable Continue therefore uses the ordinary level-loading
   handshake rather than this live-session opcode.
2. `gameplayJoinRuntime.handle` stores sessions by UDP address in
   `server/gameplay_udp.go:898`. A reconnect from a new address does not
   directly locate and replace the prior connection by authenticated user and
   game identity.
3. `marshalCampaignDungeonSetup` in `server/gameplay_udp.go:642` is an initial
   setup projection. It emits a blank director state, creates the squad at the
   campaign entry position, and deploys creature zero. Using it unchanged for
   rejoin would lose or contradict live instance state.
4. Shared campaign subsystems retain portions of the world, but there is no
   single coherent rejoin snapshot covering the entire instance.
5. Active games and their simulation state are process memory. Rejoin after a
   server restart is not currently supported and must not be implied by
   same-process reconnect support.

## Refactor instructions

### 1. Model rejoin as a feature operation

Create one server-owned rejoin operation that decides whether an authenticated
actor may resume a game. Blaze and RakNet adapters must decode their protocol
messages, call that operation, and encode its typed result. They must not
independently mutate user, game, zone, or session state.

The operation should consume small feature-owned ports for:

- locating the actor's claimed game;
- validating retained game membership;
- inspecting the game and campaign lifecycle;
- replacing the actor's peer generation;
- obtaining an immutable campaign snapshot; and
- committing or rejecting the replacement.

Keep transport addresses, RakNet packet IDs, Blaze TDF fields, and encoded
bytes out of the feature result.

### 2. Separate the three admission phases

Treat these as distinct phases with explicit results:

1. **Account/game discovery:** expose a nonzero current game only while the
   actor has a resumable membership.
2. **Blaze admission:** validate `JoinGame(game_id)` against that membership
   and the live game lifecycle.
3. **Gameplay activation:** bind the authenticated gameplay peer, replace the
   previous peer generation, send the selected state, publish a snapshot, and
   only then accept ordinary player commands.

A successful Blaze join must not by itself imply that snapshot publication
succeeded. Gameplay activation should remain retryable after an encoding or
transport failure.

### 3. Use stable identity and replaceable connection generations

Use `(game ID, user ID)` as the stable membership identity. Treat socket
address, RakNet peer, and peer generation as replaceable connection state.

For a newer authenticated generation:

- reject a stale generation;
- quiesce commands from the old generation;
- release old-generation action leases and scheduled connection-owned work;
- attach the new generation to the same game and zone membership;
- retain instance-owned world state; and
- make duplicate delivery of the accepted generation idempotent.

Do not key authoritative membership solely by UDP address. An address may
change across reconnects, and an old address may remain observable until its
transport timeout.

The transition must not leave two generations able to control the same hero.
If snapshot preparation or commit fails, retain a clearly defined retryable
state rather than partially activating the new peer.

### 4. Give the game lifecycle an explicit reconnect policy

Represent at least these lifecycle outcomes in the feature layer:

- resumable active game;
- resumable transition state, if supported;
- completed game with a pending result/cash-out projection;
- expired or retired game;
- actor is not a member;
- game no longer exists; and
- already active on the same generation.

Choose and document a bounded grace policy before retiring an empty active
instance. The duration is server policy because no exact retail duration has
been recovered. Keep the game playable by retaining a disconnected campaign
long enough for an ordinary login and gameplay handshake.

Do not clear the actor's current game merely because a TCP or UDP connection
closed. Clear it through the game lifecycle operation when the game is
completed, explicitly left, expired, or otherwise no longer resumable.

### 5. Define a transport-neutral campaign snapshot

The snapshot must describe the current authoritative instance rather than
reconstructing state from the reconnecting peer's former adapter. At minimum,
consider:

- game mode, client destination state, game time, and campaign phase;
- roster, player slots, connection presence, and party membership;
- the reconnecting player's squad, active creature, deployed object, health,
  power, cooldowns, modifiers, position, orientation, and movement state;
- other players and their currently visible controlled objects;
- live and dormant enemies, health, modifiers, transforms, target/threat
  state, and current actions where the client requires them;
- pickups, loot ownership/eligibility, DNA, crystals, and collected state;
- director, horde, wave, boss, and encounter completion state;
- objectives, checkpoints, gates, teleporters, interactables, and one-shot
  trigger state;
- outcome, chain voting, cash-out, victory, or defeat state when applicable;
  and
- any retained presentation baseline required before incremental events can be
  understood.

The feature snapshot should contain semantic state. A build-103 projection
adapter should explicitly allowlist and encode the necessary packets.

Do not reset the hero to the level entry, refill resources, replay one-time
spawns, reactivate completed encounters, or reroll loot as a side effect of
snapshot generation.

### 6. Project a baseline before incremental traffic

Pause or queue incremental instance events for the joining subscriber while
the snapshot is captured and projected. Establish a snapshot revision or event
cursor so events accepted during projection are delivered exactly once after
the baseline.

For a client that retained its loaded dungeon, the adapter should:

1. complete authenticated gameplay identity and slot binding;
2. create the currently relevant objects before referring to them;
3. publish component and semantic baselines in dependency order;
4. encode the feature-selected destination through
   `ReconnectPlayer(GameDungeon)` for an active campaign only after its world
   baseline exists;
5. catch the subscriber up from the snapshot revision; and
6. mark the peer active for ordinary commands.

If retail evidence later proves a different wire order, change only the
projection adapter. The feature operation and snapshot must remain
transport-neutral.

A restarted client must not use this shortcut. It needs
`GamePrepareForStart`, its normal loading and ready statuses, and `GameStart`
before the adapter publishes the restored world.

Use `0xffffffff` only for the recovered reconnect failure meaning. Map failure
reasons to typed feature outcomes first, then let the adapter choose the
appropriate client response and safe destination.

### 7. Preserve instance state across ordinary disconnects

Connection-owned work may be cancelled or transferred, but a single player's
disconnect must not reset or retire shared:

- enemies and their damage;
- encounter and director progression;
- objectives and one-shot triggers;
- pickups and loot;
- boss state;
- result state; or
- another member's actions.

Define ownership for every timer and scheduled action. Instance-owned work
continues while the instance remains active. Old-peer work is cancelled when
its generation is replaced. Avoid closures that capture mutable peer or
instance state; use small feature-owned types with named operations.

### 8. Keep restart recovery out of scope unless it is implemented explicitly

Same-process rejoin only requires retaining a live in-memory instance.
Cross-process resume requires durable snapshots, versioning, timer recovery,
content compatibility, and transactional restoration. Do not advertise or
implicitly test restart recovery until those concerns have a deliberate
design.

### 9. Make observability identify the rejoin generation

Log and trace:

- game ID;
- user ID;
- old and new peer generation;
- lifecycle admission result;
- snapshot revision;
- number of projected objects/events;
- selected client game state; and
- rejection or rollback reason.

Do not log credentials or serialize an unrestricted user, storage row, or
feature aggregate into a client-visible trace.

## Required tests

Add focused tests at the operation and adapter that owns each invariant:

1. An authenticated member reconnects to the same active game and retains the
   same player slot.
2. Repeated Blaze `JoinGame` for the retained member is idempotent.
3. A newer gameplay generation replaces the old generation exactly once.
4. Commands from the replaced generation are rejected without mutating the
   instance.
5. A nonmember cannot claim an arbitrary active game ID.
6. A missing, expired, or completed non-resumable game returns a typed
   rejection and performs no storage or instance write.
7. Snapshot or projection failure does not leave two active peer generations
   or consume one-shot world state.
8. The reconnect snapshot preserves hero position, health, power, active
   creature, and modifiers.
9. The reconnect snapshot preserves a partially completed encounter,
   objective, live enemy damage, pickups, and boss/director state.
10. Snapshot-plus-catch-up delivers events accepted during projection once and
    in order.
11. One member reconnecting does not reset another member or the shared world.
12. The build-103 adapter emits an exact four-byte
    `ReconnectPlayer(GameDungeon)` fixture.
13. The failure fixture emits the recovered `0xffffffff` payload only for a
    rejected reconnect.
14. A human test disconnects the client during campaign 1-1, logs back in
    before expiry, and resumes the same instance without duplicated spawns,
    rerolled loot, or an entry-position reset.

## Remaining evidence targets

The following need a retail trace, debugger observation, or additional client
analysis before they can be called exact:

- the reconnect grace duration;
- whether the client retries automatically or requires user confirmation for
  each disconnect class;
- exact ordering and RakNet reliability/channel for `ReconnectPlayer` and the
  following snapshot;
- which transition, voting, cash-out, cinematic, spectator, and game-over
  states retail considered resumable;
- the exact late-join/reconnect representation of live director, horde, boss,
  threat, pickup, and objective state; and
- the failure presentation selected after `0xffffffff`.

Until those are recovered, keep policy conservative, server-authoritative, and
compatible with campaign 1-1. Record any significant fallback in
`notes/help.md` using the workspace decision format.
