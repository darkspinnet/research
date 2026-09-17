# Hero kit coverage

This is the implementation ledger for all 100 heroes: 25 archetypes with four
variants each. It reports authored build-103 content separately from compiler
and live-runtime support so an imported icon or ID is never mistaken for a
working ability.

### 2026-09-01 Field Medic Basic weapon damage

- **Resolved - Field Medic Basic:** `FieldMedicBasic` is chunk 86 (`Abilities/0x806F01E8.lua`, SHA-256 `9732b41812fd1a432123703ad565bab24f42326ac85f4d76089587f8483d6327`). Its impact callback obtains both damage bounds from `nClient.GetWeaponDamage`, but the projectile template also leaves a nonzero inherited damage table in the compiled definition. Player-basic catalog loading now preserves the callback contract explicitly for Field Medic Basic, ensuring Meditron's live weapon range replaces that inherited template damage at projectile impact.

### 2026-09-01 Shadow Sting melee admission

- **Implemented - Shadow Sting:** `ShadowRavagerActive` carries both the melee template's `hitArcLength` and its target vulnerability `modifier`. The generic modifier discriminator previously returned first and classified the whole targeted strike as a self buff, causing every index-two request to be rejected before pursuit or impact. The hero catalog now restores its authored melee shape and `shadow_sting_explode_effect.ServerEventDef`, allowing the shared active-melee path to pursue the selected hostile, deal its 12-18 Supernatural/Physical damage, and apply the six-second 50-percent physical vulnerability on an accepted hit.

### 2026-08-29 Seraph-XS Trapper Grenade release

- The attached Seraph-XS capture recorded six accepted `Trapper_Grenade` toss-basic commands without a targeting or authority rejection. The shared toss path emitted its launch and release responses but, unlike the standard projectile-basic path, never reset the casting animation. The canonical Trapper Grenade data releases the agent at 0.5 seconds, so toss basics now send the animation reset at that authored release boundary rather than retaining the ranged-cast presentation and its camera or locomotion lock.

## Current coverage

| Boundary | Coverage | Status |
|---|---:|---|
| Authored slot identity | 500/500 | Implemented for passive, basic, random, special one, and special two |
| Basic definition compiler | 100/100 | Implemented; six reusable execution shapes cover every hero noun |
| Active definition compiler | 68/300 slots; 16/70 unique assets | Implemented where the constrained Lua registration succeeds |
| Recovered active fallback | 240/300 slots; 38 additional unique assets | Build-103 contracts retained for definitions blocked by absent aliases or specialized decoder shapes; 232 use recovered runtimes and 8 recovered overrides also compile |
| Passive property projection | 100/100 identities | Passive-slot modifier properties now join the same content-derived projection as active abilities; runtime-dependent callbacks remain separate |
| Passive base runtime | 100/100 assigned slots | Every archetype has an initial authoritative rank-one passive runtime; specialized overdrive, co-op propagation, presentation, and pet-enhancement branches remain follow-up work |
| Active runtime | 300/300 assigned slots | Generic shapes plus named modifier, line, starter, trap, channel, healing, timed-area, status-area, infection, aura, projectile-status, and charge paths cover every assigned active slot |

The client wire mapping currently supported by evidence is basic index `0`,
special two index `2`, random index `3`, and squad/deck special one indices
`6`-`8`. Passives begin with hero deployment. The catalog deliberately stores
authored slot names and leaves this index mapping in the gameplay adapter.

## Archetype kits

Random abilities differ by Alpha/Beta/Gamma/Delta variant; the set column lists
the four authored possibilities without assigning gameplay behavior that has
not compiled.

| Archetype | Passive | Basic | Random set | Special one | Special two |
|---|---|---|---|---|---|
| Andromeda | GravityTempestPassive | ForcePulse | SpacetimeRandom, SpacetimeRandom1, SpacetimeRandom2, SpacetimeRandom3 | RepulsionWave | GravityStorm |
| Arakna | SoulRavagerPassive | SoulRavagerBasic | AfflictionBolt, CastSoulLink, NecroRandom, PhantomCharge | SoulRavagerSupport | SoulRavagerActive |
| Arborus | ThornBarkModifier | SplinteringCleave | BioRandom2, CastEnrage, RootingPlague, SummonSprite | ArborealMight | EntanglingRush |
| Blitz | LightningRoguePassive | LightningRogueBasic | PlasmaRandom, PlasmaRandom_FireWave, PlasmaRandom_LightningBall, PlasmaRandom_WebbedLightning | LightningRogueSupport | LightningRogueActive |
| Char | FireTempestPassive | FireTempestBasic | PlasmaRandom, PlasmaRandom_FireWave, PlasmaRandom_LightningBall, PlasmaRandom_WebbedLightning | FireTempestSupport | FireTempestActive |
| Goliath | EnergySentinelPassive | EnergySentinelBasic | ClaymoreTrap, TechRandom, TechRandom2, TechRandom3 | EnergySentinelActive | EnergySentinelSupport |
| Jinx | VoodooTempestPassive | VoodooTempestBasic | AfflictionBolt, CastSoulLink, NecroRandom, PhantomCharge | VoodooTempestSupport | VoodooTempestActive |
| Krel | FireRavagerPassive | FireRavagerBasic | PlasmaRandom, PlasmaRandom_FireWave, PlasmaRandom_LightningBall, PlasmaRandom_WebbedLightning | FireRavagerSupport | FireRavagerActive |
| Lumin | LightningTempest_Passive | LightningTempest_Basic | PlasmaRandom, PlasmaRandom_FireWave, PlasmaRandom_LightningBall, PlasmaRandom_WebbedLightning | LightningTempest_Support | LightningTempest_Active |
| Magnos | BinarySentinelPassive | BinarySentinelBasic | SpacetimeRandom, SpacetimeRandom1, SpacetimeRandom2, SpacetimeRandom3 | BinarySentinelSupport | BinarySentinelActive |
| Maldri | QuantumPositioning | QuantumStrike | SpacetimeRandom, SpacetimeRandom1, SpacetimeRandom2, SpacetimeRandom3 | QuantumState | QuantumBlink |
| Meditron | FieldMedicPassive | FieldMedicBasic | ClaymoreTrap, TechRandom, TechRandom2, TechRandom3 | FieldMedicSupport | FieldMedicActive |
| Orion | LightspeedTempestPassive | LightspeedTempestBasic | SpacetimeRandom, SpacetimeRandom1, SpacetimeRandom2, SpacetimeRandom3 | LightspeedTempestSupport | LightspeedTempestActive |
| Revenant | GraspingDead | Weakness | AfflictionBolt, CastSoulLink, NecroRandom, PhantomCharge | Terrify | Psistorm |
| SRS-42 | MissileTempestPassive | MissileTempestBasic | ClaymoreTrap, TechRandom, TechRandom2, TechRandom3 | MissileTempestSupport | MissileTempestActive |
| Sage | SupportHealerPassiveModifier | SupportHealerBasic | BioRandom2, CastEnrage, RootingPlague, SummonSprite | SupportHealerSupport | TreeOfLife |
| Savage | BeastSentinelPassive | BeastSmash | BioRandom2, CastEnrage, RootingPlague, SummonSprite | SummonBeast | BeastCharge |
| Seraph-XS | TrapperStealthModifier | Trapper_Grenade | ClaymoreTrap, TechRandom, TechRandom2, TechRandom3 | TurretTrap | PipeBomb |
| Skar | ShadowRavagerPassive | ShadowRavagerBasic | AfflictionBolt, CastSoulLink, NecroRandom, PhantomCharge | ShadowRavagerSupport | ShadowRavagerActive |
| Titan | TCShieldedSentinelPassive | TCShieldedSentinelBasic | ClaymoreTrap, TechRandom, TechRandom2, TechRandom3 | TCShieldedSentinelSupport | TCShieldedSentinelActive |
| Tork | RampantGrowth | Sprout | BioRandom2, CastEnrage, RootingPlague, SummonSprite | Sporogenesis | SleepingCloud |
| Vex | TimeRavagerPassiveModifier | TimeRavagerBasic | SpacetimeRandom, SpacetimeRandom1, SpacetimeRandom2, SpacetimeRandom3 | TimeRavagerSupport | TimeRavagerActive |
| Viper | LFPoisonRavager_Passive | LFPoisonRavager_Stab | BioRandom2, CastEnrage, RootingPlague, SummonSprite | LFPoisonRavager_PoisonNova | LFPoisonRavager_Expunge |
| Wraith | CrushingDread | Pummel | AfflictionBolt, CastSoulLink, NecroRandom, PhantomCharge | Ghostform | DeathsEmbrace |
| Zrin | PlasmaSentinelPassive | PlasmaSentinelBasic | PlasmaRandom, PlasmaRandom_FireWave, PlasmaRandom_LightningBall, PlasmaRandom_WebbedLightning | PlasmaSentinelSupport | PlasmaSentinelActive |

## Active definitions currently compiled

| Shape | Assets |
|---|---|
| Melee | EnergySentinelActive, LightningTempest_Support, ShadowRavagerActive, TimeRavagerActive, VoodooTempestActive |
| Projectile | FireRavagerActive, LightspeedTempestSupport, PlasmaRandom_FireWave, PlasmaRandom_LightningBall, SoulRavagerSupport, SpacetimeRandom1 |
| Cursor area | SpacetimeRandom3 |
| Point blank | DeathsEmbrace |
| Targeted area | SupportHealerSupport |
| Area healing | TreeOfLife |
| Teleport strike | LightningRogueActive |

Compilation remains distinct from runtime admission. All 68 last-audited
compiled slots and 240 independently recovered slot contracts retain typed
definitions. Every assigned active slot has a shared shape runtime or an
evidence-backed named runtime. Eight recovered contracts overlap definitions
that already execute through compiled paths.
Other definitions blocked by missing modules or unsupported registration
semantics remain fail-closed rather than being substituted with generic damage.

### Live generic projectile admission

The gameplay adapter now admits compiled projectile definitions in random
index `3` and special-two index `2`. Each authored ability ID receives an
independent content-derived cooldown key. Admission projects the selected
hero's cooldown, projectile speed, damage and mana cost; commits power,
movement stop, projectile lifetime, hit/critical/death/loot/stat transitions,
release, and rollback through the existing shared projectile authority.

This makes `PlasmaRandom_FireWave`, `PlasmaRandom_LightningBall`,
`SpacetimeRandom1`, and `FireRavagerActive` reusable across every hero variant
that owns those slots. Electron Sphere's secondary lightning scan remains
specific to `PlasmaRandom_LightningBall`; other projectile skills do not
inherit it. Starter point-blank, targeted-area, healing, and teleport
definitions retain their existing named runtime paths.

### Live generic melee and cursor-area admission

Compiled melee special-two definitions now use a single activation rather than
mutating the basic combo sequence. The shared path spends authored power,
reserves the content-derived cooldown, selects authored hit timing (including
multi-hit schedules), commits live-target damage and critical/death/loot/stat
transitions, and rolls admission back if scheduling fails. This enables
`ShadowRavagerActive`, `TimeRavagerActive`, and `VoodooTempestActive` for every
owning variant.

Compiled cursor-area random definitions use the same active resource and
rollback boundary while preserving cursor-centered target selection and area
damage. This enables `SpacetimeRandom3` for its owning Andromeda, Magnos,
Maldri, Orion, and Vex variants. Active abilities never inherit held-basic
repeat or combo sequencing.

Compiled generic `special_1` definitions now use the same executor when the
client sends them through squad slots 6-8. The selected squad member supplies
the authored ability identity while the deployed hero remains the actor and
resource owner, matching the existing support-ability contract. This enables
`LightningTempest_Support`, `LightspeedTempestSupport`, and
`SoulRavagerSupport` across their hero variants without named handlers.

## Recovered active definitions

`LightningRogueSupport`, `CastEnrage`, `Ghostform`,
`EnergySentinelSupport`, `BinarySentinelActive`, `ArborealMight`,
`FireTempestActive`, `GravityStorm`, `FieldMedicActive`, `LightningTempest_Active`,
`MissileTempestActive`, `TechRandom`, and `LFPoisonRavager_Expunge` use
rank-one contracts independently
recovered from build-103 content. Their catalog definitions are shared with the
named runtime instead of being duplicated as starter-only data. Enrage and
Zetawatt Beam admission uses the selected hero kit's authored random-slot
identity, so the behavior is no longer restricted to Sage or the original
starter mapping.

All existing named special paths now select the authored kit entry as well:
Plasma Wreath, Arc Weld, Strangling Briars, and Ghost Form resolve `special_1`;
Ride the Lightning, Shockwave, Death's Embrace, and Tree of Life resolve
`special_2`; Enrage and Zetawatt Beam resolve the variant's random slot. Their
definitions and cooldown identities no longer come from duplicated starter
singletons or basic-attack-name checks. Generic squad-slot routing uses the
same content boundary.

The shared compiler now resolves authored template-specific identities before
generic field fallbacks. Ride the Lightning's transit modifier therefore no
longer misclassifies the ability as a self buff, and Wraith's Pummel hit arc no
longer misclassifies the basic as a point-blank special. The live failure that
looked like a Death's Embrace soft lock was the subsequent Pummel rejection;
Death's Embrace itself completed admission in the same trace.

Death's Embrace now gives every surviving, non-immune area target its authored
five-second `DeathsEmbraceDebuff` modifier instead of recording fear only as a
hidden attack-suppression timestamp. The modifier publishes native Terrified
feedback, owns an exact create/delete lifecycle, and clears its authoritative
fear status on expiry or session cleanup. Its movement reuses Terrify's
projectile-independent navigation loop: each affected enemy repeatedly chooses
a reachable destination away from the casting hero and flees until the shared
fear authority expires. Terrify retains the same loop but no longer depends on
its projectile object remaining alive after impact.

### Live Cast Soul Link admission

The recovered `CastSoulLink` contract now uses the shared self-modifier path
for every Necro random-slot owner and owns its link visual as a forced
attachment on the active hero. The attachment therefore follows the hero
instead of remaining at the cast point, and its slot is hard-stopped and
released when the 12-second modifier expires or another self modifier replaces
it. Admission preserves its 24-second cooldown,
16 plus 0.08 Power cost, 0.233333-second hit boundary, 1.3-second release,
12-second unique modifier, cast animation, and link effect. While active,
hostile NPC melee and projectile damage is divided evenly among all living
squad members; inactive member health and HUD resources update with the local
damage transition, and total damage-taken accounting retains every share. The
modifier's overdrive-only borrowing of inactive squad passives is not
fabricated. **Implementation boundary:** chunk 417 already proves that this
branch reads the live overdrive-state signal and current squad-passive GUID
list on activation and every scheduler yield. No additional content research
is required; gameplay authority must expose those two live inputs before the
recovered branch can be wired without treating ordinary ability rank as
overdrive state.

### Live Rooting Plague admission

The recovered `RootingPlague` contract now uses a shared recursive infection
runtime for every Bio random-slot owner. The primary target receives the
eight-second `LifePlague` and `BioEntangleModifier` lifecycles, takes eight
one-second 4-point Life/Energy ticks, and remains rooted while retaining any
attack already in range. Every accepted tick can seed uninfected hostiles
within radius 5; each spread independently deals six one-second 3-point ticks
and can continue the same graph. Per-run immunity prevents duplicate infection,
and modifier/root authority is released on natural expiry, hero switch, reset,
or schedule failure.

### Live healing-tick admission

Sporogenesis and Field Medic Support now share one healing-tick runtime. It
owns authored power, cooldown, cast/release timing, healing coefficients,
per-tick health/resource publication, floating healing text, statistics, and
reset-safe cancellation. Sporogenesis uses its five immediate-then-one-second
ticks and mushroom presentation, and its first pulse now removes every
authoritative tracked debuff from the current self-target without excluding
channel modifiers. Field Medic Support uses its six half-second channel ticks,
selected living co-op hero or cursor-nearest friendly resolution within range
10, self-target fallback, and unique 15-second 25-percent maximum-health
modifier. Targetable living friendly companions participate in the same exact
selection and cursor-nearest fallback. The caster owns healing-dealt statistics
while the selected owner owns healing-received statistics and authoritative
hero or companion resource projection. Field Medic Support now also switches
between its authored ally and self animations, starts the caster-to-ally beam
only for an ally, attaches the ordinary target or self-target effect to the
actual recipient at the first healing pulse, and releases that attachment on
completion, invalidation, reset, or scheduling failure. Normal release and
early invalidation also reset the caster animation at the corresponding
authoritative channel boundary.

### Live ground-aura admission

Psistorm and Sleeping Cloud now execute through one stationary live-membership
aura runtime. It owns power, cooldown, cast/release timing, fixed cast centers,
quarter-second enter/exit admission, exact modifier instance deletion, and
cleanup-safe cancellation. Psistorm applies its recovered radius-6 silence and
six immediate-then-one-second 7-point Supernatural/Energy damage pulses over
five seconds. Sleeping Cloud applies its recovered radius-7 sleep for the
six-second channel and uses sleep as NPC action/locomotion suspension. The
client-facing modifiers remain zero-duration and therefore live until explicit
exit or teardown, matching the authored aura contract.

Sleeping Cloud's non-DoT damage wake event is connected to the generic damage
dispatcher, and Psistorm now creates and deletes the recovered
`shadow_spirit_pool_aoe.Noun` object. Psistorm records authoritative silence
membership, but the
NPC scheduler still needs a basic-versus-special action distinction before
silence can suppress only special actions without incorrectly suppressing basic
attacks.

Sleeping Cloud sleep now participates in every shared NPC combat boundary:
retained attacks cannot land, pursuit corrections pause and resume after the
remaining sleep duration, special push/pull payloads cannot commit, and
corpse-consumption movement waits without abandoning its claim. This keeps the
zero-duration aura modifier as the presentation lifetime while ordinary NPC
authority remains suspended until exit, wake-on-damage, or aura teardown.

### Radial projectile-burst admission

Soul Ravager Active now reuses projectile-burst authority with an explicit
radial targeting mode. Its six rank-one projectiles launch at recovered
0.10-0.15-second deadlines, divide the full circle evenly, travel ten units,
admit hostile collisions within radius 3, and cap one target at four accepted
hits for the activation. The runtime preserves its 7-14 Supernatural/Energy
damage, power, cooldown, critical, death, statistics, projectile presentation,
and reset-safe cleanup contracts. The shared passive soul counter now expands
that volley through the recovered 6/8/9/10/11/12 projectile table while
preserving the four-hit-per-target activation cap.

### Projectile-delivered status admission

Terrify and Affliction Bolt share delivery authority while preserving their
different projectile contracts. A valid Terrify impact creates its exact
client modifier and authoritative fear state, then deals five immediate-and-
one-second 6-point Supernatural/Energy ticks over 4.1 seconds. Affliction Bolt
instead retains its projectile for 12 seconds, scans radius three every 100
milliseconds, admits every distinct uncursed non-fixture hostile it crosses,
and gives each contact an independent `AfflictionCurse`, four 5-point ticks at
2, 4, 6, and 8 seconds, and 8.25-second expiry. The projectile retargets the
nearest eligible hostile within 20 units and publishes the recovered native
turn rate six. Both paths own power, cooldown, release, projectile identity,
damage, critical, death, statistics, modifier expiry, and cancellation-safe
rollback.

Terrify now repeatedly selects navigation-safe destinations away from the
caster and suspends the feared NPC's ordinary pursuit and attack damage until
the retained fear modifier expires. The exact shared fear template samples a
uniform distance from 5 through 10 units on a uniform full-circle heading,
projects the result through `GetClosestPosition`, and makes at most three
reachability attempts. It applies the authored `0.5` transient speed modifier,
moves exactly to the point until within `1.5 * footprint`, and samples the next
destination only after that movement/control loop completes; there is no fixed
repath timer. Stealther NPCs now expose generic zone-owned stealth state from
cast through hit or interruption cleanup, so Affliction Bolt's initial target,
contact trigger, and retarget validator all reject stealthed candidates.

### Target-owned charge admission

Beast Charge, Entangling Rush, and Phantom Charge now share one charge runtime
across their 4/4/5 assigned slots. It validates the selected hostile and range,
commits power and cooldown atomically, increases movement speed for the authored
charge, admits contacts along the traveled segment, restores ordinary movement
speed, and publishes damage, critical, death, statistics, modifier, release,
switch, reset, and failure cleanup. Beast Charge applies its one-second knock-up
to accepted contacts. Entangling Rush damages its selected target and roots all
valid hostiles within six units of the endpoint for six seconds. Phantom Charge
damages and silences accepted path contacts for two seconds.

Beast Charge now interrupts a living tracked Beast companion's ordinary attack
or pursuit, orders it toward the same selected target at the recovered
speed-three and range-125 contract, applies the inherited hero 18-28 damage and
one-second knock-up once to every hostile crossed within radius four, and then
commits its separately recovered 8-12 terminal damage against the selected
target. Both paths use ordinary critical, death, loot, statistics, and
floating-text authority without collapsing the two authored payloads.

### Initial passive execution

Passive-slot `nModifier_*` properties now enter the immutable creature-template
projection instead of being discarded by the active-ability-only content
query. The gameplay binding consumes these properties without hero-name gates.
Lightning Rogue Passive adds the authored 0.50 critical-damage increase.
Crushing Dread applies its authored 0.15 physical and energy reductions when
the attacking NPC is inside its twelve-unit aura. The suppressed NPC now deals
the same reduced damage to Wraith, co-op heroes, and allied companions, using
the strongest eligible unique aura once. Rampant Growth adds its authored 0.25
damage- and healing-over-time increases to the owner and snapshots the same
bonuses onto nearby living co-op heroes within its recovered radius 20. Multiple
Tork auras select the strongest contribution once. Soul Ravager Passive gains one soul per
confirmed hostile death, caps at five, and rebuilds its authoritative
DamageBuff by 0.05 per soul. Soul Ravager Active consumes those souls and uses
the recovered 0-5 stack table to launch 6/8/9/10/11/12 radial projectiles;
Soul Ravager Support consumes the same pool for 3/3/4/4/5/5 homing projectiles
whose snapshots retain the authored 50-percent lifesteal. Gravity Tempest
Passive snapshots its radius-fifteen projectile aura onto friendly projectiles
launched by the owner or nearby co-op heroes, adding 0.25 projectile speed and
0.10 projectile damage once per cast snapshot. Time Ravager Passive Modifier
includes its owner and nearby co-op heroes within radius 12, adding the recovered
0.10 attack speed to cast snapshots and 0.06 movement speed to live movement
authority. Same-passive and overlapping allied auras use their strongest
contribution once. Poison Ravager
Passive applies one poison stack after each accepted non-DoT hit, caps at four,
and resets the three-second modifier duration on every application. Each stack
adds one 2-point plus 0.05-coefficient Life/Energy tick per second.
Binary Sentinel Passive projects its recovered 0.01 defense conversion into
the shared defense-to-basic-damage operand, so Magnos gains damage from the
same physical and energy defense inputs used by the client-derived formula.
Thorn Bark applies its recovered 0.20 physical-damage reduction to Arborus and
reflects its content-authored percentage of accepted post-mitigation melee
damage to the attacking NPC. Reflection uses the ordinary NPC damage, death,
loot, encounter-transition, floating-text, and statistics publication path.
Plasma Sentinel Passive starts a six-second 0.04 all-source damage-reduction
stack after Zrin takes an accepted hit. Further hits refresh the duration and
increase the reduction through the recovered five-stack cap.
Missile Tempest Passive gains one 0.05 DamageBuff stack per full stationary
second, caps at five, and clears the stack clock whenever authoritative player
movement begins. Basic and active damage snapshots consume the same projected
creature state. Fire Tempest Passive attaches its charge presentation for eight
seconds, waits while no hostile is within radius five, then removes the charge,
deals the recovered 4-6 Elemental/Physical damage with a 0.05 coefficient after
0.1 seconds, stuns surviving accepted targets for two seconds, and rearms after
two seconds. Deployment, switching, death, reset, and connection cleanup cancel
the passive cycle. Lightspeed Tempest Passive continuously derives its rank-one
attack-speed and movement-speed increases from the deployed hero's current
health percentage, reaching 0.20 and 0.25 respectively at full health and
reprojecting automatically after damage, healing, or a squad switch. Its
half-second health poll also selects and replaces the exact recovered
`LT_jetpack_effect_lvl1` through `LT_jetpack_effect_lvl5` attachment, removing
the presentation immediately after death or switching away.
Voodoo Tempest Passive applies Voodoo Charm to Jinx and every nearby living
co-op hero inside its recovered radius eight. A charmed hero that defeats a
hostile creature recovers two percent of maximum mana and publishes the
authored recovery effect. Rank-one health return is exactly zero.
Fire Ravager Passive now counts accepted basic melee activations per deployed
Krel. Every fourth activation applies the recovered one-second
`FireRavagerStunModifier` to surviving targets actually damaged by that basic,
then resets the counter. Switching away, death, and mission reset clear the
counter; active abilities never advance or consume it.
Energy Sentinel Passive waits one second after Goliath enters play, then every
four seconds selects one living hostile within the authored radius sixteen.
The selected target receives `EnergySentinelVulnerability`, including client
modifier presentation and the exact rank-one 0.15 increase to incoming
physical and energy damage. Retargeting, squad switching, death, reset, and
session cleanup remove the prior authoritative vulnerability.
Grasping Dead applies its authored rank-one 0.15 movement-speed reduction to
hostile NPC pursuit while Revenant is within the passive's radius-eight aura.
The server recalculates the aura from current authoritative positions during
each pursuit step, matching the client modifier's dynamic trigger volume.
Lightning Tempest Passive projects the exact rank-one 0.50 share of Lumin's
current Critical Rating onto the deployed owner and every nearby living co-op
hero inside its recovered radius thirty. Overlapping Lumin auras select the
strongest contribution once rather than stacking duplicate shares.
Support Healer Passive's existing campaign companion lifecycle matches the
recovered rank-one contract: two `HelperMelee.Noun` saplings, one-second spawn
delay, radius-five placement, owner/team/base-attribute projection, distance
cleanup, switch/death cleanup, and delayed replacement ownership. Hostile melee,
projectile, chained-area, and push-area damage now connect a defeated sapling to
its authored 20-second replacement deadline, remove its dead combat actor, and
recreate the same stable companion slot at full health independently of the
other sapling's deadline. Spawn
completion now immediately runs their autonomous target acquisition,
pursuit, and melee scheduler instead of waiting for the owner to move.
TC Shielded Sentinel Passive now derives total Energy Defense from the selected
hero's base profile and equipped-part contribution. After the exact rank-one
15-second charge, it absorbs `25 + (EnergyDefense - 100) * 0.25` incoming
damage, carries partial shield strength between hits, and restarts the charge
only when the shield is depleted. Generation and hit effects are published,
and the persistent `status_shielded` attachment now remains on Titan between
hits. Depletion and squad switching explicitly detach it, while connection and
zone teardown release its attachment-slot ownership.

Field Medic Passive is now extracted through its complete rank-one owner and
laser pair. Rank one creates one `SentryDrone.Noun` in a one-unit ring around
Meditron, mirrors owner/team/base attributes, makes the drone non-targetable,
projects that point through the active navigation layer, and retains it until
the owner modifier deactivates. `SentryDroneLaser` is now
a first-class content program: range 15, cooldown 0.7 seconds, hit delay 0.17
seconds, speed 20, distance 50, and 1-3 Technology/Energy projectile damage
with coefficient 0.05. Companion authority now distinguishes a combat-capable
actor from a targetable actor. The deployed drone now acquires hostile NPCs,
fires its repeating projectile through the standard collision, damage,
floating-text, loot, and death pipeline, and is removed on switch, owner death,
reset, or disconnect.

- **Corrected - Field Medic laser identity and pet stat offset:**
  `SentryDroneLaser` is chunk 954 (`Abilities/0xC8F3AE7E.lua`, SHA-256
  `b936cd027489cd316508c460cbe52cad304d2de4e0dcd3cdcaa9675c2b64e71b`),
  not chunk 989. Chunk 954 registers `nAbility_SentryDroneLaser` and authors
  the 0.7-second cooldown, range 15, 0.17-second hit time, speed 20, distance
  50, and 1-3 Technology/Energy damage with coefficient 0.05. Chunk 989
  (`Abilities/0x5A5AAFAF.lua`) is the separate ordinary `Fireball` definition;
  its `petPrimaryStatOffset = 1` must not be imported into the sentry laser.
  Field Medic Passive chunk 382 (`Modifiers/0x2CC4DEFE.lua`, SHA-256
  `7f46544508eb7b34ea302436911f3eeead0adf633866cad0ae796b4bd38548e6`)
  attempts to copy `nAbility_SentryDroneLaser.petPrimaryStatOffset` into its
  `damageCoefficient`, but the real laser defines no such field, so that read
  is Lua `nil`. The passive independently authors its own
  `petPrimaryStatOffset = 1`; that modifier-owned value is the exact unit
  offset used by its Pet Damage token projection.

Quantum Positioning now applies its exact rank-one 100-percent Physical Defense
increase when projecting every Maldri variant. The modifier's rank table keeps
Energy Defense at zero until the overdrive ranks, where it becomes 50 percent.
**Implementation boundary:** the rank table is complete, but applying its
overdrive branch requires gameplay authority to expose the live overdrive
state. This is production-state wiring rather than missing reverse-engineering
evidence; ordinary ability rank must not substitute for that signal.

Beast Sentinel Passive's owner modifier is also fully audited: activation only
creates private state and waits forever. Its actual behavior is delegated to a
living `BeastSentinelPet.Noun`, so the generic owner modifier correctly remains
inert. Summon Beast now pays its authored rank-one cost and cooldown, places the
pet four units forward, publishes its spawn presentation, and tracks the living
pet until switch, owner death, reset, or disconnect. Placement rotates the
client-authored local `+Y` basis through the caster quaternion instead of
mistaking raw quaternion components for a direction. The pet pursues hostile
NPCs and repeats its exact 1.1-second, range-one, 4-7 Life/Physical melee basic
with coefficient 0.05 projected from Pet Damage. Recasting to apply Beast Pet
Enrage now costs half the stat-adjusted summon cost and retains the existing
pet. After the authored 0.36-second boundary, its eight-second modifier adds
50-percent pet damage, heals five percent of maximum health immediately and
once per second, and forces hostile NPCs within radius five to reacquire the pet.
Expiry, owner switching, owner death, reset, and disconnect remove the modifier
and its authoritative damage increase.

Shadow Ravager Passive now applies its exact rank-one 25-percent multiplier to
direct basic damage when retained NPC facing proves the attacker is behind the
target. NPC targeting and locomotion now retain authoritative facing rather
than deriving this result from client presentation. Unknown facing receives no
bonus. The initial rear classification uses the rear hemisphere; the exact
Lua contract is modifier chunk 864 (`Modifiers/0xE2D2FCDB.lua`, SHA-256
`96442a01f5ffa8a60f91095bb577a11a2966b84cf3550e66fc4d2921666d9740`): outside
overdrive it adds `0.25` to attacker attribute 20
(`BehindDirectDamageIncrease`), while overdrive instead adds `0.30` to
attribute 21 (`BehindOrSideDirectDamageIncrease`). The script performs no
geometry test. The angular boundaries consumed by those attributes belonged to
the retired authoritative damage engine and are not present in build 103; the
current rear-hemisphere classification is therefore an explicit server-policy
fallback, not an outstanding client-content decode.

Trapper Stealth now runs its authored passive cycle for every Seraph-XS
variant: twelve seconds of cooldown, three seconds of Technology stealth,
native enter/exit effects, and early stealth removal when the owner takes or
deals damage or commits its first accepted ability. Rejected ability attempts
do not reveal the hero. Deployment, switching, death, reset, and disconnect invalidate
the previous cycle. While stealthed, the zone excludes the hero from NPC target
selection, releases existing hostile targets, and reacquires another living
player-aligned actor when available. Exit restores target admission and resumes
ordinary NPC scheduling. The client-facing state uses the build-103 agent-blackboard
stealth field and the exact recovered enum: None 0, Technology 1, Supernatural
2, and Fully Invisible 3.

Summon Sprite now executes for every Bio random-slot owner through retained
zone companion authority. Each cast pays the authored rank-one power cost,
starts its 30-second cooldown, places a separate `SpritePet.Noun` four units
forward using the client quaternion basis, and publishes the exact muzzle,
spawn, animation, and trail presentation. Concurrent sprites remain allowed as
authored. Each sprite follows its owner inside a two-unit leash and restores
three percent of the active hero's maximum health each second, including
healing-received modifiers and combat statistics. Each sprite independently
emits its spawn-out effect and is deleted after 30 seconds; owner death, squad
switching, reset, defeat, and disconnect remove all owned sprites early. The
recovered passive contains only presentation and lifetime authority; follow and
healing are the playable fallback for the client-owned Healing Spirit behavior
reported from the live game.

BioRandom2, Roar of Derision, now executes for every Bio hero through the
shared target-owned charge runtime. It uses the recovered range-30, doubled
movement-speed charge, rank-one power and cooldown, and radius-eight endpoint
payload. The payload deals no damage: it forces nearby hostile NPCs to acquire
the caster for the authored five-second taunt presentation and grants the
caster the exact 25-percent incoming-damage reduction for six seconds. The
charge, taunt, buff, release, expiration, switch, death, reset, and disconnect
boundaries share the retained zone and modifier authorities.

SpacetimeRandom2, Delaying Sphere, now executes for every Spacetime hero as a
retained caster-position zone object. It pays at the recovered 0.233333-second
boundary, creates the radius-eight team-owned sphere for 12 seconds, and scans
membership every 250 milliseconds. Hostile NPCs entering receive the unique
zero-duration drag modifier, 40-percent movement speed, and 60-percent attack
speed; leaving or sphere cleanup removes that exact modifier instance and both
authoritative adjustments. The sphere, friendly globe presentation, hostile
effect, modifier identities, cooldown, power, release, and final deletion are
all content-backed. **Implementation boundary - hostile projectile drag:** the
recovered Lua intentionally delegates motion to native projectile locomotion:
the same zero-duration drag modifier applies `-0.60 MovementSpeedBuff` and
`-0.40 AttackSpeedScale` to admitted projectiles and is deleted on exit. It
authors no replacement trajectory or deadline. The retained server projectile
scheduler must therefore make velocity and remaining-flight time live
modifier-aware instead of keeping a fixed impact deadline. No additional Lua
or package operand remains to recover.

Delaying Sphere now includes retained hostile projectiles in its live
membership scan. Entry preserves the current position, applies the exact
visible drag modifier, and reduces authoritative flight speed to 40 percent;
exit or sphere teardown deletes that modifier and restores full speed. Impact
polling follows the live remaining-flight clock, so leaving the sphere also
accelerates the pending collision instead of retaining a stale slowed deadline.

Delaying Sphere's `AttackSpeedScale=-0.40` now stretches each admitted NPC
action's hit, release, and cooldown boundaries by the same `1 / 0.60` factor.
The prior cooldown-only projection produced the correct lower repeat rate but
left projectile contact on its full-speed windup, making ranged attacks appear
to fire rapidly without a matching impact.

Time Ravager Support now executes for every Vex variant through a reusable
navigation-validated teleport-area shape. It pays the recovered rank-one power
and cooldown, preserves movement during the windup, publishes both wormhole
destinations, teleports to the reachable target or cursor position, and deals
the recovered radius-four Spacetime/Energy area damage. Every surviving target
whose damage was accepted is authoritatively stunned and silenced for three
seconds. The teleport commits at the authored 0.266667-second boundary and the
area damage and control scan follows 0.1 seconds later, preserving the separate
wormhole-exit presentation, and
**implementation boundary - hostile projectile freezing:** chunk 419 requests
the same Time Ravager modifier on hostile projectiles without damage, and chunk
962 supplies the exact 3/1-second Frozen lifetime. The Lua delegates pause and
resume to native projectile state and carries no alternate trajectory or new
impact deadline. Preserve the projectile's remaining flight state while Frozen
and resume it after modifier removal; this requires scheduler support, not
additional content reverse engineering.

Time Ravager Support now scans retained hostile projectiles in the same
radius-four payload snapshot, applies the exact visible modifier, freezes each
projectile at its live position for the rank-shaped three/one-second lifetime,
and resumes its preserved remaining flight afterward. NPC projectile impact
producers now defer against that live remaining-flight clock instead of
resolving at their stale launch-time deadline.

Turret Trap now executes for every Seraph-XS variant through retained trap and
zone companion authority. The cast pays its recovered rank-one power and
cooldown, creates a targetable `TurretTrap.Noun` at the requested point, scans
the authored ten-unit range once per second, and applies the recovered 3-5
Technology/Energy laser damage to the nearest hostile. It repacks after the
exact ten-second lifetime, while a turret reduced to zero health emits the
authored broken effect and is deleted early. Every accepted shot now publishes
the exact `TurretLaser` animation, source-to-target
`cyber_laser_turret_beam` event, and `cyber_common_hit_small` target impact
recovered from chunk 513. The targeting, damage, critical, death, statistics,
lifetime, cleanup, and client presentation paths are authoritative.

Repulsion Wave now executes for every Andromeda variant through a shared
control-nova runtime. Ranks one through four correctly deal no damage, ranks
five and six add their recovered 10-14 Spacetime/Energy area hit with a 0.04
coefficient, and ranks seven and eight add the recovered four-second,
50-percent movement slow without damage. Odd ranks retain six-second cooldowns
and even ranks use four seconds. Every hostile in the wave receives one
navigation-safe outward displacement and client forced-movement presentation. Hostile
projectile reflection is already fully recovered by chunk 292: clear the old
target, replace team and caster attribute snapshot, redirect radially from the
nova center, set horizontal speed by the authored radius falloff from 20 to 7,
zero vertical speed, and force a client update. This is an implementation
follow-up using the exact contract recorded in the resolved Repulsion Wave
entry below, not an open evidence gap. The imported `KnockbackModifier` is exact packaged chunk
391 (`Modifiers/0x105CE7AE.lua`, SHA-256
`ea9db2c8c4b5d29ae7a9374357ad80007c0db0238cb91851cbeb8a2d1686992e`)
over knockback template chunk 921 (`Modifiers/0xB3BDD679.lua`, SHA-256
`80acd83d6a22194f61ad75c36e9cfef7a37d3a4836efac8322260ef9199137c3`).
It is a unique `Knockedback` modifier, rejected by root or positive
`ImmuneToKnockback` outside chain games, that computes the normalized direction
from the initiator's supplied ground point to the recipient and calls
`JumpInDirection` with desired distance 8, speed 20, minimum jump height 0,
maximum jump height 4, and range-to-maximum-height 4. It plays
`react_knockback`, waits for jump completion, stops locomotion, then retains a
one-second outro before resetting animation. Repulsion Wave now uses the
authored eight-unit, speed-20 displacement through navigation-safe
forced movement. The client owns the recovered zero-to-four-unit jump arc and
one-second animation outro after receiving that movement intent.

This is base rank-one execution, not a claim that all twenty-five passives are
complete. Soul Ravager's orbit objects remain a closed missing-content boundary.
**Implementation boundary:** Fire Tempest's recovered overdrive charge/damage
variants and other overdrive-only branches require gameplay authority to expose
the live overdrive-state input before they can execute. Their content operands
are not an open RE gap, and ordinary rank must not be used as a proxy. All archetype passives now
have an initial base runtime; remaining work is specialized co-op, overdrive,
presentation, and pet enhancement.

**Closed evidence boundary - Soul Ravager Passive orbit:** shipped chunk 124 proves five non-targetable
`TrappedSoul.Noun` slots, radius-one placement outside the owner's footprint,
height 0.5, and empty/filled effects, but its required misspelled module
`Modifers!modifier_soulravager_soulbag.lua` is absent from all indexed content.
The exact `SoulOrbitThread` symbol occurs only at its two call sites in chunk
124; neither package tree, the authoritative content database, the canonical
client decompiler, nor the retained reference-server corpus contains its body
or a renamed equivalent. Continuous orbit movement and cleanup are therefore
not recoverable from the available build-103 artifacts. Keep soul-slot world
presentation disabled instead of fabricating five static objects; only a
retail behavioral capture or additional original-server content can replace
this conclusion.

- **Retired-server boundary - Physical and Energy Defense:** Build 103 has no
  target-side consumer for attributes 7 or 9. Its only recovered damage use is
  attacker-side basic preview through `DefenseBoostBasicDamage *
  (PhysicalDefense + EnergyDefense)`. Quantum Positioning changes the correct
  rating; no target reduction should be invented without a retail server trace.
- **Retired-server boundary - Shadow Ravager geometry:** Modifier chunk 864
  proves the exact `0.25` rear attribute and overdrive-only `0.30` rear-or-side
  attribute, but delegates both classifications to the retired damage engine.
  Build 103 contains no recoverable angular cutoff.

## Optional and specialized runtime assets

No assigned hero-kit runtime asset remains deferred. Remaining work is shared
modifier breadth, co-op/companion projection, overdrive branches, and exact
presentation rather than another missing named active.

### Recovered implementation findings

- **Resolved - packaged identities:** case-insensitive FNV-1 lookup resolves 44 of the 46 direct imports: `modifier_affliction_bolt_curse` -> chunk 798, `template_ability_charge` -> 574, `modifier_roar_buff` -> 1002, `modifier_roar_taunt` -> 338, `modifier_soul_link` -> 417, `ability_claymoretrap_detonate` -> 323, `template_trap` -> 94, `modifier_claymoretrap_passive` -> 39, `modifier_silence` -> 817, `modifier_entanglingrush` -> 461, `modifier_fieldmedichealthbuff` -> 829, `ability_fireball` -> 989, `modifier_fireelementalenrage` -> 493, `modifier_supportrangedelemental` -> 948, `modifier_poisonnova` -> 666, `modifier_MissileTempest_Support_FlakUpgrade` -> 888, `modifier_rocketslow` -> 28, `modifier_necro_random` -> 89, `modifier_phantom_silence` -> 297, `modifier_pipebomb_passive` -> 596, `modifier_plasma_random` -> 549, `modifier_PlasmaRandom_WebbedLightning_Shock` -> 721, `ability_plasmasentinelpet_basic` -> 165, `modifier_plasmasentinelpet` -> 307, `template_ability_cone` -> 170, `AuraUtils` -> 739, `PrivateTableUtils` -> 84, `NavigationUtils` -> 245, `Vector` -> 439, `template_ability_nova` -> 913, `modifier_RepulsionWave_SlowUpgrade` -> 791, `template_ability_instantcast` -> 50, `modifier_shadowravager_stealth` -> 859, `modifier_stackable_silence` -> 1006, `modifier_delaying_sphere_drag` -> 238, `ability_beast_pet_attack` -> 885, `Global` -> 827, `modifier_pet_tracker` -> 850, `modifier_sprite_healer_passive` -> 613, `modifier_tcshieldedsentinel_active_taunt` -> 103, `modifier_omni_shield` -> 396, `modifier_charged_fist_taunt` -> 139, `modifier_fear` -> 580, and `ability_turret_laser` -> 513. The remaining two authored import paths are absent under their expected FNV filenames, but registration-name scanning recovers their exact shipped definitions under different source identities: `FireRavagerEnergyVulnerabilityModifier` is chunk 33 (`Modifiers/0xE3E9D302.lua`) and `TCShieldedSentinel_Support_Snare` is chunk 740 (`Modifiers/0x7BDF4917.lua`). All 46 dependency behaviors are therefore present; these last two require linker aliases rather than fallback design.
- **Resolved - Fire Ravager Support:** `FireRavagerSupport` is chunk 243 with projectile template chunk 38 and the alias-resolved energy-vulnerability modifier chunk 33. It pursues a creature target within range 20, costs 8 mana with coefficient 0.04, has cooldown 3, and launches `Ability_Fireball.Noun` homing projectiles at speed 40 for distance 21. Every projectile deals 10-16 Elements/Physical damage with coefficient 0.05. Ranks 1-6 launch twice at cumulative times 0.2 and 0.5 seconds; ranks 7-8 launch three times at 0.2, 0.4, and 0.6 seconds; caster release is 0.95 seconds. On every accepted rank-1-6 hit, the template's 100-percent modifier path requests the stacking vulnerability. It lasts 2 seconds at ranks 1-4 or 4 seconds at ranks 5-6, caps at two stacks, adds 0.50 `EnergyDamageIncrease` per stack, rebuilds the attribute to stack count times 0.50 on a successful stack, and refreshes duration. Ranks 7-8 disable the modifier and use the extra projectile instead.
- **Implemented - Fire Ravager Support vulnerability:** every accepted rank-one fireball now applies or refreshes its two-second modifier, caps at two stacks, and adds 50-percent Energy-source damage taken per stack. The NPC damage boundary keeps this operand separate from Energy Sentinel Passive's all-damage vulnerability, and known hero damage paths carry their authored Physical/Energy source into that boundary. Modifier refresh, expiry, switch, reset, and teardown retain independent authority.
- **Implemented - Fire Ravager Support rank branches:** the shared volley no longer inherits Lightning Tempest Active's randomized target/channel cadence. Ranks one through six retain their selected hostile across two homing fireballs at 0.2 and 0.5 seconds; ranks five and six extend Energy vulnerability to four seconds. Ranks seven and eight instead launch three shots at 0.2, 0.4, and 0.6 seconds and correctly omit the vulnerability modifier.
- **Resolved - TC Shielded Sentinel Support:** `TCShieldedSentinelSupport` is chunk 413 with the alias-resolved snare chunk 740 and silence-upgrade chunk 53. It targets a position at range 20, costs 12 mana with coefficient 0.06, has manual cooldown 6, pays at 0.17 seconds, and releases at 0.45. It scans valid hostiles inside `4 * (1 + AoERadius)` around the saved target position. The selected target takes 20-30 Technology/Physical damage at odd ranks or 16-24 at even ranks, coefficient 0.05; other targets take 10-16 at ranks 1-6, while ranks 7-8 raise secondary damage to the same 20-30/16-24 pair as the primary. Only accepted damage receives control. Ranks 5-6 apply `TCShieldedSentinel_Support_SilenceUpgradeModifier`, a silence lasting 3/1 seconds that is blocked by `ImmuneToSilence` outside chain games. All other ranks apply `TCShieldedSentinel_Support_Snare`, the shared speed template with a five-second duration and 0.50 speed adjustment. The authored descriptors are `IsAoE | IsEnergyDamage` even though the damage source field is Physical; preserve both rather than normalizing the mismatch.
- **Implemented - TC Shielded Sentinel Support rank branches:** even ranks use 16-24 primary damage while odd ranks retain 20-30. Ranks five and six replace the ordinary snare with their accepted-hit three/one-second silence modifier. Ranks seven and eight raise every secondary target to the same ranked primary damage and use the recovered 24-power plus 0.12 coefficient cost.
- **Resolved - tempest projectile ordering:** Lightning Tempest Active (chunk 675) snapshots the caster origin, then performs `30 * (1 + AoEDurationIncrease)` shots separated by `(0.265 + random[0,0.1]) * (1 - ChannelTimeDecrease)`, clamped to at least 0.05; it pays once after the first wait, resets the cast animation after 1.9 seconds, and releases after 2.0 seconds when those thresholds are crossed inside the loop. Each shot independently takes a 30% hit branch, chooses one random valid hostile within 18 of the original origin that has never previously been selected, and launches downward from target `z + 10`; an empty hit candidate set falls through to a downward miss at a random point 3-18 units from that origin, and each projectile owns collision/deletion independently. Missile Tempest Active (chunk 287) captures the cursor, creates a radius-6 reticle, then fires `15 * (1 + AoEDurationIncrease)` shots every 0.3 seconds, paying once after the first wait; it alternates the transformed +/-0.8 muzzle offset, chooses a uniformly random radius and angle around the cursor, launches an initial projectile upward, waits for its range-16 flight, replaces it at the chosen ground point plus 15 Z with a downward projectile handed to the shared tracker, creates a ground shadow until that descent projectile dies, and deletes the launch projectile. Deactivation changes to the relax animation and deletes only the reticle; the scheduler owns release at 4.5 seconds and launched projectiles continue independently.
- **Resolved - Binary Sentinel pull:** Binary Sentinel Active (chunk 713) applies cone damage first and requests a pull only when `TakeDamage` accepts it: ordinary targets receive `BinarySentinelPullModifier` (chunk 782), bosses receive `PushPullBossModifier` (chunk 11), with the caster snapshot, rank, and scalar 1. The ordinary modifier rejects pull-immune targets outside chain games and does nothing to rooted targets; otherwise it computes center distance minus both footprint radii and, only while positive, starts `react_pulled`, attaches the caster-target attraction effect, and immediately calls `JumpInDirection` toward the caster at speed 25 for that edge distance, waits for jump completion, resets animation, then waits a 0.25-second outro. The boss modifier does not move the boss: unless rooted, it chooses a forward/back step animation from the boss-to-caster angle and the modifier float property (pull versus push), optionally attaches the attraction effect for pull, waits 1.6 seconds, and resets animation.
- **Implemented - Binary Sentinel Active pull:** every accepted surviving ordinary target now moves along a navigation-valid direct path toward Magnos at speed 25 until their footprint edges meet, with `react_pulled` and a modifier lifetime derived from travel plus the exact 0.25-second outro. Rooted targets retain the modifier without displacement. Bosses receive the separate 1.6-second `PushPullBossModifier` and remain stationary. Modifier instances, forced positions, expiry, reset, and teardown share existing zone movement and effect authority.
- **Resolved - Expunge:** Expunge (chunk 343) performs its template melee hit at 0.2 seconds, then at 0.5 seconds enumerates every target modifier carrying `IsDoT`; it resolves each modifier's ability namespace and skips entries without a namespace table or `GetRemainingDamage`. For every resolvable DoT it uses that modifier's original initiator snapshot, calls `GetRemainingDamage` for minimum, maximum, and damage type, reapplies that remaining damage once with Expunge's 0.05 coefficient and `IsMelee | IsDoT` descriptors, then marks the source modifier for deletion regardless of whether the amplified `TakeDamage` succeeds. This is descriptor-based consumption rather than a poison GUID lookup, consumes all resolvable DoTs in iteration order, preserves unresolvable DoTs, and deletes only after each amplification request; release remains at 0.73 seconds.
- **Implemented - Expunge retained damage consumption:** At the recovered 0.5-second boundary, Expunge now consumes every retained Poison Nova, Poison Ravager Passive, Terrify, Affliction Bolt, and primary or spread Rooting Plague damage-over-time modifier on its selected target. Each family reports its unexecuted tick count, original initiator creature snapshot, damage type, and damage source; Expunge applies that remaining base range once with its separate 0.05 coefficient, then cancels the old schedule and deletes its modifier even if an earlier consumed family defeats the target. Rooting Plague retains `LifePlague` independently from `BioEntangleModifier`, so consuming the disease does not remove the root's separate lifetime or cleanup.
- **Resolved - Arboreal Might and Fire Tempest Active:** Arboreal Might's modifier (chunk 312) starts at one stack, caps at 5/2 by rank, and on `StackModifier` increments first, rebuilds DamageBuff as stack count times 0.10/0.20 plus BodyScale as stack count times 0.06, then refreshes its 15-second duration; a maxed stack neither changes nor refreshes. `NearbyDeath` never adds a stack: it only refreshes duration when the dead unit is hostile and its event distance is strictly below 20. Fire Tempest Active (chunk 22) places `SupportRangedElementalModifier` on the hero; first application summons `RangedElementalPet.Noun`, assigns the hero as owner, and mirrors base attributes/team, while the first accepted stack asks the stored living elemental to receive `FireElementalEnrage`. The owner modifier replaces a living pet only after it exceeds 50 units; it does not respawn a dead or invalid pet. That unique 15-second pet modifier waits 1.25 seconds, adds 0.2 BodyScale and releases the pet, adds 0.5 DamageBuff, then every second deals 5-6 Elements/Energy damage with coefficient 0.05 to hostiles within 5; the hero never owns the enrage aura.
- **Resolved - Fire Tempest Active lifecycle:** `FireTempestActive` is chunk 22 with owner modifier chunk 948, pet passive/beam-out chunk 60, and enrage chunk 493. It has manual cooldown 8, hit/release timing 0.3/1.3, and requests the owner modifier only after paying at 0.3 seconds. The first cast uses the summon animation; any cast made while the owner modifier exists uses the enrage animation. Its custom mana callback returns exactly 20 for the first cast despite declaring coefficient 0.10 and `PetDamage` as the primary stat. With an existing owner-modifier stack it returns `(20 + PetDamage*0.10)*0.50 - PetDamage*0.10`, while the displayed `enrageCost` token separately rounds `(20 + PetDamage*0.10)*0.50`; preserve this authored payment/display mismatch rather than silently charging a generic half cost. The owner modifier lasts 300 seconds, uses stacking activation with cap 2, and on first activation emits the summon event at a closest valid point five units away in a random half-circle, waits to an exact 0.7-second deadline, then creates `RangedElementalPet.Noun`; its declared rank `spawnDelay` values of 1/0.5 seconds are never read. The pet receives owner, mirrored base attributes, team, and `SupportRangedElemental_Passive`. That passive immobilizes the pet during its two-second spawn animation, then releases it; after pet death it requests `PlasmaTempestPetBeamOut`, whose 0.8-second immobilized beam-out deletes the pet. The owner loop exits on dead/invalid pet and does not create a replacement. While the pet remains alive, crossing strictly beyond owner distance 50 deletes it and immediately runs the spawn path again. An accepted `StackModifier` event requests enrage only if the stored pet does not already have it; it neither refreshes nor replaces an existing enrage. Enrage begins its radius-5 pulse immediately after the 1.25-second warmup and repeats every second until the 15-second modifier expires; damage acceptance is ignored for presentation, so every valid hostile receives the impact effect after the 5-6 Elements/Energy damage request. Owner-modifier deactivation sends a living pet through the same beam-out path rather than deleting it directly.
- **Implemented - Fire Tempest Active:** Char's first cast now pays the authored flat 20 power, creates the five-minute owner modifier, presents the summon at a navigation-valid random half-circle point, creates `RangedElementalPet.Noun` after 0.7 seconds, and releases its two-second spawn immobilization. The first retained stack cast uses the recovered enrage animation and `10 - PetDamage*0.05` payment, then applies the unique 15-second pet modifier, waits its 1.25-second warmup, and commits radius-five 5-6 Elements/Energy pulses from the pet every second using Pet Damage as the primary attribute. The owner loop now retains a living pet inside the strict 50-unit boundary; crossing beyond it deletes the old elemental, cancels any modifier bound to that old pet, and immediately reuses the navigation-valid 0.7-second summon plus two-second release path with a fresh actor identity. A dead or invalid pet still ends the loop without replacement as authored. Hero switch, death, reset, disconnect, and owner expiry cancel owner, enrage, monitor, and replacement schedules and beam the current elemental out.
- **Corrected - Fire Tempest elemental action set:** `RangedElementalPet.Phase`
  resource `003848_eeeb0e31_00000000_00000000446a725a.bin` contains the
  follow/death lifecycle, while its adjacent 731-byte AI action resource
  `003844_30728ce7_00000000_00000000446a725a.bin` supplies the omitted combat
  branch: follow the owner, select the closest target inside radius 20, and
  invoke `Fireball`. Ordinary Fireball is exact indexed chunk 989 and authors
  cooldown 1, range 20, 0.17-second launch, speed 18, distance 50, 3-5
  Elements/Energy damage with coefficient 0.05 and Pet Damage offset 1,
  `Ability_Fireball.Noun`, `fireball01.ServerEventDef`, and
  `fireball_impact.ServerEventDef`.
- **Implemented - Elemental Guardian Fireball:** Char's living released
  Elemental Guardian now participates in combat, repeatedly acquires the
  closest hostile inside the recovered 20-unit radius, and launches the exact
  ordinary Fireball through authoritative projectile collision, Pet Damage,
  floating damage, death, and loot publication. Enrage's companion damage and
  attack-speed operands project onto the shot, while pet replacement, hero
  switch, death, reset, disconnect, and owner expiry cancel any in-flight
  attack reservation.
- **Resolved - Gravity Storm:** `GravityStorm` is chunk 515 with target modifier chunk 352 and effect-owner modifier chunk 984. It is a channelled enemy-position cast with range 50, cooldown 24, declared mana 24 plus coefficient 0.12, and `IsAoE | IsChannel | IsPhysicalDamage`; its custom Tick never calls `PayCooldownAndMana`, so do not invent an in-script payment point. On activation it snapshots the caster, requests the unique effect-owner modifier on the caster, saves the cursor once, computes `radius = 5 * (1 + AoERadius)`, creates `GravityStorm.Noun` at the cursor with that value as its object scale, starts the cast animation, and scans damageable objects only after 0.333333 seconds. Every then-valid hostile in that one radius snapshot receives the unique `RaisedIntoTheAir` stun/debuff modifier; only non-none modifier IDs are retained, keyed by target object ID. The caster waits another 0.075 seconds, removes the area effect, then waits `clamp((2 - 0.075) * (1 - ChannelTimeDecrease), 0.05, same)` with interruption enabled before playing the 1.2-second slam. The separately reported release time is `0.333333 + clamp(2 * (1 - ChannelTimeDecrease), 0.05, same) + 1.2`, so preserve this authored distinction rather than collapsing both clocks. Each admitted target stops locomotion, plays lift, and independently waits until its modifier elapsed time reaches `clamp(2 * (1 - initiator ChannelTimeDecrease), 0.05, same)`. If not interrupted before that boundary it plays slam, deals 16-24 Spacetime/Physical damage with coefficient 0.05 at 0.466667 seconds, waits the remaining 0.733333 seconds, and, only when still alive, plays a 1.333333-second get-up. An `InterruptModifier` event instead selects the no-damage 0.966667-second drop path; the modifier is `finishOnDeath` and `deactivateOnInterrupt`, and its deactivation resets the target animation. Ability deactivation always resets the caster animation and asks the effect owner to shut down; while the caster's private `interruptable` flag remains true it additionally sends `InterruptModifier` only to retained targets that are still valid, while natural completion clears that flag after the caster slam. Effect shutdown removes `spacetime_aoe_levitateSlam.ServerEventDef`, signals its owner loop, waits exactly two seconds, then returns false so modifier deactivation marks the effect object for deletion; destruction of the caster instead deactivates that owner modifier and deletes any existing effect object directly.
- **Implemented - Gravity Storm:** Andromeda's active now validates the cursor within range 50, creates the separately identified `GravityStorm.Noun` at the saved cursor with radius-five scale, and attaches the recovered levitate-slam effect. It snapshots radius-five hostiles at 0.333333 seconds, presents and applies the recovered Raised Into The Air modifier, removes the area effect after the authored additional 0.075 seconds, and deletes the effect object exactly two seconds later. The channel honors channel-time reduction for the lift, plays the caster slam, deals 16-24 Spacetime/Physical damage at the exact slam hit, releases after the authored 1.2-second slam, and clears target stun/modifier authority after the 1.333333-second get-up. Cleanup retains hard-stop and object-deletion fallbacks for interrupted runs.
- **Resolved - Field Medic transfer policy:** Field Medic Active (chunk 536) pays at 0.23 seconds and scans creatures within 10. On targetable allies it removes every `IsDebuff` modifier except GUIDs whose ability descriptors include `IsChannel`; on enemies it removes every `IsBuff` modifier without a channel exclusion and without requiring targetability during collection. Each side deduplicates by modifier GUID, retaining only the highest rank and that winning instance's original initiator snapshot; equal ranks keep the first instance, and no source stack count, remaining duration, or lifetime is preserved. A second pass applies captured ally debuffs to hostile targetable enemies and captured enemy buffs to allies, using the caster as initiator object but the retained original snapshot and rank. After release at 0.75 seconds it waits 30 seconds and deletes only the newly transferred debuffs recorded on enemies; transferred buffs on allies remain, while all qualifying originals were already marked for deletion during collection.
- **Implemented - Field Medic Active:** Meditron now casts the authored radius-ten, 0.23-second-hit, 0.75-second-release active through shared power, cooldown, animation, and effect presentation. The zone owns a modifier inventory for authoritative transferable effects. Field Medic cleanses every registered non-channel sleep, stun, silence, and root from living nearby heroes, deduplicates each modifier GUID by highest rank, and recreates the winning statuses on every nearby targetable hostile NPC with cancellation-safe modifier presentation and a maximum 30-second cleanup. The inverse pass now removes the authoritative Zelem Energy Buff, Carrion Shambler corpse-consumption damage buff, and Zelem Haste Buff from nearby hostile NPCs, deduplicates them by the same recovered rule, and transfers their recovered energy-damage, stacked general-damage, attack-speed, cooldown, and movement operands to every nearby living hero and targetable allied companion for each modifier's normal lifetime. Zelem Haste Buff's chunk 475 and shared AttributeUtils chunk 87 prove exact additive `+0.25 AttackSpeedScale`, `+0.25 CooldownScale`, and `+0.50 MovementSpeedBuff`; build-103 timing formulas resolve these to 20-percent shorter basic-attack intervals, 25-percent shorter ordinary cooldowns, and 50-percent faster movement. Companion attack cadence and pursuit speed consume those operands directly. Generic descriptors remain the expansion boundary for future NPC buffs.
- **Resolved - missing globals/runtime surfaces:** Lightspeed Tempest Active is chunk 998 and its exact `modifier_lightspeed_slow` dependency is chunk 234; beyond ordinary radius/damage calls it requires modifier inspection/deletion natives (`GetModifiersMatchingDescriptor`, `GetModifierGUID`, `GetRank`, `GetInitiatorAttributeSnapshot`, `GetStackCount`, `MarkForDelete`) to steal every `IsHaste` stack from accepted hostile hits while preserving GUID, rank, snapshot, and stack count. Quantum State is the self-contained chunk 1013 plus exact knockback/speed template chunks 921/453 and requires `nAbilityPrimaryStat.BuffDuration`, `nAbilityEventFlags.TookDamage`/`DealtDamage`, and `nDebuffDescriptors.IsBanish` plus event-payload/effect helpers. Sleeping Cloud is chunk 398 (the `SleepingCloud` string in chunk 591 is only a Sporogenesis effect-owner label) and requires `nAbilityPrimaryStat.ChannelDuration`, `nAbilityEventFlags.StartedNewAction`/`TookDamage`, `nDebuffDescriptors.IsSleep`, and `nAbilityFns.HandleEvent` plus channel/area callbacks. Time Ravager Support is chunk 419 with modifier chunk 962 and Vector chunk 439; it requires `nAllowsMovement.Always`, `nAbilityEventFlags.BreakRoot`, navigation calls `FindGoodMeleePosition`, `IsPositionReachable`, and `FindBallRollPosition`, teleport-object support, and BreakRoot event delivery.
- **Resolved - movement definitions:** `QuantumBlink` is chunk 577, `BeastCharge` is chunk 78, `EntanglingRush` is chunk 388, and `PhantomCharge` is chunk 964. Beast Charge, Entangling Rush, and Phantom Charge all instantiate `nAbility_Charge_Template` and therefore use a target-owned charge path, not the shared teleport-strike destination contract: Beast Charge hits each hostile once for 18-28 Life/Physical damage and accepted hits receive knock-up, while also ordering the tracked beast pet to charge the same target; Entangling Rush charges the selected target and on completion roots all valid hostiles within 6 of the reported target position; Phantom Charge slides through the selected target, damages/silences each valid hostile once, and uses a per-target private-table key for deduplication. Quantum Blink is the exception that consumes the reported target position as a cursor destination and owns a custom repeated-strike operation. It does not deduplicate strike targets and does not restore the cast origin: it saves its return point only after the initial slide. Its destination validation can reuse the teleport-strike boundary, but its slide, cyclic strike, camera, and cleanup sequence require their own runtime operation.
- **Resolved - Quantum Blink:** `QuantumBlink` is chunk 577. It targets the cursor at range 35, costs 18 mana with coefficient 0.09, has manual cooldown 12/18, and deals 4-16/2-8 Spacetime/Physical melee damage with coefficient 0.05 per hit. Activation plays a 0.1-second warmup, pays, sends `BreakRoot`, adds `Immune=1`, disables nav collision, clears the locomotion target, and scans valid hostiles within an unscaled radius 10 of the cursor. It chooses the hostile nearest the cursor as the initial target and slides at speed 100 either to that target's position when already footprint-overlapping or to a footprint-separated point on the actor-to-target line; with no hostile it slides to the cursor. Only after that slide completes does it save the actor position and lock the camera, so cleanup returns to this post-slide point rather than the cast origin. The chosen initial target takes one immediate hit, then a separate loop performs up to five strikes by advancing circularly through the original hostile list and skipping entries that are no longer valid. Targets are not removed after a hit: with one surviving hostile, it receives all five loop strikes in addition to the initial hit, matching the translation token's six-hit ceiling. Each loop strike teleports the caster directly to the target's current position with a random horizontal facing, chooses one of four strike animations, waits 0.1 seconds to contact, deals damage, and waits the remaining 0.1 seconds. The referenced `timeBetweenStrikes` field is never authored, so there is no evidenced additional numeric inter-strike delay. Deactivation teleports to the saved post-slide point, unlocks the camera, re-enables nav collision, and resets animation; `EndImmunity` performs no explicit attribute removal, so scoped ability teardown must own removal of the unretained `Immune` handle.
- **Implemented - Quantum Blink:** Maldri's active now validates a range-35 cursor destination against the current navmesh, slides at the authored speed 100 to combined-footprint range of the nearest radius-10 cursor target or to the cursor, retains that post-slide return point, and executes the initial hit plus five cyclic strikes. Follow-up strikes teleport to each target's current position, rotate through all four recovered strike poses, apply the paired-rank 4-16/2-8 Spacetime/Physical melee damage, and emit the recovered large-hit effect. Odd ranks retain the 12-second cooldown while even ranks use 18 seconds. Server damage authority treats Maldri as immune for the scoped run, then returns him to the saved point, plays the one-second relaxation, clears immunity, and releases. Camera locking and nav-collision toggling remain client-owned presentation invoked by the accepted packaged ability.
- **Resolved - Bio random pool:** `BioRandom2`/Roar of Derision is chunk 428 and inherits the target-owned charge template. It charges up to range 30 at speed multiplier 2, and only `onFinishCharge` performs the payload around the caster's resulting position: every other valid hostile creature within radius 8 receives `RoarTaunt` (chunk 338), then the caster receives `RoarBuff` (chunk 1002). The taunt is the shared taunt template with duration 5/3 seconds by rank; the unique self buff grants exactly 0.25 DamageReduction for 6/3 seconds. The callback does not deal damage, and the area taunt and self buff are requested after charge completion rather than on pass-through.
- **Implemented - Roar of Derision paired ranks:** odd ranks retain the five-second area taunt and six-second self protection, while even ranks use three seconds for both retained effects.
- **Resolved - Necro random pool:** `NecroRandom`/Life Force Siphon is chunk 64 with modifier chunk 89. After a 0.3-second warmup it requires the selected target still be a valid hostile creature, then pays once, grants the caster a unique 50%/25% DamageReduction modifier, attaches caster-to-target beam effects, and performs six drain ticks. A tick deals fixed 10/8 Supernatural/Energy damage with coefficient 0.05 and `IsChannel | IsDoT`; only accepted damage heals the caster for the same ranked amount with coefficient 0.05 and `IsHoT`. The first tick is immediate after admission, subsequent waits are `1.0 * (1 - ChannelTimeDecrease)` clamped to 0.05, and the loop ends early if the target becomes invalid or exceeds distance 12. Natural completion or deactivation removes effects; deactivation also deletes the caster modifier, while the modifier itself otherwise waits forever and deactivates on caster death. The authored release calculation is warmup plus all six adjusted intervals plus the 0.766667-second ending animation.
- **Implemented - Life Force Siphon protection:** the first valid rank-one drain tick now activates the recovered 50-percent caster damage reduction. Incoming NPC melee and projectile authority consumes it until the channel completes, loses its target, is interrupted, or is canceled by the hero lifecycle.
- **Resolved - Plasma random pool:** `PlasmaRandom`/Meteor Strike is chunk 904 with stun modifier chunk 549. It waits to 0.1 seconds, emits muzzle and cursor impact presentation, waits another 0.5 seconds, then pays and scans `3 * (1 + AoERadius)` around the cursor; each valid hostile takes 12-18 Elements/Physical AoE damage with coefficient 0.05 and only accepted hits receive the unique stun for 3/2 seconds. `PlasmaRandom_WebbedLightning` is chunk 840 with shock modifier chunk 721: after a channel-adjusted 1-second charge it divides the full circle into 12 arcs, picks at most one best target per arc (using temporary range-40 endpoint objects when an arc is empty), and launches the shared piercing projectile path at the collected targets. A target can take standard 6-30 Elements/Energy projectile damage only once per activation; the modifier chance is 100%. Shock lasts 3/1 seconds plus 50% of that base duration per `LightningRogueActiveModifier` stack, is blocked by `ImmuneToStunned` outside chain games, and immobilizes the recipient. Deactivation clears every temporary endpoint and the template-owned projectile state.
- **Implemented - Webbed Lightning:** the shared burst runtime accepts all twelve authored projectiles at their common one-second launch boundary, executes the recovered sector target selection, range-40 endpoint fallback, one-hit-per-target collision rule, and 6-30 Elements/Energy damage, then retains one visible three-second shock modifier on every uniquely damaged survivor. Endpoint positions remain authoritative without publishing the Lua template's temporary targetable endpoint objects.
- **Closed authored orphan - Webbed Lightning shock scaling:** the shipped modifier adds 50 percent of its base duration for every `LightningRogueActiveModifier` stack, but the complete indexed content corpus contains that identifier only in Webbed Lightning chunk `721`. Ride the Lightning chunk `129` creates an empty `nModifier_LightningRogueActive` helper table, registers only `LightningRogueActive`, and never registers or requests `LightningRogueActiveModifier`; no shipped content path can create a stack. The exact shock duration is therefore the unscaled base: three seconds on odd ranks and one second on even ranks.
- **Resolved - Spacetime random pool:** `SpacetimeRandom`/Dimensional Rift is chunk 772 and is a one-tick targeted-area modifier cast: at 0.17 seconds it applies `BanishedModifier` (chunk 370) to valid hostiles within radius 6 and releases at 0.7 seconds. Banish is unique for 8/6 seconds, is rejected by `ImmuneToBanish` outside chain games, carries `IsRoot | IsBanish | IsSilence`, and sets Intangible, Immobilized, Immune, and RejectModifiers. `SpacetimeRandom2`/Delaying Sphere is chunk 760 with drag modifier chunk 238. At 0.233333 seconds it pays and creates a team-owned sphere at the caster's current position, not the cursor, with radius 8/5.5 and lifetime 12 seconds. Every scheduler yield it admits hostile projectile types or organic damageable types entering the sphere, applies one unique zero-duration drag modifier using the cast snapshot/rank, and removes that exact GUID as soon as the object leaves; final cleanup removes all remaining instances, the effect, and the sphere object. Drag applies -0.60 MovementSpeedBuff and -0.40 AttackSpeedScale.
- **Resolved - Vex runtime gaps:** `TimeRavagerBasic` is normalized to its authored melee kind while retaining its multi-hit fifth animation, `TimeRavagerSupport` accepts destinations connected by a valid multi-polygon navmesh route, and `SpacetimeRandom2` owns one attached sphere effect for the full duration without replaying the sphere visual for each entering enemy or projectile.
- **Implemented - Time Ravager Basic slow:** every accepted nonlethal Time Ravager Basic hit now requests the packaged `TimeRavagerBasicModifier`. One retained target modifier lasts five seconds, refreshes on every hit, and stacks to five; each stack applies the authored `-0.10 MovementSpeedBuff` and `-0.05 AttackSpeedScale`, giving maximum movement and attack scales of `0.50` and `0.75`. The same modifier instance publishes its current stack count, participates in the zone effect inventory, and clears authoritative speed operands and presentation on expiry or session teardown. The fifth combo row's two separate damage commits can therefore add two stacks as authored.
- **Implemented - Dimensional Rift:** all twenty Spacetime random-slot owners now use shared status-area authority. The cast validates its range-20 cursor center, pays and enters cooldown, presents the muzzle and radius-six rift at 0.17 seconds, and applies an eight-second `BanishedModifier` to every admitted hostile. Banish pauses NPC attack/pursuit authority and rejects damage while active; each modifier has an independent instance and exact-expiry cleanup, so overlapping server casts cannot let an older expiry clear a newer banish.
- **Resolved - Tech random pool:** `TechRandom2`/Omni Shield is chunk 717 with modifier chunk 396. At 0.1 seconds it pays, deletes every modifier on the caster matching `IsDebuff`, then applies the shield and releases at 0.8 seconds. For 4/3 seconds the shield's `ImmuneToDamage` callback unconditionally returns true and it also adds `ImmuneToDebuffs = 1`; deactivation removes its visual. `TechRandom3`/Charged Fist is chunk 980 with taunt chunk 139 and otherwise uses the melee template: at 0.3 seconds its 4-unit, 90-degree melee arc deals 30-50/25-35 Technology/Physical damage with coefficient 0.05, and accepted recipients receive the shared taunt with duration 6/3 seconds and aggro 100; release is 0.9 seconds.
- **Resolved - trap placement contract:** The shared trap template is chunk 94. It snapshots caster ID, attributes, target position, and orientation before the cast wait, pays after `castTime`, creates the authored trap noun at that saved target position and orientation, assigns caster team and owner ID, mirrors base attributes, and optionally emits muzzle/trap effects. Unless `aggroSelf` is true it temporarily redirects the caster's aggro delegate to the trap, retakes the caster snapshot for the trap, then restores the old delegate; `TurretTrap` sets `aggroSelf = true` and skips that redirect. `ClaymoreTrap` (chunk 54) costs 12 with coefficient 0.06, cooldown 12, range 2.5, cast/release timing 0.2/0.7, and creates `ClaymoreTrap.Noun`; its passive (chunk 39) releases the trap after 0.4 seconds and requests `ClaymoreTrapDetonate` after 15 seconds if nothing else triggers it. Detonation (chunk 323) waits 0.2 seconds, performs a full-circle radius-4 cone for 15-21 Technology/Energy damage with coefficient 0.05, applies `ClaymoreTrap_Daze` (chunk 554; 6/3 seconds, 0.5 speed adjustment) through the cone path, then deletes the trap. `TurretTrap` (chunk 328) costs 18 with coefficient 0.09, cooldown 12, range 4, cast/release timing 0.3/0.7, and creates `TurretTrap.Noun`. Its required `TurretLaser` (chunk 513) is a no-mana instant cast with cooldown 1, range 10, 3-5 Technology/Energy damage with coefficient 0.05, immediate hit timing, and 0.25-second release.
- **Resolved - Pipe Bomb contract:** `PipeBomb` (chunk 350) is custom rather than a shared trap-template instance. At 0.3 seconds it creates `PipeBomb.Noun` exactly 3.5 units forward from the caster using the caster position/facing, not the cursor, copies team/snapshot/base attributes, and finishes placement at 0.6 seconds; its Lua does not call `PayCooldownAndMana` or assign an owner ID even though the definition declares mana 24 with coefficient 0.12, cooldown 12, and range 4. Client native `GetFacing` (`sub_A01010` -> `sub_A176A0`) rotates a fixed local basis through the object's XYZW quaternion using `sub_44BE50`; initializer `0xF77F20` copies the executable triplet at `0x117416C`, exactly `(0,1,0)`, into that basis. Therefore the authored forward axis is local `+Y`, not `+X`: after normalizing `q=(x,y,z,w)`, facing is `(2*(x*y-w*z), 1-2*(x*x+z*z), 2*(y*z+w*x))`, and placement is caster position plus `3.5 * facing`. The adjacent natives independently confirm the basis ordering: right uses local `+Z` and up uses local `+X`. The unique passive (chunk 596) immediately taunts valid hostiles within radius 8 using raw modifier GUID `0x658bb220`, waits 3 seconds, removes those taunts, then scans the same radius 8 again: the authored `radius = 5` field is never read. Every valid hostile receives `PipeBombStunModifier` (chunk 26; 2/1 seconds, blocked by `ImmuneToStunned` outside chain games, Immobilized) before taking 9-15 Technology/Energy AoE damage with coefficient 0.05, so stun is neither conditional on accepted damage nor ordered after it; the bomb deletes itself afterward. A `TookDamage` event that reduces the bomb to zero HP instead emits its broken event and deletes it without detonating.
- **Implemented - Pipe Bomb control:** the placed bomb is now a targetable player-aligned zone actor. Every valid hostile within radius eight receives the raw bomb-owned taunt for the full three-second fuse and pursues the bomb rather than Seraph-XS. At detonation, every surviving radius-eight target receives the exact two-second `PipeBombStunModifier` before the 9-15 Technology/Energy damage packets are published, independently of damage acceptance. Taunt expiry reacquires ordinary player-aligned targets, while explosion removes the bomb actor and visual object. Damage that destroys the bomb now cancels its retained fuse, publishes `cyber_trapper_trapBroken.ServerEventDef`, and deletes it without detonating; Turret Trap shares that immediate destruction boundary.
- **Resolved - trap noun AI boundary:** The decoded `AssetData_Binary.package` noun, AI-definition, phase, and `NonPlayerClass` resources close this boundary. `ClaymoreTrap.Noun` links an AI definition but no `NonPlayerClass`, so native `sub_9E4040` uses its strict 20-unit missing-class perception-circle fallback; its AI definition has one phase and that phase's sole ability is `ClaymoreTrapDetonate`. Chunk 39 arms it by waiting 0.4 seconds and calling `ReleaseAgent`, while independently waiting 15 seconds and requesting the same detonation at the trap position with no target. Model early detonation from the trap's actor-local hostile/best-target path once a target enters that strict 20-unit perimeter, with the 15-second request as the fallback; exact threat ranking and scheduler tick remain generic AI policy, not trap-authored constants. `TurretTrap.Noun` links `TurretTrap.NonPlayerClass`, whose decoded `aggroRange=12` and `alertRange=4` give a strict 12-unit perception circle, plus a one-phase AI definition whose sole ability is `TurretLaser`; ordinary admission still requires the laser's 10-unit range and one-second cooldown. Chunk 808, not the shared placement template, enforces the exact 10-second lifetime: it waits 10 seconds, emits `cyber_trapper_autoturret_repack.ServerEventDef`, and marks the turret for deletion. If damage first reduces it to zero HP, its `TookDamage` handler instead emits `cyber_trapper_trapBroken.ServerEventDef` and deletes it.

Claymore Trap now executes across all five owning random-slot variants. The
shared trap path validates the ground point and authored range, spends power,
reserves its content-derived cooldown and release, creates an owned
`ClaymoreTrap.Noun`, retains concurrent trap identities, detonates after the
recovered 15-second timeout plus 0.2-second fuse or after an armed hostile
enters its strict 20-unit perception circle and starts that same fuse. Accepted blast hits receive the
six-second `ClaymoreTrap_Daze` 50-percent movement slow. The runtime preserves
15-21 Technology/Energy radius-four damage, normal critical/death/stat
publication, concurrent modifier ownership, trap deletion, and reset-safe
cancellation.

Meteor Strike and Charged Fist now retain their recovered rank-one definitions
across all ten owning random slots. Meteor Strike presents its recovered
cursor-centered impact at 0.1 seconds and performs its radius-three damage scan
at 0.6 seconds. Ranged cursor casts require an in-range cursor but no longer
require the hero to possess a walkable path to that cursor; movement abilities
retain their shape-specific navigation validation. Charged Fist uses
the shared pursuing melee path with its four-unit arc, 90-degree facing, and
30-50 Technology/Physical damage. Meteor Strike now applies the recovered
three-second stun and modifier lifecycle to every accepted surviving hit.
Meteor Strike now also maps its recovered `fire_summon_effect` cursor impact
to the shared cursor-area hit contract, so Fiery Eruption is admitted and its
native impact sound/effect occurs instead of the cast being rejected before
presentation.
Charged Fist now also applies its recovered six-second taunt to each accepted
surviving hit, immediately transferring NPC action authority to the caster and
returning the NPC to ordinary nearest-hostile selection when the taunt expires.

Omni Shield now executes across all five owning random slots through the same
self-modifier lifecycle. It spends the recovered power, reserves its cooldown,
publishes the shield modifier/effect for four seconds, and rejects hostile NPC
melee and projectile damage while active. Activation now removes every tracked
debuff modifier targeting the caster, cancels its expiry, clears the matching
sleep, stun, silence, root, poison, fear, or Physical/Energy vulnerability
authority, and publishes modifier deletion for both inventory-backed and
specialized retained runs.
The shared NPC modifier-admission boundary now also rejects newly arriving
poison, silence, sleep, stun, root, fear, and physical or Energy vulnerability
effects for the shield's full lifetime without suppressing harmless attack
presentation.

Binary Sentinel Support now executes across all four Magnos variants through
the shared cone runtime with its recovered 150-degree, ten-unit scan,
distance-scaled 30 Spacetime/Energy damage, power, cooldown, and timing. Ranks
five and six replace distance falloff with the authored fixed 0.625 multiplier;
ranks seven and eight use 24 power plus its 0.12 coefficient, and rank eight
amplifies captains and bosses by 1.5 with the large-target impact. Its ordinary
knockback modifier supplies its own eight-unit distance and speed-20 operands;
the absent request scalar is therefore intentional. Bosses use the stationary
1.6-second push reaction because the omitted float property reads as zero.
Binary Sentinel Active's separately
recovered pull now executes through shared navigation-safe forced movement.

- **Resolved - Summon Beast:** The player ability is chunk 37; chunk 91 is the separate `TNXSummonBeast` variant and chunk 78 only references it from Beast Charge. Summon Beast has cooldown 12/16, range 50, mana 28 with coefficient 0.14, manual cooldown, and release 0.75. It looks up the caster's unique zero-duration `BeastPetTracker`; if that tracker has a living pet it first turns/waits near that pet, pays, and requests `BeastPetEnrage` on the existing pet. If the tracker is absent it pays and creates it; if no living pet is recorded it chooses the closest valid spawn position a caster-footprint-plus-4 distance forward, creates `BeastSentinelPet.Noun`, assigns owner/team and mirrored base attributes, stores it in the tracker, plays its spawn animation, and requests `BeastSentinelPetSpawn`. Recasting with a living pet has an intended effective mana cost of 50% of the normal stat-adjusted summon cost. The beast-specific tracker contract is chunk 651: setting a replacement deletes the old pet, dead pets resolve as none, and it clears its reference on scheduler yields; this is the namespace the root actually calls even though the unresolved import spelling `modifier_pet_tracker.lua` hashes to generic tracker chunk 850.
- **Resolved - Beast pet enrage:** `BeastPetEnrage` is chunk 59. At 0.36 seconds the pet receives unique `BeastPetRage`, then, only if its owner is alive and owns `BeastSentinelPassive`, every valid hostile within radius 5 receives `BeastPetRoarTaunt` using the pet's current snapshot at rank 1, or rank 3 while that owner's overdrive is active; release is 0.75 seconds. Rage lasts 8 seconds, adds 0.50 PhysicalDamageDoneIncrease and 0.20 BodyScale, and repeatedly heals the pet for 5% of its current maximum HP with an `IsHoT` descriptor, starting immediately and waiting 1 second between heals.
- **Resolved - Summon Sprite:** `SummonSprite` is chunk 529 with passive chunk 613. It declares cooldown 30, range 50, mana 24 with coefficient 0.12, and release 0.9, but its custom Lua tick never calls `PayCooldownAndMana`. Each activation independently chooses the closest valid point a caster-footprint-plus-4 distance forward, creates `SpritePet.Noun`, assigns owner/team and mirrored base attributes, and plays its spawn presentation; there is no tracker, uniqueness check, replacement, or target/cursor use in this Lua path. The pet's unique passive adds its trail, polls every 0.5 seconds until either its owner dies or 30 seconds elapse, emits the spawn-out effect at the pet's then-current position, and deletes the pet; deactivation removes the trail. Thus multiple concurrent sprites are permitted by the authored Lua if admission allows a recast.
- **Resolved - Sporogenesis:** `Sporogenesis` is chunk 591, not a summon. It declares cooldown 8 and mana 25 with coefficient 0.125, but its custom tick does not call `PayCooldownAndMana`. At 0.1 seconds it scans all damageable objects within radius 8 around the caster and, for each friendly-valid target, emits the friendly impact, deletes every modifier matching `IsDebuff` without a channel exception, and requests `SporogenesisHot`; it releases at 0.7 seconds. That inline `IsBuff | IsHoT` modifier uses the cast snapshot, heals exactly 6 with coefficient 0.05 five times beginning immediately with 1-second waits between subsequent ticks, and has authored duration 4.1 seconds. Its `Shroom_Cloud` asset being preloaded under the `SleepingCloud` owner label is presentation only and does not create a persistent area object.
- **Resolved - Affliction Bolt:** `AfflictionBolt` is chunk 765 with curse chunk 798 and projectile-template chunk 38. It declares range 80, mana 24 with coefficient 0.12, cooldown 15, cast/release timing 0.18/0.6, projectile speed 12, lifetime 12, and no direct damage payload. The owned homing projectile ignores ordinary creature and ground collision, carries a radius-3 trigger, turn rate 6, and requests `AfflictionCurse` directly on each valid hostile entering that trigger if the target is neither already cursed nor stealthed; applying the curse is not gated by an impact-damage result. Server contact tests the complete interval between simulation samples against the hostile's raised collision body so the trigger cannot tunnel through a moving target or compare an elevated projectile to a ground anchor. Its homing validator rejects cursed/stealthed targets and searches distance-sorted alternatives within 20 of the projectile while excluding destructibles, allowing the same projectile to curse multiple distinct contacts before expiry. `AfflictionCurse` is `UniqueIrreplaceable`, lasts 8.25 seconds, is blocked by `ImmuneToCursed` outside chain games, and has no initial damage; it deals four ticks at 2, 4, 6, and 8 seconds for exactly 5 Supernatural/Energy damage with coefficient 0.05 and `IsDoT | IsDebuff | IsEnergyDamage`.
- **Implemented - Affliction Bolt multi-contact flight:** Affliction Bolt now remains active across its authored 144-unit, 12-second flight instead of resolving once against the selected target. A 100-millisecond live scan admits every distinct uncursed, unstealthed, published, non-fixture hostile inside radius three, creates an independent curse modifier, schedules that contact's four exact damage ticks and expiry, and retargets the nearest eligible alternative within 20 units through a turn-rate-six projectile locomotion update. Initial admission rejects already cursed or stealthed selected targets. Stealther casts own the corresponding zone stealth state through successful hit, interrupted attack, rollback, and modifier cleanup. Switch, reset, disconnect, scheduling failure, and final projectile expiry clean every remaining contact.
- **Resolved - Rooting Plague:** `RootingPlague` is chunk 546 and uses the instant-cast template with `LifePlague` chunk 282 and `BioEntangleModifier` chunk 246. It has range 30, mana 24 with coefficient 0.12, cooldown 16, hit timing 0, release 1.2, and an explicitly zero damage/coefficient payload; the selected target instead receives both modifiers. Bio Entangle is the shared root policy for 8/6 seconds and is rejected by root immunity outside chain games. Life Plague is a unique 8-second `IsDisease | IsDoT | IsDebuff` with no initial damage and eight 1-second ticks of 4 Life/Energy damage at coefficient 0.05. After every accepted tick it scans radius 5 around the infected target and requests `LifePlagueSpread` on each other alive damageable enemy lacking the plague-immunity, primary-infection, and spreading GUIDs. The spread is itself unique, lasts 6 seconds, deals six 1-second ticks of 3 with the same type/source/coefficient, and recursively performs the same accepted-tick spread. On primary deactivation the target is unconditionally asked to receive the plague-immunity marker; spread deactivation requests that marker only when it is not already active.
- **Resolved - Terrify:** `Terrify` is the self-contained chunk 418 plus shared fear chunk 580 and shared fear-template chunk 132 (`Modifiers/0xB2D63002.lua`, SHA-256 `07d4bbb7a06c8e1841d631619412ce1dbb8618a147079fb655c5ce19107539e6`). It is a creature-target projectile with range 35, speed 40, mana 18 with coefficient 0.09, cast/release timing 0.17/0.6, and rank-indexed cooldown 8 on odd ranks or 12 on even ranks. The primary modifier is ordinary `TerrifyDebuff` on ranks 1-4 and 7-8, but `TerrifyFreezeDebuff` on ranks 5-6. Both are curse/fear DoTs blocked by `ImmuneToTerrified` outside chain games and run damage concurrently with control: odd ranks deal five immediate-and-then-1-second ticks of 6 Supernatural/Energy damage, even ranks deal three such ticks of 7, all at coefficient 0.05; authored lifetimes are 4.1 and 2.1 seconds respectively. Ordinary Terrify invokes the template's transient `0.5` speed modifier, uniformly samples a distance from 5 through 10 units and a full-circle heading from the target's current position, projects through `GetClosestPosition`, retries at most three times until reachable, then uses exact-point movement until within `1.5 * footprint`. It chooses the next destination when that movement/control loop completes rather than on a fixed repath timer. The rank-5/6 freeze branch immobilizes instead. Only ranks 7-8 have radius 3: when the primary damaged target is an NPC, other valid hostiles around it receive `TerrifyDebuffNoDot`, which applies the fear movement without the damage component. The primary target and caster are excluded from that splash.
- **Implemented - Terrify rank branches:** even ranks now use the recovered twelve-second cooldown, three fixed-seven ticks, and 2.1-second control lifetime; odd ranks retain five fixed-six ticks over 4.1 seconds. Ranks five and six replace flee movement with the exact retained `TerrifyFreezeDebuff` stun. Ranks seven and eight now apply `TerrifyDebuffNoDot` to every other living non-fixture hostile inside radius three around the primary impact, exclude the primary target, and run the ordinary fear movement without adding secondary damage ticks. Every auxiliary modifier shares the projectile run's expiry, reset, and teardown cleanup.
- **Implemented - Arboreal Might paired ranks and growth:** odd ranks retain five stacks at 10-percent damage each, while even ranks cap at two stacks worth 20 percent each. Both variants keep the recovered 15-second lifetime and nearby-hostile-death refresh. Every accepted stack now publishes the authored six-percent Body Scale operand, while expiry and squad switching restore it to zero. Recasting at the applicable stack cap preserves both the existing scale and expiry instead of refreshing either. The September 6 report also exposed the missing retained `status_enraged.ServerEventDef` explicitly authored by modifier chunk 312; this now uses the shared attached-effect lifetime, independently of the one-shot `Arboreal_Might_Activate` event.
- **Resolved - Psistorm:** `Psistorm` is chunk 425 with its silence modifier inline. It declares range 35, mana 18 with coefficient 0.09, cooldown 15, cast/release timing 0.17/1.0, pool radius 6, pool duration 5/3, and fixed 7 Supernatural/Energy AoE DoT damage at coefficient 0.05; its custom tick does not call `PayCooldownAndMana`. At 0.17 seconds it reads the target position, creates `shadow_spirit_pool_aoe.Noun` there with the cast snapshot, and starts a trigger-volume thread. The aura requests one zero-duration `PsistormDebuff` for each enemy while inside, removes that exact instance on exit, and clears all remaining instances when the pool ends; the modifier is blocked by `ImmuneToSilence` outside chain games and adds `Silence = 1` while active. Damage is independent of modifier admission and applies immediately to every currently tracked aura target, then once per second. The authored loop count is `(poolDuration / 1 + 1) * (1 + AoEDurationIncrease)`, producing six base ticks at rank 1 and four at rank 2 before final modifier, trigger, and pool-object cleanup.
- **Implemented - Psistorm:** the rank-one cast now creates and retains the authored `shadow_spirit_pool_aoe.Noun` at the cursor for five seconds, dynamically admits and removes hostile NPC silence modifiers, and deals all six fixed-7 Supernatural/Energy pulses through shared damage, critical, death, loot, floating-text, encounter, and statistics authority. The pool object and every retained modifier are removed at the final boundary or lifecycle teardown.
- **Implemented - Psistorm presentation safety:** the pool spawn now publishes only its authored `.Noun` object instead of also misusing that noun hash as a `ServerEventDef`, preventing the client access violation captured immediately after the first delayed tick. Aura admission still supports a separate activation effect when one is actually authored.
- **Implemented - Psistorm rank cadence:** even ranks now shorten the retained pool from five seconds and six pulses to the recovered three seconds and four pulses while preserving dynamic silence membership and cleanup.
- **Implemented - Psistorm special-action suppression:** NPCs carrying the live Psistorm silence can no longer begin rooted or direct healing, Reparatron repair, resurrection, charge-up, energy-buff, Voltroid discharge, pack cower, Grappling Pulsar pull, Chrono Striker flee, Nomad shield or grenade, Ruption magma, or Citadel special behavior. Melee families fall through to their ordinary attack profile, while resurrection retries when silence clears. A delayed special/support commit already in progress reschedules itself for the greatest remaining sleep, stun, or silence lifetime and then invokes its original continuation, preventing the action from landing through silence without discarding its scheduler.
- **Resolved - Cast Soul Link:** `CastSoulLink` is chunk 518 with `SoulLinkModifier` chunk 417. It declares cooldown 24, mana 16 with coefficient 0.08, manual cooldown, hit/release timing 0.233333/1.3, and applies the unique-irreplaceable 12-second buff to the caster immediately after explicitly paying. The modifier adds `DistributeDamageAmongSquad = 1` and the link effect for its entire lifetime. On activation and every scheduler yield it reads the player's current squad-passive GUID list, current deck index, and overdrive state: while overdrive is active it requests every squad passive except the current deck slot on the linked hero and stores the returned modifier IDs; when overdrive ends it deletes all stored borrowed passives, and deactivation does the same cleanup before removing the effect. The Lua does not select allies or calculate shares itself; actual damage sharing is the native meaning of `DistributeDamageAmongSquad`.
- **Resolved - Soul Ravager Active:** `SoulRavagerActive` is chunk 125 and uses projectile-template chunk 38. It declares range 50, mana 20 with coefficient 0.10, cooldown 12, 7-14 Supernatural/Energy AoE projectile damage with coefficient 0.05, radius 3, speed 20, distance 10, first-shot timing 0.1, subsequent-shot interval 0.01, and release 0.75. At cast start it consumes up to five souls through `SoulRavagerPassive`; soul counts 0-5 produce 6/8/9/10/11/12 non-homing projectiles distributed evenly around the full circle. One per-activation target counter allows the shared standard-damage callback at most four times for any target across the entire barrage. Although this chunk imports `modifier_stackable_silence.lua` and exposes its duration token for presentation, it never requests that modifier; do not add silence to the server payload without further evidence.
- **Resolved - Soul Ravager Support:** `SoulRavagerSupport` is chunk 145 and uses projectile-template chunk 38. It is shared, pursues a creature target within range 20, costs 12 mana with coefficient 0.06, has cooldown 4/8, and launches homing radius-zero projectiles at speed 12 for distance 25. Damage is 9-15 at rank 1 or 6-12 at rank 2, Supernatural/Energy with coefficient 0.05; first-shot timing is 0.3, the interval is 0.4, and release is 2. It consumes up to five souls through `SoulRavagerPassive`; soul counts 0-5 produce 3/3/4/4/5/5 shots. Immediately before the inherited projectile tick it temporarily adds `LifeSteal = 0.5`, retakes the caster attribute snapshot, and removes that live attribute modifier, so the launched projectiles retain 50% lifesteal in their snapshot rather than granting the caster a persistent buff.
- **Implemented - Soul Ravager Support rank branches:** the volley now retains its selected hostile and authored 0.3-second first shot plus 0.4-second intervals instead of inheriting Lightning Tempest Active's randomized target/channel policy. Even ranks use 6-12 damage and an eight-second cooldown; odd ranks retain 9-15 and four seconds. The recovered soul table still expands both variants to 3/3/4/4/5/5 shots with 50-percent lifesteal.
- **Implemented - equipped Life Steal:** authenticated item attribute 35 now restores the deployed hero from every accepted player-caused damage transition using the recovered applied-damage fraction, target healing reduction, and maximum-health cap. Soul Ravager Support retains its separately snapshotted 50-percent bonus, so equipment and authored ability Life Steal remain additive rather than replacing one another.
- **Resolved - Shadow Ravager Support:** `ShadowRavagerSupport` is chunk 128 with stealth modifier chunk 859. It declares cooldown 15, mana 14 with coefficient 0.07, release 0.6, and no descriptors; its custom Lua does not call `PayCooldownAndMana`. On activation it starts the support animation and immediately requests both `DamageImmunityModifier` and `ShadowRavagerStealthModifier` on the caster from the cast snapshot. The unique stealth modifier lasts 6 seconds at every rank, sets supernatural stealth, and adds `AutoCrit = 1`; deactivation clears stealth if the caster still exists, while modifier-owned attribute cleanup removes AutoCrit. It handles both `TookDamage` and `UsedAbility`: the damage event returns false directly, and the first ability-use event records a private one-shot guard, explicitly clears stealth, and returns true, while a repeated ability-use event returns false. Preserve those separate event outcomes rather than treating the buff as an unconditional six-second invisibility window.
- **Implemented - Shadow Ravager Support ability break:** the shared accepted-action boundary now ends the active six-second stealth, damage immunity, AutoCrit bonus, and modifier presentation on the first subsequently accepted character ability. Rejected attempts do not consume stealth, damage still does not consume it, and the activation acknowledgment is explicitly excluded so the modifier does not remove itself.
- **Implemented - Trapper Stealth ability break:** the same accepted-action boundary now removes Seraph-XS's active Technology stealth on its first committed character ability. Rejected commands leave the passive untouched, while its existing incoming/outgoing-damage break, target admission, three-second exit, and twelve-second re-entry cycle remain authoritative.
- **Resolved - Binary Sentinel Support:** `BinarySentinelSupport` is chunk 816. It pays at 0.23 seconds, scans damageable objects from furthest to nearest inside radius `10 * (1 + AoERadius)`, excludes the caster, and admits valid hostiles in the forward 150-degree cone. Its base payload is fixed 30 Spacetime/Energy AoE damage with coefficient 0.05 and a distance multiplier `0.5 + 0.5 * (1 - distance / 10)`, clamped at the radius; ranks 5-6 instead force the multiplier to 0.625 regardless of distance. Rank 8, but not rank 7, multiplies damage by another 1.5 against non-minions and minions carrying `EliteModifier`, and emits the large-target impact for that same class. Every accepted hit then receives `BinarySentinel_KnockbackModifier`, except bosses receive `PushPullBossModifier`; the request uses the caster snapshot and rank but no explicit push/pull scalar. Mana is 18 with coefficient 0.09 at ranks 1-6 and 24 with coefficient 0.12 at ranks 7-8, cooldown is 10, and release is 1.0.
- **Implemented - Binary Sentinel Support rank branches:** ranks five and six now replace ordinary distance falloff with the recovered fixed 0.625 damage multiplier. Ranks seven and eight use 24 power plus the 0.12 coefficient, while rank eight alone multiplies captain and boss damage by 1.5 and emits the recovered large-target impact.
- **Resolved - Binary Sentinel Support knockback:** concrete modifier chunk 253 (`Modifiers/0xD9B0C112.lua`, SHA-256 `83b1a051d6f73bed547478e3378ff769b1e5977ff3857d7b6264ff52f0929346`) inherits the shared knockback template and supplies its own exact operands: speed `20`, desired distance `8`, `react_knockback`, `binary_sentinel_push_attractor.ServerEventDef`, and a `0.25`-second outro. Activation derives the push origin from the valid initiator's ground position, so the support request intentionally needs no movement scalar. Root and knockback-immunity admission, jump completion, locomotion stop, animation reset, and cleanup follow the already recovered shared template. Bosses receive chunk 11 `PushPullBossModifier`; its absent float property reads as zero, selecting the stationary push reaction rather than the scalar-one pull branch, with no attraction effect and the authored `1.600000024`-second release.
- **Resolved - Fire Tempest Support:** The live `FireTempestSupport` is chunk 185; chunk 683 registers the separate legacy `FireTempestSupportOld` channel and must not replace it. The live targeted-area cast has range 35, radius 6, cooldown 14, mana 20 with coefficient 0.10, pays at 0.17 seconds, and releases at 0.8. Its effect-object thread performs four base iterations at one-second intervals, multiplied by `1 + AoEDurationIncrease`; the shared template computes an AoERadius-scaled radius but accidentally scans with the unscaled authored radius 6. On the initial iteration all valid hostiles receive the unique one-second `FireTempestHealDebuff`, but only the first hostile encountered across the whole cast takes the one-time 9-18 Elements/Energy burst with coefficient 0.05; `dealDamageOnInitialTick = false` suppresses the ordinary tick damage. Each later iteration deals fixed 4 damage with the same type/coefficient to every valid hostile, and every iteration refreshes the heal debuff, which adds `HealingReduction = 0.5`. Friendly targets pass through a zero-point healing call only. The effect object is deleted after the final interval.
- **Implemented - Fire Tempest Support healing reduction:** every hostile admitted by each Sphere of Transfusion pulse now receives or refreshes its unique one-second `FireTempestHealDebuff`, including initial-pulse hostiles outside the one-time damage target. The retained NPC modifier reduces ordinary healing and corpse-consumption healing by exactly 50 percent and owns modifier presentation, refresh, expiry, reset, and teardown cleanup independently from the area's damage results.
- **Resolved - Voodoo Tempest Support:** `VoodooTempestSupport` is chunk 564 with `VoodooTempestWeaken` chunk 644 and targeted-area template chunk 956. It has range 35, radius 8, cooldown 20, mana 16 with coefficient 0.08, pays at 0.17 seconds, and releases at 0.7. Its single base iteration, multiplied by `1 + AoEDurationIncrease`, applies the modifier to every valid hostile around the cursor without dealing damage; as in Fire Tempest Support, the template calculates but does not use the AoERadius-scaled scan radius. The unique `IsDebuff | IsCurse` modifier is rejected by `ImmuneToCursed` outside chain games, lasts 10/6 seconds, and applies rank-one/rank-two attributes respectively: `DamageBuff = -0.33/-0.25`, `PhysicalDamageIncrease = 0.50/0.25`, and `EnergyDamageIncrease = 0.50/0.25`. Its apparently opposing general-damage reduction and source-specific increases are both authored and should be preserved.
- **Implemented - Voodoo Tempest Support rank branches:** even ranks now use the recovered six-second curse with -0.25 general damage and +0.25 Physical/Energy damage-taken operands; odd ranks retain ten seconds, -0.33, and +0.50/+0.50.
- **Implemented - Voodoo Tempest Support:** all four Jinx variants now use shared status-area authority. The rank-one cast applies its radius-eight curse at the cursor with exact power, cooldown, timing, muzzle, area presentation, modifier identity, and ten-second lifecycle. NPC damage projection retains all three authored attributes additively: -0.33 general damage plus +0.50 physical and +0.50 energy ability damage, rather than collapsing the curse into a generic damage reduction.
- **Resolved - Field Medic Support:** `FieldMedicSupport` is chunk 65 (`Abilities/0x514433BB.lua`, SHA-256 `05267909b18f0a4fe33a302d4ad26912460a40ac398dddf4c1a056ac45001204`) with `FieldMedicHealthBuff` chunk 829 (`Modifiers/0x2D5E5A52.lua`, SHA-256 `93d97660d49c1df6f4b0af41eac47a36fd40d4540a3121313b23f76a344378bb`). The channel has range 10, cooldown 5, mana 32 with coefficient 0.16, a 0.4-second warmup, six fixed-10 healing ticks with coefficient 0.05, and an authored 0.5-second tick interval shortened by `ChannelTimeDecrease` but clamped to 0.05 seconds; release is the warmup plus all six adjusted intervals. It first resolves the selected friendly, otherwise the nearest valid friendly to the cursor within radius 10, otherwise the caster, and immediately requests the 15-second unique health buff on that target before warmup, payment, or revalidation. The buff adds 25 percent `MaxHealth`. At 0.4 seconds an invalid target aborts without paying; otherwise the cast pays once, heals immediately and after each interval until six ticks complete, stopping early if the target becomes invalid. An ally cast uses `cast_fieldmedicsupport`, attaches `effect_fieldmedicsupport_beam.ServerEventDef` to the caster with the ally as the effect target, and attaches `effect_fieldmedicsupport_target.ServerEventDef` to the ally. A self-cast instead uses `cast_fieldmedicsupport_self`, creates no beam, and attaches `effect_fieldmedicsupport_targetself.ServerEventDef` to Meditron. Deactivation resets the caster animation, removes the beam from the caster, and removes the corresponding target/self-target effect. Chunk 829 authors no additional world effect for the health buff; its presentation boundary is the registered modifier/icon lifecycle.
- **Implemented - Field Medic Support friendly targets:** the support path now honors an explicitly selected living co-op hero or targetable friendly companion within range 10, otherwise selects the nearest such friendly to the cursor within range 10, and falls back to Meditron. The chosen target receives all six healing ticks and the unique 15-second `FieldMedicHealthBuff`, which increases authoritative and displayed maximum health by exactly 25 percent. Hero and companion resource changes project directly to the owning player; the caster receives healing-dealt statistics and the selected owner receives healing-received statistics. A recipient that dies, switches, disconnects, despawns, or otherwise becomes invalid immediately cancels and releases the channel instead of leaving the caster's retained action occupied. Recasting on the same target refreshes rather than stacks the modifier. Expiry, reset, and teardown restore the exact base maximum, clamp excess current health, release the modifier identity, and publish the restored resource state. Ally casts now use `cast_fieldmedicsupport`, lease the authored caster beam with the selected ally as its secondary object, and attach the ordinary target effect there at the first pulse. Self-casts use `cast_fieldmedicsupport_self`, omit the beam, and attach the self-target effect. Normal completion and early invalidation reset the caster animation and hard-stop both leased presentation slots; reset and scheduling failure release retained attachment ownership. The retained health buff has no independently authored world effect.
- **Resolved - Poison Nova:** `LFPoisonRavager_PoisonNova` is chunk 548 with pointblank template chunk 324, `PoisonNovaModifier` chunk 666, and cooldown marker chunk 171. It pays at 0.233333 seconds, releases at 0.85, and scans valid hostiles inside `6 * (1 + AoERadius)`. Cooldown is 6 and mana is 12 with coefficient 0.06. Each accepted target takes 2-6 Life/Physical AoE damage with coefficient 0.05, and only a successful direct hit receives the poison: five fixed-4 Life/Energy DoT ticks with coefficient 0.05, every two seconds over 10 seconds at ranks 1-6 or every second over 5 seconds at ranks 7-8. At ranks 5-6, a cast made without `LFPoisonRavager_Support_CooldownUpgradeModifier` fully removes Poison Nova's newly applied cooldown after the attack and installs that unique marker; while the marker remains, later casts do not reset. The marker lifetime is the authored six-second cooldown scaled by the caster's cooldown scale and it deactivates on death. Ranks 7-8 only use doubled cast presentation and do not use this reset policy.
- **Implemented - Poison Nova:** every accepted surviving rank-one hit now starts the separate five-tick `PoisonNovaModifier` lifecycle, dealing its projected fixed-4 Life/Energy damage every two seconds. Recasts refresh that target's Poison Nova sequence and modifier instance without disturbing Viper's independently stacked passive poison. Ticks use ordinary critical-independent damage, floating-text, death, loot, statistics, switch, reset, and teardown authority.
- **Implemented - Poison Nova rank branches:** ranks five and six now reset the newly committed six-second cooldown on the first cast, present the recovered cooldown-upgrade modifier for that six-second window, and preserve the ordinary cooldown on the next cast while the marker exists. Ranks seven and eight retain five poison ticks but execute them every second over five seconds instead of every two seconds over ten seconds.
- **Implemented - Rooting Plague paired roots:** odd ranks retain an eight-second root while even ranks use six seconds. The primary and recursively spread disease remains eight seconds at every rank and keeps its independent modifier lifecycle.
- **Implemented - Summon Beast paired cooldowns:** odd ranks retain the recovered 12-second cooldown while even ranks use 16 seconds through the specialized pet runtime.
- **Implemented - summon enrage growth:** Summon Beast's eight-second Beast Pet Rage publishes its recovered 0.20 Body Scale operand when the retained modifier begins. The shared 15-second Fire Elemental Enrage used by Fire Tempest Active and Plasma Sentinel Active publishes that same growth only after its authored 1.25-second warmup. Both restore scale when the retained modifier expires; pet deletion remains the cleanup boundary for interrupted owner, replacement, switch, death, and teardown paths.
- **Implemented - Fire Elemental Enrage damage buff:** the first post-warmup pulse now installs the authored additive 0.50 companion DamageBuff. It amplifies every elemental aura pulse and Plasma Sentinel pet melee while combining additively with transferred companion damage buffs, then is removed on natural expiry, rollback, replacement, switch, death, reset, and teardown.
- **Implemented - Beast Pet Rage healing presentation:** every accepted five-percent maximum-health rage heal now publishes the healing combat event before the authoritative companion health update, including the immediate heal and each one-second retained tick.
- **Implemented - Enrage growth:** Sage's Enrage now publishes its recovered 0.12 Body Scale operand on the selected living ally alongside the unique modifier. Recasting on the same recipient replaces the modifier without compounding scale, and final retained expiry restores scale together with direct-attack damage.
- **Implemented - TC Shielded Sentinel Active paired taunt:** odd ranks retain the five-second accepted-hit taunt while even ranks now use the recovered three-second duration.
- **Implemented - shared even-rank projection:** one content-backed projection now applies the recovered even-rank operands before runtime-shape admission. Meteor Strike uses a two-second stun; Claymore Trap uses a three-second daze; Pipe Bomb uses a one-second stun; Life Force Siphon uses fixed-eight damage/healing ticks and 25-percent caster damage reduction; Webbed Lightning uses a one-second base shock; Entangling Rush uses 16-24 damage and a four-second root; Phantom Charge uses a one-second silence; Dimensional Rift uses a six-second banish; Time Ravager Support uses a one-second freeze; Omni Shield lasts three seconds; and Charged Fist uses 25-35 damage with a three-second taunt. Odd ranks retain their rank-one catalog projection.
- **Resolved - Missile Tempest Support:** `MissileTempestSupport` is chunk 693 with `RocketSlow` chunk 28 and `MissileTempest_Support_FlakUpgradeModifier` chunk 888. It targets the cursor at range 35 and scans radius `5 * (1 + AoERadius)`; declared mana is 8 with coefficient 0.04 and rank cooldowns are `1/6/1/6/6/12/1/6`, but the custom Lua never calls `PayCooldownAndMana`. Each burst ignores friendlies, triggers false collision on every hostile projectile, and requests `RocketSlow` before dealing 3-6 Technology/Physical AoE damage with coefficient 0.05 to each damageable hostile; the slow is 50 percent for alternating rank durations `6/4/6/4/6/4/6/4`. Ranks 5-6 fire at the saved cursor immediately, again after 0.4 seconds, and a third time 0.7 seconds later; other ranks fire once. At ranks 7-8 each burst removes the caster's existing Flak upgrade, yields once, then adds one stacking ten-second modifier per hostile projectile destroyed, capped at 10/4 stacks; each stack adds 5/12 percent attack speed, for caps of 50/48 percent. Creature hits do not add stacks.
- **Implemented - Missile Tempest Support Rocket Slow:** each accepted surviving creature hit receives the recovered 50-percent movement slow with retained modifier creation, expiry, reset, and teardown cleanup. Odd ranks use six seconds and even ranks use four seconds.
- **Implemented - Missile Tempest Support projectile collision:** retained hostile projectiles expose an authoritative live spatial snapshot and explicit deletion boundary. Each chaff burst destroys every active hostile projectile inside its authored cursor-centered radius five and publishes the projectile deletion alongside ordinary creature damage and Rocket Slow. Ranks five and six retain the saved cursor and execute the authored three bursts immediately, after 0.4 seconds, and after another 0.7 seconds even though the caster has already released. At ranks seven and eight, the burst removes the prior Flak modifier and grants one ten-second stack per projectile actually destroyed, capped at 10 stacks of 0.05 attack speed or 4 stacks of 0.12 respectively. The server publishes the resulting aggregate stack count once while preserving the authored gameplay result of repeated stacking requests.
- **Resolved - Repulsion Wave:** `RepulsionWave` is chunk 292 with nova template chunk 913 and slow chunk 791. It declares alternating rank cooldowns `6/4/6/4/6/4/6/4` and mana 10 with coefficient 0.05, hits at 0.466667 seconds, and releases at 0.833333, but neither the custom ability nor nova template calls `PayCooldownAndMana`. The spawned collision ring begins at radius 2, expands at speed 11 to radius 8, and deduplicates each creature and projectile for the life of the nova. Every valid hostile creature first receives the knockback modifier; ranks 5-6 then deal 10-14 Spacetime/Energy AoE damage with coefficient 0.04, ranks 7-8 instead apply a four-second 50-percent slow with zero damage, and ranks 1-4 add neither damage nor slow. Friendly projectiles are unchanged. Hostile projectiles are reflected rather than destroyed: their target is cleared, caster team and attribute snapshot are replaced, direction becomes radial from the nova center, horizontal speed falls linearly from 20 at radius 2 to 7 at radius 8, vertical speed becomes zero, and a client update is forced.
- **Implemented - Repulsion Wave rank branches:** all eight ranks now share the authored knockback while preserving alternating six/four-second cooldowns. Ranks five and six publish their 10-14 Spacetime/Energy area damage through ordinary critical, death, loot, floating-text, encounter, and statistics authority. Ranks seven and eight instead apply the exact four-second 50-percent movement slow through retained NPC status and modifier presentation.
- **Implemented - Repulsion Wave projectile reflection:** every retained hostile projectile inside Andromeda's radius-eight wave now cancels its former hostile collision schedule, clears its target, changes to the player team, snaps to its live authoritative position, redirects horizontally away from the nova center, and forces a client locomotion update. Reflected speed falls linearly from 20 at the authored inner radius two to 7 at outer radius eight. The reflected visual continues along its original authored range while the canceled hostile run can no longer damage a hero.
- **Resolved - Plasma Sentinel Support:** `PlasmaSentinelSupport` is chunk 723 with cone template chunk 170. It is a movement-permitted full-circle column around the caster with authored radius 5, cooldown 15, mana 18 with coefficient 0.09, hit timing 0.2, duration 8, interval 0.5, and fixed 3 Elements/Energy AoE DoT damage with coefficient 0.05 per tick. At 0.2 seconds the template pays, the override attaches the column effect and immediately releases the caster, then the template damages every valid hostile and repeats at 0.5-second intervals for 15 base ticks before waiting out the eight-second boundary; the declared 1.4-second release field is not used by this path. `AoEDurationIncrease` scales the duration, but although the cone template calculates an `AoERadius`-scaled radius, its object scan mistakenly uses the unscaled authored radius 5. Deactivation removes the column effect.
- **Implemented - AoE Duration item projection:** equipped `AoEDurationIncrease` now multiplies Lightning Tempest Active's 30-shot and Missile Tempest Active's 15-shot numeric loops, Psistorm's ranked six/four pulse loop, Fire Tempest Support's four iterations, and Voodoo Tempest Support's single iteration using Lua's whole positive loop admission. Plasma Sentinel Support instead scales its continuous eight-second lifetime and corresponding 15-tick cadence. The policy is content-marked to these six proven consumers and leaves all other ability, modifier, and control durations unchanged.
- **Implemented - Plasma Sentinel Support:** all four Zrin variants now use the shared timed-area runtime. It commits the exact rank-one power/cooldown contract, immediately releases movement at the 0.2-second hit boundary, follows the caster for fifteen half-second radius-five damage scans, publishes each accepted damage/death/stat transition, and owns attachment cleanup on completion, switch, reset, and teardown.
- **Resolved - Plasma Sentinel Active:** `PlasmaSentinelActive` is chunk 67 with `PlasmaSentinelPetModifier` chunk 307 and pet melee chunk 165. The ten-second self buff declares cooldown 30 and mana 20 with coefficient 0.10, but its custom Lua never calls `PayCooldownAndMana`; it listens for `TookDamage` and accepts at most two triggers. The shipped modifier chunk creates one `PlasmaSentinelPet.Noun` on its first activation and applies Fire Elemental Enrage when stacked, but the recovered player-facing Pain Hounds contract instead defines two orbiting shards, each of which breaks on an incoming hit and independently spawns a hound for 30 seconds. Server compatibility follows that explicit gameplay contract. Each pet assigns owner/team and mirrored base attributes and uses a one-second-cooldown, range-1.5 melee for 3-6 Elements/Physical damage with coefficient 0.05.
- **Implemented - Plasma Sentinel Active:** Zrin's active accepts without inventing the declared-but-unpaid power/cooldown call, attaches its exact ten-second two-shard shield, counts accepted incoming-damage events, and presents the recovered rock hit event. Each of the first two hits consumes one shard and creates a separately owned, separately expiring `PlasmaSentinelPet.Noun`; deterministic left/right navigation projection keeps the hounds from overlapping at spawn. The second hit removes the shield instead of enraging the first pet. Both hounds use specialized companion pursuit and attack authority, and owner switch, death, reset, disconnect, or individual 30-second expiry cancels only the appropriate schedules and removes their modifier authority.
- **Resolved - Plasma Sentinel pet attack stall:** the 0.6.17 trace created only one companion object and repeatedly logged `damageSelect: invalid damage range`. The active now creates both contract-required Pain Hounds, and companion buffs floor the minimum and ceil the maximum after scaling so every pet retains a valid authored-style integer damage roll.
- **Resolved - TC Shielded Sentinel Active:** `TCShieldedSentinelActive` is chunk 98 with pointblank template chunk 324 and taunt chunk 103. It pays at 0.5 seconds, releases at 0.72, and deals 12-20 Technology/Energy AoE damage with coefficient 0.05 to valid hostiles inside `8 * (1 + AoERadius)`. Only an accepted damage result receives `Modifier_TCShieldedSentinelActive_Taunt`, which uses the shared taunt policy with aggro 1000 and duration 5/3 seconds. Cooldown is 10 and mana is 16 with coefficient 0.08; the cast is explicitly untargeted and centered on the caster.
- **Implemented - accepted-hit taunts:** Charged Fist and TC Shielded Sentinel Active now share retained zone taunt authority. Accepted surviving hits force their NPC recipients onto the caster for the recovered six- and five-second rank-one durations, restart the NPC action against that caster, preserve co-op replacement when a hero dies, and re-enter ordinary nearest-hostile acquisition at expiry. Refresh, modifier presentation, reset, and teardown use one target-scoped lifecycle.
- **Resolved - Quantum State:** `QuantumState` is self-contained chunk 1013. It pays at 0.2 seconds, applies its unique eight-second buff to the caster, and releases at 0.5; cooldown is 20 and mana is 16 with coefficient 0.08. The buff adds no dodge or incorporeal attribute—the translation callbacks explicitly report both as zero—and instead watches `TookDamage` and `DealtDamage`. Outside a 0.1-second lockout and while no effect is being applied, either event stores its GUID-slot-one object as the pending target; the scheduler validates the most recently stored hostile, uniformly chooses one of five reactions, applies it, clears the pending target, and yields. The reactions are a 25-percent slow for 3/1 seconds, immobilizing stun for 1/0.5 seconds, three-unit knockback at speed 12, one-second banish, or a caster-centered radius-`3 * (1 + AoERadius)` burst for 4-8 Spacetime/Energy AoE damage with coefficient 0.06. The stun and banish use their respective immunity checks outside chain games; banish adds Intangible, Immobilized, Immune, and RejectModifiers. The AoE helper ignores the triggering enemy and centers on the buff owner.
- **Implemented - Quantum State:** every Maldri variant now casts the exact eight-second owner modifier with authored power, cooldown, hit, release, animation, and shader presentation. The localized client name is Probability Assault, and its packaged activation/deactivation pair calls `AddEffect`/`RemoveEffect` for `spacetime_quantumform_shader_effect.ServerEventDef`; server presentation therefore owns the shader as an attachment and hard-stops it on expiry, replacement, or hero switch. Accepted damage dealt or taken reserves one hostile reaction outside the 0.1-second lockout and uniformly selects the recovered slow, stun, three-unit navigation-safe knockback, owner-centered area burst excluding the triggering hostile, or banish path. Each control reaction owns an exact server status expiry and matching modifier create/delete presentation; the area branch publishes damage, critical, death, loot, encounter, and player-stat transitions through shared zone authority.
- **Implemented - Quantum State rank control:** even ranks now shorten the random slow from three seconds to one second and the random stun from one second to 0.5 seconds. Odd ranks retain the base lifetimes, while knockback, banish, and area-burst branches remain rank-invariant as authored.
- **Resolved - Sleeping Cloud:** `SleepingCloud` is chunk 398; chunk 591 only uses its name as a Sporogenesis presentation owner. The channel pays at 0.1 seconds, snapshots the caster's then-current position, and creates a stationary radius-7 trigger there; cooldown is 12 and mana is 24 with coefficient 0.12. Movement is authored to resume at 1 second, while the cast/trigger remains until the six-second animation boundary or earlier deactivation. Each non-caster opposing-team entrant receives one zero-duration `SleepingCloud_SleepModifier` and is tracked by object ID; the entry callback performs no hostile-target validation. The modifier is blocked by `ImmuneToSleep` outside chain games, immobilizes indefinitely, and handles damage so non-DoT `TookDamage` returns false while DoT damage returns true. Deactivation destroys the trigger and deletes every tracked sleep modifier. Preserve awareness of an authored bug in the ordinary exit callback: it tests the private `idModifiers` map but then indexes global `table.idModifiers` when calling `MarkForDelete`, so exit cleanup cannot use the stored modifier ID as written.
- **Implemented - Sleeping Cloud damage wake:** accepted non-DoT hero damage now immediately clears the struck NPC's retained sleep state and modifier presentation, while damage-over-time ticks leave that target asleep. The existing aura continues to admit entrants, clear targets that leave, and clean every retained modifier at the six-second boundary.
- **Implemented - Sleeping Cloud action suspension:** a sleeping NPC can no longer complete a retained ordinary attack, advance pursuit, commit a push/pull payload, continue corpse-consumption movement, commit direct special damage, heal, repair, resurrect, charge up, apply an energy buff, or complete Citadel repair. Ordinary delayed attack and pursuit paths resume through their existing scheduler after sleep clears. Every delayed support commit reschedules itself for the greatest remaining sleep or stun lifetime and then invokes its original continuation, preventing disabled execution without discarding the action and stranding the NPC loop. Retained projectiles and status damage already in flight remain independent, preserving authored projectile travel and damage-over-time wake behavior.
- **Resolved - Time Ravager Support:** `TimeRavagerSupport` is chunk 419 with modifier chunk 962. It declares range 35, cooldown 14, mana 16 with coefficient 0.08, but its custom Lua never calls `PayCooldownAndMana`. It immediately sends `BreakRoot`, snapshots the raw target position, emits entrance presentation at the caster and an initial exit at that raw position, and waits 0.266667 seconds. A valid selected hostile uses `FindGoodMeleePosition` with fallback to the hostile's position; otherwise the cursor position must be reachable. An invalid candidate falls back through `FindBallRollPosition`, and failure aborts before teleport or payload. A valid result emits the second exit, teleports the caster, waits 0.1 seconds, and scans the unscaled authored radius 4. Valid hostile creatures take 8-14 Spacetime/Energy AoE damage with coefficient 0.05 and receive the modifier only after accepted damage; hostile projectiles receive the same modifier request without damage. The unique 3/1-second modifier is labeled `IsStun` but rejects `ImmuneToBanish` outside chain games, then adds Immobilized, Silence, and Frozen. The cast waits another 0.766667 seconds after the scan; deactivation is empty.
- **Implemented - Time Ravager Support split timing:** the navigation-approved destination now receives its second exit effect and authoritative hero teleport at 0.266667 seconds. The radius-four damage and retained stun/silence scan executes independently 0.1 seconds later at 0.366667 seconds, rather than combining movement and payload into one delayed commit. Release remains at the recovered 1.133334-second boundary.
- **Implemented - Time Ravager Support Frozen locomotion:** each accepted hostile receives an authoritative movement stop at its hit position when Frozen lands, preventing an already-issued pursuit from sliding the model through the one-second immobilization.
- **Resolved - Lightspeed Tempest Active:** `LightspeedTempestActive` is chunk 998 with `LightSpeedSlow` chunk 234. It targets the cursor at range 35, waits 0.1 seconds, and scans the unscaled authored radius 7 even though it separately calculates `7 * (1 + AoERadius)`; cooldown is 8 and mana is 18 with coefficient 0.09, but its custom Lua never calls `PayCooldownAndMana`. Every valid hostile takes 14-21 Spacetime/Energy AoE damage with coefficient 0.05, and only an accepted hit receives the five-second 40-percent slow. The server now enforces that movement slow and its modifier create/delete lifecycle on every accepted surviving hit. After that same accepted hit, every target modifier carrying `IsHaste` is transferred: for each original stack the caster receives a request with the original modifier GUID, rank, and initiator snapshot, then the source modifier instance is marked for deletion. Transfer does not preserve remaining duration, and its presentation fires once per target when at least one haste modifier, not stack, was stolen. Release is 0.55 seconds.
- **Implemented - Lightspeed Tempest Active haste theft:** the shared modifier inventory now classifies retained haste effects explicitly. Each accepted Lightspeed Tempest Active target contributes every retained haste stack to Orion with its original GUID, rank, initiator identity, complete authored lifetime, attack-speed, cooldown, and movement operands; the source modifier is then canceled, deleted from the NPC, and removed from inventory. Unknown future haste families automatically enter the same transfer path once their authoritative modifier record is tagged `IsHaste`. Indexed chunk 998 proves that a target contributing at least one modifier emits `spacetime_AOE_hasteMark_effect.ServerEventDef` exactly once at that target's current position after all matching modifier stacks have been copied and their source instances marked for deletion.
- **Implemented - Orion custom contract admission:** Temporal Siphon (`LightspeedTempestActive`) now always selects its recovered cursor-area damage, slow, and haste-theft contract instead of trusting the generic projection of its custom callbacks. Orion Delta's Celestial Comet (`SpacetimeRandom3`, chunk 213) likewise selects a recovered cursor-area contract with cooldown 8, range 35, radius 4, a summon effect at 0.1 seconds, the authored 0.3-second `delayAfterEffect`, damage at 0.4 seconds, 0.8-second release, 12-20 Spacetime/Physical area damage with coefficient 0.05, mana 16 with coefficient 0.08, and its authored `Comets_Muzzle` and `spacetime_random04_summon_effect` presentation. The cursor summon and damage scan are independently scheduled so the comet visibly reaches its target before damage commits. This keeps both reported abilities on production runtimes with complete visible payloads even when the constrained Lua projection accepts only their custom registration shell.
- **Resolved - Beast Charge:** `BeastCharge` is chunk 78 with charge template chunk 574 and knock-up chunk 692. The target-owned charge has range 50, cooldown 10, mana 18 with coefficient 0.09, speed multiplier 4, and the shared template pays before movement. Its radius-4 pass-through trigger and terminal-target callback both call a per-activation deduplicated hero hit: each valid hostile is marked before damage, takes 18-28 Life/Physical damage with coefficient 0.05, and only accepted damage receives `BeastChargeKnockup`; the knock-up uses the shared knockback template with speed 0.75, desired distance 0.2, and a one-second outro. While invoking a pass-through hit, the caster temporarily receives `RejectModifiers = 1`. If `BeastPetTracker` has a living pet, activation computes a nearby charge destination around the same selected target, fully removes that pet's `BeastPetCharge` cooldown, and requests the pet charge independently. The pet charge declares cooldown 4, range 125, zero mana, speed multiplier 3, a 0.3-second movement delay, and terminal damage 8-12; however, it inherits the hero's pass-through callback, so pass-through contacts use the hero's 18-28 damage/knock-up contract while its inherited terminal callback still uses the pet's own 8-12 payload.
- **Implemented - Beast companion charge:** a living tracked Beast Sentinel pet now interrupts its ordinary melee or pursuit when its owner activates Beast Charge, independently charges the selected hostile with the recovered 0.3-second movement delay, range 125, speed multiplier 3, and footprint-aware stopping distance, applies the hero's separate 18-28 Life/Physical damage and one-second knock-up once to every hostile crossed within radius four, then deals its Pet Damage-projected 8-12 terminal Life/Physical hit to the selected target. Arrival restores ordinary companion speed and position authority; cancellation restores its pre-charge position, and modifier cleanup retains the complete knock-up lifetime.
- **Resolved - Entangling Rush:** `EntanglingRush` is chunk 388 with charge template chunk 574 and root chunk 461. The target-owned charge has range 25, cooldown 8, mana 15 with coefficient 0.075, speed multiplier 1.6, and stops two extra units from its target; the shared template pays before movement. It has no pass-through callback. The inherited terminal callback deals 20-30 Life/Physical damage at rank one or 16-24 at rank two, coefficient 0.05, to the selected hostile. After the charge finishes, independently of that damage result, it publishes the authoritative endpoint and an explicit locomotion stop before scanning the unscaled authored radius 6 around the reported target position and requesting `EntanglingRushModifier` on every valid hostile creature. The shared root lasts 6/4 seconds and uses normal root-immunity admission. Its ending animation lasts one second and prediction is explicitly disabled.
- **Implemented - Entangling Rush cleanup:** movement or another interruptible command after impact no longer cancels the scheduled modifier lifecycle. The completed charge detaches from action admission while its generation-owned cleanup remains active, then clears every server root and publishes every modifier deletion at the authored 6/4-second boundary so the root trap cannot remain indefinitely.
- **Implemented - Summon Sprite locomotion:** Healing Spirit creation now publishes the same movement-speed attributes used by other companions before its one-second owner-follow loop begins. The sprite retains its 30-second lifetime and healing cadence while the client can execute each server follow order instead of leaving the flying model at its spawn point.
- **Resolved - Phantom Charge:** `PhantomCharge` is chunk 964 with charge template chunk 574 and silence chunk 297. The sliding target-owned charge has range 35, cooldown 8, mana 14 with coefficient 0.07, speed multiplier 8, and pays after its 0.1-second pre-move delay. Its radius-4 pass-through callback keeps a private object-ID set; on each first valid contact it temporarily gives the caster `RejectModifiers = 1`, deals 9-13 Supernatural/Energy AoE damage with coefficient 0.05 without checking acceptance, unconditionally requests `PhantomSilence`, emits the hit, and records the target. Silence lasts 2/1 seconds and is blocked by `ImmuneToSilence` outside chain games. The inherited terminal-target callback remains enabled and independently performs the template's standard 9-13 damage against the selected target; it does not consult the pass-through set and does not add silence, so the two authored paths must not be collapsed. Deactivation removes the caster's charge effect.

No active hero-kit blocker now requires additional reverse engineering. Future
sweeps should take an entire implementation family at once; named one-off tests
remain deferred until all families have been attempted.

The targeted-area decoder now treats damage, healing, effect-object, muzzle,
and root-modifier operands as independently optional while requiring at least
one real payload. This supports damage-only, healing-only, and modifier-only
registrations without fabricating zero-valued fields. The final compiler audit
will determine which deferred assets advance through that shared shape.

Active registrations may also omit `timetohit`, `timeToHit`, or
`timetorelease`; the compiler preserves those authored omissions as immediate
boundaries. This avoids rejecting instant modifier and utility casts merely
because they do not repeat the projectile template's timing fields.

The shared decoder also preserves explicit cone, modifier, modifier-area,
projectile-barrage, and melee-without-hit-event shapes instead of flattening
them into projectiles. `LFPoisonRavager_Expunge` has an independently recovered
melee fallback and now executes both its authored base strike and its separate
retained damage-over-time consumption without fabricating a hit effect.

Projectile-barrage definitions retain shot count, firing cadence, cadence
randomness, and authored hit chance in addition to projectile geometry and
effects. Lightning Tempest Active and Missile Tempest Active now execute as one
activation with independently owned projectile identities, launch/collision
deadlines, per-shot damage and critical resolution, targetless ground casts,
death/stat publication, release, rollback, and reset cancellation.

- **Implemented - Lightning Tempest Active barrage policy:** Lumin now waits the recovered randomized 0.265-0.365-second channel interval before every shot, snapshots the original cast origin for all selection, takes the independent 30-percent hit branch, selects a random hostile inside radius 18 that no earlier shot selected, and otherwise fires at a random miss point between radii 3 and 18. Every projectile launches vertically from ten units above its chosen endpoint. The caster releases independently at the recovered two-second boundary while already-launched and later channel projectiles retain their own collision and cleanup lifetimes.
- **Implemented - Missile Tempest Active bombardment geometry:** SRS-42 now preserves the selected cursor as the barrage center, independently scatters every shot across its radius-six reticle, launches the damaging rocket downward from 15 units above that shot's ground endpoint, resolves the nearest hostile inside the impact radius, and retains the authored projectile trail and impact presentation. This restores the visible overhead bombardment instead of routing all fifteen shots horizontally from SRS-42 into one cursor-resolved target.

Self modifiers, summon buffs, and channel areas are also distinct typed shapes.
Arboreal Might now executes through shared self-modifier admission with authored
power, cooldown, animation, activation effect, 15-second lifetime, modifier
create/update/delete packets, paired five-by-10-percent and two-by-20-percent
stack policies, and authoritative DamageBuff projection. Recasts below the cap
refresh the modifier lifetime; recasts at the cap preserve the old expiry.
Every hostile death strictly within 20 units now refreshes
the existing 15-second lifetime without adding a stack, matching the separate
authored `NearbyDeath` event.

Fire Tempest Active and Gravity Storm now use their specialized summon/enrage
and lift/slam runtimes rather than being flattened into self stat buffs.

Field Medic Active is retained as a modifier-area shape with its authored
Syndrome Shift ally/enemy effects and 30-second eligibility boundary. Its
zone-owned descriptor inventory transfers every currently authoritative
non-channel ally crowd-control family and enemy buff family across heroes,
companions, and NPCs. Newly transferred enemy debuffs retain the authored
30-second cleanup boundary, while transferred ally buffs retain their source
family lifetime. Modifier-area abilities are admitted by the shared active
dispatcher, so Syndrome Shift now reaches this runtime and commits its projected
power cost, cooldown, effects, transfer, release, and rollback lifecycle.

Cone abilities now share active admission, facing, animation, power, cooldown,
range-and-angle target selection, distance-scaled damage, critical resolution,
and death transitions. `BinarySentinelActive` executes its authored cone and
accepted-hit attraction across all four owning hero variants through shared
navigation-safe forced movement.

Teleport strikes now resolve the deployed hero kit's authored special-two
definition and content-derived cooldown key. Ride the Lightning remains the
only compiled simple teleport strike, while Quantum Blink, Beast Charge,
Entangling Rush, Phantom Charge, and Roar of Derision use their specialized
target-owned movement, collision, and save-and-return runtimes.

### Shared alias sweep

The exact build-103 instant-cast template (chunk `50`, SHA-256
`daf011a759c793297ba89f4eab1bf0b7a7ad140bf16ea0cb9065b8d850aef664`)
is now linked narrowly while compiling `CastEnrage`, `RootingPlague`,
`Sporogenesis`, and `Terrify`. Its indexed dependencies are merged with
conflict checks. This removes the common missing-template boundary without
pretending that each ability's modifier body or runtime policy is complete;
remaining failures stay in the deferred ledger after the final sweep audit.

## Next implementation order

Lumin's Webbed Lightning now uses each recovered arc endpoint for both launch
presentation and collision instead of launching all twelve projectile objects
at the actor and only correcting their impact positions. Thunderstorm now
projects its recovered 30-shot randomized cadence before the shared burst
definition guard, making its specialized retarget-channel runtime reachable.
Equal-time Webbed Lightning arcs are prepared as one simulator deadline before
that deadline advances, preventing its first arc from invalidating the other
eleven. Arc endpoints and targetless Thunderstorm strikes remain positional
rather than binding their projectile presentation to the actor role.
Thunderstorm resets Lumin's cast animation at the authored two-second release
while its independently owned ground strikes continue through their recovered
cadence.
The 0.6.12 capture proves all 30 `lightning_tempest_active.ServerEventDef`
hashes reached the client in a fields-6/10/11 envelope even though none
rendered. Thunderstorm impacts now use the fields-6/10 world-event envelope
used by native cursor effects, and each collision emits that impact once.
`SoulRavagerBasic` chunk 225 retains its
`shadow_soul_power_up_shot.ServerEventDef` trail and additionally attaches the
authored `_1`, `_2`, or `_3` projectile effect selected by the current soul
count (`0-1`, `2-3`, or `4-5`).
Webbed Lightning holds `el_random03_lightning_charge` for its recovered
one-second channel, transitions to `el_random03_lightning_cast` when the arcs
launch, and resets the cast state at the 1.8-second release. Arc collision uses
the authored 40-unit web reach rather than reapplying the 15-unit cast-admission
radius, so a valid outer arc no longer aborts cleanup for the whole burst.

1. Finish exact retained-effect presentation and cleanup where shared gameplay
   authority is already implemented.
2. Extend remaining self-only projections to evidenced co-op heroes and
   companions without coupling kit logic to one peer session.
3. Keep Webbed Lightning on its proven three/one-second base shock duration;
   shipped Ride the Lightning content never creates the orphaned stack type.

### 2026-08-27 basic cadence corrections

Projectile basics now consume the same authored animation sequence as melee,
area, toss, and cloud-lob basics. This removes the server's former behavior of
replaying the first animation and timing on every ranged shot while Orion's
client advanced through `lightspeedtempest_basic_a`, `_b`, and `_c`; the third
animation now has one matching authoritative projectile presentation. Held
repeat scheduling uses the selected sequence cooldown rather than a separate
projectile-only estimate.

Rejected sequence admission now restores the pre-request snapshot for area,
melee, projectile, toss, and cloud-lob basics. A cooldown rejection can no
longer retain a newer held generation and invalidate the already scheduled
repeat, which was the shared source of attack cadence stopping, accelerating,
or appearing to overlap after campaign mission transitions.

Titan's point-blank spread no longer requires a selected enemy or an enemy that
has already acquired Titan. It admits the targetless 360-degree cast and scans
every living hostile inside the projected authored radius at impact time.

### 2026-08-27 Krel, Tork, and Virulent Vines follow-up

The Krel 0.6.12 capture showed one `FireRavagerBasic` projectile per accepted
activation. Canonical chunk 88 (`Abilities/0x4921A499.lua`, SHA-256
`0fec260366671974062821489b98095c387906fd4bd25e9d0ae0feafdd7d06c5`)
authors `projectilesPerLaunch = 3`, spread angles `{-15, 15, 0}`, muzzle X
offsets `{0.4, -0.4, 0}`, Y offsets `{-1, -1, -1}`, and Z offsets
`{1, 1, 1.2}`. The basic catalog now retains those parallel arrays and the
projectile runtime rotates each trajectory independently instead of collapsing
the launch to one center shot.

Canonical Sprout chunk 982 (`Abilities/0x009B7C57.lua`, SHA-256
`1bc7aaed152ba52dd17dd8e07e94f338566c2470e4c921a0b3d1bc718972139e`)
defines left and right projectile offsets alongside a two-animation basic
sequence. The cloud-lob runtime formerly created both projectile objects and
both cloud objects for every accepted animation. It now uses the selected
animation index to launch one corresponding muzzle and reserves one projectile
and one cloud, allowing successive basics to alternate sides through the
existing sequence authority. Each created projectile now also carries the
ballistic initial velocity derived from that lob's destination, height, and
flight time; the reliable lob-locomotion packet remains the trajectory contract,
while the creation velocity prevents the client presentation from remaining at
the muzzle until detonation.

The Virulent Vines capture recorded `RootingPlague` rejected with target zero
before `Virulent_Vines_muzzle.ServerEventDef` could be published. Canonical
chunk 546 (`Abilities/0x52DE0C2E.lua`, SHA-256
`c6bf60424f9a2fac8605b382716b35331da20752e20713b575d32109eccbe8a0`)
retains the targeted interface. Targetless wire requests now resolve the
nearest hostile at the reported cursor before ordinary range and faction
admission, preserving server authority while restoring the cast presentation.

### 2026-08-27 0.6.13 combat follow-up

The Celestial Comet report showed one scheduled summon followed by one copied
summon effect for every area result. Cursor-area presentation now remains owned
by the single cursor impact, so a cast creates one descending comet regardless
of how many enemies occupy its damage radius.

Webbed Lightning now retains the object selected for each of its twelve
authored angular sectors when that arc launches and resolves the same object at
impact. Accepted, surviving hits are collected once and receive the recovered
Shock stun after the final equal-time impact; empty sectors remain positional
misses. This replaces the former impact-time rescan that discarded the launch
target and commonly turned visible projectiles into misses.

The 0.6.15 Webbed Lightning report predates this retained-target implementation:
the current runtime already transitions from the one-second charge to the
authored cast, launches and collides along the same twelve arc endpoints, resets
the cast presentation, and applies the content-backed Shock stun to every
accepted survivor. The paired Char report did expose fallback noun geometry
aiming above the authoritative collision body; targeted ranged basics now
replace only that synthetic client point with the server-derived body aim while
preserving ordinary finite player-directed trajectories.

Ranged basics now preserve a finite target position reported by the client for
their authoritative trajectory. The selected object remains the collision
candidate, but selecting one NPC no longer bends a shot aimed elsewhere onto
that NPC's current server position.

### 2026-08-27 Sage Dendrone respawn identity

The 0.6.13 2-1 crash capture ended in client access violation `0xc0000005`
after Sage's Dendrone object `2289` had been deleted and recreated under that
same world-object ID three times. The final respawn was followed by a valid
movement resolve and then the client terminated; the nearby crystal message
resolved to the known `crystal_wild_aoedamage.Noun` asset and used its correct
AoE Damage type, leaving repeated destroyed-object reuse as the captured
lifecycle violation.

Each Dendrone death now retains its stable passive role but reserves a fresh
zone object ID for the replacement. The passive outbox rebinds that role before
dispatching the spawn, so later death, separation, attack, and follow behavior
tracks the replacement while the client never receives a create for an object
identity it has already destroyed.

### 2026-08-29 Magnos ability mechanics

The Magnos captures confirm his authored kit is bound as
`BinarySentinelSupport` (Kinetic Wave), `BinarySentinelActive` (Gravity Well),
and `SpacetimeRandom1` (Shooting Star). Kinetic Wave now turns each accepted
non-boss hit into authoritative navigation-projected knockback after applying
its area damage. Gravity Well retains its recovered pull-to-hero movement and
boss-specific pull modifier for accepted hits.

Shooting Star's Lua defines initial speed `0.1`, acceleration `40`, and one
additional damage per speed unit. Projectile scheduling and live snapshots now
integrate that acceleration, use the resulting arrival time, and add the
impact-speed damage. This replaces the constant-speed interpretation that
could delay a short Shooting Star flight by several minutes.

The 0.7.2 Kinetic Wave and Gravity Well captures showed both cone actions being
accepted with valid targets, then failing at their scheduled hit with `cone
direction unavailable`. Admission had replaced the reported aim point with the
cone plan's center, which is the caster position, so the live target refresh
constructed a zero-length direction and never reached damage, knockback, or
pull authority. Cone hit schedules now retain the admitted aim point while
point-blank and cursor-area actions continue retaining their authored centers.

The native `/taunt` command reaches build 103 through the same tail-less emote
action as `/dance`; the packet does not retain the typed command name. It is now
listed in server help without inventing a server-only discriminator or changing
the animation selected through that native action.

### 2026-08-29 Char, Savage, and Meditron report sweep

Canonical `FireTempestBasic` chunk 459 uses the selected hostile directly when
it is valid and otherwise scans a radius of 0.5 around `GetTargetPosition()`.
The area-basic adapter now centers an explicit Char target on that hostile's
authoritative position instead of preferring the unrelated cursor field. The
Elemental Guardian capture also proved that its authored `Fireball` has a
0.17-second shot delay and zero release delay; the companion-only projectile
normalizes release to the shot deadline because activation, release, and
cooldown publication are suppressed on that internal attack.

Content class health for `BeastSentinelPet`, `RangedElementalPet`, and other
summons is authored as an owner-health fraction (`0.25`, `0.25`, and similar),
not literal fractional hit points. Shared companion creation now resolves
values at or below one against the active hero's maximum health before applying
Pet Health Increase. Savage Ally remains immobilized for the recovered
1.4-second `BeastSentinelPetSpawn` release and only then restores movement and
begins target acquisition.

Wild Charge now stops outside the selected target using both actors' footprint
radii and publishes `react_knockup` for accepted targets alongside its retained
one-second knock-up modifier. Meditron's Reconstruct release now resets the
caster animation, while Field Medic Passive removes all owner-bound Sentry
Drones found in the companion inventory before creating the current drone; a
lost tracker or repeated hero switch can no longer accumulate visible copies.
