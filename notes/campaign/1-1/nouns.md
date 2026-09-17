# `zelems_1` director-pool nouns

## Scope and provenance

This note inventories noun facts referenced by all four configuration groups in
the build-103 `zelems_1` level director. It does not choose a pool, choose a
noun, or infer spawn or loot policy.

The authoritative normalized source is
`bin/darkspinner/darkspin/cache/content.db`. Its manifest is
`source_build=103`, `content_release=build-103-content`, recipe `19`, input
fingerprint
`b088d6767e4485d90ff108c4eb091dd729b5cc8604b99b459ccc94cd1223b3ec`.
`level.id=56` is `zelems_1`; `level_director_entry.id=830..853` supplies the
24 pool rows below. The linked noun, non-player-class, AI-definition, and phase
records were decoded from content package row `1`,
`AssetData_Binary.package` (SHA-256
`faf7b72a27b7f6b6f2c497c9aa5379bd2f20b1e8fe7b59b0d3798a2232774f2b`).
Package resource types are:

- noun `0x76A8F7D8`;
- non-player class `0xD117AFCA`;
- AI definition `0xEEEB0E31`;
- phase/gambit definition `0x30728CE7`.

Every resource is group `0`. `content_source_resource.id` equals the package
ordinal plus one for this first imported package. Focused raw extraction is
under `bin/game/logs/1-1-nouns`; decoded values below were read from the DBPF
members, not from the non-authoritative converted XML tree.

## Four director pools

`configuration_ordinal` is the normalized database grouping. The database
retains `config_kind="unknown"` and `spawn_kind="unknown"`; independent
build-103 reflection evidence in `notes/campaign/1-1/director.md` maps ordinals `0`-`3`
to `minion`, `special`, `agent`, and `captain`, respectively (the reflected
`boss` array is empty). All 24 rows author `is_horde_legal=1`.

| Config / reflected pool | Entry | DB ID | Noun | Inclusive difficulty |
| ---: | ---: | ---: | --- | ---: |
| 0 / `minion` | 0 | 830 | `ZelemBasicRanged.Noun` | 1-24 |
| 0 / `minion` | 1 | 831 | `ZelemBasicRanged_2.Noun` | 25-48 |
| 0 / `minion` | 2 | 832 | `ZelemBasicRanged_3.Noun` | 49-72 |
| 1 / `special` | 0 | 833 | `ZelemSpecialHaster_Captain.Noun` | 1-24 |
| 1 / `special` | 1 | 834 | `ZelemSpecialOne_Captain_2.Noun` | 25-48 |
| 1 / `special` | 2 | 835 | `ZelemSpecialTwo_Captain_3.Noun` | 49-72 |
| 2 / `agent` | 0 | 836 | `ZelemBasicRanged.Noun` | 1-24 |
| 2 / `agent` | 1 | 837 | `ZelemBasicHybrid.Noun` | 1-24 |
| 2 / `agent` | 2 | 838 | `ZelemBasicRepair.Noun` | 1-24 |
| 2 / `agent` | 3 | 839 | `ZelemBasicRanged_2.Noun` | 25-48 |
| 2 / `agent` | 4 | 840 | `ZelemBasicPackMelee_2.Noun` | 25-48 |
| 2 / `agent` | 5 | 841 | `CitadelSpecificThree_2.Noun` | 25-48 |
| 2 / `agent` | 6 | 842 | `ZelemBasicRanged_3.Noun` | 49-72 |
| 2 / `agent` | 7 | 843 | `ZelemBasicRepair_3.Noun` | 49-72 |
| 2 / `agent` | 8 | 844 | `Sloth_3.Noun` | 49-72 |
| 3 / `captain` | 0 | 845 | `ZelemSpecialHaster.noun` | 1-24 |
| 3 / `captain` | 1 | 846 | `NomadSnipe.Noun` | 1-24 |
| 3 / `captain` | 2 | 847 | `NomadWithDrone.Noun` | 1-24 |
| 3 / `captain` | 3 | 848 | `ZelemSpecialOne_2.Noun` | 25-48 |
| 3 / `captain` | 4 | 849 | `CitadelSpecialTwo_2.Noun` | 25-48 |
| 3 / `captain` | 5 | 850 | `Boomer_2.Noun` | 25-48 |
| 3 / `captain` | 6 | 851 | `ZelemSpecialTwo_3.noun` | 49-72 |
| 3 / `captain` | 7 | 852 | `NomadShielder_3.noun` | 49-72 |
| 3 / `captain` | 8 | 853 | `NocturnaSpecialHomer_3.noun` | 49-72 |

The casing shown is the exact level payload spelling. Asset lookup is
case-insensitive, so the lower-case `.noun` rows resolve to the same packaged
asset keys shown in the definition table.

## Noun and class definitions

All 21 noun records have geometry string `builtins!sphere`, physics property
`DefaultPhysics.prop`, and locomotion/profile string `npcCreature`. The proven
built-in sphere has base horizontal radius `0.5`; therefore footprint is
`graphicsScale * 0.5`. This is a source-derived geometry result, not a generic
assumption that all noun scales are radii.

The `Loot` column reports the exact packaged repeated `dropType` field. Each of
these classes has count zero; the following storage word is `0xFFFFFFFF`.
This proves that no selector is authored on these class records. It does **not**
prove no drops: native/default director loot policy is outside these noun
definitions.

| Noun asset | Noun resource ID / instance | NPC-class resource ID | AI-definition string | Scale | Footprint | Locomotion | Loot |
| --- | --- | ---: | --- | ---: | ---: | --- | --- |
| `ZelemBasicRanged.Noun` | 9353 / `0x8F291AF3` | 9354 | `ZelemBasicRanged.AIDefinition` | 1.30 | 0.650 | `npcCreature` | `dropType=[]` |
| `ZelemBasicRanged_2.Noun` | 12489 / `0xE39A54D0` | 12490 | `ZelemBasicRanged.AIDefinition` | 1.30 | 0.650 | `npcCreature` | `dropType=[]` |
| `ZelemBasicRanged_3.Noun` | 12495 / `0xE39A54D1` | 12496 | `ZelemBasicRanged.AIDefinition` | 1.30 | 0.650 | `npcCreature` | `dropType=[]` |
| `ZelemSpecialHaster_Captain.Noun` | 6980 / `0xFC016279` | 6979 | `ZelemSpecialHaster.AIDefinition` | 2.75 | 1.375 | `npcCreature` | `dropType=[]` |
| `ZelemSpecialOne_Captain_2.Noun` | 1027 / `0x06630659` | 1026 | `ZelemSpecialOne.AIDefinition` | 3.40 | 1.700 | `npcCreature` | `dropType=[]` |
| `ZelemSpecialTwo_Captain_3.Noun` | 8555 / `0xBF99639A` | 8554 | `ZelemSpecialTwo.AIDefinition` | 3.20 | 1.600 | `npcCreature` | `dropType=[]` |
| `ZelemBasicHybrid.Noun` | 11839 / `0xBDA683F8` | 11840 | `ZelemBasicHybrid.AIDefinition` | 1.25 | 0.625 | `npcCreature` | `dropType=[]` |
| `ZelemBasicRepair.Noun` | 3683 / `0xA63F868D` | 3682 | `ZelemBasicRepair.AIDefinition` | 1.10 | 0.550 | `npcCreature` | `dropType=[]` |
| `ZelemBasicPackMelee_2.Noun` | 1902 / `0x8294BE8C` | 1903 | `ZelemBasicPackMelee.AIDefinition` | 1.30 | 0.650 | `npcCreature` | `dropType=[]` |
| `CitadelSpecificThree_2.Noun` | 1723 / `0xE9437932` | 1722 | `CitadelSpecificThree.AIDefinition` | 1.35 | 0.675 | `npcCreature` | `dropType=[]` |
| `ZelemBasicRepair_3.Noun` | 13239 / `0x2246E54B` | 13238 | `ZelemBasicRepair.AIDefinition` | 1.10 | 0.550 | `npcCreature` | `dropType=[]` |
| `Sloth_3.Noun` | 966 / `0xC832CC69` | 967 | `Sloth.AIDefinition` | 1.30 | 0.650 | `npcCreature` | `dropType=[]` |
| `ZelemSpecialHaster.Noun` | 6404 / `0xA1FDCCD8` | 6405 | `ZelemSpecialHaster.AIDefinition` | 1.85 | 0.925 | `npcCreature` | `dropType=[]` |
| `NomadSnipe.Noun` | 791 / `0x1DDB0187` | 790 | `NomadSnipe.AIDefinition` | 1.85 | 0.925 | `npcCreature` | `dropType=[]` |
| `NomadWithDrone.Noun` | 1475 / `0xB366BC74` | 1476 | `NomadWithDrone.AIDefinition` | 1.90 | 0.950 | `npcCreature` | `dropType=[]` |
| `ZelemSpecialOne_2.Noun` | 3909 / `0x8ADE3664` | 3908 | `ZelemSpecialOne.AIDefinition` | 2.60 | 1.300 | `npcCreature` | `dropType=[]` |
| `CitadelSpecialTwo_2.Noun` | 11717 / `0x756F4805` | 11716 | `CitadelSpecialTwo.AIDefinition` | 1.45 | 0.725 | `npcCreature` | `dropType=[]` |
| `Boomer_2.Noun` | 8475 / `0x7C406FD2` | 8476 | `Boomer.AIDefinition` | 2.00 | 1.000 | `npcCreature` | `dropType=[]` |
| `ZelemSpecialTwo_3.Noun` | 12498 / `0x0C7AFB3B` | 12499 | `ZelemSpecialTwo.AIDefinition` | 2.40 | 1.200 | `npcCreature` | `dropType=[]` |
| `NomadShielder_3.Noun` | 8007 / `0xB2EAFB9E` | 8008 | `NomadShielder_2.AIDefinition` | 2.00 | 1.000 | `npcCreature` | `dropType=[]` |
| `NocturnaSpecialHomer_3.Noun` | 12100 / `0x051DECF7` | 12099 | `NocturnaSpecialHomer.AIDefinition` | 1.55 | 0.775 | `npcCreature` | `dropType=[]` |

`NomadShielder_3` really names `NomadShielder_2.AIDefinition`; it is not a
normalization typo. The phase evidence below is reported at the shared family
definition because tier/captain nouns explicitly link back to those definitions.

## AI types and packaged abilities

The AI-definition and phase IDs below are exact
`content_source_resource.id` values. "Ability strings" are only strings in the
phase gambit records occupying ability positions; condition/property strings
were excluded. This inventories available authored gambits but does not infer
which gambit the native director/brain will select.

| AI family | AI resource ID | Phase resource ID | Packaged ability strings |
| --- | ---: | ---: | --- |
| `ZelemBasicRanged.AIDefinition` | 9355 | 9351 | `ZelemBasicRanged_Blink` |
| `ZelemBasicHybrid.AIDefinition` | 11841 | 11837 | `ZelemBasicHybridProjectile`, `ZelemBasicHybridMelee` |
| `ZelemBasicRepair.AIDefinition` | 3681 | 3685 | `Repair`, `ArcWeldingMelee` |
| `ZelemBasicPackMelee.AIDefinition` | 7037 | 7033 | `ZelemBasicPackMeleeCower`, `ZelemBasicPackMeleeAttack` |
| `ZelemSpecialHaster.AIDefinition` | 6406 | 6402 | `CastZelemHasteBuff`, `ZelemHasterAttack` |
| `ZelemSpecialOne.AIDefinition` | 2828 | 2832 | `ZelemSP1_TeleportGun` |
| `ZelemSpecialTwo.AIDefinition` | 7901 | 7905 | `ZelemSpecialTwo_Push`, `ZelemSpecialTwo_Pull` |
| `CitadelSpecificThree.AIDefinition` | 2766 | 2762 | `CitadelGrenadeRoll` |
| `CitadelSpecialTwo.AIDefinition` | 10723 | 10719 | `CitadelSpecialTwo_HomingStun`, `CitadelSpecialTwo_Melee` |
| `NomadSnipe.AIDefinition` | 789 | 793 | `NomadSnipe_Slow`, `NomadSnipe_Melee` |
| `NomadWithDrone.AIDefinition` | 1477 | 1473 | `NomadWithDronePunch` |
| `NomadShielder_2.AIDefinition` | 7702 | 7706 | `NomadShielderPositionForAttack`, `NomadShielderShield`, `NomadShielderBash`, `NomadShielderGrenade` |
| `NocturnaSpecialHomer.AIDefinition` | 3643 | 3647 | `NocturnaSpecialHomer`, `NocturnaSpecialHomerMelee` |
| `Sloth.AIDefinition` | 5667 | 5671 | `TailZap` |
| `Boomer.AIDefinition` | 3463 | 3467 | `BoomerCharge`, `Smash` |

The resource-type identity, exact noun link, phase link, and ability strings
are recovered content facts. Activation range, cooldown, damage, targeting,
ability registration, gambit conditions, director budgeting, and loot
probabilities require separate ability/native analysis and are deliberately
not assigned here.
