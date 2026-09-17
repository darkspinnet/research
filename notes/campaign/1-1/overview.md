# `zelems_1` Lua inventory

## Scope and evidence rule

This note inventories the Lua chunks joined to level `56`, `zelems_1`, by the
authoritative runtime `content.db` at
`bin/darkspinner/darkspin/cache/content.db`. It records authored links and
decoded Lua 5.1 instructions. It does not treat a link as server authority and
does not infer encounter policy from a callback name.

The level row is backed by content resource `10356` (decoded SHA-256
`ba952b41e36b111055c80a668ee9229173558eca270133726c40887517b5afe0`).
There are 34 `level_script` rows, 19 distinct `level_event` rows, and eight
distinct Lua chunks. The 34 rows arise because each of 15 obelisk events links
to both an objective chunk and an ability chunk, in addition to four tutorial
jobs.

## Direct chunk inventory

| Chunk | Server-data resource | Source identity | Size | SHA-256 | Linked callback / registration |
| ---: | ---: | --- | ---: | --- | --- |
| `5` | `13523` | `lua/0x409C9F2B.lua` | 2,356 | `e168a19cf64bddaff0b483b7d1b2546011d14cc4a72b5528331105c4b2133047` | callback `InteractWithObelisk`; registers objective `TouchAllObelisks` |
| `62` | `13584` | `0x24F78AA1/0x90CE5ECC.lua` | 1,172 | `b67c0aace5d108e5f6f7b2432f5d8be1aaee3f00d1204fdd56c6b03abcd878c4` | `nTutorial_SoloSupportUnlock.main` |
| `163` | `13694` | `lua/0x12410255.lua` | 2,187 | `ce065a01ba48775543cbbcb24807465398c2a3a8a1a94af8866c6f2dde99beda` | callback `InteractHealthObelisk`; registers objective `DontUseHealthObelisks` |
| `175` | `13709` | `0xA35BED24/0xB005B821.lua` | 1,081 | `afb48133fc171b8f9eca258893bceaa63e1721b609eddf81349decdde8249966` | `nTutorial_SoloSupportUnlockClient.main` |
| `279` | `13821` | `Abilities/0xFB748FEF.lua` | 2,380 | `d0e958f339e7d8e62357324f2ed56195cd4c001c8b8bad7cff813d454d9ab2c6` | registers `InteractHealthObelisk` |
| `468` | `14022` | `0xA35BED24/0x22EC11F2.lua` | 1,007 | `ff7b4677d624fb57c73dc9078d2d4f00fb71a7e6d03aed8d3315903aaab469e3` | `nTutorial_RandomUnlockClient.main` |
| `735` | `14307` | `0x24F78AA1/0x5C4C5BBF.lua` | 1,051 | `b9bf494972e98b73e60fe527bbfcaec6b66434abea4545adacbdb343c6364878` | `nTutorial_RandomUnlock.main` |
| `736` | `14308` | `Abilities/0xDAE4DF3D.lua` | 2,383 | `3adbc7c9dfa720d75c4e5c1ebd59f1b35f14398fa18956afc95c08171c8f82bb` | registers `InteractWithObelisk` |

The bytecode copies used for instruction inspection are under
`bin/game/logs/1-1-lua/chunk-<id>.luac`; their lengths and hashes match the
`lua_chunk` rows above.

## Require edges

The database's `lua_dependency` rows preserve every decoded direct
`require`. All targets are `NULL`; consequently the database proves the edge
names but no packaged target chunk or transitive closure.

| Chunk | Dependency rows in ordinal order |
| ---: | --- |
| `5` | dependency `4`: `Lua!GlobalDefinitions.lua` |
| `62` | dependencies `46`-`48`: `Lua!GlobalDefinitions.lua`, `Lua!TargetUtils.lua`, `Lua!Vector.lua` |
| `163` | dependency `122`: `Lua!GlobalDefinitions.lua` |
| `175` | dependencies `131`-`133`: `Lua!GlobalDefinitions.lua`, `Lua!TargetUtils.lua`, `Lua!Vector.lua` |
| `279` | none |
| `468` | dependencies `354`-`356`: `Lua!GlobalDefinitions.lua`, `Lua!TargetUtils.lua`, `Lua!Vector.lua` |
| `735` | dependencies `556`-`558`: `Lua!GlobalDefinitions.lua`, `Lua!TargetUtils.lua`, `Lua!Vector.lua` |
| `736` | none |

For the six chunks with requires, top-level bytecode has the corresponding
`GETGLOBAL require`, `LOADK <dependency>`, `CALL` sequences: chunk `5` PCs
`0`-`2`; chunks `62`, `175`, `468`, and `735` PCs `0`-`8`; chunk `163` PCs
`0`-`2`.

## Tutorial marker jobs

All four jobs are in marker set `1793`,
`zelems_1_design_spawners.Markerset`, content resource `6590`, ordinal `4`,
weight `1`, decoded SHA-256
`8562a2f52110359ca3d98dfec8b37248f0e3de39c14b04de39c60dd2b790a217`.

| Event / script row | Marker row; authored ID; position | Authored trigger fields | Bytecode facts |
| --- | --- | --- | --- |
| event `32147`; script `1023`; chunk `735` | `126382`; `164537526`; `TriggerZone.Noun-3` at `(648.4210,-10.6177,14.5211)` | `luaCallbackOnEnter=nTutorial_RandomUnlock.main`; radius `20`; once-only; not server-only | `main` prototype `0.0`: unbeaten-player loops at PCs `12`-`20` and `34`-`45`; waits use constants `2`, `4`, `6`, `5`, `2` at CALL PCs `26`, `30`, `49`, `53`, `57`; `UnlockNextAbility` is the GETTABLE/CALL at PCs `41`-`44`. No spawn/director call occurs. |
| event `32148`; script `1024`; chunk `175` | `126383`; `3299474850`; `TriggerZone.Noun` at `(947.2180,675.6621,0.1637)` | `luaCallbackOnEnter=nTutorial_SoloSupportUnlockClient.main`; radius `60`; once-only; not server-only | `main` prototype `0.0`: waits `2`, `4`, `0.5`, then blinks `Ability_Support1/2/3`, then waits `6.5` (CALL PCs `12`, `16`, `20`, `25`, `30`, `35`, `39`). This is group `0xA35BED24` client content. |
| event `32153`; script `1025`; chunk `62` | `126389`; `223774364`; `SpawnPoint_DirectorBoss.Noun-876848874` at `(948.3736,674.0089,0.0880)` | normalized callback `nTutorial_SoloSupportUnlock.main`; radius `0`; not once-only; not server-only | `main` prototype `0.0`: waits `2`, `4`, then unlocks unbeaten players, waits `2`, `5`, `2`, resolves the player's controlled object, and calls `nGameDirector.ActivateHordeSpawn` at PCs `63`-`67`. The call's return is discarded. The bytecode does not identify the native implementation or prove that it spawns anything. |
| event `32154`; script `1026`; chunk `468` | `126390`; `2302189799`; `TriggerZone.Noun-2` at `(646.2349,-12.5915,17.3011)` | `luaCallbackOnEnter=nTutorial_RandomUnlockClient.main`; radius `20`; once-only; not server-only | `main` prototype `0.0`: waits `2`, `4`, `0.5`, blinks `Ability_Enrage2`, then waits `5.5` and `5` (CALL PCs `12`, `16`, `20`, `25`, `29`, `33`). This is group `0xA35BED24` client content. |

Each job chunk's prototype `0.1` performs
`nObjectManager.CreateObject(<module>.jobObject)` at PCs `0`-`4`, followed by
`nThread.CreateThreadForObject` at PCs `5`-`12`. The asset is
`LuaJobObject.Noun`, established by the top-level `nUtil.GetAsset` sequence at
PCs `15`-`20`. These are job-object/thread construction calls, not enemy spawn
calls.

## Obelisk callbacks and markers

The 15 authored callback events are distributed across three weighted marker
sets. Each loot callback links chunks `5` and `736`; each health callback links
chunks `163` and `279`.

| Marker set | Content provenance | Callback marker rows (authored IDs) |
| --- | --- | --- |
| `1806`, `zelems_1_Obelisk_1.Markerset` | resource `8209`; ordinal `17`; weight `2`; SHA-256 `806c66684fc836e0bffbe3ac6d9fa141febe3daf0896420f6bd144f08e2d458b` | health `127727` (`2561056523`), loot `127729` (`2169929122`), health `127730` (`3944848041`), loot `127731` (`3381128876`), loot `127732` (`3119011996`) |
| `1807`, `zelems_1_Obelisk_2.Markerset` | resource `8212`; ordinal `18`; weight `2`; SHA-256 `7c371eb9cea147bfc89e60119de1435224d59898706921c6714a957fac874ef2` | health `127734` (`2721417117`), loot `127735` (`508067386`), health `127736` (`35454217`), loot `127737` (`2145860742`), loot `127738` (`2145860740`) |
| `1808`, `zelems_1_Obelisk_3.Markerset` | resource `8214`; ordinal `19`; weight `2`; SHA-256 `f8637290d261f8094c32b6fc145357a9e3ef969c949f8890a46fac3df61ca008` | health `127739` (`501523709`), loot `127740` (`2145860743`), loot `127742` (`2145860739`), health `127743` (`3584110962`), loot `127744` (`2145860741`) |

The corresponding event IDs are `32447`-`32461` in the same order after
accounting for absent marker ordinals. The normalized event kind is either
`callback` or `listener_or_trigger`; all have radius zero and neither once-only
nor server-only set. Some normalized `event_name` strings are neighboring
marker names or `obelisks`. They are retained database facts, not evidence of
publisher semantics.

### Objective chunks

- Chunk `5` defines `nObjective_Obelisks`, handles
  `nObjectiveEvents.TouchedObelisk`, and registers `TouchAllObelisks` at
  top-level PCs `53`-`55`. Its `Init` prototype (`0.0`) calls
  `GetNumInteractableObjects(SPID("InteractWithObelisk"))` at PCs `9`-`15`.
  Prototype `0.2` reads the event GUID, checks/increments interactable use, and
  publishes objective integer data through CALL PCs `8`, `12`, `18`, `36`,
  `44`, and `54`. Status selection is in prototype `0.3`.
- Chunk `163` defines `nObjective_HealthObelisks`, handles
  `TouchedHealthObelisk`, and registers `DontUseHealthObelisks` at top-level
  PCs `33`-`35`. Its `Init`, event handling, and status prototypes have the
  same instruction shape as chunk `5`, with health-obelisk constants. These
  chunks register and update objectives; they do not create enemies.

### Ability chunks

- Chunk `279` defines and registers `nAbility_InteractHealthObelisk` /
  `InteractHealthObelisk` at top-level PCs `60`-`64`. Authored constants set
  range `1`, `timetohit=0.4000000059604645`,
  `timetorelease=0.6000000238418579`, and
  `finalWaitTime=2.740000009536743`; no numeric assignment adjacent to
  `timeUntilLootDrops` is present. Prototype `0.0` stops locomotion, sets the
  animation, emits `TouchedHealthObelisk`, waits, changes graphics/adds the
  effect, calls `DropStuffForObject(..., nDropTypes.Orb)` at PCs `88`-`94`,
  releases the agent, and removes physics at PCs `108`-`115`.
- Chunk `736` defines and registers `nAbility_InteractWithObelisk` /
  `InteractWithObelisk` at top-level PCs `60`-`64`. Constants set range `1.5`,
  `timetohit=0.6000000238418579`, `timeUntilLootDrops=1`, and
  `finalWaitTime=2.740000009536743`; no numeric assignment adjacent to
  `timetorelease` is present. Prototype `0.0` emits `TouchedObelisk`, waits,
  then calls `DropStuffForObject` with `nBit.Or(nDropTypes.Loot,
  nDropTypes.Crystal)` at PCs `87`-`98`; later waits/releases and removes
  physics at PCs `99`-`119`.

## Complete opcode inventories

These counts cover all prototypes in each immutable chunk and provide a
compact cross-check against the PC citations above.

| Chunk | Opcode counts |
| ---: | --- |
| `5` | `ADDx1, CALLx22, CLOSUREx4, EQx6, GETGLOBALx52, GETTABLEx44, GETUPVALx3, JMPx14, LEx3, LOADBOOLx10, LOADKx14, MOVEx7, NEWTABLEx1, RETURNx10, SETGLOBALx1, SETTABLEx18, SUBx1, TESTx1` |
| `62` | `CALLx19, CLOSUREx2, FORLOOPx2, FORPREPx2, GETGLOBALx23, GETTABLEx17, GETUPVALx1, JMPx3, LENx1, LOADBOOLx2, LOADKx13, MOVEx13, RETURNx3, SETGLOBALx1, SETTABLEx3, SUBx2, TESTx3` |
| `163` | `ADDx1, CALLx19, CLOSUREx4, EQx6, GETGLOBALx45, GETTABLEx41, GETUPVALx3, JMPx14, LEx3, LOADBOOLx10, LOADKx11, MOVEx7, NEWTABLEx1, RETURNx10, SETGLOBALx1, SETTABLEx15, SUBx1, TESTx1` |
| `175` | `CALLx16, CLOSUREx2, GETGLOBALx23, GETTABLEx17, GETUPVALx1, JMPx1, LOADKx8, MOVEx6, RETURNx3, SETGLOBALx1, SETTABLEx3, TESTx1` |
| `279` | `CALLx31, CLOSUREx2, EQx1, GETGLOBALx55, GETTABLEx42, JMPx2, LOADKx9, MOVEx22, NEWTABLEx1, RETURNx5, SETGLOBALx1, SETLISTx1, SETTABLEx17, TESTx1` |
| `468` | `CALLx15, CLOSUREx2, GETGLOBALx20, GETTABLEx14, GETUPVALx1, JMPx1, LOADKx9, MOVEx6, RETURNx3, SETGLOBALx1, SETTABLEx3, TESTx1` |
| `735` | `CALLx17, CLOSUREx2, FORLOOPx2, FORPREPx2, GETGLOBALx21, GETTABLEx15, GETUPVALx1, JMPx3, LENx1, LOADBOOLx2, LOADKx13, MOVEx9, RETURNx3, SETGLOBALx1, SETTABLEx3, SUBx2, TESTx3` |
| `736` | `CALLx32, CLOSUREx2, EQx1, GETGLOBALx57, GETTABLEx44, JMPx2, LOADKx9, MOVEx21, NEWTABLEx1, RETURNx5, SETGLOBALx1, SETLISTx1, SETTABLEx17, TESTx1` |

## What this inventory does not establish

- `ActivateHordeSpawn` appears once, in chunk `62`, after the boss-marker
  callback. The bytecode supplies neither pool selection nor native behavior.
- No linked chunk names or invokes a `SpawnPoint_DirectorHorde` marker.
- The obelisk ability chunks call native drop selectors, but do not identify a
  concrete loot item or random-selection policy.
- `is_server_only=false` on all 19 linked events. Group/source provenance
  distinguishes the two client tutorial jobs, but content linkage alone is
  not execution authorization for any chunk.
