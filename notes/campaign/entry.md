# Campaign level entry transforms

## Result

No retail-authoritative initial player transform is recoverable from the
examined build-103 client/content corpus.

All 24 campaign levels have exactly four authored `CameraSpawnPoint.Noun`
markers in either `<level>_design.Markerset` or, for `cryos_1` and
`scaldron_1`, `<level>_design_spawners.Markerset`. A retired reference server
turns those camera markers into **position-only** player spawn choices:

1. load `<level>_design.Markerset`;
2. collect `CameraSpawnPoint.Noun` markers in authored marker order;
3. if the collection is empty, repeat with
   `<level>_design_spawners.Markerset`;
4. choose `point[playerID]`;
5. apply only its XYZ position to all three character objects.

That policy is exact for the retired source, but it is not evidence of the
original Game server's policy. It ignores the marker's authored rotation
and scale, supplies no player-facing rotation, and leaves a zero-initialized
position if the player ID is outside the four-point array. Therefore the
camera-marker rows below are implementation-ready evidence for reproducing or
removing the retired behavior, not authority to install them as a new
fallback.

The current darkspin constant
`(-123.8707,-151.64705,10.037109)` is a different retired local choice:
`zelems_1_design.Markerset` row `125492`,
`SpawnPoint_Affix.Noun`, marker ID `1472396301`. It is not the first
`CameraSpawnPoint.Noun`, is not selected by the retired reference algorithm,
and cannot be generalized to the other 23 levels.

## Source boundary

### Chain

`AssetData_Binary.package` is SHA-256
`faf7b72a27b7f6b6f2c497c9aa5379bd2f20b1e8fe7b59b0d3798a2232774f2b`
and contains 13,515 resources. Inventory entry `692`, synthetic name
`000692_a8a25294_00000000_00000000304f6f19.bin`, is
`ChainLevels.ChainLevels`: type `0xA8A25294`, group `0`, instance
`0x304F6F19`, decoded size `6780`, decoded SHA-256
`5c5e4419c1f4711f473ec1f4b94c4dc46330e2d166bc2101dc164ed379db20c6`.
It decodes to 72 contiguous zero-based slots and 24 unique `.Level`
references, each used three times.

The table deliberately begins with `zelems_3.Level` as requested. The raw
asset has `zelems_1.Level` at slot `0`; beginning at slot `1`, `zelems_3` is
the first unique level and `zelems_1` does not recur until slot `40`.

### Marker payloads

The normalized `content.db` rows come from
`AssetData_Binary.package`, not `Levels.package`. Every cited marker-set
resource has DBPF type `0xA11D3144`, group `0`; the per-row decoded SHA-256
below identifies the exact payload. Positions, rotations, scale, authored
marker IDs, names, and marker ordinals are decoded directly from the fixed
marker records. Package inventory calls these resources `refpack`; the
`source_compression=zlib` value on normalized marker-set rows describes
database payload storage, not the DBPF codec.

### Retired server choice

The retired source is
`bin/game/logs/recap_server-reference/game_server/source/game/instance.cpp`.
Its level-load path at lines `157-173` selects the design marker set, filters
`CameraSpawnPoint.Noun`, preserves the vector order returned by
`Markerset::GetMarkersByType`, and falls back to `design_spawners`. The parser
in `source/game/level.cpp` lines `266-289` appends markers to the per-noun
vector in XML/authored order. `instance.cpp` lines `409-422` indexes that
vector with `player->GetId()` and calls `SetPosition`; it never reads marker
rotation or scale.

This source postdates retail, is not an EA server capture, and contains
explicit emulator choices elsewhere. Its mapping is consequently classified
as **retired server choice**, with high confidence about what that source did
and no confidence transfer to retail authority.

### Client-local camera lookup

Build-103 `Game.c` `sub_4E75C0` at `0x004E75C0` (notably lines
`320303-320325`) hashes `CameraSpawnPoint.Noun`, finds the first matching
loaded object, and passes its XYZ twice to `sub_52D750`. `sub_52D750` at
`0x0052D750` (lines `370945-370967`) writes two XYZ triples into the
client camera-controller state and optionally snaps its current camera
position. It does not mutate a simulation/player object and does not construct
a RakNet message.

This independently proves that the noun is a client-local camera anchor. It
does not prove that the retail server also used the same marker as a player
spawn. The marker's authored Euler rotation and scale belong to that camera
asset and must not be copied into a player transform merely because the
retired server reused the XYZ.

### Packaged Lua and scenario data

`ServerData.package` is SHA-256
`845ef186dcc752071fff8fa43385a0bcd88e14ff8230001bc26b75f44a30892f`
and contains 1,101 resources. The 24 campaign levels link 954 marker callback
instances to 15 distinct packaged Lua chunks. Across those linked chunks,
`lua_string_constant` contains none of `SetPosition`, `TeleportObject`,
`TeleportAndFace`, or `CameraSpawnPoint.Noun`. Other unlinked ability and
modifier chunks do contain teleport natives, showing that the inventory can
observe those names when authored. No linked callback supplies an entry
transform.

The `.Level` payloads name marker sets, nav/physics assets, rendering and
planet configuration, music, level types, and camera pitch/yaw/distance.
No decoded scenario property names an initial player transform. Unknown
unlabeled binary fields cannot be promoted into one without a consumer.
`nScenarioManager` in this client exports only `GetIsClient`; no scenario
property lookup bridges the level asset to player creation.

The examined evidence therefore classifies the possibilities as follows:

| Candidate source | Finding |
| --- | --- |
| Level marker | Four exact `CameraSpawnPoint.Noun` records per level, but they are camera anchors; no marker is retail-proven as the player transform. |
| Scenario property | No named/consumed initial-transform property recovered. |
| Packaged Lua callback | Ruled out for the 15 chunks linked to these 24 levels by bytecode string inventory. |
| Client-local lookup | Exact first-camera-anchor lookup at `0x004E75C0`; it updates camera state only. |
| Retired server choice | Exact position-only `point[playerID]` reuse in the reference source; not retail authority. |
| Current darkspin choice | One `zelems_1` affix marker hard-coded for all campaign setup; local and retired, not evidence for other levels. |

## Per-level evidence table

Slots are zero-based raw `ChainLevels` ordinals. `P0..P3` are the four
`CameraSpawnPoint.Noun` positions in authored marker order and thus the
positions selected by the retired server for player IDs `0..3`. Coordinates
are stored IEEE-754 `float32`. Every P0 decimal triple (the singular initial
transform requested here) round-trips exactly to its stored float32 bits;
P1..P3 are compact formation-point context and `content.db` remains the exact
source for those additional players. The retired policy uses no rotation.
“Resource” is
`content_source_resource.id`; the SHA column is the full decoded marker-set
payload digest.

Every row has the same confidence boundary:

- authored marker identity/XYZ/rotation/scale: **exact**;
- retired `point[playerID]` position policy: **high**, direct source;
- retail-authoritative player position and orientation: **unresolved**.

| Level asset | Chain slots | Marker-set evidence | Retired P0..P3 positions |
| --- | --- | --- | --- |
| `zelems_3.Level` | `1,27,53` | `zelems_3_design.Markerset`; resource `8642`; SHA `c732927b6b09f376c821c4f3b6a3f466dc7cd0963e9847ef5a005cc84176558a` | P0 row `131747`, ord `23`, ID `2147159935`: `(127.230278,-816.944214,20.0413342)`; P1 `(127.002922,-823.417175,19.9372444)`; P2 `(123.170731,-828.064941,19.8330975)`; P3 `(118.203888,-816.175720,20.0818787)` |
| `nocturna_4.Level` | `2,42,54` | `nocturna_4_design.Markerset`; resource `10942`; SHA `de395df43d3b5afa13b6d35cca1c301ffadd2dc4f17e43926711c718d3f85a3a` | P0 row `68594`, ord `24`, ID `1003`: `(-262.022247,-204.411102,30.0433159)`; P1 `(-259.185883,-197.987656,30.0433178)`; P2 `(-265.607880,-199.476166,30.0433178)`; P3 `(-263.521301,-210.042938,30.0433159)` |
| `nocturna_1.Level` | `3,36,69` | `nocturna_1_design.Markerset`; resource `6873`; SHA `958120776805248890422ac9dc65238c8195a33286483a2a4b9c10b2baec12c7` | P0 row `38521`, ord `63`, ID `3549645868`: `(405.220367,486.460876,40.0368958)`; P1 `(394.623108,493.704193,40.0371361)`; P2 `(406.100006,480.601990,40.0364609)`; P3 `(400.504242,491.475555,40.0373573)` |
| `verdanth_1.Level` | `4,44,51` | `verdanth_1_design.Markerset`; resource `4633`; SHA `00d0cfb6eaabbc365964cdf7c377cafa6c078b332c70226fc89dd82840f30308` | P0 row `99844`, ord `3`, ID `2998884203`: `(-56.3271980,64.2435608,50.0176773)`; P1 `(-61.5976906,60.1535797,50.0206108)`; P2 `(-52.6886482,55.1491470,50.0259476)`; P3 `(-58.9585114,51.6897621,50.0282593)` |
| `verdanth_3.Level` | `5,31,60` | `verdanth_3_design.Markerset`; resource `4239`; SHA `bfca11e37a60803b12ab2224dabe6e8063340c457e7f5c68549e631948a829f9` | P0 row `110388`, ord `4`, ID `1819468259`: `(97.7455902,55.9422607,14.9954090)`; P1 `(102.729797,67.8011398,15.0045195)`; P2 `(95.7911987,64.1287994,15.0149279)`; P3 `(105.274734,55.6464996,14.9657078)` |
| `zelems_2.Level` | `6,26,56` | `zelems_2_design.Markerset`; resource `9747`; SHA `7614e5c29b1c300e06048e6420590feb3e4db805fb73705212983eb8d5ac20bb` | P0 row `129092`, ord `24`, ID `315235496`: `(1493.19824,-3150.38013,9.95715046)`; P1 `(1502.62000,-3160.59700,9.98392963)`; P2 `(1493.74121,-3156.89111,9.98922729)`; P3 `(1500.64185,-3154.73560,9.92217445)` |
| `zelems_4.Level` | `7,41,62` | `zelems_4_design.Markerset`; resource `13434`; SHA `ce5d75065d7286330bed51e83d995c3c90d488f8a44d626ec22a663c234e606b` | P0 row `134686`, ord `23`, ID `3266214076`: `(-98.2347870,1101.48328,40.1302261)`; P1 `(-101.436523,1094.20105,40.1416893)`; P2 `(-90.1145935,1102.17432,40.1393318)`; P3 `(-92.5616379,1092.73315,40.1395988)` |
| `cryos_4.Level` | `8,34,68` | `cryos_4_design.Markerset`; resource `4221`; SHA `265c2dbe456cfa0b6825643d2919f6c23ae1d513bf071bbca607d66531f5f174` | P0 row `14822`, ord `2`, ID `3031818957`: `(108.281197,-4.23311710,25.0106335)`; P1 `(112.724739,-9.22493744,25.0102005)`; P2 `(110.781013,2.28470802,25.0104904)`; P3 `(102.496933,-2.09710407,25.0109215)` |
| `cryos_3.Level` | `9,35,61` | `cryos_3_design.Markerset`; resource `9691`; SHA `cbae4e02d8cd7c7d848bef2ca6dbe6a8fb8f9a4ce7f47db74b6d3a3404ae1438` | P0 row `9151`, ord `81`, ID `3804531419`: `(49.3136063,26.5832863,50.0642509)`; P1 `(43.6367149,35.9682083,50.0642471)`; P2 `(55.0267868,33.2415962,50.0690384)`; P3 `(41.0837364,28.0658798,50.0642471)` |
| `verdanth_2.Level` | `10,30,66` | `verdanth_2_design.Markerset`; resource `534`; SHA `08464d0a31498f00541a51d233f7bd54542c3dcf0bf581c1e99af7489078dcf1` | P0 row `104369`, ord `2`, ID `622672447`: `(272.054016,569.855530,35.0471039)`; P1 `(267.417389,564.684204,35.0500832)`; P2 `(278.851196,566.470520,35.0462494)`; P3 `(274.525208,577.235474,35.0437775)` |
| `verdanth_4.Level` | `11,45,57` | `verdanth_4_design.Markerset`; resource `5152`; SHA `45fda9af9313fc5af8ff03117e9c01594521ded8f1575663f7e1d72c106efa77` | P0 row `117435`, ord `0`, ID `811440006`: `(325.322205,275.971161,30.0495815)`; P1 `(322.519012,271.282471,30.1645298)`; P2 `(319.702362,275.898468,30.1741142)`; P3 `(312.492371,270.073669,30.4839001)` |
| `infinity_2.Level` | `12,39,64` | `infinity_2_Design.Markerset`; resource `12266`; SHA `a4d8998db0a1e24c90932a931c8e6f89b5585e02b1835281386b0b3a22962d99` | P0 row `22513`, ord `1`, ID `2420523425`: `(-109.558868,144.863068,90.0850830)`; P1 `(-105.387627,149.727264,90.0903473)`; P2 `(-116.663986,143.482193,90.0880661)`; P3 `(-112.154358,152.854065,90.1000900)` |
| `infinity_3.Level` | `13,29,55` | `infinity_3_Design.Markerset`; resource `7065`; SHA `42254b41c19453cb4ff4e70aa8df6f59b5617dd79804b59685d2435331bd4e42` | P0 row `29192`, ord `0`, ID `2245657126`: `(950.028320,164.910233,110.088005)`; P1 `(949.805725,157.939972,110.088005)`; P2 `(956.093018,163.249466,110.088005)`; P3 `(950.750610,170.727295,110.088005)` |
| `cryos_1.Level` | `14,24,59` | `cryos_1_design_spawners.Markerset`; resource `12118`; SHA `5f666762f9bcf612262228654ff87570c132b125e71656ef60bcafd91ec212df` | P0 row `1874`, ord `2`, ID `1986976104`: `(11.3884268,150.754959,5.03761101)`; P1 `(3.59952593,150.178223,5.03970480)`; P2 `(12.3430367,161.007294,5.03830194)`; P3 `(7.37055492,156.196777,5.02965593)` |
| `cryos_2.Level` | `15,25,50` | `cryos_2_design.Markerset`; resource `10925`; SHA `bf9b5c769d18b4aadecc4a0821bb9b545cf42704a8718f042d14f71600c6d2f3` | P0 row `3407`, ord `19`, ID `1938005043`: `(-308.704071,287.523010,100.041999)`; P1 `(-316.096497,289.612091,100.036469)`; P2 `(-315.938507,281.390808,100.039940)`; P3 `(-308.566376,280.825531,100.075737)` |
| `nocturna_3.Level` | `16,43,48` | `nocturna_3_design.Markerset`; resource `4150`; SHA `f5acaf75cbd96cd7762ca91ff9eb3ebd38407f806475f3fcd2fe31338eeb1d43` | P0 row `55082`, ord `0`, ID `3969560720`: `(268.640137,-1.68312705,5.03932810)`; P1 `(262.793396,1.02901196,5.03594589)`; P2 `(280.603821,7.62057686,5.03932619)`; P3 `(275.608368,-0.236110002,5.03932619)` |
| `nocturna_2.Level` | `17,37,63` | `nocturna_2_design.Markerset`; resource `11783`; SHA `793b891d5d68d7d20bcf2f280600f0594171d53fa9f791855fa5e64b26a44fc7` | P0 row `46556`, ord `0`, ID `2713129948`: `(5.03196621,237.215012,5.03919411)`; P1 `(17.4333916,237.663696,5.04199505)`; P2 `(21.5890350,231.397690,5.04199600)`; P3 `(11.1304321,239.439499,5.04199219)` |
| `infinity_1.Level` | `18,28,70` | `infinity_1_design.Markerset`; resource `4427`; SHA `d940208b7ce8488c3e6f4ba8befa7e683d2c3744b7b3e389dd8146b4d0cd4c70` | P0 row `18712`, ord `1`, ID `1013566276`: `(9.30158710,14.2323074,140.032318)`; P1 `(12.2881622,20.6728077,140.035004)`; P2 `(2.76123905,13.0481949,140.033188)`; P3 `(-3.75751710,12.8804207,140.041382)` |
| `infinity_4.Level` | `19,38,49` | `infinity_4_Design.Markerset`; resource `8293`; SHA `10a54e15dc4b9010309dc791a57201a22e222e33d791531bd355ca5f29616e16` | P0 row `35135`, ord `0`, ID `3968340616`: `(-1237.44739,-179.059601,2.06617999)`; P1 `(-1231.23389,-185.128342,2.06634307)`; P2 `(-1224.17029,-192.091934,2.06634402)`; P3 `(-1227.60242,-188.683121,2.06634188)` |
| `scaldron_2.Level` | `20,32,65` | `scaldron_2_Design.Markerset`; resource `4667`; SHA `ca487a9ca318f9bcbe2aa84f391e52a7d375bcbac3299e49b844edbd076671bf` | P0 row `77401`, ord `56`, ID `140318677`: `(1215.17822,1523.42810,25.1437531)`; P1 `(1223.54358,1522.66492,25.0731792)`; P2 `(1214.30139,1515.04199,25.0731869)`; P3 `(1223.10327,1515.12195,25.0731792)` |
| `scaldron_1.Level` | `21,47,52` | `scaldron_1_design_spawners.Markerset`; resource `2072`; SHA `0ad83920096f2c657eb6d0e9cb99caf56272cfa75bb98eecee99131e3b567c5c` | P0 row `74347`, ord `0`, ID `2048578052`: `(-279.789886,1.99795902,10.2887115)`; P1 `(-282.475708,6.65713596,10.2829361)`; P2 `(-278.341125,-3.40992904,10.3118668)`; P3 `(-273.430817,-6.09780788,10.3225994)` |
| `scaldron_3.Level` | `22,46,71` | `scaldron_3_Design.Markerset`; resource `11185`; SHA `033a2970ab91540b517383fea42b7353a609cb4a2ded2df95fc5403b0473e9b7` | P0 row `81395`, ord `9`, ID `2494180667`: `(527.950623,120.319176,107.088005)`; P1 `(523.719727,124.320473,107.088020)`; P2 `(533.533752,116.363335,107.088005)`; P3 `(533.289246,124.335403,107.088005)` |
| `scaldron_4.Level` | `23,33,58` | `scaldron_4_Design.Markerset`; resource `6732`; SHA `1fe5016daeff36c6abb0e4a81396dcec555bd33c8215ca935124f577c02adf7a` | P0 row `84789`, ord `15`, ID `726722643`: `(-600.271729,388.112976,50.0880013)`; P1 `(-600.390930,395.150085,50.0880051)`; P2 `(-598.485840,380.303680,50.0880013)`; P3 `(-600.366455,401.822449,50.0879936)` |
| `zelems_1.Level` | `0,40,67` | `zelems_1_design.Markerset`; resource `7327`; SHA `4efc6f55e086969cd477cccf8e4e7842da16a8e682dbdc366a5600e713723c77` | P0 row `125442`, ord `1`, ID `619379378`: `(-122.593285,-154.478271,10.0478916)`; P1 `(-118.367218,-155.748184,10.0314198)`; P2 `(-124.804848,-150.001633,10.0304089)`; P3 `(-125.441238,-158.406952,10.0311642)` |

## Authored rotations are not player rotations

Each of the 96 camera markers also has an exact authored Euler rotation and
scale in `content.db.marker`. Those fields are deliberately omitted from the
compact table because neither identified player-position consumer uses them:

- the retired server copies only `marker->GetPosition()`;
- the client camera lookup passes only the marker object's XYZ to
  `sub_52D750`.

Examples show why treating the rotation as player facing would be unsafe:
`zelems_3` P0 has authored rotation approximately
`(0.00000239,-0.00000165,120.71299)`, while `zelems_1` P0 has
`(0.151152,-0.988323,24.034088)` and `verdanth_1` P0 has scale
`1.15979505`. These are exact camera-object transforms, not recovered
character transforms. Any implementation requiring initial facing remains
blocked on retail server evidence.

## Rejected lookalikes

The marker payloads also contain many plausible but semantically different
positions:

- `MapCamera.noun` markers are elevated heat-map/minimap cameras.
- `CameraSpawnPoint.Noun` markers are the four camera anchors catalogued
  above; only the retired server reuses their XYZ for players.
- `TeleporterSpawnPoint.Noun`, `SecurityTeleporter.Noun`,
  `BossSecurityTeleporter.Noun`, `HordeGateTeleporter.Noun`, and ordinary
  `Teleporter.Noun` rows form traversal/encounter destinations.
- `SpawnPoint_Director*` rows are director loci and do not identify a player
  entry.
- `SpawnPoint_Affix.Noun` is heavily repeated along authored affix paths. The
  unsuffixed `zelems_1` row happens to be near the camera cluster and became
  darkspin's current hard-coded local choice; name/proximity do not establish
  the other 23 starts.

No one of these classes may be substituted for an unresolved player transform
without new authority evidence.

## Unresolved retail boundary

All 24 levels remain unresolved for the original server's initial player
**position and orientation**. Resolving them requires at least one of:

- an original server source/configuration artifact that names the selection
  policy;
- a retail protocol capture containing the initial player `ObjectCreate` or
  authoritative teleport for known level/player indices;
- a server-authored scenario payload with a proven consumer; or
- additional executable/server code that constructs the player creation
  transform from these marker sets.

Until then, the exact camera-marker transforms and the retired selection
algorithm should remain evidence, not a fallback.
