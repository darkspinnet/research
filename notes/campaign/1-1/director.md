# `zelems_1` director pools and placements

## Result

No authored `SpawnPoint_Director*` marker in `zelems_1` proves an
unconditional opening population. All nine `SpawnPoint_DirectorHorde.Noun`
placements are listeners: four wait for `boss triggered`, and five wait for
`horde triggered`. The one `SpawnPoint_DirectorBoss.Noun` is at the level exit
inside the four-listener boss cluster. Consequently this evidence does **not**
authorize spawning any of these markers at campaign status 8, nor does it
establish whether `zelems_1_Ai_Horde_1` or `_2` is encountered first.

When one of the nine horde placements is activated, the build-103 director
classifies `SpawnPoint_DirectorHorde.Noun` as spawn kind `5`. The kind-5 branch
uses the `agent` director class. For `zelems_1`, that is normalized content-DB
configuration ordinal `2` (rows `836`-`844`), not the `minion`, `special`, or
`captain` array. At any one authored difficulty band it offers three eligible
nouns. Selection remains budget-driven and capped at fifteen returned nouns;
neither the level record nor these marker sets supplies a proven numeric
opening budget.

This separates three facts which must not be flattened together:

1. the level supplies eligible noun classes;
2. the marker-set weight and coordinates supply placement content; and
3. marker callbacks gate the boss and horde placements.

## Provenance

The authoritative level row is content DB `level.id=56`, `name=zelems_1`,
source resource `10356`, decoded size `2746`, SHA-256
`ba952b41e36b111055c80a668ee9229173558eca270133726c40887517b5afe0`.
It was extracted with `darkrun db level bget source_payload where id=56
--decode zlib` to `bin/game/logs/zelems_1.level.bin`.

The four nonempty arrays are identified by both serialized order and native
reflection, rather than by noun-name intuition. In build 103, the `LevelConfig`
descriptor registers arrays in this order at the following instructions:

| Serialized field | Reflection evidence | Content DB configuration | Entries |
| --- | --- | ---: | ---: |
| `minion` | `0x00F715E1` pushes `"minion"`; `0x00F7162A` registers `DirectorClass` | `0` | 3 |
| `special` | `0x00F716BA` pushes `"special"`; `0x00F71704` registers `DirectorClass` | `1` | 3 |
| `boss` | `0x00F7178A` pushes `"boss"`; `0x00F717D5` registers `DirectorClass` | empty | 0 |
| `agent` | `0x00F7185E` pushes `"agent"`; `0x00F718A9` registers `DirectorClass` | `2` | 9 |
| `captain` | `0x00F7192F` pushes `"captain"`; `0x00F71979` registers `DirectorClass` | `3` | 9 |

The raw level bytes independently preserve the four nonempty runs. Each entry
is exactly 16 bytes: noun asset pointer, inclusive minimum difficulty,
inclusive maximum difficulty, and `hordeLegal`. The runs begin at decoded
offsets `0x069A`, `0x0710`, `0x07C6`, and `0x0924`; their adjacent C strings
begin at `0x06CA`, `0x0740`, `0x0856`, and `0x09B4`. Every recovered
`zelems_1` entry has `hordeLegal=1`.

## Pool inventory

All IDs below are `level_director_entry.id` values in `content.db`. Difficulty
ranges are copied from the 16-byte records; no visit-state labels or selection
policy are inferred.

| Class / DB ordinal | Difficulty | Content DB IDs and nouns | Eligible count in band |
| --- | --- | --- | ---: |
| `minion` / `0` | 1-24 | `830` `ZelemBasicRanged.Noun` | 1 |
| | 25-48 | `831` `ZelemBasicRanged_2.Noun` | 1 |
| | 49-72 | `832` `ZelemBasicRanged_3.Noun` | 1 |
| `special` / `1` | 1-24 | `833` `ZelemSpecialHaster_Captain.Noun` | 1 |
| | 25-48 | `834` `ZelemSpecialOne_Captain_2.Noun` | 1 |
| | 49-72 | `835` `ZelemSpecialTwo_Captain_3.Noun` | 1 |
| `agent` / `2` | 1-24 | `836` `ZelemBasicRanged.Noun`; `837` `ZelemBasicHybrid.Noun`; `838` `ZelemBasicRepair.Noun` | 3 |
| | 25-48 | `839` `ZelemBasicRanged_2.Noun`; `840` `ZelemBasicPackMelee_2.Noun`; `841` `CitadelSpecificThree_2.Noun` | 3 |
| | 49-72 | `842` `ZelemBasicRanged_3.Noun`; `843` `ZelemBasicRepair_3.Noun`; `844` `Sloth_3.Noun` | 3 |
| `captain` / `3` | 1-24 | `845` `ZelemSpecialHaster.noun`; `846` `NomadSnipe.Noun`; `847` `NomadWithDrone.Noun` | 3 |
| | 25-48 | `848` `ZelemSpecialOne_2.Noun`; `849` `CitadelSpecialTwo_2.Noun`; `850` `Boomer_2.Noun` | 3 |
| | 49-72 | `851` `ZelemSpecialTwo_3.noun`; `852` `NomadShielder_3.noun`; `853` `NocturnaSpecialHomer_3.noun` | 3 |

The `agent` ownership of horde spawn kind `5` is instruction evidence:

- `sub_9FA9B0` at `0x009FA9B0` maps noun-class hash `0x45E6B07B`
  (`SpawnPoint_DirectorHorde.Noun`) to `5`; `sub_9FB420` calls it at
  marker registration and stores the kind in the registered marker record.
- `sub_9FE270` at `0x009FE270` dispatches kind `5` through its case beginning
  with `case 5`. That branch reads the director class at dword indices
  `210`/`211`, the fourth reflected class (`agent`), filters records by the
  current difficulty using inclusive comparisons, and repeatedly calls
  `sub_9FBA70`.
- `sub_9FBA70` at `0x009FBA70` chooses an index with `sub_9BCE90`, reads the
  noun's director cost from `[noun-data+0x24]`, compares the accumulated cost
  with the caller-supplied budget, and emits the noun. The kind-5 loop stops on
  budget failure or at count `15`. There is no per-entry weight in the
  16-byte level records; the observed choice operation is over the eligible
  vector. The numeric budget arrives as an argument and is not recovered from
  `zelems_1.Level` or the marker-set rows.

## Placement sets and weights

`level_marker_set.weight` is a set-level authored float. It is not evidence of
a noun-selection weight, enemy count, or temporal order.

| Set | DB identity and source | Weight | Director ownership |
| --- | --- | ---: | --- |
| `zelems_1_design_spawners.Markerset` | set `1793`, ordinal `4`, source resource `6590`, size `3458`, SHA-256 `8562a2f52110359ca3d98dfec8b37248f0e3de39c14b04de39c60dd2b790a217` | 1 | Four horde listeners around the exit/boss marker; activated by `boss triggered` |
| `zelems_1_Ai_Horde_1.Markerset` | set `1827`, ordinal `38`, source resource `10489`, size `5091`, SHA-256 `b588c2110abe3aabe50f06e8b39bd7b5318fcf3bfff7cc2571e7540ad4903462` | 2 | Three horde listeners paired with `SpawnPoint_HordeTrigger.Noun-1`; activated by `horde triggered` |
| `zelems_1_Ai_Horde_2.Markerset` | set `1828`, ordinal `39`, source resource `10484`, size `4740`, SHA-256 `d7ae7f2268d4bd1592e95682b8a7ce312069ad873a3119bda567962258928f7a` | 2 | Two horde listeners paired with `SpawnPoint_HordeTrigger.Noun-2`; activated by `horde triggered` |

### Boss cluster (`design_spawners`, weight 1)

All four horde rows have callback `HordeSpawner_Register` and event
`boss triggered` in level-event rows `32149`-`32152`.

| Marker row / authored ID | Marker ordinal and name | Position |
| --- | --- | --- |
| `126384` / `2145860737` | `2`, `SpawnPoint_DirectorHorde.Noun-4` | `(940.4082, 642.7053, 0.0880)` |
| `126385` / `2145860735` | `3`, `SpawnPoint_DirectorHorde.Noun-1` | `(959.2801, 698.7213, 0.0880)` |
| `126386` / `2145860738` | `4`, `SpawnPoint_DirectorHorde.Noun-8` | `(924.6718, 699.2067, 0.0880)` |
| `126387` / `2145860736` | `5`, `SpawnPoint_DirectorHorde.Noun-3` | `(972.6259, 652.4561, 0.0880)` |

The cluster center is marker row `126389`, authored ID `223774364`, ordinal
`7`, `SpawnPoint_DirectorBoss.Noun-876848874`, at
`(948.3736, 674.0089, 0.0880)`. `LevelExitPoint.Noun-651115636` is marker row
`126388` at `(941.0225, 672.3367, 0.1636)`. Those co-locations and the literal
`boss triggered` listener names establish boss/exit ownership; they are not an
opening placement.

### Horde set 1 (weight 2)

Level-event rows `32472`, `32473`, and `32475` bind these markers to
`horde triggered` / `HordeSpawner_Register`:

| Marker row / authored ID | Marker ordinal and name | Position |
| --- | --- | --- |
| `128241` / `3319475621` | `4`, `SpawnPoint_DirectorHorde.Noun-2` | `(594.6318, 4.3739, 10.0209)` |
| `128244` / `3319475618` | `7`, `SpawnPoint_DirectorHorde.Noun-5` | `(602.3293, 20.7509, 10.0208)` |
| `128247` / `150361665` | `10`, `SpawnPoint_DirectorHorde.Noun-10` | `(583.9745, 20.7509, 10.0208)` |

The owning contact marker is row `128245`, authored ID `2244983668`, ordinal
`8`, `SpawnPoint_HordeTrigger.Noun-1`, at `(592.4940, 11.3251, 10.0210)`.

### Horde set 2 (weight 2)

Level-event rows `32476` and `32477` bind these markers to `horde triggered` /
`HordeSpawner_Register`:

| Marker row / authored ID | Marker ordinal and name | Position |
| --- | --- | --- |
| `128251` / `1408208687` | `3`, `SpawnPoint_DirectorHorde.Noun` | `(-525.2462, 561.1359, 0.1342)` |
| `128252` / `1067966471` | `4`, `SpawnPoint_DirectorHorde.Noun-7` | `(-544.6460, 568.9210, 0.1342)` |

The owning contact marker is row `128253`, authored ID `2142467792`, ordinal
`5`, `SpawnPoint_HordeTrigger.Noun-2`, at
`(-533.6099, 561.4774, 0.1342)`.

## Limits

- Candidate-array size is not spawn count. The `agent` array has nine records,
  but only three match any one authored difficulty band.
- The native count of fifteen is an upper bound, not an authored wave count.
  Actual count depends on noun costs, the caller's budget, and repeated random
  choices.
- Set weights `1`, `2`, and `2` are preserved placement metadata. No examined
  instruction proves that they choose between these encounters or multiply a
  director budget.
- Marker coordinates and event links do not prove route order. The ordered
  route must come from trigger/teleporter connectivity or a runtime trace.
- The normalized DB currently labels `config_kind` and `spawn_kind` as
  `unknown`. The names above are evidence annotations from build-103 reflection
  and dispatch, not changes to the database schema.
