# TutorialSloth TailZap locomotion boundary

## Result

`TutorialSloth` does not need to put its center within `0.75` of the player.
Build 103 treats an ability's authored range as clearance between collision
footprints:

```text
center distance <= sloth footprint + target footprint + TailZap range
                <= 1.30 + 0.80 + 0.75
                <= 2.85
```

The ability-range predicate is inclusive at that boundary. The corresponding
locomotion near-goal predicate uses the same three terms and a strict `<`
comparison. Thus collision radii expand the effective center-to-center reach;
they do not consume the authored `0.75`.

At an accepted attack start, native ability activation replaces the current
translation goal with a turn-in-place locomotion state, publishes that state,
and only then starts the Lua ability coroutine. The coroutine immediately sets
`cast_tailzap`, then waits until `t=0.430000007s`. TailZap itself issues no
movement calls.

The Go simulator therefore needs a mutable, authoritative Sloth world position.
It must not reconstruct the hit from the spawn marker. At minimum an accepted
TailZap run must retain the attack-start actor position and facing used by the
melee coroutine, while the world state continues to retain the current actor
position. At `t=0.43`, target validity and position are live again, and the hit
effect facing is recomputed from the live actor and hit-target positions after
accepted damage.

One boundary remains deliberately unresolved. The authoritative horde scheduler
is absent from the retained build-103 client: `ActivateHordeSpawn` reaches a
server stub. The exact retry cadence and whether the server chooses the range
boundary or a small inset cannot be recovered. The range geometry, native
object-follow goal, facing transition, Lua ordering, and authored idle behavior
on both sides of that missing owner are recoverable.

## Content identity

The authoritative local source is
`bin/darkspinner/darkspin/cache/content.db`. Focused extracts and IDA logs are
under `bin/game/logs/sloth_tailzap`.

| Role | Identity | Relevant authored data |
| --- | --- | --- |
| Sloth AssetData family | instance `0xf5a88155`; phase ordinal `3337`; noun `3339`; AI `3341` | one phase, sole combat ability `TailZap` |
| Sloth noun | `TutorialSloth.Noun` | footprint radius `1.3` |
| TailZap | chunk/resource `411 / 13962`, `Abilities/0xFB57253F.lua` | range `0.75`, animation `cast_tailzap`, hit `0.430000007s`, release `1.200000048s`, cooldown `2s`, damage `5..10`, arc length `1.25` |
| shared melee template | `832 / 14409`, `Abilities/0x7BF2D7DD.lua` | pursuit enabled; face target; common hit coroutine |
| Sloth combat idle | AI `combatIdle` -> `nBehavior_StrafeOrIdle` | lateral strafe or 4.4-second flavor idle |
| first aggro | `FirstAggro_BeamIn_Tutorial`, specialized chunk `509` over shared chunk `848` | `character_teleport_in`, duration `1.291667s` |

The tutorial hero collision radius used by the simulator/content reconstruction
is `0.8`. Sloth's noun record stores `1.3` in the same footprint field already
identified for the other tutorial nouns.

There are two indexed Lua chunks which both register
`nBehavior_StrafeOrIdle`:

| chunk/resource | source | strafe chance | SHA-256 |
| --- | --- | ---: | --- |
| `400 / 13950` | `behaviors/0x88C12B7B.lua` | `0.8` | `0b3025a8563367572eb132fb452569015aa7c2cf16fd8160899d402092289679` |
| `451 / 14004` | `behaviors/0xF0BE738E.lua` | `0.5` | `5b99271259878547788ec635844191ac423a6138080b28266e599050368b019` |

Their code is otherwise identical. No dependency or alias row proves which
duplicate global registration wins in the retail server load order. The
behavior is exact; the branch probability remains the explicit set `{0.5,
0.8}`.

## First aggro and entry into combat

Sloth's AI record supplies:

- `preAggroIdle = nBehavior_Invisible`;
- `passiveIdle = nBehavior_Wander`;
- `combatIdle = nBehavior_StrafeOrIdle`;
- `firstAggroAbility = FirstAggro_BeamIn_Tutorial`;
- `firstAlertAbility = FirstAggro_BeamIn_Tutorial`;
- `faceTarget = true`.

`nBehavior_Invisible.Activate` hides the actor and retains `Intangible` and
`InvisibleToSecurityTeleporters` modifiers. Behavior-tree replacement is
ordered: the old child is deactivated before the new child is activated.
Consequently first aggro begins only after Invisible's Deactivate has restored
visibility/tangibility and removed both retained modifiers.

`FirstAggro_BeamIn_Tutorial` then makes the actor visible/unstealthed, sets
`character_teleport_in`, and waits `1.291667s`. It does not move toward the
player. After it releases, the combat owner may select TailZap; an out-of-range
result requires pursuit, while an in-range result starts the attack.

The ordinary target-entry path is partially recoverable. `Aggro_Trigger`
validates a hostile object; first insertion records it on the object's
blackboard, posts stimulus `0x20` for ten seconds, and consumes the first-aggro
flag. Sloth's locomotion and idle leaves subsequently call `GetBestTarget` on
that per-object blackboard. The horde nouns have zero authored aggro and alert
ranges, so the horde/director owner must inject or explicitly stimulate the
tutorial player. Its server implementation is the missing part; no evidence
supports inventing a proximity poll or a fixed delay.

The safe entry sequence is therefore:

```text
spawn/invisible
  -> server-only owner supplies hostile target and requests first aggro
  -> Invisible.Deactivate (visible, tangible, modifier cleanup)
  -> FirstAggro_BeamIn_Tutorial (teleport-in animation, 1.291667 s)
  -> phase selection
  -> pursue if outside the footprint-expanded TailZap envelope
  -> accepted TailZap
```

The separate horde spawn modifier has its own exact `0.5s` stop,
`Immobilized`, `horde_beam_in`, and cleanup sequence. The available artifacts
do not order that modifier relative to first aggro, so the two durations must
not be blindly added.

## How Sloth closes to range

### Range is measured between surfaces

Native ability admission (`sub_9DE710`, exposed as `IsAbilityInRange`) obtains
both footprint radii and compares authored range with the remaining surface
gap. Ignoring unrelated range modifiers, its decision is equivalent to:

```text
surfaceGap = distance(attacker.position, target.position)
           - attacker.footprintRadius
           - target.footprintRadius

isInRange = surfaceGap <= authoredRange
```

For TailZap against the tutorial hero:

```text
surfaceGap <= 0.75
centerDistance <= 1.30 + 0.80 + 0.75 = 2.85
```

The `hitArcLength = 1.25` is separate hit-selection geometry at continuation;
it is not an extra `1.25` of activation range.

### The retained object-follow primitive uses the same envelope

`nLocomotion.MoveToObject(attacker, target, range)` reaches `sub_A00430`.
Before installing its tracked-object locomotion goal it adds the target's
footprint radius to the supplied range. Core goal construction in `sub_A149C0`
then adds the attacker's footprint radius. Its stop radius is therefore:

```text
attacker radius + target radius + requested range
```

The paired native near-goal predicate `sub_A15170` checks:

```text
attacker radius + target radius + requested range > center distance
```

This is strong evidence for the required pursuit boundary: navigation follows
the target until Sloth is inside the same footprint-expanded envelope used by
ability admission. It also updates against the target object rather than
committing to its old point.

`nLocomotion.FindGoodMeleePosition` independently confirms the collision
model. It constructs candidate melee positions displaced from the target by
the sum of attacker and target footprint radii, then navigation-projects the
candidates. That helper aims at surface contact (`2.1` center distance here),
which is safely inside TailZap's `2.85` admission envelope. The retained
content does not prove that Sloth's missing phase owner selects this helper
rather than the ordinary ranged object-follow goal, so contact must not be
claimed as Sloth's exact stop point.

What is exact and what is not:

- exact: out-of-range admission does not create an ability instance or pay
  cooldown/mana;
- exact: the native object-follow path tracks a moving target and includes both
  radii plus requested range;
- exact: admission succeeds at the inclusive `2.85` center boundary in the
  unmodified tutorial case;
- unresolved: the server-only owner call site, its retry tick, and any chosen
  inset between `2.1` contact and `2.85` maximum reach.

The simulator should navigate toward the live player and retry admission on
its simulation schedule, without requiring Sloth's center to enter a
center-only `0.75` sphere.

## Facing, movement, and animation ordering

### Pursuit to accepted activation

The native `MoveToObject` goal owns translation toward a live object. When
ability admission succeeds, `sub_9E0AE0` creates the ability runtime and checks
TailZap's `faceTarget` property before starting the Lua callback. For a valid
external target it calls `sub_A15610` with the target's current position.

`sub_A15610` clears the previous locomotion goal, installs the actor's current
position as the new goal, and writes the normalized actor-to-target direction.
`sub_9E0AE0` immediately calls `sub_A20210` to publish that locomotion state.
Thus this is a turn in place which terminates chase/strafe translation; it is
not merely an animation-facing hint.

The exact accepted-start ordering is:

```text
1. pursuit/strafe has already advanced authoritative Sloth position
2. ability admission accepts the live target and footprint-expanded range
3. allocate/link TailZap runtime
4. replace movement with turn-in-place toward current target position
5. publish locomotion/facing state
6. start TailZap Lua coroutine
7. PlayAnimationState("cast_tailzap")
8. obtain hit time and wait until t=0.430000007
```

There is no TailZap Lua `Stop`, `MoveToObject`, or later turn call. The native
pre-callback turn is the attack-start movement boundary.

### Hit continuation and release

The shared melee template snapshots at activation:

- agent and target IDs;
- agent team and attribute snapshot;
- `TargetInRangeAtStart`;
- attacker position, initial target position, and normalized initial attack
  direction;
- release time and animation index.

The animation call occurs synchronously before the timed wait. At `t=0.43`,
the coroutine resumes and, in order:

```text
1. validate the hostile target again
2. pay the 2.0-second cooldown and mana cost
3. obtain ranked range and construct the hit arc from retained attack-start
   geometry
4. select a currently valid target in that arc
5. call OnHitTarget
6. call TakeDamage(5..10)
7. if accepted damage is positive, query GetObjectDirection(attacker, hitTarget)
   from live object positions and notify charge_impact_small_effect
8. call OnDamageTarget and optional shared-template extras
9. remain owned by the ability until release at t=1.200000048
```

The continuation therefore mixes retained and live authority intentionally.
The attack arc must preserve the accepted wind-up's origin/direction; hostile
validity and the actual hit candidate are continuation-time decisions; effect
facing is a live post-damage attacker-to-target direction. Replacing all three
with either spawn-time geometry or only the player's `t=0.43` position changes
the authored semantics.

Damage mutation precedes the hit effect. Cancellation before `t=0.43` pays no
cooldown and deals no damage.

### Between attacks: StrafeOrIdle

TailZap releases at `t=1.2`, but its cooldown was paid at the hit and does not
mature until `t=2.43`. Sloth may therefore spend about `1.23s` in its authored
combat idle before TailZap can be accepted again. The behavior leaf can be
replaced by higher-priority pursuit/ability selection; its waits are not attack
cooldowns.

Each `nBehavior_StrafeOrIdle` loop chooses one of these branches:

**Strafe branch**

```text
1. GetMyObjectID
2. GetBestTarget
3. read Sloth and target positions
4. calculate the direction between them
5. choose left or right with a second 50/50 random draw
6. construct a lateral destination exactly 8 units away
7. project it through GetClosestPosition
8. MoveToPointWhileFacingTarget(Sloth, projectedPoint, target)
9. WaitForNearGoal(Sloth, GetFootprintRadius(Sloth) * 1.5)
10. choose again
```

For Sloth the near-goal threshold is `1.3 * 1.5 = 1.95`. The behavior table's
authored `radius = 5` is never read; bytecode uses literal `8`. The movement
primitive continuously faces the target during lateral translation.

**Idle branch**

```text
1. Stop(Sloth) and publish the stop
2. reacquire GetBestTarget
3. if valid, TurnToFaceTargetObject(Sloth, target)
4. SetAnimationState(Sloth, "wander_flavor")
5. WaitForXSeconds(4.4)
6. choose again
```

So the idle animation never starts before translation is stopped and the
one-shot target-facing request is made. A moving target does not cause this Lua
branch to issue another facing update during the 4.4-second wait. The strafe
branch, by contrast, encodes target-facing in its locomotion goal.

Passive `nBehavior_Wander` is precombat/passive behavior. It must not be used
as TailZap pursuit merely because it is also present in Sloth's AI record.

## Simulator position-authority contract

The current server representation has a mutable `tutorialEnemyState.position`,
but horde creation initializes it from `definition.position`, candidate
selection compares that stored position to a center-only `ability.Range`, and
melee-run construction/effect facing use `definition.position` again. Without
an authoritative movement update, that definition field remains the spawn
marker throughout the attack.

For parity, the locomotion/ability boundary needs these distinct concepts:

| State | Authority and lifetime |
| --- | --- |
| spawn position | immutable definition/marker; creation only |
| current Sloth position | mutable server-authoritative world position, updated by pursuit and strafe |
| current Sloth facing | mutable locomotion facing; target-facing during strafe and replaced by the native-equivalent turn at attack start |
| attack-start origin/direction | latched into the TailZap run after admission and the turn boundary; retained through the `0.43s` continuation |
| current target position | live authority for pursuit, continuation validation/selection, and effect-time direction |

At accepted activation the run must receive the current Sloth position, not
`tutorialEnemyDefinition.position`. It should retain that attack-start geometry
until the hit continuation. At `t=0.43`, the simulator must still be able to
read the current Sloth position and the current target position; this is needed
for the live `GetObjectDirection` used by the hit effect and for any legitimate
world correction during wind-up. In the ordinary case the turn-in-place state
keeps Sloth's current position equal to its attack-start origin, but that is a
locomotion result, not permission to substitute the spawn marker.

The native ability runtime's `TargetInRangeAtStart` is an owner-supplied byte;
its Lua accessor only reads the retained byte. The sole client-shaped caller
passes false because the authoritative NPC owner is absent. Go must compute and
latch it from the accepted, footprint-expanded envelope rather than copy that
client placeholder.

Ability-envelope checks at admission, and any equivalent continuation guard in
the simulator, must use:

```text
distance(currentSloth, currentTarget) <=
    slothRadius + targetRadius + TailZapRange
```

with hit-arc selection retaining the separate attack-start geometry. This is
the smallest authority boundary that preserves the recovered build-103
ordering without requiring Go code changes in this investigation.

## Evidence limits

The following should remain explicit rather than filled with compatibility
guesses:

- which duplicate `nBehavior_StrafeOrIdle` registration wins (`0.5` versus
  `0.8` strafe chance);
- how the absent horde owner injects its initial target and exactly selects the
  first-aggro ability;
- the ordering of the separate 0.5-second horde spawn modifier against the
  1.291667-second first-aggro ability;
- the server scheduler's pursuit retry period and whether it requests exact
  maximum reach or an inset/contact position;
- the simulation tick interval and equal-deadline actor visitation order.

None of those gaps changes the footprint-expanded `2.85` admission boundary,
the native turn-before-animation ordering, the StrafeOrIdle instruction order,
or the position authority required at TailZap's `0.43s` continuation.
