# Level mob inventory

## Scope and confidence

This is a build-103 content inventory of mobs that a level can ask the game
director to spawn. It covers all 61 records in `content.db`, not only the
tutorial. It is not a transcript of one playthrough: the director can choose a
subset according to difficulty, visit state, encounter budget, and spawn-point
type.

Coverage is 24 campaign maps, 11 tutorial/Spectra/TNX/arena records, 18 mode or
non-combat variants, and 8 AI zoos: 61 of 61 level rows.

The evidence comes from three independent parts of the runtime content:

- `level` contains the decoded binary level records. Printable `*.Noun` strings
  at the end of those records are the authored director candidates.
- `level_marker_set` and `marker` prove where a level supports wanderer, spike,
  and horde spawns. Campaign maps generally place director spawn points rather
  than fixed creature nouns.
- the eight `test_AI_zoo*` maps place fixed creature nouns and provide a useful
  cross-check on family and science classification.

Raw decoded level records used for this pass are under
`bin/game/logs/level-research/`. The database remains authoritative; the files
are only inspection artifacts.

The display-name cross-reference uses `localization_text` from the same
`content.db`. Every key below is an `en-us` row in table `1882572231`. Decoded
`ClassAttributes` resources from `AssetData_Binary.package` contain both the
developer asset name and an `AssetStrings!0x...` reference, so this is a direct
asset-to-locale join rather than a spelling guess. Package inspection artifacts
are under `bin/game/logs/noun-locale-package/`.

### Reading the names

- `.Noun` versus `.noun` is authored casing, not a gameplay distinction.
- `_2` and `_3` are progressively tiered variants of the same family.
- `_Captain`, `_Captain_2`, and `_Captain_3` are named Captain variants; Captain and runtime Elite are distinct classifications.
- The tables below collapse those suffixes to keep the inventory readable. A
  listed family means that at least one exact form is embedded in that level.
- `Director pool` is stronger evidence than a planet-theme inference, but it
  does not prove that every candidate is legal at every difficulty.
- `Fixed` means a creature noun occurs directly in a marker set. The ordinary
  campaign maps instead contain director spawn-point nouns.

## Campaign level pools

Campaign positions are calculated from the 72-entry `chain_level` resource.
The chain makes three 24-level passes and intentionally reuses each of the 24
campaign maps three times.

| Level | Campaign positions | Collapsed director families |
| --- | --- | --- |
| `zelems_1` | 1-1, 11-1, 17-4 | `ZelemBasicRanged`, `ZelemBasicHybrid`, `ZelemBasicRepair`, `ZelemBasicPackMelee`, `ZelemSpecialHaster`, `ZelemSpecialOne`, `ZelemSpecialTwo`, `CitadelSpecificThree`, `CitadelSpecialTwo`, `NomadSnipe`, `NomadWithDrone`, `NomadShielder`, `NocturnaSpecialHomer`, `Sloth`, `Boomer` |
| `zelems_3` | 1-2, 7-4, 14-2 | `ZelemBasicMelee`, `ZelemBasicRangedHoming`, `ZelemBasicFlyingMelee`, `ZelemSpecialOne`, `ZelemSpecialTwo`, `ZelemSpecialThree`, `ZelemSpecialHaster`, `NomadSnipe`, `NomadSpecialThree`, `VerdanthBasicPlunge`, `VerdanthBasicHealer`, `VerdanthSpecialOne`, `VerdanthSpecialTwo`, `CryosBasicPoison`, `CitadelSpecificThree`, `NocturnaSpecialHomer` |
| `nocturna_4` | 1-3, 11-3, 14-3 | `NocturnaBasicHealthDrain`, `NocturnaBasicStealth`, `NoctBasicFlyer`, `NoctBasicHopper`, `NocturnaSpecialLeech`, `NocturnaSpecialMunch`, `nct_lieu_su_stealther`, `CitadelBasicMelee`, `CryosBasicLightningRanged`, `CryosSpecialTwo`, `CryosElementalSpecialThree`, `VerdanthBasicSkeet`, `VerdanthSpecialThree`, `NomadBioSpecialTwo`, `Rezzer`, `Boomer` |
| `nocturna_1` | 1-4, 10-1, 18-2 | `NocturnaBasicRangedSilence`, `NoctBasicHopper`, `NoctBasicMeleeDog`, `nct_lieu_su_stealther`, `NocturnaSpecialLeech`, `NocturnaSpecialMunch`, `VerdanthBasicDiseased`, `VerdanthBasicPlunge`, `VerdanthSpecialOne`, `CryosBasicPoison`, `CryosBasicLightningRanged`, `CryosSpecialThree`, `NomadBioSpecialTwo`, `ZelemSpecialThree`, `Boomer` |
| `verdanth_1` | 2-1, 12-1, 13-4 | `VerdanthBasicRanged`, `VerdanthBasicDiseased`, `VerdanthSpecialOne`, `VerdanthSpecialTwo`, `VerdanthSpecialThree`, `CryosBasicPoison`, `CryosSpecialThree`, `CitadelBasicShield`, `CitadelBasicGunner`, `CitadelSpecialFour`, `ZelemBasicRepair`, `ZelemBasicPackMelee`, `ZelemSpecialThree`, `Boomer` |
| `verdanth_3` | 2-2, 8-4, 16-1 | `VerdanthBasicRootmob`, `VerdanthBasicHealer`, `VerdanthBasicSkeet`, `VerdanthSpecialOne`, `VerdanthSpecialTwo`, `NomadSpecialOne`, `NomadScope`, `NoctBasicMeleeDog`, `NocturnaBasicStealth`, `NocturnaSpecialDrift`, `NocturnaSpecialLeech`, `nct_lieu_su_stealther`, `CryosSpecialThree`, `CitadelBasicMelee`, `Shooter` |
| `zelems_2` | 2-3, 7-3, 15-1 | `ZelemBasicChargeup`, `ZelemBasicFlyingMelee`, `ZelemBasicHybrid`, `ZelemSpecialOne`, `ZelemSpecialTwo`, `ZelemSpecialThree`, `NomadSnipe`, `NomadScope`, `CitadelSpecificThree`, `CitadelBasicGunner`, `CitadelBasicShield`, `CitadelSpecialThree`, `NoctBasicHopper`, `NocturnaSpecialDrift` |
| `zelems_4` | 2-4, 11-2, 16-3 | `ZelemBasicPackfly`, `ZelemBasicRangedHoming`, `ZelemSpecialOne`, `ZelemSpecialTwo`, `ZelemSpecialThree`, `ZelemSpecialHaster`, `VerdanthBasicMelee`, `NoctBasicGhostCharger`, `NoctBasicFlyer`, `NocturnaSpecialHomer`, `NocturnaSpecialDrift`, `nct_lieu_su_stealther`, `ScaldronBasicCopter`, `NomadBioSpecialTwo`, `Shooter` |
| `cryos_4` | 3-1, 9-3, 18-1 | `CryosBasicFiery`, `CryosBasicRanged`, `CryosElementalSpecialThree`, `CryosSpecialOne`, `NomadRuption`, `NomadWithDrone`, `NomadShielder`, `NoctBasicHopper`, `NocturnaSpecialHomer`, `CitadelBasicMelee`, `CitadelBasicRanged`, `CitadelBasicGunner`, `CitadelSpecialFour`, `ZelemBasicRepair`, `VerdanthSpecialThree` |
| `cryos_3` | 3-2, 9-4, 16-2 | `CryosBasicMelee`, `CryosBasicLightningRanged`, `CryosBasicRanged`, `CryosSpecialTwo`, `CryosElementalSpecialThree`, `NomadRuption`, `NomadDrag`, `NomadSpecialThree`, `NomadSnipe`, `ZelemBasicPackMelee`, `ZelemBasicFlyingMelee`, `ZelemSpecialTwo`, `VerdanthBasicMelee`, `VerdanthBasicDiseased`, `NocturnaSpecialHomer` |
| `verdanth_2` | 3-3, 8-3, 17-3 | `VerdanthBasicPicky`, `VerdanthBasicSkeet`, `VerdanthBasicPlunge`, `VerdanthBasicMelee`, `VerdanthSpecialOne`, `VerdanthSpecialTwo`, `VerdanthSpecialThree`, `CryosBasicCharge`, `CryosSpecialThree`, `ZelemBasicHybrid`, `ZelemSpecialOne`, `NoctBasicGhostCharger`, `NocturnaSpecialDrift`, `NomadWithDrone`, `NomadSpecialOne`, `Boomer` |
| `verdanth_4` | 3-4, 12-2, 15-2 | `VerdanthBasicOoze`, `VerdanthBasicDiseased`, `VerdanthBasicHealer`, `VerdanthSpecialTwo`, `NoctBasicFlyer`, `nct_minn_su_drainer`, `NocturnaSpecialHomer`, `NocturnaSpecialMunch`, `NomadSpecialOne`, `NomadDrag`, `Rezzer`, `ZelemBasicPackfly`, `ZelemSpecialHaster` |
| `infinity_2` | 4-1, 10-4, 17-1 | `CitadelBasicSuicide`, `CitadelBasicGunner`, `CitadelSpecificThree`, `CitadelSpecialThree`, `CitadelSpecialFour`, `NomadShielder`, `NomadScope`, `NomadSpecialThree`, `CryosBasicCharge`, `CryosBasicRanged`, `CryosElementalSpecialThree`, `ScaldronBasicMonk`, `nct_minn_su_drainer`, `NocturnaSpecialDrift`, `Boomer` |
| `infinity_3` | 4-2, 8-2, 14-4 | `CitadelSpecificOne`, `CitadelBasicRanged`, `CitadelSpecialTwo`, `CitadelSpecialFour`, `NomadShielder`, `NomadBioSpecialTwo`, `NomadSpecialOne`, `VerdanthBasicSkeet`, `VerdanthBasicHealer`, `VerdanthBasicPlunge`, `VerdanthSpecialThree`, `ZelemBasicRepair`, `ZelemBasicPackMelee`, `CryosSpecialThree`, `Boomer` |
| `cryos_1` | 4-3, 7-1, 15-4 | `CryosBasicFireWave`, `CryosBasicCharge`, `CryosBasicLightningRanged`, `CryosBasicPoison`, `CryosSpecialOne`, `CryosSpecialTwo`, `CryosElementalSpecialThree`, `NoctBasicGhostCharger`, `NocturnaSpecialDrift`, `nct_lieu_su_stealther`, `NomadRuption`, `NomadBioSpecialTwo`, `NomadDrag`, `Rezzer`, `Shooter` |
| `cryos_2` | 4-4, 7-2, 13-3 | `CryosBasicLightningMelee`, `CryosBasicRanged`, `CryosBasicCharge`, `CryosSpecialOne`, `CryosSpecialTwo`, `CryosElementalSpecialThree`, `ZelemBasicFlyingMelee`, `ZelemBasicRangedHoming`, `ZelemSpecialHaster`, `CitadelBasicRanged`, `VerdanthSpecialThree`, `NomadRuption`, `NomadWithDrone`, `NomadSnipe`, `NomadSpecialThree` |
| `nocturna_3` | 5-1, 11-4, 13-1 | `NocturnaBasicStealth`, `NocturnaBasicHealthDrain`, `NoctBasicHopper`, `NocturnaSpecialMunch`, `NocturnaSpecialLeech`, `Rezzer`, `Shooter`, `VerdanthBasicPlunge`, `VerdanthBasicSkeet`, `VerdanthBasicHealer`, `VerdanthSpecialTwo`, `ZelemBasicRepair`, `NomadSpecialThree`, `NomadWithDrone`, `NomadBioSpecialTwo`, `CryosSpecialThree` |
| `nocturna_2` | 5-2, 10-2, 16-4 | `nct_minn_su_drainer`, `nct_lieu_su_stealther`, `NoctBasicGhostCharger`, `NoctBasicFlyer`, `NocturnaSpecialLeech`, `Rezzer`, `CryosBasicLightningRanged`, `CryosBasicCharge`, `CryosSpecialOne`, `CryosSpecialTwo`, `ZelemBasicFlyingMelee`, `ZelemSpecialThree`, `NomadScope`, `NomadDrag` |
| `infinity_1` | 5-3, 8-1, 18-3 | `CitadelSpecificFour`, `CitadelSpecificThree`, `CitadelBasicShield`, `CitadelBasicMelee`, `CitadelSpecialTwo`, `CitadelSpecialThree`, `NomadShielder`, `NomadRuption`, `NomadSpecialThree`, `ZelemBasicRangedHoming`, `VerdanthBasicRanged`, `CryosElementalSpecialThree`, `NocturnaSpecialHomer`, `Boomer` |
| `infinity_4` | 5-4, 10-3, 13-2 | `CitadelSpecificTwo`, `CitadelBasicGunner`, `CitadelBasicShield`, `CitadelSpecialTwo`, `CitadelSpecialThree`, `CitadelSpecialFour`, `ZelemBasicRangedHoming`, `ZelemBasicHybrid`, `ZelemSpecialTwo`, `CryosBasicRanged`, `NomadDrag`, `NomadSnipe`, `NomadScope`, `NomadShielder`, `NocturnaSpecialDrift` |
| `scaldron_2` | 6-1, 9-1, 17-2 | `ScaldronBasicMines`, `ScaldronBasicThorno`, `ScaldronBasicDog`, `ScaldronBasicNestle`, `ScaldronBasicSinkhole`, `ScaldronBasicBlink`, `NomadBioSpecialTwo`, `NomadScope`, `NomadWithDrone`, `VerdanthSpecialThree`, `CryosSpecialOne`, `ZelemSpecialThree`, `Rezzer`, `Boomer` |
| `scaldron_1` | 6-2, 12-4, 14-1 | `ScaldronBasicCopter`, `ScaldronBasicMonk`, `ScaldronBasicSinkhole`, `ScaldronBasicDog`, `ScaldronBasicMines`, `NomadScope`, `NomadBioSpecialTwo`, `NomadShielder`, `NomadSpecialThree`, `NomadRuption`, `VerdanthBasicRootmob`, `VerdanthSpecialThree`, `ZelemSpecialOne`, `ZelemSpecialThree`, `CitadelSpecialThree` |
| `scaldron_3` | 6-3, 12-3, 18-4 | `ScaldronBasicNestle`, `ScaldronBasicBlink`, `ScaldronBasicMaser`, `ScaldronBasicDoppler`, `ScaldronBasicCopter`, `ScaldronBasicDog`, `NomadWithDrone`, `NomadScope`, `NomadSpecialThree`, `VerdanthSpecialTwo`, `ZelemSpecialOne`, `NocturnaSpecialHomer` |
| `scaldron_4` | 6-4, 9-2, 15-3 | `ScaldronBasicDoppler`, `ScaldronBasicSinkhole`, `ScaldronBasicMines`, `ScaldronBasicThorno`, `ScaldronBasicBlink`, `VerdanthSpecialThree`, `NomadWithDrone`, `NomadScope`, `NomadSnipe`, `NocturnaSpecialDrift`, `ZelemBasicHybrid`, `ZelemSpecialHaster`, `ZelemSpecialThree`, `CryosBasicRanged`, `CryosSpecialOne` |

All 24 campaign maps have wanderer, spike, and horde marker sets. Outside that
chain, the four `Spectra_*` records and `tnx173_3` have spike and wanderer sets
but no authored horde set; `tnx173_2` and `TNX_173` do have horde sets. This
describes map capability, not the number of waves selected at runtime.

## Community wiki cross-reference

The [Game Wiki enemy category](https://gamegame.fandom.com/wiki/Category:Enemies)
uses player-facing names and classes, while the build stores developer noun
names. Its 127 category members are not 127 one-to-one noun families: the list
also includes named captains, destructors, variants, category pages, and other
non-base entries.

The useful public taxonomy is:

| Build naming signal | Wiki terminology | Boundary |
| --- | --- | --- |
| Most `*Basic*` nouns | Minion or Shooter | `Basic` does not distinguish melee minions from shooters. |
| Most `*Special*` / `*Specific*` nouns | Lieutenant | Strong role correlation, but not a string-level alias by itself. |
| `_Captain` nouns | Named Captain derived from a lieutenant | The first-pass position table below provides the strongest alias evidence. |
| `*Boss` nouns | Destructor / planet boss | Zoo placement proves assets, not campaign boss scheduling. |
| `MutationAgent` | Mutation Agent | Exact name match; the wiki describes it as a distinct gated-horde actor that can turn another mob into an Elite. |

### Locale-backed archetype aliases

These aliases are present in build 103. For each row, the decoded
`ClassAttributes` payload names the internal asset and references the listed
locale key; `localization_text` resolves that key to the exact English display
name. This materially strengthens the wiki cross-reference and, in particular,
directly establishes that `CryosSpecialOne` is the retail `Quadrakiller`
archetype seen as Quadra in the beta footage.

| Build family | Locale key | Build-103 English display name |
| --- | --- | --- |
| `ZelemSpecialHaster` | `0x4e974cb8` | Haster |
| `ZelemSpecialOne` | `0xf991099b` | Warp Spawner |
| `ZelemBasicMelee` | `0xcf2fcfc4` | Scorpiod |
| `ZelemBasicRangedHoming` | `0xd80b8cdd` | Homing Striker |
| `VerdanthBasicPlunge` | `0x532534bf` | Tentacler |
| `NomadSpecialThree` | `0x211b931c` | Acid Shell |
| `NocturnaBasicHealthDrain` | `0x664164ee` | Draining Simian |
| `NoctBasicFlyer` | `0x54eafec3` | Necrodactyl |
| `NocturnaBasicRangedSilence` | `0xe80e1438` | Muting Leucopod |
| `NoctBasicHopper` | `0xd7418123` | Vampiric Leaper |
| `NocturnaSpecialLeech` | `0x9abda773` | Necrotic Leech |
| `nct_lieu_su_stealther` | `0x52563455` | Pterodyne |
| `VerdanthSpecialOne` | `0x5b76f1b5` | Botanical Tunneler |
| `NomadSpecialOne` | `0x14988edd` | Ragetusk |
| `NomadSnipe` | `0x1b47ac41` | Decelerator |
| `ZelemSpecialTwo` | `0x6c12b023` | Magnetic Master |
| `CryosElementalSpecialThree` | `0x645d29f0` | Ray Killer |
| `NomadRuption` | `0x289c256c` | Molten Crawler |
| `CryosSpecialThree` | `0x294cdb67` | Hypno Mantis |
| `VerdanthSpecialTwo` | `0x6dd5450c` | Mending Tanglid |
| `CitadelSpecialFour` | `0x977b37db` | Reconstructionist |
| `NomadShielder` | `0x55dc95ee` | Shielded Grenadier |
| `CryosSpecialTwo` | `0xe0720906` | Terrorsaur |
| `CryosSpecialOne` | `0xe0650396` | Quadrakiller |
| `NocturnaSpecialMunch` | `0xce3606cf` | Carrion Shambler |
| `Rezzer` | `0x39c6a45a` | Animus |
| `CitadelBasicMelee` | `0x6fc008b5` | Pyro |
| `Boomer` | `0xba4f920b` | Lightning Juggernaut |
| `ZelemBasicFlyingMelee` | `0xa49597e5` | Sting Raider |
| `ZelemBasicPackMelee` | `0x5ce84787` | Pack Brawler |
| `NomadDrag` | `0x868fb56d` | Dimensionist |
| `ZelemBasicChargeup` | `0x7b2822a2` | Pincering Carapace |
| `CitadelSpecificThree` | `0x7299e673` | Robo-bomber |
| `ZelemSpecialThree` | `0x0a9ab998` | Raytheoid |
| `ZelemBasicPackfly` | `0xcb832e76` | Strafing Drakon |
| `VerdanthBasicMelee` | `0xaf30bebc` | Chrono Striker |
| `Shooter` | `0x9008bf85` | Blasting Fiend |
| `NocturnaSpecialHomer` | `0x7e5de4c5` | Arachno Striker |
| `NocturnaSpecialDrift` | `0xce308fbc` | Shade Drifter |
| `nct_minn_su_drainer` | `0xb8f27624` | Parasitic Thresher |
| `CryosBasicLightningRanged` | `0x0d204d50` | Electron Burster |
| `CitadelSpecialThree` | `0x94d3e072` | Laser Tank |
| `CitadelSpecialTwo` | `0x50584aa0` | Suppression Mechanoid |
| `NomadBioSpecialTwo` | `0x3d0bc0f6` | Pouncing Stalker |
| `NomadScope` | `0xec75ec19` | Undermind |
| `NomadWithDrone` | `0x87cb094a` | Invincitron |
| `VerdanthSpecialThree` | `0x40bfec9e` | Grappling Pulsar |

Additional direct joins used elsewhere in this note are `Sloth` / Electric
Sloth (`0xe5a08eb0`), `ScaldronBasicSinkhole` / Sinkhole (`0x7553e38e`), and
`MutationAgent` / Mutation Agent (`0xe23f759f`). The first two names and keys
also occur in their decoded `ClassAttributes` resources; Mutation Agent has an
exact build/wiki name and locale row.

### First-pass captain aliases

The wiki's [Game roster](https://gamegame.fandom.com/wiki/The_Game)
lists one named Onslaught captain at each threat level. Matching that unique
position to `chain_level` and the level's base `_Captain` entry gives the threat
association below. The decoded build-103 Captain classes now independently
confirm every listed internal family, display name, base class, and affix list;
only assignment of a Captain as the terminal actor on an X-4 slot conflicts with
the six dedicated `*Boss` classes and remains a scheduling question. See
`notes/campaign/enemy-class-audit.md` for exact class ordinals and discrepancies.

| Threat | Level | Build base captain noun | Wiki captain and base archetype |
| --- | --- | --- | --- |
| 1-1 | `zelems_1` | `ZelemSpecialHaster_Captain` | Illust, the Accelerator (`Haster`) |
| 1-2 | `zelems_3` | `ZelemSpecialOne_Captain` | Zunh, the Singularity Void (`Warp Spawner`) |
| 1-3 | `nocturna_4` | `NocturnaSpecialLeech_Captain` | Mizod, the Life Leech (`Necrotic Leech`) |
| 1-4 | `nocturna_1` | `nct_lieu_su_stealther_Captain` | Khoo, the Unseen (`Pterodyne`) |
| 2-1 | `verdanth_1` | `VerdanthSpecialOne_Captain` | Contagion, the Deadly Epidemic (`Botanical Tunneler`) |
| 2-2 | `verdanth_3` | `NomadSpecialOne_Captain` | Roark, the Blood-Fury (`Ragetusk`) |
| 2-3 | `zelems_2` | `NomadSnipe_Captain` | Mikella, the Temporal Impeder (`Decelerator`) |
| 2-4 | `zelems_4` | `ZelemSpecialTwo_Captain` | Edict, the Treacherous Genius (`Magnetic Master`) |
| 3-1 | `cryos_4` | `CryosElementalSpecialThree_Captain` | Persi, the Lightning Zealot (`Ray Killer`) |
| 3-2 | `cryos_3` | `NomadRuption_Captain` | Krife, the Ascendant Crater (`Molten Crawler`) |
| 3-3 | `verdanth_2` | `CryosSpecialThree_Captain` | Yegg, the Hypno-Lord (`Hypno Mantis`) |
| 3-4 | `verdanth_4` | `VerdanthSpecialTwo_Captain` | Delphi, the Chieftain (`Mending Tanglid`) |
| 4-1 | `infinity_2` | `CitadelSpecialFour_Captain` | Ciminator, the Unyielding (`Reconstructionist`) |
| 4-2 | `infinity_3` | `NomadShielder_Captain` | TCD-24, the Grand Bombardier (`Shielded Grenadier`) |
| 4-3 | `cryos_1` | `CryosSpecialTwo_Captain` | Zain, the Hellfire Death (`Terrorsaur`) |
| 4-4 | `cryos_2` | `CryosSpecialOne_Captain` | Ronin, the Vindicator (`Quadrakiller`) |
| 5-1 | `nocturna_3` | `NocturnaSpecialMunch_Captain` | Kane, the Chief Cannibal (`Carrion Shambler`) |
| 5-2 | `nocturna_2` | `Rezzer_Captain` | Weaver, the Final Judge (`Animus`) |
| 5-3 | `infinity_1` | `CitadelSpecialThree_Captain` | MGP-2, the Impassable Fury (`Laser Tank`) |
| 5-4 | `infinity_4` | `CitadelSpecialTwo_Captain` | RED-D-TOR, the Karmic Destroyer (`Suppression Mechanoid`) |
| 6-1 | `scaldron_2` | `NomadBioSpecialTwo_Captain` | Feng, the Relentless Predator (`Pouncing Stalker`) |
| 6-2 | `scaldron_1` | `NomadScope_Captain` | Kaai, the Craven Blaster (`Undermind`) |
| 6-3 | `scaldron_3` | `NomadWithDrone_Captain` | DLS-1227, the Impenetrable Defender (`Invincitron`) |
| 6-4 | `scaldron_4` | `ScaldronBoss` | The Corruptor |

The 2-4 row identifies its ordinary director captain only. It is not the final
boss mapping: build 103 has a separate `ZelemBoss` family directly localized
as Polaris, The Gravity Manipulator, and the `zelems_4` boss listener owns that
dedicated encounter. Edict/Magnetic Master must remain in the director-captain
catalog rather than replacing Polaris.

The locale/package join directly confirms `CryosSpecialOne` as the build family
for the public `Quadrakiller`/Quadra archetype and `ZelemSpecialHaster` as
`Haster`. Direct asset/locale matches additionally support `Sloth` as
[Electric Sloth](https://gamegame.fandom.com/wiki/Electric_Sloth),
`ScaldronBasicSinkhole` as
[Sinkhole](https://gamegame.fandom.com/wiki/Sinkhole), and `MutationAgent`
as [Mutation Agent](https://gamegame.fandom.com/wiki/Mutation_Agent).

### Why the wiki and binary unions differ

The wiki's [Zelem's Nexus](https://gamegame.fandom.com/wiki/Zelem%27s_Nexus)
page explicitly says its sector lists apply to the first visit, while replays
randomize ordinary enemies; mini-bosses remain level-specific, and X-4 boss
behavior depends on prior discovery. This explains why a level's 21 embedded
director nouns are broader than a six-enemy first-visit wiki list.

The public Genesis names also correct the internal fixture labels:

- internal `chrono` test content corresponds to public **Quantum**;
- internal `supernatural` test content corresponds to public **Necro**;
- Cryos is primarily Plasma, Verdanth primarily Bio, Infinity primarily Cyber,
  Nocturna primarily Necro, and Zelem's Nexus primarily Quantum;
- [Scaldron](https://gamegame.fandom.com/wiki/Scaldron) deliberately mixes
  all five Genesis types, so a `Scaldron*` prefix must not be treated as one
  science type.

Planet and prefix therefore describe content origin better than guaranteed
Genesis type. For example, the wiki classifies Sinkhole as Quantum even though
its build noun is `ScaldronBasicSinkhole`.

## Tutorial, Spectra, TNX, and arena records

| Level | Recovered mob evidence |
| --- | --- |
| `Game_Tutorial_cryos_1` | Director: `TutorialBasicPoison`, `TutorialBasicDiseased`, `TutorialBasicRanged`, `TutorialSloth`, `TutorialSpecialOne`. Fixed markers additionally prove `TutorialBasicPoisonNoOrbs` and `TutorialSpecialOne_Intro`. |
| `Spectra_1` | Same 21-entry director pool as the other Spectra records: `NocturnaBasicHealthDrain`, `NoctBasicHopper`, `NocturnaSpecialMunch`, `NocturnaSpecialLeech`, `nct_lieu_su_stealther`, `Rezzer`, `Shooter`, `VerdanthBasicPlunge`, `VerdanthBasicSkeet`, `VerdanthBasicHealer`, `CitadelSpecificThree`, `NomadSpecialThree`, `NomadWithDrone`, `NomadSpecialOne`, `NomadScope`, `CryosSpecialThree`. |
| `Spectra_2` | Same pool as `Spectra_1`. |
| `Spectra_3` | Same pool as `Spectra_1`. |
| `Spectra_4` | Same pool as `Spectra_1`. |
| `test_AI_arena` | Director: `VerdanthBasicRanged`, `VerdanthBasicDiseased`, `VerdanthSpecialOne`, `VerdanthSpecialTwo`, `VerdanthSpecialThree`, `CryosBasicPoison`, `CryosSpecialThree`, `CitadelBasicShield`, `CitadelBasicGunner`, `CitadelSpecialFour`, `ZelemBasicRepair`, `ZelemBasicPackMelee`, `ZelemSpecialThree`, `NomadSpecialOne`, `NomadShielder`, `Boomer`. Fixed markers include 16 `Sloth` instances. |
| `test_survivor_arena` | Director: `ZelemBasicMelee` tiers 1-3. These are the only entries whose normalized rows are explicitly horde-legal in the current extraction. |
| `test_VFX_arena` | Director: `Sloth`, `Boomer`. |
| `tnx173_2` | `CitadelSpecificTwo` tiers 1-3; `CitadelSpecialTwo` and its captain tiers; `CitadelSpecificThree`, `CitadelSpecialFour`, `ZelemBasicRangedHoming`, `NomadDrag`. |
| `tnx173_3` | The `tnx173_2` set plus `VerdanthBasicPicky`, `VerdanthBasicSkeet`, `VerdanthBasicPlunge`, `VerdanthBasicMelee`, `VerdanthSpecialOne`, `VerdanthSpecialTwo`, `VerdanthSpecialThree`, `NomadWithDrone`, `NomadSpecialOne`, and `NocturnaSpecialDrift`. |
| `TNX_173` | Director: `VerdanthBasicRanged` tiers 1-3. |

## Variant and non-combat records

This table accounts for every remaining level row. "Same director pool" means
the binary contains the same mob noun set, even when the map mode may not use
the campaign director in normal play.

| Level | Result |
| --- | --- |
| `cryos_1_SM` | Same director pool as `cryos_1`. |
| `cryos_2_PVP` | Same director pool as `cryos_2`. |
| `infinity_1_PVP` | Only `CitadelBasicSuicide` is embedded as a director noun. |
| `infinity_4_SM` | Only `CitadelSpecificTwo` tiers 1-3 and captain forms of `CitadelSpecialTwo` and `CitadelSpecialThree` are embedded. |
| `nocturna_2_PVP` | Only `nct_minn_su_drainer` tiers 1-3 are embedded. |
| `nocturna_3_SM` | Same director pool as `nocturna_3`. |
| `scaldron_1_PVP` | Same director pool as `scaldron_1`. |
| `scaldron_4_SM` | Same director pool as `scaldron_4`. |
| `verdanth_3_PVP` | Same director pool as `verdanth_3`. |
| `verdanth_4_SM` | Same director pool as `verdanth_4`. |
| `zelems_1_SM` | Same director pool as `zelems_1`. |
| `zelems_2_PVP` | Only `ZelemBasicRanged` is embedded as a director noun. |
| `Creature_Vid_Capture` | No director mob nouns. Marker contents are camera/lighting/stage objects. |
| `CreatureEditor_EL` | No director mob nouns. Marker contents are editor scenery and lighting. |
| `front_end_ship` | No director mob nouns. |
| `Juggernaut_Mode_Testing` | No director or fixed mob nouns recovered. |
| `test_Creature_Vid_Capture` | No director or fixed mob nouns recovered. |
| `test_holodeck` | No director or fixed mob nouns recovered. |

## Fixed AI zoo catalog

The zoo maps are test fixtures rather than campaign spawn pools, but their
fixed markers establish the broader mob catalog and its authored grouping.
Tier-2, tier-3, and captain forms are collapsed below.

| Level | Fixed mob families |
| --- | --- |
| `test_AI_zoo` | Player-creature test lineup only (`PC_*`); no enemy director pool. |
| `test_AI_zoo_bio` | `CitadelBoss`, `CryosBoss`, `CryosBasicPoison`, `CryosSpecialThree`, `NomadBioSpecialTwo`, `NomadSpecialOne`, `NomadSpecialThree`, `ShadowBoss`, `VerdanthBasicDiseased`, `VerdanthBasicHealer`, `VerdanthBasicOoze`, `VerdanthBasicPicky`, `VerdanthBasicPlunge`, `VerdanthBasicRanged`, `VerdanthBasicRootmob`, `VerdanthBasicSkeet`, `VerdanthBoss`, `VerdanthSpecialOne`, `VerdanthSpecialTwo`, `ZelemBoss`, `ZelemSpecialHaster`. |
| `test_AI_zoo_chrono` | `MutationAgent`, `NocturnaSpecialHaunt`, `NomadCyberOne`, `NomadDrag`, `NomadPets`, `NomadSnipe`, `NomadSpacetimeAgent`, `NomadSpecialFour`, `VerdanthBasicMelee`, `VerdanthSpecialThree`, `ZelemBasicChargeup`, `ZelemBasicFlyingMelee`, `ZelemBasicHybrid`, `ZelemBasicMelee`, `ZelemBasicPackfly`, `ZelemBasicPackMelee`, `ZelemBasicRanged`, `ZelemBasicRangedHoming`, `ZelemSpecialHaster`, `ZelemSpecialOne`, `ZelemSpecialTwo`. |
| `test_AI_zoo_cyber` | `CitadelBasicGunner`, `CitadelBasicRanged`, `CitadelBasicShield`, `CitadelBasicSuicide`, `CitadelSpecialTwo`, `CitadelSpecialThree`, `CitadelSpecialFour`, `CitadelSpecificOne`, `CitadelSpecificTwo`, `CitadelSpecificThree`, `CitadelSpecificFour`, `Critter`, `NomadShielder`, `NomadWithDrone`, `SackOfHitPoints`, `ZelemBasicRepair`, `ZelemSpecialThree`. |
| `test_AI_zoo_elites` | Captain forms of `CitadelSpecialTwo/Three/Four`, `CryosElementalSpecialThree`, `CryosSpecialOne/Two/Three`, `nct_lieu_su_stealther`, `NocturnaSpecialLeech/Munch`, `NomadBioSpecialTwo`, `NomadRuption`, `NomadScope`, `NomadShielder`, `NomadSnipe`, `NomadSpecialOne`, `NomadWithDrone`, `Rezzer`, `VerdanthSpecialOne/Two/Three`, and `ZelemSpecialHaster/One/Two`. |
| `test_AI_zoo_plasma` | `Boomer`, `CitadelBasicMelee`, `CryosBasicCharge`, `CryosBasicFiery`, `CryosBasicFireWave`, `CryosBasicLightningMelee`, `CryosBasicLightningRanged`, `CryosBasicMelee`, `CryosBasicRanged`, `CryosElementalSpecialThree`, `CryosSpecialOne`, `CryosSpecialTwo`, `NomadRuption`, `NomadScope`, `Sloth`. |
| `test_AI_zoo_supernatural` | `nct_lieu_su_stealther`, `nct_minn_su_drainer`, `NoctBasicFlyer`, `NoctBasicGhostCharger`, `NoctBasicHopper`, `NoctBasicMeleeDog`, `NocturnaBasicHealthDrain`, `NocturnaBasicRangedSilence`, `NocturnaBasicStealth`, `NocturnaSpecialDrift`, `NocturnaSpecialHomer`, `NocturnaSpecialLeech`, `NocturnaSpecialMunch`, `Rezzer`, all nine recovered `ScaldronBasic*` families, and `Shooter`. |
| `test_AI_zoo_ugc` | Fixed base-form samples: `Boomer`, `CitadelSpecialThree`, `CryosBasicFiery`, `CryosBasicMelee`, `CryosBasicPoison`, `CryosBasicRanged`, `CryosElementalSpecialThree`, `CryosSpecialOne`, `CryosSpecialThree`, `nct_lieu_su_stealther`, `nct_minn_su_drainer`, `Rezzer`, `Shooter`, `VerdanthSpecialThree`, and `ZelemBasicRanged`. |

## Important limits

1. The normalized `level_director_entry` decoder now follows each contiguous
   noun run back to its immediately preceding 16-byte records. This preserves
   interleaved director configurations instead of scanning only the prefix
   before the first noun. Each row now retains its contiguous configuration
   and within-configuration ordinals. Retail-package tests recover all six
   tutorial rows in three pools and all 24 `zelems_1` rows in four pools.
   Build-103 reflection and instruction evidence now labels `zelems_1`'s four
   configurations `minion`, `special`, `agent`, and `captain`. Other levels'
   `config_kind` and every entry-level `spawn_kind` remain `unknown`; marker
   spawn kind `5` is not flattened onto the agent pool rows.
2. The binary order strongly suggests multiple director configurations and
   difficulty bands, but their native vector boundaries have not yet been
   instruction-checked. This note does not label a noun "normal", "first
   visit", "spike", or "horde-only" without that proof.
3. Boss nouns found in zoo fixtures prove that the assets exist, not that a
   specific campaign level's random director can spawn them.
4. A director candidate is eligibility evidence, not evidence that every run
   will select it. Runtime traces are still required for weights, budgets, and
   first-visit substitutions.
