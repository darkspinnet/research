# 1-1 enemy runtime baselines (build 103)

## Result

At authored difficulty `1..24`, the `zelems_1` director's `minion` pool has
one eligible noun, `ZelemBasicRanged.Noun`. It is therefore the content
baseline for a Wanderer member and for any Spike member drawn from `minion`.
The low-band `captain` pool has three eligible nouns:
`ZelemSpecialHaster.Noun`, `NomadSnipe.Noun`, and
`NomadWithDrone.Noun`.

The noun/class resources recover exact **unscaled base** combat resources and
attributes. Native combatant initialization fills current HP from attribute
`4` (`MaxHealth`) and current mana from attribute `5` (`MaxMana`), so a fresh,
unmodified actor starts full:

| Runtime role if selected | Noun | HP / MaxHealth | Mana / MaxMana | STR / DEX / MIND | PDEF / EDEF / CRTR | Noun graphics scale | Footprint |
| --- | --- | ---: | ---: | ---: | ---: | ---: | ---: |
| Wanderer minion or Spike minion | `ZelemBasicRanged.Noun` | `20 / 20` | `75 / 75` | `10 / 10 / 10` | `60 / 60 / 45` | `1.30` | `0.650` |
| low-band Cannonator fallback | `ZelemBasicHybrid.Noun` | `22 / 22` | `75 / 75` | `10 / 10 / 10` | `60 / 60 / 45` | `1.25` | `0.625` |
| low-band Reparatron fallback | `ZelemBasicRepair.Noun` | `18 / 18` | `75 / 75` | `10 / 10 / 10` | `60 / 60 / 45` | `1.10` | `0.550` |
| eligible Spike captain | `ZelemSpecialHaster.Noun` | `100 / 100` | `75 / 75` | `10 / 10 / 10` | `60 / 60 / 45` | `1.85` | `0.925` |
| eligible Spike captain | `NomadSnipe.Noun` | `60 / 60` | `100 / 100` | `10 / 10 / 10` | `60 / 60 / 45` | `1.85` | `0.925` |
| eligible Spike captain | `NomadWithDrone.Noun` | `110 / 110` | `100 / 100` | `10 / 10 / 10` | `60 / 60 / 45` | `1.90` | `0.950` |

These are not captured retail 1-1 post-scaling values. The client/content
evidence does not recover the authoritative difficulty, captain, affix, or
director transform applied before replication. Until that policy is recovered,
the table is the exact noun-derived input and the conservative no-transform
baseline, not proof that retail always published those final maxima.

When the authoritative sender publishes full component snapshots, the
required application dependency order is:

```text
authoritative owner chooses noun/position/object ID and commits runtime state
  -> 0x8c ObjectCreate
  -> 0x97 CombatantDataUpdate and 0x96 AttributeDataUpdate baselines
  -> optional object/visibility state
  -> 0x99 AgentBlackboardUpdate when a target is assigned
  -> 0x91 locomotion/turn; if walking, immediately followed by 0x95 goal
  -> first-aggro presentation or ordinary ability-result packets
```

Native noun construction already supplies the unmodified local content state;
the retained client does not prove that retail always resent equal `0x97` and
`0x96` baselines. When those snapshots are sent, `0x8c` must precede them and
every other packet that names the new object. Target state must
exist before a target-consuming turn, pursuit, first-aggro action, or combat
ability. No separate "enable AI" opcode exists. Ordering between `0x97` and
`0x96`, director-state dirties, visibility, spawn modifiers, and the first
blackboard update is not sender-proven; only the causal dependencies above are.

## Provenance and scope

The authoritative normalized source is
`bin/darkspinner/darkspin/cache/content.db`. Its current manifest is
`source_build=103`, `content_release=build-103-content`, recipe `35`, input
fingerprint
`b088d6767e4485d90ff108c4eb091dd729b5cc8604b99b459ccc94cd1223b3ec`.
The canonical native source is `bin/game/GameBin/Game.c`, SHA-256
`1fac9cda954e076446889376d5e3e89859ce058b75c2d50739d4aedfd662b1eb`.

Focused decoded AI and non-player-class payloads are retained under
`bin/game/logs/1-1-enemy-runtime`. This note also incorporates the pool and
noun identities already established in `notes/campaign/1-1/director.md`,
`notes/campaign/1-1/nouns.md`, and the generic receiver contract in
`notes/campaign/spawn.md`.

This scope deliberately excludes `ZelemSpecialHaster_Captain.Noun`. That noun
is the low-band `special` pool entry; the reflected low-band `captain` pool is
the three-noun set above. Content eligibility alone does not prove that a Spike
contains one captain, that it contains minions, or which captain is selected.

## Exact content baselines

### Combat resources and maximum attributes

The base-instance join is exact. The noun instance is shared by its fixed
88-byte type-`0x474940A5` stat record. `content.db.non_player_class` projects
the native-backed fields without substituting tutorial constants:

| Noun instance | Stat resource | HP | Power | STR | DEX | MIND | Dodge | Resist | Critical |
| --- | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| `0x8F291AF3` (`ZelemBasicRanged`) | `9352` | `20` | `75` | `10` | `10` | `10` | `60` | `60` | `45` |
| `0xBDA683F8` (`ZelemBasicHybrid`) | `11837` | `22` | `75` | `10` | `10` | `10` | `60` | `60` | `45` |
| `0xA63F868D` (`ZelemBasicRepair`) | `3683` | `18` | `75` | `10` | `10` | `10` | `60` | `60` | `45` |
| `0xA1FDCCD8` (`ZelemSpecialHaster`) | `6403` | `100` | `75` | `10` | `10` | `10` | `60` | `60` | `45` |
| `0x1DDB0187` (`NomadSnipe`) | `792` | `60` | `100` | `10` | `10` | `10` | `60` | `60` | `45` |
| `0xB366BC74` (`NomadWithDrone`) | `1474` | `110` | `100` | `10` | `10` | `10` | `60` | `60` | `45` |

The corresponding sparse `0x96 AttributeDataUpdate` baseline, if no
authoritative transform is applied, is therefore:

```text
index 0  Strength          = 10
index 1  Dexterity         = 10
index 2  Mind              = 10
index 4  MaxHealth         = noun HP column
index 5  MaxMana           = noun Power column
index 7  PhysicalDefense   = 60
index 9  EnergyDefense     = 60
index 10 CriticalRating    = 45
```

There is no evidence for copying the current tutorial enemy's local
`CriticalRating=5`, `NonCombatSpeed=3`, or `CombatSpeed=4.5` constants into
campaign. The class records prove index `10=45`; speed indices `11` and `12`
are not projected from these records and remain unresolved for a complete
campaign `0x96` snapshot.

Native `sub_9D8E60` (`0x009D8E60`) initializes a combatant after its attribute
component exists. It calls the attribute initializer, reads selector `4`, uses
that value as current HP (falling back to `1.0` only if the selector is zero),
and reads selector `5` as current mana. This proves the full-resource initial
state represented by `0x97` mask `0x03`; it is not merely a local sender
convention.

### Noun scale, collision, and locomotion profile

All four nouns author:

- geometry `builtins!sphere`;
- physics property `DefaultPhysics.prop`;
- locomotion/profile string `npcCreature`;
- the graphics scale and resulting `0.5 * graphicsScale` footprint in the
  result table.

These values belong to the noun. They must be kept separate from the generic
object-create multiplier. `sub_A1DB90` initializes `cGameObjectCreateData`
with object `scale=1.0`, `team=0`, `hasCollision=true`, and
`playerControlled=false`. Thus object scale `1.0` preserves the authored noun
scale; sending `1.30`, `1.85`, or `1.90` as the object multiplier would apply
the noun scale a second time.

`hasCollision=true` plus the packaged sphere/physics data is the exact native
default and content-compatible baseline. The nouns do not author an enemy team
number. `team=0` is only the generic create-data default, not proof of the
retail hostile team. The authoritative server must choose a team consistent
with hostility and copy it to projectiles/abilities; the retail 1-1 value is
unresolved. The same boundary applies to owner ID, movement type, source marker
ID, and any non-default collision override in the trailing `sporelabsObject`
reflection.

`npcCreature` proves that these nouns receive ordinary creature locomotion; it
does not author an initial navigation goal. Native locomotion construction
starts without a target or walking goal. A stopped actor therefore needs no
invented `0x95`; when authority starts a walk it publishes the `0x91`
goal/flags state followed immediately by the fixed `0x95` partial-goal update.
An accepted `faceTarget=true` ability instead uses the target-facing `0x91`
shape and no `0x95`.

## Blackboard and target state

Native `sub_9E4310` (`0x009E4310`) constructs the object-local agent
blackboard, and `sub_9E4580` attaches it to game object offset `+688`. The
fresh state relevant to the five-field `0x99 AgentBlackboardUpdate` is:

```text
target ID       = 0 / no best target
isInCombat      = false
stealth byte    = 0
isTargetable    = true
attacker count  = 0
first aggro     = not consumed
```

The constructor explicitly clears the low state bytes and attacker/target
bookkeeping, sets byte `+1400` (the targetable state) to `1`, and leaves the
threat collections empty. A zero-target `0x99` is therefore a valid full
snapshot but is not required to establish the client object's native defaults.

All four linked non-player-class payloads author:

```text
aggroRange +0x38 = 0.0
alertRange +0x3c = 0.0
```

Native perception uses `max(aggroRange, alertRange)` and its point test is
strict. These actors cannot acquire a remote player from a positive
content-authored perception radius. The missing authoritative
director/AI owner must insert a hostile player or issue an explicit aggro
stimulus per object. Target selection, threat amount, multiplayer choice,
retargeting, and the insertion time are server policy. Once chosen, the
replicated `0x99` must precede any client-visible command that consumes that
target.

## Noun-specific AI startup

The four AI definitions all select phase zero initially and set
`faceTarget=true`, but their pre-aggro behavior is not uniform:

| Noun | Initial/idle authored state | First-aggro authored state | Phase abilities |
| --- | --- | --- | --- |
| `ZelemBasicRanged` | `preAggroIdle=nBehavior_Invisible`; `passiveIdle=nBehavior_Idle` | `firstAggroAbility=FirstAggro_Anim`; `firstAlertAbility=FirstAggro_FaceTarget` | `ZelemBasicRanged_Blink` |
| `ZelemSpecialHaster` | `preAggroIdle=nBehavior_Wander`; `passiveIdle=nBehavior_Idle` | no authored first-aggro string | `CastZelemHasteBuff`, `ZelemHasterAttack` |
| `NomadSnipe` | `preAggroIdle=nBehavior_Invisible`; alternate `preAggroIdle2=nBehavior_Wander` | `FirstAggro_BeamIn`; alternate `FirstAggro_FaceTarget`; first alert also `FirstAggro_BeamIn` | `NomadSnipe_Slow`, `NomadSnipe_Melee` |
| `NomadWithDrone` | `preAggroIdle=nBehavior_Idle`; `combatIdle=nBehavior_Idle` | `FirstAggro_ActivateRobot`; `SubsequentAggro_ActivateRobot`; additional `NomadWithDronePassive` hook | `NomadWithDronePunch` |

The exact strings and fixed slots come from decoded resources `9355`, `6406`,
`789`, and `1477`. The zero-radius result comes from resources `9351`, `6402`,
`793`, and `1473`.

AI activation is a simulator composition, not a packet:

1. noun construction attaches attributes, combatant, locomotion, blackboard,
   and the noun-linked AI definition;
2. the AI initializer resolves the phase array and leaves phase zero selected;
3. the behavior owner starts the applicable pre-aggro/passive leaf;
4. authoritative target insertion makes first aggro and then combat selection
   eligible;
5. native ability admission checks target legality, actor state, cooldown,
   mana, range, hit predicates, and blocking modifiers before an ability
   instance exists.

For `ZelemBasicRanged` and `NomadSnipe`, starting
`nBehavior_Invisible` also means native behavior will make the object invisible
and add `Intangible=1` and `InvisibleToSecurityTeleporters=1`; replacement
deactivates that leaf and removes both scoped attributes before first aggro.
This is noun-specific AI behavior, not a universal Wanderer/Spike spawn rule.
`ZelemSpecialHaster` can wander before aggro, while `NomadWithDrone` begins in
idle and owns its robot-specific aggro presentation.

No retained authoritative body proves when those behavior effects are flushed
relative to `0x97`, `0x96`, `0x99`, or an optional `SpawnModifier`. In
particular, do not apply the tutorial's `FirstAggro_BeamIn_Tutorial` delay or
its local spawn-modifier composition to these campaign nouns.

## Packet ordering: required facts versus missing sender policy

### Required by the build-103 receivers and consumers

1. **Create first.** `0x8c ObjectCreate` carries object ID, noun, placement,
   generic create fields, and a `sporelabsObject` tail. Every subsequent
   component packet resolves that ID, so no component, blackboard, locomotion,
   modifier, animation, effect, or ability packet may precede it.
2. **Publish resource state before combat consumes it.** `0x97` carries current
   HP/mana and `0x96` carries maximum/tuning attributes. Both follow `0x8c` and
   precede any replicated combat outcome that depends on them. The client
   receiver does not impose an order between these two baseline packets.
3. **Publish a target before target-dependent action.** A nonzero-target
   `0x99` must precede a target-facing `0x91`, first-aggro/attack animation, or
   other action packet whose presentation resolves that target.
4. **Keep locomotion pairs ordered.** An active creature walking goal is
   `0x91` followed immediately by `0x95` for the same partial goal. A stationary
   target-facing ability turn is a single flags-`0x42` `0x91` and has no
   `0x95`.
5. **Let ability execution publish consequences.** Animation, modifier,
   projectile, effect, combat-event, and later HP/attribute deltas follow the
   accepted simulator action; they are not AI-enable messages.

### Not recovered from the client/content artifacts

- whether the director sends `0x8b` state before or after the agents, and the
  correct active-wave field/container encoding;
- the final `0x8c` team, owner, movement type, source marker, rotation, and
  trailing dirty-field set;
- the exact `0x96` sparse field set beyond the proven content attributes,
  especially creature speed and any difficulty/affix additions;
- whether a zero-target `0x99` baseline is sent or omitted as equal to native
  construction state;
- the relative order of `0x97` versus `0x96`, visibility/invisibility deltas,
  blackboard target injection, and any `SpawnModifier` lifecycle;
- first target, threat weight, pursuit goal/inset, navigation retry cadence,
  spawn-to-first-think timing, and late-join replay;
- RakNet reliability/channel, application batching, and datagram flush
  boundaries.

## Role and policy boundary

The strongest safe runtime interpretation is:

```text
Wanderer selected at difficulty 1..24
  -> zero or more ZelemBasicRanged actors, count/locus chosen by authority

Spike selected at difficulty 1..24
  -> composition unresolved
  -> each minion drawn from minion uses ZelemBasicRanged baseline
  -> each captain drawn from captain uses one of Haster/Snipe/WithDrone baselines
```

The shipped client and content do not prove that every Spike has exactly one
captain, how many minions accompany it, how challenge becomes a group budget,
or which eligible captain is selected. Existing local planning that emits one
captain followed by minions is an explicit conservative server policy. It must
not be cited as recovered retail ordering.

Similarly, the exact base values in this note are sufficient to construct a
receiver-valid no-transform baseline, but a retail/server capture or recovered
authoritative director body is still required to close difficulty scaling,
hostile team, target ownership, initial dirty masks, and the complete send
order.
