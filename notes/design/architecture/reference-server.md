# ReCap.Server comparison and imported findings

## Scope

This pass compares the local `ReCap.Server` C# tree against darkspin. The C#
implementation is useful corroborating evidence, but it is not automatically a
protocol authority: its own planning notes identify hardcoded and incomplete
behavior. Retail-client analysis and captured wire behavior remain the contract.

Evidence grades in this note are intentionally distinct:

- Gameplay scalar little-endianness is capture-confirmed in the reference.
- The opcode ordering is confirmed by the retail client's message-name table.
- The session payload shapes are implemented from the reference packet classes
  and legacy server source; they still require a paired darkspin/client trace.
- The client-side game-type range is statically confirmed. The reference's
  reported UI failure mechanism is strong diagnostic evidence but still lists
  a live client verification gate.

## Feature comparison

| Area | ReCap.Server | darkspin | Result |
| --- | --- | --- | --- |
| RakNet transport | Uses RakNexus and maintains connected sessions. | Implements offline negotiation and reliability codecs, but is not started by `server.Server` and has no connected-peer lifecycle. | ReCap.Server is ahead. This remains darkspin's first multiplayer blocker. |
| Gameplay packets | Full `0x7f`-`0xcc` opcode table and 43 packet classes, including object, combat, modifier, objective, locomotion, and cooldown messages. | Had a sparse opcode table and codecs for action commands, state, and a few payloads. | The full client opcode map and safe session codecs were imported. Complex reflection packets still need independent byte tests before porting. |
| Game simulation | Has a tick-owned inbound queue, object manager, level population, movement, ability dispatch, combat/death, and partial objectives. | Has game/session models, noun and asset loading, and Lua scheduling, but not an integrated authoritative simulation. | ReCap.Server supplies a useful behavioral design. Port behind darkspin application/domain boundaries after transport is connected. |
| AI and scripting | Has native Lua 5.1 integration, ability natives, controller/aggro tests, and early AI models. Its own matrix still identifies typed noun data, collision, and production AI as gaps. | Loads Lua/assets and schedules ticks, but has no equivalent authoritative object/AI loop. | Port concepts and tests, not the C# service structure or P/Invoke boundary. |
| Persistence | Uses EF Core with SQLite repositories for accounts, creatures, parts, and decks. | Uses a `UserRepository` boundary with an XML filesystem adapter. | Keep darkspin's interface. SQLite/PostgreSQL can be added as adapters without moving policy into persistence. Deck ownership and update invariants are useful candidates to reproduce in the domain layer. |
| Blaze | Reference matrix estimates roughly 35-40% of the legacy wired handler surface. | Already supports redirector, auth/persona bootstrap, user-session notifications, rooms, messaging, playgroups, game manager, tracing, and consolidated listeners. | Do not replace darkspin's Blaze stack wholesale. Compare missing GameManager lifecycle and utility settings command-by-command. |
| HTTP and QoS | Implements more mutation endpoints and `game://` web assets, but its matrix reports missing QoS. | Has the HTTP API, static routing, readiness, tracing, and a live-confirmed QoS responder; several mutations remain partial. | Port domain mutations such as deck updates. Retain darkspin's QoS and HTTP lifecycle. |
| Operations | Conventional C# process with SQLite and file logging. | TOML config, organized `bin/game` and `bin/server` runtimes, paired traces, Mage launch/debug tasks, and injectable client hooks. | darkspin is ahead; no migration needed. |

## Findings imported in this pass

### Server-side Lua playback references

Both adjacent ReCap implementations confirm that authored Lua should run inside
the authoritative game simulation rather than being reproduced as an unrelated
client-side launch-DLL timeline.

`C:\src\recap\ReCap.Server` is the stronger architectural reference. It creates
one booted Lua 5.1 state per `GameScriptContext`, resolves packaged
`Group!Name.lua` modules through `ScriptVfs`, runs ability/objective functions as
coroutines, advances `WaitForXSeconds` against a game-owned simulation clock,
and exposes game state through an explicit `IScriptGameBridge`. Lua entry is
serialized with the game loop and script failures terminate the affected
coroutine rather than the server. This closely matches the intended darkspin
shape: per-session isolation, deterministic scheduling, and typed/allowlisted
Go operations behind native bindings.

`C:\src\recap\recap_server` is an earlier but genuinely executable behavioral
reference. Its C++ `Lua`, `LuaThread`, and `Ability` types load authored ability
scripts, use Sol coroutines with resume conditions, implement
`nThread.WaitForXSeconds`, dispatch ability ticks, and connect Lua operations to
object movement, animation, cooldown, combat, and server-event messages. Its AI
path also invokes the noun's `firstAggroAbility` once on initial aggro. It is
therefore useful for native-call semantics and authored ability lifecycle even
where the newer C# implementation is easier to adapt architecturally.

Neither tree currently completes the exact tutorial cinematic path. The C#
tree enumerates build-103 `CinematicMsgs` (`0xC9`) and documents
`nGameSimulator.StartCinematic`, but `NGameSimulatorModule` presently registers
only `GetGameTime` and `IsChainGame`, while its packet activator leaves
`CinematicMsgs` uninstantiated. The C++ scripting registrations do not expose a
working `StartCinematic` binding either. It does contain an unwired
`Server::SendCinematic` probe with the correct 25-byte width, but build-103
receiver recovery shows that its two apparent `uint32` test fields are actually
one signed little-endian `int64` duration. Three position floats and a radius
float follow, also little-endian, with no subtype byte. The probe is therefore
corroborating layout evidence, not a verified semantic implementation: the
source labels its fields unknown and does not connect it to Lua or normal
gameplay. darkspin can reuse the runtime and scheduler lessons, but still must
bridge the native to the connected authoritative tutorial session.

### Gameplay endianness boundary

The reference's `VERIFIED_FACTS.md` records raw-capture comparison against the
working C++ server: Game gameplay scalars are little-endian on the wire.
RakNet transport framing remains network/big-endian. darkspin previously used its
big-endian transport `Writer` for some application payloads even though action
commands were already decoded little-endian.

darkspin now keeps the transport codec explicitly documented as transport-only,
decodes `PlayerStatusUpdate` as little-endian, and emits game-state/objective
scalars as little-endian. Wire-byte tests cover the boundary.

### Complete opcode vocabulary

The packet ID table is now complete from `0x7f` through `0xcc`. Of particular
importance, `0x9d` through `0xa0` use the retail client's contiguous message
name table ordering (`PlayerDamage`, `LootSpawned`, `LootAcquired`,
`SystemMessage`), rather than the shifted ordering in the older C++ enum.

Declaring an opcode does not mean its behavior is implemented. The constants
make traces readable and prevent future handlers from copying the known-bad
shifted values.

### Session and dungeon-entry messages

darkspin now has application-message encoders and byte-level tests for:

- `0x7f` HelloPlayerRequest: little-endian user ID plus optional playgroup ID.
- `0x80` HelloPlayer: type, gameplay slot, raw IPv4 bytes, and little-endian port.
- `0x81` ReconnectPlayer: little-endian game state.
- `0x82` Connected: empty body.
- `0x84` PlayerJoined: one-byte slot.
- `0x85` PartyMergeComplete: little-endian timestamp.
- `0xaf` QuickGameMsgs: `reset=1` for dungeon entry.
- `0xb1` GameStart: little-endian level index.
- `0xcc` DebugPing-compatible timestamp payload through `TimestampMessage`.

These codecs are ready for the connected-session dispatcher. They are not yet
sent by the process because bypassing the RakNet connection and reliability
state would create misleading success on localhost and fail under real packet
ordering or loss.

### Persistent deck updates

The reference's deck service supplied a clean domain rule worth importing:
only creatures owned by the account enter a deck, accepted IDs compact into
exactly three positions, and an unknown active slot is a no-op. darkspin now
decodes the confirmed `pve_creatures`, `pvp_creatures`, `pve_active_slot`, and
`pvp_active_slot` request keys and calls `Manager.UpdateDecks` instead of merely
acknowledging the request.

The manager owns persistence, rejects locked squads before writing, and restores
the previous aggregate if the repository save fails. Tests cover successful
ownership filtering, rejected invariants with no write, and rollback. The code
continues to use `UserRepository`; no filesystem or future PostgreSQL detail is
visible to the HTTP adapter or domain rule.

### Shared Blaze membership

The C# server demonstrates the correct architectural direction—one game owns a
stable player index and broadcasts state to every member—but its current
`OnPlayerJoined` helper has no caller and its reset path assigns the only human
to slot zero. darkspin now allocates the lowest free game-local slot, reuses a
slot after departure, and sends a complete setup roster to the joining member
while reserving incremental joining records for existing members. Game-state and departure
notifications fan out through authenticated Blaze sessions as well.

Two-user tests assert that the existing player sees the new slot, the joining
player receives the full roster, both members receive state/departure events,
and capacity and slot reuse remain domain invariants. This improves the Blaze
control plane; it does not claim shared gameplay until RakNet peers bind to the
same membership records.

### Login cadence and durable progression

The paired 5.3.0.103 trace remains the authority for the live main-session
cadence: login (`1/0x28`), LoginPersona (`1/0x6e`) plus UserAdded/UserUpdated,
PostAuth (`9/8`), UpdateNetworkInfo (`0x7802/0x14`), GetAuthToken (`1/0x24`),
then UpdateUserSessionClientData (`0x7802/0x19`) and periodic ping. The C# flow
documentation also shows that UpdateNetworkInfo must be followed by
UserSessionExtendedDataUpdate (`0x7802/1`, `DATA` + `USID`). darkspin now sends
that notification, preserves the client-reported address union, and sets the
multiple-location suppression attribute used by the reference.

Build 103's exact `NotifyUserAdded` visitor exposes only `DATA` and `USER`; it
does not advertise Darkspin's internal TCP connection number. Presence `USID`
follows the SDK-visible identity anchored by `USER.ID`. Playgroup and game-player
`SID` are unrelated one-byte slot IDs, proven by their generated visitors, and
must use the stable zero-based roster slot rather than any user identity.

Progression writes no longer depend on a later logout. `account.unlock`,
`account.setSettings`, and `account.setNewPlayerStats` now invoke application
operations that save immediately and roll back on repository failure. Account
responses include the persisted new-player, star, cap, currency, and complete
unlock vocabulary recognized by the 5.3.0.103 parser. Settings are persisted
as data and returned when requested. A graceful server shutdown flushes all
active users as a final safety net; process termination can still interrupt
work that was never submitted as a mutation command.

### GameState mode

The reference client analysis identifies the gameplay-mode enum as
`Tutorial=1` through `DirectEntry=7`; zero is invalid. Campaign/dungeon flow
uses `Chain=2`. darkspin now models this as `GameType` and encodes the reference
GameState body as two little-endian `u64` times, one state byte, a little-endian
game type, and the trailing little-endian constant `1`.

The reference reports that `QuickGameMsgs.reset=0` and `GameType=0` route the
client into an incompatible map-room UI path. darkspin's new types make those
invalid defaults harder to send accidentally.

## Recommended import order

1. Implement connected RakNet peers, ACK/NAK retransmission state, ordering,
   fragmentation reassembly, and disconnect cleanup; start it from
   `server.Server` on the advertised gameplay port.
2. Link `HelloPlayerRequest.UserID` to the authenticated Blaze user and active
   game. Business logic must allocate the player slot; the UDP adapter only
   decodes and encodes.
3. Reproduce the reference join sequence (`Connected`, `HelloPlayer`, existing
   and joining `PlayerJoined`, `PartyMergeComplete`) with a two-client test.
4. Add a single-owner game loop. Network goroutines enqueue commands; only the
   game loop mutates objects, AI, cooldowns, and objectives.
5. Port object/reflection packets from captured fixtures, followed by level
   population, movement, combat, AI, and finally loot/cash-out. Each packet
   needs an exact byte fixture before its behavior is enabled.
6. Port persistent deck and inventory mutations through application operations
   guarded by domain ownership/invariant checks. Add PostgreSQL later as another
   `UserRepository` adapter.

This order uses ReCap.Server's strongest work without inheriting its remaining
hardcoded game lifecycle or coupling transport, simulation, and persistence.
