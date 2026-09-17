# Enemy class and affix audit

## Scope

This audit cross-checks the community `Characters & Classes` roster against immutable build-103 content. The primary sources are decoded entries from `bin/game/Data/AssetData_Binary.package`, the imported `non_player_class` and `level_director_entry` tables in `content.db`, the fixed AI-zoo marker inventory, and the packaged `EliteModifier` Lua chunk. Decoded class artifacts are retained under `bin/game/logs/captain-affix-audit`.

A decoded named class is strong evidence because one payload contains its literal display name, inherited `ClassAttributes` family, localized behavior description, and exact `*.NPCAffix` references. A packaged noun or test-zoo placement proves that an actor exists, but does not by itself prove campaign scheduling.

## Named Onslaught Captain classes

All 24 first-band Captain class records exist. Twenty-three affix pairs agree with the wiki. TCD-24 is the material exception: build 103 contains `Accurate.NPCAffix` plus `Unstoppable.NPCAffix`, not Unstoppable plus Unstoppable Aura. Edict's packaged title is also `the Insidious Mastermind`, not `the Treacherous Genius`.

| Threat | Internal noun family | Package title | Package affixes | Class ordinal |
| --- | --- | --- | --- | ---: |
| 1-1 | `ZelemSpecialHaster_Captain` | Illust the Accelerator | Swift, Swift Aura | 6978 |
| 1-2 | `ZelemSpecialOne_Captain` | Zunh the Singularity Void | Shifted, Shifted Aura | 7263 |
| 1-3 | `NocturnaSpecialLeech_Captain` | Mizod the Life Leech | Spiky, Spiky Aura | 4138 |
| 1-4 | `nct_lieu_su_stealther_Captain` | Khoo the Unseen | Reflective, Swift | 457 |
| 2-1 | `VerdanthSpecialOne_Captain` | Contagion the Deadly Epidemic | Unstoppable, Unstoppable Aura | 1575 |
| 2-2 | `NomadSpecialOne_Captain` | Roark the Blood-Fury | Surefooted, Surefooted Aura | 638 |
| 2-3 | `NomadSnipe_Captain` | Mikella the Temporal Impeder | Frenzied, Frenzied Aura | 8026 |
| 2-4 | `ZelemSpecialTwo_Captain` | Edict the Insidious Mastermind | Ghostly, Unstoppable | 725 |
| 3-1 | `CryosElementalSpecialThree_Captain` | Persi the Lightning Zealot | Ghostly, Ghostly Aura | 12548 |
| 3-2 | `NomadRuption_Captain` | Krife the Ascendant Crater | Frenzied, Tough | 8436 |
| 3-3 | `CryosSpecialThree_Captain` | Yegg the Hypno Lord | Armored, Swift | 8913 |
| 3-4 | `VerdanthSpecialTwo_Captain` | Delphi the Chieftain | Spiky, Spiky Aura | 5342 |
| 4-1 | `CitadelSpecialFour_Captain` | Ciminator the Unyielding | Shielded, Unstoppable | 9885 |
| 4-2 | `NomadShielder_Captain` | TCD-24 the Grand Bombardier | Accurate, Unstoppable | 13510 |
| 4-3 | `CryosSpecialTwo_Captain` | Zain the Hellfire Death | Persistent, Persistent Aura | 8662 |
| 4-4 | `CryosSpecialOne_Captain` | Ronin the Vindicator | Deadly, Persistent | 3828 |
| 5-1 | `NocturnaSpecialMunch_Captain` | Kane the Chief Cannibal | Reflective, Reflective Aura | 10121 |
| 5-2 | `Rezzer_Captain` | Weaver the Final Judge | Ghostly, Swift | 7093 |
| 5-3 | `CitadelSpecialThree_Captain` | MGP-2 the Impassable Fury | Accurate, Shielded | 12395 |
| 5-4 | `CitadelSpecialTwo_Captain` | RED-D-TOR the Karmic Destroyer | Carapace, Unstoppable | 6561 |
| 6-1 | `NomadBioSpecialTwo_Captain` | Feng the Relentless Predator | Deadly, Tough | 10324 |
| 6-2 | `NomadScope_Captain` | Kaai the Craven Blaster | Carapace, Persistent | 12094 |
| 6-3 | `NomadWithDrone_Captain` | DLS-1227 the Impenetrable Defender | Deadly, Shielded | 189 |
| 6-4 | `VerdanthSpecialThree_Captain` | Sekely the Grand Overseer | Accurate, Armored | 10912 |

The threat column is a roster cross-reference, not proof that the Captain is the final X-4 actor. The package simultaneously contains the dedicated boss classes below. Server scheduling must choose between those independently proven assets.

## Standalone zone-boss classes

Build 103 contains six dedicated `*Boss` families, one matching each first-pass X-4 destination. Their class records contain no Captain affixes.

| Candidate slot | Noun | Package display name | HP | Challenge | Class ordinal | Evidence status |
| --- | --- | --- | ---: | ---: | ---: | --- |
| 1-4 | `ShadowBoss.Noun` | Nashira, The Shadow Void | 1500 | 500 | 9904 | selected by current catalog; noun links `ShadowBoss` AI/animation and `boss_blank_shader_effect` |
| 2-4 | `ZelemBoss.Noun` | Polaris, The Gravity Manipulator | 1500 | 500 | 9104 | selected by current catalog |
| 3-4 | `VerdanthBoss.Noun` | Orcus, Devourer of Life | 1500 | 500 | 4530 | selected by current catalog |
| 4-4 | `CryosBoss.Noun` | Merak, The Devastator | 1200 | 500 | 4500 | selected by current catalog through the standalone boss path |
| 5-4 | `CitadelBoss.Noun` | Arcturus, the Cybernetic Colossus | 1500 | 500 | 1519 | selected by current catalog through the standalone boss path |
| 6-4 | `ScaldronBoss.Noun` | The Corruptor | 800 | 0 | 4715 | selected by current catalog through the standalone boss path; packaged passive and stage-death Lua now drive the two-stage phase controller |

The compiled animation catalog supplies a distinct death sequence and duration for every packaged Destructor family: ShadowBoss `nct_boss_su_shadowboss_death` (227 frames, 8-second rounded boundary), ZelemBoss `zlm_boss_sp_death` (250, 9 seconds), VerdanthBoss `ver_boss_lf_spawneater_death` (525, 18 seconds), CryosBoss `cry_el_boss_death` (296, 10 seconds), CitadelBoss `ctd_boss_tc_death` (480, 17 seconds), and ScaldronBoss `sca_boss_death` (662, 23 seconds). Production uses those boundaries for corpse fade and final boss-complete presentation. Merak, Arcturus, and The Corruptor are selected for base 4-4, 5-4, and 6-4 through the shared standalone encounter lifecycle. Merak's burrow, melee, Chain Lightning, Shock, and damage-triggered add behavior are recovered, and The Corruptor's complete two-stage five-element phase and portal-population controller is active; the remaining exact phase-selection work belongs to the other Destructor families.

The six-class sequence strongly supports the wiki's broad Captain-versus-X-4 Destructor rule, but the missing retail server selector prevents claiming the exact scheduling transition from asset existence alone. The Corruptor is structurally exceptional: its zero challenge, stage-two passive, stage-two death modifier, and `SetBossId` calls indicate a bespoke finale rather than a generic Captain replacement.

Nashira is independently strong content rather than a renamed Captain. `ShadowBoss`, `_2`, and `_3` class rows each retain 1500 HP, 100 power, challenge 500, player-count health scale 1, and the localized mirror-image behavior description. The base class artifact is `nashira.class` with decoded SHA-256 `492e16ee7d13b24e64ae9d9e198362485dbcd53c5184353ef43a153c0ef9a9cf`.

## Operative candidates

Five decoded classes match the wiki's Genesis-specific Operatives by exact display name and lockdown behavior.

| Public name | Internal family | Class ordinal | Internal behavior evidence |
| --- | --- | ---: | --- |
| Vaulting Amphiod | `NomadSpecialFour` | 5521 | crystallizes one hero; allies must attack the crystal to free them |
| Omicron | `NomadCyberOne` | 12663 | cages one hero; allies must kill it to free them |
| Haunt Strider | `NocturnaSpecialHaunt` | 3056 | emerges from stealth, grips one hero, and explicitly requires other heroes to kill it |
| Magmatic Brute | `NomadPets` | 1162 | summons a minion that locks down one hero plus a fire-stream minion |
| Gravitic Confiner | `NomadSpacetimeAgent` | 8862 | locks one hero in a gravity bubble and can teleport when endangered |

None of these five families appears in `level_director_entry` for the 24 campaign maps. They occur together only as fixed actors in `test_AI_zoo_chrono`. This is compatible with co-op-only runtime injection, but does not recover the missing co-op selector, player-count gate, spawn budget, or release-object composition. They should be marked as packaged Operative candidates, not statically assigned to ordinary map anchors.

## Elites and Mutation Agents

`EliteModifier` is packaged chunk 650 (`Modifiers/0x6D6C743A.lua`, SHA-256 `a8e72677986648e5bdbfa29e334c0dbbff57981a38576559f1e976cb7ad4b2dc`). Its non-minion branch adds 75 percent maximum health, 50 percent damage, and 25 percent body scale; its minion branch adds 400 percent health, 150 percent damage, and the same scale increase. Elite is therefore a runtime modifier state, not a `DirectorSpike` marker class.

`MutationAgent` and the five Operative candidates are present in the fixed chrono AI zoo but absent from normalized campaign director entries. Mutation Agent's packaged passive proves a 15-unit selection radius, one promoted ally at a time, and rejection of already-Elite targets; its friendly shot further requires a living, non-stealthed, non-destructible, different-species ally. The package still does not prove when or where the retail server inserts the agent. Static maps label its one exact AI-zoo actor; campaign population uses the documented low-frequency 2-1-and-later fallback only at ordinary lieutenant-centered Spike clusters, never boss or horde planners. See `notes/campaign/mutation-agent.md`.

## Population implications

- Use exact decoded class names and affix lists for named identities; use the wiki only as a discovery index when package data exists.
- Keep `Basic`/`Special`/`Specific` and director configuration ordinals as structural hints, not universal public-class labels. `npc_rank` is also unsuitable: most named Captains report rank 1, Illust reports rank 3, and standalone bosses report rank 1.
- Promote the five Operative family aliases into catalog metadata without assigning static campaign anchors until the co-op insertion policy is recovered.
- Recover the remaining ranked Merak geyser selection plus Arcturus's exact alternate-phase behavior behind the active shared 4-4 and 5-4 standalone encounters.
- Import each decoded class display name, localization keys, description, and up to six literal NPC-affix references into normalized `content.db` rows. Named encounter identity now comes from that catalog instead of hand-maintained boss and Captain maps, and the complete class projection is available to future `darkrun map` labeling without inventing spawn policy.
