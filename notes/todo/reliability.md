# Combat reliability

## Goal

Make every accepted combat action terminate deterministically without leaving the player, NPC, or zone busy.

## Evidence

- NPC attacks fail when scheduled producers are submitted out of chronological order.
- Failed pursuits can leave NPC action state invalid or active.
- Held and repeated basics can re-enter an incompatible runtime shape.
- Later requests are rejected as session unavailable or busy, producing the observed run-ending soft lock.
- The 2026-08-12 run emitted an identical companion 0x91/0x95 pursuit pair thousands of times in one second after a sub-frame range correction, filling the 4,096-datagram RakNet retransmission window.
- The same run rejected Plasma Sentinel pursuit arrival because the shared companion encoder only admitted Sage's basic by name, and rejected navigation for valid NPCs whose authored footprint is zero.
- Action admission previously accepted any non-empty handler output, so effect-only packets could omit the matching 0x9b response while delayed producers continued mutating a client-visible rejected action.
- Canceling a scheduled pickup previously stopped only its timer; the tracked schedule and zone reservation survived, leaving the object unavailable and `/reset` unable to recover it.
- NPC first-action admission could mark several actors started before planning or scheduled publication failed; those actors then rejected later starts while performing no action.
- Pursuit corrections did not verify the action owner, so a canceled or replaced NPC action could continue moving from an old scheduled callback.
- Squad switching committed the new deployed hero and cooldown before several packet encodes and optional schedules; a later failure returned rejection after the server had already switched, splitting client and server hero ownership.
- Hero damage committed squad and zone health before death, selection, game-over, or cleanup packets were encoded; any later error could leave the client presenting a living hero that the server had already killed.
- `/reset` canceled authoritative schedules before encoding its recovery baseline, then allowed optional summon or movement cleanup errors to prevent the baseline from ever reaching the client.
- Action scheduling helpers assumed the transport always exposed failure-reporting schedule groups; older schedule-only transports could hit a nil callback instead of admitting work under watchdog recovery.
- Player pursuit silently canceled when its target disappeared and sent no terminal action response, leaving the client latched until the general watchdog expired.
- Shared action recovery cleared basic admission but left specialized ability-run fields active, so the client could recover while the next server request still failed as already active.
- NPC pursuit and first-action callbacks used unconditional resets on failure, allowing a stale callback to erase a newer action owned by the same NPC after retargeting.
- Beast Pet, Sage passive companions, and general summons could remain registered as active when teardown presentation encoding failed after only part of their authoritative state was removed.
- Admitted movement cleared every action watchdog, including unrelated abilities, while an interrupted basic had no exact lease identity of its own.
- Equipment and catalyst pickups completed world/inventory work after acceptance without publishing a released response; full equipment inventory also ended the delayed step without any terminal response.
- Interactable use committed its one-shot script and objective state before proving that its delayed schedule could be tracked for cancellation and reset.
- Plasma Sentinel Active published an accepted response with an immediate end time but no released response, causing its recovery watchdog to reset an otherwise healthy session ten seconds later.
- Unexpected handler failures recovered action state only when delayed work had already been scheduled, so an error after admission but before the first schedule could leave the server busy.
- Client ActionCancel cleared every recovery lease without stopping unscheduled basic admission or interruptible ability runs, removing the watchdog while retaining state that blocked later actions; committed basic swings now retain their scheduled hit while release cancels only held repetition.
- A delayed player-action failure released every NPC action owned by the player's transport even though NPC schedules already perform exact owner-fenced cleanup, allowing an unrelated ability failure to stop the encounter.
- A producer racing rejected admission could create nested delayed work after the admission schedule set was sealed; the new schedule was no longer tracked and survived cancellation.
- Player pursuit arrival, target loss, and timeout could end the action without stopping authoritative movement or publishing the final stopped position, leaving the client and simulation on different movement runs.
- The 0.7.5 moving-target traces reached the scheduled melee stop radius, then lost admission when the target advanced during the callback boundary, repeatedly transferring the same pursuit while the client remained busy; pursuit retries now retain one redirect quantum of contact tolerance.
- The 0.7.5 replacement traces showed a defeated client body continuing to its visible resting pose while rejected movement left the server at an older Rooted position, then transferred that Rooted expiry to the replacement hero; selection-pending pose reports now reconcile the death location and hero-specific enemy control state ends at defeat or switch.
- Several NPC combat-family entry points returned silently when their target, source, or decoded action profile disappeared while retaining `IsActionStarted`; that NPC then rejected every later action as already active.
- A delayed NPC hit, projectile, or lob callback could fail after its schedule group had started; RakNet then canceled the remaining continuation, including the callback that normally begins the next action, while the NPC retained action ownership forever.
- Optional post-hit work such as stat persistence, reflection presentation, forced movement, poison, silence, fear, and vulnerability encoding was allowed to abort an already-committed attack and suppress its continuation.
- Specialized healer, buffer, repair, corpse-consume, Polaris, and charge callbacks could fail inside nested schedules while retaining NPC action ownership and transient run state.
- Citadel repair, resurrection, and pack-cower callbacks could stop after a scheduling or presentation error while leaving their source or newly revived NPC permanently busy.
- Scaldron Blink, leap, and directional-shield controllers had terminal destination, presentation, and schedule branches that retained their action or shield run.
- Exploder Scarab, Voltroid, and Mana Drain could commit damage, charge, or resource state and then abort on optional presentation, stat, modifier, or continuation work.
- A terminal zone could discard its checkpoint while an older safe-boundary capture was still assembling, allowing the late capture to recreate a stale Continue entry.
- A checkpoint save already executing against SQLite could finish after launcher Start Fresh deleted the row, recreating the retired mission despite its in-memory tombstone.
- Darkspinner Start Fresh deleted durable and Blaze state but left the live gameplay zone and retained rejoin sessions in memory, while a restored member's setup failure removed the entire shared game instead of only rolling back that member.
- Tutorial checkpoints use authored difficulty zero, but the restored Blaze shell rejected every difficulty-zero checkpoint before identifying tutorial mode; checkpoint validation also admitted duplicate or invalid membership and squad structure too far into restoration.

## Fix order

1. [x] Normalize every scheduled producer group into chronological order at RakNet admission while preserving authored order for equal deadlines.
2. [x] Give accepted actions exactly one completed, cancelled, or failed terminal transition. Every accepted source now has a matching release, delayed publication failures release only their exact generation-scoped admission, and rejected-admission races cannot retain nested work.
3. [x] Release attack, pursuit, held-basic, movement, squad, and interaction locks on every failure path. Player pursuit and NPC scheduling now roll back exact owned state, stale pursuit corrections reject mismatched owners, squad and death transitions pre-encode required output, rejected admission cancels its pre-response schedules, and canceled pickup work releases its tracked zone reservation.
4. [x] Send the client a terminal response when an accepted action cannot finish. Immediate output must contain a matching 0x9b response; delayed failure schedules an exact-lease rejection and retains the watchdog as fallback.
5. [x] Prevent invalid pursuit and converge moving-target pursuit without cancellation loops. Stationary NPCs wait for range; companion attacks use a range tolerance and a 50 ms minimum pursuit cadence.
6. [x] Make /reset use the same authoritative cleanup boundary instead of feature-specific best-effort clearing. Reset pre-encodes its recovery baseline, quiesces every tracked presentation run, releases player-owned NPC actions and interactions, clears hostile status gates, then resynchronizes the hero and reacquires targets.
7. [x] Admit authored melee companion abilities generically and normalize zero-footprint NPC navigation at the zone boundary.
8. [x] Pre-encode the core squad-switch response and never reject an already-committed switch because an optional passive, companion, presentation, or NPC restart failed.
9. [x] Pre-encode lethal resource, Beam Out, reserve-switch, and game-over output before health commits; make post-death effect and target cleanup best-effort so presentation teardown cannot invalidate authoritative death.
10. [x] Fall back from failure-reporting schedule groups to ordinary schedule groups when necessary, retaining cancellation cleanup and the action watchdog instead of invoking a missing transport callback.
11. [x] Reject pursuit immediately when its moving target disappears, stop authoritative movement, and guard delayed pursuit callbacks against a removed zone.
12. [x] Make failed-action and watchdog recovery stop and clear specialized projectile, area, channel, charge, summon, pet, modifier, and passive runs instead of releasing only basic attack admission.
13. [x] Replace unconditional NPC action resets in first-action, pursuit, and drain failure paths with exact owner-fenced releases so stale callbacks cannot cancel replacement actions.
14. [x] Clear pet, companion, and summon ownership before generating teardown presentation so an encoding failure cannot retain an already-removed summon as active.
15. [x] Track the exact basic-action sync stamp and pursuit lease kind so movement clears only the action it supersedes instead of disabling recovery for every active ability.
16. [x] Complete equipment and catalyst pickups with exact released responses, reject capacity failures explicitly, and track interactable schedules before committing one-shot use state.
17. [x] Complete Plasma Sentinel Active with an immediate released response while retaining its independent shield-duration schedule.
18. [x] Recover every unexpected admitted-action failure, including failures before the first delayed schedule, and publish rejection plus an authoritative movement, resource, and control baseline.
19. [x] Make ActionCancel stop unscheduled basic admission, pursuit, dance, channel, blink, and charge state before removing its recovery leases while allowing a transport-scheduled basic swing to finish its authored hit and release.
20. [x] Keep player-action schedule failures from releasing unrelated NPC combat; NPC producers retain exact owner-fenced failure cleanup.
21. [x] Cancel delayed work created concurrently after rejected admission while allowing nested schedules to continue after a successfully completed admission.
22. [x] Stop and republish player movement whenever pursuit arrives, loses its target, or times out so the next command starts from one authoritative position.
23. [x] Release NPC action ownership on terminal source, target, and profile loss across melee, charge, drain, detonation, lob, blink, leap, flee, stealth, resurrection, and Polaris action families.
24. [x] Give common NPC hit, plunge, meteor, projectile, and lob timelines named failure boundaries that retire transient objects and release action ownership when a delayed producer aborts.
25. [x] Treat optional post-hit stats, reflection, forced movement, and status presentation as best-effort after authoritative damage commits so these omissions cannot stop the NPC combat loop.
26. [x] Give healer, buffer, repair, corpse-consume, Polaris, and charge timelines named failure boundaries that clear transient state and release exact NPC action ownership.
27. [x] Make Citadel repair, resurrection, and pack-cower failures recover their source actions, and release newly revived NPCs when their first action cannot be scheduled.
28. [x] Recover Scaldron Blink and leap actions on terminal movement or scheduling failures, while keeping committed leap damage alive when optional landing presentation or stat persistence fails.
29. [x] Make directional-shield teardown resume or release its NPC even when the effect slot or stop presentation is unavailable, and recover every setup and pursuit failure.
30. [x] Recover Exploder Scarab, Voltroid, and Mana Drain timelines after failed continuations while treating post-commit effects, modifiers, and stat persistence as best-effort.
31. [x] Fence checkpoint enqueue against terminal zone state and prevent stale SQLite revisions from replacing a newer member-to-zone resume index.
32. [x] Commit hostile co-op damage through the target member's own session and bounded transport queue so remote defenses, resources, death selection, reactions, and stats cannot diverge from shared zone health.
33. [x] Keep accepted equipment and health obelisk use progressing when an optional packaged-objective callback fails instead of canceling the interaction after its delayed work was admitted.
34. [x] Retain co-op outbox batches until RakNet response commit and proactively poll connected peers so a full retransmission window or an idle remote client cannot silently lose target-owned state.
35. [x] Tombstone discarded checkpoints before storage I/O, reject stale captures for the retired zone ID, and make launcher Start Fresh wait for durable deletion so a crash or delayed capture cannot resurrect the mission.
36. [x] Serialize checkpoint store operations and recheck the discard tombstone inside the storage fence so an in-flight save cannot commit after Start Fresh wins.
37. [x] Route Start Fresh through gameplay-zone retirement and retained-session cleanup, and isolate restored Blaze setup rollback to the reconnecting member so healthy co-op participants retain their shared game.
38. [x] Admit only difficulty-zero, solo tutorial checkpoints and validate durable zone identity, reason, member slots, finite squad positions, and restorable squad state before constructing the live zone.
39. [x] Validate saved hero identities, object blocks, active squad indices, finite poses, and bounded resources, then restore exact active-hero health, mana, position, and stealth instead of silently discarding the hero record.
40. [x] Keep shared co-op encounter completion separate from each member's result commit so one player entering results cannot close the zone or reject another player's result transaction.
41. [x] Normalize a restart checkpoint whose deployed hero is dead to the first living available squad member instead of restoring directly into an unplayable selection state.
42. [x] Persist committed security-route presentation and progress so a cleared floor resumes with its teleporter active, and exclude transitional horde or boss clears from safe checkpoint creation.
43. [x] Reject negative elapsed time, duplicate or unknown crystal owners, malformed crystal slots, and duplicate or zero cleared spawn groups before a checkpoint can create live authority.
44. [x] Reject checkpoint capture while a horde wave, inter-wave transition, or boss transaction is active, and validate every captured snapshot before it can enter the persistence queue.
45. [x] Restore the complete co-op roster as disconnected zone, outcome, and vote participants without persisting process-local peer generations, retaining absent members' hero state across later safe saves.
46. [x] Reject over-capacity restored rosters, duplicate objective records, invalid medal states, inactive durable member slots, and active non-member slots before restored co-op objective completion becomes authoritative.
47. [x] Persist the per-NPC, per-member earned-XP ledger and a stable zone completion identity, then make account XP idempotent through a hidden durable result receipt so a restart cannot lose or duplicate completion progression.
48. [x] Keep the shared pre-result checkpoint until every retained co-op participant commits a result, preventing the first member's Beam Out from deleting recovery for the remaining roster.
49. [x] Rebind outcome, result-ledger, and vote ownership to replacement peer generations without resetting a committed result or an already-cast shared vote.
50. [x] Retain pursuit ownership and reschedule correction while an NPC's effective slow scale leaves it unable to move, and consume observed client-only level-object callbacks once so movement cannot recreate action or director-publication loops.

## Completion

- No action failure leaves a player or NPC busy.
- A failed scheduled step does not block later movement, attacks, abilities, or squad switching.
- Human playtesting can continue after forced action, pursuit, and scheduling failures without aborting the mission.
