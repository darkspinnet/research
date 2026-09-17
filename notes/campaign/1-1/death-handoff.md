# Campaign 1-1 ordinary death and loot handoff

## Scope and confidence

This contract covers the first accepted, non-critical live-to-zero transition
of an ordinary campaign 1-1 enemy. It does not define player death, revival
rewards, the critical `0.1 + 3` second corpse branch, the Illust final-boss
predicate, or loot probabilities and item generation already bounded in the
campaign drop notes.

Confidence labels mean:

- **Exact**: directly present in build-103 packaged Lua, native code, or
  reflected fields.
- **High**: the recovered mechanisms and negative evidence constrain the
  conclusion strongly, but the missing retail server owned the call site or
  flush timing.
- **Fallback**: a necessary conservative server-authority decision recorded in
  [`help.md`](help.md), pending a retail server trace or recovered server body.

The ordered contract distinguishes *defeat* from *corpse deletion*. An enemy
stops being a live threat and encounter member at the accepted live-to-zero
boundary; its replicated corpse remains an object for about 15 seconds.

## Evidence boundary

### Packaged Lua

Runtime `content.db` identifies chunk `949`, resource `14528`, source
`behaviors/0xD6945860.lua`, SHA-256
`fe176b54253ea7251a3d4346664323d382fe17b28323662505c7afbdad93d724`.
Its `nBehavior_Death` entry, update, and cleanup functions prove this sequence:

1. `SetTargetID(object, kObjIDNone)`, death animation selection,
   `Immobilized += 1`, locomotion stop/knockback turn, and physics/navigation
   collision disable;
2. for an ordinary non-player, wait up to `npcFadeOut=10` seconds for HP above
   zero;
3. if still at zero, set corpse-fading, attach the creature-type fade, wait
   `fadeTime=5` seconds, then `MarkForDelete`;
4. on revival/interruption, remove the scoped modifier and, unless already
   marked, reset animation, remove the fade, and restore both collision modes.

The chunk contains no `DropStuffForObject`, XP award, threat-removal,
director-membership, `horde complete`, route-clear, or boss-completion call.
That is strong negative evidence that `Behavior_Death` is corpse presentation
and cleanup, not the authoritative kill/reward/encounter transaction.

### Native code

The relevant retained build-103 bodies in
[`Game.c`](../bin/game/GameBin/Game.c) are:

| Native | Retained fact | Confidence |
| --- | --- | --- |
| `sub_9E3DC0` (`SetTargetID`, C line 1399061) | Writes only the explicit target slot at blackboard `+1360`; setting the dying actor to zero does not itself prune other threat vectors or reciprocal attacker links. | **Exact** |
| `sub_9E4640` (C line 1399544) | Threat insertion appends `(targetID, amount)` to the actor and adds the aggressor to the target's reciprocal attacker collection at `+1084..+1092`. | **Exact** |
| `sub_9E3E50` (C line 1399120) | Best-target selection skips candidates that no longer resolve or fail targetable/hostile/live checks. A zero-HP defeated actor therefore must not remain eligible merely because its corpse still resolves. | **High** |
| `sub_A035C0 -> sub_9CE8D0` (C lines 1425508 and 1381099) | `DropStuffForObject(source, context, selector?)` gates on source byte `+153`, then queues context ID, signed selector, current simulation deadline, and processing flag at combatant `+92..+104`; omitted selector is `-1`. | **Exact** |
| `sub_9CE140` (C line 1380792) | Selector bits are `0x02` orb, `0x04` crystal/catalyst, `0x08` equipment, and `0x10` DNA. The later processor creates pickups independently of XP and corpse scheduling. | **Exact** |
| `sub_A02910 -> sub_9BD430` (C lines 1424877 and 1366357) | `MarkForDelete` only sets object byte `+93`; the simulator's deletion sweep later tears the object down and ordinary replication owns `ObjectDelete`. | **Exact** |

The build-103 client contains no native caller that connects an ordinary enemy
HP-zero transition to `DropStuffForObject`, and the 1-1 director noun classes
all have `dropType=[]`. Empty class-local selectors do not mean “no drops”:
the walkthrough visibly shows ordinary DNA, health, and equipment drops, while
the native wrapper's omitted selector becomes `-1` and therefore includes all
known selector bits. The missing retail server chose the actual ordinary-death
selector and context actor.

### 1-1 content and walkthrough

The 1-1 walkthrough repeatedly shows the loop “island combat, drops, then
bridge/teleporter traversal”; ordinary kills visibly produce DNA, health, and
equipment pickups. It does not expose multiplayer ownership, a per-kill
selector, an exact same-frame packet order, or a named route-clear message.

Authored horde sets `1827` and `1828` do have marker-set-scoped
`horde complete` consumers, while the boss-security teleporter instead polls a
radius-20 live-organic hostile query and becomes active when no qualifying
threat remains. Therefore ordinary security/route clearing must not fabricate
a horde event. See [`1-1-horde-runtime.md`](1-1-horde-runtime.md),
[`1-1-completion.md`](1-1-completion.md), and the
[`1-1` indexed walkthrough](../bin/video/walkthrough/1-1/info.md).

## Ordered authoritative contract

| Order and time | Contract | Confidence |
| --- | --- | --- |
| 1. First accepted live-to-zero transition, `t=0` | Under the match/encounter epoch, accept exactly one defeat for an actor whose authoritative HP was positive and becomes zero. Latch the killing source/context before later projectiles, duplicate damage, corpse callbacks, or deletion can repeat the handoff. Killing `CombatEvent`, HP-zero state, XP eligibility, drop eligibility, and corpse behavior remain separate consequences of this one accepted transition. | **High** for the required idempotent boundary; the absent retail server owned its transaction layout. |
| 2. Death behavior entry, same simulation update | Start `nBehavior_Death`: clear the dying actor's own blackboard target to zero, select/send its death animation, apply scoped immobilization, stop locomotion, and disable physics and navigation collision. The corpse remains replicated and addressable; do not send `ObjectDelete`. | **Exact** Lua/native sequence. Relative network flushing inside the update is open. |
| 3. Retire combat authority, same transition | Remove the defeated actor immediately from every live-threat/reciprocal-attacker view, make it ineligible as anyone's best target, cancel its pending ability/AI admission, and publish affected `0x99` snapshots before any replacement target-consuming movement or action. Its own target is already zero from step 2. Do not wait for fade or deletion to reduce hero attacker counts or retarget survivors. | **Fallback** for immediate prune/flush policy; **Exact/High** that explicit target clear alone is insufficient and live filtering must exclude the corpse. |
| 4. Remove director/encounter live membership | Remove the actor exactly once from its owning director pack or horde `liveEncounterActorIDs`, recording “defeated” rather than “despawned.” Keep the corpse in the object table under the death-run owner. Corpse fade, mark, sweep, and duplicate death callbacks must not touch live membership again. | **Fallback** timing, consistent with the existing horde policy and required to prevent a 15-second false combat gate. No client/Lua body exposes the retail director removal call. |
| 5. Queue the ordinary drop selection | If source byte `+153` permits loot, invoke `DropStuffForObject(defeatedActor, killingControlledAgent)` exactly once during the accepted defeat handoff. Pending retail evidence, omit selector 3 so native `-1` enables the known orb, crystal/catalyst, equipment, and DNA branches; native chances, budgets, global gates, and eligibility may still produce zero or more pickups. Queueing is at current simulation time and is independent of byte `+154` XP eligibility. Do not defer this invocation to `t=10` fade or `t=15` deletion. | **Fallback** for caller, context actor, and `-1` selector; **Exact** queue layout, selector vocabulary, and independence from XP/corpse behavior. |
| 6. Create shared world pickups | The later drop processor samples around the corpse/source position, creates eligible pickup objects, publishes category data/events, and starts the native `2.5`-height, `0.5s` lob. The queued agent is chance/selection context, not an exclusive pickup owner; do not copy it to `ownerId`. Ordinary equipment has zero loot-instance ownership and uses the collection-time player-roll path; crystals use their capacity-checked interaction; orbs use server-owned walk-over collection. | **Exact/High**, from [`campaign-drops.md`](campaign-drops.md) and [`campaign-pickups.md`](campaign-pickups.md). Exact server flush/batching remains open. |
| 7. Re-evaluate the owning clear predicate | After membership removal is committed, evaluate only the owning scope. For horde 1/2, publish one marker-set-scoped `horde complete` only when all planned work is terminal, no spawn is pending, and live membership is empty. For an ordinary non-horde route/security pack, latch any internal pack-clear once and re-run the security/route threat query; publish no `horde complete`. For the final arena, use its separate leader-plus-add predicate and commit `mbBossComplete` only through the final-clear transaction. | **High** for event-scope separation and security polling; existing compound clear predicates are documented **Fallbacks**. |
| 8. Ordinary corpse timeout, `t=10s` | If HP is still zero, set corpse-fading, allocate/attach the creature-type fade, and begin the five-second fade interval. Loot and clear work are already independent and may have completed. If revived before this deadline, run cleanup without a fade slot. | **Exact**. |
| 9. Mark and sweep, `t=15s` | If still dead, call `MarkForDelete`. The same or a later simulator sweep performs teardown; ordinary replication emits/batches `ObjectDelete`. Teardown cleans object-owned effects and cancellation state but must not award XP, invoke the selector, remove membership, or publish clear a second time. | **Exact** mark/sweep ownership; exact datagram latency is open. |

Steps 2 through 7 occur in one authoritative simulation handoff, but the
artifacts do not prove a retail total order among death presentation, threat
pruning, director removal, reward scheduling, and clear publication. The table
uses the conservative causal order needed for correctness: establish death,
remove combat/live membership, queue rewards, then test for a last-actor clear.
No client-visible action may observe the corpse as a live target, and no clear
may be emitted before the membership mutation it depends on.

## Idempotency and failure rules

- Key defeat, drop scheduling, membership removal, and completion work by the
  match epoch plus actor ID. Duplicate lethal damage and the later deletion
  sweep return the already accepted result.
- A drop-selection or clear-publication failure remains retryable from retained
  handoff state; it must not resurrect the actor into combat membership.
- A pickup creation failure produces no phantom owned item. Equipment becomes
  durable only through its collection-time persistence transaction, not at
  enemy death.
- Revival can restore animation, modifiers, and collision before mark-for-delete,
  but it does not silently repeat a kill reward or drop. Whether campaign 1-1
  permits ordinary enemy revival at all is outside this contract.

## Replacing the fallbacks

The fallback portions can be promoted only by a retail campaign server trace
or recovered server body that reveals: threat/reciprocal-link prune timing;
ordinary enemy drop selector and context selection; director membership
mutation and last-member ordering; the ordinary route-clear publisher, if one
exists; and packet/outbox ordering for simultaneous death, drop, and clear.

## Current implementation

Campaign area, projectile, and melee damage now call one shared, actor-latched
equipment handoff on the accepted live-to-zero transition. It uses the
documented common unaffixed generation fallback, publishes the recovered
equipment creation sequence at the retained corpse position, and commits the
item only when collected through the capacity-safe durable transaction. A
generation failure never suppresses death presentation or encounter progress.
The local source amount `50` and resulting `22.5%` base attempt are recorded in
`notes/help.md`. An independent source budget `25` now drives one actor-latched
25% weighted health/mana-orb attempt through the same authoritative orb
lifecycle, using loot randomness isolated from combat. Ordinary enemies also
receive one actor-latched 4% crystal attempt using the footage-backed source
amount `26` and the existing weighted mission-crystal lifecycle. DNA now uses
the footage-backed 30% and 3-20 envelope, the recovered `DNA.Noun` and loot-data
field, automatic contact collection, rollback-safe persistence, sparse player
field-12 publication, and one-time deletion. Its unrecovered tuning and contact
constants remain explicit fallbacks in `notes/help.md`.

The handoff now also updates retained horde and boss ownership for projectile
and Wraith area damage, not only melee basics. Required follow-up waves are
reserved at the kill boundary, admitted transactionally after the retained
inter-wave delay, and enter the shared first-action scheduler; terminal boss
predicates publish the existing completion state. Scheduler rejection uses the
same immediate admission fallback as the melee path.
