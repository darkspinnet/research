# Zone rejoin and live-state projection

## Purpose and scope

This audit turns the recovered build-103 reconnect path into an implementation
plan for attaching a newly authenticated gameplay peer to an existing campaign
zone without resetting that zone. It does not repeat the already-established
client-capability conclusion in `notes/zone/rejoin-client.md`. It separates:

- same-process rejoin to an exact retained live zone;
- gameplay-equivalent restoration from a durable checkpoint after process
  restart; and
- packet projection, which belongs to the build-103 RakNet adapter rather than
  the zone feature.

No retail capture proves the exact baseline packet order, reconnect grace
duration, or complete late-join representation. Requirements below are marked
by confidence:

- **Exact**: directly established by client receiver behavior or current code.
- **High**: required by authoritative-state and concurrency invariants, with
  current code supporting the model.
- **Medium**: conservative build-103-compatible policy without a retail
  ordering or lifecycle capture.
- **Low**: a plausible future contract that still needs client or retail
  evidence.

## Implementation verdict

The repository has most of the stateful campaign owners needed for live rejoin,
but it does not yet have a rejoin transaction or a complete zone snapshot.
Current transport cleanup removes the member's hero and companions, removes
result participation, stops an empty zone, and retires it. A new gameplay hello
is indexed by UDP address and receives a new process-local generation, so a new
address cannot atomically replace the authenticated member identified by
`(game ID, user ID)`.

The implementation should first make transport presence independent of durable
game and zone membership. Then it should add a revisioned, immutable zone
baseline, an atomic generation reservation/commit, and a build-103 projection
adapter. Process-restart recovery is a later feature and should not block the
same-process path.

## Evidence map

| Contract or current behavior | Evidence | Confidence |
| --- | --- | --- |
| Account discovery exposes the active game ID | `server/game/api.go:670`; `server/sporenet/user.go:182-210` | Exact for current server |
| Gameplay authorization follows `user.CurrentGameID()` and the retained slot | `server/game/gameplay_session.go:116-172` | Exact for current server |
| Game membership and slot are keyed by user ID; adding the same member is idempotent | `server/game/manager.go:85-151`, `server/game/manager.go:188-219` | Exact for current server |
| Blaze `JoinGame` finds an existing game, calls `AddPlayer`, and projects roster notifications | `server/blaze/game_manager_component.go:163-201` | Exact for current server; retail notification policy not proven |
| Explicit Blaze remove clears the game member and therefore `current_game_id` | `server/blaze/game_manager_component.go:203-238`; `server/game/manager.go:214-230` | Exact for current server |
| Destroying the game clears all members | `server/blaze/game_manager_component.go:143-161`; `server/game/manager.go:613-623` | Exact for current server |
| Gameplay sessions are keyed by UDP address and receive a new process-local generation | `server/gameplay_udp.go:891-952`; `server/gameplay_session.go:633-660` | Exact for current server |
| Address replacement/cancellation stops the peer session | `server/gameplay_udp.go:352-397`; `server/gameplay_session.go:676-691` | Exact for current server |
| Stopping the peer calls `Zone.Leave` and cancels connection-local presentation runs | `server/gameplay_udp.go:1800-1908` | Exact for current server |
| `Zone.Join` rejects an older generation and replaces several old-generation bindings | `server/zone/zone.go:162-236` | Exact for current server, incomplete replacement boundary |
| `Zone.Leave` removes hero, companions, outcome/result participation and retires an empty zone | `server/zone/zone.go:238-310`; `server/zone/registry.go:62-76` | Exact for current server; unsuitable for transport loss |
| The current zone snapshot contains only ID, generation, lifecycle state, and members | `server/zone/zone.go:85-95`, `server/zone/zone.go:855-872` | Exact for current server |
| The projection session sequences only selected live events and keeps queues per connected generation | `server/zone/projection/session.go:23-72`, `server/zone/projection/session.go:76-252` | Exact for current server; not a journal |
| Fresh campaign initialization resolves or creates the zone only during dungeon setup | `server/gameplay_udp.go:3574-3738` | Exact for current server |
| Per-peer dungeon setup has reserve/commit/rollback guards | `server/zone/member/lifecycle.go:50-105`; `server/gameplay_udp.go:3318-3422` | Exact for current server; address-scoped |
| Build 103 accepts `ReconnectPlayer` as a four-byte game-state selection and state 6 is `GameDungeon` | `notes/client/packet.md:205`, `notes/client/packet.md:292`; `server/raknet/application.go:137-143`; `server/raknet/types.go:89-126` | Exact receiver contract |
| Exact ordering, reliability channel, and retail contents after `ReconnectPlayer` are not recovered | `notes/zone/rejoin-client.md:121-124`, `notes/zone/rejoin-client.md:374-385` | Exact statement of evidence gap |
| Darkspinner currently exposes Play, waits for process exit, clears its launch credential, and returns to “Game closed” | `app/darkspinner/app.go:350-465`, `app/darkspinner/app.go:608-620` | Exact for current launcher |
| Runtime user storage has `current_game_id`, but no durable zone table | `server/sporenet/sqlite/schema.go:8-197`; `server/sporenet/sqlite/rows.go:12-31` | Exact for current storage |

## Identity and membership correlation

### Stable identities

Use the following identities for distinct purposes:

| Identity | Owner | Lifetime | Rule |
| --- | --- | --- | --- |
| Account/user ID | account feature | durable | Authenticated principal; never inferred from UDP address or client-supplied game ID. |
| Game ID | game feature | game lifetime | Address of Blaze game membership and the zone aggregate. |
| `(game ID, user ID)` | game/zone membership | until explicit leave, completion cleanup, or expiry | Stable rejoin key. |
| Player slot | game membership | same membership lifetime | Retained across disconnect and generation replacement; not reallocated on rejoin. |
| Zone generation | zone registry | zone incarnation | Distinguishes a retired/restored zone incarnation from an old reference for the same game ID. |
| Peer generation | rejoin/member feature | connection attempt | Monotonic for one `(game ID, user ID)`; transport address and RakNet peer are attributes of it. |
| UDP address/RakNet transport peer | RakNet adapter | connection | Routing only; never authoritative membership identity. |

`current_game_id` is a discovery pointer, not proof of admission. The account
response may expose it only if the game membership service says that
`(current_game_id, user ID)` is resumable. Blaze `JoinGame` must validate the
same pair. Gameplay hello must authenticate the user and resolve the same game
and slot again. None of those stages may create a new zone or slot when the
request claims to be a rejoin.

### Required correlation flow

1. Authentication resolves an account ID from the active login credential.
2. `ResumeStatus(user ID)` checks `current_game_id`, retained membership, game
   lifecycle, zone incarnation, grace deadline, and any pending terminal result.
3. The account projection includes `current_game_id` only for a resumable
   result. A stale durable value is cleared through the membership operation,
   not opportunistically in the XML adapter.
4. Blaze `JoinGame(game ID)` authorizes the authenticated user against the
   retained membership. A same-member call is idempotent and retains its slot.
5. Gameplay hello supplies authenticated user identity through the established
   login/gameplay contract. The server resolves the game; it does not trust a
   socket address or an arbitrary client game ID.
6. Gameplay activation reserves a new peer generation for the same
   `(game ID, user ID, slot, zone generation)`.
7. The RakNet adapter binds the new address to the reserved generation,
   projects the baseline and catch-up, and commits that generation active.

If the active in-memory game is absent after process restart, a nonzero
persisted `current_game_id` is not sufficient. It is resumable only when a
compatible durable zone checkpoint and membership record can be restored.

## Lifecycle distinctions

The feature must receive an explicit cause instead of treating every connection
closure as `Leave`.

| Cause | Game membership | Zone member/hero | World progression | `current_game_id` | Result transaction |
| --- | --- | --- | --- | --- | --- |
| Transport loss or client crash | Retain through grace | Mark disconnected; retain semantic hero/squad state | Continues if another connected member remains; otherwise policy may pause deadlines or continue selected instance timers | Retain | Retain |
| Client-only soft lock while transport remains healthy | Retain | Retain until replacement commits | Continues | Retain | Retain |
| Explicit Return to Ship / Abandon / mission leave | Remove through authorized leave operation | Remove or convert according to co-op leave policy | Shared zone continues for remaining members; retire when terminal/empty | Clear atomically with membership removal | Resolve/cancel according to explicit action and reward phase |
| Account logout | Does not by itself prove mission leave | Mark disconnected and start grace unless preceded by explicit leave | Same as transport loss | Retain while resumable | Retain |
| Defeat | Retain through Game Over/result flow | Preserve terminal squad/death state | Zone enters terminal defeat state; no ordinary combat commands | Retain until terminal route explicitly leaves or expires | Preserve terminal choice/receipt state |
| Victory | Retain through result voting/cash-out/continue | Preserve completion state | No new combat progression except defined result/transition operations | Retain until cash-out/return, abandon, or successful chain handoff | Preserve and make idempotent |
| Process restart without durable zone restore | Membership becomes non-resumable after reconciliation | No live zone | None | Clear stale pointer | Only independently durable committed rewards survive |
| Process restart with compatible checkpoint | Restore membership and gameplay-equivalent zone | Restore semantic member state | Resume from checkpoint under a new zone incarnation | Retain | Restore pending/committed transaction exactly |
| Grace expiry | Remove disconnected membership | Remove member-owned semantic actors; retarget AI | Continue for connected co-op members; retire if empty under zone policy | Clear | Preserve already committed receipts; cancel or settle pending offer by explicit policy |

Recommended conservative policy: transport loss never invokes explicit leave.
A disconnected member remains a voting member only where doing so cannot
deadlock the party indefinitely. Result voting needs a separate disconnect
deadline: preserve an already-cast vote, but after the deadline choose the
conservative cash-out/return result rather than silently treating absence as
Continue.

The grace duration is server policy until retail evidence exists. Make it
configurable and record the expiry deadline in the membership state. A useful
first implementation is 10 minutes for an in-progress campaign and a shorter
explicit result-vote deadline. The existing two-minute RakNet peer retention is
a transport implementation detail, not the mission grace policy.

## Atomic old-peer to new-peer replacement

### Member connection state

Each retained zone member should own:

```text
MemberConnection {
    game_id
    user_id
    slot
    zone_generation
    active_peer_generation
    reserved_peer_generation
    presence: disconnected | reserving | projecting | active | leaving
    disconnected_at
    grace_deadline
    snapshot_revision
}
```

The peer generation must be allocated by the membership/rejoin feature, not by
the address-indexed adapter. A generation is valid only for one game, user,
slot, and zone incarnation.

### Reserve, project, commit

1. Lock the member connection record and validate the actor, game, slot, zone
   generation, lifecycle, and grace deadline.
2. If the same generation and same activation token is already active, return
   the prior result idempotently. If it is projecting, return the existing
   reservation. Reject every lower generation and every token whose game,
   user, slot, or zone generation differs.
3. Allocate/reserve `newGeneration > activeGeneration`. Set member presence to
   `projecting`. From this point, reject new gameplay commands from the old
   generation, but retain its transport long enough to send closure if useful.
4. Cancel old-generation connection-owned command leases and presentation
   runs. Do not remove semantic hero, squad, companions, objective state, loot,
   results, or instance timers.
5. Under the zone consistency boundary, capture baseline revision `R` and an
   immutable snapshot. Register the new subscriber starting after `R`.
6. Project `ReconnectPlayer(GameDungeon)`, the baseline, and journal events
   `R+1..C` to the new peer. Ordinary commands from the new generation remain
   closed.
7. Recheck the reservation token and zone generation. Atomically set the new
   generation active and open its command gate. Detach the old transport
   routing entry.
8. If encoding or transmission setup fails before commit, roll back the
   reservation to disconnected/old-inactive state. The old generation stays
   command-rejected once replacement began; a retry obtains or reuses a newer
   reservation. Never reactivate two controllers.

This boundary requires one feature-owned operation. Calling `Zone.Join`,
updating an address map, and later committing `member.Setup` independently is
not atomic enough. The operation should coordinate member state and zone
subscriber registration while leaving byte encoding and delivery in the
adapter.

### Duplicate and stale rejection

- Every C2S command lookup resolves address -> authenticated peer handle ->
  `(game ID, user ID, peer generation, zone generation)`.
- The command gate validates that tuple against the active member record before
  invoking any zone mutation.
- Delayed callbacks carry the same tuple plus their operation epoch. They no-op
  or return stale when any component differs.
- A repeated hello from the same address is not proof of identity continuity.
- A newer RakNet address generation is not automatically a newer gameplay
  generation.
- Projection acknowledgements and retries include a reservation/activation
  token so a late success cannot commit a superseded attempt.

## Baseline and revision ordering

### Evidence-constrained wire order

Only the four-byte meaning of `ReconnectPlayer` is exact. The following is the
minimum safe dependency order for a client retaining its loaded dungeon, not a
claim about retail packet ordering:

1. Complete authenticated gameplay identity and retain the existing Blaze
   player slot.
2. Reserve the new peer generation and close both old- and new-generation
   ordinary command gates.
3. Capture snapshot `S` at revision `R` and begin buffering events after `R`.
4. Send game clock/mode, roster, slot, squad, and controlled-player baseline.
5. Create static and dynamic objects in dependency order:
   authored script/interactable/barrier objects, heroes and companions, live
   NPCs, then loot/pickup/effect objects.
6. Publish transforms, resources, death/visibility/collision state, modifiers,
   cooldowns, targets/actions, objectives, director/encounter state, and result
   state only after referenced objects exist.
7. Send `ReconnectPlayer(GameDungeon)` after the client has the complete world
   required by its immediate state transition.
8. Read current revision `C`, append journal events `R+1..C` exactly once, and
   repeat until the subscriber reaches a stable head or switch it atomically to
   the live outbox.
9. Commit the new generation active and admit commands.

The `16:17` launcher-restart trace proved that `0x81` cannot precede the baseline: the
client entered state 6 immediately and crashed in native world lookup before
the next ordered `0x8a` packet arrived. If a retail trace later refines which
subset must precede `0x81`, change steps 4-7 only in `zone/raknet103`; the
feature transaction remains unchanged.

That trace was a full client restart, not merely transport replacement. A
restarted process has no retained dungeon and must use `GamePrepareForStart`,
the normal ready-status handshake, and `GameStart`; it must not receive
`ReconnectPlayer(GameDungeon)` as its loading shortcut.

`0xffffffff` is the recovered client failure value. The feature should return a
typed rejection; only the adapter chooses whether that rejection maps to
`0xffffffff`, a spaceship transition, connection close, or no packet.

### Revision/journal contract

The current projection session increments a sequence but stores only
per-subscriber queues and deletes the prior generation on join. It cannot
provide a baseline cursor or recover events that occurred before the new
subscriber was registered.

Replace or extend it with a zone-owned journal:

```text
ZoneRevision {
    zone_generation
    revision
}

JournalEvent {
    revision
    semantic_kind
    idempotency_key
    semantic_payload
}
```

All mutations that affect the rejoin-visible semantic state must advance the
same monotonically increasing revision under the zone transaction boundary.
The baseline reports one revision. The journal retains a bounded suffix until
all active/projecting subscribers pass it and until any durable checkpoint
references an equal or newer revision.

Snapshot-plus-catch-up options:

- Preferred: briefly hold the zone write lock while cloning semantic state and
  registering the cursor at `R`; release it before encoding.
- Acceptable: copy-on-write immutable aggregates with a single revision fence.
- Unsafe: call each subsystem's `Snapshot` independently without a shared
  revision, because the result can combine pre- and post-event state.

Journal events are semantic facts, not encoded packets. Packet reliability,
ordering channel, timestamps, object reflection masks, and build-specific
presentation belong to the RakNet adapter.

## Complete live-zone snapshot inventory

The snapshot is a coherent immutable aggregate. Existing subsystem snapshot
methods are useful sources but do not establish coherence by themselves.

| Subsystem | Required semantic baseline | Same-process disposition | Restart/checkpoint disposition |
| --- | --- | --- | --- |
| Game identity/lifecycle | game ID, mode, level/content identity, difficulty, zone generation, zone phase, simulation time, revision | Snapshot | Persist |
| Membership/roster | user IDs, retained slots, presence, accepted generation, disconnect/grace deadlines, squad IDs, result-voter membership | Snapshot | Persist |
| Squad | three creature identities, active creature, per-creature health/power/maxima, terminal/wipe state, selection state, unlock count | Snapshot; move remaining authoritative fields out of `gameplayPeerSession` | Persist resources and selection |
| Hero | object ID, creature index, transform/orientation, collision/visibility, movement endpoint/state, health, power, maxima, death state, modifiers | Snapshot | Persist checkpoint transform or safe milestone anchor, resources, death state |
| Other players | the same visible controlled-object state for every member | Snapshot | Persist member semantic state |
| Companions/summons | identity/noun, owner, object ID, transform, health, targetability, target/pursuit, cooldown deadline, remaining lifetime | Snapshot | Persist durable companion identity if gameplay-relevant; otherwise cancel and reconstruct from an explicit owned effect |
| NPCs/fixtures | spawn identity/locus, object ID, noun/profile, origin/current transform, health, faction, defeated/published/shield/security flags | Snapshot from `zonenpc.Session` | Compact model persists defeated/consumed loci and reconstructs survivors at authored clusters |
| NPC AI | target and target owner, action owner, pursuit/action state, action epoch, cooldown/deadline, threat/aggro state | Snapshot or deterministically cancel to idle/retarget before baseline | Cancel in-flight action; reacquire targets deterministically |
| Static/script objects | authored identity, allocated object ID, transform, collision/visibility, uses/consumed state, one-shot script keys | Snapshot | Reconstruct from content plus persisted consumed/fired keys |
| Pickups/orbs/crystals | object/drop identity, position, payload, owner/eligibility, collected/consumed/reserved state, member crystal inventory | Snapshot | Persist unclaimed persistent drops when needed plus consumed identities and member crystal inventory |
| Loot/DNA/rewards | drop identity, generated item/DNA amount, ownership/eligibility, claim state, RNG consumption/idempotency key | Snapshot | Persist generated/claimed reward facts; never reroll from a milestone |
| Objectives | objective definitions/tokens, per-slot current/target counts, completion, presentation revision, Lua/objective one-shot facts | Snapshot | Persist tokens, counts, completion and fired semantic keys; reconstruct runtime |
| Encounters/population | authored seed, selected population plan, admitted loci, defeated loci, current cluster/route milestone, population policy history | Snapshot | Persist seed, consumed/defeated/admitted locus keys and milestones; reconstruct living cluster population |
| Horde/boss/director | phase/epoch, marker set, wave, completion latches, boss object/health/phase, accepted and pending named publications, one-shot listener keys | Snapshot | Persist phase/milestones and fired keys; reconstruct safe phase boundary |
| Barriers/gates | authored key, object IDs, open/closed/collision/visibility state, encounter ownership | Snapshot | Reconstruct from milestone and explicit overrides |
| Teleporters/security | teleporter identity and enabled/consumed state, security gate phase, accepted transfer milestone | Snapshot semantic state; cancel per-peer visual transfer | Persist committed milestone; never persist halfway presentation sequence |
| Cooldowns/modifiers | semantic ability/effect key, source/target, stacks, magnitude, start/end deadlines, remaining duration, gameplay stats | Snapshot | Persist gameplay-affecting modifiers/deadlines; cancel cosmetic-only effects |
| Scheduled effects | semantic task key, owner kind, operation epoch, due deadline, arguments | Snapshot instance-owned work; cancel old-peer work | Persist only reconstructible semantic deadlines |
| Object IDs | next world object ID and next projectile/effect ID cursor | Snapshot | Persist exact cursors even in compact model to avoid ID reuse |
| RNG | full state or original seed plus exact draw count for every independent stream | Snapshot full stream state | Persist full state for exact restore; seed + draw count is sufficient only if algorithm/version and draw order are frozen |
| Outcome/result | victory/defeat phase, per-member reservation, result ID, vote epoch/choices/decision/ack, continue reservation, cash-out commit/receipt | Snapshot | Persist transactionally; committed reward receipt is never reconstructed |
| Event delivery | current revision, journal suffix, per-projecting subscriber cursor | Snapshot | Persist checkpoint revision and bounded post-checkpoint journal |

The current `gameplayPeerSession` still owns authoritative-looking squad,
cooldown, movement, deployed hero, and several run fields. Before rejoin, each
field must be classified:

- move semantic gameplay state into `zone` or a feature-owned child;
- retain only adapter outbox/presentation state on the peer; or
- make it derived from a zone snapshot.

## Snapshot, reconstruct, or cancel

### Snapshot exactly

Snapshot state whose current value affects authoritative validation or future
outcomes:

- memberships, slots, active creature and resources;
- transforms used for collision/targeting;
- live object identities and ID cursors;
- NPC health, defeat, shields, targets, action ownership and authoritative
  action deadlines;
- loot generation/claim facts and member crystal state;
- objective, encounter, boss, director and one-shot state;
- gameplay modifiers and cooldown deadlines;
- RNG streams;
- outcome, vote and reward transactions; and
- zone revision and journal cursor.

### Reconstruct from durable zone state or authored content

Do not duplicate immutable content in every snapshot. Reconstruct:

- noun/profile definitions, authored marker transforms, script object
  definitions, navigation mesh, and level metadata from a content-version
  reference;
- build-103 packet masks, effect assets, and reflection bodies in the adapter;
- visible barriers, objectives, and encounters from semantic phase plus
  authored definitions;
- NPC presentation objects from live semantic NPC records; and
- after a compact restart checkpoint, surviving NPCs from the recorded spawn
  seed and authored cluster plus defeated/consumed locus facts.

### Cancel instead of replay

Presentation runs and in-flight actions without a stable semantic continuation
must not be replayed from their first packet:

- beam-in/out, teleport animation, camera/cinematic, screen fade, impact flash,
  floating text, and sound-only runs;
- projectile flight already past its authoritative collision;
- partial melee/ability animation sequences;
- old-peer unlock, obelisk, death, modifier, and teleporter packet producers;
- transport retransmissions and address-scoped scheduled packets; and
- client-local UI panels or input state.

For same-process rejoin, reduce an in-flight semantic action to a safe baseline:
publish final authoritative transform/resources/modifiers, cancel its old
presentation lease, and either resume from an explicit remaining deadline or
return it to idle. For restart restore, cancel all non-checkpointed action
stacks and reacquire targets after the world is rebuilt.

## Co-op behavior

A disconnected member remains a zone member during grace. Other connected
members continue advancing the same zone:

- enemies, director, objectives, loot and timers remain shared;
- the disconnected hero becomes command-inactive;
- choose and encode one explicit AI policy: preferably retain the hero at its
  last position as an untargetable/inert actor, or apply an authored-safe
  retreat. Do not let a stale peer control it;
- NPC actions targeting that hero are cancelled and targets are reacquired
  among connected live heroes/companions;
- member-owned companions either become inert with the owner or follow a
  documented zone policy; they are not silently deleted if they encode durable
  gameplay state;
- loot eligibility remains attached to the retained member, with expiry driven
  by loot policy rather than socket loss;
- rejoin snapshots the world at its new current revision, not the state at the
  moment of disconnect; and
- grace expiry removes only that member. It must not retire or reset the zone
  while another member remains.

Result voting needs bounded absence handling. Continue must not be selected
without the disconnected member's affirmative retained vote. A cash-out vote
may conservatively resolve the group according to the current vote rule.
Membership expiry must not accidentally turn an incomplete all-member Continue
vote into Continue.

## Soft lock, transport stall, and debug commands

### What rejoin can recover

Rejoin can recover:

- a transport stall, lost UDP state, expired RakNet peer, changed client
  address, or crashed/restarted client, provided authoritative membership and
  zone state remain;
- a client-only soft lock if closing and relaunching the client clears the
  client-local state and the server can project a complete baseline.

Rejoin cannot guarantee recovery from a client-only soft lock caused by missing
server semantic state or by a baseline packet the adapter cannot encode. It
also cannot recreate unknown client-local camera, animation, UI modal, Lua VM,
or presentation stack. The promise is authoritative gameplay continuity, not
exact local visual continuation.

### `/reset`

The current `/reset` is a developer gameplay event and may restart selected
server-side combat/presentation behavior. It must not be described as rejoin.
It can legitimately:

- cancel server-owned actions with explicit reset semantics;
- restore a server-authoritative hero from a defined checkpoint if the command
  contract says so;
- republish a current semantic baseline; and
- reacquire AI targets.

It cannot claim to clear an arbitrary client UI/input lock, invent the client's
former animation state, reset only the transport while retaining the same
generation, or recreate state absent from the zone.

### Future `/rejoin`

A debug `/rejoin` can request the normal rejoin transaction for the
authenticated user. It may close/invalidate the current generation, require a
fresh gameplay connection, and cause the adapter to project the authoritative
snapshot. It must use the same admission and generation rules as a crash
rejoin. If invoked from Blaze chat, it should return a typed “relaunch needed”
status; it must not fabricate a second peer or inject packets into an
unidentified client.

## Darkspinner Continue flow

Darkspinner should query authoritative resumability after identity selection,
after authorization, after game process exit, and when the server emits a
status change. Do not infer resumability from exit code, launcher memory, or a
nonzero database column read directly by the UI.

Feature/API shape:

```text
ResumeStatusRequest {
    authenticated_user_id
}

ResumeStatus {
    is_resumable
    game_id
    mode
    level_display
    lifecycle
    member_presence
    grace_deadline
    restore_kind: live | checkpoint
    denial_reason
}
```

The HTTP/desktop adapter may expose only display-safe fields. Darkspinner
should present:

- **Continue** when `is_resumable` is true;
- **Play** when no resumable membership exists; and
- **Abandon** as a separate confirmed action when a resume exists.

Continue uses the ordinary native launch path with a fresh short-lived
credential. No game asset or Fang gameplay patch is required. Authentication
returns the authoritative `current_game_id`; the client then takes its native
login/reconnect path and sends Blaze `JoinGame`. Darkspinner should not pass a
fabricated game ID on the command line or alter packaged UI state.

After a client crash or ordinary window close, Darkspinner waits for the
process, refreshes `ResumeStatus`, retains the selected identity, and changes
the primary action from Play to Continue when appropriate. A nonzero process
exit may be displayed as a crash while still offering Continue. Closing
Darkspinner itself currently stops the game and services; if restart recovery
is not implemented, warn that closing the all-in-one server removes the live
resume opportunity.

## Explicit Return to Ship and Abandon

Closing the client is transport loss. Return to Ship/Abandon is an authenticated
game operation with an explicit cause and idempotency key.

The operation must:

1. validate actor, game ID, membership generation, zone phase, and result phase;
2. reserve the leave so duplicate requests return the same result;
3. settle or reject pending reward/result work according to the action;
4. remove the member from zone participation, release targets/actions, and
   publish roster/member departure to connected peers;
5. remove the game member and clear `current_game_id` in the same logical
   transaction;
6. commit the leave result before projecting the spaceship transition; and
7. retire the zone only when no retained members remain and no terminal
   transaction requires it.

If persistence and in-memory mutation cannot share a database transaction, use
an operation record/outbox and idempotent reconciliation. Never clear
`current_game_id` first and leave a live member behind, or remove the zone first
and leave a resumable account pointer.

In co-op, one member's Abandon removes only that member. Remaining members keep
the current zone, enemies, objectives, loot, and result state. If the leaving
member was required for a vote, recompute under explicit leave policy without
turning absence into an affirmative Continue. Grace expiry follows the same
cleanup path with cause `expired`, but does not fabricate a client transition.

The existing Blaze remove handler is the nearest current explicit-leave path,
but it mutates `game.Instance` directly and does not coordinate zone cleanup or
result policy. It should become a transport adapter over the feature operation.

## Process-restart persistence

### Full versioned snapshot

A transport-neutral zone snapshot can be persisted in `darkspin.db`, but the
current repository cannot restore one. Add feature-owned persistence ports and
a SQLite adapter rather than serializing `Zone`, a storage row, or RakNet
packets directly.

Required envelope:

```text
ZoneCheckpoint {
    schema_version
    semantic_version
    game_id
    zone_generation
    content_manifest_id
    level_id
    mode
    difficulty
    lifecycle
    saved_at
    simulation_time
    event_revision
    snapshot_payload
    payload_checksum
}
```

Full exact restoration additionally requires:

- all semantic state listed in the snapshot inventory;
- full state/index/draw count for every RNG stream;
- world, projectile, modifier, publication, result, and event ID cursors;
- objective state and reconstructible Lua execution state;
- all authoritative deadlines expressed against simulation time or persisted
  absolute time with a declared offline-time policy;
- pending named director/objective transactions;
- member presence, slot, resources, grace and result state;
- active result IDs, reservations, commit receipts and idempotency keys;
- journal suffix after the snapshot revision; and
- content/version compatibility validation before restore.

Persisting arbitrary Lua VM stacks is brittle and should not be the first
contract. Existing semantic Lua/objective operations should be checkpointed at
named boundaries. A snapshot taken while a non-checkpointable callback is
executing is either delayed until its transaction commits or records the
transaction's semantic prepare/commit state.

### Migration/versioning

- `schema_version` governs storage envelope migrations.
- `semantic_version` governs interpretation of gameplay state.
- `content_manifest_id` prevents restoration against changed authored content.
- Readers migrate only explicitly supported older versions.
- Unknown newer versions and incompatible content are non-resumable; reconcile
  membership and clear stale `current_game_id`.
- A restore creates a new zone incarnation/generation while preserving semantic
  game identity and event revision. Old peer handles cannot address it.
- Checkpoint and journal writes use a transaction. The checkpoint references
  the exact highest included event revision.
- Keep at least the newest valid checkpoint and optionally one previous
  checkpoint for corruption recovery.

## Compact gameplay-equivalent checkpoint

### Proposed model

The compact model persists:

- authored spawn/population seed and algorithm/content version;
- defeated, consumed, admitted, or permanently suppressed locus identities;
- fired one-shot Lua callback keys and accepted named-event/publication keys;
- objective tokens, counts, completion and presentation revision;
- route, security, horde, encounter, boss and chain milestones;
- generated and collected persistent rewards with stable result/drop IDs;
- member slots, squad/active creature, health, power, death/wipe state,
  crystals, cooldowns that matter at the checkpoint, and a safe transform or
  milestone anchor;
- object-ID, projectile-ID, result-ID/publication cursors;
- RNG stream seed plus state/draw cursor where future loot/combat depends on
  exact continuity;
- active result/vote/continue/cash-out transactions; and
- checkpoint event revision and a bounded semantic journal after it.

On restore, load authored content and deterministic population plans, omit
defeated/consumed loci, reconstruct surviving NPCs at their authored cluster
positions with either recorded health classes or full health according to the
checkpoint contract, rebuild barriers/objectives from milestones, allocate the
recorded IDs, and place members at a safe recorded checkpoint anchor.

This is gameplay-equivalent restoration, not exact visual continuation:

- exact continuation preserves current transforms, partial health, target
  choices, animation/action phase, projectile positions, and subsecond
  deadlines;
- compact restoration preserves cleared content, objectives, rewards,
  resources and forward progress, but may reset surviving NPC transforms,
  targets, in-flight attacks and cosmetic effects within the current authored
  cluster.

### Duplication hazards

Milestones must carry stable idempotency keys whenever reconstruction could
repeat a side effect:

- locus admission and death reward;
- horde wave start/completion;
- boss spawn, phase, death and completion;
- objective increment/completion;
- script callback and named director event;
- loot/DNA/crystal generation and claim;
- teleporter/security completion;
- victory/defeat publication;
- result offer creation, Continue reservation and cash-out commit; and
- chain progression/account reward writes.

Never infer “reward not granted” only from a missing world pickup. Persist both
the generated reward fact and its claim/commit fact. Never rerun a one-shot Lua
callback merely because its presentation object is reconstructed.

### Estimated size and cadence

These are design estimates, not measured repository values:

- envelope, membership, objectives, milestones, cursors and RNG: roughly
  8-32 KiB per zone;
- defeated/consumed/fired-key sets and generated reward facts: commonly
  4-32 KiB, with a conservative campaign upper bound near 128 KiB;
- bounded semantic journal: target 64-512 KiB compressed, capped by revision
  count and byte size;
- practical compact total: usually 32-256 KiB, with a 1 MiB hard guard before
  forcing a new checkpoint or marking the zone non-persistable.

Write on semantic milestones, not every tick:

- member join/leave/disconnect and grace changes;
- objective/encounter/horde/boss/route transitions;
- persistent drop generation or claim;
- hero death/revive and checkpoint arrival;
- result/vote/reward transaction changes;
- periodic safety checkpoint every 30-60 seconds while meaningful state is
  dirty; and
- clean server shutdown.

Coalesce high-frequency transform/resource changes and flush their latest
semantic values at the periodic checkpoint. A checkpoint plus bounded journal
is safer than tick writes: it limits write amplification, preserves atomic
milestones, and makes duplicate detection explicit. It is also safer than a
checkpoint alone because events committed during snapshot serialization remain
recoverable.

## Feature-owned interfaces and typed operations

Suggested package boundary: add rejoin/resume operations to the `zone` feature
or a cohesive `zone/member` child that consumes zone-owned ports. Do not place
the operation in Blaze or RakNet.

```go
type MembershipStore interface {
    ResumeStatus(context.Context, uint64) (ResumeStatus, error)
    ReservePeer(context.Context, ReservePeerRequest) (PeerReservation, error)
    CommitPeer(context.Context, CommitPeerRequest) (PeerActivation, error)
    RollbackPeer(context.Context, RollbackPeerRequest) error
    Leave(context.Context, LeaveRequest) (LeaveResult, error)
}

type ZoneFinder interface {
    ActiveZone(uint64) (*Zone, bool)
}

type SnapshotSource interface {
    CaptureRejoinSnapshot(CaptureSnapshotRequest) (RejoinSnapshot, error)
    EventsAfter(ZoneRevision, uint32) (EventBatch, error)
}

type CheckpointStore interface {
    Save(context.Context, ZoneCheckpoint) error
    Load(context.Context, uint64) (ZoneCheckpoint, error)
    Delete(context.Context, uint64, uint64) error
}
```

The concrete types should contain domain nouns, not wire fields:

```go
type ReservePeerRequest struct {
    GameID uint64
    UserID uint64
    ExpectedSlot uint16
    TransportAttemptID uint64
}

type PeerReservation struct {
    Token uint64
    GameID uint64
    UserID uint64
    Slot uint16
    ZoneGeneration uint64
    OldPeerGeneration uint64
    NewPeerGeneration uint64
}

type CaptureSnapshotRequest struct {
    ReservationToken uint64
    GameID uint64
    UserID uint64
    PeerGeneration uint64
}

type RejoinSnapshot struct {
    Version uint32
    Revision ZoneRevision
    Game GameSnapshot
    Members []MemberSnapshot
    World WorldSnapshot
    Result ResultSnapshot
}

type ActivatePeerRequest struct {
    ReservationToken uint64
    ProjectedRevision ZoneRevision
}

type ActivatePeerResult struct {
    PeerGeneration uint64
    LiveRevision ZoneRevision
    IsReplay bool
}
```

The build-103 adapter owns:

- mapping typed destination to `raknet.StateMessage`;
- semantic snapshot-to-packet ordering and allowlisting;
- object reflection masks and packet timestamps;
- RakNet reliability/ordering/channel;
- address routing and transport close; and
- mapping typed denial to `0xffffffff` or another safe client route.

The zone feature owns:

- semantic state and its revision;
- membership/presence/generation rules;
- coherent capture and journal;
- mutation authorization;
- instance-owned timers and RNG;
- result transaction continuity; and
- checkpoint semantics.

The game feature owns game identity, slot membership, and the externally visible
`current_game_id` relationship. One application operation must coordinate game
membership and zone membership for leave/expiry; protocol adapters must not
mutate either aggregate directly.

## Ownership changes required in current code

1. Replace address-keyed authoritative lookup with two indexes:
   transport address -> peer handle in RakNet, and
   `(game ID, user ID)` -> member connection in the feature.
2. Split `stopGameplayPeerSession` into:
   `DisconnectTransport` for peer/presentation cleanup and
   `LeaveZone` for explicit/expired membership cleanup.
3. Stop removing hero, companions, result ledger and vote membership during
   ordinary transport cancellation.
4. Move squad, active hero, movement, cooldown, semantic modifiers, and
   gameplay-affecting scheduled state out of `gameplayPeerSession`.
5. Retain adapter-only runs and packet schedulers on the peer and make every
   callback generation/token guarded.
6. Expand zone snapshot from its current four fields into an immutable
   aggregate captured under one revision.
7. Change projection from connected-subscriber queues into a revisioned journal
   plus live subscriber cursors. Per-peer outboxes may remain adapter state.
8. Make `Zone.Join` a lower-level mutation used by the rejoin transaction, not
   the externally visible replacement operation.
9. Adapt Blaze `JoinGame`, Blaze remove/destroy, account XML, gameplay hello and
   Darkspinner status to feature operations.
10. Add checkpoint storage only after same-process rejoin invariants pass.

## Staged implementation sequence

### Stage 1: lifecycle and identity

- Introduce typed resumability and leave causes.
- Add disconnected presence and grace deadlines without changing packet flow.
- Separate transport disconnect from `Zone.Leave`.
- Reconcile `current_game_id` against live membership during account discovery.
- Add invariants for retained slot and co-op advancement.

Exit criterion: closing a gameplay transport retains the zone/member and a
second member can continue; explicit Blaze leave removes only the actor.

### Stage 2: atomic peer replacement and command gate

- Move peer generation ownership to `(game ID, user ID)`.
- Add reserve/rollback/commit token flow.
- Index authenticated peer handles independently of UDP address.
- Guard all commands and delayed callbacks by zone and peer generation.
- Cancel old-peer presentation without deleting semantic actors.

Exit criterion: a newer authenticated generation replaces once; stale and
duplicate generations cannot mutate the zone.

### Stage 3: coherent live snapshot

- Define versioned semantic snapshot DTOs in feature packages.
- Move remaining authoritative peer state into zone-owned children.
- Add a single revision fence across mutation, capture and journal append.
- Inventory every build-103 object dependency and implement snapshot fixtures.

Exit criterion: a snapshot taken mid-1-1 contains every state owner in the
inventory and does not consume one-shot state.

### Stage 4: build-103 rejoin projection

- Implement `zone/raknet103` projection ordering.
- Send the baseline, `ReconnectPlayer(GameDungeon)`, catch-up, then open commands.
- Make projection failure retryable.
- Add exact four-byte state fixtures and semantic-to-packet allowlist tests.

Exit criterion: one client disconnects and rejoins the same 1-1 zone with
position/resources/objectives/enemies/loot preserved while another client
keeps playing.

### Stage 5: launcher and explicit abandonment

- Expose authenticated `ResumeStatus`.
- Add Darkspinner Continue/Abandon UI and post-exit refresh.
- Route Return to Ship, Abandon and expiry through the coordinated leave
  operation.
- Add result/vote absence deadlines.

Exit criterion: crash/close offers Continue; explicit Abandon clears membership
and `current_game_id`; co-op peers remain in the live zone.

### Stage 6: durable checkpoints

- Add versioned checkpoint/journal tables and a feature-owned SQLite adapter.
- Implement compact milestone checkpoint first.
- Restore under a new zone generation and validate content/version.
- Add full exact snapshots only if exact post-restart continuation is worth the
  additional runtime serialization contract.

Exit criterion: process restart restores gameplay-equivalent 1-1 progress
without duplicate objectives, spawns, loot or rewards.

## Required verification

Feature tests:

- same member/game retains slot through disconnect and rejoin;
- transport loss leaves `current_game_id`, hero and shared world intact;
- explicit leave and grace expiry clear membership and current game exactly
  once;
- new generation reserves and commits atomically;
- stale, duplicate and wrong-zone generations cannot mutate;
- snapshot failure and projection failure leave no active duplicate;
- snapshot revision plus catch-up delivers each concurrent mutation once;
- disconnected co-op member does not stop another member's zone progression;
- result vote/reward transaction survives rejoin and cannot double commit; and
- empty-zone grace and final retirement follow the declared policy.

Snapshot tests:

- partially damaged/moving hero and surviving squad resources;
- dead hero or terminal squad;
- damaged NPC, dormant/admitted population, companion target/action;
- claimed and unclaimed loot/DNA/crystals;
- partial objective, fired script key, closed/open barrier and teleporter;
- horde/boss/director phase and scheduled deadline;
- cooldown/modifier and cancelled presentation runs;
- active Continue/cash-out result transaction;
- RNG and object-ID cursor continuity; and
- deterministic ordering of snapshot collections.

Adapter tests:

- exact `0x81` four-byte `GameDungeon` and failure fixtures;
- referenced objects are created before updates/targets/modifiers;
- ordinary C2S packets are rejected until activation;
- journal catch-up switches to live delivery without a gap;
- replacement address routes only the active generation; and
- failure never sends fresh-dungeon setup.

Human tests:

- solo 1-1 crash/relaunch at opening, mid-horde, boss, defeat, victory vote and
  cash-out;
- two clients with one disconnected while the other kills enemies, advances an
  objective and collects eligible loot;
- client-only presentation soft lock followed by relaunch;
- transport stall without process exit;
- explicit Return to Ship versus window close;
- grace expiry with and without a connected co-op member; and
- checkpoint restore after process restart at each idempotency-sensitive
  milestone.

## Remaining retail-capture and client-analysis blockers

The following must remain configurable or adapter-local until evidence is
recovered:

- exact reconnect grace duration and per-disconnect-class policy;
- whether each connection loss enters reconnect automatically or via prompt;
- exact packet order before and after `ReconnectPlayer`;
- RakNet reliability, ordering channel and acknowledgement behavior for `0x81`
  and baseline packets;
- exact build-103 late-join reflection for director, horde, boss, threats,
  objectives, pickups, result voting and game-over states;
- whether a reconnecting peer may receive roster/game setup before `0x81`;
- client behavior when `0xffffffff` is received in each login state;
- which transition, cinematic, cash-out and completed states retail permits as
  resumable;
- client handling of an actor whose in-flight animation/action is reduced to
  authoritative idle;
- exact Return to Ship/Abandon Blaze reason codes outside the proven tutorial
  reason; and
- whether retail pauses an empty zone or advances its deadlines during grace.

Capture priorities are: a two-client mid-zone disconnect/rejoin; reconnect
during a live NPC action and objective change; reconnect during victory/result
voting; explicit Return to Ship versus process termination; and a transport
stall where the client process remains open. Each capture should correlate
account XML, Blaze RPC/notifications, gameplay application packets, RakNet
address/generation, game ID, player slot and timestamps.
