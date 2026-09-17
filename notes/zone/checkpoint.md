# Zone safe checkpoints

## Contract

A checkpoint restores a gameplay-equivalent safe boundary, not an exact frame.
It is intended to support rejoin after a process restart without replaying an
unsafe combat instant or fabricating client presentation state.

The server captures an immutable in-memory snapshot and returns immediately to
gameplay. A coalescing writer persists the newest revision for each zone to the
runtime `darkspin.db` on a separate goroutine. Shutdown drains pending writes.

## Save boundaries

- Save after an encounter spawn group is fully cleared. The safe anchor is the
  surviving hero position at that committed boundary.
- Save after a durable equipment or crystal pickup only when no living NPC has
  an active target. Inventory ownership commits before the checkpoint is
  queued.
- Do not save every movement, attack, projectile, effect, or AI tick.
- A later safe boundary replaces the older pending revision for the same zone.

## Snapshot contents

- Zone ID, generation, level, difficulty, revision, reason, and timestamp.
- Retained member identity, slot, ability count, connection state, and peer
  generation for correlation diagnostics.
- Complete three-character squad resources, deployed slot, and safe position.
- Active hero identity, creature slot, footprint, maximum resources, and
  stealth state.
- Complete reconstructable NPC spawn plans plus current and home positions,
  facing, health, power, encounter identity, publication state, and defeated
  state.
- Explicit cleared spawn-group IDs so retired corpses cannot make an encounter
  respawn during restoration.
- Per-member crystal inventory committed inside the active zone.
- Objective snapshots and committed authored script-use counts so fired
  interactions are not replayed after restoration.
- Committed security-route and teleporter-presentation indices so an already
  cleared floor cannot resume behind an inactive traversal gate.
- Elapsed mission time so time-based objective evaluation remains continuous
  across a process restart.

Inventory, account progression, and awarded parts remain owned by their
existing transactional profile persistence. A checkpoint does not duplicate
those aggregates.

## Restore normalization

When durable restoration is admitted, it must start a new zone incarnation and
rebuild semantic state before any build-103 baseline packets are published:

1. Reconstruct the durable game membership and original player slots.
2. Restore the complete squad resource state, not only the deployed hero.
3. Keep defeated NPCs defeated and restore surviving NPCs at their loose saved
   positions in idle/home state with no target or action owner.
4. Restore heroes at the saved safe anchor. If invalid, fall back in order to a
   cleared encounter anchor, teleporter arrival, then zone entrance.
5. Discard in-flight attacks, projectiles, scheduled callbacks, temporary
   modifiers, and presentation queues.
6. Rebuild population resolution, object-ID allocation, objectives, result
   state, and pickup ownership from their authoritative owners.
7. Publish one revision-stable rejoin baseline and only then open gameplay
   commands for the replacement peer generation.

## Current implementation boundary

Safe checkpoint capture, asynchronous SQLite persistence, and restart
restoration are implemented. The member index recreates the Blaze game shell
with its original game ID, player slot, expected member count, and deterministic
lowest-slot host. Every retained member must cross the restored ready barrier
before RakNet creates a new zone incarnation from the complete squad, safe hero
position, surviving and defeated
NPCs, cleared spawn groups, crystal inventory, objective state, and committed
script uses. The snapshot also retains every accepted per-NPC, per-member XP
award and one stable completion identity. Account persistence records that
identity in a private campaign-experience ledger, making a repeated completion
after process restart return the original cumulative XP receipt rather than
applying the award twice. Committed security progress restores the active state of every
already-presented teleporter; horde-wave and boss transitions are deliberately
excluded from checkpoint boundaries until their dedicated encounter state is
durable. Surviving NPCs resume idle without targets or in-flight actions,
and future object IDs advance past every restored actor. Completed and
explicitly abandoned games discard their checkpoint.

Durable membership contains only account identity, stable player slot, and
gameplay ability count. Peer generations and connected flags are process-local
transport state and are never persisted. On restoration, every saved member is
reserved as disconnected in the zone, result, and vote authorities until a new
peer binds. A checkpoint captured while another member remains disconnected
retains that member's prior hero record rather than shrinking the shared run.

Failed checkpoint saves and deletes remain queued for periodic retry. An
orderly server shutdown performs a bounded final drain and reports an
incomplete flush instead of silently losing the last restore point.
Terminal zone state fences the final in-memory enqueue, so a safe-boundary
capture that began before completion cannot recreate a checkpoint after the
result transaction discarded it. SQLite also rejects stale snapshot revisions
before changing the member resume index.
All manager-owned store operations share one storage fence. A save rechecks the
process-lifetime discard tombstone while holding that fence, which establishes
an unambiguous winner between an already-running save and Start Fresh: either
the save finishes first and is then deleted, or deletion finishes first and the
save is skipped.

Darkspinner queries this same member index before launch. Continue retains the
checkpoint for normal Blaze and RakNet restoration. Start Fresh removes only
the selecting member's durable member, squad, hero, crystal, objective-slot,
and experience state, then removes that member's live or retained gameplay
session and Blaze membership. The remaining checkpoint keeps original player
slots while reducing its expected readiness count, so another established
member can restore without the player who declined. The final member's Start
Fresh invalidates the checkpoint, retires the gameplay zone and retained
sessions, and removes the Blaze shell. Both paths complete their durable write
before launching to the ship, and the manager filters later safe captures
through its process-lifetime declined-member set so stale work cannot recreate
the removed Continue entry.

Same-process reconnect still uses the exact retained live zone. Process-restart
restore intentionally drops transient pickups, projectiles, callbacks,
modifiers, combat targets, and presentation queues. Result state is not
checkpointed because current safe boundaries precede a terminal result
transaction; broader mid-result checkpointing must add that owner first.
The restored Blaze shell also reapplies the fresh client's capacity, network,
and topology envelope while retaining the checkpoint's level, difficulty,
member count, and player slots. Later co-op members are rejected if their
checkpoint metadata disagrees with the already-restored shell.
Validation rejects malformed zone identity, save reason, duplicate members or
slots, non-finite squad positions, unrestorable squad state, invalid hero
object blocks, mismatched deployed indices, and non-finite or out-of-bounds
hero resources before live construction. It also rejects negative elapsed
time, malformed per-member crystal inventories, and duplicate or zero cleared
spawn groups, and requires one squad, crystal inventory, and active-hero record
for every durable member rather than silently normalizing corrupted state.
Objective records must be unique, bounded to known medal states, active for
every durable member slot, and inactive for every unowned player slot.
Experience records must have unique NPC identities, positive base and member
awards, known member owners, and overflow-safe per-member totals.
Every newly captured snapshot passes the same validation before entering the
asynchronous persistence queue. The restored active hero's exact
position, health, mana, and stealth are applied to the fresh transport
generation before its client baseline is published. If that saved active hero
is dead but another selected hero is alive, restoration deploys the first
living squad member and deliberately drops the dead hero's transient pose and
stealth instead of reopening an impossible selection state. Tutorial restoration
requires its authored difficulty zero and single-player roster, while ordinary
campaign checkpoints require a positive difficulty.

## Next integration milestone

Validate restart restoration from a genuine saved mission in the retail
client, and capture result ownership before admitting checkpoint boundaries
during a terminal result transaction. A production launcher/server startup and
shutdown cycle has been exercised; the full client restore remains the human
acceptance boundary because no safe checkpoint existed during that cycle.

The acceptance run is intentionally narrow:

1. Enter a campaign and clear one complete spawn group, then wait for
   `Zone checkpoint saved` with the expected zone, user, and member count.
2. Exit Darkspinner without completing or abandoning the mission, then restart
   it against the same `darkspin.db`.
3. Select the same profile and confirm the launcher offers Continue with the
   expected mission label and difficulty.
4. Choose Continue and confirm `Zone checkpoint resume found on disk` appears
   before the restored ready barrier opens RakNet gameplay.
5. Confirm the hero resources and safe position are retained, cleared enemies
   stay dead, surviving enemies resume idle, and committed loot is still owned.
