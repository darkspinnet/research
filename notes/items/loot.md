# Theoretical mob loot table

## Status and scope

This document is a **design hypothesis**, not a recovered build-103 loot table.
It proposes a usable campaign-wide loot policy from:

- the authored names and combat themes of the tutorial enemy families;
- the native drop categories recovered from build 103;
- beta footage showing equipment from Quadra-class enemies;
- build-103 localization entries that preserve some, but not all, beta item
  names.
- the complete level/director mob inventory in `notes/campaign/mobs.md`.
- the community [Loot & Loot Types](https://gamegame.fandom.com/wiki/Loot_%26_Loot_Types)
  summary, used only for public terminology and broad behavior.

The percentages below are balancing choices. They must not be cited as retail
probabilities. A future server trace or recovered authoritative table supersedes
them.

The inspected runtime cache contains `content.db` only; no sibling `meta.db` is
present in this Darkspinner installation. Consequently, all build-local claims
below come from `content.db`, its retained package payloads, decoded Lua, or the
installed DBPF packages.

## Campaign-footage observations

A player-authored reference sheet derived from campaign footage records the
following repeated pattern across levels 1-1 through 2-4. The sheet also starts
a level 3-1 entry, but supplies no enemy rows for it.

- Every populated combat-enemy row assigns separate 30% chances to DNA, health,
  and power. The recorded DNA range is 3-20, health is 10-67, and power is 7-25.
- The crystal chance is usually 4% for ordinary enemies and 15% for stronger
  enemies and bosses. This is not purely an enemy-family property: Lightning
  Juggernaut is recorded at 4% in 1-1 and 15% in 1-3, while Distracted Mongrel
  is 4% in 1-2 and 15% in 2-2.
- Bosses Mizod, Khoo, Illust, Roark the Blood Fury, Zunh, Contagion the Deadly,
  and Nashira are each assigned a 30% chance for an unspecified other item.
- Most rows assign 0% to epic equipment. The exceptions in the supplied sheet
  are Elite Dread Root at 100%, and Ragetusk and Elite Ragetusk at 30%.
- Fixture-like plants, vines, totems, regulators, and mutation agents have blank
  drop columns rather than explicit zeroes. Those blanks are unknown data and
  must not be interpreted as proven no-drop behavior.

The covered footage uses squad levels 13-17 and enemy levels 0-18. References:
[1-1](https://www.youtube.com/watch?v=wZK5bU45Tmw&list=PLC983810FF604D725&index=1),
[1-2](https://www.youtube.com/watch?v=emP3_a5sGRE&list=PLC983810FF604D725&index=2),
[1-3](https://www.youtube.com/watch?v=omSRnbj3s8I&list=PLC983810FF604D725&index=3),
[1-4](https://www.youtube.com/watch?v=QVwCtkIP8rg&list=PLC983810FF604D725&index=4),
[2-1](https://www.youtube.com/watch?v=-a8PJ0i5hT4&list=PLC983810FF604D725&index=5),
[2-2](https://www.youtube.com/watch?v=-a8PJ0i5hT4&list=PLC983810FF604D725&index=6),
[2-3](https://www.youtube.com/watch?v=-a8PJ0i5hT4&list=PLC983810FF604D725&index=7),
[2-4](https://www.youtube.com/watch?v=-a8PJ0i5hT4&list=PLC983810FF604D725&index=8),
and [3-1](https://www.youtube.com/watch?v=-a8PJ0i5hT4&list=PLC983810FF604D725&index=9).

Treat these figures as footage-derived annotations whose counting method and
sample size are unknown. They are stronger balancing evidence than thematic
guesswork, especially for the repeated three-way 30% pattern, but they are not
an authoritative retail table. The encounter-dependent crystal values also
argue against keying every probability only by enemy noun.

## Recovered constraints

The theoretical table preserves these content and native-code facts:

- drop selector `1` means no drop;
- bit `0x02` enables health, power, or resurrection orbs;
- bit `0x04` enables catalysts;
- bit `0x08` enables equipment loot;
- bit `0x10` enables DNA;
- `TutorialBasicPoison` has the orb-drop selector, while
  `TutorialBasicPoisonNoOrbs` explicitly has the no-drop selector;
- loot eligibility and XP eligibility are independent;
- resurrected enemies and spawned duplicates can be marked unable to drop loot;
- equipment is assembled from a rigblock, suffix, zero to two prefixes, item
  level, and rarity. A display name does not by itself identify the complete
  item roll;
- build-103 `ClassAttributes` directly joins `CryosSpecialOne` to
  `AssetStrings!0xE0650396`, and `localization_text` resolves that key to
  `Quadrakiller`. Quadra is therefore a display/footage name for a known build
  family, not merely a thematic guess.

No recovered Lua death behavior chooses a named item. Named equipment should be
selected only after the authoritative death owner accepts a drop.

## Community-wiki constraints

The wiki adds useful player-visible constraints, but no exact enemy drop
probabilities:

- The four in-level loot types are DNA, health/power capsules, catalysts, and
  items. DNA is described as common; capsules as less common.
- DNA, capsules, and catalysts may also come from destructible objects. Gear is
  limited to enemy Game and obelisks.
- Capsules heal all heroes in the player's and allies' squads by a base 10
  percent, subject to catalyst/item modifiers.
- Catalysts last until the end of a mission chain. Their functional colors are
  purple/general, red/offense, blue/defense, green/utility, plus prismatic
  wildcards. Obelisks have a relatively high catalyst chance.
- In-level drop gear uses Common/white, Uncommon/green, Rarified/blue, and
  Purified/purple. Cash-out yellow/orange/red gear is a separate end-of-chain
  reward and must not be emitted by an enemy death. The chain-result Purified
  band remains disabled until completing 5-1; earlier results redistribute that
  range to Rarified rewards.
- The stated item-level offsets are Common `level - 5`, Uncommon `level`,
  Rarified `level + 5`, and Purified `level + 10`.
- Weapons may drop only for heroes the player has activated.

These are community-documented behaviors, not substitutes for build-103
probability tables. They do, however, invalidate a mutually exclusive roll in
which rare DNA competes directly against the more common capsule result.

## Proposed drop procedure

For each accepted enemy death:

1. Reject loot when the creature is a no-drop variant, duplicate, resurrected
   copy, or otherwise marked unable to drop.
2. Roll the DNA, capsule, catalyst, and equipment chances independently from
   the enemy-family table below. A kill can therefore emit more than one loot
   type, subject to an encounter pickup budget.
3. If a capsule roll succeeds, use the native resource-weighting idea: favor
   health when the squad's average health fraction is lower and power when its
   average power fraction is lower.
4. If a catalyst roll succeeds, select a functional color from the mission
   state and catalyst pool; do not treat cash-out gear colors as catalyst rarity.
5. If the result is equipment, roll from the family's themed name pool, then
   apply normal item-level, rarity, class, science, and part-slot eligibility.
6. Give an Intro Quadra its guaranteed tutorial roll only once per mission.

Percentages in a row are independent and do not sum to 100. If no roll succeeds,
the accepted death creates no pickup.

## Enemy-family category table

| Enemy noun | DNA | Health capsule | Power capsule | Catalyst | Equipment | Rationale |
| --- | ---: | ---: | ---: | ---: | ---: | --- |
| `TutorialBasicPoisonNoOrbs` | 0% | 0% | 0% | 0% | 0% | Preserves the authored no-drop distinction despite sharing the visible Poison family. |
| `TutorialBasicPoison` | 45% | 20% | 12% | 5% | 5% | The recovered selector proves capsule eligibility. Its melee/poison identity makes health recovery slightly more likely than power recovery. |
| `TutorialBasicDiseased` | 48% | 18% | 6% | 8% | 10% | Poison Cloud suggests Bio/disease-themed equipment and a health-oriented recovery bias. |
| `TutorialBasicRanged` | 48% | 6% | 18% | 8% | 13% | Plasma Lightning supports a power-capsule bias and electric/plasma equipment theme. |
| `TutorialSloth` | 52% | 10% | 14% | 10% | 18% | TailZap and the wiki's Electric Sloth identity support an electric theme without guaranteeing gear. |
| `TutorialSpecialOne` | 65% | 5% | 10% | 15% | 50% | Quadra/Quadrakiller-class BurstShot and the beta footage support a substantially elevated equipment chance. |
| `TutorialSpecialOne_Intro` | 0% | 0% | 0% | 0% | 100% | Proposed one-time tutorial guarantee. This models the footage without claiming a recovered build-103 guarantee. |

For mixed horde waves, roll independently per accepted creature death. Do not
award one wave-wide attack or loot token. If this produces too many pickups,
apply a separate encounter budget rather than silently changing individual mob
identity.

## Proposed equipment pools

The named pools are thematic allowlists, not exclusive retail associations.
The generator may substitute another eligible rigblock with the same theme and
slot when a listed beta name is unavailable in the active content release.

| Enemy family | Relative weight | Candidate display name | Content status | Theme |
| --- | ---: | --- | --- | --- |
| Poison | 45 | `Thermic Shard` | Present in build 103; rigblock ID `381` | Compact biological/offensive part; use as a low-tier rare result rather than a common orb replacement. |
| Poison | 35 | Bio- or poison-eligible generic rigblock | Generated from the active build | Keeps ordinary Poison drops compatible with class, science, and slot restrictions. |
| Poison | 20 | `Thermic Ridge-Eye` | Present in build 103; rigblock ID `722` | Very rare sensory part shared with the disease/Quadra pools. |
| Diseased | 50 | Bio- or disease-eligible generic rigblock | Generated from the active build | Primary pool for Poison Cloud's biological theme. |
| Diseased | 30 | `Thermic Ridge-Eye` | Present in build 103; rigblock ID `722` | Diseased sensory/ocular theme. |
| Diseased | 20 | `Thermic Shard` | Present in build 103; rigblock ID `381` | Secondary offensive result. |
| Ranged | 35 | `Photonic Repulsor` | Beta-only name; no exact build-103 localization entry | Light/plasma ranged theme. |
| Ranged | 35 | `Pulsar Compensator` | Beta-only name; no exact build-103 localization entry | Ranged weapon stabilization theme. |
| Ranged | 30 | Electric- or plasma-eligible generic rigblock | Generated from the active build | Retail fallback when a beta identity cannot be resolved. |
| Sloth | 45 | Electric- or plasma-eligible generic rigblock | Generated from the active build | TailZap theme. |
| Sloth | 30 | `Pulsar Compensator` | Beta-only name | Heavy electrical/mechanical result. |
| Sloth | 25 | `Thermic Shard` | Present in build 103; rigblock ID `381` | Secondary charged offensive result. |
| Quadra | 25 | `Pulsar Compensator` | Beta footage; no exact build-103 localization entry | Burst weapon component. |
| Quadra | 20 | `Thermic Watcher` | Beta footage; no exact build-103 localization entry | Sensor/eye family. |
| Quadra | 20 | `Thermic Ridge-Eye` | Beta footage and build-103 localization; rigblock ID `722` | Strongest cross-build match. |
| Quadra | 15 | `Hyperion Oculus` | Beta footage; no exact build-103 localization entry | High-tier ocular part. Build 103 retains `Hyperion's` as an affix and `Thermic Oculus` as a different full name, but that does not prove equivalence. |
| Quadra | 10 | `Photonic Repulsor` | Beta-only name | Rare ranged-energy result. |
| Quadra | 10 | `Thermic Shard` | Beta footage and build-103 localization; rigblock ID `381` | Cross-build fallback with an exact surviving identity. |

Relative weights apply only after the independent equipment roll succeeds. Generic
entries should be resolved from the active rigblock catalog, not stored as
literal inventory names.

## Proposed rarity and affix policy

The percentages in this table are an optional beta-inspired design proposal.
They are not recovered build-103 rarity probabilities, and they were not
derived from the authored stat-growth bands or campaign difficulty curve. The
client-backed loot tuning controls how item level, rarity blocks, and selected
affixes contribute stats *after* rarity is known; it does not establish the
chance of selecting each rarity. The affix topology is therefore more strongly
supported than these selection weights, but the complete retail generation
policy remains unresolved. The 65/27/7/1 row is enabled as the conservative
campaign-wide fallback by the explicit attached-bug request; later recovered
source-specific tables supersede it.

| Enemy family | Common | Uncommon | Rarified | Purified | Affix policy |
| --- | ---: | ---: | ---: | ---: | --- |
| Basic Poison, Diseased, or Ranged | 65% | 27% | 7% | 1% | Common: none; Uncommon: suffix; Rarified: suffix plus one prefix; Purified: suffix plus two distinct prefixes. |
| Sloth | 50% | 34% | 13% | 3% | Same affix structure, with electric/plasma eligibility preferred. |
| Ordinary Quadra | 25% | 40% | 27% | 8% | Bias toward a complete named and affixed reward. |
| Intro Quadra guarantee | 0% | 55% | 35% | 10% | Never create an unaffixed basic item for the one-time tutorial reward. |

Rarity is rolled after the equipment roll succeeds. It must not increase the
chance of receiving equipment a second time.

## Obelisk tables

Obelisks use separate ability-owned selectors and should not inherit an enemy
family table.

| Source | Proposed output | Evidence boundary |
| --- | --- | --- |
| Loot obelisk (`InteractWithObelisk`) | One equipment roll plus one crystal roll, published after the authored one-second loot delay. For the opening tutorial fixture, retain Electro Claws unless a beta-mode profile explicitly requests `Photonic Repulsor` or `Thermic Shard`. | Lua proves the combined Loot and Crystal selector, but not a named equipment result. |
| Health obelisk (`InteractHealthObelisk`) | Two guaranteed orb selections plus two additional 50% selections, each independently weighted between health and power from squad need. Maximum output: four capsules. | Lua proves the Orb selector. The count is a theoretical tuning choice shaped to match beta horde-room footage; native code supports multiple budgeted selections. |

## Suggested implementation identities

Do not persist display strings. A theoretical runtime table should use stable
identities shaped like:

```text
mob family
  -> independent DNA/capsule/catalyst/equipment chances
  -> equipment pool entries
       -> rigblock asset or metadata ID
       -> required active-build availability
       -> class/science/slot predicates
       -> relative weight
  -> rarity weights
```

The two exact surviving candidate identities are:

| Display name | Build-103 rigblock ID |
| --- | ---: |
| `Thermic Shard` | `381` |
| `Thermic Ridge-Eye` | `722` |

The other beta names require an explicit compatibility alias or a newly
authored asset. Similar wording is not sufficient to equate two rigblocks.

## Validation targets

Before treating any part of this proposal as parity behavior, capture or
recover:

1. the authoritative enemy-death caller that supplies the drop selector;
2. the effective selector and drop amount for ordinary and Intro Quadra;
3. whether Quadra has a private rigblock pool or only uses the global filtered
   generator;
4. the exact difficulty, player-count, and encounter-class multipliers;
5. the beta asset identities behind the four names absent from build 103;
6. whether the horde-room obelisk count is fixed, budget-derived, or random;
7. inventory state before and after collecting each footage pickup.

Until those checks succeed, this table is appropriate for an optional
`theoretical` or `beta-inspired` ruleset, not the default build-103 parity
profile.

## Campaign-wide family extension

The tutorial table above remains the most specific proposal because footage
constrains it. Every additional mob recovered in `notes/campaign/mobs.md` can be handled
without inventing hundreds of unrelated per-noun percentages: first classify
the exact noun by rank and role, then select equipment by its authored science
or faction theme.

For player-facing reporting, use the locale-backed aliases in `notes/campaign/mobs.md`.
Those 24 special-family names are direct build-103 asset/localization joins;
the named captain titles remain position-based community-wiki aliases. Loot
logic should continue to key on the internal noun family so localization cannot
change gameplay behavior.

This is still a theoretical policy. The level binaries prove spawn eligibility;
they do not contain a named-item loot table.

### Rank and role category table

Choose rank with the precedence Boss, Captain, Special, then Basic; reject a
fixture first. Apply one role adjustment afterward. Clamp every independent
chance to the inclusive range 0-100 percent.

| Mob class | DNA | Health capsule | Power capsule | Catalyst | Equipment | Classification |
| --- | ---: | ---: | ---: | ---: | ---: | --- |
| Basic / shooter | 45% | 12% | 10% | 7% | 10% | Any ordinary `*Basic*` noun, plus `Shooter`, `Sloth`, or `Boomer`. |
| Special / lieutenant / operative | 55% | 10% | 12% | 12% | 30% | Any `*Special*` or `*Specific*` noun, the `nct_*` families, and named agents such as `MutationAgent`, `Rezzer`, or Nomad role mobs. |
| Captain | 65% | 8% | 10% | 18% | 55% | Any `_Captain` form, including tier-2 and tier-3 captains. |
| Boss / destructor | 100% | 10% | 10% | 25% | 75% | Explicit `*Boss` nouns only. Use an encounter lockout so a boss cannot be farmed by duplicate death events. |
| Test dummy / critter | 0% | 0% | 0% | 0% | 0% | `SackOfHitPoints`, `Critter`, player creatures, and other fixture-only actors. |

| Name token / role | Adjustment | Reasoning from the authored name |
| --- | --- | --- |
| `Melee`, `Dog`, `Charge`, `Charger`, `Munch`, `Plunge` | +6 health capsule, -4 power capsule | Close-contact attrition favors recovery. |
| `Ranged`, `Gunner`, `Shooter`, `Scope`, `Snipe`, `Maser`, `Homer` | -5 health capsule, +5 power capsule | Ranged/energy identities favor power recovery. |
| `Poison`, `Diseased`, `Ooze`, `Drainer`, `Leech`, `Mines`, `Sinkhole` | +3 health capsule, +2 equipment | Hazard identities receive a small themed-equipment bump. |
| `Repair`, `Healer`, `Shield`, `HealthDrain`, `Rezzer` | +4 health capsule, +4 power capsule | Support enemies become resource-control targets. |
| `Flyer`, `Flying`, `Packfly`, `Copter`, `Blink`, `Stealth` | +3 equipment | Mobility/avoidance modestly rewards the harder target. |
| `Suicide` or spawned duplicate | Terminal override: all chances 0% | Prevent chain-spawn and self-destruct farming unless a future trace proves otherwise. |

Suffixes do not change role. `_2` and `_3` instead raise the item-level floor by
one and two tuning steps. Captain and boss rank replace, rather than stack with,
the ordinary rank table.

### Faction and science equipment pools

After an equipment result, use the prefix/theme table below as an eligibility
filter on the active rigblock catalog. Names not tied to an exact build-103 ID
remain compatibility aliases, not fabricated database identities.

| Noun prefix or family | Primary equipment theme | Named candidates and fallback |
| --- | --- | --- |
| `Cryos*` | Plasma default; inspect role/content metadata for Bio exceptions such as poison | `Photonic Repulsor`, `Pulsar Compensator`, `Thermic Shard`; otherwise eligible plasma/electric or explicitly Bio rigblocks. |
| `Citadel*` | Cyber, armor, shields, guns, targeting | `Pulsar Compensator`, `Thermic Watcher`, `Hyperion Oculus`; otherwise cyber armor/weapon/sensor rigblocks. |
| `Noct*`, `Nocturna*`, `nct_*` | Necro, stealth, drain, ocular | `Thermic Watcher`, `Thermic Ridge-Eye`, `Hyperion Oculus`; otherwise necro/stealth/drain rigblocks. |
| `Scaldron*` | No prefix default: Scaldron intentionally contains all five Genesis types | Resolve the noun's actual Bio/Cyber/Necro/Plasma/Quantum metadata first; `Thermic Shard` and `Thermic Ridge-Eye` remain thematic candidates only. |
| `Verdanth*` | Bio default; inspect metadata for imported Quantum/Necro/Cyber families | `Thermic Shard`, `Thermic Ridge-Eye`; otherwise eligible bio/toxin/growth rigblocks. |
| `Zelem*` | Quantum default; inspect repair/mechanical exceptions for Cyber | `Photonic Repulsor`, `Pulsar Compensator`, `Hyperion Oculus`; otherwise quantum/energy/mobility rigblocks. |
| `Nomad*`, `MutationAgent`, `Rezzer` | Use the role token and actual Genesis metadata before a prefix default | Scope/Snipe/Drone prefer sensor or ranged parts; Bio/Mutation prefer bio parts; Spacetime prefers Quantum; Shielder/Rezzer prefer defense/support. |
| `Sloth`, `Boomer`, `Shooter` | UGC/general pool, biased by attack identity | Sloth: electric/plasma; Boomer: explosive/area; Shooter: ranged. Use the active generic catalog when no exact named asset survives. |
| `ShadowBoss` or another unmatched combat noun | Encounter-theme fallback | Resolve from the boss arena/science metadata when available; otherwise use the active generic boss pool rather than guessing from the display string. |

The zoo grouping uses developer labels: `chrono` maps to public Quantum and
`supernatural` to public Necro. The wiki also shows that planet prefixes are
defaults, not guarantees, and that Scaldron contains all five Genesis types.
Always classify the spawned noun from content metadata, not merely the current
planet or noun prefix.

### Proposed rarity by rank

| Rank | Common | Uncommon | Rarified | Purified |
| --- | ---: | ---: | ---: | ---: |
| Basic mob equipment result | 65% | 27% | 7% | 1% |
| Special mob equipment result | 45% | 35% | 16% | 4% |
| Captain equipment result | 15% | 38% | 35% | 12% |
| Boss equipment result | 0% | 25% | 50% | 25% |

Difficulty-tier suffixes affect item level, not these rarity percentages. That
keeps noun tier, item level, and rarity as separate knobs and avoids silently
triple-counting difficulty.

Apply the wiki-documented in-level item offsets after rarity selection:

| Rarity | Proposed item level |
| --- | --- |
| Common / white | Encounter level minus 5, clamped to level 1 |
| Uncommon / green | Encounter level |
| Rarified / blue | Encounter level plus 5 |
| Purified / purple | Encounter level plus 10 |

The encounter level is the build-103 item-level coordinate derived from the
active mission difficulty: decompose the one-based difficulty into four-mission
major/minor coordinates, calculate `10 * major + minor`, and add the native
six-level boundary bonus on each fourth mission. Thus 4-1 is difficulty 13 but
encounter level 41, producing the screenshot-observed level-36 Common drop.
Crogenitor level is a separate generator input used only for account progression
and equipment-slot availability; it must not raise or cap the base level of
equipment dropped by a mission. If no compatible rigblock covers that exact
item level, the server may use the nearest compatible rigblock as the item's
presentation base, but the generated item's level remains mission-derived.

Yellow, orange, and red cash-out gear belongs to the chain reward system, not
this mob table.

### Coverage rule

Every combat mob listed in `notes/campaign/mobs.md` is covered by the following lookup:

```text
exact noun
  -> reject fixture/no-drop/duplicate cases
  -> strip .Noun and tier suffix for family classification
  -> select rank (basic, special, captain, boss)
  -> apply one role-token adjustment
  -> on equipment: select faction/science pool
  -> enforce active-build class, slot, item-level, and rarity predicates
```

When several role tokens match, use the first row in the role table. Do not add
all adjustments together; names such as `NocturnaBasicRangedSilence` would
otherwise receive accidental compound bonuses.
