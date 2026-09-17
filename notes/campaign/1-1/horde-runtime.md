# 1-1 horde runtime policy proposal

## Purpose

This note turns the recovered `zelems_1` topology into a concrete, playable
server policy for horde 1, horde 2, and the final arena. It is a proposal, not
a claim that the missing retail authority has been recovered.

Three evidence labels are used throughout:

- **Proof** means build-103 client code, decoded content, Lua bytecode, or the
  native reflection layout directly establishes the fact.
- **Walkthrough** means behavior visible in
  `bin/video/walkthrough/1-1/1-1.mkv`, its extracted frames/contact sheets, or
  `bin/video/walkthrough/1-1/info.md`. It constrains the placeholder shape but
  does not establish exact server values.
- **Fallback** means an educated server-authority choice made to keep 1-1
  deterministic and playable until stronger evidence replaces it.

The walkthrough is one edited run. Its timestamps, visible enemy counts,
mutation rolls, implied difficulty, and rewards are not exact authority
values. In particular, the visible `Illust the Accelerator`, `Swift Aura,
Swift`, adds, drops, and reward roll constrain presentation only.

## Policy at a glance

| Encounter | Accepted trigger | Scoped listeners | Gates while active | Single-player fallback | Clear latch |
| --- | --- | --- | --- | --- | --- |
| Horde 1 | row `128245`, radius `15`, once-only | rows `128241`, `128244`, `128247` | gate rows `128237`, `128239`, `128240` plus blocker rows `128242`, `128243`, `128246` | one 3-slot agent wave | all planned work terminal and live membership empty |
| Horde 2 | row `128253`, radius `5`, once-only; requires horde 1 complete | rows `128251`, `128252` | gate rows `128254`, `128256`, `128257` plus blocker rows `128248`-`128250` | 2-slot then 3-slot agent waves | all planned work terminal and live membership empty |
| Final arena | fallback: post-horde-2 arrival intersecting row `126383`'s radius `60` publishes `boss triggered` once | add rows `126384`-`126387`; leader anchor row `126389` | boss-security transfer is the admission boundary; no final-arena horde gate is authored | one 4-slot leader proxy, then 4-slot and 2-slot add waves | leader defeated, all planned work terminal, and live membership empty |

The slot counts and final trigger publisher in this table are fallback. Marker
IDs, horde radii/once flags, listener ownership, and gate/blocker topology are
proof unless a later section says otherwise.

## Common policy

### Encounter identity and event scope

Each encounter is keyed by `(match epoch, marker-set ID)`:

| Encounter | Marker set | Authored activation name | Runtime event key |
| --- | ---: | --- | --- |
| Horde 1 | `1827`, `zelems_1_Ai_Horde_1.Markerset` | `horde triggered` | `zelems_1/1827/horde` |
| Horde 2 | `1828`, `zelems_1_Ai_Horde_2.Markerset` | `horde triggered` | `zelems_1/1828/horde` |
| Final arena | `1793`, `zelems_1_design_spawners.Markerset` | `boss triggered` | `zelems_1/1793/boss` |

**Proof:** the names and marker-set ownership are authored content. The same
`horde triggered` string is reused by sets `1827` and `1828`, while the boss
cluster uses `boss triggered`.

**Fallback:** named-event fan-out is restricted to listeners in the owning
marker set. A horde 1 contact must not awaken horde 2 merely because both sets
use the same event string. Listener execution is sorted by authored marker
ordinal for determinism; the retail same-event listener order is unknown.

### Eligibility and admission

Accept a contact only from an alive, deployed, player-controlled creature
owned by a participant in the current match. One accepted entrant starts the
encounter for the party; no quorum is required. Ignore spectators, stale
object IDs, summons, future movement goals, client-published named events, and
the raw trigger `onExitEvent=horde complete`.

**Proof:** the two horde contacts are player-entry callbacks, and current
campaign trigger evaluation can test the accepted authoritative movement
segment. The client enter callback is inert and cannot be encounter authority.

**Fallback:** actor eligibility, party-wide admission, and the lack of a quorum
are server policy. They should be revisited with a retail multiplayer trace.

Admission uses a reservation:

1. `dormant -> arming` reserves the encounter and closes its gates.
2. Validate the complete spawn plan, object-ID capacity, noun profiles, and
   publication owner.
3. Publish the first wave and then commit the once-only trigger.
4. If planning or publication fails, remove the reservation, restore the
   pre-entry gate snapshot, and leave the trigger retryable.

This matches the existing `CampaignDirectorSession` boundary: observation can
produce a pending publication, but `Accept` consumes once-only policy only
after the authoritative operation succeeds.

### Noun candidates

Every `SpawnPoint_DirectorHorde` is native spawn kind `5`. **Proof:** kind `5`
selects from the difficulty-eligible `agent` class, allows repeated choices,
uses a caller-supplied float budget, and stops at budget failure or 15 accepted
nouns. No per-entry weights or 1-1 caller budget were recovered.

The candidate set for horde actors and final-arena adds is therefore:

| Effective difficulty | Authored `agent` candidates |
| --- | --- |
| `1-24` | `ZelemBasicRanged.Noun`, `ZelemBasicHybrid.Noun`, `ZelemBasicRepair.Noun` |
| `25-48` | `ZelemBasicRanged_2.Noun`, `ZelemBasicPackMelee_2.Noun`, `CitadelSpecificThree_2.Noun` |
| `49-72` | `ZelemBasicRanged_3.Noun`, `ZelemBasicRepair_3.Noun`, `Sloth_3.Noun` |

**Fallback:** filter by effective difficulty, deterministically shuffle from
the match seed and encounter key, and exhaust all eligible candidates before
allowing a repeat. This preserves authored eligibility while avoiding a
single unlucky repeated noun in a small placeholder wave. Do not infer noun
weights, mutations, or captain status from the walkthrough.

The native cost at noun-data `+0x24` and the retail caller budget are not yet
available through the runtime catalog. Until they are, the budgets below use
**normalized actor slots**: one ordinary `agent` costs one slot. These are
explicit fallback caps, not the native kind-5 float budget and not marker-set
weights.

### State machine and predicates

All three encounters use this internal state machine:

```text
dormant -> arming -> activeWave -> interWave -> activeWave
                               \-> clearing -> complete
```

- `dormant`: trigger is available; gates are open.
- `arming`: admission is reserved; gates/blockers are active; the spawn
  transaction is not yet committed.
- `activeWave`: at least one wave has been committed. The encounter owns a set
  of planned, live, defeated, and authoritatively despawned actor IDs.
- `interWave`: the prior wave is terminal and a later planned wave is pending.
  Gates remain active.
- `clearing`: all waves are terminal and the live membership set is empty;
  completion side effects are being committed.
- `complete`: completion is latched; gates are open; no later contact, death,
  despawn, or scheduler callback may reopen the encounter.

For horde 1 and horde 2:

```text
isActive = state in {arming, activeWave, interWave, clearing}
isClear  = every planned wave is spawned or explicitly abandoned
           AND no spawn transaction or scheduled wave is pending
           AND liveEncounterActorIDs is empty
```

Only an accepted death or an authority-owned terminal despawn removes an actor
from `liveEncounterActorIDs`. Corpse cleanup does not count twice. Leash,
out-of-bounds, or failed-create handling must either return the actor to play
or record one explicit terminal despawn; an unobserved missing object cannot
silently clear the room.

For the final arena:

```text
isActive = final state in {arming, activeWave, interWave, clearing}
isClear  = leaderActorID is defeated
           AND every planned add wave is spawned or explicitly abandoned
           AND no spawn transaction or scheduled add wave is pending
           AND liveEncounterActorIDs is empty
```

**Fallback:** requiring both leader defeat and an empty authoritative encounter
set prevents boss death from completing while adds remain. The walkthrough
shows a named large-health-bar leader with adds and completion after the leader
dies, but it cannot prove the exact compound predicate.

## Horde 1 policy

### Authored map

| Role | Marker evidence |
| --- | --- |
| Entry trigger | row `128245`, authored ID `2244983668`, `SpawnPoint_HordeTrigger.Noun-1`, `(592.4940,11.3251,10.0210)`; radius `15`; once-only; `horde triggered` on enter and raw `horde complete` on exit |
| Listener/spawn markers | row `128241`, ID `3319475621`, ordinal `4`; row `128244`, ID `3319475618`, ordinal `7`; row `128247`, ID `150361665`, ordinal `10` |
| Gate teleporters | rows `128237`, `128239`, and `128240`, each with `HordeGateTeleporter_OnEnter` contact behavior |
| Adjacent blockers | rows `128242`, `128243`, and `128246`, ordinals `5`, `6`, and `9`, all `TestDoor_design_blockin_horde_open.Noun` variants |

**Proof:** the entry radius and once-only flag are normalized event values;
the three listeners are `horde triggered -> HordeSpawner_Register`. The gate
asset maps `horde triggered` to activation and `horde complete` to
deactivation. Its modifier repels/teleports an entrant while active.

### Recommended wave

Use one wave with a normalized budget of **3 actor slots** and a maximum
concurrency of **3** for one player. Spawn one difficulty-eligible `agent` at
each listener marker in ordinal order. Add one slot per additional active
player, capped at **6** total; distribute extra actors round-robin across the
three markers with collision-safe local offsets.

This is a **fallback**. Three listener markers prove three candidate loci, not
three enemies or one wave. One compact wave is the least speculative playable
interpretation. The walkthrough shows repeated compact traversal arenas, but
its visible counts and difficulty do not set this budget.

### Transitions and completion

On accepted radius-15 entry, reserve `arming`, activate all three horde gates
and their adjacent blockers, build the wave, publish all accepted actors, set
internal horde-active state, and commit the trigger. Transition directly to
`activeWave`; there is no fabricated inter-wave delay.

Implemented fallback: admission creates the exact three gate and three blocker
nouns with stable server object IDs and authored transforms, visibility, and
collision flags. Scoped completion deletes those six IDs. The object roster is
exact content; create/delete synchronization remains the documented fallback
because the retail server packet sequence is absent.

When `isClear` becomes true, atomically transition through `clearing`, publish
one scoped `horde complete`, deactivate all three gates/blockers, clear
horde-active state, and latch `complete`.

## Horde 2 policy

### Authored map

| Role | Marker evidence |
| --- | --- |
| Entry trigger | row `128253`, authored ID `2142467792`, `SpawnPoint_HordeTrigger.Noun-2`, `(-533.6099,561.4774,0.1342)`; radius `5`; once-only; `horde triggered` on enter and raw `horde complete` on exit |
| Listener/spawn markers | row `128251`, ID `1408208687`, ordinal `3`; row `128252`, ID `1067966471`, ordinal `4` |
| Gate teleporters | rows `128254`, `128256`, and `128257`; row `128254` also has an authored `horde triggered` edge |
| Adjacent blockers | rows `128248`, `128249`, and `128250`, ordinals `0`, `1`, and `2`, all `TestDoor_design_blockin_horde_open.Noun` variants |

### Recommended waves

Use **two waves** for one player:

| Wave | Normalized budget | Placement |
| ---: | ---: | --- |
| 1 | `2` slots | one agent at each listener marker |
| 2 | `3` slots | one agent at each marker, then one extra at the ordinal-3 marker with a collision-safe offset |

Add one slot to each wave per additional active player, with per-wave caps of
`4` and `6`. Begin wave 2 only after wave 1 is terminal; use a **1.5-second**
server-clock telegraph/inter-wave delay and keep gates active throughout.

This is a **fallback**. Set ordinal `39` after `38`, the `_2` suffix, and the
walkthrough's visibly denser late traversal support rising intensity, but do
not prove route order, two waves, these counts, or this delay. The marker-set
weight `2` is not used as a wave count or multiplier.

### Transitions and completion

On accepted radius-5 entry, use the same reservation and gate transaction as
horde 1. `activeWave(1) -> interWave -> activeWave(2)` is driven only by
authoritative wave membership. After wave 2 satisfies `isClear`, publish one
scoped completion, deactivate all three gates/blockers, clear horde-active
state, and latch `complete`.

**Fallback route prerequisite:** horde 2 admission requires horde 1 to be
`complete`. The content suffix/ordinal and the walkthrough's linear island
route support this ordering, but the normalized target IDs do not prove it.

## Final-arena policy

### Authored and walkthrough map

| Role | Evidence |
| --- | --- |
| Presentation trigger | row `126383`, ID `3299474850`, radius-60 once-only `TriggerZone.Noun`; runs `nTutorial_SoloSupportUnlockClient.main` |
| Boss/listener anchor | row `126389`, ID `223774364`, `SpawnPoint_DirectorBoss.Noun-876848874`, `(948.3736,674.0089,0.0880)`; listens to `boss triggered` and runs server support-unlock chunk `62` |
| Add listener markers | rows `126384`-`126387`, authored IDs `2145860737`, `2145860735`, `2145860738`, and `2145860736`; all `boss triggered -> HordeSpawner_Register` |
| Exit anchor | row `126388`, ID `258231375`, `LevelExitPoint`, `(941.0225,672.3367,0.1636)` |
| Arrival candidate | design row `125479`, ID `1131366622`, `TeleporterSpawnPoint`, `(928.1883,668.9045,0.0880)` |

**Walkthrough:** the support-unlock presentation is visible near the final
transfer, the chamber visibly arms, a named large-health-bar `Illust the
Accelerator` fights with adds, and the results flow follows its defeat. The
edited time gap is not an exact spawn delay. The clearest extracted references
are `frame-770.png` (support unlock), `frame-778.png` and `frame-790.png`
(arming/start), `frame-840.png` (leader and adds), and `frame-870.png`
(post-defeat drops).

**Proof:** the boss director array is empty. The four horde listeners use the
`agent` pool, while the boss marker is kind `9`. Neither content nor native
code identifies Illust's internal noun or a server boss constructor.

### Recommended leader and add waves

Create one explicit leader role at row `126389` and label it `Illust the
Accelerator` for presentation. Until its noun is recovered, use the
difficulty-eligible `special` entry as its combat-profile proxy:

| Effective difficulty | Fallback leader proxy |
| --- | --- |
| `1-24` | `ZelemSpecialHaster_Captain.Noun` |
| `25-48` | `ZelemSpecialOne_Captain_2.Noun` |
| `49-72` | `ZelemSpecialTwo_Captain_3.Noun` |

The low-band Haster proxy is consistent with the walkthrough's visible
Accelerator/Swift presentation, but this is still **fallback**, not a recovered
noun or mutation assignment. Do not roll the walkthrough's `Swift Aura,
Swift` as a guaranteed mutation.

Use one leader plus two add waves for one player:

| Phase | Normalized budget | Rule |
| --- | ---: | --- |
| Leader | `4` slots | one leader at row `126389`; the four-slot cost is local accounting only |
| Add wave 1 | `4` slots | one difficulty-eligible agent at each of rows `126384`-`126387` |
| Add wave 2 | `2` slots | after add wave 1 is terminal and leader HP is at or below 60%, use the first two listener markers in ordinal order |

For each additional active player, add one slot to each add wave, capped at
`8` and `4`. Never exceed the native kind-5 ceiling of 15 add actors in one
selection. The wave counts, slot budgets, HP threshold, proxy noun, scaling,
and marker reuse are all **fallback**. The walkthrough proves only the broad
leader-plus-adds shape.

### Activation and director-state transitions

**Fallback activation:** require horde 2 complete and a successful arrival at
the final-arena destination. The first authoritative movement segment that
intersects the radius-60 presentation trigger reserves the final encounter and
publishes one marker-set-scoped `boss triggered`. Spawn authority does not wait
for or depend on the first-clear support-unlock Lua job; replays that skip the
unlock must still work. Use a **2-second** server-clock arming telegraph before
leader/add publication, solely as placeholder presentation timing.

Use the recovered director fields as follows:

| Boundary | Proposed reflected/internal state |
| --- | --- |
| Dormant | `mbBossSpawned=false`, `mbBossHorde=false`, `mbHordeSpawned=false`, no boss ID, zero internal active waves |
| Leader and add wave 1 committed | `mbBossSpawned=true`, `mbBossHorde=true`, `mbHordeSpawned=true`, `mBossId=leaderActorID`; one internal active add wave |
| Waiting for HP threshold | boss fields retained; `mbHordeSpawned=true`; zero internal active add waves but the final encounter remains active |
| Add wave 2 active | boss fields retained; `mbHordeSpawned=true`; one internal active add wave |
| Clearing | prevent new abilities/spawns and freeze membership; retain boss ID for the terminal snapshot |
| Complete | `mbHordeSpawned=false`, zero internal active waves, `mbBossComplete=true`; retain the terminal boss ID until match teardown |

**Proof:** the field identities and `mbBossComplete` Beam Out gate are exact.
**Fallback:** every earlier field transition and the retained terminal boss ID
are proposed policy. `mActiveHordeWaves` is maintained internally but should
not be serialized until its four-byte element semantics are recovered.

After final `isClear`, commit `mbBossComplete=true` exactly once. That fact,
not the exit marker, objective state, visible drops, or client exit event,
authorizes the normal Beam Out UI. Reward and chain transactions remain
outside this encounter policy.

## Idempotency, replay, and race rules

Completion is one compare-and-swap style transaction on the encounter epoch:

1. Re-evaluate the complete predicate under the encounter lock.
2. Change `activeWave` or `interWave` to `clearing`; any other source state is
   a no-op.
3. Freeze new spawn and ability admission for encounter-owned actors.
4. Commit the terminal director snapshot and one completion outbox record
   keyed by `(match epoch, marker-set ID, completion)`.
5. Apply gate/blocker deactivation from that same record.
6. Change `clearing -> complete` only after the durable/in-memory operation
   required by the session owner succeeds.

Duplicate deaths, simultaneous last deaths, delayed projectiles, repeated
contacts, duplicate client events, stale scheduled callbacks, and replayed
completion work return the already-committed snapshot without spawning,
opening twice, awarding twice, or emitting another named event. A completion
publication failure remains retryable from the retained outbox record; it does
not roll the encounter back to active combat.

Late join and reconnect receive the current encounter snapshot: trigger latch,
state, gate/blocker state, planned wave ordinal, live actor IDs and baselines,
leader ID/HP when applicable, and the latest director fields. They do not
replay entry contacts or completion side effects. A new match epoch resets all
three encounters; ordinary movement or session reconnect does not.

Success wins a same-tick race only after the final clear transaction reserves
`clearing`; otherwise an already-committed mission failure remains terminal.
The wider 1-1 failure predicate is unresolved and is not defined here.

## Evidence and replacement queue

The proposal is grounded in:

- `notes/campaign/1-1/route.md` for marker-set topology, gate rows, and route anchors;
- `notes/campaign/1-1/director.md` and `notes/campaign/1-1/budget.md` for kind-5 candidates,
  native selection, listener fan-out, and missing budget authority;
- `notes/campaign/1-1/native.md` and `notes/campaign/1-1/completion.md` for director fields,
  final completion, gates, Beam Out, and missing server boundaries;
- `notes/campaign/1-1/nouns.md` for the candidate noun families;
- `bin/video/walkthrough/1-1/1-1.mkv`, the extracted `frame-*.png` and
  `contact-*.png` images, and `bin/video/walkthrough/1-1/info.md` for the
  separately labelled behavioral observations.

Replace the fallback values when a retail authority trace, server callback
implementation, effective `challengeOverride`/`waveOverride`, noun director
costs plus caller budgets, exact boss noun, or multiplayer encounter trace is
recovered. Do not tune the policy to reproduce the edited timestamps, one
run's counts/mutations/difficulty, or visible reward rolls.
