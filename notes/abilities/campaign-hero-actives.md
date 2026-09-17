# Campaign hero active/support runtimes

## Plasma Wreath

Blitz's `special_1` is `LightningRogueSupport`, shown as Plasma Wreath. Chunk
`462` (`Abilities/0x60AD4000.lua`, SHA-256
`44e6dafb1d60edfe2b86a650061dda129a4736fe656516b8ddc3907fdc20a2fe`)
proves a 15-second cooldown, `20 + primary power * 0.10` cost, 450 ms release,
self-targeted unique modifier request, and
`el_lightningravager_support` animation.

Modifier chunk `136` (`Modifiers/0x1A7E0FE5.lua`, SHA-256
`d151ba5e35c826c3fd9bdc7f7b5bb09a7d5be2f99014c09c0a5f0c6364221035`)
proves the rank-one six-charge shield, five-unit hostile scan, two-second
interval, elemental energy damage `8 + primary attribute * 0.05`, shield/orb
effects, chained strike beam, impact effect, unique activation, and cleanup on
agent death. The campaign authority implements that rank-one contract; the
unrecovered reserve-hero lifetime choice is recorded in `notes/help.md`.

## Result

The four rejected campaign requests do not map to the similarly named first
active/support fields by intuition. Runtime profile data, the build-103 content
database, and the registered Lua definitions establish this exact routing:

| Hero and live request | Slot | Exact `creature_template_ability.asset_name` | GUID | Root/register table | Runtime kind and target contract |
| --- | --- | --- | ---: | --- | --- |
| Blitz Alpha, index `3`, target `0`: Electron Sphere | `random` | `PlasmaRandom_LightningBall` | `0xD02C30E2` | chunk `696`; `nAbility_PlasmaRandom_LightningBall` registered as `PlasmaRandom_LightningBall` | projectile; cursor/point-directed, no target object required |
| Sage Alpha, index `3`, target `0`: Enrage | `random` | `CastEnrage` | `0x5E0C8BF6` | chunk `247`; `nAbility_Cast_Enrage` registered as `CastEnrage` | instant friendly buff/HoT; nearest friendly around point, then self fallback |
| Goliath Alpha, index `2`, target `0`: Shockwave | `special_2` | `EnergySentinelSupport` | `0x1C46A6FE` | chunk `105`; `nAbility_EnergySentinel_Support` registered as `EnergySentinelSupport` | untargeted melee-template frontal AoE; target object optional |
| Goliath Alpha, index `3`, target `0`: Zetawatt Beam | `random` | `TechRandom` | `0xCC1E09D0` | chunk `172`; `nAbility_Tech_Random` registered as `TechRandom` | cursor-directed line AoE; hostile target optional |

Profile localization must use the table attached to each ability locale key.
Goliath's four Energy Sentinel abilities share the creature table, while
`TechRandom` uses table `0x46a91a19`; applying the creature table to Zetawatt
Beam produces the client's missing-text `***` label.

This corrects two earlier naming mistakes. The third Test squad member is user
creature `4`, **Goliath Alpha**, noun `PC_TE_Tank2.Noun` (`0xC940B9DF`), not
creature `3` / Wraith Alpha. Also, Electron Sphere is not
`LightningRogueActive`: that asset localizes as Ride the Lightning. Goliath's
`EnergySentinelActive` is Arc Weld and belongs to `special_1`; it is investigated
below only because it exposes the constrained compiler's missing enum seed.

The live wire mapping is therefore `index 2 -> special_2` and
`index 3 -> random`. `special_1` is the squad/support boundary exposed through
the separate deck-position actions `6` through `8`, not a second ordinary
character action at index `2` or `3`.

## Evidence and provenance

The authoritative runtime user database returns creature `1` as Blitz Alpha,
creature `2` as Sage Alpha, and creature `4` as Goliath Alpha. Their fixed and
reflected campaign records carry nouns `0x6367B6CD`, `0x2CA50A9A`, and
`0xC940B9DF` respectively. The 2026-07-22 11:30 live trace independently shows
the four requests with exactly those nouns, indexes, and target `0`. References
to Wraith in that trace predate the database/profile correction and describe
Goliath's visible Energy Sentinel kit.

The three complete database slot maps are:

| Hero | `basic` | `special_1` | `special_2` | `random` | `passive` |
| --- | --- | --- | --- | --- | --- |
| Blitz | `LightningRogueBasic` | `LightningRogueSupport` | `LightningRogueActive` | `PlasmaRandom_LightningBall` | `LightningRoguePassive` |
| Sage | `SupportHealerBasic` | `SupportHealerSupport` | `TreeOfLife` | `CastEnrage` | `SupportHealerPassiveModifier` |
| Goliath | `EnergySentinelBasic` | `EnergySentinelActive` | `EnergySentinelSupport` | `TechRandom` | `EnergySentinelPassive` |

The exact extracted roots and shared bodies are retained under
`bin/game/logs` and have these immutable identities:

| Role | Chunk / resource | Indexed source | Decoded SHA-256 |
| --- | ---: | --- | --- |
| Electron Sphere root | `696` / `14267` | `Abilities/0x8866E21B.lua` | `e28b0a6e4574d0d179c4ac6b42f12afa1f375b8d3198a9ea5eec042e2e635687` |
| projectile template | `38` / `13559` | `Abilities/0xCE0FC9AA.lua` | `0cf624bee02973c7f6986f493f2120a5f1264987a105cee689dc844c2e5bb6c7` |
| Enrage root | `247` / `13787` | `Abilities/0xDEA4E48A.lua` | `a22a6a8c09cc483756106eb82a5909bf7469333774d5912e03f112df8260966d` |
| instant-cast template | `50` / `13571` | `Abilities/0xF76B6B7F.lua` | `daf011a759c793297ba89f4eab1bf0b7a7ad140bf16ea0cb9065b8d850aef664` |
| Enrage modifier | `716` / `14288` | `Modifiers/0xB96EFF67.lua` | `9b5a3b64f1fc814e9f164a9ef0ec2f21414a37127cb607c08ff7f75a84089783` |
| Shockwave root | `105` / `13629` | `Abilities/0x7A8B8A58.lua` | `6e63f156e7a864fffbab90ace9cbd08592749d5fa42cf9b28565d790f42d16b4` |
| melee template | `832` / `14409` | `Abilities/0x7BF2D7DD.lua` | `2951fef16876a72687c235cc93377ede9dcec31286a8237a9c6a8af978da778a` |
| Shockwave stun modifier | `605` / `14169` | `Modifiers/0xA555CCB4.lua` | `3e25c94e45472fde309dabacfc32a6a40ff34f95c4460cf6007a1a2a016feb34` |
| Zetawatt Beam root | `172` / `13705` | `Abilities/0x15072D5A.lua` | `2084a1cc3ba678077de0b4776972099ccdcc892cade754d1811fbed391e90410` |
| Arc Weld root | `966` / `14546` | `Abilities/0x902660D5.lua` | `d562b856b573ec412087a54ff20bf65da140c8639df889315085291b60dc5211` |
| `GlobalDefinitions` | `659` / `14227` | `lua/0x2E64AA9E.lua` | `25927418c849f13c1f45592387f0d9d0fd1e8b67b6537daf0ac8b57d663d57b3` |
| `Vector` | `439` / `13990` | `lua/0x0EE694B4.lua` | `320d85202f00868498c62e8f6a1efb798c018be7ded8027df27dbb772521d8a6` |
| `TargetUtils` | `406` / `13957` | `lua/0xBAE104D5.lua` | `664abf0bfb49f7c17e58fa20e89127ba60f63797d8ca4554217554d43a930582` |
| `Global` | `827` / `14404` | `lua/0x57572DAC.lua` | `59566e1528d3850efd8dc905aa6e12dd2cf113d06fa750bef35cbe82ed6a6126` |

The indexed dependency graph is:

```text
696 Electron Sphere
  -> [0] Abilities!template_ability_projectile.lua -> 38
       -> [0] Lua!Vector.lua -> 439

247 Enrage
  -> [0] Abilities!template_ability_instantcast.lua -> 50
  -> [1] Modifiers!modifier_enrage.lua -> 716

105 Shockwave
  -> [0] Abilities!template_ability_melee.lua -> 832
  -> [1] Modifiers!modifier_energysentinel_stun.lua -> 605
       832 -> 0x3681d755!Global.lua -> 827
           -> Lua!TargetUtils.lua -> 406
           -> Lua!Vector.lua -> 439

172 Zetawatt Beam
  -> no `require` dependency; it constructs a direct `Class.newClass` body

966 Arc Weld
  -> [0] Abilities!template_ability_melee.lua -> 832
  -> [1] Lua!TargetUtils.lua -> 406
```

These formerly unresolved database edges are linker aliases, not absent
content. Case-insensitive FNV-1 of the authored stems independently produces
the packaged source identities for `Vector` (`0x0EE694B4`), `TargetUtils`
(`0xBAE104D5`), `Global` (`0x57572DAC`), `template_ability_instantcast`
(`0xF76B6B7F`), `modifier_enrage` (`0xB96EFF67`), and
`modifier_energysentinel_stun` (`0xA555CCB4`). Namespace, registration string,
static properties, and caller behavior agree with every mapped body.

## Shared native and wire contract

### Client request

All four are logical action message `29`, application wire `0x9c`, action type
`7` (`ActionUseCharacterAbility`). The application packet is 85 bytes including
the opcode: a 40-byte common actor header and this 44-byte tail:

```text
targetObjectID u32
cursorPosition XYZ
targetPosition XYZ
abilityIndex u32
rank i32
unknown u32
userData u32
```

The authenticated deployed creature, server-owned rank/definition, live
position, mana, cooldown, and targets are authoritative. Header actor position,
rank, target ID, and both points are requests. Target `0` is an authored valid
form for all four mappings, although each definition consumes it differently.

### Lua/native boundary

The canonical decompile registers the `nAbility` API beginning at
`Game.c:1512120`. It includes `GetAbilityID`, `GetAbilityInstanceID`,
`GetAbilityNamespace`, `GetAgentID`, `GetTargetID`, `GetTargetPosition`,
`RegisterAbility`, `CreateAbilityEvent`, `SendAbilityEvent`, `GetRank`,
`GetRankedValue`, `GetAgentAttributeSnapshot`, `PayCooldownAndMana`,
`TargetInRangeAtStart`, `PreloadAsset`, `PreloadModifier`,
`PreloadAnimation`, and `PlayAnimationSequence`. The relevant object bindings
are also present: `TakeDamage`/`HealDamage` and hostile/friendly validation near
`1433684`, projectile waiting near `1434776`, object creation and spatial
queries near `1435717`, and `GetObjectsAlongLine` at `1435749`.

This proves that Lua supplies the ordered policy and native code supplies
object lookup, movement/collision, damage/healing, modifier, cooldown/mana, and
replication side effects. The retained client does not contain the original EA
authoritative action consumer, so it cannot prove an EA rejection code,
datagram grouping, dirty-component flush placement, or numeric object IDs.

### Ranked formulas

All four roots store two identical endpoints per supported rank unless a second
rank is explicitly listed below. For damage, build 103 applies:

```text
ranked endpoint
* (1 + (classPrimary + 1) * damageCoefficient)
* (1 + attacker descriptor/type/source/equipment modifiers)
+ flat direct damage when the hit is neither DoT nor HoT
```

The minimum is floored and maximum ceiled after those attacker stages. A later
per-hit roll, critical stage, and target defenses remain separate. Ravager uses
Strength, Tempest uses Mind, and Sentinel uses Dexterity. With the unmodified
Alpha template primaries (Blitz STR `14`, Sage MIND `23`, Goliath DEX `12`) and
no equipment multiplier/flat addition, the useful sanity ranges are:

- Electron primary `12..20 -> 21..35`; secondary `4..10 -> 7..18`;
- Shockwave `10..16 -> 16..27`;
- Zetawatt Beam `20..30 -> 33..50`.

Mana is not rounded before subtraction. The coefficient operand is the
actor's class-selected primary aggregate returned by native `sub_9E3680`, not
maximum mana:

```text
0, when overdrive-charged or authored base is zero
authored base + classPrimary * manaCoefficient, otherwise
```

For the base Alpha records this is Electron `16 + 14*0.08 = 17.12`, Enrage
`14 + 23*0.07 = 15.61`, Shockwave `16 + 12*0.08 = 16.96`, and Zetawatt
`12 + 12*0.06 = 12.72`. Admission and payment must use the same float. The
heroes' maximum-mana values (`113`, `148`, and `115`) are resource capacities
and do not participate in this formula.

These non-basic durations, including cooldown and authored hit/release waits,
use `seconds * (1 - CooldownScale)` when the actor has cooldown reduction;
missing modifier evidence is the identity operation. The raw times below are
therefore the content baseline, not permission to ignore an equipped timing
profile.

### Application ordering boundary

The Lua call order is proven; exact EA packet flush order is not. A compatible
server response needs this logical sequence:

1. accept only after all admission checks and reserve an ability instance;
2. return `0xa8 ActionCommandResponse` type `0`, using the mapped ability GUID,
   requested index, start time, authored/projected hit commit, and release end;
3. publish the caster `0xa5 SetAnimationState` at start;
4. at the authored payment point, atomically commit cooldown and mana (normally
   represented by `0xc1 CooldownUpdate` and mana field `0x97`);
5. emit authored `0x9b ServerEvent`, `0xba CombatEvent`, modifier `0xa2`/`0xa4`,
   HP/mana dirty fields, and object lifecycle messages in the Lua/native order;
6. release the ability/agent at its authored release boundary while allowing
   independently owned projectile or modifier work to continue.

The ack-first packet placement is darkspin's integration contract and the
least surprising server fallback, not a recovered EA datagram golden. Rejected
requests must emit no animation, cost, cooldown, modifier, projectile, damage,
or persistent mutation. If scheduling any required continuation fails, the
reservation and committed in-memory state must roll back before success is
published.

## Electron Sphere

### Definition

Chunk `696` creates `nAbility_PlasmaRandom_LightningBall` from the projectile
template and calls:

```text
nAbility.RegisterAbility("PlasmaRandom_LightningBall",
                         nAbility_PlasmaRandom_LightningBall)
```

It localizes `0x09C202C0` as Electron Sphere and defines:

| Property | Rank 1 | Rank 2 / note |
| --- | ---: | --- |
| cooldown | `12s` | same |
| mana / coefficient | `16` / `0.08` | same |
| activation range / projectile distance | `25` / `25` | same |
| projectile speed | `4` | `6` |
| primary damage | `12..20` | `16..24` |
| secondary lightning damage | `4..10` | `6..12` |
| damage coefficient | `0.05` | primary and secondary |
| damage type/source | Elements (`3`) / Energy (`1`) | same |
| descriptor | Projectile `8192` + Energy `128` + AoE `8` = `8328` | same |
| impact radius / secondary scan radius | `4` / `6` | scaled by `1 + AoERadius` |
| secondary cadence | `0.3 + 0.6*math.random()` seconds | one wait before each scan |
| secondary candidate budget / chance | `3` / `0.5` | bytecode decrements its validated-candidate budget and gates secondary damage with the random branch |
| hit / release | `0.26666668s` / `0.6s` | same |

The exact animation/effect/noun inputs are:

| Role | Asset |
| --- | --- |
| cast animation | `el_random04_lightningball` |
| projectile noun | `Ability_LightningBall.Noun` (`0xE99FC639`) |
| projectile trail | `el_random_04_projectile_effect.ServerEventDef` (`0x6122EEF9`) |
| muzzle | `el_random_04_muzzle_effect.ServerEventDef` (`0xB7EDD853`) |
| impact | `el_random_04_hit_effect.ServerEventDef` (`0xF14DD5C7`) |
| miss | `el_random_04_miss_effect.ServerEventDef` (`0x59A455A0`) |
| secondary lash | `el_random_04_lightning_lashOut.ServerEventDef` (`0x39FD47A9`) |
| accepted-hit effect | `plasma_common_electric_hit_small_effect.ServerEventDef` (`0x3D13D4F9`) |
| internal thread owner | `LuaJobObject.Noun` (`0xFBD3ECD7`) |

`Ability_LightningBall.Noun` and `LuaJobObject.Noun` are proven authored noun
identities, but neither has a projected row in the current authoritative
`noun_physics` table. The package catalog proves their hashes, not collision
dimensions or lifetime. Exact projectile geometry is therefore the remaining
noun-data gap; it must not be invented and presented as package proof.

### Runtime

The projectile template snapshots caster attributes/team at activation, plays
the animation, waits to `t=0.26666668`, pays cooldown and mana once, emits the
muzzle, creates one projectile noun, assigns team/owner/snapshot, attaches the
trail, and starts the projectile-owned tracking thread. The cast releases at
`t=0.6`; projectile lifetime is independent and can outlive that release.

The root overrides `TrackProjectile`. It first creates a separate
`LuaJobObject.Noun` and gives that object a `LightningShots` thread, then calls
the shared projectile tracker. While the projectile is valid, the secondary
thread waits the randomized cadence, reads its live position, queries
damageable objects sorted by distance in the effective six-unit radius,
requires damageability and hostility, consumes the authored candidate budget,
and applies the `0.5` random gate before secondary `TakeDamage`. An accepted
secondary hit emits the lash event relating projectile and secondary object,
then the small electric hit on that target. When projectile tracking ends, the
root marks the job object for deletion.

The shared tracker uses ordinary non-homing, non-piercing projectile movement.
At collision it emits `impactEvent`, selects direct/radius candidates, validates
them, calls primary `TakeDamage`, and emits the configured `onHitEvent` only for
accepted damage. A terminal path without accepted damage emits `missEvent`.
The projectile is then marked for deletion. Thus the stable semantic order is:

The impact radius is an allegiance scan, not an aggro-owner scan. Every live
hostile in the effective four-unit radius is eligible even when it is idle or
currently targets another squad member; requiring `TargetObjectID == caster`
incorrectly suppresses the explosion for ordinary mixed encounters.

Campaign runtime now follows that terminal distinction: a live direct
collision owns the primary radius scan, while an absent or missed target emits
the authored miss path and never creates the impact area.

```text
t=0                 animation
t=0.26666668        PayCooldownAndMana; muzzle; create projectile; trail;
                    begin projectile locomotion and secondary lightning owner
t=0.6               caster release (projectile continues)
each secondary      wait; sorted radius scan; random gate; damage;
                    lash effect; target hit effect
primary collision   impact effect; one or more primary damage calls;
                    accepted-hit effect(s); mark projectile/job for deletion
terminal miss       miss effect; mark projectile/job for deletion
```

The current packages do not prove the noun collision shape, object allocator
IDs, exact `ObjectCreate` body, locomotion reflection grouping, or delete tick.
A server fallback should use the existing authoritative projectile sweep with
the authored speed/distance and a documented conservative collision shape
until the noun payload is projected. That fallback is not build-103 noun fact.

## Enrage

### Definition and target semantics

Chunk `247` registers `CastEnrage`, inherits the instant-cast template, and
loads modifier `Enrage`. It defines cooldown `10s`, activation range `35`, mana
`14 + classPrimary*0.07`, zero direct damage, target type Allies (`1`), target search
radius `10`, and `ifNoTargetUseNearestOrSelf=true`.

Target `0` is explicitly supported. The instant template reads the stored
target point, queries creature objects sorted by distance within ten units,
takes the first friendly-valid object, and falls back to the caster if none is
valid. A direct friendly target, when supplied, is retained. The server must
validate rather than trust a client target; for a targetless request it must
preserve the stored cursor/target point when usable and otherwise use the
caster position as the conservative query center before self fallback.

The cast assets/times are:

| Role | Value |
| --- | --- |
| other-target animation | `lf_random_04` |
| self animation | `lf_random_04_self` |
| hit time | `0.33333334s` |
| release time | `0.6s` |
| caster muzzle | `bio_cast_enrage.ServerEventDef` (`0x8297BA74`) |
| target hit | `lf_enrage_cast.ServerEventDef` (`0x0ACACB3C`) |

The template resolves and stores the target, selects self versus other-target
animation, waits to hit, then calls `PayCooldownAndMana` **before** its hit-time
target revalidation. For a valid target it emits the muzzle/hit presentation
and requests the modifier; it then waits to release. That order means a target
lost after admission can consume the committed cast without receiving the
modifier. Admission itself must still reject invalid actor/cooldown/mana state
without paying.

### `Enrage` modifier

Chunk `716` calls `nModifier.RegisterModifier("Enrage", nModifier_Enrage)`.
It is Unique, lasts `30s`, is Buff + HoT (`16 | 4096 = 4112`), and deactivates
on agent death. It does not assign `modifierPriority`; the `Enraged=625` global
therefore must not be attached to it merely because the names resemble one
another. Its visible persistent effect is
`status_enraged.ServerEventDef` (`0x2374EE96`).

On activation it reads the **recipient's** primary attribute and adds:

```text
DirectAttackDamage = 5 * (1 + (recipientPrimary - 8) * 0.05)
BodyScale          = 0.12
```

It also attaches the status effect and retains its effect index for removal on
deactivation. For base Sage self-cast, MIND `23` produces `8.75` flat direct
attack damage.

The native attribute-duration stage projects the 30-second Buff baseline as
`30 * (1 + initiator BuffDuration)`. The modifier tick retains the
**initiator's activation snapshot** and calls
`HealDamage` immediately, then every `3s` while the native 30-second modifier
lifetime owns the thread. The authored amount is `2`, coefficient `0.05`, and
descriptor HoT. With base Sage as initiator this is
`2*(1+(23+1)*0.05)=4.4` before healing/HoT bonuses and recipient healing
reduction. Expiry before the `t=30` boundary gives ten applications at
`t=0,3,...,27`; the Lua loop itself is open-ended and native modifier teardown,
not an internal loop counter, stops it. Replacement behavior must follow
Unique activation rather than stacking another copy.

The minimum replication order at hit is target/caster effects, modifier create,
its immediate activation attribute/effect state, and its immediate first heal;
later heals follow at three-second cadence, then effect/modifier removal at
expiry or death. Exact component flush placement around `0xa2`, `0x9b`, `0xba`,
and HP fields is still a capture gap.

## Shockwave

### Definition and geometry

Chunk `105` creates the melee-template-derived
`nAbility_EnergySentinel_Support` and registers `EnergySentinelSupport`. It
localizes `0x09CD8D64` as Shockwave and defines:

| Property | Value |
| --- | --- |
| cooldown | `6s` |
| mana | `16 + classPrimary*0.08` |
| activation range | `4` |
| descriptor | Energy + AoE = `136` |
| animation | `energysentinel_support` |
| hit / release | `0.26s` / `0.8s` |
| primary hit arc length / angle | `6` / `180 degrees` |
| extra-target arc range / angle / cap | `4.5` / `180 degrees` / `999` |
| damage | `10..16`, coefficient `0.05`, Technology/Energy |
| per-target hit event | `cyber_cleave_hit.ServerEventDef` (`0x875BF885`) |
| modifier | `EnergySentinelStun` |

The root does not assign `interfaceType=Targeted`; only a value equal to
`nAbilityInterfaceType.Targeted` makes the melee template require and validate
a target ID. Target `0` is therefore authored and must not be rejected. The
template constructs its forward hit arc from the caster and stored target
point/facing, selects a best primary hostile in the six-unit/180-degree arc,
then enumerates additional valid hostiles in the 4.5-unit/180-degree arc. The
`999` cap makes the authored intent effectively all returned hostiles, subject
to validation and spatial query order. `shouldPursue=true` is admission behavior
when an explicit target exists; it does not turn a targetless Shockwave into a
required-target action.

At `t=0` the template snapshots the caster, picks the animation sequence, and
plays it. At `t=0.26` it pays cooldown/mana, constructs the arcs, damages the
accepted primary and additional targets, requests the stun after accepted
damage according to the template's 100-percent default modifier chance, and
emits the hit effect with target/caster/facing/critical context. It releases at
`t=0.8`. Exact ordering among targets is the native arc query order, which is
not proven to be object ID or distance order.

The 2026-09-06 Shockwave timing correction explicitly publishes `energysentinel_support` at the acknowledged start timestamp instead of relying only on local prediction while the server schedules damage. The existing 260 ms hit and 800 ms animation reset/action release remain unchanged; failed schedules queue both terminal packets so animation authority cannot strand the action.

### `EnergySentinelStun`

Chunk `605` registers `EnergySentinelStun`. Rank 1 duration is `4s` and rank 2
duration is `2s`; the decreasing second value is authored and must not be
silently normalized. The native attribute-duration stage applies recipient
debuff reduction and then crowd-control reduction to that baseline; the
client's separate diminishing-return stage follows. It is a Debuff with
`IsStun=64`, priority Stunned `850`, death deactivation, status animation `is_stunned`, head effect
`status_stunned.ServerEventDef` (`0xE38C48EB`), and icon `debuff_stunned`.

Activation adds `nAttributeType.Immobilized = 1`; its tick waits forever and
the native modifier lifetime removes the state. `CanUseModifier` has a subtle
authored rule: when `ImmuneToStunned` (`attribute 73`) is positive it rejects
the modifier in a chain game, but returns true outside a chain game. That is the
literal bytecode branch, even though the name suggests a broader immunity.

## Zetawatt Beam

Chunk `172` directly constructs `nAbility_Tech_Random` and registers
`TechRandom`. It has no Lua module dependency. Its exact definition is:

| Property | Value |
| --- | --- |
| cooldown | `8s`, Manual (`0`) |
| mana | `12 + classPrimary*0.06` |
| descriptor | Energy + AoE = `136` |
| range / beam length | `35` / `35` |
| animation | `tc_random01_zetawatt_beam` |
| hit / release | `0.23333333s` / `0.5s` |
| damage | `20..30`, coefficient `0.05`, Technology/Energy |
| beam event | `cyber_randomAbility_1.ServerEventDef` (`0x53F51A6A`) |
| per-target hit event | `cyber_common_hit_large.ServerEventDef` (`0x5CC479CC`) |
| preloaded but unused `hitEvent` field | `fire_ignite.ServerEventDef` (`0xC2812940`) |

`alwaysUseCursorPos=true` and the tick explicitly supports target `0`. It
snapshots caster/team/attributes, plays the animation, waits to hit, and pays
cooldown/mana. It then uses a valid hostile target's live position when one is
present; otherwise it reads `nAbility.GetTargetPosition()`. It normalizes the
caster-to-point direction, extends it to exactly 35 units, emits one beam event
with the endpoint, and calls `GetObjectsAlongLine(start,end,0,
nSporeLabs.damageableObjectTypes)`. In returned `pairs` order it excludes the
caster, validates every candidate hostile, emits the per-target hit event, and
then calls `TakeDamage`. The preloaded `fire_ignite` field is never referenced
by this tick and must not be synthesized. Release follows at `t=0.5`.

The required semantic order is therefore:

```text
t=0             animation
t=0.23333333    PayCooldownAndMana
                resolve hostile live point or stored cursor point
                emit beam endpoint event
                for each hostile along line: target hit event; TakeDamage
t=0.5           release
```

The server must reject a missing/non-finite/out-of-range stored point, but must
not require a target object. A zero-length caster-to-cursor direction needs an
explicit rejection or a previously proven facing fallback; the bytecode's
vector normalization does not supply a safe direction by itself.

## `EnergySentinelActive` compiler/bootstrap recovery

### What chunk `966` actually is

Chunk `966` localizes Arc Weld (`0x09CD8ADD`) and registers
`EnergySentinelActive`; it is Goliath's `special_1`, not the requested
Shockwave. It inherits the melee template and defines range `5`, cooldown `8s`,
mana `16 + classPrimary*0.08`, Energy descriptor, animation
`energysentinel_active`, hit arc length `7`, hit time `0.13s`, release `0.9s`,
Technology/Energy damage `14..18` at coefficient `0.05`, hit effect
`cyber_energy_arcweld_hit.ServerEventDef`, chain leap radius `7`, four maximum
chains, `25%` damage increase per jump, `0.1s` between jumps, a 20-degree hit
angle, and four beam events. Unlike Shockwave it explicitly assigns:

```text
interfaceType = nAbilityInterfaceType.Targeted
```

so Arc Weld requires a valid hostile target and is not a target-`0` contract.

### Exact enum seed

The observed constrained-compiler failure
`getTable[75]: got nil for key Targeted` is a bootstrap-data failure, not a
missing bytecode opcode or decoder feature. Packaged `GlobalDefinitions`
chunk `659` creates the table with these exact numeric entries:

```text
nAbilityInterfaceType = {
    Position = 0,
    Targeted = 1,
    SelfCast = 2,
    AutoTarget = 3,
    CreatureTarget = 4,
    TerrainPoint = 5,
    CreatureTargetDefaultSelf = 6,
}
```

The safe minimal seed for chunk `966` registration is therefore
`nAbilityInterfaceType.Targeted = 1`. The safer general fix is to seed the full
table above from the pinned chunk-659 contract, or to execute that exact pinned
bootstrap before definitions. Do not seed the string `"Targeted"`: the client
uses numeric equality in the melee template.

The related exact enum/global surface needed by these four roots and their
modifiers is:

```text
nTargetType: Enemies=0, Allies=1, Both=2, None=3
nCooldownType: Manual=0
nDamageSources: Physical=0, Energy=1
nDamageTypes: Technology=0, Spacetime=1, Life=2, Elements=3,
              Supernatural=4, Generic=5
nDescriptors: IsMelee=1, IsBasic=2, IsDoT=4, IsAoE=8, IsBuff=16,
              IsDebuff=32, IsPhysicalDamage=64, IsEnergyDamage=128,
              IsHoT=4096, IsProjectile=8192
nDebuffDescriptors.IsStun=64
nActivationType: Default=0, Unique=1, CasterUnique=2,
                 CasterUniqueIrreplaceable=3, Stacks=4,
                 StacksAndCasterUnique=5, UniqueIrreplaceable=6,
                 UniqueResets=7
nDeactivationType: OnAgentDestroyed=0, OnAgentDeath=1
nAbilityPrimaryStat: Damage=1, SingleDamage=13, SingleHealing=14
nAttributeType: AoERadius=62, ImmuneToStunned=73,
                DirectAttackDamage=108, BodyScale=113
```

This exposes defects in the failing seed beyond `Targeted`: that compiler state
gave only symbolic `nTargetType.Enemies`, lacked `Allies`, lacked
`SingleDamage`/`SingleHealing`, and lacks the listed modifier attribute/debuff
fields. Those symbolic values were sufficient for narrow earlier definitions
but are not a build-103 enum contract.

For definition registration, chunk `966` has no further missing top-level enum
after `Targeted`: its remaining top-level references (`nBit.Or`, Energy,
Technology, preloads, `SPID`, `ToGUID`, and `RegisterAbility`) already have
compiler bindings. Full activation is a larger execution surface, not another
enum failure. It requires the real/pinned melee and `TargetUtils`/`Vector`/
`Global` modules and these native/helper families:

- `nAbility`: target/agent/instance/rank/snapshot/target-position, timing,
  payment, range-at-start, and release operations;
- `nTargetUtils`: `MakeArc`, `IsObjectInArc`, `FindBestTargetInArc`,
  `FindObjectsInArc`, and Arc Weld's `ChainHitTargets`;
- `nGameObject`: position/center/facing/team/direction, hostile validation,
  damage, alive state, effects, and attribute snapshots;
- `nObjectManager` radius/line queries, `nModifier.RequestModifier`,
  `nAttribute` access/modification, `nEvent.Notify`, `nThread` waits,
  `nVector`, `nMathUtil`, `GetRankedValueHelper`, `TranslateToken`, `ipairs`,
  `pairs`, `type`, and `math.random`.

Those are static bytecode references, not a recommendation to install no-op
stubs. Registration-only compilation may safely bind metadata preloads, but an
activation engine must implement the stateful operations or reject the runtime
as unsupported. The native registration list in `Game.c` corroborates the
API boundary; the enum names are absent there because chunk `659`, not C++,
owns them.

## Proven facts versus server fallbacks

### Proven

- hero identity, creature IDs/nouns, slot maps, ability GUIDs, visible names,
  wire indexes, target `0`, root chunks, dependencies, registration names, and
  every static property listed above;
- interface/target enums and numeric descriptor/damage/modifier/attribute
  constants from packaged `GlobalDefinitions`;
- Lua call ordering, spatial-query family, target fallback behavior, authored
  animation/effect/noun strings, damage/healing operands, and modifier logic;
- native availability of the called ability/object/projectile/query APIs;
- build-103 mana, timing, attacker damage, and healing projection formulas
  already recovered from the canonical executable.

### Required conservative fallbacks or remaining captures

- Electron's noun payload/collision geometry is not projected. Use a documented
  conservative projectile sweep until the exact noun body is decoded; do not
  label the fallback as authored geometry.
- No retained EA-server capture fixes ack/reliable-channel grouping, dirty HP
  and mana flush placement, object allocator IDs, projectile create/locomotion
  packet bodies for this noun, delete tick, or native spatial query iteration
  order. Preserve semantic Lua order and use existing build-103 packet shapes.
- Damage sampling, critical RNG, mitigation, death, loot, and stat publication
  should use darkspin's already recovered shared combat pipeline; the roots do
  not define replacement policies.
- A targetless Enrage with no usable stored point should query around the
  caster before self fallback. A zero-length Zetawatt direction should reject
  unless later evidence proves a facing fallback. Both choices keep campaign
  playable without inventing a target object.

## Implementation-ready acceptance criteria

### Shared

- Route only an authenticated deployed hero with the exact noun/index/GUID map
  at the top of this note. Creature `4` must be treated as Goliath Alpha.
- Decode and retain target ID, cursor point, and target point independently;
  validate finite coordinates and server-owned range/team/liveness.
- Project cooldown/hit/release timing, mana, damage/healing, criticals, and
  modifier duration from the current authenticated creature snapshot. Use one
  mana value for admission and subtraction.
- Enforce cooldown, available mana, conflicting running ability/release state,
  and phase/session ownership before success. Rejection has no side effects.
- A success ack identifies the mapped ability GUID and original live index;
  its commit/end timestamps match the projected hit/release boundaries.
- Required scheduled work is reserved before publishing success. Teardown,
  swap, cancellation, and scheduling failure cannot leak costs, cooldowns,
  modifiers, objects, or damage into a later session generation.

### Electron Sphere

- Accept Blitz noun `0x6367B6CD`, index `3`, target `0` when a finite in-range
  stored point exists; register/ack GUID `0xD02C30E2`.
- Play `el_random04_lightningball`; at projected `0.26666668s` pay exactly one
  `16 + classPrimary*0.08` cost/cooldown, emit muzzle, and create exactly one
  `Ability_LightningBall.Noun` with trail, team, owner, and activation snapshot.
- Move non-homing for ranked speed `4`/`6` and distance `25`. Caster release at
  projected `0.6s` does not cancel the object-owned projectile/job threads.
- Apply primary Elements/Energy/AoE/Projectile damage `12..20` (rank 2
  `16..24`) in effective radius `4`; emit impact before damage and small hit
  only for accepted damage; emit miss on terminal no-damage path.
- While valid, run the exact randomized secondary cadence and sorted six-unit
  scan with authored budget/gate, applying `4..10` (rank 2 `6..12`) and emitting
  lash then target hit for accepted secondary damage.
- Delete projectile and job owner at their authored terminal boundaries and
  cancel both safely on session teardown. A collision-shape fallback is
  explicitly documented until noun physics is recovered.

### Enrage

- The 0.7.7 Sage trace repeatedly sent valid index-3 casts while retaining the
  current hostile target; the server rejected that enemy as though Enrage
  required a direct ally. Hostile, empty, dead, or unavailable target IDs now
  follow the native nearest-or-self rule and resolve to the living caster,
  while a valid direct squad ally remains selected.
- Accept Sage noun `0x2CA50A9A`, index `3`, target `0`; resolve the nearest
  valid friendly creature within ten units of the stored point, or self;
  register/ack GUID `0x5E0C8BF6`.
- Select self versus other-target animation. At projected `0.33333334s`, pay
  exactly one `14 + classPrimary*0.07` cost/cooldown before hit revalidation, then
  publish muzzle/hit effects and one Unique `Enrage` modifier on a valid target.
- Apply the recipient-primary direct-damage and BodyScale formulas, status
  effect, immediate initiator-snapshot heal, three-second HoT cadence, and
  removal at projected 30-second expiry/death. A recast never stacks an
  independent second modifier; the exact native same-GUID refresh/reject result
  still needs a focused modifier-lifecycle trace and must not be confused with
  the distinct `UniqueResets` activation type.
- Release the cast at projected `0.6s`; modifier lifetime is independent.

### Shockwave

- Accept Goliath noun `0xC940B9DF`, index `2`, target `0`; do not impose Arc
  Weld's `Targeted` rule; register/ack GUID `0x1C46A6FE`.
- Play `energysentinel_support`; at projected `0.26s` pay exactly one
  `16 + classPrimary*0.08` cost/cooldown and evaluate the caster-forward authored
  six-unit primary / 4.5-unit additional 180-degree arcs.
- Damage every returned valid hostile up to cap `999` with Technology/Energy/AoE
  `10..16`, publish the cleave effect in template order, and request rank-correct
  `EnergySentinelStun` only after accepted damage.
- Replicate stun animation/head effect and Immobilized state for authored rank
  duration; preserve the literal chain-game immunity check. Release at
  projected `0.8s`.

### Zetawatt Beam

- Accept Goliath noun `0xC940B9DF`, index `3`, target `0` with a finite,
  nonzero, in-range stored cursor direction; register/ack GUID `0xCC1E09D0`.
- Play `tc_random01_zetawatt_beam`; at projected `0.23333333s` pay exactly one
  `12 + classPrimary*0.06` cost/cooldown, resolve live hostile position or cursor,
  normalize it, and extend the beam to exactly 35 units.
- Emit one `cyber_randomAbility_1` endpoint event, query damageable objects
  along the line with radius argument zero, exclude caster, validate hostility,
  and for each accepted candidate emit `cyber_common_hit_large` before
  Technology/Energy/AoE damage `20..30`.
- Never synthesize the unused `fire_ignite` field. Release at projected `0.5s`.

### Compiler/bootstrap

- Seed the exact full numeric `nAbilityInterfaceType` table (at minimum
  `Targeted=1`) from pinned chunk `659`; do not add an opcode workaround or a
  string-valued `Targeted` placeholder.
- Seed the other exact enum fields listed above so Enrage and its modifier,
  Shockwave stun, projectile AoE, and melee target comparison compile with
  build-103 values.
- A registration test for chunk `966` reaches
  `RegisterAbility("EnergySentinelActive", ...)` and records
  `interfaceType==1`; activation remains gated until every stateful helper it
  reaches has a real implementation and behavior test.
