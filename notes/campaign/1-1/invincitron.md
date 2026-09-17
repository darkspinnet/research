# Invincitron half-health phase

## Result

The half-health behavior is an authored `NomadWithDrone` noun/AI passive, not
a mutation inferred from the visible word **Invincitron**. English locale row
`0x87cb094a` names the base `NomadWithDrone` archetype `Invincitron`.
`NomadWithDrone.AIDefinition` independently names
`NomadWithDronePassive`; its phase contains only `NomadWithDronePunch`.
Consequently the passive applies wherever that AI definition is used, including
the `NomadWithDrone.Noun` captain. The label alone is not a modifier, affix, or
captain-only policy.

The implementation-ready transition is:

```text
accepted damage mutates HP
  -> TookDamage reaches the still-active passive
  -> if not previously triggered and currentHP / currentMaxHP < 0.5:
       request deletion of every active IsDebuff modifier
       request NomadWithDroneShield on the captain
       request NomadDroneRage on the owned drone
       latch the passive as triggered
  -> ordinary NomadWithDrone combat AI remains eligible
  -> shield modifier ends in its authored end-animation order
```

The comparison is strict. Exactly `50%` does not trigger; the first accepted
hit leaving the captain below `50%` does. The crossing hit is not prevented:
the passive observes `TookDamage` and reads the resulting live HP and maximum
HP. Once latched, healing above half and taking damage below half cannot replay
the phase.

## Evidence set

### Packaged noun and AI content

`content.db` identifies the build as source build `103` and preserves the
following `AssetData_Binary.package` records for instance `0xB366BC74`:

| Content | Resource ID | Type | Decoded SHA-256 | Relevant fact |
| --- | ---: | ---: | --- | --- |
| `NomadWithDrone.Noun` | `1475` | `0x76A8F7D8` | `5299d9f58d28a013a0fe25e8ceda4ee3611cfe935d3f3d81dc5827c7412c6f60` | links the noun's NPC/AI content |
| `NomadWithDrone.Phase` | `1473` | `0x30728CE7` | `dee5e3f46153a9ed5dac9dc30bda648e834aa0b73d652638dd8b13543d24a78b` | contains only `NomadWithDronePunch` |
| `NomadWithDrone.AIDefinition` | `1477` | `0xEEEB0E31` | `2bada11a91db1693e5a453da3f730223bb58319d7b3b57de239d2dc75dbda927` | idle/combat behavior, robot aggro abilities, phase, and `NomadWithDronePassive` |

The AI record authors `nBehavior_Idle` for pre-aggro and combat idle,
`FirstAggro_ActivateRobot`, `SubsequentAggro_ActivateRobot`,
`NomadWithDrone.Phase`, and the additional passive. Existing exact noun/AI
joins and base/captain reuse are indexed in
`notes/campaign/1-1/nouns.md`; the ordinary punch/aggro behavior is recovered
in `notes/campaign/1-1/enemy-actions.md`.

English localization provides two distinct identities:

| Locale key | Text | Meaning here |
| --- | --- | --- |
| `0x87cb094a` | `Invincitron` | visible base-archetype name for `NomadWithDrone`, not the shield modifier |
| `0x0ba86438` | `Invulnerable` | shield modifier tooltip name |
| `0x0ba86439` | `Your hero is immune to damage and negative status effects.` | generic authored shield description; the word `hero` does not override the NPC target selected by Lua |

### Passive Lua

The passive is `content.db.lua_chunk` row `301`, resource `13845`,
`Modifiers/0xAE8664D8.lua`, decoded size `2462`, SHA-256
`9b01648abbecc202a545206c8d9f8c9f47ad96f26a68fbf28a71941a1d3c0418`.
Its exact authored contract is:

- activation type `Unique`;
- deactivation type `OnAgentDeath`;
- handled event mask `TookDamage` only;
- on activation, create one `NomadDrone.Noun` at `(0, 0, 0)`, set its owner
  to the captain, copy the captain's team, make the drone untargetable, and set
  private `shieldActivated = 0`;
- on deactivation, mark that private drone for deletion;
- on `TookDamage`, require `shieldActivated == 0` and
  `GetHitPoints(agent) / GetMaxHitPoints(agent) < 0.5`;
- enumerate `GetModifiersMatchingDescriptor(agent, IsDebuff)` and call
  `MarkForDelete(agent, modifierInstance)` for every returned entry;
- request `NomadWithDroneShield` with captain as both recipient and initiator;
- request `NomadDroneRage` on the private drone with the captain as initiator;
- then assign `shieldActivated = 1` and return true.

The flag is set after both modifier requests but does not inspect either
request's result. A rejected or otherwise ineffective request therefore still
consumes this passive instance's only trigger. `MarkForDelete` is called before
either request, although native deletion can be deferred when the victim
modifier is inside a protected callback.

The passive requires `Abilities!ability_sentrydronelaser.lua`. Its raw
`lua_dependency.target_lua_chunk_id` is null, but registration-name and
package-identity scanning recover the exact body as chunk `954`, resource
`14534`, `Abilities/0xC8F3AE7E.lua`, SHA-256
`b936cd027489cd316508c460cbe52cad304d2de4e0dcd3cdcaa9675c2b64e71b`.
It registers `SentryDroneLaser`; chunk `989` is the unrelated ordinary
`Fireball` definition and is not a compatible alias. The passive's otherwise
unused top-level require registers this laser before it creates its sole
`NomadDrone.Noun`, establishing the intended drone-ability link even though
the passive does not directly call `RequestAbility`.

The exact laser contract is mana `0`, no global cooldown, cooldown `0.7s`,
range `15`, hit time `0.17s`, radius `0`, projectile speed `20`, maximum travel
distance `50`, and `1-3` Technology/Energy damage with coefficient `0.05`.
It uses `Ability_Fireball.Noun`, trail
`effect_sentrydrone_projectile.ServerEventDef`, muzzle
`effect_sentrydrone_muzzle.ServerEventDef`, impact
`effect_sentrydrone_hit.ServerEventDef`, and miss
`ineffective_common_small.ServerEventDef`. The content does not yet recover
the `NomadDrone` AI selector or how `AttackSpeedScale` transforms the laser's
baseline cooldown/presentation.

### Shield and drone-rage Lua

Both definitions are in `content.db.lua_chunk` row `258`, resource `13798`,
`Modifiers/0x82EC97F2.lua`, decoded size `3279`, SHA-256
`ee306fc507dd368187e0423692ed275d21e29bd1c4f3f93cb577a0759e6543ab`.

`NomadWithDroneShield` registers table
`nModifier_NomadWithDrone_Shield` with:

- `Unique` activation, `OnAgentDeath` deactivation, and `requiresAgent=true`;
- ranked total duration `[5.0, 6.5, 8.0]` seconds;
- `ImmuneToDamage(...)` returning `1` unconditionally;
- icon asset `buffSteal_cyber`;
- attached effect `cyber_omnishield.ServerEventDef`, preloaded with context
  `OmniShieldModifier`;
- animations `nomad_lieu_tc_3_shield_start`,
  `nomad_lieu_tc_3_shield_loop`, and
  `nomad_lieu_tc_3_shield_end`, all preloaded under
  `NomadWithDroneShield`;
- `timetohit=0.3300000131`, `startAnimTime=0.6000000238`, and
  `endAnimTime=0.6333330274` seconds.

`NomadDroneRage` is a separate `Unique`, `OnAgentDestroyed` modifier on the
drone. Its ranked properties are:

| Rank slot | Duration | `MovementSpeedBuff` | `AttackSpeedScale` |
| ---: | ---: | ---: | ---: |
| 1 | `6s` | `1.0` | `0.5` |
| 2 | `8s` | `1.5` | `1.0` |
| 3 | `10s` | `2.0` | `1.5` |

Rage adds those two attribute modifiers immediately and waits forever; native
modifier lifetime owns expiry. It authors no effect or animation of its own.

The difficulty noun/class joins resolve the passive rank directly: base
`NomadWithDrone` uses the five-second shield, `_2` uses 6.5 seconds, and `_3`
uses eight seconds; the corresponding captain forms retain their family rank.
Their recovered combat/noncombat locomotion is respectively `6.5/5`, `9.5/5`,
and `12.5/5`. The server applies the same strict below-half one-shot transition
and immediate immunity to all six normal/captain noun forms.

## Entry, damage, AI, and exit semantics

### Entry ordering

The threshold hit has already passed ordinary damage admission and changed HP
before the `TookDamage` handler can see a below-half ratio. The handler does
not cap that hit at half, roll HP back to half, absorb its excess, or heal the
captain. A sufficiently large crossing hit can therefore leave the captain at
any positive below-half HP; Lua supplies no minimum-health clamp.

After the strict threshold test, authored Lua ordering is exactly:

1. request deletion of all current debuffs;
2. request the captain shield;
3. request drone rage;
4. latch `shieldActivated=1`.

Do not move the shield request ahead of debuff cleanup, and do not latch before
the two requests. Also do not infer that debuff deletion has physically
completed merely because every `MarkForDelete` call has returned.

### Damage admission while shielded

Build-103 registration finalization caches whether each Lua definition exposes
`AbsorbDamage` and `ImmuneToDamage`; see `Game.c` around
`sub_9DA000` (`1390378-1390380`). This shield exposes only the latter and its
predicate always returns true. It does not author absorption, a damage budget,
damage-type exceptions, source exceptions, or a break-on-hit count. Damage
admission while the modifier is live must therefore treat the captain as
immune, not as having armor, a second health pool, or a half-health floor.

The immunity predicate belongs to the active modifier instance, not to the
visual `AddEffect` call. There is no authored `0.33s` vulnerability window:
the modifier exists when requested, while `timetohit` only delays the effect
attachment and the two explicit control-immunity attributes. Conversely,
removing the visible effect before the end animation does not by itself admit
damage; the modifier and its `ImmuneToDamage` predicate remain live through
the modifier's completion.

### Shield timeline and exit order

For selected ranked duration `D`, the shield coroutine performs:

```text
t=0                    set shield_start animation
t=0.3300000131         attach cyber_omnishield effect
                       add ImmuneToDebuffs = 1
                       add ImmuneToKnockback = 1
t=0.6000000238         set shield_loop animation
t=D-0.6333330274       remove the attached effect and clear its private index
                       set shield_end animation
t=D                    end the modifier coroutine
```

The loop wait is authored as
`D - startAnimTime - endAnimTime`, so total coroutine time is exactly the
ranked duration. The effect is present for
`D - 0.3300000131 - 0.6333330274` seconds. The modifier's scoped attribute
changes, including its damage-immunity callback, end with modifier teardown;
Lua explicitly removes only the effect index. Early deactivation also removes
the effect index if it exists, but authors no end animation on that path.

The shield's exact duration for this captain cannot be collapsed to one number
without the missing campaign AI/modifier rank selection. The complete authored
answer is `5.0 / 6.5 / 8.0s` for rank slots `1 / 2 / 3`. Selecting `5s` merely
because it is the first element would be policy, not recovered behavior.

### AI behavior during the phase

Neither passive nor shield requests a defensive behavior, clears the hostile
target, stops locomotion, cancels a current ability, changes targetability, or
changes the noun's phase. The shield is a modifier beside the ordinary
`NomadWithDrone.Phase`, whose only combat action remains
`NomadWithDronePunch`; `SetAnimationState` is presentation state and is not a
movement-admission command. Accordingly there is no content basis for a
five-second idle, retreat, stun, or attack pause. Preserve the ordinary
NomadWithDrone selector and any already-admitted action while the modifier is
live.

The only authored combat-phase change is on the owned drone: rage adds ranked
movement and attack-speed attributes. Its intended attack definition is the
recovered `SentryDroneLaser` contract above. Do not replace those exact laser
values, apply the speed buff to the captain, or invent the missing
`NomadDrone` selector and native attack-speed transform.

Production uses a conservative description-parity selector at that missing
boundary: the passive creates one untargetable owner-bound `NomadDrone.Noun`,
the drone uses Invincitron's current hero target whenever it is inside the
laser’s exact range `15`, and it repeats the exact baseline `0.7s` laser. The
owner relation is projected for native client orbit presentation while the
server retains the owner's position as the stable firing pose. Owner death
marks the drone defeated and publishes its deletion. The unrecovered rage
cooldown transform remains inactive. The laser authors no attack animation, so
its NPC admission explicitly accepts that one animationless profile and its
projectile run suppresses only activation animation/turn presentation. Shot,
projectile trail, impact, release, and the baseline cooldown remain active.

### Replay rules

- The threshold is evaluated on every `TookDamage` event only until the private
  latch changes from `0` to `1`.
- Equality at half does not consume the latch. The next accepted damage event
  can trigger if its resulting ratio is below half.
- Once triggered, shield expiry, healing, further damage, debuff application,
  or returning below half cannot replay it.
- `Unique` prevents normal duplicate passive instances, and the passive is
  authored to deactivate on agent death. Its drone is deleted during that
  deactivation.
- A brand-new actor/passive instance initializes a fresh zero latch. Whether a
  particular resurrection system reuses the old instance or constructs a new
  one is not established here and must not be guessed.
- Because the latch ignores request success, modifier-admission failure does
  not authorize a retry.

## Explicit unknowns and non-policy

- **Captain rank selection:** the retained content gives every ranked shield
  and rage value, but not the retired campaign authority's mapping from this
  captain/difficulty to modifier rank. Exact runtime durations and rage
  magnitudes remain rank-dependent.
- **Drone selector and rage transform:** the laser body, damage, timing, range,
  projectile, and presentation assets are exact. The remaining boundary is
  which `NomadDrone` AI node requests it, target/admission retry policy, and
  the native transformation from ranked `AttackSpeedScale` to its cooldown and
  animation clocks.
- **Animation arbitration:** Lua proves its three animation writes and their
  times, and proves no AI pause. It does not prove how a simultaneous punch or
  locomotion animation wins presentation arbitration on every peer.
- **Lethal crossing hit:** Lua has no death guard in `HandleEvent`, but the
  retained evidence does not close native `TookDamage` versus death teardown
  ordering for a hit that crosses both half health and zero. Do not force a
  shield or drone-rage presentation on a dead actor without that lifecycle
  evidence.
- **Rejected-damage presentation:** the modifier predicate proves that damage
  is immune while live. The shared build-103 combat-text audit recovers native
  `CombatEvent` flag `0x0040`; every rejected hit now projects that feedback to
  members without an HP packet or damage-objective progress. Exact authored
  hit-effect and `TookDamage` callback behavior remain unrecovered here.
- **External early removal:** natural exit and death cleanup are authored.
  No evidence here authorizes an additional dispel, shield-break threshold, or
  extension/refresh rule.
- **Network flush order:** semantic mutation order is recovered from Lua and
  native modifier ownership; packet batching and relative publication of
  modifier, animation, effect, attribute, and HP updates are not.

## Retained diagnostics

- `bin/game/logs/invincitron-modifier-13845.luac` and
  `invincitron-modifier-13845.dump.txt`: passive bytecode and decoded
  instructions.
- `bin/game/logs/invincitron-modifiers-13798.luac` and
  `invincitron-modifiers-13798.dump.txt`: shield/rage bytecode and decoded
  instructions.
- `bin/game/GameBin/Game.c`: canonical client decompiler output;
  Lua callback registration is at `sub_9DA000` and the modifier request/runtime
  bridge is registered around lines `1512213-1512277`.
