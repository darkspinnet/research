# Build-103 tutorial horde enemy AI

## Result

The five tutorial horde nouns are one-phase AI agents. Each phase contains one
combat ability, so ability choice is deterministic once the actor has a target:

| Noun | Sole phase ability | Activation range | Cooldown payment | Release |
| --- | --- | ---: | --- | ---: |
| `TutorialBasicPoison` | `TutorialPoisonMelee` | `0.75` | hit at `t=0.43s`; `2s` cooldown | `t=2.00s` |
| `TutorialSloth` | `TailZap` | `0.75` | hit at `t=0.43s`; `2s` cooldown | `t=1.20s` |
| `TutorialBasicRanged` | `TutorialPlasmaLightning` | `15` | launch at `t=0.30s`; `3s` cooldown | `t=0.45s` |
| `TutorialBasicDiseased` | `TutorialPoisonCloud` | `8` | launch at `t=0.17s`; `3s` cooldown | `t=1.86s` |
| `TutorialSpecialOne` | `BurstShot` | rank-one `13` | first launch at `t=1.06s`; `1s` cooldown | `t=2.90s` |

Mixed-wave attacks are **concurrent, not wave-serialized**. Every creature owns
its own brain, behavior thread, active-ability collection, cooldown state, and
ability coroutine. A running ability blocks that actor; it does not take a
wave-wide attack token. Actors are visited serially inside a simulation tick,
so equal-deadline notifications have a deterministic processing order, but
their actions and cooldown clocks overlap. Burst projectiles are even more
independent: every launch creates a separate object-owned tracking thread, and
collision completion can interleave with attacks from every other creature.

There is one important evidence boundary. Build 103's client-shaped
`Game.c` retains ability admission, behavior-tree, Lua-thread, locomotion,
and receiver code, but the authoritative server body that activates a horde is
a stub: `nGameDirector.ActivateHordeSpawn` reaches `sub_A05920`, which resolves
the horde object and calls `sub_A23560`; `sub_A23560` immediately returns zero
(`Game.c:1427077-1427103`, `1450535-1450539`). The shipped content does not
contain a Lua replacement for that owner. Consequently the exact server-only
operations that inject the player into every freshly spawned blackboard and
retry a rejected phase ability are not recoverable from the allowed artifacts.
They must not be replaced with an invented perception poll or fixed attack
timer. Everything on either side of that missing owner is recoverable and is
specified below.

## Evidence and identities

The authoritative content source is
`bin/darkspinner/darkspin/cache/content.db`; the relevant packaged records are
in `bin/darkspinner/Data/AssetData_Binary.package`. Focused decoded records are
retained under `bin/game/logs/horde-ai`, as required for generated reverse-
engineering diagnostics.

| Family | instance | phase ordinal | class ordinal | noun ordinal | AI ordinal | phase ability |
| --- | ---: | ---: | ---: | ---: | ---: | --- |
| Poison | `0x09a3ac7f` | `6675` | `6674` | `6673` | `6671` | `TutorialPoisonMelee` |
| Sloth | `0xf5a88155` | `3337` | `3338` | `3339` | `3341` | `TailZap` |
| Ranged | `0x3aec5fe2` | `2665` | `2666` | `2667` | `2669` | `TutorialPlasmaLightning` |
| Diseased | `0x52c73d4d` | `3718` | `3719` | `3720` | `3722` | `TutorialPoisonCloud` |
| Special One | `0xcc7ecbe0` | `3197` | `3198` | `3199` | `3201` | `BurstShot` |

All five phase resources have phase count one and no alternate/start phase.
The ability reference is the only nonempty combat choice in each phase. Native
`sub_A22A50` resolves the noun's AI reference, initializes the phase array, and
leaves phase index zero selected when the linked phase's start byte is zero;
all five bytes are zero.

The Lua identities used for the timelines are:

| Role | chunk / resource | source | SHA-256 |
| --- | --- | --- | --- |
| Poison melee | `898 / 14476` | `Abilities/0x2B8B0FB2.lua` | `a4d5f23fd349091f1e1cb1deefec4d2dde913cf3b8c3735d36ac156a03a81033` |
| shared melee template | `832 / 14409` | `Abilities/0x7BF2D7DD.lua` | `2951fef16876a72687c235cc93377ede9dcec31286a8237a9c6a8af978da778a` |
| Poison Cloud | `581 / 14143` | `Abilities/0x66D97AC9.lua` | `076202c42dec21692943d05402a1d42ef47a78ec224ba056fd947ad9e126eee3` |
| Plasma Lightning | `423 / 13974` | `Abilities/0xE6324A2E.lua` | `dbc9fdd0d042d0b96690e5c45f6d235e8cf25dcac024fd04c24b9fdb5cc6ed27` |
| BurstShot | `866 / 14444` | `Abilities/0x9AA1174E.lua` | `3444555bd232f41179feb4fd0adfce2125ac53f8c8de94588e6adc98be8194b2` |
| shared projectile template | `38 / 13559` | `Abilities/0xCE0FC9AA.lua` | `0cf624bee02973c7f6986f493f2120a5f1264987a105cee689dc844c2e5bb6c7` |
| combat idle | `539 / 14099` | `behaviors/0x9F7E7478.lua` | content DB indexed bytecode |
| strafe or idle, variant A | `400 / 13950` | `behaviors/0x88C12B7B.lua` | content DB indexed bytecode |
| strafe or idle, variant B | `451 / 14004` | `behaviors/0xF0BE738E.lua` | content DB indexed bytecode |
| wander | `973 / 14553` | `behaviors/0x34A01B05.lua` | content DB indexed bytecode |
| invisible | `865 / 14443` | `behaviors/0x104FCC83.lua` | `dcdab6384645e166b5dd88e23c15246cf421c72946d78b5eaba686976dd40ecf` |

`content.db` contains two modules that both register
`nBehavior_StrafeOrIdle`. Their bytecode is otherwise identical, but the
authored strafe probability is `0.8` in chunk 400 and `0.5` in chunk 451.
Neither has a dependency/alias row that proves which registration wins in the
retail server's load order. The exact lateral/idle state machine is proven;
the probability is therefore a two-valued unresolved content collision, not a
license to choose one silently.

## Acquisition and first aggro

### Spawn state

All five AI definitions have:

- `preAggroIdle = nBehavior_Invisible` at AI offset `+0x7c`;
- `firstAggroAbility = FirstAggro_BeamIn_Tutorial` at `+0x34`;
- `firstAlertAbility = FirstAggro_BeamIn_Tutorial` at `+0x54`;
- `faceTarget = true` at `+0x278`.

Poison, Ranged, and Sloth use `nBehavior_Wander` as `passiveIdle` at
`+0x170`. Diseased and Special One leave it empty. Sloth alone supplies
`combatIdle = nBehavior_StrafeOrIdle` at `+0x11c`; the other four leave the
authored combat-idle slot empty and use the generic combat-idle fallback.

`nBehavior_Invisible.Activate` makes the object invisible and adds
`Intangible=1` and `InvisibleToSecurityTeleporters=1`. Its Tick waits forever.
On replacement, Deactivate restores visibility and removes both retained
attribute handles. Native selector `sub_A329B0` deactivates the old child before
activating its replacement, so the creature is made tangible before the next
behavior starts.

The horde spawn modifier is a separate, exact `0.5s` state. Chunk 655
(`SpawnModifier`) stops the actor, adds `Immobilized`, attaches
`generic_spawn.ServerEventDef`, sets animation `horde_beam_in`, waits `0.5s`,
then removes the effect and resets animation. Native modifier teardown owns
the retained immobilization handle. The allowed artifacts do **not** prove
whether the server starts `FirstAggro_BeamIn_Tutorial` before, during, or after
that modifier; treating `0.5 + 1.291667` as a universal spawn delay would be
an invented ordering.

The first-aggro ability makes the actor visible/unstealthed, sets
`character_teleport_in`, and waits exactly `1.291667s`. It does not itself
remove the invisible behavior's attribute handles; the proven behavior-tree
replacement performs that cleanup first.

### How the player becomes the target

Every focused `NonPlayerClass` stores:

```text
aggroRange +0x38 = 0.0
alertRange +0x3c = 0.0
```

Native perception uses `max(aggroRange, alertRange)`. Therefore these horde
actors do not acquire a remote player through a content-authored radius. A
fresh horde must receive the player from the authoritative horde/director
owner (or an explicit aggro stimulus). The ordinary aggro insertion path is
otherwise exact: `Aggro_Trigger` validates a hostile entrant and calls
`sub_9E4640(blackboard, targetID, 5.0, 2, "Trigger Volume")`; a first insertion
records the target, posts stimulus `0x20` for ten seconds, and sets first-aggro
consumed byte `+1365`. `nAgent.GetBestTarget` reaches native `sub_A00D80` and
all movement/idle leaves read that per-object best target.

The missing horde activation stub prevents proving that its target injection
uses this exact helper, or assigning a delay between object creation and target
insertion. The correct recovered boundary is therefore:

```text
create actor -> apply spawn/invisible state
             -> [server-only horde owner injects player and requests first aggro]
             -> old behavior Deactivate
             -> first-aggro ability Activate
             -> combat phase becomes eligible
```

There is no evidence for round-robin target assignment, a nearest-player poll,
or one global target shared by the wave. With one tutorial player, every
injected per-object blackboard will naturally return that player.

## Pursuit, stopping, strafing, and retry

Native ability admission is `sub_9E0660`. It checks definition/rank, live and
legal target state, blocking attributes, a conflicting ability on the same
agent, cooldown, mana, range, and hit predicates. `sub_9E1540`
(`Game.c:1396858-1396903`) creates an ability instance only when admission
returns `1`. Range/cooldown/conflict rejection creates no instance and pays no
cost. The client-retained code can also return the pursuit classification `3`,
but the top-level NPC owner that converts an out-of-range phase choice into a
locomotion goal and retries it is part of the absent server horde scheduler.

The retained admission and locomotion paths imply the following minimum parity
contract. The missing server owner means the exact goal inset and retry cadence
are not recoverable:

1. Read this actor's `GetBestTarget` and its sole current-phase ability.
2. If admission is out of range, move toward a point that places the actor
   inside that ability's activation range; do not start an ability instance or
   cooldown. Whether the retail owner selected the exact boundary or an inset
   cannot be recovered.
3. Continue authoritative navigation as the target moves. Re-evaluate on a
   simulation tick; there is no authored periodic attack interval.
4. On the first accepted tick, the chase goal is no longer needed,
   `faceTarget=true` orients the actor, and the ability instance starts. The
   ability templates do not issue an additional explicit `nLocomotion.Stop`;
   ordinary stopping is the range-goal completion, while SpawnModifier and
   Sloth's idle branch are the explicit Stop call sites.
5. While the instance owns the actor, another blocking ability for that actor
   is rejected. Other creatures continue independently.
6. At ability release, that actor becomes selectable again. If cooldown still
   runs, admission rejects it; the actor enters its combat-idle leaf. If the
   target has left the envelope, the next successful scheduler choice resumes
   pursuit. If in range and cooldown is mature, it can start again on the next
   eligible simulation tick.

No allowed artifact fixes the server simulation step or the retry poll period,
so “next tick” cannot be converted into a number of milliseconds.

The generic `nBehavior_CombatIdle` loop is exact. Every iteration obtains the
best target; if the target is more than 30 degrees off facing (angle strictly
between `30` and `330` degrees), it turns to the target position. It then waits
`0.1 + math.random()*0.5` seconds, i.e. `[0.1,0.6)` for the ordinary Lua
random contract, and repeats. This delay is a facing/idle coroutine delay, not
an ability cooldown and not a wave serialization gate. The hardcoded selector
fallback activates this behavior for the four actors with no authored
`combatIdle`.

Sloth replaces that idle with `nBehavior_StrafeOrIdle` after its attack leaf
releases:

- **strafe branch:** read the live best target, build a random left/right
  lateral destination eight units from the current position, project it with
  `GetClosestPosition`, call `MoveToPointWhileFacingTarget`, and wait until the
  actor is within `1.5 * footprintRadius` of the goal; then choose again;
- **idle branch:** call `nLocomotion.Stop`, turn to a valid target, set
  `wander_flavor`, wait exactly `4.4s`, and choose again.

The branch probability is the unresolved duplicate registration described
above (`0.8` or `0.5` strafe). Movement can be interrupted by a higher-priority
ability selector as soon as admission succeeds; the 4.4-second wait is not a
minimum delay before an attack when the behavior tree replaces this leaf.

`nBehavior_Wander` is passive/noncombat behavior, not pursuit. Its Activate
chooses a navigable point within radius five, and Tick moves exactly to saved
points, waits near the goal, may play flavor idle, then chooses another point.
Its Deactivate calls `nLocomotion.Stop`. It must not be used to implement the
ability-range chase.

Chunk 826, `nBehavior_MoveToRange`, is also not proof of the horde's ordinary
combat loop. It is the behavior-tree leaf for queued event `0x10`: it consumes
thread-data command fields, calls `MoveTowardObject`/`MoveToPointWithinRange`,
then invokes `RequestAbility`, and sends action cancel on deactivation. It is
the retained action-command pursuit mechanism. The static creature tree gates
it specifically on event `0x10`; ordinary horde phase selection is owned by the
missing server scheduler.

## Exact ability state transitions

Times below are offsets from an accepted activation at `t=0`. Cooldown maturity
is measured from the template's `PayCooldownAndMana`, not from acceptance.
Release and cooldown are independent gates; the next attack cannot begin until
both the actor is released and admission accepts cooldown/range/target state.

### TutorialBasicPoison: TutorialPoisonMelee

```text
t=0.000  accept target in 0.75 range; face target; set cry_minn_lf_poison_attack1
t=0.430  revalidate/select hostile hit target; pay 2.0s cooldown
         OnHitTarget -> TakeDamage(1..3)
         on accepted positive damage: object-bound melee ServerEvent
t=2.000  release actor
t=2.430  cooldown matures
```

The actor is released before cooldown maturity, leaving a `0.43s` cooldown-only
idle window. Damage mutation and `CombatEvent` precede the melee hit effect.
Cancellation before `0.43s` pays no cooldown and deals no damage.

### TutorialSloth: TailZap

```text
t=0.000  accept target in 0.75 range; face target; set cast_tailzap
t=0.430  revalidate hit; pay 2.0s cooldown; apply 5..10 damage and hit effect
t=1.200  release actor into nBehavior_StrafeOrIdle
t=2.430  cooldown matures
```

There is a `1.23s` release-to-cooldown window in which Sloth can strafe or idle
but cannot start TailZap. If the target leaves melee range, pursuit supersedes
idle after the owning scheduler chooses/retries the phase ability.

### TutorialBasicRanged: TutorialPlasmaLightning

```text
t=0.000  accept target in 15 range; snapshot attacker attributes/team;
         set cry_minn_el_ranged_attack1
t=0.300  refresh live target position when valid; pay 3.0s cooldown once;
         create one Ability_Fireball projectile, attach trail, start tracking
t=0.450  release actor (projectile continues independently)
t=3.300  cooldown matures
```

The projectile speed is `10`, its travel budget is `50` (not activation range
15), and rank-one damage is `1..4` at accepted collision. The actor has a
`2.85s` cooldown-only window after release. Target loss before launch falls
back to the stored target position; the ordinary projectile does not home.

### TutorialBasicDiseased: TutorialPoisonCloud

```text
t=0.000  accept target in 8 range; snapshot attributes/team;
         set ver_minn_lf_diseased_attack1
t=0.170  refresh/validate launch state; pay 3.0s cooldown once;
         create projectile at caster footprint, attach trail, start tracking
t=1.860  release actor (projectile may still be in flight)
t=3.170  cooldown matures
```

The projectile speed is `6`, travel budget `12`, impact radius `1`, and damage
`1..4`. Unobstructed maximum travel time is `2s`, so it may outlive the ability
release. The actor has a `1.31s` cooldown-only window after release.

### TutorialSpecialOne: BurstShot

```text
t=0.000  accept target in rank-one range 13; snapshot attributes/team;
         set cast_burstshot
t=1.060  refresh target; pay 1.0s cooldown once; launch projectile 1
t=1.460  refresh/turn toward live target; launch projectile 2
t=1.860  refresh/turn toward live target; launch projectile 3
t=2.060  cooldown matures while the ability still owns the actor
t=2.260  refresh/turn toward live target; launch projectile 4
t=2.900  release actor; next activation may occur on the next eligible tick
```

Every shot creates a distinct `Ability_Fireball` object with speed `25`, travel
budget `50`, and independent collision thread. Rank-one damage is `2..7` per
accepted collision. Later launches track the live target between shots, but
already launched projectiles do not retarget. Here release, not cooldown, is
the final gate: the earliest possible repeat is after `t=2.9s`, not `t=2.06s`.

## Concurrency and ordering

Native `sub_9E0AE0` allocates an ability instance from the shared 2,048-entry
pool, records its agent/target/timestamps, links it to **that agent's** active
ability collection, creates a Lua coroutine, and starts callback slot 2.
Cooldown and blocking checks in `sub_9E0660` resolve against the requesting
agent. Behavior callback wrapper `sub_A68F10` similarly uses the object's own
Lua thread at object offset `+676`; the agent brain is object `+684` and its
blackboard is object `+688`.

Therefore, for a Poison + Diseased + Ranged + Sloth + Special mixed wave, all
five may be in wind-up or projectile flight at once. The only serialization is
implementation-level processing order within one simulation tick and shared
resources such as the simulator RNG/object allocator. Consequences:

- one actor's cooldown never delays another actor;
- release blocks only the casting actor;
- same-tick damage is applied in scheduler/collision completion order;
- Burst's four projectile completions need not remain launch-ordered;
- a target killed by an earlier completion changes validation for later hits;
- object IDs and RNG draws can interleave across actors and projectiles.

No field in the packaged phase records, horde state, behavior tree, or ability
admission path represents a wave attack mutex, round-robin attacker index, or
global cooldown.

## Relevant build-103 replication packets

Packet emission is an adapter consequence of authoritative state changes; Lua
waits, cooldown payment, target selection, and collision waits are not packets.
The client binary retains receivers and some local construction paths, but the
retail server sender/dirty-flush owner is absent. The following is the strongest
allowed-artifact contract.

### Creature navigation and stop

- `ObjectPlayerMove` wire `0x91` carries object ID, goal flags, and goal XYZ.
  The repository's live-proven active-goal shape is 81 bytes including opcode:
  object ID at bytes `1..4`, flags `0x01` at `5..8`, goal XYZ at `9..20`, then
  sixty zero bytes.
- `LocomotionDataUnreliableUpdate` wire `0x95` is 17 bytes including opcode:
  object ID plus `mPartialGoalPosition` XYZ. It is a goal correction, not a
  ballistic snapshot.
- The live-proven walking-NPC transition is ordered `0x91(flags=0x01)` then
  `0x95` with the same goal. It visibly moves a build-103 client actor.
- A stopped actor's known `ObjectPlayerMove` state uses flags `0x20` and its
  stationary goal. `SpawnModifier`, invisible/passive deactivation, and
  Sloth's idle branch call native `nLocomotion.Stop`; the exact retail-server
  decision to send a stopped `0x91`, and whether it also flushes `0x95`, is not
  present in the client and is not asserted here.
- Sloth strafing uses the same ordinary actor navigation component: active
  goal plus facing-target locomotion. There is no strafe-specific wire opcode.

### Ability and projectile presentation

- `SetAnimationState` is wire `0xa5`, 25 bytes: object ID, animation-state
  GUID, 64-bit simulation timestamp, overlay byte, scale, and source/echo.
  Attack animations and both beam-in animations use this family.
- Projectile launch is ordered by construction as `0x8c ObjectCreate`, `0x9b`
  attached trail, then reliable reflected `0x94 LocomotionDataUpdate`.
  `0x94` carries the projectile parameter block, direction, range, and optional
  collision prediction. Never substitute `0x95` for projectile startup.
- Direct damage invokes sparse `0xba CombatEvent`; HP component mutation is
  `0x97 CombatantDataUpdate` (HP-only mask `0x01`). The client does not retain
  the retail dirty-component flush point, so exact `0x97` placement is open.
- Poison melee's accepted-hit order is `0xba` damage, then object-bound `0x9b`
  hit effect. Projectile direct hits emit positioned `0x9b`, then `0xba`, then
  a second positioned `0x9b` when accepted-damage fallback uses the same impact
  effect. Object deletion follows the object-manager sweep; its exact packet
  tick is not retained.

Application-message order is recoverable where the Lua/native call order is
shown. Outer RakNet reliability, datagram batching, and the original server's
component-flush cadence are not recoverable from `content.db`, packages, or
the client receiver and must remain capture/server-binary questions.

## Recovered state machine

```text
CREATED
  -> INVISIBLE / SPAWN MODIFIER (Stop; spawn modifier owns exact 0.5s wait)
  -> TARGET LATCHED [missing server-only horde owner; no authored radius]
  -> FIRST AGGRO (old invisible leaf cleans up, beam-in owns 1.291667s)
  -> PHASE READY (one deterministic ability)
       -> OUT OF RANGE: PURSUE target to activation envelope
       -> IN RANGE + COOLDOWN READY: finish chase goal/FACE, create ability instance
            -> WIND-UP
            -> HIT/LAUNCH: pay cooldown once
            -> RELEASE: free this actor
                 -> cooldown running: COMBAT IDLE / Sloth STRAFE-OR-IDLE
                 -> target left range: PURSUE
                 -> ready and in range: next activation on eligible tick
       -> no valid target: passive/fallback behavior
  -> DEATH/CANCEL: per-object behavior and ability teardown
```

The labeled transitions and authored deadlines are exact. The three unlabeled
server-owner scheduling quantities—spawn-to-target latency, retry tick period,
and the precise relative ordering of SpawnModifier versus first aggro—are not
present in the allowed build-103 artifacts. Assigning fixed values to them
would be reconstruction, not recovery.
