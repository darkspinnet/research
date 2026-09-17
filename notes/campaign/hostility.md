# Campaign hostility and target injection (build 103)

## Result

Build 103 exposes a complete client-side representation for enemy creation,
blackboard target state, threat insertion, best-target filtering, and
ability-range pursuit, but it does not retain the authoritative campaign
director body that joins those mechanisms for an ordinary 1-1 population
spawn. The strongest supported contract is therefore:

```text
create a non-player actor with a hostile team relationship
  -> commit combat/attribute/object state
  -> insert one or more eligible player-controlled objects into that actor's
     server-only threat list
  -> choose the first currently valid hostile/targetable threat candidate
  -> replicate the selected target with 0x99
  -> pursue the target object to the selected ability's footprint-expanded
     activation envelope
  -> replicate the target before any pursuit, turn, or ability that consumes it
```

The best receiver-valid placeholder is an unowned, ordinary-locomotion enemy
on team `0`, against player team `1`, with no explicit movement-type tail
field. Those are conservative defaults, not recovered retail 1-1 sender
values. For target injection, the closest native analogue is the retained
spawn helper's `5.0`, type-`2`, `"Spawn Aggro"` insertion against every live
player-controlled object. That exact insertion is proven for
`ActivateMinionSpawn` / `ActivateLieutenantSpawn`, not for the missing
campaign director, so it is a clearly labeled compatibility fallback.

There is no evidence for a universal pursuit distance. Pursuit stops at the
chosen ability's authored range expanded by the attacker and target footprint
radii. Hero switching or target death must cause a fresh best-target
evaluation; it must not leave a corpse or undeployed hero in the replicated
target field. The exact retail campaign timing, multiplayer ranking, threat
weight, and dirty-flush cadence remain open.

## Evidence classes and provenance

Conclusions below use four explicit evidence classes:

1. **Native proof**: behavior visible in
   `bin/game/GameBin/Game.c`, SHA-256
   `1fac9cda954e076446889376d5e3e89859ce058b75c2d50739d4aedfd662b1eb`.
2. **Content proof**: build-103 nouns, `NonPlayerClass`, AI definitions, and
   Lua bytecode in `bin/darkspinner/darkspin/cache/content.db`, with focused
   extracts under `bin/game/logs`.
3. **Video-derived behavioral evidence**: the visible 1-1 run in
   `bin/video/walkthrough/1-1/1-1.mkv`, its extracted frames/contact sheets,
   and `bin/video/walkthrough/1-1/info.md`.
4. **Fallback**: a conservative server-authority choice made only where the
   first three classes do not recover retail policy.

The video is a 15:30.241, 1920x1080 recording with SHA-256
`363f22ac2dc1d541623c55103505df42d651763b193a0285a8d40917a8831fda`.
It is useful only for visible behavior. Edited timestamps, one run's enemy
counts, mutation rolls, selected difficulty, drops, and rewards are not exact
server-authority evidence. The recording and its coarse five-second contact
sheets also do not expose packet bytes or hidden blackboard fields.

The principal native locations are:

| Boundary | Native evidence |
| --- | --- |
| `ObjectCreate` receiver | `sub_53A760`, `0x0053a760`, C lines `381130-381187` |
| create-data defaults | `sub_A1DB90`, `0x00a1db90`, C lines `1446574-1446595` |
| object-tail reflection | `sub_A1E710`, `0x00a1e710`, C lines `1447101-1447169` |
| blackboard constructor | `sub_9E4310`, `0x009e4310`, C lines `1399367-1399402` |
| blackboard attachment | `sub_9E4580`, `0x009e4580`, C lines `1399502-1399540` |
| explicit target setter/getter | `sub_9E3DC0` / `sub_9E3DE0`, C lines `1399060-1399085` |
| threat insertion | `sub_9E4640`, `0x009e4640`, C lines `1399543-1399672` |
| alternate fixed-5 threat insertion | `sub_9E4870`, C lines `1399678-1399756` |
| best-target selection | `sub_9E3E50`, `0x009e3e50`, C lines `1399119-1399169` |
| Lua `GetBestTarget` bridge | `sub_A00D80`, C lines `1423867-1423887` |
| native spawned-NPC aggro loop | `sub_9FEBC0`, C lines `1422449-1422560` |
| ability admission | `sub_9E0660`, C lines `1396033-1396242` |
| object-follow locomotion | `sub_A00430` and core goal `sub_A149C0`, recovered in `notes/tutorial/sloth-tailzap.md` |
| ordinary death target clear | `SetTargetID` bridge `sub_A023D0` -> `sub_9E3DC0`, recovered in `notes/combat/death.md` |

## `ObjectCreate`: team, owner, and movement type

### Exact native and content facts

`cGameObjectCreateData` has ten fixed reflected fields. `sub_A1DB90`
initializes team `0`, collision enabled, player-controlled false, scale `1`,
and every other scalar/vector field to zero. The fixed all-fields image is
followed by a separate 23-field `sporelabsObject` reflection. Its relevant
tail indices are:

| Tail field | Meaning |
| ---: | --- |
| `0` | team |
| `18` | owner object ID |
| `19` | movement type |
| `22` | source marker ID |

The generic default is not proof of retail campaign policy. It does prove that
zero is the receiver's construction baseline when a field is omitted. If tail
field `0` is sent, it must agree with the fixed create-data team; publishing
contradictory teams would create a transient or persistent hostility mismatch.

The four currently eligible low-band 1-1 enemy nouns recovered in
`notes/campaign/1-1/enemy-runtime.md` use the `npcCreature` locomotion profile and do
not author a team, owner, or initial movement goal. `npcCreature` proves that
they are ordinary creature locomotion actors; it does not assign tail movement
type `3` (homing projectile) or `4` (lob). Those nonzero values are explicitly
set by projectile/lob paths and must not be copied onto a walking NPC.

`sub_9FEBC0` creates an NPC first and only afterward injects threat. No owner or
movement-type assignment appears between creation and the aggro loop. This is
strong evidence against inventing a player owner merely because the enemy is
targeting that player, although the helper is not the missing campaign sender.

### Strongest safe placeholder

| Field | Placeholder | Evidence boundary |
| --- | ---: | --- |
| enemy team | `0` | Native create default; local build-103 heroes use team `1`; native hostility predicates accept opposing teams. Retail 1-1 team was not captured. |
| owner ID | `0` / omit tail field `18` | Generic default and spawned-NPC helper show no reason to make a campaign enemy player-owned. A future capture may prove a director/marker owner. |
| movement type | `0` / omit tail field `19` | Ordinary `npcCreature` walking uses locomotion goals, not projectile/lob movement types. |

This fallback is receiver-valid and preserves hostility if players are team
`1`. It is not a claim that the retail server always encoded team zero or
omitted the corresponding tail field. The current 81-byte enemy create shape
(`0x8c`, fixed team zero, tail position and orientation only) is compatible
evidence, not a retail capture.

## Threat insertion and target choice

### Fresh blackboard state

`sub_9E4310` creates a blackboard with empty threat and reciprocal-attacker
collections. It explicitly initializes the state relevant to `0x99` as:

```text
selected target ID = 0
isInCombat         = false
stealth            = 0
isTargetable       = true
attacker count     = 0
first aggro        = not consumed
```

Therefore a full zero-target `0x99` is semantically valid, but it is optional
when all fields equal the client's native construction defaults. No evidence
requires it immediately after `ObjectCreate`.

### Exact threat behavior

`sub_9E4640(blackboard, targetID, amount, flags, reason)` is additive:

- it adjusts the supplied amount using target-side attributes `86` and `87`;
- if the target already exists in the threat vector, it adds to that entry;
- otherwise it appends `(targetID, adjustedAmount)`;
- for a first qualifying insertion it posts stimulus `0x20` for ten seconds
  (or stimulus `0x200` for the alternate flag family) and consumes the
  first-aggro latch at blackboard byte `+1365`;
- it adds the aggressor object's ID to the target blackboard's reciprocal
  attacker collection at `+1084..+1092`.

The reason string is diagnostic; it is not replicated in `0x99`.

`sub_9FEBC0` is the strongest exact spawn analogue. When its `shouldAggro`
argument is true, it enumerates every live player-controlled object and calls:

```text
sub_9E4640(newEnemy.blackboard, playerObjectID, 5.0, 2, "Spawn Aggro")
```

The wrappers are `ActivateMinionSpawn` and `ActivateLieutenantSpawn`. Their
packaged caller uses this path for portal children and immediately suppresses
ordinary first aggro, so this does not prove ordinary 1-1 director timing or
presentation. It does prove a native build-103 policy vocabulary: per-enemy,
per-live-player insertion, amount `5.0`, threat type/flags `2`, after object
creation.

All four focused 1-1 `NonPlayerClass` records author `aggroRange=0.0` and
`alertRange=0.0`. Native perception uses their maximum and a strict point
test. Consequently these actors cannot acquire a remote hero through a
positive authored perception radius. A campaign/director owner must inject
threat or issue another explicit aggro stimulus; polling a fabricated radius
is not content parity.

The linked AI definitions also prove that target use is object-local and that
activation cannot be reduced to one generic campaign animation:

| Noun | Pre-aggro/passive evidence | First-aggro evidence | Phase-zero abilities |
| --- | --- | --- | --- |
| `ZelemBasicRanged` | invisible, then idle | `FirstAggro_Anim`; alert uses `FirstAggro_FaceTarget` | `ZelemBasicRanged_Blink` |
| `ZelemSpecialHaster` | wander, then idle | no authored first-aggro string | `CastZelemHasteBuff`, `ZelemHasterAttack` |
| `NomadSnipe` | invisible, alternate wander | `FirstAggro_BeamIn`, alternate/alert face-target paths | `NomadSnipe_Slow`, `NomadSnipe_Melee` |
| `NomadWithDrone` | idle; combat idle authored | `FirstAggro_ActivateRobot`, then `SubsequentAggro_ActivateRobot` | `NomadWithDronePunch` plus its passive robot hook |

Every definition sets `faceTarget=true` and begins in phase zero. Invisible
behavior removes its scoped intangible/invisible-to-security attributes when
replaced. The exact strings and slots are content proof; the authoritative
selector timing and the order of visibility cleanup versus `0x99` are not.

### Best-target semantics

`sub_9E3E50` scans the actor's threat vector in stored order and returns the
first candidate that still resolves and passes the native eligibility gates,
including candidate blackboard existence, `isTargetable`, and the hostile/live
validation reached through `sub_9D9540`. It returns zero when none survive.
The stored threat amount is not used as a highest-number-wins comparison in
this selector. Adding more threat to an existing earlier entry therefore does
not by itself move a later candidate ahead of it.

This is the strongest evidence for retarget ordering: insertion order plus
current validity, not nearest distance, round robin, or largest threat. The
authoritative campaign producer may still control ranking by choosing insertion
order or pruning/reinserting entries; that producer is absent.

### Safe campaign fallback

For a playable single-player campaign spawn:

1. Insert each currently live, player-controlled squad object in stable squad
   order using `5.0`, type `2`, `"Spawn Aggro"`.
2. Let native-equivalent validity choose the deployed, targetable, hostile
   object. Do not treat an undeployed squad member as attackable merely because
   it exists in the object table.
3. If the campaign runtime represents only the deployed hero as eligible,
   inserting only that hero is an acceptable reduced fallback, but the threat
   list must be rebuilt when deployment changes.

The amount, type, reason, and squad-order rule are fallback policy for ordinary
campaign population. The first three numeric/string values are borrowed from
the exact native spawn helper; squad order is only deterministic tie policy.
They require a retail server trace or recovered director body before being
called exact 1-1 values.

## `0x99` target, combat, and attacker count

The full build-103 application message is 16 bytes:

```text
99
<objectID:u32-le>
1f
<targetID:u32-le>
<isInCombat:u8>
<stealth:u8>
<isTargetable:u8>
<attackerCount:u32-le>
```

The five fields are a blackboard snapshot, not a command to add threat. The
server must mutate the reciprocal threat graph first and then derive the
replicated values.

### Values by state

| Object/state | target ID | in combat | attacker count | Evidence |
| --- | ---: | --- | ---: | --- |
| fresh enemy before injection | `0` | `false` | `0` | Exact constructor baseline. |
| activated enemy with valid selected hero | selected hero ID | `true` once the combat selector is active | normally `0` | Target follows `GetBestTarget`; the enemy's attacker count describes actors threatening the enemy, not the number of heroes in its own threat list. The exact threat-to-combat transition flush is missing. |
| hero threatened by `N` live enemies | independent/usually unchanged by this operation | existing hero combat state | `N` | Threat insertion appends each aggressor to the target hero's reciprocal attacker collection. |
| enemy with no valid candidate | `0` | `false` | preserve its own reciprocal count | Safe derived state; exact retail clear flush is not sender-proven. |

Threat insertion itself does not directly write the explicit target slot at
blackboard `+1360`; `SetTargetID` is a separate mutation, and
`GetBestTarget` derives a candidate from the threat vector. Likewise,
`sub_9E4640` consumes first aggro but does not visibly write the blackboard's
`isInCombat` byte in its retained body. It is therefore too strong to claim
that one native call atomically produces target ID plus `isInCombat=true`.

The safe publication rule is:

- keep `isInCombat=false` while the enemy is merely created/hidden/idle;
- publish the selected nonzero target with `isInCombat=true` when authority
  releases it into first aggro or combat selection;
- publish target zero and `isInCombat=false` when no valid hostile remains;
- compute attacker count from reciprocal live attacker links, never from the
  enemy's outgoing threat-entry count and never hard-code it to one.

During a single-enemy spawn this may require two `0x99` updates: one for the
enemy's selected target and one for the hero's incremented attacker count.
No retained sender proves their relative packet order. Commit both sides of
the graph before emitting either snapshot so a late failure cannot leave
one-sided authority.

## Pursuit stop distance

Ability admission (`sub_9E0660` and the range helper recovered in
`notes/tutorial/sloth-tailzap.md`) uses surface separation:

```text
surfaceGap = centerDistance - attackerFootprint - targetFootprint
admit when surfaceGap <= selectedAbilityRange
```

`MoveToObject(attacker, target, range)` reaches `sub_A00430`. It adds the
target footprint to the requested range, and core goal construction in
`sub_A149C0` adds the attacker footprint. The paired near-goal test uses a
strict comparison against the same sum. Thus the evidence-backed goal is:

```text
stopDistance = attackerFootprint + targetFootprint + selectedAbilityRange
```

This is a live object-follow goal. As the target moves, authority must update
against its current position rather than chase a stale injection-time point.
When admission first succeeds, ordinary chase completion stops the actor;
`faceTarget=true` then publishes a target-facing turn before the ability.

The 1-1 AI definitions contain different phase abilities and behavior leaves.
They do not justify one campaign-wide stop radius. Melee, ranged, buff, blink,
summon, or alternate phase choices must use the range of the ability actually
selected. A buff or self-only ability must not create a chase merely because
another phase ability is ranged.

Safe fallback when an ability has a recovered range is the exact formula
above with no invented inset. If an ability definition/range is unavailable,
the conservative placeholder is surface contact (`selectedAbilityRange=0`)
for navigation only and fail-closed ability admission; do not invent a broad
attack range. Retail may have used a small inset or navigation tolerance, but
no allowed artifact fixes it.

The walkthrough supports only the qualitative conclusion: visible 1-1 packs
close on the hero and different enemy families engage from visibly different
distances. The five-second contact sheets are too coarse for a numeric radius,
and perspective, footprints, animation wind-up, and player motion prevent a
reliable screen-space measurement. This behavioral observation is consistent
with ability-specific envelopes; it is not a source for exact stop distance.

## Retarget after hero switching or death

### Native/content proof

- `GetBestTarget` reevaluates the stored threat candidates and skips candidates
  that no longer resolve or fail targetable/hostile/live validation.
- AI movement, combat-idle, and ability selection read the actor-local best
  target again; they do not own one immutable wave-global target.
- ordinary death explicitly clears the dying object's own outgoing target via
  `SetTargetID(0)` before stopping locomotion. That does not directly rewrite
  every enemy pointing at the dying object; their next best-target evaluation
  supplies the replacement.
- object-follow pursuit tracks a target object and must be replaced or stopped
  when that target becomes invalid.

No retained build-103 campaign body proves how hero deployment changes
`isTargetable`, whether old threat is removed or merely becomes ineligible,
how quickly reevaluation occurs, or which surviving player wins in multiplayer.

### Video-derived conclusion and limit

The 1-1 recording visibly shows the three Q/W/E squad slots available and
continuous ordinary combat, but the reviewed MKV, extracted timestamp frames,
and contact sheets do not provide a clear controlled hero-switch-under-aggro
or hero-death transition. They therefore prove that squad switching is part of
the visible mission context, but they do **not** resolve target migration,
packet order, reevaluation latency, or multiplayer ranking. No one-run enemy
count or mutation sequence is used here.

### Safe retarget fallback

At a legal hero switch:

1. Atomically change deployment/targetability authority.
2. Reevaluate every live enemy's existing threat list.
3. If the newly deployed hero is absent, insert it with the same compatibility
   threat policy before selection; invalidate or remove the undeployed hero as
   an eligible target.
4. Publish a new enemy `0x99` before any locomotion goal, facing turn, or
   ability that consumes the new target.
5. Replace an active object-follow pursuit goal with one tracking the new
   object; do not preserve the old hero's position as a point goal.

At target death or deletion, perform the same reevaluation. Choose the first
remaining valid threat candidate. If none exists, publish target `0`, set
`isInCombat=false`, stop/cancel target-follow locomotion at the enemy's current
authoritative position, and schedule no further targeted action. If campaign
rules auto-deploy a surviving squad hero, wait until that deployment is
committed and then retarget to it; do not attack a corpse during its visible
death/fade lifetime.

This fallback preserves native selection semantics and playability. Immediate
same-tick reevaluation, squad insertion order, and the choice to remove versus
retain invalid threat entries are not recovered retail timing/policy.

## Packet ordering

### Initial spawn and activation

The strongest causal order is:

```text
server chooses and commits object ID, noun, transform, team, and runtime state
  -> 0x8c ObjectCreate
  -> 0x97 CombatantDataUpdate / 0x96 AttributeDataUpdate as needed
  -> optional 0x8d visibility/object state and noun-specific spawn presentation
  -> server-only threat insertion and reciprocal attacker-link commit
  -> 0x99 enemy selected-target/combat snapshot
  -> 0x99 hero attacker-count snapshot when that count changed
  -> first-aggro presentation and/or pursuit
  -> walking: 0x91 active goal immediately followed by 0x95 partial goal
  -> accepted target-facing ability: one 0x91 flags-0x42 turn, then animation,
     effect/projectile, combat event, and component deltas
```

Required facts:

- `0x8c` precedes every packet that names the new enemy object ID.
- a target object must already exist before a nonzero-target `0x99` names it;
- the nonzero-target `0x99` precedes a pursuit, facing turn, first-aggro action,
  or ability presentation that consumes that target;
- a walking `0x91`/`0x95` pair remains adjacent and ordered;
- the flags-`0x42` ability-facing turn has no appended `0x95`.

Open sender policy:

- `0x97` versus `0x96` order;
- visibility/spawn modifier versus target injection order;
- enemy target `0x99` versus hero attacker-count `0x99` order;
- whether a zero-target baseline `0x99` is omitted;
- director-state `0x8b`, batching, reliability/channel, and datagram boundaries.

### Retarget and target loss

```text
commit new deployment/death validity and reciprocal threat graph
  -> publish enemy 0x99 with new target (or zero/false)
  -> publish affected hero attacker-count 0x99 snapshots
  -> cancel/replace old target-follow goal
  -> if new target: 0x91 + 0x95 pursuit, or flags-0x42 turn on admission
  -> if no target: stationary stop only; no targeted action
```

Only the target-before-consumer dependency is native-required. The placement
of attacker-count snapshots relative to movement and the exact stop delta are
safe ordering policy pending a retail sender capture.

## Evidence-ranked decision table

| Question | Strongest conclusion | Confidence |
| --- | --- | --- |
| enemy team | Team is a fixed create field plus optional tail field; content does not author it. Team `0` versus player `1` is the safest default. | Native default exact; retail campaign value open. |
| owner | Threat target is not ownership. Ordinary spawned NPC helper performs no visible owner assignment. | Strong negative evidence; retail tail field open. |
| movement type | Ordinary `npcCreature` locomotion needs no nonzero movement type; `3`/`4` belong to homing/lob paths. | Strong native/content evidence. |
| initial `0x99` | Fresh state is target `0`, combat false, targetable true, attacker count `0`; equal baseline may be omitted. | Exact constructor; sender omission open. |
| active enemy `0x99` | Target is current `GetBestTarget`; combat true only after combat activation; enemy attacker count is its incoming reciprocal count, usually zero. | Target/count semantics strong; transition timing open. |
| initial threat | Explicit injection is required for zero-radius 1-1 actors. `5.0`, type `2`, `Spawn Aggro` per live player is the closest exact native spawn analogue. | Requirement exact; numeric campaign use fallback. |
| pursuit stop | Attacker footprint + target footprint + selected ability range, following the live target object. | Strong native/content evidence; inset/tolerance open. |
| hero switch/death | Reevaluate per-enemy best target, skip invalid target, publish `0x99` before new target-consuming motion/action, stop if none. | Strong mechanism; exact campaign cadence/ranking open. |
| video constraints | Visible families close and engage at differing distances; no clear switch-under-aggro/death case is available. | Behavioral only; no numeric or packet proof. |

## Remaining evidence needed

- a retail campaign server capture with decoded application payloads, especially
  `0x8c`, both sides of `0x99`, `0x91`, and `0x95`;
- the missing authoritative campaign director/horde activation body;
- a controlled 1-1 recording or trace that switches heroes while an enemy is
  pursuing and separately kills the active hero with a surviving squad member;
- multiplayer evidence for insertion/ranking, attacker counts, and target
  migration;
- late-join snapshots proving whether invalid threat entries are retained and
  whether baseline `0x99` is resent.

Until then, the fallbacks in this note are suitable compatibility policy, not
recovered retail constants.
