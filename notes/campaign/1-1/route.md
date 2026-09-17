# `zelems_1` marker and horde route map

## Evidence boundary

This map correlates three authored marker sets in level `56`:

| Marker set | DB ID | Content resource | Level ordinal / weight | Decoded SHA-256 |
| --- | ---: | ---: | --- | --- |
| `zelems_1_design.Markerset` | `1791` | `7327` | `2 / 1` | `4efc6f55e086969cd477cccf8e4e7842da16a8e682dbdc366a5600e713723c77` |
| `zelems_1_Ai_Horde_1.Markerset` | `1827` | `10489` | `38 / 2` | `b588c2110abe3aabe50f06e8b39bd7b5318fcf3bfff7cc2571e7540ad4903462` |
| `zelems_1_Ai_Horde_2.Markerset` | `1828` | `10484` | `39 / 2` | `d7ae7f2268d4bd1592e95682b8a7ce312069ad873a3119bda567962258928f7a` |

"Ordered" below means authored marker ordinal inside a marker set, with
geographic clusters sorted from the initial spawn only where coordinates make
the association exact. It is not a claim about runtime traversal: every
relevant normalized marker has `target_marker_id=0`, and the available
listener tuples do not encode a complete graph.

## Opening population is not established by these marker sets

The campaign start is design marker row `125492`, authored ID `1472396301`,
ordinal `51`, `SpawnPoint_Affix.Noun` at
`(-123.8707,-151.6470,10.0371)`. No `SpawnPoint_Director*` row is colocated
with that start, and no examined marker event proves an unconditional status-8
spawn. Therefore these marker sets do not identify an opening population.

The four director-horde loci in `zelems_1_design_spawners.Markerset` (DB
`1793`, resource `6590`, ordinal `4`, weight `1`) are instead clustered around
the exit/boss at about `(948,674,0)` and listen for the literal event
`boss triggered`. Their pool ownership and tuning are documented in
`notes/campaign/1-1/director.md`.

This boundary is decisive: an opening encounter must not be populated from
the boss cluster or the later `SpawnPoint_DirectorHorde` rows merely because
those rows share the level.

## Design-layer route anchors

The following pairs are authored in `zelems_1_design.Markerset`. Rows are
shown in marker ordinal order. A "pair" here means a close coordinate pair,
not a decoded target edge.

| Design ordinals | Security marker | Nearby teleporter spawn |
| --- | --- | --- |
| `45 / 40` | row `125486`, ID `1714532343`, `SecurityTeleporter.Noun-1` at `(-141.3413,83.7317,0.0340)` | row `125481`, ID `2315888312`, at `(-143.8270,77.3935,-0.1422)` |
| `46 / 36` | row `125487`, ID `1802649208`, `SecurityTeleporter.Noun-2` at `(556.1375,-38.2726,0.0175)` | row `125477`, ID `2377941581`, at `(555.5919,-31.4069,-0.4774)` |
| `47 / 41` | row `125488`, ID `952042169`, `SecurityTeleporter.Noun` at `(601.7911,42.2988,33.0519)` | row `125482`, ID `1954324840`, at `(597.2678,44.5016,33.0999)` |
| `20 / 42` | row `125461`, ID `4047011481`, `SecurityTeleporter.Noun-4` at `(-543.5760,546.1642,0.1581)` | row `125483`, ID `3461582646`, at `(-541.5687,553.5535,0.1342)` |
| `32 / 39` | row `125473`, ID `1915578781`, `SecurityTeleporter.Noun-3` at `(-625.0099,655.0704,0.1358)` | row `125480`, ID `2525303641`, at `(-621.0203,650.6170,0.6042)` |
| `21 / 37` | row `125462`, ID `1432124960`, `SecurityTeleporter.Noun-5` at `(191.8558,742.7648,0.0523)` | row `125478`, ID `1104450085`, at `(191.1970,736.2937,0.0443)` |

The boss-side design marker is row `125464`, authored ID `3977962581`,
ordinal `23`, `BossSecurityTeleporter.Noun-1` at
`(184.2345,558.9578,10.2027)`. Separately, design-spawner marker row `126389`
places `SpawnPoint_DirectorBoss.Noun-876848874` at
`(948.3736,674.0089,0.0880)`. Coordinate separation means their names alone
do not establish a direct edge.

## Later horde 1 cluster

`zelems_1_Ai_Horde_1.Markerset` occupies approximately
`x=584..611, y=-2..43, z=10`. Its authored order is:

| Ordinal / row / authored ID | Marker and position | Normalized event evidence |
| --- | --- | --- |
| `0 / 128237 / 2142959329` | `HordeGateTeleporter.Noun-2`, `(611.0300,-2.3597,10.6677)` | event `32469`: `pA -> HordeGateTeleporter_OnEnter` |
| `1 / 128238 / 805925445` | `TeleporterSpawnPoint.Noun`, `(595.1599,2.2673,10.0207)` | none |
| `2 / 128239 / 3232551417` | `HordeGateTeleporter.Noun-1`, `(601.3389,42.6983,10.0204)` | event `32470`: `pA -> HordeGateTeleporter_OnEnter` |
| `3 / 128240 / 3759343381` | `HordeGateTeleporter.Noun`, `(584.8389,38.6116,10.0204)` | event `32471`: `pA -> HordeGateTeleporter_OnEnter` |
| `4 / 128241 / 3319475621` | `SpawnPoint_DirectorHorde.Noun-2`, `(594.6318,4.3739,10.0209)` | event `32472`: `horde triggered -> HordeSpawner_Register` |
| `7 / 128244 / 3319475618` | `SpawnPoint_DirectorHorde.Noun-5`, `(602.3293,20.7509,10.0208)` | event `32473`: `horde triggered -> HordeSpawner_Register` |
| `8 / 128245 / 2244983668` | `SpawnPoint_HordeTrigger.Noun-1`, `(592.4940,11.3251,10.0210)` | event `32474`: `pA -> HordeTrigger_OnEnterPlayer` |
| `10 / 128247 / 150361665` | `SpawnPoint_DirectorHorde.Noun-10`, `(583.9745,20.7509,10.0208)` | event `32475`: `horde triggered -> HordeSpawner_Register` |

Ordinals `5`, `6`, and `9` are blocking-door objects. Thus this authored
cluster contains one player-entry trigger and three separately registered
director-horde loci. The strings do not prove trigger radius, activation
count, selected noun pool, or gate state transitions.

The cluster overlaps the design `SecurityTeleporter.Noun-2` /
`TeleporterSpawnPoint` pair at roughly `(556,-35)` and the elevated design
security/spawn pair around `(600,43,33)`. This is coordinate correlation only;
the vertical separation to the latter pair is about 23 units.

## Later horde 2 cluster

`zelems_1_Ai_Horde_2.Markerset` occupies approximately
`x=-550..-525, y=546..588, z about 0`. Its authored order is:

| Ordinal / row / authored ID | Marker and position | Normalized event evidence |
| --- | --- | --- |
| `3 / 128251 / 1408208687` | `SpawnPoint_DirectorHorde.Noun`, `(-525.2462,561.1359,0.1342)` | event `32476`: `horde triggered -> HordeSpawner_Register` |
| `4 / 128252 / 1067966471` | `SpawnPoint_DirectorHorde.Noun-7`, `(-544.6460,568.9210,0.1342)` | event `32477`: `horde triggered -> HordeSpawner_Register` |
| `5 / 128253 / 2142467792` | `SpawnPoint_HordeTrigger.Noun-2`, `(-533.6099,561.4774,0.1342)` | event `32478`: normalized raw name `" B" -> HordeTrigger_OnEnterPlayer` |
| `6 / 128254 / 244909030` | `HordeGateTeleporter.Noun-5`, `(-540.5402,546.4952,0.1342)` | event `32479`: `horde triggered -> HordeGateTeleporter_OnEnter` |
| `7 / 128255 / 3108860455` | `TeleporterSpawnPoint.Noun-2`, `(-538.4607,563.7571,0.0885)` | none |
| `8 / 128256 / 3548589009` | `HordeGateTeleporter.Noun-3`, `(-541.7651,587.6254,-0.1598)` | event `32480`: `pA -> HordeGateTeleporter_OnEnter` |
| `9 / 128257 / 786736646` | `HordeGateTeleporter.Noun-4`, `(-549.4912,548.9866,0.1342)` | event `32481`: `pA -> HordeGateTeleporter_OnEnter` |

Ordinals `0`-`2` are blocking-door objects. This cluster contains one
player-entry trigger and two separately registered director-horde loci. The
odd normalized event name `" B"` is preserved verbatim and is not interpreted
as a channel or phase.

The cluster directly overlaps the design `SecurityTeleporter.Noun-4` /
`TeleporterSpawnPoint` pair at about `(-543,550)`. The design
`SecurityTeleporter.Noun-3` pair lies farther northwest around
`(-623,653)`. Again, absent target IDs prevent proving which is entry or exit.

## Evidence-backed encounter/trigger ordering

1. **Opening state:** campaign start is design row `125492`. No examined
   `SpawnPoint_Director*` marker proves an opening population.
2. **Boss/exit cluster as authored:** four director-horde rows in marker set
   `1793` listen for `boss triggered` around the exit and boss marker. They are
   not opening placements.
3. **Horde cluster 1 as authored:** trigger row `128245` coexists with
   registered horde rows `128241`, `128244`, and `128247` inside marker set
   `1827`; gate callbacks are rows `128237`, `128239`, and `128240`.
4. **Horde cluster 2 as authored:** trigger row `128253` coexists with
   registered horde rows `128251` and `128252` inside marker set `1828`; gate
   callbacks are rows `128254`, `128256`, and `128257`.

The level ordinal (`38` then `39`) and suffix (`_1` then `_2`) provide a stable
content order for the two later clusters. They do not, by themselves, prove
that live gameplay must visit horde 1 before horde 2. No Lua chunk linked to
`zelems_1` names these horde marker IDs; native ownership remains a separate
question.
