# NPC spawn and attack facing

## Implementation — 2026-09-04

The server now uses captured-point turns (0x42 with root goal, target point, and target ID zero), and tracking projectiles use object-facing turns (0x102). AttackStart no longer uses the inert boss inversions or appends an unreliable locomotion packet to a turn. Charge cleanup uses explicit-direction flags 0x06. Self-target and coincident casts keep their heading.

Arrival, cancellation, blink, forced relocation, and growth preserve orientation through position reflection plus explicit stop semantics; these NPC adapters no longer send a zero-quaternion teleport. This also deliberately preserves the target's existing client quaternion for hero relocation instead of reconstructing an unknown orientation. Native teleport/interpolation behavior still needs the real-client acceptance pass below.

Enemy creation retains the Euler prefix instead of overwriting it with identity. Owned summons use that same rotation input. Population decisions and horde/boss listeners retain exact spawn-marker rotations; population regrouping resolves final positions back to spawn markers, excluding unrelated camera/light markers. Additional generated positions use the documented zero-Euler fallback. Snapshots initialize forward from the native Euler/quaternion convention, and restore encodes saved facing in the create prefix before publishing an explicit heading.

Action profiles carry known/suppressed facing policy from Lua, with recovered named overrides for static server profiles. First-aggro activation can turn without an animation, while robot activation and delayed scripted reveals retain pose. Shared attack callers now commit facing with the source's action generation and owner checks. Target acquisition and arbitrary SetPosition calls no longer silently turn feature state. Pursuit records target-facing for 0x41 commands and movement heading for 0x01 projectile pursuit/fleeing; strafe records target-facing. Idle leaves retain the existing 0.1–0.6 second wait and turn only beyond 30 degrees when eligible.

NPC projectile activation suppresses only the simulator's caster locomotion intent, preserving its animation and projectile behavior. A separate root/selected-target turn precedes activation; tracking volleys turn on later shots even when animation is suppressed. Projectile muzzle offsets, leading, spread, trajectory, and collision inputs remain independent.

Co-op audit confirmed gameplayProducerObserver forwards committed source presentation packets through queuePeerPresentation. The two arrival publishers no longer additionally reconstruct a generic ActionEventAttack; peers receive the actual specialized cast sequence. Existing first-aggro semantic projection remains compatible with packet deduplication.

Validation: targeted production compilation, `mage build`, and `git diff --check` passed. Visual acceptance status is tracked in [the implementation checklist](../todo/npc-facing.md). No tests were created, modified, or run. The original research below is retained as the pre-fix evidence record.

Historical research completed 2026-09-04, before the implementation described above. The ordered work list is in [NPC facing action items](../todo/npc-facing.md).

## Findings

There are several independent failures in the shared NPC path. They explain why a mob can start in the wrong orientation, appear to walk correctly, then stop turning or snap to a different orientation when attacking. This is not confined to the boss animations that previously received facing sign changes.

| Finding | Evidence | Consequence |
| --- | --- | --- |
| Shared `AttackStart` sends flags `0x20`, a facing vector, no target position, and an extra `0x95`. | Current Go and native facing consumer. | `0x20` does not enable either facing mode; the supplied vector is ignored. There are 34 production calls to this helper in `server/gameplay` at inspection time. |
| Arrival calls `MovementStop`, which sends `0x90` with quaternion `(0,0,0,0)`. | Current Go, native teleport receiver, and native transform setter. | Arriving from pursuit overwrites the orientation immediately before the ineffective attack turn. A zero quaternion is actually installed; it does not mean “preserve orientation.” |
| Enemy creation appends identity quaternion `(0,0,0,1)` after the authored Euler rotation. | Current encoder and native create/reflection order. | Even fixed tutorial actors whose spawn plans preserve rotation lose that rotation in the same create message. |
| Ordinary population, horde, and boss planners generally carry positions without rotation. | Current planning/publication types and runtime marker catalog. | Procedural spawns use the default orientation; authored spawn-marker rotation is unavailable downstream. The retired server's exact procedural-facing choice is not established merely by finding a rotated marker. |
| Server `Snapshot.Facing`, spawn `Plan.Rotation`, and client locomotion orientation are separate and inconsistently updated. | Current NPC session, pursuit, snapshot, and restored-spawn code. | Rejoin loses saved facing; server directional logic can disagree with what is rendered. |
| Ranged casts and repeated shots use a different activation path. | Current projectile and simulation code. | Some mobs already receive a real turn on their first shot, while subsequent shots suppress it; a change only to `AttackStart` is insufficient. |

These are source/content findings, not a fresh live reproduction of the user's particular mobs. No game, launcher, server, or Fang implementation was changed, and no new live session was launched. Existing notes corroborate earlier downward-facing mobs, but their historical claims of successful facing fixes must not be substituted for current client observation. Tests were not run because repository policy prohibits them without an explicit request.

## Native contract: why the attack vector has no effect

Canonical source: `bin/game/GameBin/Game.c`, build 103. Focused read-only function copies are under `bin/game/logs/mob-facing-2026-09-04`.

`sub_5393D0` (`0x005393D0`) accepts `0x91` for any eligible object with locomotion. It copies flags to component `+0x144`, facing to `+0x178`, target position to `+0x1AC`, and target identity to `+0x90`. Receiving a nonzero vector does not itself rotate an NPC.

The decisive consumer is `sub_9E6400` (`0x009E6400`), called by mover update `sub_9EA1A0` (`0x009EA1A0`):

1. If the command has target-facing bit `0x40` or object-facing bit `0x100`, has a resolvable target ID, and its transient state permits it, use that object's position.
2. Otherwise, with bit `0x40`, use the command's target-position vector.
3. Otherwise, with bit `0x04`, use the command's facing vector.
4. Otherwise, clear any previous explicit facing mode when necessary.

`AttackStart` has only `0x20`, which takes none of the first three branches. Changing the sign of its facing vector cannot repair that command. Adding only `0x40` without filling target position would instead make it face the zero/default target point.

The native constructors distinguish three operations:

| Operation | Constructor | Flags | Target identity | Target point |
| --- | --- | --- | --- | --- |
| Stop locomotion | `sub_A14D70`, `0x00A14D70` | `0x20` | Cleared | Cleared; facing also cleared |
| Turn toward a captured position | `sub_A15610`, `0x00A15610` | `0x42` | Zero | The intended world position |
| Turn toward an object | `sub_A157A0`, `0x00A157A0` | `0x102` | The target object | Separate target-position field remains cleared |

Both turn constructors retain the current actor position as the stationary goal and compute normalized target-minus-actor facing. The object-facing constructor also sets the stop distances from the actor footprint. Do not conflate object tracking with the ordinary cast's captured point by adding a target ID to every `0x42` command.

Native ability activation `sub_9E0AE0` (`0x009E0AE0`) checks the ability's turn-on-create flag at definition `+0x15C`. For an applicable external target, it calls `sub_A15610`, then `sub_A20210` (`0x00A20210`) before running the Lua ability. When its send gates pass, `sub_A20210` sends the fixed 80-byte locomotion body, logical message `0x12` / wire `0x91`. This activation does not call the `0x95` sender. Self-target and other native eligibility checks remain relevant; “every animation should face the hero” is not the contract.

**Correction to earlier research:** the canonical `sub_A20210` has both a role guard and a `sub_9E4970(actor)` guard. `sub_9E4970` reads actor byte `+92`, the player-controlled flag. Therefore its invocation by an NPC-capable Lua binding is not proof that this helper directly sends ordinary NPC packets in the retained binary. The exact ordinary retail NPC dirty-flush sender remains unproven here. What is proven independently is the NPC turn constructor, the unrestricted NPC receiver path, and the mover's flag consumption. One `0x91` carrying the recovered turn state is a supported server implementation; earlier notes overstate the direct NPC-sender evidence. Do not claim a recovered retail NPC packet cadence from this function.

The useful ordinary attack command is:

```text
0x91
objectID       = caster
goalFlags      = 0x42
goalPosition   = current actor root position
facing         = normalize(captured target position - actor root position)
targetPosition = captured target position
targetObjectID = 0
then the authored animation
```

External force and stop-distance retention should be handled consistently with existing feature state; the native constructor does not reset every field. Preserve the current ordered transport. There is no need for a new opcode, a client patch, or an animation-name-dependent direction convention.

See also [melee activation](melee-arc.md#exact-turn-in-place-command) and [NPC locomotion](../campaign/npc-client-locomotion.md). The latter's broad reference to an “applicable target identity” needs the more precise point-versus-object distinction above when implementing this fix.

## Spawn orientation is lost twice

### Planning and admission

`content/sqlite/level.go` reads marker Euler components from `+0x28/+0x2C/+0x30`. `server/game/contentsqlite/director.go` preserves them in `CampaignDirectorMarker.Rotation`.

The subsequent paths diverge:

- `server/zone/npc/fixture.go:planMarkers` preserves actual actor placement rotation, including fixed tutorial actors.
- `server/zone/population/population.go:candidate/Decision` retains collections of positions without corresponding transforms. Both ordinary and authored-noun plan construction in `population/plan.go` leave `SpawnPlan.Rotation` zero.
- `server/game/campaign_director.go:CampaignDirectorListenerPublication` retains position but has no rotation. Horde and named-boss planners therefore receive incomplete placement transforms. `horde/plan.go`, `boss/initial.go`, and `boss/generic.go` produce zero-rotation plans.
- Summon/developer paths generally construct plans with positions alone. `TargetedSpawn` receives a target ID but no target position; its blackboard update cannot orient the create baseline by itself.

Read-only `darkrun db marker get` inventory, paginated by `id` across the current runtime content database:

| Spawn-marker noun suffix | Marker rows | Rows with absolute Z rotation greater than 0.01 degrees |
| --- | ---: | ---: |
| `DirectorWanderer` | 11,724 | 4,233 |
| `DirectorSpike` | 1,313 | 124 |
| `DirectorHorde` | 393 | 65 |
| `DirectorBoss` | 33 | 8 |

These are catalog rows, including authored level variations, not a count of affected live mobs. Examples include boss marker row `1873` in `cryos_1_design_spawners.Markerset` with Z `78.4031677`, row `9156` in `cryos_3_design.Markerset` with Z `36.9993820`, and row `14835` in `cryos_4_design.Markerset` with Z `151.9974976`. Horde marker row `2710` in `cryos_1_AI_Horde_2.Markerset` has Z `70.2763062`.

Those rotations prove authored information is available and discarded. They do not prove that every procedural escort should inherit the director marker's rotation. Preserve exact actor placement transforms; resolve the policy for generated groups separately. Camera, light, portal, and other unrelated marker rotations must not become NPC facing.

### Wire construction

`server/zone/npc/raknet103/spawn.go:Spawn` passes plan rotation to `EnemyObjectCreateMessage`. `server/raknet/application.go:EnemyObjectCreateMessage.EncodePayload` writes:

```text
create prefix: noun, position, plan.Rotation XYZ, ...
reflection:   field 6 = position
              field 7 = quaternion (0, 0, 0, 1)
```

Native `sub_53A760` first constructs the object using `sub_9D6910` / `sub_9D1DF0`, then reads its base-object reflection through `sub_A1E710`. The constructor converts Euler degrees into the object's quaternion; the subsequent field-7 identity is a replacement, not a default used only when rotation is absent. Consequently, preserving the Euler prefix alone is insufficient. Either omit the redundant field when constructor rotation is authoritative, or encode an equivalent valid quaternion; choose one coherent transform contract.

Owned NPCs other than `NomadDrone.Noun` take `ObjectCreateMessage` instead. That DTO has no rotation/orientation input and writes a zero Euler prefix. Include that branch when fixing summoned mobs.

The later spawn `ObjectUpdateMessage` contains only position and visibility, so it does not restore orientation. `AgentBlackboardUpdateMessage` does not replace a turn command either.

## Arrival and other pose resets

`MovementStop` in `server/zone/npc/raknet103/action.go` starts with `ObjectTeleportMessage{ObjectID, Position}`. The encoder writes all four quaternion components as supplied. Omitted Go fields are zero, producing a non-unit all-zero quaternion.

Native receiver `sub_5392F0` (`0x005392F0`) stops locomotion and calls `sub_9D9690` (`0x009D9690`). The latter directly copies position and all four quaternion components into the actor and triggers the transform-change path. There is no “zero quaternion means leave it alone” condition. The precise rendered result can depend on subsequent mover/graphics processing, so this research does not claim that every zero quaternion displays as one particular compass direction.

Two high-frequency callers make this especially visible:

- `server/gameplay/npc_pursuit.go:produceStep`, on arrival: `MovementStop` followed by the attack callback.
- `server/gameplay/combat.go`, first action already in range: prepend `MovementStop` to the produced attack packets.

The effective sequence for many melee and special attacks is therefore `0x90(q=0) -> 0x8D(position/visibility) -> 0x91(stop) -> 0x95 -> 0x8D(position) -> 0x91(stop with ignored facing) -> 0x95 -> animation`.

The same omitted-orientation pattern exists in NPC adapter `Blink`, `CancelAction`, `OozeGrowthCast`, and `ChargeCleanup`. `RandomTeleport` and `ForcedMovement` affect the target rather than necessarily the caster but have the same transform defect. Position-only synchronization already has an `ObjectPositionUpdateMessage`; ordinary stopping need not teleport. Genuine teleports need a valid, intentional orientation, not a blanket replacement with identity that would still erase heading.

## Action coverage, exceptions, and tracking

### Shared stop-based attacks

`AttackStart` is used by ordinary melee, Nomad Drone, buffs, drain/channel attacks, cones, lob attacks, leaps, special poses, and several bosses. `ChargeStart` and `ChargeCleanup` separately repeat the flags-`0x20` plus facing-vector mistake. `FirstAggro`, `FirstAggroActivate`, and `StationaryCastStart` use stop semantics; whether each needs a turn depends on its authored phase.

The animation reversal list in `AttackStart` covers `sca_boss_attack*`, `zlm_boss_sp_attack3`, `boss_lf_spawneater_attack2`, `cry_el_boss_chain_lightning`, `cry_el_boss_melee_attack`, and `ctd_boss_tc_attack2_part_1`. Under the current flags these sign changes cannot drive native facing. Treat them as unproven compensation when moving to the correct contract. Any remaining visual inversion must be distinguished between the actor root and the rendered animation/model after the packet defects are fixed.

### Authored turn-on-create overrides

The current `ActionProfile` has no turn-on-create field. It cannot represent the exceptions that exist in imported Lua. The authoritative runtime DB contains 29 chunks with the explicit `faceTargetOnCreate` constant: root assignments are false in 24 and true in 5. This catalog includes hero abilities as well as NPC abilities, and absence from this explicit-override list is not proof of a default value.

Every matching chunk was extracted with `darkrun db ... bget --decode zlib` and disassembled with `darkrun lua`. Metadata, bytecode, disassembly, and an assignment index are in the diagnostic directory. Representative NPC cases:

| Chunk | Definition | Explicit assignment | Implication |
| ---: | --- | --- | --- |
| 290 | `nAbility_NomadBioSpecialTwo_JumpAttackMelee` | false, root PC 27 | Preserve the jump's established pose at its melee continuation. |
| 1015 / 288 / 737 | Nomad Shielder position / grenade / bash | false, PCs 23 / 37 / 26 | Setup and locked shield-facing behavior need their own pose ownership. |
| 179 / 4 | Citadel Specific Four position / bolt | false, PCs 23 / 41 | Do not automatically turn each shot or setup animation. |
| 568 | Scaldron Basic Maser shot | false, PC 32 | A firing animation need not turn on activation. |
| 443 | Noct Ghost Charger pose | false, PC 37 | Preserve the authored directional pose. |
| 100 | Zelem Basic Pack Melee cower | false, PC 35 | Cower is not a target-facing attack. |
| 71 / 875 | First Aggro Activate Robot / Special Robot | false, PC 10 | Preserve authored activation direction. Subsequent variants also disable turning. |
| 669 | First Aggro Anim NonFacing | false, PC 10 | Explicit non-facing entrance. |
| 922 | Scaldron Basic Blink Attack Blink | true, PC 12 | A blink-family name does not imply that turning is disabled. |

`FirstAggro_FaceTarget`, chunk `575` / resource `14137`, only registers its ability and yields once in Tick. Native activation owns its facing. A scheduler that reproduces only a visible animation cannot reproduce this leaf.

`nBehavior_CombatIdle`, chunk `539` / resource `14099`, turns to the target position when the angle is between 30 and 330 degrees, then waits `0.1 + random()*0.5` seconds. The idle branch of `produceBoundedStrafeOrIdle` currently schedules the wait without publishing that turn. This is a separate reason an otherwise stationary mob may stop following the hero with its body between attacks.

### Ranged activation is different

`server/gameplay/npc_projectile_zelem.go:produceZelemShotWithVolley` uses `NewProjectileRun -> sim.StartPoisonCloud`. When activation is not suppressed, `server/sim/poison.go` emits `LocomotionStopIntent{IsTurn:true}`; `server/sim/raknet103/encoder.go` correctly emits `0x42` plus target position. This explains partial success and why ranged bodies can recover from an arrival reset that melee bodies cannot.

However:

- `isActivationSuppressed` is true for an empty animation name or every shot after the first. That suppresses the turn as well as animation.
- `IsProjectileTrackingBetweenShots` currently changes the projectile aim-point selection, while later activation remains suppressed. It does not emit a caster tracking turn. Packaged BurstShot evidence distinguishes refresh of projectile target coordinates from the ranked `TurnToFaceTargetObject` behavior; see [ranged tutorial contract](../tutorial/horde-ranged.md).
- The input actor position is projectile source/muzzle position, and its target position can already contain spread or lead adjustments. The shared simulation uses those same inputs for the caster turn. Keep actor root/cast-facing intent separate from projectile source/trajectory intent during the fix.
- `handler.go:marshalZoneProjectionEvent` reconstructs `ActionEventAttack` using the stop-based `AttackStart` helper, rather than the ranged simulation's turn policy. The initial/in-range and pursuit-arrival publishers are confirmed callers. Inspect surrounding event delivery before claiming every observer currently receives only this helper; ensure the final implementation emits consistent semantic facing to every subscriber without duplicate or contradictory commands.

## Feature-state and restore consistency

`Session.Add`, `AddDormant`, and `AddStaged` do not initialize `Snapshot.Facing`. `AcquireTargets` and `Retarget` set it from the selected target, but ID-only `AcquireTarget` cannot. `StartAction` does not update facing. `PlanAttackWithProfile` computes an attack plan without committing a facing change.

`AdvancePursuit` updates `Plan.Position` on both navigation and straight-line branches without updating facing. Conversely, `SetPosition` changes facing to the displacement direction for every position change, including callers whose action may need a retained or target-facing pose. `FacePosition` is only explicitly used by the shielder and ghost-charge gameplay paths at inspection time; the shielder setup call changes server state without publishing a turn at that point.

This matters beyond appearance: `isShieldDamageImmune` uses `npc.Facing` for its directional dot product. A server-side facing assignment alone does not prove that the shield graphic points in the same direction.

Checkpoint `Restore` retains `snapshot.Facing`, but `marshalRestoredZoneNPCs` publishes only `npc.Plan` through `DormantSpawns`. Saved facing never reaches the create baseline. Its zero-facing fallback points from the saved position back toward the original spawn location, which is also zero if the NPC has not moved; it is not recovery of a saved orientation.

Give facing one feature-owned meaning across spawn, accepted casts, movement, intentional pose locks, teleport, and restore. Distinguish desired facing from interpolated rendered facing if needed; the server cannot claim to know the client's exact current turn progress merely because a target was selected. Keep RakNet fields and quaternion serialization in adapters.

## Spawn drift correction from the 0.7.27 report

The `theres-a-huge-desync-probleme-when-enemies-spawn-they-just-d` report captured a concrete stationary-turn regression. Enemy 43 (`NocturnaBasicRangedSilence.Noun`) remained at server root `(350.172,450.181,33.183)` through consecutive casts. At client time `18398140`, its root was `(318.723,442.574,30.088)`, while its main goal still held the spawn root, its partial goal was zero, and its flags had changed from `0x42` to `0x43`. At `18400406` it had drifted farther to `(319.559,430.441,28.342)`. These are before-apply client probes associated with the immediately preceding wire object ID, not inferred server positions.

Canonical `sub_9EA1A0` explains that combination: its remote-mover branch, gated by `sub_9BCF80`, ORs bit 1 into the flags and calls `sub_9E8700` with component `+340` (the partial goal), even for a stationary turn. `sub_5393D0` copies the main goal and facing state from `0x91` but never writes `+340`. Therefore the native local turn constructor alone is insufficient as a remote NPC replication recipe. This supersedes any earlier implication here that one `0x91` is sufficient for an NPC turn.

NPC captured turns, object-tracking turns, restored headings, and charge cleanup now prepend a reliable sparse `0x94` field-6 partial-goal update containing the stationary root before the existing `0x91`. Shared simulation cast turns use the same sequence. This preserves the complete command's transient-state reset and facing semantics while replacing zero or a previous pursuit goal. Both packets remain in the same returned presentation batch and are retained by co-op presentation filtering. No client hooks or shipped assets change. Formatting and source review were completed; builds were not run at the user's request, tests are prohibited by repository policy, and a fresh live-client replay remains necessary to confirm visual behavior.

## Implementation boundaries and remaining uncertainty

The core packet defects are proven and can be fixed without more reverse engineering. Preserve ability timing, damage, navigation, projectile collision, and installed game assets while doing so.

The procedural spawn-facing policy still needs an explicit decision where no actor transform or authored facing action answers it. A conservative proposed order is exact actor placement transform, then an applicable authored target-facing introduction using the admitted target, then a stable valid default for an untargeted generated actor. Do not silently treat all director-marker rotations as actor transforms. If implementation adopts a significant fallback, record the decision and missing evidence in `notes/help.md` at that time; no fallback was enacted during this research pass.

Exact visual behavior after installing a zero quaternion, remaining boss model/animation offsets, and co-op presentation require a fresh real-client observation after the shared fixes. Use the acceptance matrix in the action list. No automated tests, new compatibility hooks, package edits, or native client modifications are needed for this research handoff.
