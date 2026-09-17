# Party invite and two-player zone join audit

## Current implementation increment

Zone hero identity reserves one three-creature object-ID block for each of the
four stable game slots. Shared script objects, fixtures, NPCs, effects, and loot
begin only after those twelve IDs. Zone admission rejects duplicate slots,
activates each member's objective slot, and projects remote rosters, movement,
resources, squad deployment, damage, defenses, death selection, reactions, and
stats through the target member's own connection-local session. Blaze publishes
a stable host and complete deterministic roster to joiners, waits for the
authored expected member count to join and report ready, and retries the
all-member start publication until it commits. RakNet proactively drains
revisioned peer outboxes every 100 ms, so remote state no longer waits for the
target player to send another packet.

This removes the identity and target-ownership collisions that previously made
a second member architecturally unsafe. Party-backed Blaze reservation and the
native Invite flow remain required before a two-client run is ready.

The first live two-client campaign attempt on 2026-08-20 recovered a concrete
member-side ordering. After the leader's reset-dedicated-server request, both
clients received one complete two-player `NotifyGameSetup`. The reserved member
then sent `DestroyGame` followed by `UpdateMeshConnection`; the leader sent
`FinalizeGameCreation` followed by its mesh update. The old handler allowed the
member's destroy request to retire the shared instance, so the leader's first
RakNet Hello failed with `gameplay game not found` and the client displayed its
generic account/start-planet errors. Game destruction is now host-authorized,
both `PHST.HSLT` and `THST.HSLT` carry the frozen host's actual game slot, and a
reserved member's mesh update satisfies that member's ready barrier. This is
observed build-103 behavior, while the reason the client emitted the member-side
destroy remains a follow-up capture item if the corrected host projection does
not suppress it.

The second live attempt confirmed that correction, while also exposing two
independent roster-completion defects. The shared start publisher first emitted
a duplicate peer `NotifyPlayerJoinCompleted`; it now sends only the shared game
state and player statuses. Gameplay admission also used to publish
`PartyMergeComplete` before both frozen slots connected. It now answers the
local Hello, exchanges `PlayerJoined` for each actually connected slot, and
publishes merge-complete only when the connected count and slot mask equal the
frozen game roster. This ordering is a client-constrained compatibility fallback
pending a retail multiplayer capture.

The 2026-08-20 03:30 detached-client minidump then proved a remaining Blaze
identity defect. The process faulted with `0xc0000005` at
`Game.exe+0x83aa07` (`sub_C3A9D0`,
`cBlazeGameManager::onPlayerJoinComplete`) while dereferencing a null joining
player at offset `+0x58`. The trace contains only Rawr's expected local
join-completed notification, after a complete two-player setup, and RakNet was
still correctly waiting at connected mask `0x1` for expected mask `0x3`.
Both `ReplicatedGamePlayer` records had initially been projected with `SID=0`,
and a first correction incorrectly changed that field to the member's persona
ID. The exact visitor at `0x00DEC0B0` proves `SID` is an 8-bit player slot ID,
while `SLOT` is the `SlotType` enum, `PID`/`UID` are identities, and `TIDX` is the
team index. Game player projections now send each stable player slot through
`SID`, public-participant type through `SLOT`, preserve persona IDs in
`PID`/`UID`, and publish the host SDK identity through `HSES` instead of a fixed
placeholder.

## Milestone and scope

The implementation milestone is a two-client human playtest in which two authenticated launcher profiles use the ship Invite UI, become members of one party and Blaze game, enter campaign 1-1, see and affect the same live world, and independently disconnect or leave without corrupting the remaining member.

This work has two feature boundaries:

1. **Party and initial admission** own ship invitation, party membership, game reservation, stable player slots, and the first bind of an admitted user to a zone.
2. **Zone rejoin** owns replacement of a disconnected member's transport generation and delivery of a coherent current-world snapshot. Its snapshot and generation work is shared with initial projection, but it must not become the authority for party formation or first-time admission. See [zone rejoin](rejoin.md) and its [build-103 client evidence](rejoin-client.md).

This is an implementation audit, not protocol completion. No client asset or runtime database change is required or permitted.

The baseline review covered `notes/todo/co-op-instance.md`, `notes/ship/overview.md`, `notes/client/packet.md`, the current rejoin audit at `notes/zone/rejoin.md`, its client evidence at `notes/zone/rejoin-client.md`, build-103 `bin/game/GameBin/Game.c`, and the current Blaze GameManager, Playgroups, Messaging, Rooms, game, gameplay, zone, projection, and RakNet paths cited below.

## Executive result

Build 103's ship Invite route is not a Rooms invitation and is not a single Playgroups request. It is a combination of:

- **Playgroups** for party creation, membership, roster, leader, join controls, leave, and destroy;
- **Messaging** for the direct invitation envelope, addressed to one user and carrying the inviter's playgroup ID in message attribute `720897`;
- **GameManager** after acceptance, for the separate live game ID, player slots, game setup, player notifications, and start state;
- **Rooms** only for lobby/social discovery and lobby chat. Rooms do not own the party.

The original server could not complete that route. Playgroups commands returned empty success responses without state or notifications, and Messaging expected a chat body that the native invite did not supply. Those admission and invitation boundaries are now implemented; the remaining work is live validation of the corrected shared-start transition and the first two RakNet gameplay binds.

Even if both users are forced into one game, campaign play is not yet safe. Both peers assign their three squad creatures object IDs `1`, `2`, and `3`; most player movement and action output is returned only to the invoking peer; an already active world has no complete snapshot; and an idle peer receives projection only when its own RakNet traffic happens to trigger polling. These are milestone blockers, not polish.

The shortest safe route is therefore:

1. implement an in-memory, feature-owned direct party invite flow for two known active local users;
2. freeze the accepted party roster into a game admission record with stable slots and host ownership;
3. require both Blaze joins before campaign start;
4. allocate collision-free hero identities and construct the zone once from the frozen roster;
5. add a snapshot-plus-cursor projection boundary and a transport outbox that can wake an idle peer;
6. broadcast player and shared-world semantic changes to both members;
7. retain a disconnected member during a bounded rejoin grace period, while explicit leave removes only that member.

## Evidence map and confidence

Confidence meanings:

- **Exact**: a build-103 branch, generated Blaze contract, or current executable server path directly establishes the behavior.
- **Constrained**: several exact observations bound the implementation, but a wire capture is still needed to settle ordering or optional fields.
- **Fallback**: client/content evidence does not determine server authority; the proposed policy is conservative and keeps campaign 1-1 playable.

| Finding | Confidence | Evidence |
| --- | --- | --- |
| Ship Invite uses Playgroups plus direct Messaging | Exact | `Game.c:361500` sends the invitation after ensuring a playgroup; `Game.c:1891500` builds the Messaging request; `Game.c:1891834` consumes attribute `720897` as the sender's playgroup ID |
| Rooms do not own party membership | Exact | Current Rooms component and `sporenet.RoomManager` expose lobby/category/room membership; the client Invite callback reaches the Playgroups/Messaging facade instead |
| Invite prompt lifetime is 30 seconds | Exact for client UI | Incoming and outgoing invite state stores `30.0` near `Game.c:360700` and `Game.c:361500`; expiry UI is near `Game.c:361761` |
| Server must enforce its own expiry | Constrained | Client-local expiry is exact, but no captured server cancellation/expiry exchange establishes the retail authority |
| Playgroup maximum is four and creation uses network topology `4`, join control `1` | Exact values, semantics partly constrained | `cBlazePlaygroupsManager` creation path near `Game.c:1887119` |
| `current_playgroup_id` and `current_game_id` are distinct | Exact | Account projection in `Game.c:270600` and `Game.c:270616`; current runtime user also stores separate claims |
| GameManager creation consumes the active playgroup and publishes `ExpectedPlayerCount` and `PlaygroupKey` | Exact | Game creation/join completion near `Game.c:1894000` and `Game.c:1894921` |
| Exact retail trigger by which the second accepted member learns and joins the game | Blocked | Client paths prove JoinGame and playgroup-aware game creation, but the initiating notification/order is not captured |
| RakNet Hello can carry user ID plus playgroup ID | Exact | [packet notes](../client/packet.md), `server/raknet/application.go`, and `server/gameplay_udp.go:1911` |
| `PlayerJoined`, `PartyMergeComplete`, and `PlayerDeparted` exist | Exact | [packet notes](../client/packet.md), `server/raknet/types.go:12` |
| Their retail multi-peer ordering | Constrained | The server now uses conservative Hello, `PlayerJoined`, then merge-complete ordering after every frozen participant connects; retail capture and `PlayerDeparted` remain outstanding |
| Current projection polling is sufficient for continuous co-op | Disproved | `server/raknet/server.go:613` polls on client-driven connected traffic; no bounded server push cadence exists |
| Build-103 party health and damage formulas | Exact as client calculation | [difficulty notes](../campaign/difficulty.md) and `server/sim/difficulty.go:46` |
| Retail server population, loot ownership, reward, and wipe policy | Blocked | Client content provides inputs and presentation calculations, not the missing server-authority decisions |

## Build-103 ship Invite route

### UI entry and invitation envelope

The ship social browser registers `inviteToGroup` in the main web view near `Game.c:173382` and `Game.c:193900`. Party presentation uses `MaxisParty`/`Party.swf`; the modal response uses `MaxisPartyInvite`/`popup_mini.swf` and `MaxisPartyInvite.OnChoiceClicked` near `Game.c:190622`.

The route is:

1. The inviter selects `inviteToGroup` for a user.
2. The client asks the Playgroups facade for its current group.
3. If it has no group, it creates one. The client builds a name of the form `<persona>'s Playgroup`, a maximum of four members, network topology `4`, and join control `1`.
4. The client sends a direct Blaze Messaging message to the target user. Its attribute map contains key `720897`; the value is the inviter's playgroup ID formatted as a decimal string.
5. The recipient Messaging dispatcher finds key `720897`, parses the playgroup ID, stores the invitation, and opens the 30-second party invitation prompt.
6. Accept invokes Playgroups Join by ID. Decline sends a Messaging response with status `2`. The exact outer TDF fields for that decline response remain capture-dependent.

The build-103 generated visitors constrain the accepted join projection. The
`NotifyJoinPlaygroup` envelope contains `INFO`, `MLST`, and the joining persona
as integer `USER`; `NotifyMemberJoinedPlaygroup` contains member structure
`MEMB` plus `PGID`. Each `PlaygroupMemberInfo.USER` is a complete
`UserIdentification` with `AID`, `ALOC`, `EXBB`, `EXID`, `ID`, and `NAME`, while
`PlaygroupInfo.HNET` is a union and `XNNC`/`XSES` are binary fields. Substituting
`INFO` for `MEMB`, omitting the scalar `USER` or nested `ID`, or changing those
wire types leaves the client with an invalid joined roster and can crash during
acceptance cleanup. `PlaygroupMemberInfo.SID` and `PlaygroupInfo.HSID` are
one-byte zero-based member slots, while nested `USER.ID` carries the persona.
Publishing the persona or zero for every member prevents the existing member's
incremental notification from adding the invitee even when the invitee's full
roster works.

The recipient explicitly logs that an invitation without the sender's playgroup ID is invalid. Therefore a server-side shortcut that merely emits a generic chat or UI notification is not compatible with the native route.

### Playgroups contracts

The generated Playgroups command enumeration near `Game.c:2313873` is:

| Command | ID |
| --- | ---: |
| CreatePlaygroup | `0x01` |
| DestroyPlaygroup | `0x02` |
| JoinPlaygroup | `0x03` |
| LeavePlaygroup | `0x04` |
| SetPlaygroupAttributes | `0x05` |
| SetMemberAttributes | `0x06` |
| KickPlaygroupMember | `0x07` |
| SetPlaygroupJoinControls | `0x08` |
| FinalizePlaygroupCreation | `0x09` |
| LookupPlaygroupInfo | `0x0a` |
| ResetPlaygroupSession | `0x0b` |

The generated notifications near `Game.c:2313964` are:

| Notification | ID | Minimum role in milestone |
| --- | ---: | --- |
| DestroyPlaygroup | `0x32` | Clear both clients' party state |
| JoinPlaygroup | `0x33` | Give joining client full group plus member list |
| MemberJoined | `0x34` | Update existing member roster |
| MemberRemoved | `0x35` | Remove leaver/kicked member with reason |
| PlaygroupAttributesSet | `0x36` | Carry `PlaygroupKey`/`StayInParty` if used |
| MemberAttributesSet | `0x4b` | Preserve compatible per-member state |
| LeaderChange | `0x4f` | Update party leader |
| MemberPermissionsChange | `0x50` | Not required until permission behavior is recovered |
| JoinControlsChange | `0x55` | Keep party admission UI consistent |

`MemberRemoved.MLST` contains the departing member's zero-based playgroup slot
(`SID`), not their account or persona ID. The client indexes its native roster
by that slot; projecting the account ID acknowledges the leave but leaves the
remote member row resident.

Known request/notification tags include:

- destroy: `PGID`, `REAS`;
- join: `PGID`, `PNET`, `UKEY`, `USER`;
- leave: `PGID`;
- kick: `EID`, `PGID`, `REAS`;
- member removed: `MLST`, `PGID`, `REAS`;
- leader change: `HSID`, `LID`, `PGID`;
- group/member structures include `INFO`, `JTIM`, `ATTR`, `NTOP`, `OWNR`, `PRES`, `UPRS`, `UUID`, `VOIP`, `XNNC`, and `XSES`.

Known Playgroups errors are:

| Condition | Error |
| --- | ---: |
| Not in group | `65542` |
| Not authorized | `131078` |
| Group full | `196614` |
| Invalid entity | `262150` |
| Group not found | `327686` |
| Group closed | `393222` |
| User not in any group | `458758` |
| Already exists | `524294` |
| Already in group | `589830` |

The minimum implementation must return the exact structures consumed by build 103, not just empty successes. A Blaze fixture test should be added for every command and notification used by the two-client path.

### Lifecycle semantics

Party state is now a feature aggregate rather than state hidden in Blaze sessions:

```text
Party
  ID
  LeaderUserID
  JoinControl
  Revision
  Members[UserID] = {joinOrdinal, attributes}
  Invites[InviteID] = {inviter, invitee, createdAt, expiresAt, status}
```

Required operations and outcomes:

| Operation | Accepted result | Rejection rules and notifications |
| --- | --- | --- |
| Invite | Create pending invite; ensure inviter's party exists; send native Messaging envelope | Reject self, unknown/offline target for the direct-local milestone, target already in party, full/closed group, duplicate pending invite |
| Accept | Atomically consume pending invite and add invitee | Reject expired/cancelled/consumed invite, changed/full party, or invitee in another party; no membership write on rejection |
| Reject | Mark invite rejected and send response to inviter | Idempotent for the same terminal invite; never mutate roster |
| Cancel | Mark inviter-owned pending invite cancelled and notify invitee | Only inviter or current party leader; capture still needed for the exact native cancellation presentation |
| Expire | Mark pending invite expired at server deadline | Client's 30-second UI timer is not sufficient authority |
| Leave | Remove actor, update both rosters and user projection | Non-leader leaves independently; explicit leave is not a transport disconnect |
| Disband | Destroy party and clear all members | Leader-only while more than one member unless leader migration is selected |
| Leader leave | Promote lowest surviving join ordinal, then notify | Conservative fallback; build 103 has a leader-change notification but retail selection policy is not recovered |

For the first milestone, leader migration is safer than silently destroying the other user's party. Once a live game exists, party leadership and game host ownership must remain separate: the dedicated server does not need to migrate a network host when the ship party leader changes.

The current implementation lives in `server/party`. It serializes membership and invitation mutations, enforces the 30-second server deadline, projects committed membership onto active users through `server/party/local`, and exposes immutable snapshots to Blaze, chat, and GameManager. Blaze now forwards the native attribute-`720897` invitation without requiring a chat body and rejects offline recipients before recording an unusable invitation. Party-led campaign reset freezes the current ordered roster into compact stable slots, closes admission, and sends the complete Blaze game setup and player roster to every authenticated member, so a later reserved-member JoinGame call is idempotent while an arbitrary user cannot enter by guessing the game ID. Login reconciliation restores the party projection, and same-ID game admission replaces a stale user pointer without changing the member slot or host identity.

The client maintains `StayInParty` and `PlaygroupKey` attributes. Preserve and round-trip them, but do not make untrusted client attributes the authority for membership, admission, or game ownership.

## Minimum direct-local party flow

A general friends system and matchmaking are not prerequisites. The minimum route can resolve an exact active profile name through the authenticated user directory already used by direct Messaging:

1. Both profiles authenticate and have distinct live user sessions.
2. Inviter names the second profile through the native ship UI.
3. `party.Invite` resolves the exact active user, creates the inviter's party if necessary, records a pending invitation with a server deadline, and emits domain events.
4. The Playgroups adapter returns the creation/full-roster structures.
5. The Messaging adapter encodes the direct invitation with attribute `720897=<partyID>` and queues it to every authenticated session of the invitee.
6. `party.Accept` validates the pending invitation and commits membership atomically.
7. The adapter sends the full JoinPlaygroup notification to the new member and MemberJoined to every existing member, followed by presence projection updates.

Do not use Rooms membership as a prerequisite. Do not persist a fake friend relationship. Offline delivery can be rejected for this milestone with an explicit result; durable inbox behavior is later scope.

### Feature-owned interfaces

The party package should consume small ports such as:

```go
type UserDirectory interface {
    ActiveUserByName(context.Context, string) (User, error)
}

type EventPublisher interface {
    PublishPartyEvents(context.Context, []Event) error
}

type Clock interface {
    Now() time.Time
}
```

Its service should expose typed `Invite`, `Respond`, `Cancel`, `Leave`, `Disband`, and `Expire` commands. The service owns authorization and atomic aggregate mutation. Blaze Playgroups and Messaging decode requests, call those operations, and encode returned events. They must not edit `sporenet.User`, a repository, or a party map directly.

For the in-memory milestone, a mutex-protected party store is sufficient. Keep store mutation and user claim projection in one operation with rollback on failure; otherwise a failed notification or claim can leave the two users disagreeing about membership.

## Party, game, slot, squad, and Hello correlation

These identifiers have different lifetimes:

| Identity | Owner | Lifetime |
| --- | --- | --- |
| User ID | authenticated user/session feature | Account/session |
| Playgroup ID | party feature | Ship party through leave/disband |
| Game ID | game admission feature | One campaign game |
| Player slot | game admission record | Stable for that user's game membership |
| Peer generation | gameplay admission/rejoin | One UDP binding; replaceable |
| Hero object ID | zone identity allocator | Stable world identity for one admitted player's squad creature |

`current_playgroup_id` and `current_game_id` must be projections of committed feature state. The existing persistent account fields may be retained as resume hints only if their recovery semantics become explicit; they must not authorize a join. The runtime `User.ClaimGame` and playgroup equivalent likewise cannot be the transaction boundary.

Game creation should consume a frozen party snapshot:

```text
GameAdmission
  GameID
  PartyID
  PartyRevision
  LeaderUserID
  HostUserID
  State = reserving | joining | ready | active | ending
  Players[UserID] = {slot, joinOrdinal, presence, selectedSquad}
  DifficultyParticipantCount
```

The leader starts campaign 1-1. The operation verifies party leadership, freezes the two accepted members and party revision, assigns stable slots `0` and `1`, records the host separately, and reserves both users. A later JoinGame is accepted only for a reserved user and is idempotent for that user. No arbitrary caller may add or remove another player by supplying `GID`/`PID`.

Build 103 passes the active playgroup into GameManager creation and uses `ExpectedPlayerCount`; on join completion it also associates a `PlaygroupKey`. Encode both as compatibility projections, but use the typed admission record as authority.

Each member selects and owns their own default PvE squad. `game.GameplayJoin` already resolves the authenticated active user, current game membership, stable slot, and that user's default squad. Extend that boundary rather than selecting one squad for the whole zone.

`HelloPlayerRequest` must correlate:

- authenticated RakNet user ID;
- committed `current_game_id`;
- the same admitted game roster and stable slot;
- optional playgroup ID, when present, equal to the game's frozen party ID;
- a newly allocated peer generation.

For a two-player party, a mismatched nonzero Hello playgroup ID is a hard rejection. An absent optional ID may remain compatible for solo. Whether build 103 ever omits it for an admitted party is a capture blocker and should be tested before making it mandatory in all co-op cases.

## Evidence-constrained join ordering

The exact retail notification that causes the accepted second member to call GameManager JoinGame has not been recovered. The safe server ordering below is therefore constrained by known client callbacks and packet dependencies, not claimed as exact retail behavior.

### Blaze phase

1. Both users have received the accepted Playgroups roster at the same party revision.
2. The leader requests campaign entry.
3. Game admission freezes both users, slots, party ID/revision, host, campaign level, and `DifficultyParticipantCount=2`.
4. GameManager creates/resets one dedicated game with `PGID`, `PlaygroupKey`, and `ExpectedPlayerCount=2`.
5. The host receives game setup plus all player entries.
6. The second client is directed through the recovered JoinGame path; it receives the same game setup, fixed host identity, and all player entries. Existing members receive the second player's entry.
7. Finalization publishes `StatePreGame`, after which each admitted member may establish its RakNet session. A captured build-103 solo tutorial performs ResetDedicatedServer, FinalizeGame, and then HelloPlayer without first sending UpdateMeshConnection or advancing to `StateInGame`; requiring the later ready state at Hello creates a circular startup dependency.
8. The complete ready barrier still owns the later transition to `StateInGame`, and state/start notifications are broadcast to every game member rather than only the request session. A pre-start timeout cancels an incomplete reservation without creating a mutable zone.

The current `gameSetupFields` uses the request user as administrator and host. Replace that with `GameAdmission.HostUserID`; otherwise sending setup to the second user incorrectly makes that user host.

### RakNet and zone phase

1. Either peer may connect first, but Hello is accepted only after its Blaze game membership is committed.
2. Server replies with Hello carrying the member's stable slot.
3. The joining peer receives existing player-slot announcements; existing peers receive `PlayerJoined` for the new slot.
4. `PartyMergeComplete` is sent only when the admitted roster is coherent for that peer. Conservative ordering is Hello, roster joins, then merge complete; capture must verify the exact retail sequence.
5. Status `2` projects the local player's full LabsPlayer state and the remote player's required baseline.
6. Status `4` sends the same level and frozen `PlayerMask` to both.
7. Both members reach the ready/status-`8` barrier. The server creates one zone from the frozen roster and subscribes both members before mutable encounter time starts.
8. Each client receives GameStart and a snapshot at one revision, then incremental events after the snapshot cursor.
9. Gameplay commands are accepted only after that member's snapshot commit.

Do not let the first peer independently prime campaign 1-1 while the second is still loading. The current `publishCampaign` path is peer-local and consumes one-time opening state, so whichever peer runs second can miss already published NPCs and objects.

## World identity blocker

`zonehero.ObjectID` in `server/zone/hero/hero.go:67` returns `1 + creatureIndex`. Current setup uses it without the player slot in `server/gameplay_udp.go:685`, `server/gameplay_udp.go:732`, and `server/gameplay_udp.go:2811`. Both users therefore claim hero objects `1`, `2`, and `3`.

Use a stable per-slot hero namespace:

```text
heroObjectID = 1 + playerSlot * squadSize + creatureIndex
```

For four game slots and three creatures, reserve object IDs `1..12` before allocating script objects, fixtures, population, companions, and projectiles. Replace every range check and conversion that assumes hero IDs are only `1..3`, including targeted-ability logic near `server/campaign_ability_special.go:1918`. The client packet contracts carry object identity independently from player slot, so this is evidence-constrained and must be validated in the two-client trace.

The allocator must be initialized once per zone. The present `firstObjectID` calculation at `server/gameplay_udp.go:3607` begins after only the local squad and can collide with a later peer.

## Initial and late-join projection

### Required snapshot

`zone.Zone.Snapshot` at `server/zone/zone.go:855` currently returns only zone identity, generation, state, and members. It is not a world snapshot. A coherent snapshot at revision `R` must include:

- game state, world time, chain level, seed, and director revision;
- roster, stable slots, presence, player mask, leader/host projections;
- every member's selected squad, deployed hero, hero object IDs, position, facing, locomotion, health, mana, modifiers, and any visible action;
- companions, including owner, position, health, and action;
- every published or dormant NPC/fixture needed by the client, with object ID, noun, position, health, target, action, and defeated/published state;
- script objects, doors, teleporters, barriers, security terminals/routes, and activation state;
- pickups, DNA, crystals, health/mana orbs, equipment loot, claim/visibility state, and ownership eligibility;
- objective text/state/tokens and current encounter, horde, boss, or opening phase;
- result/outcome state, extraction/cashout vote, and member receipt state.

Projection order must respect dependencies:

1. roster and player identities;
2. object creates;
3. component/resource baselines;
4. transforms and locomotion;
5. modifiers, targets, and active actions;
6. objectives, barriers, teleporter/security state, and encounter phase;
7. incremental events strictly after cursor `R`.

The snapshot operation must hold or version the relevant shared state so events cannot fall between snapshot creation and subscription. The required abstraction is `SnapshotAt() -> (WorldSnapshot, Cursor)` plus `Subscribe(member, cursor)`, or an equivalent atomic operation. Packet encoding belongs in the build-103 RakNet adapter.

### Initial join versus rejoin

Initial co-op admission authorizes a reserved party member and creates their member state. Rejoin authorizes an already retained member and replaces only its peer generation. Both use the same snapshot and cursor machinery.

For the milestone:

- admit both first-time members before the dungeon starts;
- reject a first-time late admission after the zone becomes active;
- allow a retained disconnected member to rejoin during the grace period with the same game ID, slot, squad, hero identities, and member resources;
- discard all queued output for the old generation before installing the replacement;
- send `ReconnectPlayer` state where required, then the coherent snapshot.

This deliberately keeps the unsolved general late-add policy outside the milestone while implementing the hard snapshot primitive once. The generation replacement and UDP identity details remain owned by [zone rejoin](../zone/rejoin.md).

## Incremental projection and idle-peer delivery

The current zone projection covers only objectives, selected NPC spawn/damage/death/action events, hero resource, and companion damage (`server/zone/zone.go:1137` through `server/zone/zone.go:1186`). Missing co-op events include:

- remote player movement and facing;
- remote player creature switch, casts, attacks, effects, and interruption;
- remote player damage, death, revive, and resource changes beyond the current narrow publication;
- companion create/move/action/death;
- pickup, loot, orb, and object lifecycle;
- barrier, security, teleporter, director, horde, boss, vote, and result transitions.

`server/campaign_movement_command.go` mutates the shared hero pose, but its movement packets return only to the invoking peer. The same peer-local response pattern occurs throughout gameplay setup and interaction paths. Every accepted command must instead produce:

1. a typed feature result for the actor; and
2. zero or more semantic zone events for other subscribers.

The adapter maps those events to per-recipient packets, redacting or specializing fields as required. Zone code must not construct RakNet bytes.

### Polling is not sufficient

`gameplayJoinRuntime.poll` drains result and projection queues, but `raknet.Server.pollConnected` is reached from client-driven connected traffic. An otherwise idle second client may eventually drain output because it sends transport traffic, but there is no bounded cadence and no guarantee suitable for movement or combat. Current polling is acceptable only as a recovery safety net for low-frequency events.

Add a peer outbox/wakeup boundary:

```go
type PeerOutbox interface {
    Enqueue(PeerKey, []Delivery)
    Wake(PeerKey)
}

type Delivery struct {
    Payload []byte
    Reliability Reliability
    OrderChannel uint8
}
```

The gameplay adapter subscribes each active generation to semantic events, encodes recipient-specific deliveries, and wakes the RakNet loop. Preserve reliability and ordering metadata per packet; the current poll path's blanket reliable-ordered response cannot safely represent locomotion traffic.

For the first human milestone, publish every accepted remote movement/action at its recovered packet reliability. Add coalescing or a server tick only after a capture establishes client cadence and back-pressure needs. Never drop state transitions such as creature switches, damage, death, object activation, or loot claims.

## Per-member and shared-zone ownership

| State | Ownership and milestone policy |
| --- | --- |
| NPC population, health, target, action, death | Shared zone; one authoritative transition, projected to both |
| Director, scripts, objectives, encounter/horde/boss phase | Shared zone |
| Script objects, barriers, teleporters, security | Shared zone |
| World pickups and loot objects | Shared lifecycle; eligibility/reward application per member |
| Player squad selection and reserve creature resources | Per member, retained across connection generation |
| Deployed hero pose/health/mana | Per-member actor stored in shared world so every participant sees it |
| Cooldowns, modifiers, cast/action state | Per member; retain on disconnect/rejoin rather than reset with UDP session |
| Companions | Shared actors with a per-member owner |
| Rewards and result receipt | Per member; committing one receipt must not consume another's |
| Death/defeat | Per member for squad death; shared defeat only when no participating member remains alive |
| Vote | Shared ledger with one vote per admitted current member |
| Disconnect | Presence change only during grace; retain slot, resources, actors, and vote eligibility policy |
| Explicit leave | Remove that member from active roster, world actors, wipe calculation, and pending vote; keep zone alive for survivors |
| Final leave | Retire only after result/cleanup or grace policy, never as a side effect of another member's departure |

Current `Zone.Leave` at `server/zone/zone.go:238` immediately removes heroes, companions, actions, and leases, and retires a final-member zone. Split it into `Disconnect`, `Rejoin`, and `Leave` operations. A transport timeout must not call explicit leave.

The current conservative vote policy in `notes/help.md`—any cashout vote resolves cashout, Continue requires all current members, and leave reevaluates quorum—can remain until server-authority evidence replaces it. Death policy, disconnect grace length, revive behavior, and absent-member vote eligibility require explicit fallback entries when implemented.

## Party-size combat, population, and reward inputs

Build-103 evidence proves calculations, not every retail server policy:

- health uses `baseHealth * (1 + partyCoefficient * playerCountHealthScale)` before difficulty;
- party damage/healing multipliers are `1.0`, `1.4`, `1.8`, and `2.2`;
- the party coefficient array is `0`, `1`, `2`, and `3`;
- `playerCountHealthScale` is a NonPlayerClass field with default `1.0`; the inspected campaign 1-1 classes use `1.0`;
- crystal/drop code contains loops and chance inputs based on cached player count.

`game.Instance` now freezes the expected roster cardinality into each gameplay binding, and `zone/difficulty.ProjectDirector` applies that count to the exact health and damage formulas before the shared zone is constructed. Disconnect and rejoin therefore cannot rescale an existing population.

Use these conservative milestone policies:

1. Freeze `DifficultyParticipantCount` from the admitted roster before zone creation. Two accepted, joined members produce count `2`; rejoin does not change it.
2. Use the exact client health and damage/healing formulas as compatibility fallbacks for server simulation.
3. Do not rescale an already spawned NPC when a member disconnects or explicitly leaves.
4. Keep authored population counts unchanged until a content or retail capture proves party-count population scaling.
5. Keep rewards per member and world drops single-instance. Do not multiply equipment or currency merely because a cached-player-count branch exists; recover ownership/eligibility first.
6. Do not allow first-time post-start joins while difficulty and reward admission policy is unresolved.

The frozen health/damage policy is recorded in `notes/help.md`. Population, reward, and death fallbacks remain unimplemented until their authority boundaries are selected.

## Current architecture conflicts

### Blaze

- `server/blaze/social_components.go` treats Playgroups commands `0x01..0x0b` as empty stubs.
- Messaging command `0x01` requires chat body attribute `ff02`; the native invitation is a direct message whose significant payload is attribute `720897`.
- UserSession notification queues can already target authenticated users and should be reused by adapters.
- Rooms are sufficiently separate and should remain out of party authority.

### GameManager and game

- Reset creates a game and adds only the requester.
- Join accepts an arbitrary game ID without party reservation, state, or actor authorization.
- A joining client receives player notification but not a complete, host-stable game setup.
- Remove and destroy trust supplied player/game IDs.
- Finalize/start notifications are request-session-local instead of member-wide.
- Advance mutates game info outside the instance's ownership boundary.
- `game.Instance` has players and stable slots but no typed party ID, frozen roster, host, admission state, presence, or disconnect policy.

Move these rules into `game.Admission` operations. Blaze GameManager becomes an adapter. A persistent mutation must cover admission record, user claims, and rollback; a rejected operation must write nothing.

### Gameplay and zone

- Hello ignores its optional playgroup ID.
- Peer sessions are keyed by UDP address; generation replacement only addresses the same endpoint.
- Party merge is declared complete immediately after one Hello.
- joined/departed packets are never produced.
- player hero IDs collide across slots.
- each peer independently drives status/setup and can consume one-time opening state.
- player movement/actions and many world changes are peer-local responses.
- the snapshot is metadata only and projection event coverage is incomplete.
- shared resources remain partly inside peer sessions, so disconnect destroys or resets state needed for rejoin.
- projection has no direct delivery wakeup and loses packet delivery semantics.

The zone should own world and durable member state. Gameplay admission should own active peer generation. RakNet should own addresses, delivery metadata, retry, and packet encoding only.

## Staged implementation plan

### Stage 1: direct ship invitation

- Add the party aggregate, mutex-protected store, clock, active-user directory port, and typed lifecycle operations.
- Implement Playgroups create/join/leave/destroy/lookup plus full roster, member, leader, and destroy notifications.
- Extend Messaging to accept and fan out the native invite envelope with attribute `720897`, without routing it through chat-body validation.
- Implement accept, reject, cancel, server expiry, duplicate/idempotency rules, full/closed checks, and deterministic leader promotion.
- Project committed party ID into active user sessions.
- Add build-103 TDF fixtures for every used request, response, error, and notification.

**Human gate:** two ship clients invite by known profile name, accept and reject, see identical rosters, observe expiry, and independently leave/disband.

### Stage 2: party-backed Blaze game admission

- Add `GameAdmission` with frozen party revision, reserved roster, fixed slots, leader, host, campaign identity, and participant count.
- Make create/reset leader-only and JoinGame reservation-only.
- Send complete, host-stable game setup and all player entries to each member.
- Broadcast join/remove/state/start notifications to all admitted sessions.
- Make `current_game_id` and `current_playgroup_id` committed projections and rollback them on operation failure.
- Capture the second-client GameManager trigger and exact notification order; implement the recovered route before claiming native ship-to-game completion.

**Automated gate:** authorization, duplicate join, full roster, storage/claim rollback, leader-only start, stable slots, and complete two-recipient Blaze fixtures.

### Stage 3: two-player gameplay admission and identities

- Correlate Hello user, game, party, slot, and generation.
- Allocate per-slot hero object IDs and reserve all player blocks before shared objects.
- Audit every hero-ID range/conversion and every actor lookup.
- Validate the implemented `PlayerJoined` and `PartyMergeComplete` order against a retail capture, then produce `PlayerDeparted`.
- Add a two-member ready barrier and construct one zone from the frozen roster.

**Automated gate:** either Hello order produces the same slots, player mask, hero IDs, shared object IDs, and one zone; stale or mismatched party/game/generation is rejected without mutation.

### Stage 4: coherent initial world

- Move member resources needed after disconnect from peer sessions into a zone member aggregate.
- Implement atomic world snapshot plus cursor.
- Project both player heroes/squads, companions, all existing NPCs, fixtures, objects, loot, objectives, teleporter/barrier/security state, and current encounter phase.
- Replace one-peer opening setup with one shared baseline and per-recipient local-control selection.

**Automated gate:** join orders `A,B` and `B,A` yield equivalent world snapshots; no duplicate IDs or missing opening NPCs; an event concurrent with snapshot appears exactly once.

### Stage 5: live multi-peer projection

- Emit semantic events for remote movement, actions, resources, death, companions, objects, loot, objectives, director, and encounter transitions.
- Add recipient-specific packet adapters and the peer outbox/wakeup boundary.
- Preserve reliability/order metadata and retain polling only as recovery.
- Add representative two-subscriber traces rather than a unit suite per ability.

**Automated gate:** one member can remain application-idle while receiving bounded-latency movement, combat, NPC, object, and objective updates.

### Stage 6: disconnect, rejoin, and independent leave

- Split disconnect presence from explicit leave.
- Retain slot, squad, resources, cooldowns, hero identities, rewards, and shared-world membership for a bounded grace interval.
- Replace peer generation and use the Stage 4 snapshot for rejoin.
- Remove only an explicit leaver; publish departed state and recompute wipe/vote participation without resetting survivors.
- Retire the zone only after its last retained/admitted member is gone under the selected cleanup policy.

**Automated gate:** stale-generation output is discarded; rejoin keeps IDs/resources; second-member leave cannot reset or retire the first member's zone.

## Two-client human test

Use two distinct local launcher profiles with valid three-creature PvE squads and campaign 1-1 access.

1. Start the loopback authentication broker and server with the normal workspace commands.
2. Launch profile 1 and profile 2 using the two profile-specific tutorial/launcher commands.
3. From the ship, profile 1 invites profile 2 by exact known name.
4. Verify the incoming prompt, acceptance, identical two-member roster, leader display, and both `current_playgroup_id` projections.
5. Have the leader select campaign 1-1. Verify both clients obtain the same game ID, slots `0/1`, host, level, and player mask before either enters active gameplay.
6. Verify both see both player heroes at distinct object IDs and the same opening NPCs, objects, objective, barriers, and teleporter state.
7. Move each player while the other stands application-idle. Verify bidirectional movement and creature-switch observation.
8. Have each player use basic and special abilities. Verify actions/effects, resource changes, the same NPC health/death, companions, pickups, loot lifecycle, and objective progress on both clients.
9. Disconnect profile 2 without leaving. Continue moving/fighting on profile 1; reconnect profile 2 within grace and verify the same slot, heroes, resources, current NPCs/objects, and encounter phase without duplicates.
10. Explicitly leave with profile 2. Verify profile 1 remains in the live zone and can move, fight, progress, vote, and exit normally.
11. Repeat with the leader leaving first and verify the selected leader/game-host policy.

Record protocol event captures under `bin/server/darkspin/logs/traces` and client/decompiler diagnostics under `bin/game/logs`. The milestone passes only from observed two-client behavior, not from two synthetic Hello requests.

## Remaining capture blockers

| Blocker | Why it matters | Suggested capture |
| --- | --- | --- |
| Exact Messaging invite/decline/cancel outer TDF and response codes | Avoid accepting the UI prompt but failing sender feedback | Ship invite accept, decline, expiry, and inviter cancellation |
| Exact Playgroups create/join defaults and optional fields | Empty/default field differences can prevent the client object from becoming active | Solo group creation followed by second-member join |
| Corrected GameManager lifecycle validation | The client tables and failure trace now define connecting setup, pre-game finalize, mesh completion, and in-game order, but the fixed exchange still needs a native pass | Capture both clients from leader campaign selection through both RakNet Hellos and compare with [the recovered lifecycle](../protocol/game-lifecycle.md) |
| Host-session identity after setup | This is the next likely incompatible field only if the receiver still requests non-host destruction after duplicate joining records are removed | Capture the receiver's setup processing, destroy reason, and host predicates |
| Hello party field and `PlayerJoined`/merge/depart order | Determines RakNet admission and visible remote slot construction | First join, second join, second disconnect, second explicit leave |
| Remote hero movement/action packet reliability and cadence | Required for direct delivery without over-ordering locomotion | One stationary observer and one moving/casting player |
| Live-world baseline packets | Required for initial/rejoin snapshot encoding | Join/rejoin during opening, live encounter, loot on ground, and active objective |
| First-time late join policy | General late admission is intentionally excluded from milestone | Retail join attempt after dungeon start |
| Leader migration and live-game ownership | Prevents leader leave from corrupting survivor | Leader leaves in ship and during live zone |
| Disconnect grace, revive, wipe, vote, loot, and reward policy | Client content does not establish server authority | Controlled disconnect/death/drop/cashout cases |
| Party-count population scaling | Current content proves combat formulas but not spawn-count authority | Compare retail 1-player and 2-player campaign 1-1 population traces |

## Definition of done

The audit's implementation is complete when:

- the native ship Invite UI completes creation, notification, accept/reject/cancel/expiry, roster, leader, and leave/disband for two authenticated local profiles;
- both users are authorized members of one GameManager game with stable distinct slots and one fixed host;
- both Hellos bind to that admission and one shared zone without object-ID collision;
- both clients receive a coherent initial world and bounded-latency remote-player/shared-world changes;
- disconnect/rejoin replaces only transport generation and restores a snapshot without gaps or duplicates;
- explicit leave removes one member without resetting, retiring, or corrupting the survivor;
- the complete campaign 1-1 human test passes and produces reviewable Blaze and RakNet traces.
