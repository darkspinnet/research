# `zelems_1` opening population

## Result

The build-103 content does **not** contain a fixed opening enemy roster. The
level contains no preplaced `Zelem*`, `Nomad*`, `Citadel*`, `Sloth`, `Boomer`,
or other combatant noun. Instead it contains director control objects:

- 341 ungated `SpawnPoint_DirectorWanderer.Noun` loci in three marker sets;
- 54 ungated `SpawnPoint_DirectorSpike.Noun` loci in twelve marker sets;
- nine `SpawnPoint_DirectorHorde.Noun` loci whose listener events gate them
  behind `boss triggered` or `horde triggered`; and
- one `SpawnPoint_DirectorBoss.Noun` at the exit/boss cluster.

The 395 Wanderer/Spike controls are therefore the only ungated, ordinary
director-placement inputs in the level content that could contribute an
opening population. They are not 395 enemies, they do not author an enemy noun
per position, and the examined evidence does not recover whether or how the
retail server consumes them.
The linked Lua chunks create tutorial job objects, update obelisks, and make
one late boss callback call to `ActivateHordeSpawn`; none creates the opening
population.

Build 103 proves how the client-shaped `cAIDirector` registers these control
objects and how its noun selector treats the marker classes. It does not expose
the authoritative retail-server call, initial budget, chosen loci, chosen
nouns, object IDs, or replication order. Consequently the exact live opening
count, noun sequence, positions, and send order remain unknown. The tables
below report candidate content order and positions, never a claimed live
spawn roster.

## Evidence boundary and provenance

The authoritative runtime source is
`bin/darkspinner/darkspin/cache/content.db`, manifest build 103. Level row
`56` is `zelems_1`; its source resource is `10356`, decoded size `2746`, and
SHA-256
`ba952b41e36b111055c80a668ee9229173558eca270133726c40887517b5afe0`.
The campaign start is design marker row `125492`, authored ID `1472396301`,
ordinal `51`, `SpawnPoint_Affix.Noun` at
`(-123.8707,-151.6471,10.0371)`.

The level has 41 ordered marker-set references. AI placement begins at level
marker-set ordinal `23`; the later horde sets are ordinals `38` and `39`.
`level_marker_set.weight` is preserved exactly below. No examined instruction
proves that this number is a probability, a number of chosen markers, a
population multiplier, a budget, or a temporal order.

The canonical build-103 client evidence is
`bin/game/GameBin/Game.c` and the paired build-103 executable:

- `sub_9FA9B0` at `0x009FA9B0` classifies the noun-class hashes:
  `SpawnPoint_DirectorWanderer.Noun` (`0xA3E463F5`) as kind `7`,
  `SpawnPoint_DirectorSpike.Noun` (`0x0218BCC7`) as kind `8`,
  `SpawnPoint_DirectorHorde.Noun` (`0x45E6B07B`) as kind `5`, and
  `SpawnPoint_DirectorBoss.Noun` (`0xD12A4EFE`) as kind `9`.
- `sub_9FB420` at `0x009FB420` registers a nonzero, non-6, non-9 marker in the
  director's spatial store. It copies the classified kind, noun handle,
  position, orientation, and source metadata. It does not create an enemy.
  Kinds `6` and `9` enter a separate spatial registry.
- `sub_9FE270` at `0x009FE270` is a noun-list selector. Kind `7` dispatches to
  `sub_9FDD50`; kind `8` calls `sub_9FD260` and can return one noun; kind `5`
  uses the level's `agent` array and can return up to 15 budget-accepted nouns.
  The complete build-103 xref audit finds only one direct caller,
  `sub_9FE7D0` at call site `0x009FEA1B`; it passes kind `1`, a literal budget
  of `10`, and current difficulty. No shipped client caller passes kind `5`,
  `7`, or `8` to this selector. None of these routines is the missing
  retail-server object-create sender.
- Kind `7`'s `sub_9FDD50` directly gathers difficulty-eligible mode-0 and
  mode-1 candidates, an adjacent-mode-0 set, and the director's cached agent
  candidates. It applies director tuning and a caller budget and stops at 15
  accepted nouns. The cache construction and planet/section overlays prevent
  reducing that routine to a fixed `zelems_1` noun list from the level bytes
  alone.
- Kind `8` calls `sub_9FD260(this, 2, difficulty, -1)`. Mode `2` is the
  reflected `boss` slot in the current level configuration. The base
  `zelems_1.Level` boss array is empty. The native helper can consult inherited
  or overlay configuration when present, so the empty base array proves no
  base Spike noun, not that every Spike marker is necessarily inert in retail.

## Ordinary ambient director loci

These fifteen sets own all 395 non-horde director controls. None of their
normalized marker rows has a `level_event`. All marker nouns are control nouns,
not combatant nouns. "First" and "last" mean marker ordinal inside that one
set. Bounds summarize every exact position in `marker.position_x/y/z`.

| Level ordinal | Marker set | Weight | Kind | Count | First / last authored position | Complete bounds |
| ---: | --- | ---: | ---: | ---: | --- | --- |
| 23 | `zelems_1_AI_WandererA.Markerset` | 1 | 7 | 106 | `0` `(-156.5351,-62.8340,0.0473)` / `105` `(603.4646,33.5749,10.0193)` | x `-195.79..603.46`, y `-92.12..79.09`, z `-0.28..10.02` |
| 24 | `zelems_1_AI_WandererB.Markerset` | 1 | 7 | 141 | `0` `(604.6926,17.0451,10.0200)` / `140` `(565.5222,-16.7841,32.5030)` | x `513.38..667.64`, y `-76.15..61.40`, z `10.02..33.39` |
| 25 | `zelems_1_AI_WandererC.Markerset` | 1 | 7 | 94 | `0` `(205.1850,586.5431,9.8386)` / `93` `(135.4905,608.5067,5.1089)` | x `-623.69..271.58`, y `552.58..693.53`, z `0.13..10.12` |
| 26 | `zelems_1_AI_SpikeA.Markerset` | 1 | 8 | 4 | `0` `(-174.5169,-35.5528,-0.1374)` / `3` `(-162.8052,-32.6617,-0.1374)` | x `-174.52..-156.39`, y `-46.69..-32.66`, z `-0.14` |
| 27 | `zelems_1_AI_SpikeA2.Markerset` | 1 | 8 | 4 | `0` `(-152.5186,59.8994,0.7673)` / `3` `(-164.2209,66.7669,0.7673)` | x `-164.22..-144.58`, y `57.33..66.77`, z `0.77` |
| 28 | `zelems_1_AI_SpikeA3.Markerset` | 1 | 8 | 2 | `0` `(589.5574,2.6150,10.0190)` / `1` `(598.6703,7.2518,10.0220)` | x `589.56..598.67`, y `2.61..7.25`, z `10.02` |
| 29 | `zelems_1_AI_SpikeB.Markerset` | 1 | 8 | 6 | `0` `(555.6336,0.5702,33.7040)` / `5` `(541.7663,-16.7180,33.7061)` | x `536.90..558.12`, y `-16.72..3.57`, z `33.39..33.71` |
| 30 | `zelems_1_AI_SpikeB2.Markerset` | 2 | 8 | 2 | `0` `(562.1500,17.6349,33.7112)` / `1` `(537.1849,41.4385,33.7114)` | x `537.18..562.15`, y `17.63..41.44`, z `33.71` |
| 31 | `zelems_1_AI_SpikeB2b.Markerset` | 2 | 8 | 12 | `0` `(573.5119,44.4682,33.7053)` / `11` `(536.0914,33.0854,33.7112)` | x `536.09..574.33`, y `26.33..50.35`, z `33.38..33.71` |
| 32 | `zelems_1_AI_SpikeB2c.Markerset` | 2 | 8 | 11 | `0` `(596.1297,50.9434,33.3885)` / `10` `(583.6135,58.4935,33.3886)` | x `578.28..596.13`, y `46.26..60.36`, z `33.38..33.39` |
| 33 | `zelems_1_AI_SpikeC.Markerset` | 1 | 8 | 1 | `0` `(-539.7283,561.3768,0.1342)` | exact point |
| 34 | `zelems_1_AI_SpikeC2.Markerset` | 1 | 8 | 4 | `0` `(187.0261,562.5551,10.8358)` / `3` `(190.6066,576.7438,10.8378)` | x `187.03..210.07`, y `562.56..576.74`, z `10.83..10.84` |
| 35 | `zelems_1_AI_SpikeC3.Markerset` | 1 | 8 | 3 | `0` `(149.3298,666.5987,5.0500)` / `2` `(155.1611,656.6558,5.0427)` | x `149.33..158.99`, y `656.66..666.60`, z `5.04..5.82` |
| 36 | `zelems_1_AI_SpikeC4.Markerset` | 1 | 8 | 4 | `0` `(214.7422,635.5497,5.8265)` / `3` `(205.6790,634.6757,5.8170)` | x `202.82..214.74`, y `617.55..635.55`, z `5.82..5.83` |
| 37 | `zelems_1_AI_SpikeC5.Markerset` | 1 | 8 | 1 | `0` `(261.9312,682.4788,10.0391)` | exact point |

The three Wanderer sets form dense placement fields rather than authored
enemy groups. The closest Wanderer marker to the campaign start is set `23`,
marker ordinal `50`, row `127892`, authored ID `2304648816`, at
`(-159.7800,-92.1250,0.0444)`, 70.23 three-dimensional units from the start.
The next five are ordinals `23`, `16`, `15`, `80`, and `24`, all in set `23`,
at distances `71.40`, `76.82`, `82.28`, `82.99`, and `83.76`. Proximity does
not prove which are selected or created first.

### Exact Spike locus order

The smaller Spike sets can be listed losslessly. Duplicate coordinates are
preserved: set `32` ordinals `0` and `1` are distinct authored markers at the
same position.

| Set ordinal | Marker ordinals in authored order: `(x,y,z)` |
| ---: | --- |
| 26 | `0 (-174.5169,-35.5528,-0.1374)`; `1 (-172.4892,-46.6871,-0.1375)`; `2 (-156.3938,-44.3835,-0.1374)`; `3 (-162.8052,-32.6617,-0.1374)` |
| 27 | `0 (-152.5186,59.8994,0.7673)`; `1 (-159.1954,63.4200,0.7673)`; `2 (-144.5759,57.3323,0.7673)`; `3 (-164.2209,66.7669,0.7673)` |
| 28 | `0 (589.5574,2.6150,10.0190)`; `1 (598.6703,7.2518,10.0220)` |
| 29 | `0 (555.6336,0.5702,33.7040)`; `1 (536.8992,-5.5863,33.7093)`; `2 (558.1236,-9.2961,33.6232)`; `3 (541.1439,-0.1382,33.3856)`; `4 (548.3451,3.5689,33.7025)`; `5 (541.7663,-16.7180,33.7061)` |
| 30 | `0 (562.1500,17.6349,33.7112)`; `1 (537.1849,41.4385,33.7114)` |
| 31 | `0 (573.5119,44.4682,33.7053)`; `1 (547.8395,39.7592,33.3894)`; `2 (557.8315,30.6482,33.7111)`; `3 (573.9824,50.3471,33.7084)`; `4 (542.0598,45.2421,33.7117)`; `5 (541.4079,40.0703,33.3893)`; `6 (548.9026,43.7075,33.3894)`; `7 (574.3304,40.4948,33.3799)`; `8 (570.1772,40.1495,33.3846)`; `9 (569.1590,45.8351,33.3865)`; `10 (563.8970,26.3324,33.7114)`; `11 (536.0914,33.0854,33.7112)` |
| 32 | `0 (596.1297,50.9434,33.3885)`; `1 (596.1297,50.9434,33.3885)`; `2 (587.8324,46.9066,33.3865)`; `3 (583.8101,54.9665,33.3879)`; `4 (588.7513,58.3008,33.3887)`; `5 (578.2783,58.1254,33.3886)`; `6 (578.4402,49.1697,33.3855)`; `7 (593.6906,60.3648,33.3891)`; `8 (582.2514,46.2611,33.3848)`; `9 (593.6741,48.5956,33.3880)`; `10 (583.6135,58.4935,33.3886)` |
| 33 | `0 (-539.7283,561.3768,0.1342)` |
| 34 | `0 (187.0261,562.5551,10.8358)`; `1 (210.0720,565.6699,10.8342)`; `2 (197.5175,567.5135,10.8255)`; `3 (190.6066,576.7438,10.8378)` |
| 35 | `0 (149.3298,666.5987,5.0500)`; `1 (158.9871,665.4090,5.8192)`; `2 (155.1611,656.6558,5.0427)` |
| 36 | `0 (214.7422,635.5497,5.8265)`; `1 (206.9845,626.2043,5.8251)`; `2 (202.8171,617.5496,5.8295)`; `3 (205.6790,634.6757,5.8170)` |
| 37 | `0 (261.9312,682.4788,10.0391)` |

Every one of the 341 Wanderer rows is likewise retained losslessly in
`content.db` in `(level_marker_set.ordinal, marker.ordinal)` order. Their
individual coordinates are candidate positions, not proven spawn positions;
the bounds and opening-nearest records above are the useful population facts.

## Event-gated controls excluded from the opening

These ten controls are not part of an unconditional opening population.

| Set / level ordinal / weight | Control | Authored positions and event |
| --- | --- | --- |
| `zelems_1_design_spawners.Markerset` / `4` / `1` | four kind-5 Horde listeners | `(940.4082,642.7053,0.0880)`, `(959.2801,698.7213,0.0880)`, `(924.6718,699.2067,0.0880)`, `(972.6259,652.4561,0.0880)`; all `boss triggered -> HordeSpawner_Register` |
| same set | one kind-9 Boss control | row `126389`, ID `223774364`, ordinal `7`, `(948.3736,674.0089,0.0880)`; boss/exit object-lifetime event also owns the linked tutorial callback |
| `zelems_1_Ai_Horde_1.Markerset` / `38` / `2` | three kind-5 Horde listeners | `(594.6318,4.3739,10.0209)`, `(602.3293,20.7509,10.0208)`, `(583.9745,20.7509,10.0208)`; all `horde triggered -> HordeSpawner_Register` |
| `zelems_1_Ai_Horde_2.Markerset` / `39` / `2` | two kind-5 Horde listeners | `(-525.2462,561.1359,0.1342)`, `(-544.6460,568.9210,0.1342)`; both `horde triggered -> HordeSpawner_Register` |

Horde trigger controls are separate from those listener loci: set `38` row
`128245` is at `(592.4940,11.3251,10.0210)`, and set `39` row `128253` is at
`(-533.6099,561.4774,0.1342)`. Their missing positive radius/activation
operand and their route order remain unresolved. The empty
`zelems_1_Ai_Horde_4.Markerset` is level ordinal `40`, weight `5`, decoded size
`46`, and contributes no marker.

Build 103 also does not activate either horde trigger on the client. Static
startup binds `HordeTrigger_OnEnterPlayer` to `sub_9FACB0`; every path through
that handler returns false, it does not call the selector or mutate director
state, and the generic trigger dispatcher therefore does not publish the
authored `horde triggered` event. This establishes a client/server boundary;
it does not establish what the retail server does when either volume is
entered.

## Noun and AI candidates

The base `zelems_1.Level` supplies 24 difficulty-bounded director entries in
four nonempty reflected pools. At a selected difficulty only eight rows are
eligible, representing seven distinct nouns because `ZelemBasicRanged` appears
in both `minion` and `agent`. For the authored low band (`1..24`) those rows
are:

| Reflected pool | Eligible base noun | Packaged AI definition / ability strings |
| --- | --- | --- |
| `minion` | `ZelemBasicRanged.Noun` | `ZelemBasicRanged.AIDefinition`; `ZelemBasicRanged_Blink` |
| `special` | `ZelemSpecialHaster_Captain.Noun` | `ZelemSpecialHaster.AIDefinition`; `CastZelemHasteBuff`, `ZelemHasterAttack` |
| `agent` | `ZelemBasicRanged.Noun` | same Ranged definition above |
| `agent` | `ZelemBasicHybrid.Noun` | `ZelemBasicHybrid.AIDefinition`; `ZelemBasicHybridProjectile`, `ZelemBasicHybridMelee` |
| `agent` | `ZelemBasicRepair.Noun` | `ZelemBasicRepair.AIDefinition`; `Repair`, `ArcWeldingMelee` |
| `captain` | `ZelemSpecialHaster.Noun` | `ZelemSpecialHaster.AIDefinition`; `CastZelemHasteBuff`, `ZelemHasterAttack` |
| `captain` | `NomadSnipe.Noun` | `NomadSnipe.AIDefinition`; `NomadSnipe_Slow`, `NomadSnipe_Melee` |
| `captain` | `NomadWithDrone.Noun` | `NomadWithDrone.AIDefinition`; `NomadWithDronePunch` |

The three ordinary presentation-name mappings are package-exact, not
walkthrough guesses. Type-`0xD117AFCA` class-attribute package ordinal `11839`
(instance `0xBDA683F8`, 506 bytes, SHA-256
`c9b45a212f1172afd5cf4eca60345ef7704084d16f130d625e31c526b6a83537`)
stores `Cannonator`, localization `AssetStrings!0xD8BB91DA`, and
`ZelemBasicHybrid.ClassAttributes`. Package ordinal `3681` (instance `0xA63F868D`,
486 bytes, SHA-256
`04164053585b5c6163510bb63e0ec1582e2844250c6db97ea5683c8c8d94c9b1`)
stores `Reparatron`, `AssetStrings!0x944FE251`, and
`ZelemBasicRepair.ClassAttributes`. Package ordinal `9353` (instance `0x8F291AF3`,
462 bytes, SHA-256
`c33e81a9ea10cbc72c01ce42035ee4ea81c755f7e428a4ad2217670bcbec542f`)
stores `Space Barracuda`, `AssetStrings!0x1EC89234`, and
`ZelemBasicRanged.ClassAttributes`. The installed English localization table
independently resolves the first two GUIDs to Cannonator and Reparatron; the
third label is literal in its class payload. These identities do not determine
which eligible noun the missing retail selector assigns to any particular
marker.

The four other ordinary names visible in the indexed walkthrough are also
package-exact identities, although their appearance does not prove which
level overlay or selector admitted them. Type-`0xD117AFCA` package ordinal
`5606` maps `Chrono Striker` / `AssetStrings!0xAF30BEBC` to
`VerdanthBasicMelee.ClassAttributes` (instance `0x029F47AB`, 485 bytes,
SHA-256 `af2a079cdb4d4dfd97573742e67bbb50d1fd7b908a09df247b0741461f521a59`).
Ordinal `7035` maps `Pack Brawler` / `AssetStrings!0x5CE84787` to
`ZelemBasicPackMelee.ClassAttributes` (instance `0xB8A61E77`, 471 bytes,
SHA-256 `4b71046e70f5d6d23bfa1aecbb33065a9683697109d2c05823f285452abf2e1c`).
Ordinal `6651` maps `Homing Striker` / `AssetStrings!0xD80B8CDD` to
`ZelemBasicRangedHoming.ClassAttributes` (instance `0x5926208F`, 477 bytes,
SHA-256 `5dbd455399719a92006bfb51a5fb6c289e53592beb0822499431d77ce0792e9c`).
Ordinal `11281` maps `Sting Raider` / `AssetStrings!0xA49597E5` to
`ZelemBasicFlyingMelee.ClassAttributes` (instance `0xC123F299`, 472 bytes,
SHA-256 `3a3f678f0c345235a3e1af0a287b17d7aaf5bd38ccbfcf2fb1cb8f4c85fdc5f2`).
Each GUID independently resolves to the same English label in
`localization_text`.

The exact higher bands are already normalized and remain relevant to replays:
rows `831/834/839..841/848..850` cover difficulty `25..48`, and rows
`832/835/842..844/851..853` cover `49..72`. All 24 entries have
`is_horde_legal=1`. Noun definitions prove `npcCreature` locomotion, packaged
AI links, scales/footprints, and empty class-local `dropType` arrays. They do
not prove the opening selector's chosen noun, count, stats after difficulty
scaling, aggro timing, or drops.

The level also names `Zelem.LevelConfig`. The exact contribution of that
planet configuration and any section/player-count overlay to the native
opening caches is not projected in `content.db`. This is why the base noun
table cannot safely be treated as the complete runtime selector input.

## Lua boundary

Level `56` joins 34 `level_script` rows to eight distinct Lua chunks. Their
decoded instructions show:

- chunks `735`, `468`, `62`, and `175` create `LuaJobObject.Noun` jobs for
  ability-unlock presentation;
- chunks `5`, `163`, `279`, and `736` implement obelisk objectives and
  interaction abilities; and
- only chunk `62`, linked to the far-end Boss control, calls
  `nGameDirector.ActivateHordeSpawn`, after its tutorial unlock waits.

No linked chunk names a Wanderer or Spike marker, chooses a `zelems_1` noun,
enumerates an opening position, or sends `ObjectCreate`. Lua therefore does
not create the opening population visible in the examined content.

Build 103 has no campaign-specific enemy-spawn packet. A materialized enemy
arrives through generic `kGmsObjectCreate` (`0x8c`) followed by ordinary
component state. `kGmsDirectorState` (`0x8b`) carries no noun, marker,
position, object ID, budget, or spawn command. The packet contract consequently
confirms create-before-component ordering only; it does not recover the
opening roster or the retail server's order of object-create packets.

## Ordering that is proven

Only content order is proven:

1. marker sets are referenced in level ordinals `23..37` for ordinary
   Wanderer/Spike controls;
2. markers inside each set retain ordinal order;
3. the event-gated Horde sets follow at ordinals `38` and `39`; and
4. the empty Horde-4 set is ordinal `40`.

This is serialization order, not spawn order, route order, activation order,
object-ID order, AI think order, or packet order. The `_A`, `_B`, `_C`, `_1`,
and `_2` suffixes and coordinate clusters are useful authored identities but
do not independently prove gameplay traversal.

## Unresolved facts

- The retail-server function that invokes the selector and constructs the
  initial enemy objects is absent from the build-103 client evidence.
- No shipped-client call edge connects the Wanderer, Spike, or Horde marker
  kinds to `sub_9FE270`; the sole direct caller passes kind `1`. Whether the
  retail server shares these selection algorithms or only their content model
  is unknown.
- It is unknown whether the opening population is built during level load,
  just before status `8`, after player readiness, by spatial streaming, or by
  another server-only phase.
- The initial director budget and all tuning inputs that choose a final count
  are unknown.
- The selected subset of the 341 Wanderer and 54 Spike loci is unknown. It is
  not proven that every set participates or that `weight` selects a set.
- The noun chosen for each selected locus is unknown. A marker fixes a control
  class and transform, not a combatant noun.
- The base Spike path addresses the empty `boss` pool, but inherited/planet
  overlays are unresolved; therefore Spike output is unknown rather than
  proven empty.
- The exact use of `Zelem.LevelConfig`, section configuration, player count,
  difficulty tuning, challenge override, and any per-marker spawn-point fields
  not yet projected from the raw marker payload is unknown.
- Rotations, scale, collision, and source-marker propagation exist in marker
  data, but the authoritative enemy-creation owner has not proven which fields
  it copies to the combatant.
- Live enemy count, duplicate-noun behavior, duplicate-position behavior,
  object IDs, creation order, component-update order, visibility/beam-in
  presentation, aggro targets, and late-join replay are unknown.
- No retail 1-1 packet capture or server trace was available to convert the
  candidate content order into a live opening roster.
- The horde contact radii are now recovered as `15` and `5` by content recipe
  34. Their missing server acceptance/budget policy and route order remain
  unknown; neither volume authorizes its listeners at the opening.

## Reproducibility

The principal database lookups were read-only `darkrun db` queries against
`bin/darkspinner/darkspin.toml`:

```text
darkrun db level get 56 --config bin/darkspinner/darkspin.toml
darkrun db level_marker_set get level_id=56 --limit 100 --config bin/darkspinner/darkspin.toml
darkrun db marker get level_marker_set_id=<1789..1829> --limit 1000 --config bin/darkspinner/darkspin.toml
```

Related detailed evidence is in `notes/campaign/1-1/overview.md`, `notes/campaign/1-1/director.md`,
`notes/campaign/1-1/native.md`, `notes/campaign/1-1/nouns.md`, `notes/campaign/1-1/budget.md`, and
`notes/campaign/1-1/route.md`.
