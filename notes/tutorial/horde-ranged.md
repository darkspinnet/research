# BurstShot and TutorialPlasmaLightning build-103 live contract

## Result

`TutorialPlasmaLightning` and `BurstShot` are ordinary, non-homing uses of the
same build-103 projectile template. They use `Ability_Fireball.Noun`, create a
separate replicated object for every launch, attach a persistent trail, start
reliable projectile locomotion, and move collision/damage/cleanup into a thread
owned by that projectile. Ability release does not terminate an in-flight
projectile.

The important authored distinction is:

| Property | `TutorialPlasmaLightning` | `BurstShot` |
| --- | ---: | ---: |
| Invoking noun / AI slot | `TutorialBasicRanged.Noun` / the sole phase's combat ability | `TutorialSpecialOne.Noun` and `TutorialSpecialOne_Intro.Noun` / each sole phase's combat ability |
| Activation range | `15` | rank 1 `13` (`15`, `17` at later ranks) |
| Animation | `cry_minn_el_ranged_attack1` (`0x3b617782`) | `cast_burstshot` (`0x4beb5fa3`) |
| Shot deadline from activation | `0.3000000119s` | cumulative `1.059999943s`, `1.459999949s`, `1.859999955s`, `2.259999961s` |
| Release deadline | inherited `0.4499999881s` | `2.900000095s` |
| Cooldown, paid at first launch | rank 1 `3s` | rank 1 `1s` |
| Projectile speed | rank 1 `10` | rank 1 `25` |
| Projectile travel budget | `50` | `50` |
| Rank-one damage | `1..4` | `2..7` |
| Impact radius | inherited `0` | inherited `0` |
| `trackBetweenShots` ranked value | false/default | `{false, true, true}`; rank one is false |

Activation range and projectile range are different fields. In particular,
neither projectile has a travel budget of `15` or `13`: both send and simulate
`mRange=50` before attribute modification. At unchanged rank-one attributes,
the nominal unobstructed flight limits are therefore five seconds for Plasma
Lightning and two seconds for each BurstShot projectile.

## Evidence labels and artifacts

- **Bytecode proof** means an exact constant, branch, call, or ordering in the
  shipped Lua 5.1 chunks.
- **Native proof** means a build-103 client routine, reflection registration,
  receiver, or shared simulation routine retained in
  `bin/darkspinner/GameBin/Game.c`. Server-only bodies absent from the
  client are not silently promoted to native proof.
- **Inference** means the smallest implementation consequence consistent with
  the proven content/native boundaries.
- **Missing capture evidence** means no retained EA-server trace independently
  fixes retail packet batching, RakNet reliability, allocator values, or a
  dirty-component flush position.

Exact content identities are:

| Role | Chunk | Source | Decoded SHA-256 |
| --- | ---: | --- | --- |
| Plasma definition | `423` / resource `13974` | `Abilities/0xE6324A2E.lua` | `dbc9fdd0d042d0b96690e5c45f6d235e8cf25dcac024fd04c24b9fdb5cc6ed27` |
| Burst definition | `866` / resource `14444` | `Abilities/0x9AA1174E.lua` | `3444555bd232f41179feb4fd0adfce2125ac53f8c8de94588e6adc98be8194b2` |
| Shared projectile template | `38` / resource `13559` | `Abilities/0xCE0FC9AA.lua` | `0cf624bee02973c7f6986f493f2120a5f1264987a105cee689dc844c2e5bb6c7` |

Chunk `531`, `Abilities/0xF8359A52.lua`, also contains the string `BurstShot`,
but registers `ScaryBurstShot`. It is not the tutorial ability. Compiling the
registration, rather than choosing the first string match, uniquely selects
chunk `866`.

The focused AI records independently bind the phase abilities:

| Family | Phase record | Phase SHA-256 | Noun / AI evidence |
| --- | --- | --- | --- |
| Ranged | instance `0x3aec5fe2`, ordinal `2665` | `c2dc65edcc88bbc324c18401d356d213ad67c761437c7cf7167f1faf859c1ced` | `TutorialBasicRanged` noun ordinal `2667`; AI ordinal `2669`; the one phase names `TutorialPlasmaLightning`. |
| Special One | instance `0xcc7ecbe0`, ordinal `3197` | `56acdd76cea236ef9cbdaa4b7ca3aeef4dcfa4156b63fbd127128e2105f575d6` | `TutorialSpecialOne` noun ordinal `3199`; AI ordinal `3201`; the one phase names `BurstShot`. |
| Intro Special One | same-instance `TutorialSpecialOne_Intro.NonPlayerClass` | `1eaeab2587b99774a201bbcf638814ca3458f46b5d36dae695e9988757dc82f9` | The noun explicitly links that class and its AI definition; the class owns `BurstShot`. Its separate `firstAggroAbility` slot is `FirstAggro_SpecialOne`, not `BurstShot`. |

All of these AI definitions have one phase. The phase combat-ability slot is
the caller in scope here; `firstAggroAbility`, `firstAlertAbility`, and behavior
slots are separate. In the fixed tutorial marker set there are six Ranged, two
ordinary Special One, and one Intro Special One placements. The level's
first-time director pool also makes Ranged and ordinary Special One horde-legal.
This proves eligible callers, not the unrecovered retail horde selection/count.

## Activation and launch ordering

**Bytecode proof.** Both registrations are enemy-target energy projectiles and
the template's `IsAbleToHit` callback requires physics line of sight from the
agent to the valid target object, or to the supplied target XYZ when no object
resolves. Plasma authors rank ranges `{15,15,15}`; Burst authors
`{13,15,17}`. These are admission/pursuit distances, not flight budgets.

**Client-native proof / authoritative parity requirement.** The generic
build-103 ability admission routine checks ability/rank resolution, target
state, blocking agent attributes, cooldown, mana, the ranked range predicate,
`IsAbleToHit`, and blocking modifiers before constructing an ability instance.
The range predicate measures object separation with both footprint radii
removed, or agent-to-supplied-point distance when no object resolves. The
retained client does not contain the original authoritative AI/server admission
body, so this establishes the build-103 validation vocabulary and ordering,
not a captured retail rejection response.

**Bytecode proof.** The template performs the following ordered work:

1. Snapshot agent ID, attribute snapshot, and team; start the configured attack
   animation.
2. Read the selected target once and validate it as hostile for these
   enemy-target abilities. An invalid result becomes `kInvalidGUID`; the
   ability's stored target position remains available.
3. Iterate the `timetohit` table as deltas and accumulate absolute launch
   deadlines. Plasma has one entry. Burst has `{1.06, 0.4, 0.4, 0.4}`.
4. While waiting after the first deadline and again at a launch, if the target
   still resolves, refresh the ability target position from its current
   position. Burst ranks two and three additionally call
   `TurnToFaceTargetObject` through `trackBetweenShots`; rank one does not.
5. Call `PayCooldownAndMana` exactly once, at the first launch reached. Mana is
   zero for both abilities.
6. For each projectile to launch, refresh the caster center/orientation, derive
   the shot offset/direction, create `Ability_Fireball.Noun`, copy team and the
   activation-time attribute snapshot, attach the trail, and start a
   `TrackProjectile` thread owned by the new projectile.
7. After all launches, wait to the independent release deadline and release the
   agent. Both ability-specific `deactivate` closures only delegate to the
   template, whose deactivate callback is empty.

The default `projectilesPerLaunch` resolves to one. Plasma therefore creates
one projectile; Burst creates exactly four, one at each cumulative deadline.
There is no four-hit payload and no reuse of one object ID.

**Native proof.** The attack animation is the exact 25-byte `0xa5`
`SetAnimationState` family: object ID, state GUID, 64-bit simulation timestamp,
overlay byte, scale, and zero source/echo field. The waits and cooldown payment
are simulation operations, not separate gameplay packets.

**Inference.** With no interruption, Plasma's cooldown matures at about
`t=3.3s`, later than release. Burst's one-second cooldown matures at about
`t=2.06s`, but its activation retains the agent until `t=2.9s`; the release
gate is therefore later. Exact AI scheduler re-entry on the first eligible
simulation tick is not recovered.

## Projectile IDs and creation wire

**Bytecode/native proof.** Every launch calls `nObjectManager.CreateObject`
separately. Let `I_n` be the allocator result for launch `n`. For Burst,
`I_1`, `I_2`, `I_3`, and `I_4` are distinct and are used consistently by that
projectile's create, trail, locomotion, effects, collision thread, and delete.

**Missing capture evidence.** The exact numeric IDs and whether the four values
are consecutive are not recoverable from content. The allocator is global to
the live simulation, so unrelated creations may interleave. A deterministic
fixture may allocate consecutive IDs only when it also proves no intervening
allocation; it must not encode a Burst-specific `baseID+n` rule.

For either ability, let `P` be launch position, `D` normalized launch direction,
`V=D*speed`, `Q` projectile orientation, `R` the three create rotations, and
`T` caster team. The generic `ObjectCreate` application message is:

```text
8c
I
ff 03
1f 95 7a f1                    # Ability_Fireball.Noun = 0xf17a951f
P
R
00 00 00 00 00 00 00 00
00 00 80 3f
T
00
00
00 T
04 V
06 P
07 Q
ff
```

It is 96 bytes. These enemy-target branches set team and the private attribute
snapshot but do not call `SetOwnerID`; no object field `18` is present. They are
ordinary projectiles, so object movement type field `19` is also absent.

The launch-position rule is the current caster center plus the rotated shot
offset. With default zero offsets and `launchFromCenter=false`, the template
advances along the caster's current facing by its footprint radius. Burst's
between-shot turn normally makes that facing agree with the newly computed
target direction when that ranked option is enabled, but they remain distinct
inputs. Rank-one Burst still computes every projectile direction from the
target's live launch-time position even though it does not request the explicit
between-shot turn. Exact world coordinates remain runtime inputs.

## Trail and locomotion snapshots

The template attaches the trail after creation and before entering
`WaitForProjectile`. The `0x9b` attached-effect message is 16 bytes:

```text
9b 01 S 04 01 06 <asset u32 LE> 07 I ff
```

`S` is the first free one-based slot on that projectile. Normally it is `1`,
but slot selection is object state, not an ability constant.

| Ability | Trail asset | Little-endian bytes |
| --- | --- | --- |
| Plasma | `plasma_lightningbolt_projectile.ServerEventDef` = `0x4654db5d` | `5d db 54 46` |
| Burst | `lightning_bullet_projectile.ServerEventDef` = `0x89a2c3e0` | `e0 c3 a2 89` |

The reliable projectile start is reflected `LocomotionDataUpdate` opcode
`0x94`, not fixed goal correction `0x95`. For the common horizontal case its
72-byte shape is:

```text
94 I
03
<speed f32>                    # Plasma 00 00 20 41; Burst 00 00 c8 41
00 00 00 00                    # acceleration
00 00 00 00                    # jink
00 00 48 42                    # range 50.0
00 00 00 00                    # spin
D                                # 3 float32
00 00 00 00                    # flags plus alignment
00 00 00 00                    # homingDelay
00 00 00 00                    # turnRate
00 00 00 00                    # turnAcceleration
00 00 00 00                    # piercing, ignore-ground, ignore-creature, pad
00 00 00 00                    # eccentricity
00 00 00 00                    # combatantSweepHeight in the common case
11 e8 03 00 00                 # reflectedLastUpdate = 1000
ff
```

If ordinary initialization predicts terrain collision, insert top-level field
`14` and its XYZ before field `17`, producing 85 bytes. Top-level target fields
`12`/`13`, initial-direction field `15`, homing flag/movement type, and owner
remain absent.

`mCombatantSweepHeight` is not universally zero. **Bytecode proof:**
`CalcSweepHeight` sets it to `casterZ-targetZ` when the object target is valid,
that difference is greater than `1`, and launch `dirZ > -0.9`. Otherwise it
stays zero. A wire golden for a vertically separated shot must serialize that
computed float in the fixed 60-byte projectile-parameter block.

`RangeIncrease` is applied once in Lua to the travel budget and again by the
native `WaitForProjectile` wrapper. Thus the effective and reflected range is
`50 * (1 + RangeIncrease)^2`; the bytes above assume zero increase.

**Proof boundary on ordering.** Construction and reflection baselining precede
motion initialization; `AddEffect` precedes `WaitForProjectile`. The recovered
replication sender behavior supports:

```text
0x8c ObjectCreate(I_n)
0x9b attached trail(I_n)
0x94 locomotion(I_n)
```

**Missing capture evidence:** no retail capture proves whether EA batched these
application messages into one datagram, the outer RakNet reliability enum, or
the exact flush boundary. The application ordering and byte recipes are the
parity contract; datagram grouping is not.

## Collision, target loss, and hit ordering

Both abilities inherit `homing=false`, `piercing=false`, acceleration/spin zero,
and impact radius zero.

**Native proof.** `WaitForProjectile` uses the same ordinary collision mode as
the recovered Poison Cloud projectile:

- normalize the launch direction once; target velocity is not read and no
  predictive intercept is solved unless the separate authored `leadTarget`
  option is enabled (it is false here);
- subtract actual displacement from remaining range every scheduler poll;
- test eligible dynamic objects by oriented-box overlap first and a swept-box
  query over that poll's displacement second;
- reject the source/explicit ignore IDs and apply the noun collision/team mode;
- test static/world collision separately with overlap then sweep;
- allow an endpoint/world/object impact only while remaining range is strictly
  positive; range exhaustion on the same poll wins and returns object ID zero,
  `isImpact=false`.

With inherited radius zero, the `Ability_Fireball.Noun` type-1 box uses half of
its authored `(1,1,3)` dimensions: half-extents `(0.5,0.5,1.5)`. Unlike Poison
Cloud's radius-one override, these abilities do not replace Z with `1`. The
target's actual physics geometry participates through the physics scene; the
caster/target footprint radii are not added as a 2-D circle test.

**Bytecode proof.** Every projectile runs its own `TrackProjectile` thread and
keeps its own already-processed target set. For a non-piercing direct hit the
ordered semantic work is:

1. resume on collision with hit object, impact position, and facing;
2. for a previously unseen direct object, emit the configured `impactEvent` at
   the returned position/facing;
3. select the direct object as the only radius-zero damage candidate;
4. calculate that hit's damage range and call `TakeDamage`;
5. only when damage is accepted, emit `onHitEvent`; because neither ability
   defines one, the template falls back to the same `impactEvent`, producing a
   second positioned impact notification;
6. finish tracking and mark the projectile for deletion.

Thus an accepted direct hit is effect -> damage -> effect, not one coalesced
hit. The two positioned events have the same asset/position/facing. The damage
native emits ordinary `CombatEvent` during `TakeDamage`, so the direct message
order is:

```text
0x9b positioned impact
0xba CombatEvent
0x9b positioned impact (accepted-damage fallback)
```

An HP-only `0x97` state delta is semantically dirty after `TakeDamage`, but its
placement relative to the direct notifications is not retained in client code
and is missing capture evidence.

Burst hit order is not forced to `I_1,I_2,I_3,I_4`. The four independent
projectiles may finish in a different order because each has a different
launch time, live launch direction, travel distance to contact, and collision
path. The authoritative scheduler's collision-completion order determines
effect, damage, death, and RNG order. A target killed by an earlier completion
will fail the later hostile/damageability check; that later projectile can
continue to its terminal miss path rather than applying damage to a corpse.

## Damage sampling

**Bytecode proof.** Damage is not sampled at activation or launch. For every
candidate reached at collision time, `CalculateDamage` constructs the ranked
damage input, then `TakeDamage` receives it with the activation-time attacker
attribute snapshot, current target state, damage type `Elements`, source
`Energy`, coefficient (`0`/default for Plasma; `0.05` for Burst), descriptors,
and multiplier. Each accepted Burst projectile makes a separate call.

**Native proof.** Build 103 owns a shared MT19937 stream on the simulator, not
per actor, ability, or projectile. Its bounded integer primitive is unbiased
multiply-high selection over `[0,limit)`. Other simulator consumers interleave
with projectile hits.

**Inference, not server-native proof.** The authoritative `TakeDamage` sampler
is a stub in the retained client. The repository's parity decision for integral
authored endpoints is one bounded-integer draw over `max-min+1`, producing
inclusive `1..4` or `2..7`. This is the strongest recovered implementation
choice, but a retail capture or server body is still required to prove integer
versus floating sampling and the exact critical/mitigation pipeline. The safe
ordering claim is that a draw, if required, occurs per collision-time
`TakeDamage` call in scheduler completion order.

## Positioned effects, miss, and deletion

The positioned `ServerEvent` recipe is 33 bytes:

```text
9b 06 <asset u32 LE> 0a <position 3xf32> 0b <facing 3xf32> ff
```

It has no object ID or effect slot.

| Ability | Impact asset | Miss asset |
| --- | --- | --- |
| Plasma | `plasma_common_electric_hit_small_effect.ServerEventDef` = `0x3d13d4f9` (`f9 d4 13 3d`) | `ineffective_common_small.ServerEventDef` = `0xe2d1cb21` (`21 cb d1 e2`) |
| Burst | `plasma_common_electric_hit_medium_effect.ServerEventDef` = `0x2402d741` (`41 d7 02 24`) | the same ineffective-small asset |

If tracking terminates without accepted damage, the configured `missEvent` is
used at the terminal position/facing. A miss is therefore visible and does not
emit damage. Because inherited `timeAfterImpact=0`, neither ability explicitly
removes its trail. The projectile is immediately `MarkForDelete`d after its
terminal presentation/damage path; the later object-manager sweep emits normal
`ObjectDelete` and owns attached-slot cleanup.

`MarkForDelete` itself sends nothing. **Missing capture evidence:** the exact
retail sweep/replication tick that places `ObjectDelete`, its batching, and a
framed golden for the two hit effects remain uncaptured.

## Release, cancellation, and target-loss matrix

| Case | Proven behavior | Evidence boundary |
| --- | --- | --- |
| Target invalid at activation entry | Validation replaces the object ID with `kInvalidGUID`; the template can still use the ability's stored target position and launch toward it. | Bytecode proof. Request-time admission that supplied the stored point is outside this callback. |
| Target disappears before Plasma launch | At the deadline `IsValidObject` fails; launch uses stored target position. It does not automatically cancel. | Bytecode proof. |
| Target moves before Plasma launch | The live position refresh becomes the launch point target. The resulting projectile is ordinary and does not home afterward. | Bytecode plus native proof. |
| Target moves between Burst launches | While valid, each later wait/launch refreshes target position and each new projectile gets a newly computed direction. Ranks two/three also turn the caster through `trackBetweenShots`; rank one does not. Earlier projectiles do not retarget. | Bytecode plus native proof. |
| Target disappears between Burst launches | Later launches fall back to the last stored target position; already-launched objects continue. | Bytecode proof. |
| Ability reaches release with projectile in flight | Agent release does not delete the projectile-bound thread. Plasma can fly long after `0.45s`; Burst projectiles can survive beyond `2.9s`. | Bytecode thread ownership; native scheduler lifetime is consistent. |
| Ordinary ability deactivation after launch | The Lua deactivate body is empty and sends no trail removal/delete. Projectile tracking still owns terminal effects and deletion. | Bytecode proof. |
| Cancellation before first launch | No projectile and no cooldown payment have yet occurred. | Bytecode ordering; the engine event that cancels the coroutine is not recovered. |
| Cancellation after some Burst launches | Spawned projectile threads are independently object-owned; unexecuted launch continuations should stop with the ability thread, while spawned objects continue. | Strong inference from thread ownership; a retained cancellation capture is missing. |
| Target dies after launch | Collision-time `CanBeDamaged`/hostility validation controls acceptance. No damage/on-hit fallback is emitted for a rejected corpse; terminal miss behavior remains. | Bytecode proof for validation and effect gating; exact dead-target collision playback is uncaptured. |
| Session/phase teardown | Explicit object deletion must cancel projectile threads and attached slots. | Required server ownership, not an authored ability-deactivate action. |

## Capture checklist

No retained retail or current live trace closes all of the following. Until it
does, these must remain labeled gaps rather than packet goldens:

1. One unobstructed Plasma shot with `0x8c -> trail 0x9b -> 0x94`, collision,
   both accepted-hit effects, `0xba`, HP `0x97`, and delete.
2. A full four-shot Burst with actual object IDs, proof of allocator
   consecutiveness or interleaving, per-launch target refresh, and collision
   completion order.
3. Target loss before the first shot and between Burst shots, including the
   exact stored target position visible in launch direction.
4. A wall/terrain hit proving field `14`, miss effect position/facing, and
   delete ordering; plus an exact range-expiry/collision tie.
5. Cancellation before first launch, between Burst launches, after release,
   and during flight.
6. Vertically separated combat proving nonzero `mCombatantSweepHeight`.
7. Retail damage samples sufficient to distinguish inclusive integer selection
   from floating sampling and to place critical/mitigation draws in the shared
   simulator stream.
8. Outer RakNet reliability, application-message batching, and the dirty HP
   component flush relative to the direct effect/combat notifications.
