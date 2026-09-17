# Build 103 target-side damage mitigation

## Gameplay reference supplied 2026-09-06

[Tales of Lumin, Game - (2-4) - The Chaos Fields](https://www.youtube.com/watch?v=m8aCk69lprc&t=359s) provides a survivability reference around six minutes. The video description explicitly calls this beta footage. Browser inspection identified Strafing Drakon and Blasting Fiend targets, green health-pickup text, and a `RESIST!` notification above the player during the surrounding fight. The orange `Crit! -23` around 6:07 appears above the attacked enemy and must not be recorded as an incoming critical hit. The level indicator shows 7, but this observation does not establish the equipped squad rating, individual hero statistics, or incoming per-hit damage. These observations support comparing avoidance, healing supply, and damage together; they do not determine a mitigation percentage or justify a global damage multiplier. No balance change was made from this clip alone.

## Applied hero defenses, 2026-09-06

The user explicitly requested implementation of the missing defenses after the item audit below. `server/combat/defense.go` now owns the common calculation, with the hero snapshot projected by `server/gameplay/defense.go`. Campaign melee, area, retained status/DoT, and projectile impact damage use this path before shields, Soul Link distribution, HP mutation, and hit reactions. Arena applies the same equipment calculation and no longer forces one damage through a fully protected hero. This supersedes the earlier decision to leave equipment defenses inert; it does not establish the absent retail formula.

- Dodge and Resist use the target's current PhysicalDefense/EnergyDefense and the mission's authored rating conversion: `chance = clamp(rating / (100 * conversion[difficulty-1]), 0, 0.75)`. One roll prevents the entire direct hit, including an already-selected critical. Physical selects Dodge, Energy selects Resist. Already-attached periodic damage does not reroll avoidance, but receives all applicable reductions. The conversion choice, 75% ceiling, full resist outcome, and periodic exemption are explicit conservative server policies.
- Percentage equipment attributes are normalized fractions: general reduction (6), source-specific reduction (8/54), science resistance (43-47), and area resistance (27). Separate categories multiply their remaining-damage factors after existing named ability reductions and same-genetype vulnerability; each fraction is clamped to [0,1]. Generic/unknown science bypasses science resistance. Unprofiled NPC attacks inherit the same Physical source fallback as their attack-range calculation.
- Source-specific flat armor (102/103) subtracts once after percentage reductions and before shield absorption. Resulting damage is clamped at zero; neither critical state nor a minimum-damage floor bypasses protection. Existing critical chance/multipliers remain unchanged. Soul Link shares the already-reduced amount without applying the active hero's defenses a second time.
- Avoided attacks publish Dodge/Resist feedback without critical damage, HP loss, or secondary hit-status eligibility. Projectile collisions still retire their presentation. Complete percentage/armor protection emits zero-damage immunity feedback and never enters the positive-damage authority call. Ghost Form retains its existing effect and receives the feedback word too.
- Equipment/catalyst percentage defenses are read from the current target snapshot at impact, including remote co-op heroes. Catalyst Dexterity/Mind changes now update derived Dodge/Resist ratings (and Dexterity's Critical Rating), including removal; Quantum Positioning's existing doubled PhysicalDefense also scales its catalyst delta. An out-of-range passive aura now removes only its own reduction instead of bypassing unrelated self-buff protection.

Validation was limited to production source-path review, Go formatting/parser acceptance, and `git diff --check`. No builds, launches, or tests were performed, as requested. Real-client balance and presentation remain unverified. These changes cover the identified hero equipment damage defenses; companion inheritance and unrelated unimplemented ability/status behaviors are not evidence of retail parity.

## Item defense audit, 2026-09-06

Read-only inspection of all 10,129 English `localization_text` rows in the runtime content database supplies primary shipped descriptions: key `0x0a4e939d` defines Dodge Rating as the chance of dodging a Physical attack, and `0x0a4e93a2` defines Resist Rating as the chance of resisting an Energy attack. Resist Rating is therefore not described as a percentage reduction applied to every energy hit. Equipment templates `0x08d0c4d2` and `0x08d0c4d4` grant Dodge and Resist Rating. The percentage defenses are separate: Area Effect Resistance (`0x0a9f557b`, attribute 27), and the More Stats labels Bio/Cyber/Quantum/Necro/Plasma Damage Reduction % (`0x0a3c7d3d` through `0x0a3c7d40`, plus `0x0a3c7f83`; attributes 43-47 mapped by science). No received-critical-damage reduction stat was identified in the attribute enum or this localization search. That is a bounded negative finding, not proof that no special ability can protect against critical hits.

Before this implementation, the equipped-item projection retained these attributes and computes PhysicalDefense/EnergyDefense, but the inspected hero incoming melee and projectile paths did not roll ordinary rating-based Dodge/Resist or consume equipped attributes 27 and 43-47. `applyPassiveDamageReduction` consumes selected named ability reductions instead. Thus displayed equipment defenses promised protection that those damage paths did not supply. This is a plausible contributor to excessive incoming damage; it does not establish that every reported half-health hit originates here. The existing 2x critical multiplier was not found to be applied twice, and the provisional 1.5x enemy cap was removed rather than retained as an unexplained balance change.

There is a confirmed special use of Resist Rating: shipped shield description `0x0a02772c` says it increases shield absorption, matching `absorbTCShield` in `server/gameplay/passive.go`. Extra health also reduces a fixed hit's fraction of maximum health, without changing the damage number. Critical Rating and CriticalDamageIncrease are offensive stats, not incoming critical protection. Any implementation of the missing general defenses must state its server-policy assumptions for rating conversion, resist outcome, damage gates, stacking, and flat-reduction order; the client stub below still does not recover those formulas. No builds or tests were run for this audit.

## Result

Build 103 does **not** contain the retail target-side damage calculation. The
client contains an exact attacker-side preview pipeline, ending in
`sub_9E4F40`, and an exact attribute accumulator, but the Lua
`nGameObject.TakeDamage` binding hands the complete damage request to a one-byte
client stub at `0x00E224A0` (`mov al, 1; ret`). No client function between
`TakeDamage` and that stub reads `DamageReduction`,
`PhysicalDamageReduction`, `EnergyDamageReduction`, a science resistance,
`AoeResistance`, either flat decrease, `PhysicalDefense`, or `EnergyDefense`.

Consequently, the retail formulas, their ordering, their final damage clamp,
and their exact target-side descriptor gates cannot be recovered from
`Game.c`, `content.db`, or the packaged Lua. Implementing the conventional
formula suggested by the names would be invention. This note separates the
recoverable contracts from that retail-server boundary.

## Exact call boundary

| Stage | Address | Exact behavior | Confidence |
|---|---:|---|---|
| Ability/UI damage preview dispatcher | `sub_43A9D0` (`0x0043A9D0`) | Calls `sub_9E5B10` for translated min/max damage tokens. This is UI computation, not `TakeDamage`. | High |
| Preview damage wrapper | `sub_9E5B10` (`0x009E5B10`) | Applies the ability/base branch, attacker coefficient, `sub_9E4F40`, simulation scaling, attacker attribute 108, the attribute 19/22 conditional multiplier, then clamps the preview to at least zero. It receives no target attribute object. | High |
| Attacker multiplier | `sub_9E4F40` (`0x009E4F40`) | Exact ordered additive attacker multiplier already documented elsewhere. It consumes attributes 7 and 9 only for the basic-damage boost described below. | High |
| Descriptor test | `sub_9DDD00` (`0x009DDD00`) | Returns `(descriptor & mask) != 0`. | High |
| Lua binding | `sub_A0B670` (`0x00A0B670`) | Registered as `nGameObject.TakeDamage` by `sub_A0E0E0` (`0x00A0E0E0`). Decodes initiator snapshot, target, two-element damage range, damage type, damage source, descriptors, optional coefficient/position, and output slots. | High |
| Missing retail implementation call | call at `0x00A0B8F8` | Passes the decoded request and output pointers to `0x00E224A0`. | High |
| Client placeholder | `0x00E224A0` | Raw build-103 instructions are exactly `B0 01 C3`: return true. It reads no argument and writes no damage result. IDA's `GSemaphore::TryAcquireCommit` name is a false library identification for this stub. | High |

The raw `TakeDamage` instructions initialize the returned damage to `0.0` and
the auxiliary boolean to false before calling the stub. Thus the client is not
hiding a target-side calculation in an inlined branch: the calculation is
absent.

An exhaustive inventory of the 101 `sub_9E3490` attribute-access call sites in
`Game.c` finds no constant read of 6, 8, 27, 43-47, 54, 102, or 103.
Their appearances outside generic attribute accumulation are enum/state
producers and presentation paths, not a second target-damage consumer.

The similarly shaped `sub_A025C0` path is registered as
`nGameObject.HealDamage`; its call to `sub_9E5D10` is healing and must not be
used as damage evidence.

## Recoverable attribute semantics

The “target gate” column below deliberately distinguishes authored/request
metadata from a proven retail consumer. The descriptor values are exact:
`IsAoE=0x8`, `IsPhysicalDamage=0x40`, and `IsEnergyDamage=0x80`;
`nDamageTypes` are Technology `0`, Spacetime `1`, Life `2`, Elements `3`,
Supernatural `4`, and Generic `5`. They are defined by packaged chunk 659,
`lua/0x2E64AA9E.lua` (`server_data` resource 14227). Their association with the
named mitigation attribute is natural and implementation-relevant, but the
retail gate is not present in the client and therefore is not proved.

| Attribute | Proven client-side value contract | Candidate request metadata, not a proven target gate | Proven target damage formula/order | Confidence and boundary |
|---|---|---|---|---|
| `DamageReduction` (6) | Aggregated as a normalized fraction. `sub_9E2E70` hard-caps the final value at `1.0`; it does not hard-floor it at zero. Packaged scripts pass fractions directly, e.g. chunk 17 (`Modifiers/0xA06DFA85.lua`, resource 13537) supplies `0.6/0.7/0.8`. | No descriptor condition implied; no gate is present in the client. | Not present. | High for storage/cap; none for retail formula. |
| `PhysicalDamageReduction` (8) | Aggregated as a normalized fraction. `sub_9E2E70` hard-caps it at `1.0` without a hard zero floor. Chunk 40 (`Abilities/0x133AA438.lua`, resource 13561) passes ranked `physicalDamageReductionPercent` directly; an authored value is `0.333`. | Damage source `Physical=0` and/or descriptor `0x40` are available to retail `TakeDamage`, but which field gates this attribute is not recoverable. | Not present. | High for storage/cap; none for retail gate/formula. |
| `EnergyDamageReduction` (54) | Stored as a normalized fraction and passed directly by Lua. Chunk 188 (`Modifiers/0xB1906946.lua`, resource 13724) adds the same ranked fraction to attribute 54 while Thorn Bark is in overdrive. Unlike 6 and 8, `sub_9E2E70` has no explicit special-case `1.0` cap for 54. | Damage source `Energy=1` and descriptor `0x80` are available, but the retail choice is absent. | Not present. | High for producer/absence of the special cap; none for retail gate/formula. |
| `TechnologyTypeResistance` (43) | Fractional attribute. Chunk 570 (`Modifiers/0xD3CB96D9.lua`, resource 14132) registers the NPC affix with raw value `0.75`. | Damage type `0` is present in the request; retail consumption is absent. | Not present. | High for producer/type mapping; none for retail formula. |
| `SpacetimeTypeResistance` (44) | Same raw-fraction producer contract as 43. | Damage type `1`; retail consumption absent. | Not present. | High for producer/type mapping; none for retail formula. |
| `LifeTypeResistance` (45) | Same raw-fraction producer contract as 43. | Damage type `2`; retail consumption absent. | Not present. | High for producer/type mapping; none for retail formula. |
| `ElementsTypeResistance` (46) | Same raw-fraction producer contract as 43. | Damage type `3`; retail consumption absent. | Not present. | High for producer/type mapping; none for retail formula. |
| `SupernaturalTypeResistance` (47) | Same raw-fraction producer contract as 43. | Damage type `4`; retail consumption absent. | Not present. | High for producer/type mapping; none for retail formula. |
| `AoeResistance` (27) | Stored as a normalized fraction. Chunk 523 (`Modifiers/0xD9A56F91.lua`, resource 14082) adds ranked values `0.75/0.8/0.9` directly. | Descriptor `IsAoE=0x8` is present in the request; retail consumption is absent. | Not present. | High for producer/descriptor definition; none for retail formula. |
| `PhysicalDamageDecreaseFlat` (102) | Absolute damage units, not a percentage. Chunks 47 and 882 add the same computed armor to 102 and 103: `baseArmor * pow(1.035, majorDifficulty * 10 + minorDifficulty)`, with `baseArmor=2` for Sloth Passive and `4` for Armored NPC Affix. | Physical source/descriptor metadata is available; retail ordering and gate are absent. | Not present. | High for units/producer formula; none for retail ordering. |
| `EnergyDamageDecreaseFlat` (103) | Same absolute-unit contract and producer formula as 102. | Energy source/descriptor metadata is available; retail ordering and gate are absent. | Not present. | High for units/producer formula; none for retail ordering. |

The exact names and indices come from packaged Global Definitions, not from the
later recap implementation.

## Attribute accumulation, normalization, and clamping

These rules describe how build 103 obtains an attribute value. They do not
establish how the retail server applies that value to damage.

1. `sub_9E2680` (`0x009E2680`) folds runtime modifiers for one attribute in
   this order:
   - type 1 values are summed and added;
   - type 2 values are summed and applied as one multiplier, `x *= 1 + sum`;
   - type 4 contributes the greatest positive additive floor/bonus and is
     added;
   - type 5 contributes the greatest positive multiplicative bonus and is
     applied as `x *= 1 + max`;
   - type 3 contributes the smallest ceiling and is applied last with `min`.
2. `sub_9E2800` (`0x009E2800`) combines the base/stat-derived value and the
   stored additive field, applies class/global maxima when nonzero, applies
   special transforms for a small unrelated index set, and then calls
   `sub_9E2680`.
3. `sub_9E2E70` (`0x009E2E70`) adds the explicit hard maximum of `1.0` only for
   indices 6 and 8. Negative 6/8 values survive this special case.
4. A generic per-index maximum array, `off_1164F14`, can additionally cap any
   attribute whose configured entry is nonzero. It is loaded at startup from
   the 115-float property hashed as `-705749729`. The available typed
   `content.db` projection does not identify its authored name or expose
   trustworthy per-index values, so no additional cap is asserted here.
5. Lua producers use fractional storage directly. There is no `/100` in
   `AddAttributeModifier` calls for these resistance/reduction attributes.
   Display helpers convert the fraction to percentage points: for example,
   chunk 944 (`Modifiers/0x5ABD4553.lua`, resource 14523) multiplies the ranked
   physical/energy reduction fraction by `100` for percent text. Therefore
   `0.75` means 75%, not 0.75%.
6. Flat decreases 102/103 are never percent-normalized by the producer and are
   expressed in damage units.

## PhysicalDefense and EnergyDefense

`PhysicalDefense` (7) and `EnergyDefense` (9) are not consumed by any recovered
target-side damage path. They do participate in the **attacker-side basic
preview** at `sub_9E4F40`:

```text
if descriptors & IsBasic:
    multiplier = 1
        + DefenseBoostBasicDamage(16)
        * (PhysicalDefense(7) + EnergyDefense(9))
```

That term is evaluated before `DamageBuff` and the remaining ordered attacker
bonuses. This proves the ratings are not themselves general target armor in the
client formula. It does not prove that the retail server ignored them for every
purpose, but there is no retail target-side evidence allowing them to be added
to mitigation.

## Display/state versus authoritative damage

| Evidence | What it proves | What it does not prove |
|---|---|---|
| Global Definitions and `nAttribute.AddAttributeModifier` scripts | The attributes exist, their indices are stable, scripts author fractions or flat units, and modifiers can change the client/simulator attribute state. | Any target damage equation. |
| Profile/“More Stats” fields such as `AOERES` | The value is player-visible/state-bearing. | That the client applies it to damage. |
| `sub_9E5B10`/`sub_9E4F40` | Exact attacker-side displayed damage preview. | Target mitigation; no target attribute object is supplied. |
| `sub_A0B670` followed by `0x00E224A0` | Retail authority is outside this executable build; the client packages the complete request. | Formula, stacking order, clamps, or rejection policy inside the retail server. |

Accordingly, none of attributes 6, 8, 27, 43-47, 54, 102, or 103 can be called
“client-authoritative damage” from the available evidence. They are real
attribute state and some are displayed; their damage consumption was a retail
server responsibility.

## Remaining retail-server boundaries

An exact implementation still requires retail-server code, a server trace that
varies one target attribute at a time, or an equivalent build artifact to
resolve all of the following:

- whether universal, source-specific, science, and AoE reductions are additive
  into one fraction or multiplicative stages;
- whether source selection uses `nDamageSources`, ability descriptor bits, or
  both when metadata conflicts;
- whether Generic damage type `5` bypasses science resistance;
- where flat decreases occur relative to percentage stages;
- whether flat decreases are individually or jointly clamped and whether any
  stage has a minimum-damage floor;
- whether negative reductions represent vulnerability and how they are capped;
- the configured generic maxima for attributes without the explicit 6/8 cap,
  especially 27, 43-47, and 54;
- whether DoT, projectile, basic, critical, reflected, or environmental damage
  changes any mitigation stage;
- whether `PhysicalDefense` or `EnergyDefense` has a retail-only target
  conversion; and
- rounding mode and the point at which float damage becomes the returned or
  wire-visible integer.

Until one of those authoritative sources is recovered, the implementation-ready
decision is to preserve the attribute values and request metadata but leave the
target mitigation formula unimplemented as a retired-server boundary rather
than encode a guessed pipeline. Exhaustive build-103 inspection has completed
the available client-side research for this question.
