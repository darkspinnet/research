# Campaign 1-2 ordinary enemy actions (build 103)

## Result

The build-103 low band for `zelems_3` contains six ordinary combat families
that are not covered by Darkspin's implemented campaign 1-1 profiles:

| Director role | Noun | AI definition | Phase action |
| --- | --- | --- | --- |
| minion, agent | `ZelemBasicMelee.Noun` | `ZelemBasicMelee.AIDefinition` | `ZelemBasicMeleeAttack` |
| agent | `ZelemBasicRangedHoming.Noun` | `ZelemBasicRangedHoming.AIDefinition` | `ZelemBasicRangedHomingProjectile` |
| agent | `VerdanthBasicPlunge.Noun` | `VerdanthBasicPlunge.AIDefinition` | `PlungeAttack` |
| captain | `ZelemSpecialOne.Noun` | `ZelemSpecialOne.AIDefinition` | `ZelemSP1_TeleportGun` |
| captain | `ZelemSpecialTwo.Noun` | `ZelemSpecialTwo.AIDefinition` | `ZelemSpecialTwo_Push` or `ZelemSpecialTwo_Pull` |
| captain | `NomadSpecialThree.Noun` | `NomadSpecialThree.AIDefinition` | `NomadSpecialThree` |

The director also has a special-role row for
`ZelemSpecialOne_Captain.Noun`. It is not an ordinary traversal family: the
row uses the separately handled special/boss role, while the ordinary captain
pool uses `ZelemSpecialOne.Noun`. Do not alias the two noun records merely
because their names share a stem.

For a natural traversal that does not suppress legal director choices, the
smallest fidelity-complete implementation is all six primary phase actions.
The reusable vertical slice is still small: one melee, three projectiles, one
retained-position delayed area attack, and one two-gambit push/pull family.
First-aggro presentation and two noun passives add visible parity, but only the
`ZelemSpecialOne` teleport modifier is part of a primary attack's functional
result.

## Evidence labels and scope

- **Exact content** is a value or operation decoded from the authoritative
  build-103 `content.db`: director rows, noun/class operands, AI and phase
  resources, Lua 5.1 chunks, or their linked templates.
- **Exact native** is a client receiver or action contract retained in
  `bin/game/GameBin/Game.c`.
- **Inference** is a narrow conclusion that joins exact records without
  inventing a missing server policy.
- **Fallback** is an implementation choice required because the retail server
  scheduler or authority body is absent.

Difficulty `1-24` proves eligibility, not ability rank. Every numeric value in
the action table below is the first element of the authored ranked operand,
reported as **rank one**. Mapping the entire low band to rank one is a fallback
until a retained difficulty-to-rank rule is found.

## Director inventory

The following are exact `level_director_entry` rows for level id `60`,
`zelems_3`, difficulty `1-24`. All are horde-legal.

| Ordinal / reflected role | Entry id | Noun |
| --- | ---: | --- |
| 0 / minion | 903 | `ZelemBasicMelee.Noun` |
| 1 / special | 906 | `ZelemSpecialOne_Captain.Noun` |
| 2 / agent | 909 | `ZelemBasicMelee.Noun` |
| 2 / agent | 910 | `ZelemBasicRangedHoming.Noun` |
| 2 / agent | 911 | `VerdanthBasicPlunge.Noun` |
| 3 / captain | 918 | `ZelemSpecialOne.Noun` |
| 3 / captain | 919 | `ZelemSpecialTwo.noun` |
| 3 / captain | 920 | `NomadSpecialThree.Noun` |

Ordinal-to-role reflection is the same build-103 mapping documented in
`notes/campaign/1-1/director.md`. None of the six ordinary noun names resolves
to Darkspin's implemented 1-1 families (`ZelemBasicRanged`,
`ZelemBasicHybrid`, `ZelemBasicRepair`, `ZelemSpecialHaster`, `NomadSnipe`, or
`NomadWithDrone`).

## Noun and movement operands

The class attribute resource is the 88-byte `0x474940A5` record. Offsets
`0x24`, `0x28`, and `0x30` decode selectors 12 `CombatSpeed`, 11
`NonCombatSpeed`, and 48 `MovementSpeedBuff`. `sub_9E49D0` proves:

```text
effectiveSpeed = selectedBaseSpeed * (1 + MovementSpeedBuff)
selectedBaseSpeed = CombatSpeed in combat, otherwise NonCombatSpeed
```

| Noun | Instance | HP | PP | Combat | Noncombat | Buff | Scale | Footprint |
| --- | --- | ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| `ZelemBasicMelee` | `0x7FD269F6` | 22 | 75 | 8.0 | 4.0 | 0 | 1.0 | 0.50 |
| `ZelemBasicRangedHoming` | `0x5926208F` | 22 | 75 | 6.0 | 4.0 | 0 | 1.4 | 0.70 |
| `VerdanthBasicPlunge` | `0x4C3F0F7C` | 20 | 75 | 6.0 | 4.0 | 0 | 1.05 | 0.525 |
| `ZelemSpecialOne` | `0x20A6AC9F` | 110 | 75 | 5.0 | 3.5 | 0 | 2.6 | 1.30 |
| `ZelemSpecialTwo` | `0x0999BCBD` | 90 | 75 | 4.5 | 3.0 | 0 | 2.4 | 1.20 |
| `NomadSpecialThree` | `0x09160F49` | 100 | 100 | 4.5 | 3.0 | 0 | 2.7 | 1.35 |

All six records also author base attributes `10`, dodge `60`, resistance `60`,
and critical `45`. With a fresh zero movement buff, the effective movement
speeds equal the two base columns. The speed-shift passive described below is
the one immediate exception.

The special noun `ZelemSpecialOne_Captain` is instance `0xF9466CD2`, scale
`3.4`, and footprint `1.7`. Its ordinary class attribute row was not present
in the imported 88-byte operand set. That absence is another reason not to
manufacture its boss profile from `ZelemSpecialOne`.

## Common action and target contract

The build-103 native contract is shared with the already recovered 1-1
families:

1. Each actor has its own blackboard, active ability, and cooldown state.
   There is no content evidence for a wave-wide attack lock.
2. The decoded AI records author `faceTarget=true`. Admission checks the live
   target, blocking and active-ability state, cooldown, mana, range, and hit
   predicates. A rejected request does not spend cooldown or mana.
3. An out-of-range result can request pursuit. Range is tested against the
   live target and object footprints; movement must not stop at a stale spawn
   position.
4. Accepted actions face the target, play the authored animation, reach their
   hit or launch time, spend costs, and remain owned until release.
5. The generic combat-idle leaf turns when the target is more than 30 degrees
   off facing and waits `0.1 + random()*0.5` seconds before reconsidering.

`sub_9E0660` is the admission path, `sub_9E1540` creates/executes the ability,
`sub_9E4310` initializes the blackboard, and `sub_9E4640` handles threat.
`sub_A08FD0`, `sub_A149C0`, and `sub_A15170` retain the follow object and use
live strict range. These client bodies prove how admitted commands are
received and continued; they do not recover the retail server's target
selection, threat insertion, or think loop.

The class records share zero authored aggro and alert operands. Consequently,
the campaign authority must insert a hostile target. Reuse Darkspin's current
1-1 fallback—nearest live deployed hero, actor-local target ownership, and a
100 ms reconsideration cadence—unless later server evidence replaces it.
Those choices are fallback policy, not exact build-103 scheduling.

## Rank-one primary actions

| Family | Kind | Cooldown | Range / radius | Damage | Projectile / speed | Animation | Hit or launch | Release |
| --- | --- | ---: | --- | --- | --- | --- | ---: | ---: |
| Basic melee | melee | 1.5 s | 0.75 / 1.25 arc | 4-7 spacetime physical | — | `zlm_minn_sp_3_attack` | 0.26 s | 0.60 s |
| Homing ranged | projectile | 4.0 s | 15 / 0 | 4-8 + 0.05, spacetime energy | `Ability_Fireball.Noun`, 6 u/s, 18 u | `zlm_minn_sp_05_attack` | 0.366667 s | 1.0 s |
| Verdanth plunge | delayed AoE | 5.0 s | 8 / 2 | 8-12 + 0.05, life physical | — | `ver_minn_lf_04_attack1` | effect 1.0 s, hit 2.5 s | 2.866667 s |
| Teleport gun | projectile | 3.5 s | 18 / 0 | 10-16 + 0.05, spacetime energy | `Ability_Fireball.Noun`, 6 u/s, 50 u | `zlm_lieu_sp_1_attack1` | 1.35 s | 1.70 s |
| Special two push | point-blank AoE | 8.0 s | 7 / 7 | 10-16 spacetime energy | — | `zlm_lieu_sp_2_attack2` | 0.56 s | 1.8666 s |
| Special two pull | instant target | 12.0 s | 10 | 0 | — | `zlm_lieu_sp_2_attack1` | 0.33 s | 1.73 s |
| Nomad projectile | projectile | 2.5 s | 15 / 0 | 6-10 + 0.05, life energy | `Ability_HealShot.Noun`, 12 u/s, 50 u | `nomad_lieu_lf_3_attack1` | 0.433333 s | 0.880952 s |

`+ 0.05` denotes an authored coefficient operand; it is not an additional
flat 0.05 damage. The melee and push records do not author that coefficient,
so implementation must not silently copy one from the projectile template.

### `ZelemBasicMelee`

**Exact content.** The AI alternates `nBehavior_Wander` and
`nBehavior_Idle`; its phase calls `ZelemBasicMeleeAttack`. The melee is basic,
melee, and physical, has `shouldPursue=true`, and uses a `1.25` hit arc. On
hit it emits `spacetime_bite.ServerEventDef`. Its AI definition has no linked
first-aggro action.

The passive `ZelemBasicMeleePassive` is unique until death. It listens for
`TookDamage`; an AoE hit applies `MovementSpeedBuff=-0.5` for four seconds and
emits `shifted_in_and_out_effect.ServerEventDef`. It checks expiry every two
seconds. Under the native speed formula, combat speed is therefore `4.0`
while shifted, then returns to `8.0`.

**Target policy.** Admission retains the selected hostile object and pursuit
retests that live target. The melee arc is evaluated at hit time by the melee
template; it is not a point sampled when pursuit starts.

**Authority gap.** The server must schedule the passive timer and decide how
simultaneous AoE hits refresh the four-second endpoint. The Lua writes the
endpoint to current time plus four and avoids stacking another modifier;
implement that overwrite behavior.

### `ZelemBasicRangedHoming`

**Exact content.** Pre-aggro uses `nBehavior_Invisible` with wander/idle
alternates. First aggro runs `FirstAggro_Anim`,
`FirstAggro_FaceTarget`, then `FirstAggro_Anim`. The generic animation helper
calls `SetAnimationStateToAggro` and waits the duration returned by native
animation state; the face helper yields once. No fixed animation duration is
authored in this AI record.

`ZelemBasicRangedHomingProjectile` has `noGlobalCooldown=true`,
`homing=true`, `homingDelay=1`, launch offsets X/Y `1/1`, projectile
`Ability_Fireball.Noun`, trail
`spacetime_erratic_shot_effect.ServerEventDef`, impact
`spacetime_bite.ServerEventDef`, and miss
`ineffective_common_small.ServerEventDef`.

**Target policy.** The projectile retains the admitted hostile target. It
travels unguided for one second, then the projectile template follows the live
target. Target loss, collision, maximum distance, or impact ends it.

**Implemented authority.** Darkspin now publishes movement type `3` with the
projectile create and sends the retained target object, target position,
initial direction, projectile parameters, and one-second homing delay through
the complete `0x94` reflection. The server preserves the original target and,
after the unguided delay, accepts impact at the initial direct-flight deadline
only while that live target remains within the authored 18-unit travel budget.
Target loss remains a miss; no replacement hero is selected.

**Authority gap.** Treating this as a hitscan or redirecting it to a replacement
hero would contradict the content.
Client initializer `sub_A2EBA0` is the relevant retained branch: it assigns
movement type `3`, stores the target object and live target-position state,
and initializes the projectile direction before `sub_A2E320` performs the
geometry prediction. The `0x94` locomotion reflection can carry projectile
parameters in field `3`, target object ID in field `12`, target position in
field `13`, and initial direction in field `15`. Chunk 38 constructs the exact
projectile parameters: ranked speed and range, launch direction, zero spin,
eccentricity, and acceleration, non-piercing, `nProjectileFlags.kHoming`, and
the authored one-second homing delay. It never assigns `mTurnRate` or
`mTurnAcceleration`, so both retain their zero defaults. Native
`sub_A0FBE0` tests projectile flags with `& 1` and enters `sub_A2EBA0` only
when that bit is set, proving `kHoming = 1`; the current flag-one and zero-turn
wire operands are exact. The retired server's curved collision sweep and
arrival timing after the homing delay do not exist in build 103 and require a
retail server trace or implementation. The bounded retained-target impact
remains the explicit `campaign-homing-collision` compatibility fallback, not
recovered retail steering physics.

### `VerdanthBasicPlunge`

**Exact content.** The AI uses wander and `nBehavior_StrafeOrIdle` and links
no first-aggro action. `PlungeAttack` is custom Lua, not a generic
point-blank template:

1. Validate the actor and selected target, play
   `ver_minn_lf_04_attack1`, and snapshot the target's world position.
2. At `1.0` second emit
   `ver_minn_lf_04_thornBurrow_emerge.ServerEventDef` at that retained
   position.
3. At `2.5` seconds call `nTargetUtils.GetTargetsInRadius` around the retained
   position with radius `2`.
4. Damage every returned hostile object and emit
   `ver_minn_lf_04_thornBurrow_hit.ServerEventDef`.
5. Release at `2.866667` seconds.

**Target policy.** The selected legal hostile seeds a fixed impact point.
Objects in the target utility's hostile set at hit time are damaged; the
ability neither follows its original target nor redirects its center.

A separately registered `FirstAggro_VerdanthBasicPlunge` chunk plays
`ver_minn_lf_04_drop_in` for `0.9` seconds and makes the actor visible.
No operand in this noun's AI links that chunk. It is availability evidence,
not authority to run it on an ordinary director spawn.

**Authority gap.** The retained retail implementation of
`GetTargetsInRadius` does not fully disclose server faction filtering,
vertical tolerance, or deterministic ordering. Reuse the campaign hostile
filter and stable object-id order; record that ordering as fallback.

### `ZelemSpecialOne`

**Exact content.** The AI uses invisible/wander/strafe/idle and the same
`FirstAggro_Anim`, `FirstAggro_FaceTarget`, `FirstAggro_Anim` sequence as
the homing minion. `ZelemSP1_TeleportGun` launches
`Ability_Fireball.Noun` with trail
`spacetime_lieu_shot_projectile.ServerEventDef`, impact
`warp_impact_effect.ServerEventDef`, and the common ineffective miss effect.

On projectile damage it requests modifier GUID `0x2ACBC7D9`, which resolves
by the content hash algorithm to `RandomTeleport`. That modifier is a unique
debuff lasting `0.2` seconds. Its `NavigationUtils.GetRandomReachablePosition`
helper makes at most 20 attempts. Each attempt consumes one global Lua
`math.random()` draw, maps it to a uniform angle over `[0, 2*pi)`, and tests the
unchanged point exactly eight units from the victim at the victim's current Z.
The native `IsPositionReachable` bridge accepts a point only when it is close
to a navigation polygon connected to the victim's current navigation region;
it returns only a boolean and does not project or replace the candidate. Lua
also requires the candidate distance to be strictly greater than the authored
minimum `2`. The authored maximum `10` and `maxTeleportAttempts = 4` fields are
never read by this helper. On success the modifier emits
`warp_impact_exit_effect.ServerEventDef` at the destination, plays
`react_teleported`, waits `0.1` seconds, and then teleports the victim. On
failure it performs none of those presentation or movement steps. Either path
yields once, removes the modifier, releases the agent, and resets the target
animation. `ImmuneToRandomTeleport > 0` rejects the modifier outside chain
games; chain games explicitly bypass that immunity.

The passive `ZelemSP1Passive` is unique until death and gives rank-one AoE
resistance `0.75`.

**Target policy.** The projectile and teleport modifier retain the selected
hostile victim. Random teleport moves that victim only; it does not select a
new combat target.

**Implemented authority.** The base and replay-tier nouns now use the authored
projectile, animation, launch/release/cooldown timing, effects, speed, range,
and damage contract. The current teleport owner needs correction: it tries
four deterministic directions, projects them, and enforces the declared
`2/8/10` operands, whereas the recovered contract makes up to 20 independent
uniform-angle attempts at fixed radius 8, uses boolean reachability without
projection, applies only the strict minimum-distance check, and ignores the
declared maximum and four-attempt fields. If no valid point is found, the
victim correctly remains in place.

**Remaining implementation.** Shared co-op ownership currently limits
teleport mutation to a hero local to the NPC action session; remote heroes and
companions retain the damage but remain in place. The campaign stat owner also
does not yet expose `ImmuneToRandomTeleport`, including the recovered chain-game
bypass. These are runtime ownership and stat-projection tasks under the exact
sampling and immunity contract above, not open content-research questions.

### `ZelemSpecialTwo`

**Exact content.** Pre-aggro uses invisible/wander/flee-facing/idle. First
aggro runs `FirstAggro_BeamIn`, `FirstAggro_FaceTarget`, then beam-in again.
Beam-in plays `gen_aggro_sp_beam_in` for `1.23` seconds and changes the actor
from invisible/stealthed to visible.

The phase has a named `Distance` condition with scalar `5.0` and two actions:
`ZelemSpecialTwo_Push` and `ZelemSpecialTwo_Pull`.

- Push is a point-blank hostile AoE. It applies modifier GUID `0x4B7CBAC5`,
  `ZelemSpecialTwo_Knockback`: speed `12`, desired distance `6`, outro
  `0.3`, and animation `react_knockback`. The shared knockback template rejects
  rooted targets unconditionally and rejects `ImmuneToKnockback > 0` outside
  chain games. On activation it takes the initiator's ground position, derives
  the outward normalized target-minus-initiator direction, plays the reaction,
  and calls `JumpInDirection` with distance 6, speed 12, and the template's
  inherited jump-height operands `2/4/4`. It waits for jump completion, then,
  if the target is alive, resets animation, stops locomotion, and waits the
  `0.3`-second outro. Interruption deactivates the modifier.
- Pull is a zero-damage selected-target cast. It applies modifier GUID
  `0x9F842FF1`, `ZelemSpecialTwo_Tractor`: pull speed `25`, stop distance
  `1`, outro `0.4`, and
  `spacetime_lieu_pull_affectedEnemy_effect.ServerEventDef`. The shared pull
  template rejects `ImmuneToPull > 0` outside chain games. A rooted target can
  receive the modifier but activation deliberately creates neither movement
  nor effect. Otherwise it computes initiator-minus-target direction and the
  edge distance after subtracting both footprint radii; only a positive edge
  distance plays `react_pulled`, attaches the effect, and calls
  `JumpInDirection` for that full edge distance at speed 25. It waits for jump
  completion, resets a surviving target's animation, waits the `0.4`-second
  outro, and removes the exact effect on deactivation.

The native `JumpInDirection` bridge passes all 11 Lua operands into
`sub_A15240`. That routine resolves terrain along the requested displacement
in `0.2`-unit increments, fails without publishing movement when it cannot
find a valid ground result, applies the agent-footprint navigation adjustment,
and stores the resulting endpoint, direction, speed, and three jump-arc
operands. `sub_A20690` publishes those ten movement fields plus object ID as
native locomotion command type `17`, payload size `44` bytes. Pull uses zero
jump-arc operands; push uses `2/4/4`. Client locomotion owns interpolation to
the command's resolved endpoint, and Lua waits on `WaitForJumpComplete` rather
than scheduling its own speed-based position ticks.

**Target policy.** Push queries hostile targets within radius seven at hit
time and independently requests knockback on every accepted damage recipient;
it is not a single-target push. Pull retains one admitted hostile and moves it
toward the caster until interruption or modifier completion. The ability's
one-unit stop operand governs its channel admission, while the pull modifier
itself uses footprint-separated edge distance for the jump.

**Resolved - branch polarity.** Decoded
`ZelemSpecialTwo.Phase` is build-103 AssetData ordinal 7904, type `Phase`,
instance `0x0999BCBD`, decoded SHA-256
`7a2627e41fb1b1ace03a53d3c30c1f5a4ca899692a2f738b698bb2d6c11fca55`.
It stores `Distance = 5`, omits the optional `GreaterThan` property, and orders
Push before Pull. The registered `Condition_Distance.lua` definition gives
`GreaterThan` an authored default of `false`; that branch returns true when
`threshold >= current object distance`. Therefore the exact phase policy is
**push at distance <= 5, pull otherwise**, inclusive at exactly five units.

**Implemented authority.** The base and replay-tier nouns use a ten-unit
decision pursuit. At cast time they choose the authored push at or inside five
units and pull outside it. Push applies its `10-16` hit at `0.56s`; pull
retains its selected target and applies no invented damage at `0.33s`. Each
uses its own animation, release, and cooldown. Every admitted local hero,
remote co-op hero, or companion moves to the furthest navigation-reachable
point toward the six-unit push endpoint or one-unit pull stop, while rooted
targets reject movement. The movement and shared pose commit together with the
authored reaction or affected-target effect. Native speed-12/speed-25 jump
interpolation and its interruption lifecycle remain a compatibility fallback.
After each Push or Pull release, Magnetic Master now completes one straight
navigation-clipped retreat and then a separate left-or-right bounded combat
strafe before cooldown resumption. This preserves the observed flee while
making the expected post-flee combat walk visible as its own movement phase.

### `NomadSpecialThree`

**Exact content.** The AI uses strafe/wander, has no linked first-aggro
action, and calls `NomadSpecialThree`. Despite its noun name,
`Ability_HealShot.Noun` is a damaging projectile here. It uses trail
`nomad_lieu_lf_3_projectile.ServerEventDef`, impact
`nomad_lieu_lf_3_projectile_hit.ServerEventDef`, and the common ineffective
miss effect.

`NomadSpecialThreePassiveModifier` is unique until death. Initially it adds
`EnergyDefense=250` and `creature_shield_effect.ServerEventDef`. At or below
50% HP it:

1. removes that shield and energy defense;
2. adds `PhysicalDamageReduction=1`, `ImmuneToDebuffs=1`,
   `ImmuneToKnockback=1`, and `ImmuneToPull=1`;
3. requests `NomadSpecialThreeTurtleModifier`;
4. restores the initial shield/energy defense and removes the turtle
   immunities when that modifier ends.

The turtle modifier plays `nomad_lieu_lf_3_attack2` and its recovery, emits
`nomad_lieu_lf_3_AOE.ServerEventDef`, and performs six rank-one ticks one
second apart in radius five. Each tick deals `8-12 + 0.05` life/energy AoE
damage and applies `TurtlePoison`. The poison lasts one second, ticks every
`0.5` seconds for `8-12 + 0.05` life/energy damage, and emits
`status_poisoned.ServerEventDef`.

**Target policy.** The primary projectile retains one selected hostile.
Turtle uses the template's hostile radius query around the Nomad on each tick;
it is not limited to the projectile victim.

**Authority gap.** Exact tick boundary behavior, reapplication of the
one-second poison, actor death during turtle, and ordering of coincident
modifier events require an authoritative modifier scheduler. Preserve the Lua
phase order and stable object-id iteration. The primary projectile can ship
before turtle, but that is an explicitly incomplete family.

## Implemented action status

- `ZelemBasicRangedHoming` now uses the retained-target projectile path with
  movement type `3`, the authored one-second homing delay, and bounded live
  target impact authority. Homing flag one and zero turn-rate/turn-acceleration
  are now exact; curved server collision and arrival stepping require an
  external retail server trace or implementation.
- `ZelemSpecialOne` now uses its authored teleport-gun projectile and performs
  the recovered 20-attempt, uniform-angle, fixed-radius-eight connected-region
  teleport for a successfully hit local hero, remote co-op hero, or companion.
- `ZelemSpecialOne_Captain` and its replay-tier variants retain their distinct
  noun identities and imported captain stats while using the same directly
  linked `ZelemSpecialOne.AIDefinition` teleport-gun action as the base noun.
- `ZelemSpecialTwo` selects Push at the exact inclusive distance-five branch
  and Pull otherwise, preserving their distinct damage, timing, and cooldown
  contracts. Push now queries every living hostile at its hit frame and applies
  its 10-16 damage independently to each footprint intersecting radius seven;
  accepted hits then use the obstacle-clipped six-unit displacement. Rooted heroes
  reject Push and Pull movement, while navigation-approved destinations commit
  to shared hero and companion pose authority for local and remote co-op
  targets. Successful commits project the exact destination, reaction, and
  effect to every other campaign member without duplicating the action owner's
  immediate presentation. Native jump operands, command shape, and client
  interpolation remain implementation work.
- `VerdanthBasicPlunge` now retains the admitted position across its authored
  animation, emerge, hit, release, and cooldown schedule. Its strict radius
  query damages every local hero, remote co-op hero, and companion admitted at
  the retained hit point.

## First-aggro summary

| Family | Linked first-aggro behavior | Implementation status |
| --- | --- | --- |
| Basic melee | none | phase-eligible on next think |
| Homing ranged | native aggro animation, face target, animation | exact sequence; animation duration comes from client state |
| Verdanth plunge | none | do not invoke the orphan drop-in chunk |
| Special one | native aggro animation, face target, animation | exact sequence; animation duration comes from client state |
| Special two | beam-in, face target, beam-in | exact `1.23 s` beam-in helper |
| Nomad three | none | phase-eligible on next think |

Do not admit the first phase ability until a linked first-aggro sequence
finishes. Do not make the no-link families wait an invented cinematic delay.

## Implementation ranking

### Required profile gate

First add data profiles for all six ordinary nouns: HP/PP, scale, footprint,
base attributes, movement operands, AI identity, and first phase. Director
selection can legally choose any of them; allowing a selected noun to fall
through to an absent profile makes traversal success depend on random pool
composition.

### Primary-action order

1. **`ZelemBasicMelee`** — guaranteed minion-pool presence and also an agent
   choice; it reuses Darkspin's melee, pursuit, facing, cooldown, and effect
   machinery.
2. **`ZelemBasicRangedHoming`** — implemented through the shared projectile
   path with one-second delayed retained-target homing and recovered flag/turn
   operands; exact curved server collision and arrival stepping require an
   external retail server trace or implementation.
3. **`VerdanthBasicPlunge`** — implemented with retained position, delayed
   presentation, multi-hostile radius admission, damage, release, pursuit, and
   repeat scheduling under the recovered fixed-center radius-query contract.
4. **`NomadSpecialThree` primary projectile** — the simplest captain action
   and almost entirely reusable projectile machinery.
5. **`ZelemSpecialOne` teleport gun** — implemented through projectile reuse
   and the recovered 20-attempt fixed-radius connected-region sampler for local,
   remote co-op, and companion target ownership.
6. **`ZelemSpecialTwo` push/pull** — implemented with the recovered branch,
   distinct action schedules, root gates, obstacle-clipped endpoint authority,
   and multi-target local, remote co-op, and companion ownership.

This ordering describes incremental value, not a license to declare parity
after step one. The minimal visible vertical slice—profiles plus basic melee—
makes guaranteed minion packs fight. The minimal **natural traversal-safe**
set is steps 1-6 because any agent or captain candidate may be selected without
suppression.

### Follow-on parity

After all six primary actions:

1. validate native aggro-animation duration lookup and presentation packet
   order for the two generic-animation families.

## Implementation tests

The implementation should have actor-local deterministic tests for:

- each noun's exact base speed, footprint, HP/PP, and phase identity;
- rejected admission causing no cooldown, mana, damage, or modifier write;
- pursuit retrying against a moving live target and using footprints;
- exact hit/launch and release boundaries for every primary action;
- homing staying unguided for one second, then following only its retained
  target;
- plunge retaining the initial position while selecting radius victims at
  hit time;
- teleport immunity, four failed reachable samples, and a successful
  distance-bounded teleport;
- push selecting radius victims and pull stopping at one unit;
- the fallback distance branch at `5.0`, named explicitly in the test;
- actor independence: one family's cooldown or presentation never blocks
  another actor;
- target death before hit, actor death during action, and target loss during
  projectile or forced movement;
- passive refresh/removal boundaries and Nomad turtle transition at exactly
  50% HP.

Trace assertions should allowlist player-visible action, locomotion, effect,
modifier, and damage fields. They must not serialize an internal profile or
modifier aggregate directly to the client.

## Remaining server-authority gaps

The following are not recoverable from the inspected client/content evidence
and must remain visibly labeled fallbacks:

- initial hero selection, threat insertion, retarget policy, actor iteration,
  think cadence, and RNG stream;
- difficulty-to-ability-rank conversion;
- navigation path choice and cross-session movement mutation ownership;
- projectile step interval, collision ordering, target-loss behavior, and
  authoritative packet scheduling;
- hostile-radius ordering and exact faction/vertical filters;
- modifier/tick scheduling at coincident time boundaries;
- the special-role `ZelemSpecialOne_Captain` boss policy.

These gaps do not justify inventing a client patch. Use conservative
server-owned fallbacks and keep the packaged installation read-only. Random
Teleport's angular sampler and Zelem Special Two's forced-movement client
contract are recovered above and now belong to the implementation queue, not
this evidence-gap list.

## Provenance

- Client version: `bin/game/GameBin/version_bin.txt`,
  `5.3.0.103`.
- Executable: `bin/game/GameBin/Game.exe`, SHA-256
  `3c7248a4c6614450290a66c22ab7fcda86052b29a7b555e834d7c23f9b73ee5b`.
- Canonical decompiler:
  `bin/game/GameBin/Game.c`, SHA-256
  `1fac9d309c9d5f90ffb8150b3b139baee0a56dfa223f8ff73da4e240777b1eb`.
- Authoritative imported content:
  `bin/darkspinner/darkspin/cache/content.db`, SHA-256
  `13fd50e130c5725c9a0037c301872002740e6cf221dd18c55f0f196545bb04`.
- Packaged source:
  `bin/game/Data/AssetData.package`, SHA-256
  `faf796f4da15a5a476269709d900371344a28cc7a9ef4cd4a2b5c3d2574f2b`.

Key Lua resources:

| Chunk | Content resource | Source identity | SHA-256 |
| ---: | ---: | --- | --- |
| 589 | 14151 | `Abilities/0x9CCA38BC.lua` | `2ad271a195389732f959480489212818ba7762ae08452ecc3e6c01933aacb24d` |
| 490 | 14045 | `Abilities/0x917178B8.lua` | `74e78cbfd752fa7a1df0b3070aa077bd5693ae88b2d9fff6d05302395a6a6b92` |
| 455 | 14009 | `Abilities/0xDE424078.lua` | `28f384025c8239869ec2970c4ec3de63a1e9fba1e4622f5c8f58220e0341a852` |
| 856 | 14433 | `Abilities/0xEF9FD403.lua` | `a52e6bbc51b89695545718575607804919a585449034411324528ac73ac5f7c4` |
| 533 | 14093 | `Abilities/0x766CD5C1.lua` | `38f26ad191c76f15e92cf8f585194299585d131fbd8c3b358f62471d033b0f45` |
| 387 | 13935 | `Abilities/0x6F6CCAC8.lua` | `7b16027e33e728d6ac9092fa459ad78d20936422039954465049189074296bbd` |
| 214 | 13752 | `Abilities/0x649DA5DA.lua` | `27c028d3a4037d18242382222704234a843f717e069e0a8d1b79386d6d3ddd05` |
| 678 | 14249 | `Modifiers/0x340A4CFC.lua` | `adad5ba90e0f0c637f5ce7d57807b482c6b10f351da7ff9730484784ad726c07` |
| 77 | 13600 | `Modifiers/0xB6F1EB76.lua` | `b00179f0aa39ef2b56e79067acc58bd6793e4839bcf8478f34834dac2a854bb9` |
| 523 | 14082 | `Modifiers/0xD9A56F91.lua` | `b2dce205888bd4d1e54dbe7f0a57bbfd50df1a9f13b843a910ba06806576cb62` |
| 2 | 13518 | `Modifiers/0x40582E31.lua` | `c6567b31ad849e6cc86b6b90dc936a8ffb7bb5805d23a791d89be8c978a33425` |
| 712 | 14284 | `Modifiers/0x56471355.lua` | `518a458d0e7d0509b7036610c8bca86fa8dc68a3e04bb0b9ba59af689101dc8b` |
| 356 | 13904 | `Modifiers/0x3F376EEF.lua` | `e7de41bb4b7c0486d5f205ab9e3f91973c1f001a450e54d25995d6523f9aa5fe` |
| 647 | 14212 | `Modifiers/0x7AC2DD8E.lua` | `d9d53ad937ad5acc4e049414ab77878ea12e7f373ccde5ae4efb6ccf81d65bd0` |

The AI/noun resource adjacency and content hashes used to bind these chunks
were checked in `content.db`; the stable instance ids and source identities
above are the implementation lookup keys. No packaged resource was modified
or extracted into the game installation.
