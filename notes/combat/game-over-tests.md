# Game-over and restart implementation/test oracle

## Purpose and authority

This note turns the recovered build-103 behavior in
`game-over-owner.md`, `death-live-contract.md`, and `tutorial-restart.md`
into acceptance criteria. It does not describe the current implementation.

The words **must** and **must not** below are the implementation oracle. Two
boundaries remain intentionally qualified:

- The one-player tutorial wipe rule is recovered with very high confidence.
- The multiplayer reduction over all participating players is the best
  supported policy, but is not proven by an original server call site. Keep it
  behind a match-policy function so a later retail capture can replace it.

RakNet reliability, ordering channel, datagram sequence numbers, and exact
flush latency are not recovered. Tests must assert application-message bytes
and their order in the accepted match outbox, not invent transport framing.

## State and ownership model

One serialized match aggregate owns all of the following for one gameplay
epoch:

- participating players and departure state;
- three selected match-character slots per player;
- the cached HP of each match character;
- the deployed slot and its live replicated-object binding;
- the object manager, simulation clock, encounter state, effect-slot pools,
  scheduled work, and cancellation scopes;
- the terminal latch and the once-per-peer game-over publication record; and
- an epoch/generation token captured by every asynchronous callback.

The aggregate may be referenced by `game.Instance`, but Blaze membership in
`Instance.players` is not combat state. `gameplayPeerSession.heroHitPoint` is
also not an acceptable authority: one peer-local scalar cannot represent three
characters or a multiplayer party.

The damage/death operation is the owner of the wipe decision. The RakNet
handler may decode a command and publish already accepted output, but must not
independently decide that the match is over. Persisted `sporenet.User` and
`sporenet.Creature` records must not be mutated to record match damage or death.

### Character HP source

For player `p` and slot `i` in `[0, 3)`:

```text
hp(p, i) = live object combatant HP, if i is the deployed slot
           cached match-character HP, otherwise

living(p)     = count(i where slot i is available and hp(p, i) > 0)
player_dead(p) = living(p) == 0
```

An absent or unavailable slot contributes zero lives. HP equal to zero is
dead; HP greater than zero is alive. Negative damage results must be clamped to
zero before evaluating the predicate. Persistent creature health, visible
object count, `IsDeployed`, and the most recently damaged object are not
alternative sources.

The deployed object's HP and the corresponding character cache must change in
the same serialized operation. `Character.IsDead`, if retained, must be a
derived/cache field updated in that operation; it must never disagree with
`hp <= 0` or act as a second authority.

Deploying a reserve must copy/use that reserve's match HP, bind its live object,
and make the formerly deployed character's final live HP its reserve cache.
Deployment is rejected for an absent or zero-HP character. Killing the active
character is not terminal while either reserve has HP greater than zero.

## Terminal latch

For the solo tutorial:

```text
party_wiped = player_dead(the sole participating player)
```

The isolated multiplayer policy is:

```text
party_wiped = every participating, non-departed player is player_dead
```

An empty participant set must not satisfy `party_wiped`. One player's squad
wipe must not end a multiplayer match while a participating teammate has a
living character. Do not add a pending-resurrection or available-orb exception:
no such retail latch input has been recovered.

The lethal command is one transaction at the game-loop boundary:

```text
apply/clamp HP
-> update character death/cache state
-> recompute affected player_dead
-> recompute party_wiped
-> compare-and-set terminal Active -> Failed
-> cancel all work owned by the old epoch
-> enqueue AD 01 once for each connected match peer
```

Only the successful `Active -> Failed` transition has side effects. The latch
is monotonic for that epoch: revival, late healing, a duplicated death callback,
or a peer retransmission cannot reopen it. A failed match can leave `Failed`
only through explicit return/retirement or creation of a replacement epoch.

The implemented `sim.Squad` boundary enforces this independently of transport
cancellation. After the latch, duplicate HP updates are mutation-free and do
not relatch; availability and mana changes are rejected, and the tutorial
squad-healing adapter emits no healing results.

The current compatibility restart path also reserves terminal `status=8`
admission before resetting durable tutorial experience. A concurrent duplicate
is ignored, reset failure rolls the reservation back, and replacement requires
the same generation to remain terminal and reserved.

Successful Beam Out uses a separate at-most-once reservation. Its complete
deployed-hero presentation batch is encoded before the durable completion
operation; encoding failure rolls the reservation back and makes no persistence
call. Only then may the route commit and update the account.

The delayed positive tutorial snapshot is authorized only while the same
generation remains the committed completion. A Hello or other session
replacement before its deadline suppresses the stale producer rather than
publishing `C8 00` into the replacement match.

Once the latch changes, gameplay commands, damage, pickups, deployments,
encounter advances, rewards, and persistence completion from that epoch must be
rejected or ignored. Cancellation must cover encounter and horde runs, death
continuations, melee/cloud/projectile attacks, ability/cooldown jobs, marker
waits, teleporter handoffs/contact, loot/obelisk jobs, and any future
object-scoped producer. Every callback must additionally compare its captured
epoch token with the current epoch before reading, mutating, or publishing.

## Application packet contract and ordering

All bytes in this section begin at the GMS application boundary. Payload widths
are exact:

| Meaning | Direction | Exact bytes | Width |
| --- | --- | --- | --- |
| Select GameOver | S2C | `AD 01` | 2 |
| Game-over-state no-op | S2C | `AD 00` or `AD 01` | 2 |
| Select Spaceship | S2C | `AF 00` | 2 |
| Reload current level | S2C | `BF` | 1 |

No player ID, game ID, object ID, epoch, timestamp, reflection terminator, or
other byte follows any of these messages. There is no C2S game-over ACK,
restart-button message, `AE`, or `BF`. RakNet ACKs are transport ACKs only.

### Failure order

The old epoch's killing combat/HP replication must be accepted before `AD 01`.
The terminal latch and cancellation happen before `AD 01` is enqueued. Exactly
one `AD 01` is enqueued per connected match peer for the epoch. No stale
old-epoch producer may enqueue application traffic after it.

`AD 00`/`AD 01` are not part of the server failure handshake and must not be
generated as an acknowledgement or substituted for `AD 01`.

### Native HUD return

The recovered death button is entirely local:

```text
old-epoch lethal replication
-> AD 01
-> client enters GameOver and opens HUD_Death.swf
-> local OnExitClicked/outro
-> client selects Spaceship
```

The server sends no `AE`, `BF`, or application acknowledgement for the button.
A later tutorial entry is a fresh game-entry lifecycle and must not reuse the
failed gameplay epoch.

### Optional server-authored return

`AF 00` is a distinct server-authored return-to-Spaceship command, not the
reply to a client button packet. Its recovered handler selects Spaceship while
the active gameplay state owns that handler. If product policy chooses a direct
server return instead of presenting GameOver, the accepted application order
is:

```text
old-epoch application traffic -> AF 00 -> retire the gameplay epoch
```

It must be triggered by an explicit server lifecycle decision, never by a
fabricated C2S ACK. It retires the gameplay epoch; it is not followed by `BF`
or replacement-world traffic on that epoch. Do not automatically enqueue it
after `AC`: the executable does not prove an `AC`/`AE` pair, an application
acknowledgement, or a timing rule between them. Once `AC` has selected the
GameOver presentation, the native HUD callback owns the recovered return.

### Optional in-place restart

An in-place restart is a separate product policy. It preserves the connected
RakNet peer and reliability session but replaces all gameplay state. For a
restart of a failed match, the application order is exactly:

```text
old-epoch lethal replication
-> AD 01
-> cancel/fence and dispose the old server epoch
-> BF
-> replacement GameState/world initialization
-> replacement ObjectCreate and subsequent replication
```

`BF` is the boundary between epochs and must be serialized before every
replacement `GameState`, world, or object message. No `AE` participates. No
old-epoch update or `ObjectDelete` may appear after `BF`; the client reload
tears down the old level itself. The server may run its old-object deletion
sweep before `BF`, but deletion packets are unnecessary for the level-wide
boundary and must never leak across it.

Because `BF` has no epoch or acknowledgement, the server must not wait for or
decode a fictitious reload reply. Reusing numeric object IDs in the replacement
epoch is allowed only after `BF`, because the client and server object maps have
both been cleared and stale callbacks are generation-fenced.

The three lifecycle policies are mutually exclusive for one decision:
`AD 01` plus native HUD return, direct server-authored `AF 00` retirement, or
`AD 01` followed by server-authored `BF` reload. A duplicate decision must not
enqueue a second transition packet or a second replacement world.

## Session reset and object lifecycle

### Preserved across `BF`

- the connected UDP/RakNet peer and its reliability state;
- the authenticated gameplay identity and authorized game membership/binding;
- the selected level tuple needed to rebuild the same level; and
- durable account data listed in the persistence section below.

Preservation does not mean that a stale callback may retain a pointer to the
old match aggregate. The binding is input to constructing a new aggregate.

### Reset/recreated across `BF`

The replacement epoch gets a new generation token and fresh instances of:

- terminal state/publication flags and simulation clock;
- all three match-character HP/death records, deployed index, live object
  bindings, mana, position, cooldowns, modifiers, targets, and locomotion;
- object manager/map, object IDs/instances, combatants, collision/nav state,
  attached-effect slot pools, projectiles, enemies, pickups, and loot;
- opening, Sage, horde, arena, teleporter, ability-lesson, dungeon, encounter,
  and completion state;
- collected-orb, used-obelisk, spawn/unlock/sent flags and other match-local
  idempotency sets; and
- all timers, waits, attack/death runs, cancellation scopes, and producer
  handles.

Initial replacement HP/mana comes from the normal match-entry rules, not the
failed epoch. No old object pointer, effect handle, cooldown deadline, target,
position, corpse state, `IsMarkedForDeletion`, or presentation flag may survive.

Teardown must cancel object-owned work before dropping object maps. A callback
racing teardown may complete its local calculation, but its generation check
must prevent mutation/publication. Teardown is idempotent: repeating it neither
panics nor publishes removal twice. Replacement objects must be newly created,
even when their noun and numeric object ID match an old object.

An `AE`/native-HUD return performs cancellation and retirement but creates no
replacement gameplay aggregate. A later tutorial launch constructs one through
the normal join/start lifecycle.

## Persistence boundary

Failure and `BF` must not change durable account state merely because they
occurred.

| Persists | Match-only and reset |
| --- | --- |
| account identity/authentication | current/max HP and mana |
| owned creature roster and authored creature stats | dead flags and deployed slot/object binding |
| selected durable squad/loadout | position, targets, locomotion, modifiers, cooldowns |
| onboarding progress already committed before the match | simulation time, terminal latch, epoch token |
| inventory, currency, XP, and grants already committed by an accepted idempotent operation | objects, effects, enemies, projectiles, pickups, encounter/lesson flags |
| Blaze game membership/binding while an in-place `BF` keeps that game alive | uncommitted loot/rewards and every scheduled job |

Death is not tutorial completion. It must not emit a successful
`TutorialGameMsgs` subtype-0 completion snapshot and must not advance
`new_player_progress` to `3000`. An incomplete durable value (`0`, `1000`, or
`2000`) remains incomplete, so returning to Spaceship routes the player back
through tutorial entry rather than unlocking normal squad management.

Already committed idempotent grants are not rolled back by restart and must not
be granted again merely because the replacement epoch recreates a pickup or
lesson. Uncommitted match loot is discarded. Tests should seed both categories
explicitly instead of treating all rewards as either durable or transient.

## Ordinary NPC death interaction

Game-over ownership does not collapse the ordinary NPC death timeline into an
immediate delete. For a replicated `TutorialBasicDiseased`:

1. At `t=0`, killing combat/HP-zero and the death animation are emitted;
   target, immobilization, locomotion, physics, and navigation mutate through
   their ordinary authoritative owners.
2. At `t=10s`, if still at zero HP, corpse-fading is set and a fade effect is
   allocated from the first free slot in the object's 16-slot pool.
3. At `t=15s`, if still dead, `MarkForDelete` is set. The deletion sweep and
   normal replication own `ObjectDelete`.

For Life type, attach and soft-removal application bytes are:

```text
attach:  9b 01 SS 04 01 06 08 3c 27 ea 07 OO OO OO OO ff
remove:  9b 01 SS 02 01 07 OO OO OO OO ff
```

`SS` is the allocated internal slot plus one and `OO` is object ID in little
endian. Full slot exhaustion sends no attach packet but does not alter the
five-second deletion deadline. Revival before 10 seconds sends no removal.
Revival during fade sends exactly one soft removal, clears fading, restores
collision/navigation, removes its own immobilization handle, and sends a reset
animation with a freshly sampled simulation timestamp. The normal timeout path
sends no soft removal before the eventual `8e <object-id-le>` delete.

An in-place restart or match retirement cancels this continuation. If restart
wins before mark/delete publication, no later fade, removal, or delete from the
old epoch may cross `BF`. If the deletion sweep wins before restart, its delete
may appear only in the old-epoch portion before `BF`. Cleanup and teardown must
remain safe when both paths attempt to release the effect slot.

## Required tests

Test names are illustrative; behavior and observable assertions are normative.

### HP and latch table

| Case | Slot HP `(0,1,2)` | Deployed | Expected |
| --- | --- | --- | --- |
| Active dies, reserve lives | `(0, 1, 0)` | 0 | no latch, no `AC`; slot 1 deployable |
| Reserve cache lives | `(0, 0, 25)` | 0 | no latch, no `AC` |
| All available dead | `(0, 0, 0)` | any | one `Active -> Failed`, one `AC` per peer |
| Boundary is strict | `(0, smallest-positive, 0)` | 0 | alive; no latch |
| Missing reserves | `(0, absent, absent)` | 0 | latch |
| Stale `IsDead` disagrees | live HP `10`, flag true | live slot | HP wins or invariant failure; never latch from flag alone |
| Live/cache handoff | deployed HP changes then deploy reserve | changes | old live HP becomes cache; new live HP equals its cache |

Also test lethal overkill clamps to zero and that the latch observes the new HP,
not the pre-damage value. Run the same lethal command concurrently/duplicated
through the serialized executor and assert one latch transition and one packet
per peer.

### Party policy

- Solo: the only player's three dead slots latch failure.
- Two players: one wiped plus one living does not latch.
- Two players: the last living slot reaches zero and latches once for both
  connected peers.
- A departed player is excluded; an empty participant set does not latch.
- A dead player becoming revivable while a teammate lives does not affect the
  latch. No orb/pending-resurrection exception is consulted when the final
  living slot reaches zero.

Mark the multiplayer cases as policy tests, not recovered-server golden tests.

### Packet bytes and state-machine order

- Golden marshal: `GameOverMessage == ac 01`,
  `ReturnToSpaceshipMessage == ae 00`, and `ReloadLevelMessage == bf`.
- Assert exact widths and absence of trailing fields.
- Assert the killing HP/combat output precedes `AC`, while latch/cancellation
  are observable before the `AC` enqueue.
- Assert duplicate deaths and duplicate terminal requests do not duplicate
  `AC`, `AE`, `BF`, or replacement initialization.
- Assert no `AD`, fake C2S ACK, or restart request is awaited or generated.
- Local-return scenario: output ends at `AC`; later game entry uses a fresh
  normal lifecycle.
- Direct server-return scenario: one explicit `AE` while gameplay is active;
  no `AC`, `BF`, or new world.
- Reload scenario: `AC`, then one `BF`, then GameState/world initialization,
  then ObjectCreate/replication; no `AE` and no old-epoch output after `BF`.

The ordering assertion belongs at the application outbox/serialized publisher,
not at UDP arrival timing.

Darkspin now enforces this boundary in the gameplay adapter. The callback that
latches the last squad death is allowed to finish its damage and HP batch and
append `AD 01`. Inbound actions and all subsequently invoked delayed producers
consult the same squad terminal latch and return no packets. The original run
handles remain owned by the terminal session until peer replacement or fresh
tutorial entry acknowledges their cancellation, avoiding cancellation from
inside the currently executing scheduler callback.

The success terminal uses a separate two-step latch. Beam Out first reserves an
otherwise live, horde-complete session, performs the durable idempotent tutorial
completion, and marshals the response before committing the terminal fact.
Concurrent or duplicate requests cannot enter while pending or committed.
Persistence and response-marshal failures release the reservation for an
explicit retry; a game-over squad can never reserve successful completion.
After commit, the common terminal producer gate suppresses stale work and the
inbound status handler safely cancels every session-owned run.

### Reset, teardown, and races

- Populate every current `gameplayPeerSession` flag, map, pointer, cooldown,
  encounter, attack, death run, projectile counter, HP/mana value, position,
  object, and effect slot; restart and assert none enters the new epoch except
  the explicitly preserved binding/peer/level tuple.
- Assert every cancellable run's stop path executes once and teardown can be
  repeated safely.
- Fire callbacks captured before failure both immediately before and after the
  epoch swap. Only the callback that wins before the terminal transaction may
  affect old-epoch output; none may affect the replacement epoch.
- Race player death with `BF`: either death wins (`AC` before `BF`) or restart
  was already committed and the old death is rejected. `AC` must never occur
  after `BF` for the old epoch.
- Race NPC fade/delete/revival with teardown and assert effect slots are freed
  once, no panic occurs, and no old packet crosses `BF`.
- Assert replacement objects are distinct instances with clean components and
  jobs. If IDs are reused, the first occurrence in the new epoch is an
  `ObjectCreate` after `BF`.

### Persistence

- Snapshot the durable account before lethal damage and compare after failure,
  native return, `AE` return, and `BF` reload.
- In particular, assert match HP/death never writes creature/account storage
  and `new_player_progress` does not become `3000`.
- Seed a previously committed idempotent grant and assert it survives but is
  not duplicated after restart.
- Seed uncommitted match loot and assert it is absent from both durable storage
  and the replacement epoch.

### Ordinary death regression

Retain semantic-clock tests for death animation, pre-10-second revival,
10-second first-free-slot fade attach, fade-window revival/soft removal,
15-second mark plus sweep-owned delete, slot exhaustion, and cancellation.
Add the epoch-boundary race assertions above; do not add fabricated attribute or
collision presentation packets.

## Completion gate

The implementation is conformant only when all of these are true:

- squad death is computed from three match HP records with the live/cache split;
- the match aggregate, not transport or persistence, owns one terminal latch;
- `AC`, optional `AE`, and optional `BF` have exact bytes and mutually exclusive
  lifecycle ordering;
- restart preserves only peer/binding/durable state and creates a clean epoch;
- old objects and jobs cannot mutate or publish into the new epoch;
- failure cannot advance tutorial completion; and
- duplicate and transition-race tests prove at-most-once terminal output.

Do not claim retail fidelity for multiplayer departure/resurrection policy or
RakNet framing until a clean retained retail capture or original server sender
proves those details.
