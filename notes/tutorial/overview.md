# Tutorial and first-session onboarding

This note reconstructs Game build `5.3.0.103`'s tutorial and the
first-session flow that follows it. It distinguishes the playable tutorial
mission from the longer ship-room onboarding sequence.

## Fresh full-route replay observations (2026-07-17)

The first authored-start replay after restoring the normal spawn exposed several
presentation and ordering gaps that focused snapshots had hidden:

- Blitz had no visible player beam-in effect on initial entry. The squad UI
  initially showed Blitz in all three slots; the field-22 fix below now leaves
  exactly one visible Blitz portrait.
- The route appeared to contain no enemies before the Ride lesson. Ride later
  produced a post-cast frame glitch, and its HUD teaching arrow disappeared
  before the first use rather than on the first accepted cast.
- Voltic Slash did not show its 0.4-second cooldown. Holding the attack produced
  two left-arm swings followed by a frozen pose instead of alternating arms.
- Spawn effects worked for the two observed post-Ride groups, but pursuing mobs
  moved and dealt authoritative damage without visible movement/attack
  animation. They retained a downward-facing orientation instead of turning
  toward Blitz, and neither damage direction produced floating hit text.
- Enemy kills after the opening clear did not visibly award XP. darkspin currently
  sends a one-time cumulative jump to `101` when the first encounter stage
  completes; replace this with per-kill cumulative updates after converting the
  XP-bar percentages recovered from the tutorial video.
- Placed health and mana bars are movement pickups, not click interactables.
  Walking through them updated resources, but no blue/green consume effect was
  visible. Clicking must not collect them.
- The client presented the first loot-obelisk prompt even though the server had
  not created object `15`. The trace shows the previous clear spawned objects
  `7`-`10`; the player advanced without killing them, so the current server
  predicate withheld the obelisk. Client location prompts and server encounter
  progression can therefore diverge. Trigger/gate each encounter at its
  authored perimeter rather than spawning a follow-up group behind the player.
- The authored tutorial introduces a new enemy type with a cutscene/camera focus
  on that enemy. Recover this event and its relationship to the follow-up group
  before changing the deterministic encounter list.
- The boss teleporter still rendered twice and did not activate. The setup code
  creates object `28` once, then layers an inactive `ServerEvent`; unlock sends
  an active event. Determine whether the noun or event owns the retail visual.
  In this run activation was also blocked upstream because the absent obelisk
  prevented the loot/Sage prerequisites from completing.
- A second full-route smoke pass confirmed that movement commands currently
  advance server authority directly to the requested goal. Remote left/right
  clicks therefore damage enemies without range validation and a long movement
  request can collect a green capsule before Blitz visibly arrives. Enemy
  movement uses the same goal-only presentation, causing newly spawned mobs to
  stack at a shared point. Replace goal assignment with simulation-time motion
  before treating swept trigger checks as authoritative contact.
- Ordinary enemy corpses retained collision and could be nudged. Their death
  behavior must disable physics/navigation collision at the accepted death
  boundary while retaining the object through its authored fade/delete
  timeline.
- Voltic Slash remained on its initial animation frame before the delayed
  swing and temporarily immobilized Blitz. Ride's teaching arrow survived the
  accepted use. These are separate animation/lesson cleanup ownership gaps.
- On a cold DarkSpinner launch that also generated `content.db`, Play became
  available before the game server listener was bound and the first client
  login failed; restarting DarkSpinner succeeded. Desktop logs were also not
  consistently persisted beneath the runtime log directory. Listener
  readiness and file-backed desktop logging are launcher prerequisites for the
  next clean replay.

The client calls the persisted field `new_player_progress`. darkspin calls the
same domain and SQLite value `onboarding_progress` and retains the legacy name
only in the HTTP/XML protocol. These values do not represent campaign
completion; permanent campaign gating uses `chain_progression`.

## Evidence labels

| Label | Meaning |
| --- | --- |
| **Binary-confirmed** | Recovered from build 103 control flow, strings, or client callbacks. |
| **Asset-confirmed** | Present in extracted server data, but not necessarily tied to a recovered runtime order. |
| **Live-confirmed** | Observed in a traced build-103 launch. |
| **Video-observed** | Visible in surviving gameplay footage at the cited timestamp; the recording's exact client build and server-side mutation remain unknown. |
| **Inferred** | Best interpretation of combined evidence; requires a clean trace. |
| **Unknown** | The client behavior or server contract has not been recovered. |

## High-level flow

```text
login
  -> boot movies
  -> spaceship scene and ship tour
  -> START / navigation ready
  -> tutorial start for progress 0, 1000, or 2000
  -> Tutorial game mode (1)
  -> Game_Tutorial_cryos_1_v2
  -> TutorialGameMsgs 0xC8/subtype 0 supplies cumulative XP
  -> positive XP sets in-memory progress 3000; nonpositive resets it to 2000
  -> collection room
  -> editor room and SP_Editor
  -> map room
  -> planet screen / first campaign selection
```

The tutorial is pending while progress is below `3000`. Values from `3000`
onward skip the playable tutorial, but first-session onboarding continues
through collection, editor, map-room, and planet-screen steps toward `9000`.
For a clean end-to-end tutorial test, begin at `0`; directly placing an account
at `1000` or `2000` enters a mid-onboarding checkpoint without proving that
the earlier initialization steps ran.

## Recovered onboarding values

The transition column describes a client write from the old value to the new
value. A row does not prove that every intermediate UI action has been
identified.

| Transition / stored value | Known client behavior | Scene or UI | Confidence and gaps |
| --- | --- | --- | --- |
| `0 -> 1000` | Initializes the first spaceship/room state through `sub_52D230`. | `SP_SporeLabs/cSpaceshipState` | **Binary-confirmed**; exact visible prompt and write timing are unknown. |
| `1000 -> 2000` | Advances an early first-run step through `sub_516F50`. | Spaceship/tutorial launch path | **Binary-confirmed**; the user action or event that separates `1000` and `2000` is unknown. |
| `0`, `1000`, or `2000` | Ordinary ship-room selection is disabled. `MapRoomUI.StartGame` constructs game mode `1` (`Tutorial`). | Tutorial launch | **Binary-confirmed**. Entering Arsenal room `1` instead leaves onboarding in an invalid state. |
| tutorial completion -> `3000` | `TutorialGameMsgs` (`0xC8`), subtype `0`, supplies a signed 32-bit cumulative XP total. A positive value writes XP, derives the account level, and writes `3000`; a nonpositive value clears XP/level-like fields and writes `2000`. | End of `Game_Tutorial_cryos_1_v2`; collection-room arrival is expected to follow the return action | **Binary-confirmed in-memory contract**. Framing, reliability, send timing, server persistence, and the later account refresh remain unknown. |
| around `3000` | Emits `NewPlayerArrivedCollectionRoom`. | Collection/Arsenal squad presentation | **Binary-confirmed** event; exact mapping between the named collection room and visible Arsenal UI needs a live capture. |
| `3000 -> 4000` | Runs the recovered collection-room path through `sub_41A570`. | Collection room | **Binary-confirmed** transition; exact prompt/reward mutation is unknown. |
| `4000 -> 5000` | Runs the editor-room transition through `sub_419510`; nearby flow references `NewPlayerUnlockedEditorRoom`. | Collection to editor-room onboarding | **Binary-confirmed** transition and event family; exact event-to-value boundary needs a trace. |
| `5000 -> 6000` | Launches `SP_Editor` through `sub_4C08A0`. | Hero Editor | **Binary-confirmed**. Returning from the editor re-enters the spaceship flow. |
| `6000 -> 6500` | Emits `NewPlayerUnlockedMapRoom` through the room controller at `sub_52D230`. | Map-room unlock | **Binary-confirmed**. |
| `6500 -> 6800` | Performs planet/map initialization through `sub_527810`. | Map room / planet setup | **Binary-confirmed**; visible screen and triggering action are unknown. |
| `6800 -> 8000` | Performs a second planet/map initialization path through `sub_527810`. | Map room / planet setup | **Binary-confirmed**; distinction from `6500 -> 6800` is unknown. |
| `8000 -> 9000` | Enters the first progressed planet-screen/campaign path, including `sub_527DD0`. | Planet screen / first campaign selection | **Binary-confirmed** path; exact account-level predicate and completion event need a live trace. |
| `9000` | No later onboarding write has been recovered in the current pass. | Normal ship/map flow | **Inferred** terminal onboarding state, not yet proven to be the maximum accepted value. |

## Playable tutorial mission

The native tutorial request uses:

| Property | Value | Evidence |
| --- | --- | --- |
| Game mode | `1` (`Tutorial`) | **Binary-confirmed** and represented by darkspin `game.ModeTutorial`. |
| Level | `Game_Tutorial_cryos_1_v2` | **Binary-confirmed** client selection and current server level table. |
| Selected difficulty | `0` | Current darkspin compatibility behavior for tutorial game creation. |
| Planet/theme | Cryos | **Asset-confirmed** by level and rendering configuration. |
| Music | `music_set_tutorial` | **Asset-confirmed**. |

The extracted level asset is named
`Game_Tutorial_cryos_1.Level.xml`; the executable/server-facing level
identifier includes the `_v2` suffix. The level uses a Cryos rendering and
planet configuration, its own nav/physics meshes, and 16 marker-set layers.

### First-run enemies and encounter objects

The level's `firstTimeConfig` contains:

- `TutorialBasicDiseased.Noun`
- `TutorialBasicRanged.Noun`
- `TutorialBasicPoison.Noun`
- `TutorialSloth.Noun`
- `TutorialSpecialOne.Noun`

The ordinary level configuration only names `TutorialBasicPoison.Noun`, which
strongly suggests the authored first visit differs from later/replay behavior.
Direct extraction of `level.source_payload` through `darkrun db level bget`
confirms the build-103 director record layout. The ordinary Poison entry and
all five `firstTimeConfig` entries carry minimum difficulty `1`, maximum
difficulty `100`, and `isHordeLegal=1`. There is no per-species weight in these
16-byte records. The decoded level payload is retained as
`bin/game/logs/tutorial-cryos-level-config.bin` (SHA-256
`f97e3491d780e20514f0703e83cc5e6da5ebda0f07831c0eefaad2afab7ba10a`).
The AI marker layer also contains poison enemies with and without orb drops,
ranged and diseased enemies, two `TutorialSpecialOne` placements, and a
`TutorialSpecialOne_Intro` placement.

The design layer contains horde and boss director spawn points, health and
boss obelisks, a teleporter, and a boss-security teleporter. The separate
`Game_Tutorial_cryos_1_boss_arena_layer` asset is empty and is not one of
the level's 16 referenced marker sets; the arena logic is authored in the
design layer. The boss director uses `waveOverride=4` and publishes
`horde triggered02` to two registered horde spawners. This strongly supports
four arena waves at two spawn loci, although the exact director semantics
remain an inference until its runtime consumer is recovered. Build-103 native
classification assigns `SpawnPoint_DirectorHorde.Noun` type `5` and
`SpawnPoint_DirectorBoss.Noun` type `9`. Type `9` enters a special spatial
registry whose radius is derived from the marker collision shape. Types `5`
and `9` are both excluded from the ordinary active-spawn count. The two nouns
are therefore director control objects, not enemy nouns.

The AI marker set has 27 fixed placements: two `TutorialSpecialOne`, one
`TutorialSpecialOne_Intro`, six ranged, three diseased, four poison orb-drop,
and eleven poison no-orb enemies. There is no fixed Sloth placement despite
Sloth appearing in `firstTimeConfig`, so it is likely director-spawned or an
unused legal choice. Ten fixed smart objects provide five health and five
power orbs. The poison and no-orb variants share noun asset ID `0x5A5BAA25`
but use loot drop types `2` (`Orb`) and `1` (`None`) respectively, confirming
that the split is intentional rather than an extractor naming artifact.

### Scripted route and authority boundaries

The authored route exposes five native/Lua callback zones. Coordinates are
world-space XYZ; `serverOnly` is the asset's authority flag.

| Lesson/event | Marker ID | Position | Radius | Once | serverOnly | Callback |
| --- | ---: | --- | ---: | --- | --- | --- |
| Overdrive activation | `1359119906` | `(-349.459, -223.142, 10.088)` | `30` | false | true | `nTutorial_IntroOverdriveActivate.main` |
| Second ability | `1410140146` | `(232.287, -93.752, 10.526)` | `5` | true | true | `nTutorial_IntroAbilitySecond.main` |
| Health and power | `3802061458` | `(68.630, -38.059, 19.974)` | `15` | false | false | `nTutorial_IntroHealthAndPower.main` |
| Abilities lesson | `32670756` | `(233.674, -96.281, 10.150)` | `5` | true | false | `nTutorial_IntroAbilities.main` |
| Sage/second creature | `3912233898` | `(259.346, 81.391, 25.088)` | `25` | false | true | `nTutorial_IntroSecondCreatureUnlock.main` |

The two ability markers are only about three world units apart, which explains
their paired lesson timing while preserving separate client-visible and
server-authoritative effects. The Sage trigger is effectively colocated with
the `TutorialSpecialOne_Intro` enemy and the resistance/Sage audio cluster,
matching the footage's unlock sequence.

The callback bodies have since been recovered from `ServerData.package` as
standard Lua 5.1 bytecode and mapped source. Group `0x24F78AA1` contains eight
server-authority tutorial jobs; group `0xA35BED24` contains eight client jobs.
The group split and each marker's `serverOnly` flag are an authority boundary,
not merely an extraction detail: client jobs directly invoke `nUIManager`,
while the server jobs request player/simulation changes.

### Recovered tutorial Lua inventory

All job-style chunks require `Lua!GlobalDefinitions.lua`, `Lua!Vector.lua`, and
`Lua!TargetUtils.lua`, resolve `LuaJobObject.Noun`, create a transient object,
and run their body through `nThread.CreateThreadForObject`. This scaffolding is
retail lifetime management. A Go director can model the resulting scoped job
without exposing object creation as unrestricted script authority.

The server-authority group contains:

| Script | Confirmed authored operations | Reconstruction consequence |
| --- | --- | --- |
| `Tutorial_IntroAbilitySecondUnlock` | wait `1.0s`; enumerate players; call `UnlockNextAbility` twice per player; delete job | Go validates the lesson phase, advances the session ability count, and emits the build-103 player update. Lua supplies the one-second delay and two-step count. |
| `Tutorial_IntroOverdriveActivate` | wait `3.75s`; enumerate players; execute an empty numeric loop; delete job | Original bytecode proves the loop body is genuinely empty. This authored server job performs no overdrive mutation, packet, or event. |
| `Tutorial_IntroSecondCreatureUnlock` | wait `0.5s`; enumerate players; `UnlockSecondCreature`; create/notify event with `clientEventID = SPID("PlayerUnlockedSecondCreature")`; delete job | Hero availability is a Go squad mutation. The half-second delay and notification are authored presentation sequencing. |
| `Tutorial_CatalystUnlock` | if all players are initially unbeaten: waits `2,4s`, conditionally unlocks crystals and drops crystals per player, then waits `4,4,4s`; always waits `2s` and calls `ActivateHordeSpawn` | Exact bytecode and native path recovered. `DropCrystals` makes one attempt per simulator player; `sub_9CA3D0` is a thunk to the weighted crystal-selection, pickup-create, and launch initializer at `sub_A18600`. Successful pickup assigns the first available mission crystal slot, deletes the pickup, emits logical message `68` / wire `0xC3`, and recomputes bonuses; no account submission is visible. The trailing horde native is a literal no-op in build 103. Route membership and exact pickup-create/launch replication remain open. |
| `Tutorial_OverdriveUnlock` | if all players are initially unbeaten: waits `3,5s`, unlocks overdrive per still-unbeaten player, then waits `5,4,4s`; always waits `3s` and calls `ActivateHordeSpawn` | `UnlockOverdrive` clears lock byte `+4972` and fills meter `+4668` to the configured maximum. The trailing horde native is a literal no-op in build 103, so actual horde ownership must be elsewhere. |
| `Tutorial_RandomUnlock` | if all players are initially unbeaten: waits `2,4s`, calls `UnlockNextAbility` per still-unbeaten player, then waits `6,5s`; always waits `2s` | Exact bytecode schedule recovered; route membership and paired client presentation remain open. |
| `Tutorial_SoloSupportUnlock` | if all players are initially unbeaten: waits `2,4s`, calls `UnlockNextAbility`, then waits `2,5s`; always waits `2s` and calls `ActivateHordeSpawn` | Exact bytecode schedule recovered. The trailing native terminates in a literal no-op stub in build 103; route membership and the separate owner of any following horde remain open. |
| `Tutorial_SupportUnlock` | Same recovered control flow as Random Unlock: conditional waits `2,4,6,5s` around one ability unlock per still-unbeaten player, then an unconditional `2s` wait | Exact bytecode schedule recovered; route membership remains open. |

#### Post-`FirstAggro_SpecialOne` bytecode parity ledger

The following ledger comes from fresh `darkrun db lua_chunk get` lookups and
`darkrun db server_data bget decoded_payload ... --decode zlib` extraction from
the runtime `content.db`. Opcode counts cover the top-level prototype and every
nested prototype. The **VM-unsupported** column preserves the compiler snapshot
when this ledger was first decoded; it does not mean invalid Lua 5.1. The
compiler has since added `GETUPVAL`, `LEN`, the numeric arithmetic family,
`FORPREP`, `FORLOOP`, `NEWTABLE`, `LT`, and `SETLIST`. In the current workspace,
chunks `141`, `930`, `215`, `735`, `623`, `62`, `769`, `222`, `655`, `199`,
`426`, and `134` have no unsupported opcodes from their recorded inventories.

| Chunk | Indexed group/instance; registration | SHA-256 | Exact opcode inventory | VM-unsupported |
| ---: | --- | --- | --- | --- |
| `141` | `0x24F78AA1/0x724391F1`; `nTutorial_IntroOverdriveActivate.main` | `909a2ff8a3494f6b9bcfd26461cbe72538d8161778771cc8bc5e11792fcf1ffa` | `CALL×10, CLOSURE×2, FORLOOP×1, FORPREP×1, GETGLOBAL×14, GETTABLE×8, GETUPVAL×1, LEN×1, LOADK×7, MOVE×4, RETURN×3, SETGLOBAL×1, SETTABLE×3, SUB×1` | `FORLOOP×1, FORPREP×1, GETUPVAL×1, LEN×1, SUB×1` |
| `930` | `0x24F78AA1/0xBA0E5C06`; `nTutorial_IntroAbilitySecond.main` | `07c3a923a272fc2313dd00876044398597fd1107e833037cd0e2cf2b7e57ec64` | `CALL×12, CLOSURE×2, FORLOOP×1, FORPREP×1, GETGLOBAL×16, GETTABLE×10, GETUPVAL×1, LEN×1, LOADK×7, MOVE×6, RETURN×3, SETGLOBAL×1, SETTABLE×3, SUB×1` | `FORLOOP×1, FORPREP×1, GETUPVAL×1, LEN×1, SUB×1` |
| `215` | `0x24F78AA1/0xE6FA7F3F`; `nTutorial_IntroSecondCreatureUnlock.main` | `e98b1e855e64904689451bd79635c3487a7a20e29f1b943885c9648b61f8e30a` | `CALL×13, CLOSURE×2, FORLOOP×1, FORPREP×1, GETGLOBAL×17, GETTABLE×11, GETUPVAL×1, LEN×1, LOADK×8, MOVE×6, NEWTABLE×1, RETURN×3, SETGLOBAL×1, SETTABLE×4, SUB×1` | `FORLOOP×1, FORPREP×1, GETUPVAL×1, LEN×1, NEWTABLE×1, SUB×1` |
| `735` | `0x24F78AA1/0x5C4C5BBF`; `nTutorial_RandomUnlock.main` | `b9bf494972e98b73e60fe527bbfcaec6b66434abea4545adacbdb343c6364878` | `CALL×17, CLOSURE×2, FORLOOP×2, FORPREP×2, GETGLOBAL×21, GETTABLE×15, GETUPVAL×1, JMP×3, LEN×1, LOADBOOL×2, LOADK×13, MOVE×9, RETURN×3, SETGLOBAL×1, SETTABLE×3, SUB×2, TEST×3` | `FORLOOP×2, FORPREP×2, GETUPVAL×1, LEN×1, SUB×2` |
| `623` | `0x24F78AA1/0xC9B6860B`; `nTutorial_SupportUnlock.main` | `2079148a63af96dd765857ae51298bc86a65a9a9ae4d7a0ca8145e204a400509` | `CALL×17, CLOSURE×2, FORLOOP×2, FORPREP×2, GETGLOBAL×21, GETTABLE×15, GETUPVAL×1, JMP×3, LEN×1, LOADBOOL×2, LOADK×13, MOVE×9, RETURN×3, SETGLOBAL×1, SETTABLE×3, SUB×2, TEST×3` | `FORLOOP×2, FORPREP×2, GETUPVAL×1, LEN×1, SUB×2` |
| `62` | `0x24F78AA1/0x90CE5ECC`; `nTutorial_SoloSupportUnlock.main` | `b67c0aace5d108e5f6f7b2432f5d8be1aaee3f00d1204fdd56c6b03abcd878c4` | `CALL×19, CLOSURE×2, FORLOOP×2, FORPREP×2, GETGLOBAL×23, GETTABLE×17, GETUPVAL×1, JMP×3, LEN×1, LOADBOOL×2, LOADK×13, MOVE×13, RETURN×3, SETGLOBAL×1, SETTABLE×3, SUB×2, TEST×3` | `FORLOOP×2, FORPREP×2, GETUPVAL×1, LEN×1, SUB×2` |
| `769` | `0x24F78AA1/0xE137EF5C`; `nTutorial_OverdriveUnlock.main` | `cbec31d39922671c12c67caa2482bfaa139d795d4dd4a96266b5ad235bd2fef4` | `CALL×20, CLOSURE×2, FORLOOP×2, FORPREP×2, GETGLOBAL×24, GETTABLE×18, GETUPVAL×1, JMP×3, LEN×1, LOADBOOL×2, LOADK×14, MOVE×13, RETURN×3, SETGLOBAL×1, SETTABLE×3, SUB×2, TEST×3` | `FORLOOP×2, FORPREP×2, GETUPVAL×1, LEN×1, SUB×2` |
| `222` | `0x24F78AA1/0xB8716CFB`; `nTutorial_CatalystUnlock.main` | `13f084e59a192f55a30e8285e8ebb651bfcaa90ae0ab16f5235c3ffa28fa09db` | `CALL×21, CLOSURE×2, FORLOOP×2, FORPREP×2, GETGLOBAL×25, GETTABLE×19, GETUPVAL×1, JMP×4, LEN×1, LOADBOOL×2, LOADK×14, MOVE×14, RETURN×3, SETGLOBAL×1, SETTABLE×3, SUB×2, TEST×4` | `FORLOOP×2, FORPREP×2, GETUPVAL×1, LEN×1, SUB×2` |
| `655` | `Modifiers/0x9C2B314A`; global `nModifier_Spawn`, registered as `SpawnModifier`, callbacks `Activate/Tick/Deactivate` | `0f1ff2f4dfaf35491d91b31c894061af3e3cc293fbb05f97ae656e75ef8c47f1` | `CALL×12, CLOSURE×3, GETGLOBAL×35, GETTABLE×22, LOADK×4, MOVE×6, NEWTABLE×1, RETURN×4, SETGLOBAL×1, SETTABLE×12` | `NEWTABLE×1` |
| `134` | `Abilities/0xD71BDB7E.lua`; global `nAbility_PickUpCrystal`, registered as `PickUpCrystal` | `372fbd8cdfd921f6fd0979cbbabedb3d64a3bfa2c99501e266015cf0d5d4dbc8` | `CALL×9, CLOSURE×2, GETGLOBAL×27, GETTABLE×10, LOADK×18, MOVE×3, NEWTABLE×3, RETURN×3, SETGLOBAL×1, SETLIST×3, SETTABLE×15` | `SETLIST×3` |

**Closure and job ownership (bytecode-proven).** Each of chunks `141`, `930`,
`215`, `735`, `623`, `62`, `769`, and `222` has three prototypes. The worker
has no upvalues. The exported `main` has one upvalue and obtains the worker from
the top-level `CLOSURE` through its following Lua 5.1 binding descriptor
(`MOVE R0 R0`); `main` later executes `GETUPVAL` to pass that worker to
`CreateThreadForObject`. A parity VM must consume the binding descriptor as
part of `CLOSURE`, not execute it as an ordinary `MOVE`. `main` creates
`LuaJobObject.Noun`. Chunks `141`, `930`, and `215` call
`CreateThreadForObject(job, worker, job)`; chunks `735`, `623`, `62`, `769`,
and `222` additionally forward both marker arguments. The first three
eventually call `MarkForDelete(job)`. The other five return without marking the
job. IDA now proves the native object/thread association
and teardown cancellation described below. It does not turn worker return into
object deletion, so a typed director must keep the five non-deleting jobs
phase-scoped. Modifier chunk `655` has four prototypes and no upvalues; its
three closures are installed directly as modifier callbacks.

**Authority classification.** Chunks `141`, `930`, `215`, `735`, `623`, `62`,
`769`, and `222` are server job scripts: their waits and presentation pairing
do not make their player fields, event, object lifecycle, unlock, or drop calls
client-owned. Chunk `141` is only server timing/job lifecycle because its loop
has no body. Chunk `655` is a server modifier whose effect and animation calls
drive replicated presentation; the client does not own modifier attachment.
Chunk `134` is the server-executed interaction ability that requests crystal
collection; client input may select the ability and target, but does not own
the resulting slot mutation, pickup deletion, or rejection event.

**Chunk `134` pickup caller (content/bytecode-proven).** A fresh runtime-DB
extraction of resource `13659`, `Abilities/0xD71BDB7E.lua`, is retained as
`bin/game/logs/pickup-crystal-13659.luac`; its SHA-256 is
`372fbd8cdfd921f6fd0979cbbabedb3d64a3bfa2c99501e266015cf0d5d4dbc8`.
The module registers `PickUpCrystal` with `requiresAgent=true`, cooldown and
mana cost `0`, `noGlobalCooldown=true`, descriptor
`nDescriptors.IsInteract`, animation `pickup_catalyst`, cast time `0.3s`,
animation time `0.7s`, release time `1s`, and five authored range entries all
equal to `2`. Its two no-upvalue closures are installed at numeric ability
callback slots `2` and `3`. Slot `2` has no arguments, waits, loops, or
branches. It resolves `agentID=nAbility.GetAgentID()`,
`playerID=nPlayer.GetPlayerIdForObject(agentID)`, calls and discards
`nAbility.GetAgentAttributeSnapshot()`, resolves
`targetID=nAbility.GetTargetID()`, and calls
`nPlayer.PickupCrystal(playerID,targetID)`. Slot `3` immediately returns. No
`lua_module_alias` or `lua_dependency` rows exist for this chunk;
runtime globals `Class`, `nAbility`, `nPlayer`, `nBit`, and `nDescriptors`
therefore remain part of the native bootstrap boundary. Recovered test chunk
`705` names `0x3681d755!GlobalDefinitions.lua`; the raw
`lua_dependency.target_lua_chunk_id` is null, but the package/linker audit and
successful definition compile map that exact authored import to chunk `659`,
resource `14227`, `lua/0x2E64AA9E.lua`. The helper body is present; only the
native runtime surfaces it consumes remain a parity boundary. Typed authority
must require an accepted interaction ability, an
agent owned by the resolved player, a live target with `cLootData`, and the
authored range `2`.

**Chunk `134` request validation (client-native-proven; server parity
required).** `nAbility.RequestAbility` enters `sub_9E1540`, which calls the
single generic admission routine `sub_9E0660` before creating the ability
instance. That routine rejects an unresolved ability, ranks outside `1..8`, a
locked player ability, invalid target state for target-requiring descriptors,
blocking agent attributes, cooldown, insufficient mana, range failure,
`IsAbleToHit` failure, and blocking modifiers. Its range predicate
`sub_9DE710` selects the authored rank range (`ability + 268 + 4*rank`), falls
back to rank one where applicable, applies agent modifiers, and measures the
agent-to-target distance minus both object radii; if no target resolves it
checks the supplied target position instead. For chunk `134`, every authored
rank entry is `2`. The callback does not call `IsAbilityInRange` or
`IsAbilityAbleToHit`, and a complete xref audit finds no second invocation of
either generic predicate in the ability-instance creation/execution path.
Consequently range/hit admission is proven at request acceptance, not at the
later Lua callback. If the selected target disappears before the callback,
`PickupCrystal` simply fails its object resolution and makes no mutation.
Server parity should preserve that no-op release race while still binding the
accepted request to the requesting player's controlled agent. Diagnostic
`bin/game/logs/ida-pickup-ability-validation.log`, SHA-256
`b59e5cb03f728198cbb36b907952a7176406669f70b5911ad37816c045a1418a`,
retains the native call and predicate audit.

Chunk `134` also sets `shouldPursue=true`. When the generic range predicate
fails, `sub_9E0660` returns result `3` for this pursuit-enabled ability instead
of the ordinary out-of-range error `-9996`; it does not create the ability
instance until validation later returns `1`. This distinguishes an
out-of-range interaction request from an accepted cast and prevents a typed
director from collecting the crystal immediately merely because pursuit was
requested. The exact locomotion/resubmission owner remains in the missing
client action/bootstrap layer.

**Client crystal selection (content-proven presentation/input only).** Runtime
chunk `705`, `LuaTestScripts/0x89404DF4.lua`, resource `14276`, SHA-256
`7a1625acaf4229acfbea77fef0e5e6bd6be8bb0afd2b9357866731c21e4c26a7`,
is automated load-test code, not a level or authoritative server script. Its
`interestingThings` array gives `kType_Crystal` a separate category from
creatures, health/mana orbs, loot, destructible ornaments, and generic
interactables. In its in-game action body, bytecode checks
`ClosestCrystal != kObjIDNone`, calls client-side
`nPlayer.IsCrystalSlotAvailable()`, and only then calls
`nAction.PickUpCrystal(ClosestCrystal)`. This proves the intended client target
class and a local capacity prefilter. It does not prove a server target
allowlist, a wire command shape, or permission to trust the client-provided
object ID. The retained fresh extraction is
`bin/game/logs/pickup-crystal-test-14276.luac`.

**Action-command wire boundary (client-native-proven).** Build 103's sole
client action sender at `sub_5370F0` allocates logical message `29` / wire
`0x9c` with exactly `104 + sub_A17440(type)` bits, copies a fixed `40`-byte
common header, then copies the type-specific tail. It transmits reliable
ordered using literal arguments `1,3,0`. The common header is byte `type`,
three unknown bytes, a `uint32` input sync stamp, a `uint32` actor object ID,
position XYZ, and orientation XYZW. Before serialization it replaces the actor
ID at common offset `8` with that object's network/runtime ID. For types `7`,
`8`, `9`, and `11`, `sub_536D50` likewise replaces the first tail `uint32` with
the selected object's network/runtime ID, or zero if resolution fails.

The type-`9` constructor `sub_44CAF0` instruction-links this command to the
crystal path. It requires a clicked target with object component pointer
`+744` and a non-null noun-definition field at `+148` through `sub_9D9770`;
the generic interactable fallback constructs type `11`. Reflection strings
strongly support interpreting noun field `+148` as `CrystalDef`/`crystalDef`,
but its exact structure name is not instruction-linked. The exact type-`9`
tail is target local object ID at `+0`, target current object position XYZ at
`+4/+8/+12`, signed byte `-1` at `+16`, and three uninitialized stack-padding
bytes at `+17..+19`. It produces a `60`-byte body, and the sender normalizes
only the first word. The current decoder's final `Rank uint32` interpretation
is therefore false: only byte `+16` is meaningful, and a deterministic parity
encoder must write `ff 00 00 00` rather than reproduce native stack garbage.
Character/squad ability types `7/8` instead use the separately proven
`44`-byte target/cursor/target-position/index/rank/unknown/user-data tail and
an `84`-byte body. This disproves encoding the crystal interaction as an
ordinary character-ability request.

**Pickup input and missing server consumer (client-native-proven; authority
inferred).** Client click predicate `sub_44A7C0` accepts the component path only
when component float `+64 <= 0`. This is exactly the DNA-loot amount:
`nObject.IsDNALoot` binding `sub_A03B40` tests the same field as `> 0`, and the
component decoder reports it as `LABS_local_ADD_DNA_VALUE`. The gate therefore
excludes DNA loot but remains a local input filter, not authorization. The
following type-`9` constructor's noun-definition predicate supplies the
crystal-specific classification. `sub_4DFB40` stages exactly the supplied
20-byte tail, and `sub_4DF5B0` emits it once at action start; the release path
does not send a second type-`9` command. Incoming `0x9c` handler `sub_9C0F70`
reflection-decodes the 40-byte header plus the type-sized tail and appends a
108-byte `(peer/player ID, action)` record to the simulator vector at
`+0x7528`. An executable displacement audit found this vector only in that
append path, its constructor, and its destructor; no client-side consumer or
type-`9` execution switch is recovered. This supports, but does not
instruction-prove, that the original authoritative consumer was server-only
and is absent from `Game.exe`.

Typed Go intent should accept only the target ID from the authenticated peer,
derive the actor from that peer's deployed agent, ignore client XYZ and selector
byte for authority, resolve current target state, and validate phase ownership,
live crystal/cLoot identity, request-time range, and `IsAbleToHit` before
scheduling chunk `134`'s release. The original server action-to-ability mapping
and rejection response remain unresolved.

**Crystal component precondition (native-proven; enforcement owner
unresolved).** `PickupCrystal` checks only that the player and target object
resolve. On the successful-slot branch, `sub_A17F40` resolves the target again
and immediately reads `*(target + 744) + 8` to obtain the 16-bit crystal level;
there is no null/component guard before that dereference. On the full-slot
branch it may instead apply lob state to the supplied object. Thus the retail
native assumes an upstream `cLootData`/crystal-object invariant but the
recovered generic ability admission routine does not establish it. A typed
authoritative command must require a live phase-owned crystal pickup with the
expected `cLootData` component before either branch, rejecting arbitrary world
objects without mutation. Whether the original server enforced that at packet
decode, action dispatch, or noun construction remains unresolved.

**Exact authoritative bodies and evidence.** Argument and result shapes below
are Lua-visible shapes; native callbacks return zero results unless stated.

| Chunk | Bytecode-proven calls, waits, loops, and mutations | Native/live evidence and typed director validation |
| ---: | --- | --- |
| `141` | Worker `(job)`: `WaitForXSeconds(3.75)`, `players=GetPlayerIds()`, empty numeric loop `0..#players-1`, `MarkForDelete(job)`. | **Native-proven:** no overdrive call exists. **Typed intent:** accepted server-job scope plus a cancellable `3.75s` wait and terminal job deletion only. **Route:** exact Cryos owner corrected below; do not infer overdrive behavior from the module name. |
| `930` | Worker `(job)`: wait `1s`; loop `0..#GetPlayerIds()-1`; call `UnlockNextAbility(index)` twice; delete job. | **Native-proven:** `sub_A05120(number)->()` resolves the player slot, increments reflected `mLockedAbilityMin` (`int32 +4976`, field `21`), and rewrites results above `5` to `9`. **Live-observed:** the accepted Cryos marker advances boundary `1→2→3` and exposes Ride. **Typed validation:** correct phase/trigger, live player slot, expected pre-boundary, independently owned jobs, ordered two-step replication. The authored once flag is not enforced because `main` returns no boolean; exact boundary below. |
| `215` | Worker `(job)`: wait `0.5s`; call `UnlockSecondCreature(index)` for every `0..#players-1`; create `{clientEventID=SPID("PlayerUnlockedSecondCreature")}`; `nEvent.Notify(event)`; delete job. | **Native-proven:** `sub_A050C0(number)->()` increments player `int32 +4984` only while it is `0` or `1`; the exact reflected field is `mDeckScore`, index `23`, not an authored field named creature count. `sub_A11560(table)->()` reflection-decodes `ServerEvent 0x8619ff24` and sends it only when non-default; the sender is logical `28`/wire `0x9b`. **Typed validation:** valid hidden second-creature backing data for every target; mutate all accepted players before broadcasting the event; mission-local unlock must not imply persistence. Exact combined callback ownership remains unresolved. |
| `735` | Worker `(job, callbackArg0, callbackArg1)`: resolve but do not otherwise use `GetPlayerIdForObject(callbackArg1)`; obtain player count; initial loop computes `areAllUnbeaten`. If true, wait `2s`, then `4s`; recheck each slot and call `UnlockNextAbility` only for still-unbeaten slots; wait `6s`, then `5s`. Always wait `2s`; return without deleting job. | **Bytecode/native-proven; exact non-Cryos owner corrected below.** Its trigger first requires a continuous native `2.0s` dwell, so the earliest unlock is entry `+8s`, not `+6s`. Worker code is instruction-for-instruction identical to chunk `623`; only module identity and native trigger timing differ. Snapshot/validate contiguous player slots, recheck beaten state at mutation time, and give the object-associated thread explicit level-instance cancellation. |
| `623` | Same worker instructions as `735`: conditional waits `2,4,6,5s`, one `UnlockNextAbility` per still-unbeaten slot, unconditional `2s`, no delete. | **Bytecode/native-proven; exact non-Cryos owner corrected below.** Its native activation delay is zero, so an eligible callback starts immediately. Module identity is `nTutorial_SupportUnlock`, but bytecode does not encode support-slot semantics. Do not infer three support grants from the name or paired client UI. |
| `62` | Same initial unbeaten scans; if all unbeaten, wait `2s`, `4s`, unlock once per still-unbeaten slot, wait `2s`, `5s`; always wait `2s`; resolve `controlled=GetPlayerControlledObjectID(GetPlayerIdForObject(callbackArg1))`; call `ActivateHordeSpawn(callbackArg0, controlled)`; no delete. | **Native-proven:** `GetPlayerIdForObject(object)->number`, `GetPlayerControlledObjectID(number)->objectID`, and `ActivateHordeSpawn(object, ignored)->()`; the latter reads only argument 1 and reaches literal no-op `sub_A23560`. The listener supplies `(sourceObjectID, listenerObjectID, otherObjectID)`, of which Lua retains the first two. **Typed validation:** preserve the exact event ID, source, owning boss-spawn listener, discarded third object, and phase lifetime; no horde intent may be synthesized from the no-op. |
| `769` | Initial unbeaten scans; if all unbeaten, wait `3s`, `5s`; call `UnlockOverdrive(index)` for each still-unbeaten slot; wait `5s`, `4s`, `4s`; always wait `3s`; resolve controlled object as above; call the no-op horde native; no delete. | **Native-proven:** `sub_A05180(number)->()` clears lock byte `+4972` and fills meter `+4668` with the configured maximum. **Typed validation:** use the exact non-Cryos owner corrected below, player and controlled-object ownership, unbeaten recheck, idempotent transition, and phase cancellation; do not attach horde progression to the no-op tail. |
| `222` | Initial unbeaten scans; if all unbeaten, wait `2s`, `4s`; for every slot, redundantly retest the still-true outer flag, recheck beaten state, call `UnlockCrystals(index)` only if still unbeaten, then call `DropCrystals(index)` regardless of that per-slot recheck; wait `4s` three times. Always wait `2s`; resolve controlled object; call the no-op horde native; no delete. | **Native-proven:** `UnlockCrystals(number)->()` clears lock byte `+4973`. `DropCrystals(number)->()` requires a live player and controlled object, then attempts one weighted drop per simulator player around that controlled object. **Typed validation:** use the exact non-Cryos owner corrected below, live slot/object, crystal eligibility, difficulty/RNG ownership, phase-scoped pickups, and replication; preserve the authored unconditional per-slot drop inside the initially-all-unbeaten branch. |
| `655` | Module initialization sets unique/recall metadata, preloads `generic_spawn.ServerEventDef` as `SpawnModifier`, and stores animation `horde_beam_in`, duration `0.5`. `Activate()`: get agent, stop locomotion, add `Immobilized=1`. `Tick()`: add effect, set animation, wait `0.5s`. `Deactivate()`: remove effect and reset animation; it does **not** explicitly remove the immobilization attribute. | **Bytecode/native-proven server-authoritative modifier with replicated presentation; attachment owner inferred only.** Native modifier teardown runs `Deactivate` first and then removes every still-registered scoped attribute handle, including this immobilization handle. Typed validation requires a live phase-owned spawned NPC, one modifier instance, effect/animation allowlists, cancellable tick wait, and guaranteed engine-equivalent teardown. |

**Current `content.db` route-link correction (content-index-proven; supersedes
the older “unlinked” route labels in the table above).** The completed marker
decoder now links eight of the nine prioritized chunks. Only modifier chunk
`655` remains without a `level_script` row.

| Chunk | Exact normalized owner | Authority and validation consequence |
| ---: | --- | --- |
| `141` | Cryos level `9`; server-only radius-`30` enter event `6084`; marker row `17733`, authored ID `1359119906`, `TriggerZone.Noun-2` at `(-349.4592,-223.14235,10.088047)`; not once-only. | This is the exact owner of the empty-loop `3.75s` job. Preserve authored repeatable entry while keeping every created job independently cancellable; no overdrive mutation may be invented from the module name. |
| `930` | Cryos level `9`; server-only, authored once-only radius-`5` enter event `6085`; marker row `17734`, authored ID `1410140146`, at `(232.2869,-93.751595,10.526188)`. | Exact second-ability owner. Validate trigger and phase before the ordered `1s` wait and two boundary increments. Native proof below shows this no-return Lua callback never sets the runtime once latch, so a parity model must not infer enforced one-shot behavior from the content flag. |
| `215` | Cryos level `9`; server-only radius-`25` enter event `6098`; marker row `17742`, authored ID `3912233898`; not once-only. | Exact Sage-unlock owner. Re-entry must not create concurrent half-second jobs or duplicate the event; this link still does not own Quadra aggro. |
| `735` | Level `56`, `zelems_1`; non-server-only, authored once-only radius-`20` enter event `32147`; marker row `126382`, authored ID `164537526`, `TriggerZone.Noun-3` at `(648.4210,-10.617704,14.521069)`. | `Tutorial_RandomUnlock` is campaign-level content, not Cryos. Preserve its unbeaten recheck and level-instance-owned persistent thread; do not schedule it in the tutorial route. Its no-return `main` does not set the native once latch. |
| `623` | Level `60`, `zelems_3`; non-server-only, authored once-only radius-`20` enter event `33371`; marker row `131724`, authored ID `2750774706`, displayed as `BossSpawnScriptClient.Noun-1` but using `TriggerZone.Noun`, at `(687.50525,-1602.73035,0.033192)`. | `Tutorial_SupportUnlock` is not Cryos. Its name does not prove a support-count mutation beyond the one bytecode-proven boundary increment per eligible player, and its no-return `main` does not set the native once latch. |
| `62` | Level `56`, `zelems_1`; marker row `126389`, authored ID `223774364`, ordinal `7` in marker set `1793`; `SpawnPoint_DirectorBoss.Noun-876848874` at `(948.3736,674.0089,0.087999)`. Raw pair: event `boss triggered` -> `nTutorial_SoloSupportUnlock.main`. | This is an object-lifetime event-listener callback, not a radius-zero contact trigger. The generic native listener tuple is proven below; the concrete publisher and third object remain unresolved. The trailing `ActivateHordeSpawn` remains a native no-op and authorizes no horde intent. |
| `769` | Level `50`, `verdanth_1`; marker row `99848`, authored ID `409195075`, ordinal `7` in marker set `1513`; `SpawnPoint_DirectorBoss.Noun-166917082` at `(87.55838,65.14283,-0.09692)`. Raw pair: event `boss triggered` -> `nTutorial_OverdriveUnlock.main`. | `Tutorial_OverdriveUnlock` is not Cryos. Preserve director-event ownership and do not attach horde behavior to its no-op tail. |
| `222` | Level `23`, `nocturna_4`; marker row `68601`, authored ID `2147048951`, ordinal `31` in marker set `827`; `SpawnPoint_DirectorBoss.Noun` at `(-370.07767,-387.91943,10.034988)`. Raw pair: event `lvl 4 boss arena` -> `nTutorial_CatalystUnlock.main`. | `Tutorial_CatalystUnlock` is not Cryos crystal ownership. Preserve the conditional unlock and unconditional per-slot drop inside the initially-all-unbeaten branch, with director-event activation rather than an invented contact radius. |

**Raw event-link correction for chunks `62`, `769`, and `222`
(content-proven).** Their normalized `event_kind=triggerVolume`,
`event_slot=luaCallbackOnEnter`, and radius `0` fields are decoder
misclassifications. Each owning `SpawnPoint_DirectorBoss.Noun` record instead
contains a same-record event string immediately before its Lua callback. Chunk
`62` and chunk `769` use `boss triggered`; chunk `222` uses
`lvl 4 boss arena`. The pairs do not occupy the recovered trigger-volume
layout and are not stored in a `luaCallbackOnEnter` slot. Their `main` functions
receive two arguments and forward both into their jobs. The generic native
listener path below now types those as object IDs, while the concrete event
publisher and the discarded third object remain unresolved. The defensible
intent is `directorEvent(eventName, sourceObjectID, listenerObjectID,
otherObjectID, phaseGeneration)`, not `triggerEntered`.

Each event also has same-payload `SpawnPoint_DirectorHorde.Noun` records paired
with `HordeSpawner_Register`: four markers share level `56`'s
`boss triggered`, three share level `50`'s `boss triggered`, and five share
level `23`'s `lvl 4 boss arena`. This is content-link proof of a shared
director-event fanout, not proof of delivery order or spawn timing. The
tutorial callback and horde registrations are sibling authored consumers of
the same event. They are not consequences of the tutorial worker's later
`ActivateHordeSpawn` call; that Lua binding still reaches the separately
proven native no-op.

Focused ASCII inventories account for every textual event occurrence in each
payload: marker set `1793` has five `boss triggered` strings for four horde
listeners plus the tutorial listener; set `1513` has four for three plus one;
set `827` has six `lvl 4 boss arena` strings for five plus one. There is no
additional same-payload textual publisher record. This excludes an omitted
publisher beside these listeners, but not an external marker set, hashed
server state, or unrecovered director code.

The exact source payloads are marker set `1793`, SHA-256
`8562a2f52110359ca3d98dfec8b37248f0e3de39c14b04de39c60dd2b790a217`;
marker set `1513`, SHA-256
`00d0cfb6eaabbc365964cdf7c377cafa6c078b332c70226fc89dd82840f30308`;
and marker set `827`, SHA-256
`de395df43d3b5afa13b6d35cca1c301ffadd2dc4f17e43926711c718d3f85a3a`.

**Native event-listener contract (IDA/native-proven).** Reflection metadata
defines `EventListenerData` as an exact `0x28`-byte record with `event` at
`+0x0c`, native `callback` at `+0x1c`, and `luaCallback` at `+0x24`.
`EventListenerDef` is an array of those records, and `SpawnPointDef` embeds its
`eventListenerDef` at `+0x08`. This is the exact schema occupied by the raw
event/callback pairs above. It is distinct from `TriggerVolumeEvents`, whose
separate `0x20`-byte reflected layout contains only `onEnterEvent` at `+0x0c`
and `onExitEvent` at `+0x1c`.

When an object with an event-listener definition is activated, `sub_A1D190`
allocates its listener, fixes the registered context to that object, and
`sub_9C2030` registers every 40-byte entry in the simulator's event-ID map.
Teardown `sub_A1D290 -> sub_9BF490` removes all of that listener's entries.
Dispatch `sub_9BF5A0 -> sub_A1D0F0` invokes the resolved native callback first,
if present, then invokes the Lua callback with exactly three object IDs in
this order: `(sourceObjectID, listenerObjectID, otherObjectID)`. Chunks `62`,
`769`, and `222` declare only two fixed `main` parameters, so Lua deliberately
discards `otherObjectID`; their persistent jobs retain the source and the
owning `SpawnPoint_DirectorBoss` object. Consequently their later
`GetPlayerIdForObject(callbackArg1)` is applied to that listener owner, not to
the discarded entrant/target argument. The subsequent horde call still cannot
gain behavior from this lookup because `ActivateHordeSpawn` ignores argument
two and reaches its proven no-op.

**Complete client publisher inventory (IDA/native/content-index-proven).** IDA
has exactly six direct code xrefs to `sub_9BF5A0` in build 103:

| Publisher | Event source and delivered objects | Boss-route consequence |
| --- | --- | --- |
| Ability/interaction start, `sub_9E0AE0` | Reads the interaction target's `startInteractEvent`; dispatches `(targetObjectID, listenerObjectID, actingAgentID)`. | Fixed by the target definition, not an arbitrary director-event name. |
| Ability/interaction teardown, `sub_9E09D0` | Reads the interaction target's `endInteractEvent`; dispatches the same target/actor shape. | Fixed interaction lifecycle; it does not publish merely because a boss-spawn listener exists. |
| Lua `nEvent.NotifyObjects`, `sub_A057B0` | Hashes argument 1 as the event SPID and dispatches Lua arguments 2 and 3 as `(sourceObjectID, otherObjectID)`. | This is the only shipped arbitrary-name Lua publisher. Exact `content.db` lookups find zero `NotifyObjects`, `boss triggered`, or `lvl 4 boss arena` constants across all 1,029 recovered chunks. |
| Lua `nEvent.NotifyOptionalInteractibleEvent`, `sub_A05810` | Reads argument 1's `optionalInteractEvent` and dispatches `(argument1ObjectID, argument2ObjectID)`. | The corpus has one method-name user, chunk `318`, but neither boss event name; this path cannot establish either named route. |
| Trigger-volume enter/exit/stay, two branches in `sub_A15FB0` | Uses the authored trigger event and dispatches `(triggerOwnerObjectID, entrantObjectID)`. | The focused payload inventories above contain no additional textual record that could author either boss event on a trigger. |

Reflection metadata independently fixes `startInteractEvent`,
`endInteractEvent`, and `optionalInteractEvent` at definition offsets
`+0x20`, `+0x30`, and `+0x40`; their compact runtime event IDs are read at
component offsets `+24`, `+28`, and `+32`. The complete direct-xref inventory
therefore excludes a hidden client-native director publisher for these names.
It does not exclude an indirect call in missing server code. Diagnostics are
`bin/game/logs/ida-event-publisher-xrefs.log`, SHA-256
`06e726fe71899b5bda0765db089f8bd0a1905fa2baeee7d844437996f4ba22b3`,
and `bin/game/logs/ida-object-event-fields.log`, SHA-256
`81d2c29d57e7b6b1e849d5edf79b9bcfbbde6e22eaef180e1bb27d9d440b0278`.

The publisher that calls `sub_9BF5A0` for each named boss event, the semantic
role of `otherObjectID`, same-event consumer ordering, delivery multiplicity,
and the missing server native resolver for `HordeSpawner_Register` remain
server-director homework. The listener lifetime and registered-context
identity are no longer unresolved. IDA diagnostics are
`bin/game/logs/ida-event-listener-schema.log` and
`bin/game/logs/ida-event-listener-strings.log`, with respective SHA-256 hashes
`868f1fb2addbcf4c804819e445f546698be6d744173ba9ce2370de7503577cc6`
and `eb04ed25094e6cb78246170509cfe5850c515c940bb807f61c9fc1328d8449fe`.
Do not use the normalized zero radius as evidence for programmatic
trigger-volume activation.

These links remove chunks `735`, `623`, `62`, `769`, and `222` from the
post-Quadra Cryos homework list. They remain useful cross-level bytecode parity
fixtures, but importing their mutations or no-op horde tails into level `9`
would now contradict content evidence.

Chunk `141` is spatially part of the recovered post-Quadra handoff even though
it has no mutation. The teleporter's authored destination
`(-347.57553,-224.60829,10.08803)` is exactly `2.386878` units from event
`6084`'s radius-`30` center, so an accepted teleport places the entrant inside
this server-only volume. **Content/native-proven:** after its enter dispatch,
chunk `141` waits `3.75s`, performs an empty player-count loop, marks its job
object for deletion, and returns. **Inferred boundary:** the physics/contact
update that notices the teleported entrant may occur after the teleport's
authoritative position change; no recovered instruction fixes it to the same
simulation update. Because the event is not once-only, later genuine exits and
re-entries can author another job. This chunk cannot own the arena's first
wave, overdrive unlock, or a client event: none of those calls exists in its
bytecode.

The recovered Lua corpus also supplies no separate Cryos presentation owner
for that delay. **Content-index-proven negative:** no string constant, module
alias, or level event names `nTutorial_IntroOverdriveActivateClient`. The
complete `PlayOverdriveBlink` constant inventory has only two chunks. Chunk
`14`, resource `13533`, `0xA35BED24/0x40449A11.lua`, SHA-256
`d274bc0c751d36bfe4c5c0ed2b2f83fcd8cbbc53142f676454c7e636a0b54180`,
is `nTutorial_OverdriveUnlockClient.main` and links only to levels `50` and
`53`; chunk `79`, resource `13602`, `PopupTip/0x9F55CF26.lua`, SHA-256
`be37a79c43a654595c469ead2543788671439a7052f2743094d723c5bf41eb70`,
has no `level_script` row. This excludes packaged overdrive-blink Lua as a
level-9 companion. It remains a bounded content negative: unrecovered client
code or data-driven UI may still react independently, but chunk `141` itself
authorizes only its job lifetime.

**Chunk `222` crystal tuning correction (content/native-proven).** The runtime
`LabsTuning/0x3B01D7F6.prop` payload is retained at
`bin/game/logs/labs-tuning.bin`, SHA-256
`b57ca3826416ecc6f23db0ad92b9d6bdce2faa376809a5505bb2869a0014bcce`.
Its integer property `0x35820747` is `4`, overriding the executable fallback
`2` used by `dword_1164B94`. `sub_A0BB10` selects
`max(currentDifficulty, 4)`, not a cap, before calling `sub_A18600`.
`sub_A184E0` rejects values below that same minimum; the tutorial's chance
argument becomes `1000 * 0.15 = 150`, while `sub_9BCE90(100)` is proven by
`sub_AE5480` to return an integer in `[0,100)`. Each attempted tutorial drop
therefore passes the probability gate; remaining failure points are player and
controlled-object resolution, weighted subtype availability, object creation,
and the required pickup component. Typed parity must use effective difficulty
`max(difficulty, 4)` and must not model this property as a maximum or a 2% gate.

**Chunk `222` create/launch ordering (native-proven; wire unresolved).**
`sub_A179E0` creates the selected pickup at the source object's current
position through the ordinary `sub_9D6A30 -> sub_9D1DF0` object path. Object
construction calls `sub_9D1810`, seeding reflection baselines for both
`cLocomotionData` (`0xB6F447EF`) and the pickup component
(`0x2ADB076A`), before returning. Only afterward does `sub_A179E0` write the
selected 16-bit subtype to pickup-component offset `+8` and call
`sub_A2EF00(pickup, destination, 2.5, 0.5, 0, false, 0, 0, false)`.
`sub_A2EF00` changes authoritative locomotion/trajectory state and object state
byte `+97` to `4`; it constructs no logical GMS message and invokes no packet
sender. Therefore both subtype and launch are post-baseline reflected changes.
The build-103 `ObjectCreate` reflection section decodes only type
`0x258cf09d`, `sporelabsObject`; its trailing loop accepts only fixed
`(uint8,uint32,uint32,vec3)` records and never dispatches either component
reflection. Consequently `cLootData` and `cLocomotionData` cannot be folded
into `ObjectCreate`: they require their separate `0x9a` and `0x94` application
messages after object creation. Their relative order and exact locomotion field
selection remain an authoritative-sender gap, not a dedicated crystal-launch
message.

**Crystal pickup component reflection (IDA/native-proven; publication shape
not sender-proven).** Case-insensitive FNV-1 identifies component hash
`0x2adb076a` as `cLootData`. Its ten registered fields are, in order:
`crystalLevel` (`int32`, `+0x08`), `mLootItem.id` (`uint64`, `+0x10`),
`mLootItem.rigblockAsset` (`+0x18`), `mLootItem.suffixAssetId` (`+0x1c`),
`mLootItem.prefixAssetId1` (`+0x20`), `mLootItem.prefixAssetId2` (`+0x24`),
`mLootItem.itemLevel` (`+0x28`), `mLootItem.rarity` (`+0x2c`),
`mLootInstanceId` (`uint64`, `+0x38`), and `mDNAAmount` (`float`, `+0x40`).
Thus `sub_A179E0`'s post-baseline write at component `+8` changes exactly
field `0`, `crystalLevel`, to the selected crystal subtype as an `int32` (the
selector supplies it through a 16-bit member, but the reflected field is not a
16-bit wire type). Client dispatch case `27` is
`kGmsLootDataUpdate`, wire `0x9a`; `sub_539D10` reads an object ID, resolves
that object's `cLootData`, and reflection-decodes the remainder. A minimal
receiver-valid subtype delta is therefore
`9a <objectID:u32> 00 <crystalLevel:i32> ff`. This proves a legal update vocabulary, not that the
retail authoritative sender's batching, reliability, or ordering. The separate
component update itself is required because `ObjectCreate` cannot decode
`cLootData`.

**Crystal lob state (IDA/native-proven; outbound field selection unresolved).**
`sub_A2EF00` sets locomotion field `0` `lobStartTime` from the authoritative
simulation clock, resets field `1` `lobPrevSpeedModifier` to zero, and fills
field `2` `lobParams`; it also sets object movement type field `19` to `4` and
recomputes object orientation. `cLobParams` has nine reflected fields:
`planeDirLinearParam` (`+0x48`), `upLinearParam` (`+0x4c`),
`upQuadraticParam` (`+0x50`), `lobUpDir` (`+0x18`), `planeDir` (`+0x3c`),
`bounceNum` (`+0x30`), `bounceRestitution` (`+0x34`),
`groundCollisionOnly` (`+0x38`), and `stopBounceOnCreatures` (`+0x39`). For
the crystal call, `lobUpDir=(0,0,1)`, bounce count/restitution and both booleans
are zero, and `planeDir` is the normalized start-to-destination displacement
projected perpendicular to `lobUpDir`. Let `L` be that projected distance,
`U` the signed displacement along `lobUpDir`, and `H=2.5+max(U,0)`. The native
call's `0.5` argument is duration: active-lob detection compares elapsed
simulation time against exactly `0.5 * 1000` milliseconds. The initializer
stores `planeDirLinearParam=L/0.5` and the unreflected height control `H`, then
`sub_A2E3E0` derives the three reflected coefficients. For nondegenerate `L`,
its apex distance is `X=L/2` when `U` is approximately zero, otherwise
`X=(H-sqrt(H*H-U*H))/U*L`; it writes
`upLinearParam=2*H/X` and `upQuadraticParam=-H/(X*X)`. These values plus
`planeDirLinearParam` define the client trajectory. Wire `0x94` can decode
this locomotion reflection, and a separate `0x94` is required because
`ObjectCreate` cannot decode `cLocomotionData`. The authoritative sender's
exact changed-field selection and ordering relative to the required `0x9a`
remain unproven. Diagnostic `bin/game/logs/ida-pickup-reflection.log`, SHA-256
`f9f9f7835ccfc7d2b77bad8d6cfda7c1c6c95ce932722bab42dfd77d77edaa77`,
records the registrations and receiver boundary.

The receiver-valid Go codec now publishes the three fields dirtied directly by
the initializer: field `0` as the `uint64` simulation-millisecond start, field
`1` as zero `float32`, and field `2` as the registered 84-byte `cLobParams`
memory image. The generic reflection receiver copies a nested field's complete
registered size, so that image includes the unreflected start/destination,
height, duration, and padding as well as the nine reflected members; this is the
same rule already proven for the 60-byte `cProjectileParams` image. The complete
application packet is 105 bytes including wire ID `0x94`. The codec deliberately
omits field `17` because the initializer does not dirty `reflectedLastUpdate`.
This closes the legal receiver shape, not the absent sender's ordering,
reliability, or its publication of object movement type `4` and recomputed
orientation.

**Missing generic component sender (IDA-negative evidence).** A complete IDA
xref audit of the sole protocol message factory `sub_A8FC00` found no
construction call for logical `21` (`kGmsLocomotionDataUpdate`, wire `0x94`)
or logical `27` (`kGmsLootDataUpdate`, wire `0x9a`). Each decoder wrapper has
one call site: `sub_539900 -> sub_A1DBF0` for locomotion and
`sub_539D10 -> sub_A1DEC0` for loot. The apparent logical-`21` constructor in
an initial scan was rejected after exact argument recovery: `0x15` was the
payload length of logical message `37`. Thus this client-shaped executable
proves both required receive formats but does not contain the authoritative
generic replication constructors. Exact field selection, reliability, and
`0x94`/`0x9a` ordering require an original-server capture or server-binary
source; they cannot be recovered honestly from another `Game.c` search.
Diagnostic `bin/game/logs/ida-crystal-sender-search.log`, SHA-256
`37af65db63cf235b3847f12f911d912789ba3ea202ba9a52b3ca0607b0c1db83`,
retains the corrected enumeration.

**Crystal collection and full-slot rejection (native-proven).** Registered
`PickupCrystal(player,pickup)` resolves both arguments and silently performs no
mutation if either is invalid. Fresh chunk `134` proves its caller is the
registered `IsInteract` ability with authored range `2`; generic native ability
admission checks range and hit eligibility once at request acceptance. The Lua
callback does not recheck either predicate at release. A typed authoritative
caller must therefore validate request ownership and the initial range/hit
contract, retain the selected object ID through the cast, and allow a missing
release-time target to become the same silent no-op. When one of the nine
eligible mission slots is free, `sub_A17F40` copies the pickup noun and 16-bit
`crystalLevel` into that slot, deletes the world pickup, sends the acquired
`0xc3` crystal message, and recomputes bonuses in that order.

When no slot is free, the pickup is retained. `sub_A04B40` defines an active
lob as movement type `4` with elapsed simulation time below the stored lob
duration. If the current lob has finished, the native relaunches the pickup
toward its own current position with height `3.0`, duration `0.5s`, zero
bounce count/restitution, and `groundCollisionOnly=true`; an attempt during the
active half-second does not restart it. Both full-slot branches send client
event `0x6ea4091e` through the same `ServerEvent` sender with selector byte
`~(1 << playerIndex)`. The client has a dedicated branch that presents the
event for `6000ms` using localization key `0xc0a38338`; the adjacent client
feedback key array begins with that same key repeated three times. Its authored
symbolic event name and localized text are not recovered: a focused
`darkrun db localization_text get locale_key=...` lookup returns no row for
that key in any of the five installed locale projections. The downstream
selector-byte interpretation also remains unnamed. Repeated rejected
contacts can therefore present the event repeatedly and can relaunch only at
least `0.5s` apart. No explicit delete or natural-expiry write occurs in this
binding; any noun lifetime and phase teardown remain separate owners.

The typed full-slot adapter preserves that selector as transport recipient
scope rather than adding it to the reflected event. Its event body is the
seven-byte `9b 0f 1e 09 a4 6e ff` application packet. The zero-displacement
native relaunch retains the initializer's exact degenerate-path values: zero
plane direction, plane speed `1`, height `3`, duration `0.5`, upward linear
coefficient `12`, quadratic coefficient `-12`, and
`groundCollisionOnly=true`. Applying the relaunch advances the pickup's lob end
to `now+0.5s`; subsequent contacts during that interval emit only the feedback
event. The adapter returns event and locomotion publications separately because
the client-shaped executable still cannot prove their datagram ordering.
The plane-speed fallback is literal float bytes `00 00 80 3f` at executable VA
`0x00fcd3f8`; the zero direction global at `0x015ba0b0` lies in the zero-filled
tail of `.data`. `sub_A2E3E0` then takes its nonzero-height/degenerate-distance
branch and derives `4*3` and `-4*3` for the two upward coefficients.

**Crystal slot packet identity (IDA/native-proven; receiver still
unresolved).** The build-103 protocol-name table entry at `0x011843c0` pairs
`kGmsCrystalMessage` with logical ID `0x44` (`68`); its immediate neighbors are
logical `0x43` `kGmsCrystalDragMessage` and logical `0x45`
`kGmsKillRacePlayerMsgs`. `sub_A8FAC0` iterates all 78 `(name, logical ID)`
entries to construct the transport maps, and `sub_A204D0` requests logical
`68`, proving the packet family's client-authored name. With the recovered
transport ordering this is wire `0xc3` (`CrystalMessage`). The IDA diagnostics
are `bin/game/logs/ida-gms-name-range.log`, SHA-256
`4451df5eaa5cce3ec7b31a4f89a7d285c2c76a8746b16798ee9012032e95ddf0`, and
`bin/game/logs/ida-crystal-name-pointer.log`, SHA-256
`4d81938dde901923402498ab0210df8c07f045d73c5087d2cff2f2c4ef18b92a`.
Native sender `sub_A204D0` obtains offset `+13` from `sub_9D9870`, which reads
the noun's color property and returns only `0..4`; it is not the catalyst's
stat identity. The table names the packet but does not expose its concrete
inbound handler;
the sender's uninitialized/reserved body bytes require the consumer audit below.

**`kGmsCrystalMessage` consumer (client-native-proven).** Client dispatcher
`sub_53ADC0` maps logical `68` to `sub_53A910`. That receiver reads exactly 29
bytes and appends them unchanged to the global HUD queue; it performs no slot
mutation synchronously. HUD update `sub_4235F0` drains the queue in 29-byte
steps and then sets its end pointer equal to its begin pointer. Its exact
subtypes are:

| Byte `0` | Fields consumed | Deferred client behavior |
| ---: | --- | --- |
| `0` | slot `int32 +1`; crystal noun `uint32 +5`; five-way crystal color `int32 +13` | Resolve the noun, mark the slot occupied, call `setSlotContents(slot, color, rarity+1, slotList)`, and emit telemetry `LABS_CATALYST_PICKUP` with the color and authored rarity. |
| `2` | source slot `int32 +21`; destination slot `int32 +25` | Call `moveAccepted(true)`. If source is below `9` and destination is below `9` or exactly `-1`, clear the source; occupy a real destination, or emit `LABS_CATALYST_DROPPED` for `-1`. |
| `3` | no payload field after the subtype affects behavior | Call `moveAccepted(false)` and leave local slot occupancy unchanged. |

Any other nonzero subtype is drained without a HUD action. Server sender
`sub_A204D0`'s acquisition body initializes subtype `0`, slot `+1`, noun `+5`,
type `+13`, and selected subtype `+17`; the client does not read `+17`.
Offsets `+9..+12` and `+21..+28` are uninitialized stack bytes in that native
sender and are also ignored by this consumer. `sub_A205A0` fully initializes
the subtype-`2`/`3` response body and packs source/destination at `+21/+25`.
A typed server fixture must zero every ignored byte rather than reproduce
undefined stack contents, while preserving the consumed offsets and deferred
queue semantics. This closes the packet/HUD consumer gap; mission teardown and
late-join reconstruction of the nine crystal slots remain unresolved.

**Player unlock reflection (IDA/native-proven).** The top-level Labs player
reflection type `0x15ff16e2` registers the relevant fields in this exact order:

| Field | Authored name | Player offset | Tutorial writer |
| ---: | --- | ---: | --- |
| `10` | `mEnergyPoints` | `+0x123c` / `+4668` | `UnlockOverdrive` fills it to the configured maximum. |
| `19` | `mbLockedOverdrive` | `+0x136c` / `+4972` | `UnlockOverdrive` writes `false`. |
| `20` | `mbLockedCrystals` | `+0x136d` / `+4973` | `UnlockCrystals` writes `false`. |
| `21` | `mLockedAbilityMin` | `+0x1370` / `+4976` | `UnlockNextAbility` advances it. |
| `22` | `mLockedDeckIndexMin` | `+0x1374` / `+4980` | Not written by the examined unlock natives. |
| `23` | `mDeckScore` | `+0x1378` / `+4984` | `UnlockSecondCreature` advances `0` to `1` to `2`; the native use supplies the second-creature consequence, but the reflected name is not `CreatureCount`. |

`sub_53ADC0 -> sub_539E00 -> sub_A1E070` proves that wire `0xa1`
(`kGmsLabsPlayerUpdate`) reflection-decodes this top-level type. Consequently
the exact minimal crystal-unlock payload after the packet ID is
`slot, 00 10, 14, 00, ff`: player slot, top-level mask `0x1000`, field `20`,
boolean false, terminator. The minimal overdrive transition is
`slot, 00 10, 0a, <float32 maximum>, 13, 00, ff`. Ability and deck-score
updates use encoded field bytes `0x15` and `0x17` respectively followed by their little-
endian `uint32` values and `ff`; this matches the already live-proven ability
and Sage update shapes.

The client constructor initializes both lock bytes false, its copy operation
preserves both bytes, and the examined build has no local write that restores
either byte to true. A locked tutorial start therefore requires an
authoritative initial player reflection with fields `19`/`20` true; unlock is
mission-player state, with no account submission in these native paths. A new
mission must reconstruct locks from server policy rather than relying on the
previous player object. Diagnostic `bin/game/logs/ida-player-unlock-fields.log`
has SHA-256
`d80adbbcce2259433384f6c432195df91feb253faeac27c66cc3f01e2bb0394`.

**Scoped attribute lifetime (native-proven).** `AddAttributeModifier`
`sub_A04000` stores the target object ID and returned handle in the first free
slot of a 32-entry cleanup array when Lua is executing in an ability/modifier
context. An explicit `RemoveAttributeModifier` (`sub_A04110`) both removes the
attribute and clears the matching cleanup entry through `sub_9DDD90`. Modifier
termination calls the Lua deactivate callback through `sub_9E09D0` first,
then `sub_9DDD40` iterates all 32 entries and removes every handle that remains. A 33rd
simultaneous scoped handle is still returned by the attribute container but is
not registered because the cleanup array is full; none of these tutorial
modifiers approaches that limit. Therefore chunks `199`, `349`, and `655` must
model immobilization/intangibility as modifier-scoped state: preserve the
authored empty or presentation-only deactivate body, then perform guaranteed
native-equivalent cleanup after it. Cancellation must run the same ordering.

**`SpawnModifier` attachment and release boundary (content-negative and
native-proven).** The authoritative DB identifies chunk `655` as resource
`14222`, `Modifiers/0x9C2B314A.lua`, and its case-insensitive FNV-1 registration
GUID is `SPID("SpawnModifier") == 0xd2a08fed`. A focused
`level_script lua_chunk_id=655` query returns no rows. Across the 1,029 recovered
Lua chunks, the only `SpawnModifier`, `horde_beam_in`, or `generic_spawn`
references are in chunk `655` itself. `Game.c` contains neither the
registration string nor that GUID, and the focused decoded Cryos level, marker,
guard, horde-gate, and boss-security evidence contains neither the GUID nor the
chunk instance `0x9c2b314a`. These are negative linkage results, not proof that
the modifier is unused: its attaching command remains an undocumented
server/external owner. They do not identify a particular route marker, wave, or
enemy as that owner.

The native request path separates modifier authority from its presentation
thread. It first emits packed logical message `36` / wire `0xa2`
`ModifierCreated` (37 bytes), then allocates and prepares the callback-2 `Tick`
coroutine, and finally invokes callback 1 `Activate` synchronously (after an
optional request callback). Preparing the coroutine pushes its closure and
argument and records it as runnable; it does not resume Lua. Consequently the
authoritative `Stop` and scoped `Immobilized=1` mutation happen before the
scheduler first runs `Tick`, which adds the effect/animation and begins the
`0.5s` wait. **Native-proven:** normal Tick completion only removes the Lua
thread from the runnable queue. It does not invoke `Deactivate`, release the
attribute, or emit deletion, and neither `animTime=0.5` nor `recallTime=0`
proves a modifier lifetime.

Explicit removal follows `sub_9E12D0 -> sub_9E11F0`: it invokes callback 3
`Deactivate`, performs the 32-slot scoped-handle cleanup, removes the instance
from its owner's modifier collection, and then emits packed logical message
`38` / wire `0xa4` `ModifierDeleted` (target object plus instance ID, eight
bytes). Logical message `37` / wire `0xa3` is the separate 21-byte modifier
update path and is not a deletion substitute. A typed spawn-modifier intent
therefore needs the phase-owned target and source object IDs, exact modifier
GUID and unique instance ID, validated effect/animation assets, and an explicit
cancel/release owner. Release must be idempotent and preserve Deactivate ->
scoped cleanup -> deletion replication ordering. The retail condition and time
that choose that release remain unresolved; the bytecode's half-second wait
must not be promoted into an inferred deletion timer.

**Native collection semantics (native-proven).** `GetPlayerIds()->table<number,
number>` is `sub_A04980`. It traverses the simulator player hash map at
`+33468` into a fixed 32-byte native index buffer, skips entries whose player
state has bit `0x40` set, resolves every emitted key again through
`sub_9BE1B0`, omits failed resolutions, and returns a one-based Lua table. Each
table value is the resolved player's reflected
`uint8 mPlayerIndex` at `+4560` (`+0x11d0`), not the adjacent `uint64
mPlayerOnlineId` at `+4568`. The hash-map traversal does not establish stable
ordering. `GetPlayerIdForObject(object)->number` is a distinct value: it
requires the object's player link at `+716` and returns the object's byte
`mPlayerIdx` at `+85`.

The eight jobs use only `#table` and deliberately pass contiguous numeric
indices `0..count-1` to `HasBeatenThisLevel`, unlock, and drop bindings; they
never index the returned table. Native disconnect `sub_9BF360` erases exactly
the supplied player-index key and does not compact or renumber the remaining
map. Native connect `sub_5362D0` passes the session callback's byte index
unchanged to player creation `sub_9C2340`, which stores it as `mPlayerIndex` and
uses it as the map key. Therefore the bytecode assumes an external invariant
that active tutorial player indices form a dense prefix beginning at zero;
`GetPlayerIds` does not enforce that invariant. A typed director must validate
that density before executing these authored loops (and reject/cancel on a
hole), rather than silently interpreting `count` as arbitrary returned IDs.
Whether the original authoritative session allocator always preserved density
across disconnect/reconnect remains unresolved. Reflection evidence is retained
in `bin/game/logs/ida-player-id-fields.log` (SHA-256
`68093486d87dd828c234a74c09e01d77331468e8e182419aaceefd2b01848dfe`) and
`bin/game/logs/ida-player-runtime-fields.log` (SHA-256
`bf4d45b07b33ebada151289fe04ee9f9dee515b472e9724c2384446578bb2195`).

`HasBeatenThisLevel(number)->boolean` (`sub_A054C0`) returns false for an
unresolvable slot. Otherwise it compares the reflected `uint32
mChainProgression` at `+4700` / `+0x125c` with the current simulator difficulty
and returns `mChainProgression >= difficulty`. This is a progression comparison,
not a stored boolean, and explains why the jobs snapshot all-player eligibility
before waiting but recheck each player immediately before mutation. The
reflection listing is retained in
`bin/game/logs/ida-player-level-field.log` (SHA-256
`79eeb57916e5e24d8d13f2a50d0f07a242bc9f4b88e45259cdeb1c3906ff9baa`).
`WaitForXSeconds(number)->()` clamps negative input to
zero, returns immediately for nonpositive input, and otherwise installs a
continuation/deadline on the current Lua thread. `MarkForDelete(object)->()`
sets object byte `+93`; it is an authoritative lifecycle mutation, not an
immediate delete packet. `CreateObject(noun)->objectID` accepts the one-argument
job-object form with default transform fields.

**Object-associated thread ownership (IDA/native-proven).** Direct IDA
disassembly recovers the body omitted as weak `sub_A00050` in `Game.c`.
The retained listing is `bin/game/logs/a00050-disassembly.txt`, SHA-256
`cef72c0ed9429bf03ecab47a9582b456f5ff1cf29e34486e746ecd2c1ac635a0`.
`CreateThreadForObject(object, worker, ...)->()` resolves argument `1`, requires
argument `2` to be a Lua function (Lua type `6`), and returns no Lua values. It
is a no-op for an invalid object, a non-function worker, or an object whose
single thread-ID slot at object offset `+676` is already nonzero. Otherwise it
allocates a Lua thread through the global thread manager, stores that thread's
numeric ID at object `+676`, copies the worker and every trailing Lua argument
into the new state, and performs the initial resume. `sub_8F0FB0` records
internal status `0` for a Lua yield, `-1` for normal completion, and `-2` for
an error; `CreateThreadForObject` submits the record to the thread manager for
either non-error status and skips that submission only on `-2`.
The copied call shape differs between the two authored families. Chunks `141`,
`930`, and `215` define `main(callbackArg0)` but do not forward that argument:
their exact call is `CreateThreadForObject(job, worker, job)`, so the worker
receives only `worker(job)`. Chunks `735`, `623`, `62`, `769`, and `222` define
`main(callbackArg0, callbackArg1)` and call
`CreateThreadForObject(job, worker, job, callbackArg0, callbackArg1)`, so their
worker receives all three arguments. This distinction matters for typed
fixtures: the Cryos jobs cannot observe, retain, or validate their initiating
entrant after `main` creates the job, while the five campaign jobs retain both
callback arguments in the Lua thread state across every wait.

None of the wrappers checks the result of `CreateObject`. If allocation returns
`kObjIDNone`, the next `CreateThreadForObject` resolves an invalid owner and is
a native no-op, after which `main` returns without a mutation, notification, or
retry. This is a bytecode/native-proven failure boundary. Chunks `141` and
`215` are not once-only, but their failure still must not execute the worker
without an owning object/thread lifetime.

**Native trigger admission and timer boundary.** IDA reflection metadata gives
the complete relevant definition layout: `timeToActivate=+72` (float),
`persistentTimer=+76` (byte), `triggerOnceOnly=+77`,
`triggerIfNotBeaten=+78`, `triggerActivationType=+80` (int32),
`luaCallbackOnEnter/Exit/Stay=+84/+88/+92`, dimensions `+96..+116`, and
`serverOnly=+120`. `sub_A16F40` copies the timer to runtime `+48`, persistence
to `+52`, unbeaten filtering to `+61`, and activation type to `+64`.

Activation type `2` is the native any-player transition. On entry,
`sub_A16130` rejects a non-player, records the entrant's player index, and
accepts only when no tracked player was already inside. On exit,
`sub_A161D0` clears that index and accepts only when no tracked player remains.
For a positive timer, `sub_A16810` creates a pending contact instead of calling
Lua. `sub_A165B0` advances its elapsed value by simulation delta and invokes
the enter callback when elapsed reaches the configured threshold. A normal
exit through `sub_A168A0` removes the pending contact; with
`persistentTimer=false`, a later entry starts again from zero. Destroying the
volume still does not synthesize an exit callback.

The exact authored values are content/native-proven. Cryos chunk `930` has
`timeToActivate=0`, `persistentTimer=false`, `triggerOnceOnly=true`,
`triggerIfNotBeaten=false`, and activation type `2`. Campaign chunk `735` has
`2.0s`, false, true, false, and type `2`; chunk `623` has `0s`, false, true,
false, and type `2`. Thus chunk `735` requires the first accepted entrant to
remain continuously inside for two native seconds before `main`; after the
worker's `2s+4s` waits, its earliest unlock checkpoint is entry `+8s`.
Chunk `623` and chunk `930` dispatch `main` immediately on the accepted
any-player entry transition. None uses the native unbeaten filter.

The exact payloads are
`tutorial-trigger-zelems1-design-spawners.markerset` (chunk `735`, SHA-256
`8562a2f52110359ca3d98dfec8b37248f0e3de39c14b04de39c60dd2b790a217`),
`tutorial-trigger-zelems3-design.markerset` (chunk `623`, SHA-256
`c732927b6b09f376c821c4f3b6a3f466dc7cd0963e9847ef5a005cc84176558a`),
and the previously retained Cryos Audio payload (chunk `930`, SHA-256
`e7ce460e02f7c76459563ac13f9a24e525bb296c7f9fdef65fe8315db10a0bc4`).
Typed validation must distinguish contact admission, dwell cancellation, Lua
job creation, and job lifetime; the Lua waits begin only after native
admission completes.

**Native trigger once-only boundary.** IDA reflection metadata proves
`triggerOnceOnly` is trigger-definition offset `0x4d` (`+77`) and
`luaCallbackOnEnter/Exit/Stay` are `0x54/0x58/0x5c` (`+84/+88/+92`).
`sub_A16F40` copies `+77` to runtime byte `+60`, converts the enter Lua callback
at `+84` into runtime reference `+24`, and initializes runtime latch `+62` to
zero. Enter dispatch `sub_A16810 -> sub_A15FB0` suppresses a call only when
both `+60` and `+62` are nonzero. For this direct-Lua slot, `sub_A15F00` invokes
the callback as `(runtimeTriggerID, entrantObjectID)`, requests one boolean
result, and `sub_A15FB0` sets `+62=1` only when that result is true.

All eight prioritized level-script `main` prototypes end with Lua 5.1
`RETURN B=1`, meaning zero return values. The protected call requests one
result, so Lua supplies `nil`; `sub_8EEFB0` converts output type `5` through
Lua truth conversion `sub_8F1B30`, producing false. **Bytecode/native-proven
retail behavior:** the authored `triggerOnceOnly=true` flag does not latch for
chunks `930`, `735`, or `623`; their runtime byte `+62` remains zero after a
normal callback. A genuine later enter can dispatch another job unless some
separate phase owner destroys/disables the volume or rejects the callback.
There is no bytecode-proven such guard in these wrappers.

This also fixes the direct-trigger callback tuple: chunk `930` receives the
runtime trigger ID as its sole fixed argument and discards the entrant; chunks
`735` and `623` retain `(runtimeTriggerID, entrantObjectID)` and forward both to
their workers. Their `GetPlayerIdForObject(callbackArg1)` therefore resolves
the entering object. Chunks `62`, `769`, and `222` use the separate
`EventListenerData` path: their two retained arguments are
`(sourceObjectID, listenerObjectID)`, while the third `otherObjectID` delivered
by native dispatch is discarded by Lua. They must not be assigned the
trigger-volume tuple.

There is no once-token rollback contract around chunk `930` job allocation.
Whether `CreateObject` succeeds or returns `kObjIDNone`, `main` returns no value
and the native once latch remains clear. A typed director seeking build-103
parity must permit a later genuine re-entry; enforcing the authored once flag
would be a deliberate correctness deviation and must be labeled as such.
The retained diagnostic
`bin/game/logs/ida-trigger-field-xrefs.log` has SHA-256
`44af063af8def7632e2b378f572c03b0b4654f90098fbbfac32af76208ec3a41`.

The matching full-object teardown `sub_9D1B30` checks object `+676`, resolves
that ID through the same thread manager, destroys the thread record with
`sub_8EFFE0`, and clears the slot. `MarkForDelete` itself still only sets byte
`+93`; the object-manager sweep later notices the flag and dispatches object
removal. Cancellation is therefore object-owned and guaranteed when full
teardown occurs, but it is deferred after `MarkForDelete`. The bytecode does
not prove the exact sweep tick or packet ordering, and worker return does not
mark the five persistent job objects. Typed parity needs one active worker per
job object, rejection/no-op on duplicate creation, cancellable waits, and
phase/session teardown for jobs that never mark themselves.

The simulator update order narrows the deferred timing. `sub_9BF1B0` advances
the Lua-thread manager with `sub_8F10C0` near the start of the authoritative
update, runs the remaining simulator systems, and calls deletion sweep
`sub_9BD430` at the tail. A job that sets `+93` while resumed by that scheduler
pass is therefore eligible for full teardown later in the same update, after
the intervening systems; a mark made after that sweep waits for a later update.
This is native ordering evidence, not a packet-order guarantee.

Adjacent thread bindings provide a useful negative boundary (native-proven).
`GetCurrentThreadID()->number` returns the first word of the current Lua thread
record. `WaitForever()->()` clears the current record's two continuation-data
words and installs the engine's never-ready continuation predicate. `WakeUp`
accepts a numeric thread ID, resolves that record through the global Lua-thread
manager, and clears its sleep byte at record offset `+240`; it does not accept
an owning object. None of the eight tutorial jobs calls `GetCurrentThreadID`,
`Sleep`, or `WakeUp`, and `CreateThreadForObject` returns no Lua result to
retain. These bindings expose independent numeric scheduler identity while the
object retains its one owned ID at `+676`. They still provide no worker-return
path that marks or deletes the five persistent job objects.

**Runtime dependencies.** All eight job chunks explicitly require
`Lua!GlobalDefinitions.lua`, `Lua!Vector.lua`, and `Lua!TargetUtils.lua`.
The authored module aliases are not present in `lua_module_alias`, so those
dependency strings do not resolve directly. Content inspection nevertheless
recovers the `GlobalDefinitions` body as chunk `659`, resource `14227`,
`lua/0x2E64AA9E.lua`, SHA-256
`25927418c849f13c1f45592387f0d9d0fd1e8b67b6537daf0ac8b57d663d57b3`.
Its exact inventory is `CALLx35, EQx1, GETGLOBALx49, GETTABLEx35, JMPx1,
LOADKx528, NEWTABLEx40, RETURNx1, SETGLOBALx75, SETLISTx4, SETTABLEx374`;
only `SETLISTx4` is unsupported by the current VM. It is one top-level
prototype with no closures or upvalues. The content defines the expected
global enums and type collections. Although the raw dependency edge is null,
the package/linker audit and successful compile probes establish
`Lua!GlobalDefinitions.lua` and `0x3681d755!GlobalDefinitions.lua` as exact
aliases of chunk `659`, not absent content. The same audit closes the other two
aliases: case-insensitive FNV-1 maps `Lua!Vector.lua` to
chunk `439`, resource `13990`, `lua/0x0EE694B4.lua`, decoded SHA-256
`320d85202f00868498c62e8f6a1efb798c018be7ded8027df27dbb772521d8a6`,
and `Lua!TargetUtils.lua` to chunk `406`, resource `13957`,
`lua/0xBAE104D5.lua`, decoded SHA-256
`664abf0bfb49f7c17e58fa20e89127ba60f63797d8ca4554217554d43a930582`.
Their namespaces and caller behavior agree with those identities; these are
linker aliases, not absent helper bodies. Standalone parity still requires
typed definitions for `Class.newClass`, `namespace`, `LuaJobObject.Noun`,
`nUtil.GetAsset`, and the native namespaces. Chunk `655` has no explicit
`require`, but depends on preexisting
globals `nActivationType`, `nModifierPriorities`, `nAbilityFns`,
`nAttributeType`, and the modifier/ability namespaces. The normalized
`level_script` index now establishes the exact owners listed in the route-link
correction above: chunks `141`, `930`, and `215` belong to Cryos level `9`;
chunks `735`, `623`, `62`, `769`, and `222` belong to other campaign levels;
modifier chunk `655` is attached by its requesting spawn lifecycle rather than
a direct `level_script` row.

**Original three parity fixtures (opcode support now present):**

1. **Chunk `930`, second ability.** Exercise closure binding descriptors plus
   `GETUPVAL`, `LEN`, `SUB`, `FORPREP`, and `FORLOOP`.
   Required bindings: `require`/typed module shims, `Class.newClass`,
   `nUtil.GetAsset`, `nObjectManager.CreateObject`,
   `nThread.CreateThreadForObject`, `nThread.WaitForXSeconds`,
   `nPlayer.GetPlayerIds`, `nPlayer.UnlockNextAbility`, and
   `nGameObject.MarkForDelete`. Its live `1→2→3` transition is the strongest
   parity oracle. Assert the exact arities: `main` and `worker` each have one
   fixed parameter, and `CALL A=2 B=4 C=1` invokes
   `CreateThreadForObject(job, worker, job)` with no retained marker/entrant
   argument. An injected `CreateObject` failure must produce no thread, unlock,
   notification, or deletion. Assert that both successful and failed `main`
   calls return zero Lua values and therefore leave the native once latch clear;
   a second genuine enter must be dispatchable. A fixture that enforces the
   authored once flag is testing an intentional policy correction, not
   build-103 parity.
2. **Chunk `215`, second creature and event.** Reuse the fixture-one opcodes and
   exercise `NEWTABLE` and `SETTABLE` with an RK string key.
   Required bindings: `UnlockSecondCreature`, `nUtil.SPID`, `nEvent.Notify`,
   plus the common job/thread/delete bindings. Assert the same one-parameter
   `main`/`worker` shape and `CreateThreadForObject(job, worker, job)` call: the
   delayed worker retains no entrant identity. Assert mutation-before-event
   ordering and the one-field `ServerEvent` rather than persistent ownership.
3. **Chunk `655`, spawn modifier.** Exercise `NEWTABLE` module initialization and
   direct no-upvalue callback closures. Required bindings/enums:
   `nModifier.RegisterModifier/GetMyAgentID`, `nAbility.PreloadAsset`,
   `nLocomotion.Stop`, `nAttribute.AddAttributeModifier`,
   `nGameObject.AddEffect/RemoveEffect/SetAnimationState/ResetAnimationState`,
   `nThread.WaitForXSeconds`, `nActivationType.Unique`,
   `nModifierPriorities.Recall`, `nAbilityFns.Activate/Tick/Deactivate`, and
   `nAttributeType.Immobilized`. Assert the `0.5s` wait and the intentional
   absence of immobilization removal in `Deactivate`.

The client group contains:

| Script | Confirmed authored operations | Authority consequence |
| --- | --- | --- |
| `Tutorial_IntroAbilities` | wait `2.0s`; `PlayAbilityBlink(Ability_Enrage1)`; delete job | The ability arrow is client-owned Lua presentation. The server must expose the correct ability and cause the authored lesson callback; it must not reproduce `nUIManager` in Fang. |
| `Tutorial_IntroHealth` | wait `3.0s`; `PlayHealthBlink` | Client-only HUD presentation. |
| `Tutorial_IntroHealthAndPower` | wait `1.0s`; blink health every `1.5s` for `7s`; then blink mana every `1.5s` for `8s`; delete job | Uses `nGameSimulator.GetGameTime`; confirms presentation timers share the game timeline but do not mutate authoritative resources. |
| `Tutorial_OverdriveUnlockClient` | if beaten, return immediately; otherwise wait `3`, `5`, `2.5`, `2.5`, `2.5`, `1.5`, and `4s`, with four `PlayOverdriveBlink` calls interleaved after the first two waits; return without deleting its created job object | Client presentation paired with the later server unlock. |
| `Tutorial_RandomUnlockClient` | if beaten, return immediately; otherwise wait `2`, `4`, and `0.5s`; blink `Ability_Enrage2`; wait `5.5` and `5s`; return without deleting its job object | Client presentation paired with the later server unlock. |
| `Tutorial_SupportUnlockClient` | if beaten, return immediately; otherwise wait `2`, `4`, and `0.5s`; consecutively blink `Ability_Support1`-`3`; wait `5.5` and `5s`; return without deleting its job object | Client presentation paired with the later server unlock. |
| `Tutorial_SoloSupportUnlockClient` | if beaten, return immediately; otherwise wait `2`, `4`, and `0.5s`; consecutively blink `Ability_Support1`-`3`; wait `6.5s`; return without deleting its job object | Client presentation paired with the later server unlock. |
| `Tutorial_CatalystUnlockClient` | if the local player is unbeaten, waits `2s` then four `4s` intervals; otherwise returns immediately | Original bytecode proves there are no calls between the waits. This is authored no-op client timing, not a missing presentation or server mutation. |

This inventory changes the implementation target. darkspin should not execute all
sixteen chunks in a server VM. It should reconstruct server jobs into validated
typed intents and allow the retail client to execute client-owned callbacks
when their level triggers and prerequisites are correct. Server execution of a
client chunk would duplicate UI timing and encourage launch-DLL substitutions.

### Remaining tutorial command coverage

“Command” spans three different boundaries here, and they should not be mixed
in one packet checklist:

| Boundary | Covered | Still missing or incomplete |
| --- | --- | --- |
| Player -> server | movement/ground click, targeted/basic and Ride actions, W/Q creature deploy, loot/health-obelisk interaction, Beam Out status and final remove-player reason | authored squad-death/game-over acknowledgement and restart-button request; Sage's tutorial-required ability requests beyond deploy; any special command used by a still-unrecovered objective |
| Server -> client presentation/state | object create/delete for existing hero/enemy fixtures, movement goal, combat event/state, animation, visibility, ability availability/cooldown, objective/client event, squad deploy, interactable state, cinematic layout, director boss-complete, XP update, pickup/full feedback, and tutorial-complete payload shape | generic projectile/dropped-pickup create and launch replication; complete Quadra camera/object ordering; exact game-over/failure overlay and restart response; terminal tutorial-complete reliability/order; any result payload between Beam Out and ship if a clean-profile trace still exercises one |
| Lua/native -> authoritative director | waits, object-scoped jobs, create/delete, visibility, animation, event notify, cinematic, ability unlock, second-creature unlock, overdrive/crystal locks, later unlock-job schedules, projectile wait/damage/death, loot scheduling, orb selection, crystal pickup creation/launch-state initialization, and the separate Cryos arena director entry | route membership for the later unlock jobs; exact crystal create/launch wire and persistence; retail horde random budget/composition and exit transition; failure/restart primitives and objective dependencies not present in the recovered named chunks. The registered `ActivateHordeSpawn` native itself is a literal build-103 no-op. |

This means the normal input vocabulary is close to closed; most remaining
work is determining when an already-known message is legal and in what authored
order it is emitted. A missing Lua native does not automatically imply a new
RakNet command: `WaitForXSeconds`, collision waits, drop selection, the separate
Cryos director, and phase cancellation are server-local simulation operations.
`ActivateHordeSpawn` is an even stronger negative example: its build-103 callee
immediately returns false without changing state.
Conversely, UI-only calls such as `PlayAbilityBlink` remain client callbacks
and must not become launch-DLL or invented server commands.

The separate AI ability `FirstAggro_SpecialOne` is the bounded server-timeline
fixture. Its recovered values are `animationTime=1.291667`,
`startDelayTime=2`, `endDelayTime=1`, and `cinematicRadius=100`. On activation
it resolves the agent position, starts a `4.291667s` cinematic, hides the
agent, waits two seconds, shows it, then the shared template repeats the
visible state and plays `character_teleport_in`. It waits one animation-length
interval of `1.291667s`, waits the final one-second tail, then deactivates. Go
still decides that the enemy has entered valid first aggro
exactly once; the authored ability supplies the accepted presentation timeline.

That acceptance predicate is now explicitly unresolved for Quadra rather than
a recovered radius. **Content/native-proven:** its linked
`TutorialSpecialOne_Intro.NonPlayerClass` stores both `aggroRange` and
`alertRange` as `0.0`, and the native perception helper selects their maximum,
so its authored perception radius is zero. The normal first-aggro entry event
is `Aggro_Trigger`; its first hostile aggro-list insertion posts timed AI
stimulus `0x20` for `10s`. The AI definition separately stores
`FirstAggro_SpecialOne` in `firstAggroAbility`, but the native edge selecting
that string from stimulus `0x20` is not recovered. Lua `nObject.AlertObject` is
a separate explicit path that posts stimulus `0x200`. Neither path identifies
the external director/event producer that wakes Quadra. The generic `18`-unit
movement sphere is therefore not build-103 evidence for this noun and must not
be used as the parity oracle. Full evidence is in
[`quadra-perception.md`](quadra-perception.md).

The adjacent content links narrow that missing producer without assigning it to
the wrong route phase. **Content-index-proven:** Quadra is marker row `17718`
(`marker_id=3112855115`, AI marker set `291`, ordinal `26`) at
`(259.345642,81.391426,25.088490)`, and it has no `level_event` row. The
co-located server trigger is a different marker, row `17742`
(`marker_id=3912233898`, marker set `293`, ordinal `13`) at
`(259.345673,81.391434,25.088488)`. Its sole event is row `6098`, the
radius-`25` `luaCallbackOnEnter` binding to
`nTutorial_IntroSecondCreatureUnlock.main`; level-script row `133` links that
event only to chunk `215`. Queries for `event_name=Aggro_Trigger`,
`callback_name=Aggro_Trigger`, and `callback_name=FirstAggro_SpecialOne` return
no normalized `level_event` rows. Thus exact position equality does not make the
Sage trigger the authored Quadra activation owner.

**Recovered-bytecode-proven negative:** the current Lua corpus has no
`Aggro_Trigger` or `TutorialSpecialOne_Intro` constant. Its only four
`AlertObject` callers are unrelated registered combat abilities: chunk `180`
(`nAbility_ScaldronBasicDoppler_Clone`, SHA-256
`176b81e1f0f90eacc37756e60329c9363d3f556aa991afd688644cbf9d1c8606`), chunk
`790` (mutation-agent projectile, SHA-256
`2d776a8b16da4c21ef729e30fa668fb9175244c63029c8da13abdc3a3178dae7`), chunk
`970` (`nAbility_Shadow_Boss_Duplicate`, SHA-256
`12ff4109c28ad14411f410a9841462f2764a6ba728f189c2c5ae6c4c7a252555`), and
chunk `1000` (`nAbility_ShadowTeleport`, SHA-256
`f77185af76574f0b44c00a1d5f05164d9bf89fbde32b40988c680cf8c575a3a0`).
Their calls alert an ability-selected target or a newly created clone/duplicate;
none links to marker `17718`, event `6098`, or chunk `502`. This excludes those
four scripts as evidence for the missing producer, but does not exclude an
engine/director producer whose dispatch is data- or hash-driven. **Required
director validation:** entering the Sage radius or completing chunk `215` may
unlock the second creature, but must not by itself authorize Quadra aggro. A
separate, authenticated phase intent is required until its producer and
predicate are recovered.

The other explicit Lua aggro-control entrances are equally bounded.
**Recovered-bytecode-proven:** all five `AddAggroForObject` callers are chunk
`302` (`nModifier_CitadelSpecificTwo_Taunt`), `307`
(`nModifier_PlasmaSentinelPet`), `454` (`nModifier_Taunt_Template`), `497`
(`nAbility_Entangle`), and `688` (`nAbility_Babytoss`). The only
`OverrideFirstAggro` callers are chunk `527`
(`nModifier_Citadel_Boss_Passive`) and chunk `1000`'s Shadow Boss path; the
only `IgnoreFirstAggro` callers are chunk `347`
(`nModifier_EnemyPortal_Passive`) and chunk `970`'s Shadow duplicate. None has
a tutorial, Cryos, Quadra, marker-`17718`, or chunk-`502` content link. These
calls cannot supply the missing authored owner merely because they reach the
same native aggro subsystem.

**Native-proven dispatch boundary:** `sub_A173A0` hashes callback names with
FNV-1a and registers them in a process callback map; `Aggro_Trigger` hashes to
`0xA3A887FE`. The only native callback-map consumer is the
`TriggerVolumeDef` post-load resolver `sub_A17210`, installed by the type
registration at `0x00F77384`. It resolves three authored hash slots into
function pointers: definition `+0x00 -> +0x7c`, `+0x10 -> +0x84`, and
`+0x20 -> +0x80`. `sub_A16F40` copies those pointers into runtime callbacks
and assigns them physics masks `1` (enter), `2` (exit), and `4` (stay), in
that order. A full executable scan found no embedded `0xA3A887FE` immediate;
the literal name occurs only at the callback registration. This favors a
data-authored `TriggerVolumeDef`/generic physics dispatch over a hard-coded
tutorial caller, but the particular definition or owner feeding Quadra is
still not recovered. The focused IDA log is
`bin/game/logs/quadra-perception/ida_aggro_dispatch.log`, SHA-256
`7e845586693220d59f7d1a12902340c6ec0cd6d19edd6bbe62d4b9663caaa10b`.

The raw-content audit closes the remaining hidden-fan-out possibility inside
the recovered Cryos level. `darkrun db ... bget` extracted level `9` and all
sixteen marker-set payloads (`281..296`). Neither the level payload nor any
marker set contains FNV key `FE 87 A8 A3`, literal `Aggro_Trigger`, or literal
`FirstAggro_SpecialOne`; marker set `291` contains only the expected Quadra noun
strings, and co-located set `293` contains the Sage callback but no aggro
callback. Level `9` has six normalized director noun entries, but none contains
the missing aggro trigger or publisher. **Content-proven conclusion:** recovered
Cryos authorship links marker `3112855115` through the noun and AI first-aggro
slot to chunks `502 -> 848`, but contains no level-event or trigger-definition
producer for the initiating insertion.

**Native-proven acceptance contract:** `Aggro_Trigger -> sub_A22620` rejects
client/non-authoritative mode (`sub_9BCF80()==1`), disabled source byte `+93`, a
failed source-team/target eligibility check, or a source without an agent brain
at `+684`. An accepted call synchronously invokes
`sub_9E4640(blackboard,targetID,5.0,2,"Trigger Volume")`. The `0x20` first-aggro
stimulus is posted for `10s` only when agent state is not `2`, the target is not
already recorded, the list was empty, first aggro byte `+1365` was not consumed,
and flags contain `2` or `4`; this callback supplies `2`. Let `t_a` be that
accepted insertion. The stimulus exists at `t_a`, but no recovered instruction
proves that chunk `502` is selected on the next AI evaluation or fixes the
native delay between those events. Once an authoritative selector starts chunk
`502`, its proven `0..4.291667s` presentation timeline applies. No recovered
ordering joins `t_a` either to that start or to chunk `215`'s Sage unlock at
trigger-entry plus `0.5s`. Typed parity must therefore validate an
authoritative first hostile insertion into an empty, unconsumed Quadra list and
a separately latched first-aggro ability start—not player distance,
Sage-trigger entry, or completion of chunk `215`.

**Current playable compatibility owner:** the missing authoritative producer
prevents a complete route if no server policy supplies the first insertion.
After chunk `215` has independently completed and Sage authority is committed,
darkspin now inserts the controlled player as Quadra's first hostile target
with trigger flag `0x2`. The normal aggro state posts/evaluates stimulus `0x20`,
then compiled chunk `502` owns the exact presentation timeline. The same
cancelable schedule creates object `42` in encounter combat authority before
the cinematic, so its defeat can reach the separately documented five-guard
teleporter transition. This is compatibility policy, not recovered evidence
that chunk `215`, its marker, player distance, or Cryos Lua initiated aggro.

The remaining native `"Spawn Aggro"` entrance is now excluded as Quadra's
missing producer. **Native-proven:** `sub_9E4640` has exactly three direct
callers: Lua `AddAggroForObject`, `Aggro_Trigger`, and `sub_9FEBC0`. The latter
is called only by `ActivateMinionSpawn` and `ActivateLieutenantSpawn`; their
wrappers decode `(spawnPosition,npcList,shouldAggro)`, pass list selector `0` or
`1`, and return the newly allocated object ID. Only after `sub_9FE7D0` creates
that object does a true `shouldAggro` enumerate live player-controlled objects
and call
`sub_9E4640(newObject.blackboard,controlledObjectID,5.0,2,"Spawn Aggro")`.
It cannot target an already placed marker object such as Quadra.

**Bytecode/native-proven cancellation within the sole packaged owner:** only
chunk `347`, `nModifier_EnemyPortal_Passive` (resource `13894`, SHA-256
`071de2f42edfe21e3a94d8a4d1bc9bd333f4c1a20d1e0ce24417d1104910e225`), calls
either spawn binding. Both call sites pass literal `true`. On each successful
return it immediately calls its no-upvalue `SetupSpawnedObject(child,portal)`:
first `AddDependency(child,portal)`, then `IgnoreFirstAggro(child)`, with no
wait or yield between the native return and suppression. Native
`sub_9E3CC0` sets consumed byte `+1365` and removes stimulus mask `0x2a0`, which
includes the just-posted `0x20`. The portal child therefore uses explicit
portal spawn effects while suppressing the ordinary first-aggro family; this
path cannot schedule Quadra chunk `502`. Neither the five decoded Quadra assets
nor any of the sixteen Cryos marker sets contains `EnemyPortal`, either spawn
binding name, or `Spawn Aggro`. Focused native evidence is in
`bin/game/logs/quadra-perception/ida_spawn_aggro.log`, SHA-256
`949dcddd5a2ba57b1f39a5d2980097f66c4d206b2632b02692fb524f37071d4a`.

This fixture spans two packaged chunks. `Abilities/0xC57828E2.lua` is the small
`FirstAggro_SpecialOne` specialization (`lua_chunk.id=502`, resource `14061`,
size `1237`, SHA-256
`da18208fc597c67ce3251de30d12ad694ae4d644af0e1cb79b5383cc12ec6d8d`). It
requires the shared `nAbility_FirstAggro_Template` implementation in
`Abilities/0x78AA702F.lua` (`lua_chunk.id=848`, resource `14425`, size `1060`,
SHA-256
`a799a493587c70e6386be91aaa1dfbdb9cdf4cfd4562f0322c5776046df39671`). The
specialization is one ability sequence, not the tutorial's master director.
`content.db` records the authored module alias and resolves chunk 502's require
edge directly to chunk 848. The constrained Go compiler now retains the exact
chunk ID and Lua PC for every emitted intent.

The first sequence-level parity fixture is therefore:

| Simulation offset | Recovered Lua intent | Typed Go expectation |
| ---: | --- | --- |
| `0` | resolve the accepted agent position | Resolve the stable boss-intro role to one live object and snapshot its authoritative position. |
| `0` | `StartCinematic(4.291667, x, y, z, 100)` | Emit one cinematic presentation intent tied to the current phase generation; do not surrender simulation authority. |
| `0` | `SetIsVisible(agent, false)` | Set authoritative presentation visibility false and emit the build-103 visibility update. |
| `2.000000s` | `SetIsVisible(agent, true)` and clear stealth | Restore visibility/target presentation only if the same object and phase generation remain live. |
| `2.000000s` | the shared first-aggro template repeats `SetIsVisible(agent, true)` | Retain the idempotent second visibility intent in semantic parity traces; do not collapse an authored call merely because its resulting state is unchanged. |
| `2.000000s` | `SetAnimationState(character_teleport_in)` | Emit the accepted animation event with a timestamp from the shared simulation clock. |
| `3.291667s` | template animation wait completes | Release the animation step; no combat or movement is inferred merely from the wait ending. |
| `4.291667s` | end-delay completes and ability deactivates | Complete the sequence and cancel its remaining lifetime token. The cinematic client deadline ends at the same authored offset. |

Restart, game over, object death, or phase exit before any row invalidates the
fixture token; later rows become no-ops. This is the parity-test pattern for
every subsequent tutorial sequence: recovered Lua order on one side, validated
Go state and typed presentation on the other, both advanced by the same fake
simulation clock.

The build-103 receiver now confirms `CinematicMsgs` exactly. Logical GMS type
`0x4a` maps to wire opcode `0xc9`; handler `sub_450F20` reads one signed
little-endian `int64` duration followed by four little-endian `float32` values:
focus `x/y/z` and radius. There is no subtype byte, so the complete application
message is 25 bytes including the opcode. A positive duration is added to the
client's current gameplay clock and the receiver enters game state `8` when
`sub_450DD0` accepts the request. The adjacent C++ `SendCinematic` probe had the
right byte count but mislabeled the single 64-bit duration as two independent
`uint32` fields (`9500`, `10000`). It remains useful corroboration, not a
semantic specification. The retail Lua seconds-to-integer-milliseconds rounding
policy, replacement/cancellation behavior, RakNet send ordering, and live
verification still require recovery before the Go director emits this message.
The receiver's proximity gate, state-8 camera interpolation, deadline cleanup,
and return to normal state `6` are now statically confirmed in the protocol
note.

### Proposed Go director vocabulary

The reconstruction should replace independent session booleans with named
phases. The initial vocabulary, pending complete director and footage
correlation, is:

1. `deployBlitz`
2. `teachMovementAndBasicAttack`
3. `unlockRideTheLightning`
4. `practiceRideTheLightning`
5. `teachHealthAndPower`
6. `activateLootObelisk`
7. `introduceResistance`
8. `unlockSage`
9. `introduceQuadra`
10. `fightQuadra`
11. `clearTeleporterGuards`
12. `enterHordeArena`
13. `fightHorde`
14. `offerBeamOut`
15. `completeTutorial`

Failure is not a numeric continuation of this route. `gameOver` and
`restarting` are explicit terminal/re-entry states that cancel phase-scoped
jobs and construct a clean session.

### Ordered implemented route table

The fifteen names above remain the evidence-oriented lesson vocabulary. The
live Go director intentionally groups those lessons into the following nine
authoritative phases. This is the single implemented route order; optional
lesson facts do not create hidden phase numbers.

| Order | Go phase | Entry authority and active route content | Completion fact and next phase | Evidence status |
| ---: | --- | --- | --- | --- |
| 1 | `openingCombat` | A fresh tutorial session deploys the player, owns the opening encounter and placed-orb set, and accepts movement and abilities. | `openingEncounterCleared` -> `claimOpeningLoot` | Object placement and combat content are recovered/live-tested. The server's opening reveal radius remains labeled compatibility policy. |
| 2 | `claimOpeningLoot` | The loot obelisk becomes the phase-owned interactable after the encounter clear. Reaching the authored obelisk can make the claim available even when optional opening enemies were bypassed. | Transactional `openingLootClaimed` -> `approachSage` | Chunk `736` timing and the storage rollback boundary are implemented. Exact retail policy for bypassed opening enemies remains unproven. |
| 3 | `approachSage` | Movement, abilities, placed orbs, and the exact chunk-215 Sage marker are active. | `sageMarkerEntered` -> `introduceQuadra` | Marker identity, position, radius, server-only flag, and half-second unlock job are content-backed. |
| 4 | `introduceQuadra` | Sage and Quadra roles become active; chunk `502` owns the cinematic, visibility, teleport animation, and release sequence. | `quadraIntroduced` -> `defeatQuadra` | Lua timing and build-103 presentation shapes are parity-tested. The current hostile-target insertion that starts first aggro is an explicit compatibility owner because the retail producer is missing. |
| 5 | `defeatQuadra` | Quadra enters ordinary authoritative combat with the unlocked squad. | `quadraDefeated` -> `clearSecuritySet` | Quadra HP, BurstShot behavior, death, and transition are implemented. Exact retail camera/clear ordering still needs a retained live trace. |
| 6 | `clearSecuritySet` | The authored five-member guard set and dormant platform teleporter are active. | `securitySetCleared` -> `enterTeleporter` | Placements and enemy families are content-backed; reveal/pursuit cadence retains documented compatibility policy. |
| 7 | `enterTeleporter` | The powered teleporter accepts its recovered radius-2 contact and chunk-144/349 modifier handoff. | `teleporterEntered` -> `clearHorde` | Packet/timing order through teleport-in is covered. The first-wave `+2s` delay is live-compatible, not retail-server proven. |
| 8 | `clearHorde` | Horde enemies and arena health-obelisk roles are active; movement, squad switching, abilities, and health-obelisk use are accepted. | `hordeCleared` -> `beamOut` | Four waves are live-confirmed. Composition, budgets, `1.5s` inter-wave delay, and final alert placement remain compatibility policy pending the missing server director. |
| 9 | `beamOut` | Combat and health-obelisk commands close; movement, squad switching, and Beam Out remain accepted. | Reserved/committed `tutorialCompleted`; no later mission phase | Persistence and the single positive `C8 00` completion/ship transition are transactional. `AF 00` must not be stacked with it. |

The route also carries orthogonal, idempotent facts rather than inserting
extra phases:

| Branch | Where it may occur | Authority and outcome |
| --- | --- | --- |
| Movement/basic/Ride lessons | Primarily `openingCombat`; completion facts remain valid afterward | Recovered markers and typed ability admission publish lesson presentation. A failed schedule restores the prior fact. These lessons do not authorize later route transitions by coordinate alone. |
| Second-ability unlock job | The authored server-only marker while its route command is accepted | Chunk `930` runs once per session generation. Failure reopens the fact for retry. |
| Placed orb pickup | Every phase retaining `placedOrbSet` | Contact is optional. Accepted pickup mutates only the deployed resource under the current evidence-constrained policy; full-resource feedback does not collect a placed capsule. |
| Sage deployment and abilities | From `introduceQuadra` through `beamOut` | Squad availability owns switching. Blitz and Sage retain independent HP/mana and per-ability cooldowns. No route phase requires a particular optional switch. |
| Arena health obelisks | `clearHorde` only | Each phase-owned obelisk can create one placed health capsule. Use is optional and cannot complete a wave. |
| Opening-enemy bypass | Before the loot claim | Reaching the authored obelisk may advance to `claimOpeningLoot`; leftover opening authority is discarded when Quadra completion installs the security set. This is implemented compatibility behavior, not yet a recovered retail branch. |
| Debug checkpoint | Developer chat only | `!skip` constructs the selected phase with all preceding route facts satisfied. It is test/developer injection and is never an authored route edge. |

Terminal and re-entry flows are deliberately outside the numeric route:

| Terminal branch | Trigger | Ordered authority result | Remaining uncertainty |
| --- | --- | --- | --- |
| Squad game over | Every available squad character reaches zero HP in any combat-capable phase. | The simulator squad latches terminal once, publishes the build-103 game-over state, rejects later match mutation, and cancels captured session work during replacement. | The exact retail squad-death acknowledgement and failure overlay/button exchange are not recovered. |
| Restart | Status `8` while the squad is terminal. | Reserve restart once in the squad aggregate -> reset durable tutorial XP/level -> verify the same peer generation and reservation -> replace the complete match session -> rebuild from `openingCombat`. Persistence failure rolls the reservation back for retry. | Status `8` is the live-compatible request boundary; the authored restart-button request/response packet sequence remains open. |
| Boss completion | Quadra death or the final horde death. | Quadra death advances to the security set. Final-horde completion owns the defeated alert, delayed director boss-complete update, and `hordeCleared` transition to `beamOut`; schedule failure rolls the marker back. | Retail horde director fan-out, terminal alert timing, and boss-complete reliability remain open. |
| Successful exit | Accepted Beam Out in `beamOut`. | Reserve completion -> persist XP/level, Blitz/Sage squad, and reward choice -> commit route terminal fact -> publish the hero beam -> retire the completed Blaze game shell -> publish one positive `C8 00` completion snapshot. Its native handler updates in-memory progression and selects the ship state in one operation. Any pre-commit failure restores the route reservation/state. | Reports proved that substituting `AF 00` reaches a scene-ready/UI-ready ship with stale tutorial progress and then crashes, while retaining the completed tutorial shell makes later launches rejoin its initial chain vote and crash during account loading; the corrected retirement plus single-`C8 00` path needs real-client validation. |
| Session abandonment | New Hello, peer replacement, shutdown, or transport retirement. | Stop every match-owned run and invalidate the old epoch. A later tutorial entry constructs a fresh route and transient state; no old callback may publish. | This is server lifecycle authority rather than an authored gameplay branch. |

### Route evidence reconciliation

The ordered route above supersedes the first hard-coded reconstruction where
stronger evidence disagrees with it. The archived checklist records the
original symptoms, but an unchecked historical box is not by itself evidence
that the old implementation or ordering should be preserved. This table gives
each material disagreement a current disposition.

| Earlier observation or reconstruction | Stronger correlation | Disposition |
| --- | --- | --- |
| The opening path appeared empty, and the old server began with the western five-enemy group before creating later enemies behind the player. | Exact level markers plus the walkthrough's 16 pre-obelisk defeats establish the four-, four-, five-, then three-enemy traversal order. | **Resolved in the route.** The corrected content-backed order replaces the hard-coded order. A clean live pass is still required for reveal timing and for proving that the loot prompt cannot bypass a required authored clear. |
| Ride lesson state, enemy spawning, the HUD arrow, and post-cast behavior were coupled to movement-handler booleans. | Recovered lesson chunks, build-103 action decoding, and typed director facts separate marker entry, ability admission, first accepted cast, and encounter authority. | **Resolved structurally.** The old boolean ordering is obsolete. HUD-arrow lifetime, Ride's visible post-cast animation, and Voltic's live alternating presentation remain verification gaps. |
| The obelisk could be absent because its server predicate depended on the incorrectly ordered encounter scaffold. | Chunk `736`, the authored obelisk location, walkthrough presentation, and the transactional claim operation establish a distinct claim boundary after opening combat. | **Resolved in the route.** `claimOpeningLoot` owns the interactable and reward transaction. Whether retail permits the implemented coordinate-triggered opening-enemy bypass remains evidence-blocked. |
| An early focused checkpoint introduced Quadra before Sage and treated Quadra's death as the unlock condition. | Frame-by-frame footage shows the Sage portrait and switch lesson first; chunks `215`, `930`, and `502` establish the marker, half-second unlock job, then Quadra introduction timeline. | **Resolved and superseded.** Sage is revealed before Quadra packets and Quadra defeat advances to the security set. The missing retail producer for first aggro and the complete camera/control sequence remain open. |
| A later enemy group looked spurious, and the security teleporter sometimes stayed inactive or appeared duplicated. | Level marker correlation accounts for the late pre-obelisk trio and the five-member security set. Chunks `144`/`349` and the level-authored teleporter noun establish one platform object with inactive/active presentation. | **Resolved in route ownership.** The group is retained, the security clear gates `enterTeleporter`, and the server does not create a second teleporter noun. Live presentation and activation cadence still need confirmation. |
| The first horde reconstruction inferred a small fixed sequence and timing from one playable run. | The walkthrough visibly spans multiple waves and content supplies `waveOverride=4`; no recovered retail server director yet supplies exact budgets, composition, or delays. | **Partly resolved.** Four waves are authoritative input. Current composition, the first-wave `+2s`, `1.5s` inter-wave delay, and final alert timing remain explicitly labeled compatibility policy. |
| The old implementation granted a one-time XP `101` stage reward and had no reliable per-kill ledger. | The walkthrough's rendered bar, build-103 strict level thresholds, exact authored encounter count, and `LabsPlayerUpdate` receiver support the 16 pre-obelisk awards and Quadra `+54`. | **Corrected with constrained compatibility data.** The two horde jumps are currently assigned to deterministic actors only as policy; the retail XP formula and persistence cadence remain unrecovered. |
| Capsules were clickable/deletable fixtures and their resource ownership was ambiguous. | Native pickup callbacks, authored orb resources, footage, and current walk-over tests distinguish contact collection, deployed-resource mutation, and full-resource feedback. | **Resolved for admission and lifetime.** Walk-over collection is authoritative and click does not consume a capsule. Exact effect presentation and whether Sage shares or owns the recovered resource remain open. |
| New-enemy focus, player beam-in, floating hit text, pursuit facing, and attack animation were inferred from visible defects rather than an ordered director. | Build-103 receiver and packet work identifies typed presentation boundaries, but neither footage nor retained traces close every producer, timing, or camera transition. | **Still valid presentation gaps.** They do not alter phase order and must be closed by binary evidence or a clean live trace, not additional route booleans. |
| Game over/restart and Beam Out were treated as ordinary status continuations. | The simulator now owns terminal squad state, transactional restart reservation/session replacement, completion reservation/persistence, completed-game retirement, and one positive `C8 00` snapshot-and-ship transition; the video ends before the return click. | **Resolved server authority; real-client validation pending.** Reports establish that `AF 00` crashes after reaching the ship and that retaining the completed tutorial game breaks later launches; exact failure overlay, restart-button messages, retail reliability, and account-refresh ordering remain open. |

The remaining live-observation disagreements therefore fall into three bounded
classes: presentation verification, missing retail policy/formulas, and missing
client request/response traces. None requires restoring the superseded enemy
order, Quadra-before-Sage checkpoint, one-time XP scaffold, clickable capsules,
duplicate teleporter noun, or session-level route booleans. ; the milestone boxes in
`notes/tutorial/history.md` remain a historical snapshot.

Typed director output should remain small and semantic: object spawn/despawn,
visibility, animation, effect, cinematic, client event, objective, ability
availability, squad availability, interactable state, encounter activation,
combat activation, reward request, delay, and phase transition. Each event
refers to stable roles such as `activeHero`, `sage`, `quadra`, and
`bossTeleporter`; the build-103 adapter resolves those roles to session object
IDs only when encoding packets.

Lua/native calls do not bypass validation. `UnlockSecondCreature`, for example,
becomes a squad-unlock request accepted only in `unlockSage`; `SetIsVisible`
becomes a presentation request accepted only for an object owned by the active
phase; and every delayed continuation carries its phase generation so it is
discarded after restart or transition.

### Current hard-coded path migration map

The existing `gameplayPeerSession` is useful as a protocol harness but does not
encode the authored director. Its booleans and movement-handler branches map to
the proposed model as follows:

| Current state/branch | What it presently controls | Director replacement |
| --- | --- | --- |
| `isDungeonSetupSent` | constructs most tutorial objects during a debug-ping response | one idempotent `deployBlitz` phase-entry transaction |
| `isAbilityLessonSent` / `isAbilityLessonDone` | coordinate intersection, HUD-related packets, test enemy, and first accepted Ride cast | separate client lesson trigger, server ability-unlock job, and `practiceRideTheLightning` completion condition |
| `isEncounterSpawnSent` | suppresses several unrelated opening/focused spawns | encounter instance identity and phase-owned spawn roles |
| `isLootObeliskSpawned` / `isLootObeliskUsed` | object availability, interaction acceptance, reward, and Sage prerequisite | explicit obelisk object state plus a completed-objective fact; reward remains a Go transaction |
| `isSageUnlocked` / `isSageEnemySpawned` | squad availability, command authority, Sage object creation, and Quadra creation | distinct `unlockSage` and `introduceQuadra` phases; the retail half-second unlock job precedes the first-aggro cinematic |
| `isTeleporterUnlocked` / `isHordeArenaEntered` | enemy-clear gate and coordinate teleport | `clearTeleporterGuards` completion followed by an accepted teleporter interaction/trigger transition |
| `isHordeComplete` | wave scheduling, damage routing, success messages, and exit acceptance | horde encounter completion fact followed by `offerBeamOut`; persistent completion occurs only after the accepted exit path |
| `openingEncounter` / `horde` pointers | mutable enemy state for multiple route segments | phase-owned encounter IDs behind a common authoritative combat port |
| `packet.Schedule` callbacks | first-aggro movement, attacks, deaths, shock removal, waves, and boss completion | per-session simulation scheduler entries carrying phase generation and referenced-object lifetime tokens |

Today the movement action handler detects several segment intersections and
immediately mutates unrelated tutorial state. Segment/sphere intersection is a
valid way to prevent a long click from skipping an authored volume, but its
output should be only a typed `triggerEntered(markerRole)` command. The director
then checks the current phase, once-only policy, prerequisites, and activation
type before scheduling any consequence.

Migration should be incremental. First route new typed commands and timeline
events alongside the existing packet encoders; then move one phase at a time
behind the director. Remove each old boolean only after timeline tests and a
live build-103 pass prove equivalent or corrected behavior. This avoids a
single rewrite while still treating the current hard-coded route as temporary.

### Live traced route

A continuous build-103 traversal established the playable Cryos route rather
than inferring it from enemy marker order. The first observed lower-platform
route position is approximately `(177.755, -239.854, 0.088)`, facing toward the
upper-left path. The packaged tutorial camera marker is
`(176.288162, -239.948944, -0.058266)`, about 1.47 units away; current entry
uses that marker so the initial hero/camera handoff has no positional nudge.
The previous darkspin start `(44, 0.47, 17.5)` was not on the authored opening
route. A clean rebuilt run visually confirms deployment at the corrected
lower-platform position. The authored initial facing remains unencoded because
the current create/update messages do not carry a confirmed quaternion.
Representative stopped waypoints, in traversal order, are:

`(177.76,-239.85,0.09)` -> `(158.08,-240.77,0.03)` ->
`(140.80,-220.70,0.03)` -> `(146.42,-207.62,0.03)` ->
`(176.27,-190.15,0.09)` -> `(199.63,-158.80,1.51)` ->
`(220.02,-135.73,10.09)` -> `(230.06,-117.34,10.09)` ->
`(232.82,-98.29,10.09)` -> `(212.09,-76.94,13.12)` ->
`(182.17,-80.43,15.09)` -> `(139.49,-93.72,15.09)` ->
`(94.96,-85.61,15.09)` -> `(74.01,-50.57,20.12)` ->
`(79.10,-17.87,20.09)` -> `(114.51,11.99,25.09)` ->
`(155.08,19.41,25.09)` -> `(193.33,33.63,30.09)` ->
`(209.92,78.03,30.09)` -> `(252.19,86.48,25.09)` ->
`(293.28,129.33,25.09)` -> `(302.47,172.00,25.09)` ->
`(292.23,229.36,20.09)` -> `(267.13,233.87,20.09)`.

The route entered the authored abilities sphere at movement goal
`(232.816,-98.350,10.088)`, about `2.24` units from its center, and darkspin
logged the one-time lesson callback. The initial boundary is count `1`; the
separate server-only callback waits one second and advances it through `2` to
`3`, matching its two native `UnlockNextAbility` calls and exposing Ride the
Lightning at HUD index `2`. The objective and `vo_ship_tut_abilities` lesson
then produce the client-authored `Press 1` arrow, which clears on the first
accepted cast. This matches the retail automatic milestone unlock without a
pickup or persistent inventory grant. The adjacent server-only marker remains
separate: its script waits one second and calls `UnlockNextAbility` twice, so
is implemented independently from the client-visible lesson trigger. A clean
focused replay crossed both markers in one movement, exposed only Ride, and
recorded accepted blink availability `0x10002` while the white HUD arrow was
visible.

The initial three-slot snapshot uses Blitz asset/noun data for both locked
crash-guard slots because build 103 requires every fixed record to resolve.
Keeping valid fixed records while zeroing only the trailing reflected
noun/assets is not accepted: a clean build-103 run reached status `8` and
dungeon setup, then forcibly closed. The locked representation therefore needs
a resolvable noun/asset in both sections.
A second controlled test made both fixed and reflected locked records zero,
matching the old server-source empty-slot shape; build 103 rejected it too,
terminated gameplay, and returned to login with `80040000`. That experiment
formerly mislabeled top-level field `23` as an `UnlockSecondCreature` counter.
The complete reflection map now proves it is deck score. Exhaustive build-103
inspection finds no aggregation path beyond copying the field and the bounded
tutorial increment, and the preserved server source only comments out a call
to nonexistent `Squad::GetScore`; the current fallback therefore keeps it zero
unless external retail-server evidence appears. It does not control portrait
creation: all three valid Blitz-backed records remain visible.
Using the live-resolved
`TutorialBasicPoisonNoOrbs` noun/asset consistently for both locked records
also failed: the client reached dungeon setup and deployment, then exited.
Locked records therefore require more than a generic resolvable noun. The HUD
qualification is not owned by those records: top-level `LabsPlayer` field `22`
is the locked deck-index minimum. darkspin formerly hardcoded it to `0xff`, which
exposed every valid backing record as a portrait. Two controlled variants that
changed the outer mask to `0x1001` and omitted locked-slot reflections both
crashed after dungeon setup, with either zero or valid fixed backing records.
Restoring the complete `0x1fff` transport and all three valid character plus nine crystal reflections while
sending field `22 = 1` kept the client alive and live-confirmed exactly one Blitz
portrait at the normal authored start. A controlled delayed unlock then exposed
why the first Sage attempt produced two Blitz portraits: build 103 cached the
fixed squad identities from the initial snapshot, whose two hidden crash-guard
records were both Blitz. Sending a later full Sage snapshot created Sage's world
object but did not replace that cached HUD identity. darkspin now preloads Sage's
real noun/asset/type in fixed slot two during the initial snapshot while keeping
field `22 = 1`, so she remains hidden at launch. Raising
field `22` to `2` on unlock live-confirmed exactly one Blitz portrait and one
Sage portrait; pressing W then selected Sage's world object and Bio ability bar.
The ten-second launch probe used for this verification was removed, leaving the
authored route trigger as the only squad unlock path. Slot three retains a valid
hidden Blitz-backed crash guard because it is never exposed by this tutorial.
Routine movement now reserves `ObjectTeleport` for actual teleport mechanics.
A left-click is replicated as active-goal `ObjectPlayerMove` (`0x91`, flags
`0x01`) followed by the fixed object-ID/goal
`LocomotionDataUnreliableUpdate` (`0x95`). A clean build-103 test visibly walked
Blitz and tracked the camera without the old snap.

The August 17 playtests showed that changing hidden-object/deploy ordering, the
eventual hero position, ready-response packet ordering, and the timing of
`GamePrepareForStart` did not change the opening camera snap. A timed primer
then proved that build 103 accepted an invisible Blitz create, positioned
update, controlled-object binding, and `PlayerCharacterDeploy` after scene
activation and before the ordinary baseline, but the camera still ignored the
hidden object. That experiment is reverted. Static client analysis confirms
that the online `B0`/`B1` path does not use the level's local
`CameraSpawnPoint`; the camera becomes meaningful through the online startup
state instead. Commit `81c121f` changed fresh starts from the earlier two-phase
status-8 response (`GameStart` plus handshake ping, then the world baseline on
the following exchange) to a combined `GameStart`/baseline response for
Continue support. Tutorials cannot Continue, so their status-8 path now uses
the original two-phase handshake while checkpoint-capable campaign starts keep
the combined publication path.
Blitz now enters at the packaged camera marker
`(176.288162,-239.948944,-0.058266)` and retains the normal beam-in
presentation; the first live-confirmed route point remains
`(177.75531,-239.85399,0.08799999)`. The same
playtest also isolated an Infector locomotion split: server simulation stopped
at the 8.6-unit ranged attack distance while target-follow packets let the
client independently close toward melee range. Projectile pursuit now sends a
plain active-goal move to the projected server standoff point and redirects to
a newly projected point when the hero moves. The projectile's attack turn then
anchors both sides at the server arrival position; melee families retain their
target-follow pursuit contract.

The minimum combat slice also omits the retail encounter cadence. Tutorial
enemies should begin idle or invisible, aggro only after the hero enters their
perception perimeter, play the first-aggro beat, pursue the hero, and attack in
range. The opening pair is no longer included in the dungeon-setup snapshot;
movement now creates them once when the hero enters the authored 18-unit alert
perimeter. A clean live run confirmed the one-time create at player goal
`(97.636,-89.381)`, 17.6 units from the first marker. Standing directly on that
marker confirmed both nouns remain invisible in their authored pre-aggro state.
The extracted build-103 `ability_firstaggro_beamin_tutorial.lua` resolves that
transition: it makes the agent visible and non-stealthed, plays
`character_teleport_in`, and waits `1.291667` seconds. darkspin now creates the
nouns at the boundary, publishes visible/targetable combat blackboard state,
targets the hero, plays that state, and sends an authoritative stop at each AI
marker. A live run confirmed the stop prevents the prior northward escape, but
did not visibly show the beam-in animation despite the state packet. Because
the client sends no periodic post-setup `DebugPing`, the pause cannot depend on
a later inbound callback. The RakNet server now supports delayed reliable
application sends through the shared gameplay/QoS UDP mux. A clean build-103
run acknowledged every scheduled datagram after the exact authored delay.
Neither a scheduled `ObjectPlayerMove` (`0x91`) nor ten acknowledged
`ObjectUpdate` (`0x8c`) position snapshots moved an existing NPC in isolation,
and the failed position-snapshot bridge was removed. The build-103 message
registry confirms `LocomotionDataUnreliableUpdate` as logical message `22` / wire
`0x95`. Its fixed body is object ID plus a three-float goal vector. Because the
first-aggro stop leaves goal flags at `0x20`, the working transition sends
`ObjectPlayerMove` with active-goal flags `0x01`, followed immediately by
`0x95`. A clean live run visibly walked both infectors from their AI markers to
the hero after `1.291667` seconds and left them settled around the target.
Enemy retaliation now uses execution-time packet production rather than a
precomputed one-shot payload. Each tick reads the latest session position,
excludes defeated infectors, sequences the hero HP after each attacker, and
reschedules while an opening enemy is alive. A clean build-103 run visibly
confirmed repeated damage down to the deliberate 10-HP tutorial safety floor.
That immediate ten-damage path has now been removed. The live opening Poison
attackers use the shared timed-melee simulator: animation at activation, range
and target revalidation at the authored `0.43s` hit continuation, stable
same-deadline HP sequencing, two-second release/cooldown ownership, and the
same tutorial safety floor. Rank-one `1-3` damage is selected from the shared
simulator-owned deterministic RNG stream. The melee presentation is the exact
object-bound ServerEvent shape rather than a positioned effect. The build-103
`TutorialPoisonCloud` source and its projectile template are now recovered:
rank-one cooldown is `3s`, activation range is `8`, damage is `1-4`, projectile
speed is `6`, independent travel distance is `12`, impact radius is `1`, attack animation is
`ver_minn_lf_diseased_attack1`, time-to-hit/release is `0.17s`/`1.86s`, the
trail is `life_disease_spore_projectile.ServerEventDef`, the projectile noun is
`Ability_Fireball.Noun`, and impact presentation is
`life_disease_spore_cloud.ServerEventDef`. The template plays the attack
animation first, waits until the `0.17s` shot time, pays cooldown, creates a
separate projectile at the attacker footprint, attaches its trail, tracks it
toward the validated target, applies damage and the impact event on collision,
then deletes the projectile. Later `TutorialBasicDiseased` actors now use that
typed live run: a per-session projectile object, the `0.17s` shot,
collision-time player-position and HP refresh, impact/damage/deletion,
independent `1.86s` release, and cooldown retirement at `3.17s`. Damage samples
the authored rank-one `[1,4]` range from the same simulator RNG. The actor
footprint radius is the recovered `1.4`. The live adapter now performs the
recovered initial-overlap and continuous box sweep for a stationary target,
using projectile half-extents `(0.5,0.5,1)`, target-relative bounds
`(-0.5,-0.5,0)..(0.5,0.5,1)`, and strict range-expiry precedence. A retained
tracker now advances the projectile in 16 ms Go authority steps, subtracts
actual displacement from the remaining range, samples the player's latest
position every step, and sweeps only that displacement while preserving the
fixed launch direction. This matches the recovered non-predictive moving-target
contract. Static/world collision, oriented-box physics, and evidence for the
exact retail simulation cadence remain open. Controlled-object floating combat
text still requires live verification.

Both poison definitions are compiled at startup from the Lua chunks selected
by their exact registered-name constants in `content.db`. The Go runtime
receives the authored cooldowns, ranges, timings, damage bounds, animations,
events, projectile assets, and projectile parameters as typed definitions; the
server binary no longer duplicates those Lua-owned values as tutorial constants.

Blitz's basic attack is likewise an authored sequence rather than one repeated
state. `LightningRogueBasic` selects five animation states in order:
`el_lightningravager_basic_a` through `_e`. Their hit times are `0.233s` for
the first three and `0.3s` for the last two; release times are `0.366s`,
`0.4s`, `0.433s`, `0.366s`, and `0.8s`. The authored cooldown is `0.366s`,
range `1.75`, hit-arc length `3.5`, and the hit effect is
`plasma_common_electric_hit_small_player_effect.ServerEventDef`. Damage occurs
at the selected animation's hit time, not when the input packet arrives. The
demo's repeated arm swing and freeze are consistent with failing to preserve
the per-ability animation-sequence index and failing to separate hit time,
release time, and cooldown. Go should own target/range/damage/cooldown and the
sequence index; the presentation adapter should emit the corresponding
animation and hit effect at those accepted timeline points. Darkspin now
compiles this complete sequence from chunk 935 in `content.db`, including the
`nAbilityAnimationSelection.Sequence` enum, rather than duplicating its five
entries as Go constants. Accepted input advances the sequence and locks the
next request through the selected release deadline. The simulator revalidates
the live target and positions at the selected hit continuation, then owns the
damage, hit effect, death handoff, XP/progression, and cooldown publication.
The selected release continuation clears the run; session replacement and
schedule failure cancel it, with initial scheduling failure rolling back the
input lock and sequence cursor.

Build 103 currently simulates ordinary enemy pursuit client-side and includes
the target's live position in its targeted ability command. Until darkspin's Go
navigation owner replaces that client movement, the gameplay adapter accepts
that pose only for an already-known live encounter object and uses it for range
admission and the later hit recheck. Retaining the authored spawn position made
every observed Voltic Slash miss server range after enemies pursued Blitz,
which prevented all deaths, the loot-obelisk spawn, and therefore Sage unlock.

The opening Poison pack now also has a bounded compatibility pursuit owner.
While no creature is within `TutorialPoisonMelee` center range, each living
creature advances at its replicated `4.5` movement speed toward a stable
object-specific slot just inside that range. The adapter publishes build-103
`ObjectPlayerMove` active-target flags `0x43` followed by the ordinary `0x95`
goal snapshot. Bit `0x02` makes the client consume the explicit live target
facing vector while bit `0x01` keeps the movement goal active. This keeps the
four actors from sharing one destination and
lets the existing `0.233s` or `0.3s` continuation perform its live-position
arc check and apply bite damage. Exact retail pathfinding and avoidance remain
owned by the future general navigation simulation.

Voltic Slash admission and its delayed hit recheck use the same distinction.
The Lua `1.75` activation range is added to Blitz's authored `0.8` footprint
and the selected target noun's footprint; it is not itself a center-distance
radius. The prior raw-range check rejected live targets only 2-3 units away,
so the client played uncommitted swing input without receiving the accepted
timeline or cooldown. Accepted input now starts the content-compiled sequence
lock (`0.366s` through `0.8s`, depending on the selected animation) and pays
the `0.366s` cooldown at its authored hit continuation.

Held-click melee pursuit retains one server movement owner for the same actor,
ability, and target even when build 103 publishes a fresh sync stamp. The
client's targetless hold updates and repeated targeted clicks still receive
their required pursuit acknowledgements, but they do not restart the progress
and timeout schedules or republish an unchanged movement goal. Arrival, target
loss, and timeout clear every pursuit response lease for that actor. Ability
admission also samples its action time while holding the session lock, after
any scheduled pursuit update, so an older input timestamp cannot regress the
authoritative movement clock. This changes pursuit bookkeeping only; accepted
Voltic Slash swings retain their content-authored hit and release cadence.

The tutorial AI phase graph now resolves every authored primary combat ability:

| AI definition | Phase ability | Rank-one authored combat timeline |
| --- | --- | --- |
| `TutorialBasicDiseased` | `TutorialPoisonCloud` | Projectile; activation range `8`, travel distance `12`; cooldown `3s`; animation `ver_minn_lf_diseased_attack1`; shot at `0.17s`; release `1.86s`; speed `6`; damage `1-4`; radius `1`; projectile trail and impact cloud. |
| `TutorialBasicPoison` and `TutorialBasicPoisonNoOrbs` | `TutorialPoisonMelee` | Melee; range `0.75`; cooldown `2s`; animation `cry_minn_lf_poison_attack1`; hit `0.43s`; release `2s`; damage `1-3`; `life_common_melee_hit` effect. |
| `TutorialBasicRanged` | `TutorialPlasmaLightning` | Projectile; range `15`; cooldown `3s`; animation `cry_minn_el_ranged_attack1`; shot at `0.3s`; inherited release `0.45s`; speed `10`; damage `1-4`; lightning trail, electric impact, and ineffective miss effect. |
| `TutorialSloth` | `TailZap` | Melee; range `0.75`; cooldown `2s`; animation `cast_tailzap`; hit `0.43s`; release `1.2s`; damage `5-10`; `charge_impact_small_effect`. Its AI uniquely uses `nBehavior_StrafeOrIdle` between attacks. |
| `TutorialSpecialOne` and `TutorialSpecialOne_Intro` | `BurstShot` | Four-shot projectile burst; range `13`; cooldown `1s`; animation `cast_burstshot`; cumulative shot times `1.06s`, `1.46s`, `1.86s`, and `2.26s`; release `2.9s`; speed `25`; damage `2-7` per accepted hit; lightning bullet trail and medium electric impact. Later shots track between launches. |

The server now compiles all five rows from their packaged Lua registration
chunks during startup. Exact-name lookup considers every chunk containing the
name and accepts only the unique candidate that compiles into a registration
with that identity, so common references such as `BurstShot` cannot select an
AI consumer accidentally. Multi-shot `timetohit` tables become ordered
cumulative deadlines in the typed definition; single-hit abilities retain a
one-entry deadline list and the same first-hit compatibility field.

All seven AI definitions use `faceTarget=true`, begin with
`nBehavior_Invisible`, and use a first-alert beam-in. Six use
`FirstAggro_BeamIn_Tutorial`; `TutorialSpecialOne_Intro` replaces its first
aggro with the longer `FirstAggro_SpecialOne` cinematic while retaining the
ordinary beam-in for first alert. This means facing, visibility, first-aggro,
pursuit, attack animation, hit/shot timing, effect, damage, release, and
cooldown are separate state-machine concerns. A single periodic retaliation
callback cannot preserve retail parity.

The second parity fixture should use rank-one `TutorialPoisonCloud` with a
deterministic target distance. At offset zero, Go validates the actor, target,
team, phase, and eight-unit activation range, snapshots the attack attributes,
orients the actor, and emits the attack animation. At `0.17s`, the template
pays the three-second cooldown once, creates a distinct
`Ability_Fireball.Noun` projectile at the caster center plus facing times the
caster footprint radius, copies team and attribute snapshot, attaches the
projectile trail, and begins deterministic tracking at speed `6` with a base
travel budget of `12`. Collision—not the original input time—selects targets in
the one-unit impact radius, applies authoritative `1-4` damage, emits the cloud
impact event, and deletes the projectile. Ability release occurs at `1.86s`;
the cooldown is measured from the `0.17s` payment point and may extend beyond
release. Target death, projectile deletion, phase exit, and restart each have
explicit cancellation outcomes in the Go fixture.

The same distinction applies to Blitz: `LightningRogueBasic` pays cooldown and
applies damage at the selected animation's hit offset (`0.233s` or `0.3s`),
while release controls when the actor can leave that animation/action. Input
acceptance, animation start, hit, cooldown start, effect, damage, release, and
next sequence index must therefore be independently observable events.

The earlier 1.5-second opening-enemy deletion delay has been replaced.
Decompiled `Behavior_Death.lua` and the corresponding build-103 natives drive
the implemented stateful death behavior with separate ordinary, critical,
revival, and cleanup paths. Animation duration does not decide when an ordinary
corpse is deleted. Entry also publishes a reliable flags-`0x20`
`ObjectPlayerMove` at the corpse position so an earlier pursuit goal cannot
continue after death.

During opening-combat iteration, player deployment temporarily uses the last
live-confirmed safe point outside that perimeter,
`(102.38726,-91.42871,15.087998)`. The authored route start remains preserved
as `(177.75531,-239.85399,0.08799999)` and must be restored for the final
tutorial replay.

Floating damage numbers are driven by `CombatEvent` (`0xBA`). Build-103 client
analysis confirms field `deltaHealth` must be the positive display magnitude;
the signed mutation remains in `integerHpChange`. darkspin was sending `-10` in
both fields, selecting the client's no-display branch, and now sends `10` plus
`-10` for hero-to-enemy damage. A live retest updated the health bar but still
did not show a rising number. The first enemy-to-player events now update the
hero health bar but also show no rising number, which makes the client's
local-controlled-object binding the next evidence target.

The match damage adapter no longer clamps incoming tutorial damage at 10 HP.
Every accepted melee or projectile result updates the deployed entry in the
three-slot match-local squad. Reaching zero publishes `AD 01` exactly once only
when no available squad entry remains alive. After Sage is unlocked, Blitz can
therefore die without ending the tutorial while Sage retains positive cached
match HP; a later lethal update to the deployed Sage latches the solo wipe.
This state is transient and does not mutate the persisted creature roster.

Build-103 now explains that remaining prerequisite. GMS logical message `63`
maps to wire `0xba`; dispatcher `sub_53ADC0` routes it to `sub_539F60`, which
reflection-decodes the eight-field, 40-byte in-memory `CombatEvent` and calls
presentation handler `sub_4E2CA0`. Its field layout is:

| Field | Memory offset | Presentation use |
| --- | ---: | --- |
| flags | `0` | Damage style and secondary combat statuses. The receiver can enter numeric damage from `deltaHealth` alone, but the native damage constructor sets bit `0`; bit `2` marks a killing blow and bit `3` selects the critical damage style. |
| `deltaHealth` | `4` | A positive value enters the damage/absorb presentation branch; a negative value enters healing. Zero with no absorbed amount skips the ordinary numeric branch. |
| `absorbedAmount` | `8` | A positive value can display absorb text when `deltaHealth` is not positive. |
| `targetID` | `12` | Anchors the presentation and determines local/ally/enemy relationship. |
| `sourceID` | `16` | Determines whether damage was done by the local creature or an ally. |
| `abilityID` | `20` | Retained for combat context; it is not the rising number itself. |
| `damageDirection` | `24` | Vector used by combat presentation, not the numeric value. |
| `integerHpChange` | `36` | The actual integer passed to the damage-number presentation. Damage keeps this negative while `deltaHealth` is positive. |

With all eight reflection fields selected, the application message is `0xba`,
mask `0xff`, then the existing 39-byte body: 16-bit flags, two floats, three
object/ability IDs, one vector, and the signed integer HP change. There is no
second server-to-client floating-number packet. `sub_4E2CA0` selects a local
`combattext_damage*.ServerEventDef` style and queues it through `sub_506D90`
after accepting the `CombatEvent`.

The authored Build-103 sender does not normally select all eight fields.
`sub_A207F0` looks up reflection type `0x5f1cc727`, computes the nondefault
field set, emits logical message `63`, and reflection-encodes only that set.
The ordinary HP-change constructor `sub_9E4D90`, reached from `sub_9E5800`,
sets flags bit `0` for positive damage, also sets bit `2` when the target has
reached zero HP, and adds bit `3` for a critical hit. It stores the negated
signed HP mutation as positive `deltaHealth`, leaves `absorbedAmount` zero,
copies target and source IDs, leaves `abilityID` zero, uses a zero direction in
this generic path, and preserves the signed integer HP change.

Consequently ordinary noncritical tutorial damage with a live source selects
fields `0`, `1`, `3`, `4`, and `7`: mask `0x9b`. Its minimal application packet
is exactly:

```text
[0xba][0x9b]
[flags: uint16 little endian; 0x0001 ordinary or 0x0005 killing blow]
[positive damage magnitude: float32 little endian]
[target object ID: uint32 little endian]
[source object ID: uint32 little endian]
[negative integer HP change: int32 little endian]
```

That is 20 bytes including the application ID. A critical hit ORs `0x0008`
into the same flags value without changing the mask. A null source omits field
`4` and changes the mask, but tutorial ability damage has a real caster. The
current `0xff` all-fields form can decode, yet it is not the native ordinary
damage shape and should not be used as the parity golden. It is also incorrect
to populate `abilityID` merely because damage came from an ability: this
recovered generic authored path leaves it at its reflected default.

The paired combatant-state message has a similarly narrow, but not yet fully
closed, boundary. Wire `0x97` / logical `24` receiver `sub_539AA0` reads the
four-byte object ID, resolves the object, takes its combatant component at
object offset `716`, and calls `sub_A1DD10`. That decoder hardcodes reflected
type `0xfab3107f`, Build-103 `cCombatantData`, whose two fields are hit points
and mana points. Their component offsets are `64` and `68`; the damage path
changes the hit-points float at `64`. Object initialization seeds this type's
two-field comparison cache at combatant offset `76` through
`sub_9D1810`/`sub_9D1790`.

No direct native mapper call for logical `24` appears in the client sender
table, so the exact authoritative delta sender remains outside or indirect in
this binary. The state evidence strongly predicts that damage-only replication
selects just field `0`: `[0x97][object ID][0x01][new HP float]`, ten application
bytes, while mana-only uses field `1` and a combined snapshot uses mask `0x03`.
That HP-only packet is a constrained capture target, not yet a parity golden.
Sending unchanged mana with every HP mutation is decodable, but should not be
described as the recovered retail delta until a trace or authoritative sender
confirms it.

That local synthesis is gated twice. First, `sub_4E57B0` must resolve the
currently controlled simulation object from the active world plus either its
explicit player index or the client session's player slot. If that lookup
returns null, the entire positive damage/absorb path is skipped even when both
health and combat packet values are correct. Second, the matching game option
must be enabled: local source uses `ShowDamageDoneByMe`; an allied source uses
`ShowDamageDoneByAllies`; local target uses `ShowDamageDoneToMe`; and an allied
target uses `ShowDamageDoneToAllies`. The negative/healing branch similarly
uses `ShowHealingDoneToMe`, `ShowHealingDoneToAllies`, or
`ShowHealingDoneToEnemies` according to the target relationship. The retail
defaults still need live confirmation, but changing signs or inventing a
ServerEvent cannot repair a missing controlled-object binding.

The missing server reflection is now identified and implemented. Build-103's
captured `LabsPlayer` metadata maps top-level field `9` to the object ID at
player offset `+4664` / `+0x1238`, exactly the field read by
`GetPlayerControlledObjectID` and the combat-text gate. The previous initial
snapshot skipped fields `9` through `11`, while `PlayerCharacterDeploy` alone
was only proven to update selection presentation. Initial tutorial and
Sage-unlock snapshots now set field `9` to Blitz object `1`; every squad swap
publishes the exact sparse field-9 `0xa1` reflection before the deploy message,
binding it to object `1` or Sage object `17`. A live run with the updated Fang
trace must still confirm a nonzero `combat_control_relation` and visible text
under the enabled preference.

The initial full snapshot still arrived before Blitz's world `ObjectCreate` in
darkspin's status/setup exchange. Because the reflection decoder resolves the
wire `tObjID` into a client object handle stored at `+0x1238`, that early field
could not establish a durable local-object association. Dungeon setup now sends
the sparse field-9 update again immediately after Blitz `ObjectCreate` and
before `PlayerCharacterDeploy`. A sequence regression locks
`ObjectCreate -> LabsPlayer field 9 -> PlayerCharacterDeploy`; the next live
run must determine whether this produces relation bit `1` or `2` and restores
the locally synthesized number.

Target and source IDs should continue to identify live replicated objects when
the event is handled. Immediate deletion is additionally unsafe for a killing
blow because the client-generated presentation is anchored by `targetID`,
although the earlier nonfatal no-text test proves deletion timing is not the
only current blocker. Finally, `ObjectDelete` removes the object immediately
and cannot drive a death animation. darkspin now has a golden-tested encoder for the exact
25-byte build-103 `SetAnimationState` (`0xa5`) body. Build-103 native sender
`sub_A1FDD0` lays it out as object ID, state ID, 64-bit simulation timestamp,
overlay byte, scale float, and a final 32-bit source/echo-gate field. The
`nGameObject.SetAnimationState` and overlay wrappers at `sub_9FC000` and
`sub_9FC1A0` always pass zero for that final field. The older C++ implementation
instead repeated the state ID there; darkspin inherited that guess, which forced
the receiver past its local/current-state suppression branch and could restart
predicted animations. darkspin now emits the native zero. The five enemy nouns
reference four exact type-`0x17BBCE29` CharacterAnimation resources in
`AssetData_Binary.package`. Their first noncritical death entries identify the
ordinary melee states: Poison and Diseased use
`zlm_minn_sp_3_death_melee`, Ranged and Special One share Plasma's
`gen_death_melee`, and Sloth uses Quad's `gen_death_melee_quad`. Recipe 13
links those source resources to `noun_physics`, and the live death behavior
consumes the projected state. The later critical/element/knockback entries are
retained in the raw resources for descriptor-specific selection.

### Authored death behavior

`Behavior_Death.lua` supplies the deletion scheduler that the animation packet
does not. Its entry phase clears the target, calls
`SetAnimationStateToDeath`, applies an `Immobilized` modifier, stops ordinary
movement (or turns a knocked-back non-player toward its hit direction), and
disables physics and navigation collision. Build 103 maps
`SetAnimationStateToDeath` to `sub_A018D0`. That native resolves the creature's
`characterAnimationData`, chooses a death state from the last hit descriptors,
and calls the exact `sub_A1FDD0` `SetAnimationState` sender. Its descriptor
tests include `Critical=0x1`, `SourcePhysical=0x2`, `Technology=0x8`,
`Life=0x20`, `Elements=0x40`, `Supernatural=0x80`, and
`Knockback=0x400`; these are hit-descriptor bits and must not be confused with
the separately encoded `CombatEvent.flags` bits.

| Death case | Authoritative sequence | Client-visible consequences |
| --- | --- | --- |
| Ordinary NPC or boss | Wait until HP rises above zero or the ten-second NPC timeout expires. Return immediately on revival. If HP remains zero, set corpse-fading state, attach the creature-type fade effect, wait five seconds, then mark the object for deletion. | Death `0xa5` begins at entry. The elemental fade begins at about `t+10s`; replicated deletion follows at about `t+15s`. |
| Critical, non-player, non-boss NPC | Set corpse-fading state, wait `0.1s`, stop locomotion, wait `3s`, then mark for deletion. | Critical death animation begins at entry and deletion follows at about `t+3.1s`. This branch does not explicitly attach the elemental fade effect. |
| Player-controlled object | Wait indefinitely for HP to rise above zero (`timeout=0`). If revived, leave the death behavior through cleanup. | No authored 30-second delete/respawn occurs here despite the unused `playerRespawnTime=30` constant. Game-over/respawn ownership lies elsewhere. |
| Interrupted or revived behavior | Remove the stored Immobilized modifier. If the object was not marked for deletion, reset its animation, remove its elemental fade effect, and restore physics and navigation collision. | A phase cancellation or revival must undo presentation and collision state instead of leaving a dead or non-collidable live object. |

The fade selection is deterministic: Technology uses `fadeaway_cyber`,
Spacetime uses `fadeaway_spacetime`, Life uses `fadeaway_bio`, Elements uses
`fadeaway_plasma`, and the fallback uses `fadeaway_necro`. Packaged
`TutorialBasicDiseased.ClassAttributes` has `creatureType=2`, and build-103
`nDamageTypes` maps `2` to Life, so its ordinary corpse effect is
`fadeaway_bio.ServerEventDef` (case-insensitive FNV-1 asset ID `0xea273c08`).
Native `AddEffect` uses the already recovered
one-based, force-attached 16-byte `ServerEvent` slot recipe.

The scheduler helpers do not themselves emit gameplay packets.
`WaitForHitpointsAbove` (`sub_9FFE60`, predicate `sub_9FFB20`) stores the object,
threshold, and timeout and completes when the object vanishes, corpse-fading is
already set, time expires, or HP exceeds the threshold.
`WaitForFadeOutInXSeconds` (`sub_A07BC0`, predicate `sub_9FFE10`) only counts
down simulation time. `SetCorpseFadingAway` (`sub_A02B10`) sets combatant byte
`+52`; `MarkForDelete` (`sub_A02910`) sets object byte `+93`. Normal object
replication subsequently produces `ObjectDelete`; neither Lua wrapper sends an
immediate delete packet.

Cleanup animation is packet-closed as well. `ResetAnimationState`
(`sub_A02130`) clears the object's stored animation state and calls
`sub_A1FDD0` with state ID zero, a fresh simulation timestamp, overlay false,
scale `1`, and source/echo zero. Revival therefore emits the same exact 25-byte
`0xa5` shape with `state=0`; it is not merely a Go-local flag clear.
`SetTargetID` (`sub_A023D0`) delegates to simulation mutation `sub_9E3DC0`.
`SetObjectAsCollidable` (`sub_A00310`) calls the physics manager, while
`SetNavCollision` (`sub_A03CC0`) writes the inverted enabled byte at object
offset `644` and rebuilds navigation through `sub_9E6930`. None of those three
Lua wrappers directly calls a GMS sender, so any target/collision wire delta is
an ordinary replication responsibility and remains a trace/sender-recovery
target rather than a packet to invent in the death behavior adapter.

A death parity trace must keep damage and lifecycle events distinct. At `t=0`,
emit the killing `CombatEvent` (`0xba`, ordinarily mask `0x9b` and flags
`0x0005`; add `0x0008` for a critical presentation), replicate HP zero, and
emit the selected death `SetAnimationState` (`0xa5`). Go must also retain the
authoritative hit descriptors that select the native death state and critical
fast branch. Ordinary `TutorialBasicDiseased` then has a revival decision at
ten seconds, a Life fade interval from ten to fifteen seconds, and deletion
only afterward. Required fixtures are ordinary timeout/deletion, revival before
timeout, critical non-boss fast deletion, boss critical taking the ordinary
path, player indefinite wait, and phase/session cancellation with complete
cleanup and no stale delete callback.

### Kill rewards, XP update, and corpse lifetime

Reward acceptance is separate from `Behavior_Death`. That behavior contains no
XP or drop call, and its ordinary NPC can remain in the world for fifteen
seconds after HP reaches zero. The tutorial footage instead moves the account
XP bar at the visible defeat. Go should therefore commit a kill, XP award, and
eligible drop exactly once when authoritative combat first transitions the
target from live HP to zero; corpse revival/fade/delete scheduling must neither
delay nor repeat that transaction. A later revival may restore the simulation
object, but it cannot silently duplicate the already accepted reward.

Build 103 closes the in-mission XP presentation packet. GMS logical message
`35`, wire `0xa1` (`kGmsLabsPlayerUpdate`), dispatches through
`sub_53ADC0 -> sub_539E00`. The receiver reads the player slot, resolves the
corresponding Labs player, and calls reflection decoder `sub_A1E070`. Its
top-level reflected type is `0x15ff16e2`; mask bit `12` (`0x1000`) selects the
base Labs player record without selecting any of the three creature records
(`0x504ccfda`) or nine trailing records (`0x14d310c1`). Base fields `15` and
`16` are the account/Crogenitor level and cumulative XP float respectively.

The exact minimal application packet for a combined level/XP update is:

```text
[0xa1]
[player slot: uint8]
[top-level selection mask: uint16 little endian = 0x1000]
[base field index: 0x0f]
[account level: uint32 little endian]
[base field index: 0x10]
[cumulative XP: float32 little endian]
[base reflection terminator: 0xff]
```

This is 15 bytes including the application ID. It is a cumulative snapshot,
not an XP delta: after accepting an award, Go computes the new cumulative XP
and strict-threshold level, persists them atomically, and emits both resulting
fields. The already recovered thresholds begin `100, 200, 3000`, and
`sub_9CEA00` advances while XP is strictly greater than the current bound, so
XP `100` remains level 1 and XP `101` is level 2. `TutorialGameMsgs 0xc8`
subtype zero remains the end-of-tutorial cumulative XP/onboarding snapshot; it
must not be substituted for these per-kill `0xa1` updates.

The native loot boundary is also distinct from XP. Objects initialize byte
`+153` (`can drop loot`) and byte `+154` (`worth XP`) to true.
`MarkCantDropLoot` clears `+153`; `MarkNotWorthXP` (`sub_A03F70`) clears
`+154`. `DropStuffForObject` maps to `sub_A035C0`, which resolves the defeated
object and recipient/source object, accepts an optional signed drop selector
(default `-1`), and calls `sub_9CE8D0`. For a combatant it queues, rather than
immediately serializes, the following authoritative state:

| Combatant offset | Queued meaning |
| ---: | --- |
| `+92` | Recipient/source object ID, or zero. |
| `+96` | Optional signed drop selector; omitted Lua argument becomes `-1`. |
| `+100` | Current simulation timestamp plus the native delay argument (the Lua wrapper passes zero). |
| `+104` | Drop-processing flag; the Lua wrapper passes false. |

`sub_9CE8D0` returns without doing anything when object byte `+153` is false.
The eventual `sub_9CE140` drop processor branches over the queued selector,
creates the selected orb/item simulation objects, and in one branch sends a
reflected `ServerEvent` through `sub_A20910` before launching the created
pickup. It never mutates Labs player XP. Conversely, the separate `worth XP`
flag does not authorize loot. Tests and the director model must preserve these
orthogonal decisions: `award XP`, `schedule drops`, and `run corpse behavior`.

Build-103 native code now fixes the selector vocabulary independently of the
older server projects. `sub_9CE140` tests bit `0x02` for orbs, `0x04` for
catalysts, `0x08` for loot, and `0x10` for DNA. Selector `1` reaches none of
those branches and is therefore the no-drop value. This makes the tutorial
pair exact: `TutorialBasicPoison.dropType=2` selects the orb branch, while
`TutorialBasicPoisonNoOrbs.dropType=1` selects no drop. The old C++ server has
the same enum, but its implementation is only an incomplete comparison
scaffold: it hard-codes health orbs and has no collection path. It is not the
authority for this mapping.

The native orb branch treats its scaled drop amount as a budget in hundred
point units. Each complete `100` guarantees one selection; the final remainder
is the percentage chance of one additional selection. The budget first passes
through a level/difficulty-indexed scaling table. For each accepted selection,
`sub_9FB360` computes resource weights from the average current fractions of
the player's roster: health weight is `2 + 5 * (1 - healthFraction)` and mana
weight is `2 + 5 * (1 - manaFraction)`. It chooses mana when the random fraction
falls outside health's share of the combined weight; otherwise it chooses
health unless a separately gated resurrection-orb rule succeeds. The three
return values map in order to `HealthOrb.Noun`, `manaorb.Noun`, and the
resurrection orb, paired with their matching `health_orb_drop`,
`mana_orb_drop`, and `resurrect_orb_drop` events. Resurrection eligibility has
session counters and timers (defaults include count `3` and `30`-second
parameters), but their complete gameplay meanings remain unnamed.

For an accepted orb, native simulation creates the pickup at the computed drop
position, initializes a zeroed `ServerEvent`, writes the event asset at
structure offset `0x10`, writes the three coordinates at offsets
`0x24..0x2c`, emits it through `sub_A20910`, and starts authoritative launch
motion with parameters `2.5` and `0.5`. Build-103 reflection registers `0x24`
as field `10` `position`; field `13` `targetPoint` starts at `0x4c`. Thus the
drop presentation is the 20-byte application recipe
`[9b][06 asset-u32][0a position-3xf32][ff]`, not the old server's
asset-plus-`targetPoint` interpretation. This proves the semantic create,
presentation, and launch sequence, but not immediate RakNet framing order:
object replication and motion may be flushed after the simulation call stack.

The typed simulator now implements the native-proven health, mana, and
resurrection branches. Its input is the already-scaled orb budget owned by the
missing loot authority. It consumes one resource choice draw for every
guaranteed hundred, then the final remainder chance, then one more choice draw
only when that chance succeeds. Resource
weights use the average current fraction across the supplied roster before
applying `2 + 5 * (1-fraction)`. Each accepted choice produces a stable
phase-owned role, exact noun/event identity, injected collision destination,
and the shared height-`2.5`/duration-`0.5s` lob. When an available squad hero
is dead and no prior resurrection capsule remains in the zone, the native
health-side override produces `ResurrectOrb.Noun` with
`resurrect_orb_drop.ServerEventDef`; collection publishes the packaged pickup
effect, restores every dead squad member to maximum health, updates their
resources and checkpoint, then consumes the capsule transactionally. The
packet adapter uses a
dedicated 20-byte fields-`6/10` event codec and the generic 105-byte lob codec;
it deliberately returns those components separately and leaves generic create,
owner eligibility, lifetime expiry, and replication flush order unresolved.

Packaged tutorial data demonstrates why this matters. Both
`TutorialBasicPoison` and `TutorialBasicPoisonNoOrbs` are the same visible
Simulated Game Chlorosaur family, but their `NonPlayerClass.dropType`
entries are `2` and `1` respectively. The former is the authored orb-dropping
variant; the latter is explicitly the no-orb encounter variant. A generic
enemy-name or noun-family rule would incorrectly create pickups. The exact
pickup `ObjectCreate` form, launch-motion packet, ownership, restore policy,
and XP formula remain evidence targets; they must not be inferred from
`Behavior_Death` or the fade/delete deadline. Collection itself has no
orb-specific client message, as recovered below.

The projectile presentation boundary is now more precise. The recovered
projectile template calls `nObjectManager.CreateObject` for the projectile noun,
copies the attacker's team and authoritative attribute snapshot, and calls
`nGameObject.AddEffect(projectile, trail)`. Native `AddEffect` at `sub_A05F10`
allocates the first free one of sixteen object effect slots, stores the
`ServerEventDef`, and—when the object is replicated—emits a reflected
`ServerEvent` (`0x9b`) containing the asset, projectile object ID, one-based
effect-slot index, and `forceAttach=true`. This is a persistent attached-effect
recipe, not the minimal asset/object/position event used by current tutorial
scaffolding. Removal clears that same slot and sends the corresponding remove
recipe. `TutorialPoisonCloud` then uses `nThread.WaitForProjectile` as an
authoritative collision wait; impact emits its event at the returned collision
position/facing, applies damage to the accepted radius-one targets, removes the
trail when required, and marks the projectile for deletion. The Go parity
fixture therefore needs a stable projectile role/object ID and effect-slot
lifecycle in addition to its travel timeline; it must not collapse the trail,
impact, damage, and delete into one packet batch.

The packaged `Ability_Fireball.Noun` further constrains that role. Its build-103
noun data has `nounType=624914268`, `clientOnly=false`, `isFixed=false`,
`hasLocomotion=true`, `locomotionType=1`, and `hasNetworkComponent=true`, but
`hasCombatantComponent=false`. It uses a radius-one creature collision sphere
and a radius-`0.25` other-collision sphere. It is therefore a real, movable,
replicated simulation object, not an enemy and not a combatant. It must not be
created with the enemy/combatant state bundle merely because the current server
has an enemy-specific `ObjectCreate` recipe. Native `nObjectManager.CreateObject`
at `sub_A07F40` parses the noun and transform and calls the simulation object
factory `sub_9DB960`; it does not itself send a packet. Object replication is a
later adapter consequence of authoritative creation.

Build-103 reflection registration also removes ambiguity from the generic
create envelope. `cGameObjectCreateData` has ten reflected fields:
`noun` at offset `0`, `position` at `4`, XYZ rotation degrees at `16/20/24`,
`assetId` at `32`, `scale` at `40`, `team` at `44`, `hasCollision` at `45`, and
`playerControlled` at `46`. Those are in-memory offsets and include alignment;
the reflected wire values occupy 43 bytes. On the wire the object ID precedes
its `0x03ff` all-fields mask and those fixed values. The following reflected
`sporelabsObject` tail has 23 indexed fields:

| Index | Registered field | Projectile relevance |
| --- | --- | --- |
| `0`-`3` | team, player-controlled, input stamp, player index | Team may be repeated when non-default; player fields should remain absent. |
| `4`-`7` | linear velocity, angular velocity, position, orientation | Build-103 `WaitForProjectile` writes field `4`'s backing object state to normalized direction times speed and recomputes field `7`'s orientation. It also writes field `5`'s backing state to direction times acceleration, but Poison Cloud's acceleration is zero. Position `6` is established by object creation. |
| `8`-`12` | scale, marker scale, last animation state/time, override move/idle state | Normally defaults for `Ability_Fireball.Noun`. |
| `13`-`17` | graphics state, two graphics-state times, visibility, collision | Visibility/collision are emitted only if the authoritative snapshot differs from registered defaults. |
| `18`-`22` | owner ID, movement type, disable repulsion, interactable state, source marker ID | Poison Cloud's enemy-target branch skips owner field `18`. Homing initialization explicitly writes movement type `3`, but Poison Cloud has `homing=false` and takes the ordinary path, so field `19` should not be inferred merely from its projectile noun. |

This establishes the field vocabulary but not the exact retail projectile
snapshot. Build-103 `WaitForProjectile` narrows the useful Poison Cloud
candidates to nonzero linear velocity field `4`, position field `6`, and its
recomputed orientation field `7`. Angular velocity field `5` remains zero for
this ability, owner field `18` is skipped, and movement type field `19` is only
explicitly changed by the unused homing branch. Because the template creates
the object before initializing projectile motion, replication-tick ordering
still determines whether the first `ObjectCreate` contains the changed
velocity/orientation or a later locomotion update supplies them. A paired trace
or the authoritative object-replication sender is required. The proven
walking-NPC `ObjectPlayerMove` plus `0x95` goal pair is not evidence for
ballistic motion.

The two locomotion messages now have distinct Build-103 contracts. Logical
message `22`, wire `0x95`, is the fixed unreliable goal update: receiver
`sub_539810` reads exactly object ID plus three floats and copies that vector to
both `cLocomotionData.mGoalPosition` (component offset `0x148`) and
`mPartialGoalPosition` (`0x154`). It is suitable for the observed walking-NPC
goal correction, but it carries neither velocity nor projectile parameters and
is ruled out as Poison Cloud's ballistic startup packet.

Logical message `21`, wire `0x94`, is the variable reliable locomotion update.
Receiver `sub_539900` reads the object ID, resolves the object's locomotion
component, clears its transient float/byte at offsets `0x1a4`/`0x1a8`, and
reflection-decodes type hash `0xB6F447EF` (`cLocomotionData`) through
`sub_A1DBF0`. Its 18 registered fields are:

| Index | `cLocomotionData` field | Projectile relevance |
| --- | --- | --- |
| `0`-`3` | `lobStartTime`, `lobPrevSpeedModifier`, `lobParams`, `mProjectileParams` | Field `3` is the nested ballistic definition; the lob fields support a different trajectory mode. |
| `4`-`6` | `mGoalFlags`, `mGoalPosition`, `mPartialGoalPosition` | Navigation state; the fixed `0x95` receiver writes the two positions directly. |
| `7`-`10` | `mFacing`, `mExternalLinearVelocity`, `mExternalForce`, `mAllowedStopDistance` | Facing and external motion can participate in replicated movement but are not yet proven present for Poison Cloud. |
| `11`-`14` | `mDesiredStopDistance`, `mTargetObjectId`, `mTargetPosition`, `mExpectedGeoCollision` | Strong candidates for target-mode and predicted collision state. |
| `15`-`17` | `mInitialDirection`, `mOffset`, `reflectedLastUpdate` | Strong candidates for launch direction/origin adjustment and replication time. |

Nested `cProjectileParams` has 15 registered fields: `mSpeed`,
`mAcceleration`, `mJinkInfo`, `mRange`, `mSpinRate`, `mDirection`,
`mProjectileFlags`, `mHomingDelay`, `mTurnRate`, `mTurnAcceleration`,
`mEccentricity`, `mPiercing`, `mIgnoreGroundCollide`,
`mIgnoreCreatureCollide`, and `mCombatantSweepHeight`. This makes reliable
`0x94` the strongest known packet family for starting a replicated projectile,
but capability is not emission proof. The Build-103 client contains the
receiver and reflection vocabulary; the direct logical-message mapper search
did not expose a corresponding fixed native sender. In contrast, the same
native sender table has direct mapper calls for fixed messages such as logical
`37` and `39`, so the absence of direct logical `21` is meaningful: locomotion
delta selection is indirect or belongs to an authoritative replication layer
not present in this client-shaped binary. A retail trace or the authoritative
simulator's sender is still required to learn the selected field mask, ordering,
and whether `ObjectCreate` already carries initial velocity. The C decompile
cannot justify inventing an outbound `0x94` recipe from the receiver alone.

The component initialization path supports a reflected-delta architecture
without supplying that missing wire recipe. `sub_9D1810` passes the locomotion
component's cache at component offset `556` and reflected type
`0xB6F447EF` to `sub_9D1790`; the latter walks the reflected fields and stores
one computed value per field index. `sub_A15CA0` initially copies the
`0x50`-byte locomotion default cache into that location. This is evidence for a
per-field comparison baseline/cache, not an encoded packet buffer and not proof
of which fields an authoritative sender selects.

The call order makes that cache relevant to the launch transition.
`sub_9DB960` creates and initializes the object, inserts it into the simulation
manager, and calls `sub_9D1810` before returning the new object to Lua. The
projectile template only then calls `WaitForProjectile`, whose native wrapper
changes linear velocity and orientation. Those writes therefore occur after
the component's construction baseline was seeded. A subsequent field comparer
can observe them as changes; zero acceleration leaves angular velocity at its
default, and the ordinary branch leaves the construction movement type intact.
This strengthens fields `4` and potentially `7` as delta candidates, but does
not reveal whether the adapter folds the changes into a not-yet-sent
`ObjectCreate` or emits reliable `0x94` afterward.

`nThread.WaitForProjectile` is also now recovered far enough to place the
authority boundary. Build 103 registers it in `sub_A0FE90` with callback
`sub_A0FBE0`. The older `sub_A0A850` attribution is Build-127 reference only.
The build-103 callback resolves attacker and
projectile objects, reflection-decodes type `0x381da653` directly into the
projectile locomotion component's parameter block, applies attacker attribute
selector `67` multiplicatively to the decoded `mRange` slot, and copies a
positive attacker selector `26` (`ProjectileSpeedIncrease`) onto the projectile
as selector `48` (`MovementSpeedBuff`). Projectile step `sub_A2FC20` multiplies
velocity displacement and its collision sweep by `max(0, 1 + selector48)`, so
the effective attacker formula is `authoredSpeed * (1 + max(selector26, 0))`;
negative selector 26 is ignored by the copy branch.
normalizes the initial direction, and starts either ordinary projectile motion
with `sub_A2E320` or homing motion with `sub_A2EBA0`. It then installs scheduler
predicate `sub_A08A50` on the sleeping Lua thread. That predicate samples the
projectile's current world position, subtracts actual distance travelled since
the previous sample from remaining range, and keeps the thread asleep while
neither collision-result flag is set. Collision or range exhaustion resumes Lua
with the hit object/flags, impact coordinates, and remaining-range result used
by `TrackProjectile`. There is no client request and no `WaitForProjectile`
network message. Go must reproduce the deterministic movement/collision wait;
the client only needs the resulting object motion and presentation.

The state writes before that wait are now mapped precisely enough for a parity
oracle. The wrapper stores `direction * speed` at object offsets `52/56/60`;
the separately registered native `nGameObject.SetLinearVelocity` callback
`sub_A0B1C0` writes those same three floats, tying them to reflected tail field
`4`. It stores `direction * acceleration` at offsets `64/68/72`; native
`SetAngularVelocity` callback `sub_A0B240` validates that identity with tail
field `5`, even though using the angular-velocity property for projectile
acceleration is semantically surprising. It also recomputes object orientation
at offsets `36` through `48`. Ordinary `sub_A2E320` resets its collision-result
state, initializes its wait budget, and can store a predicted collision vector
in the locomotion component. Homing `sub_A2EBA0` additionally sets object byte
`97` to movement type `3` and installs target/homing state. Poison Cloud has
`homing=false`, so it uses the ordinary branch and must not acquire movement
type `3` in a Go parity fixture.

`GlobalDefinitions.lua` maps attribute selector `67` to `RangeIncrease`. The
generic Lua `TrackProjectile` has already changed the authored distance to
`distance * (1 + RangeIncrease)` before calling this native wrapper, and the
wrapper multiplies the decoded range by `(1 + attribute[67])` again. For a valid
caster with unchanged attributes, the recovered retail behavior is therefore
`baseDistance * (1 + RangeIncrease)^2`, not one multiplier. Tutorial actors are
expected to have zero RangeIncrease, so Poison Cloud remains distance `12` in
the ordinary tutorial fixture. A nonzero-modifier oracle case should preserve
this authored double application unless a live trace disproves it.

This address correction is independently visible in the build-103 decompile at
`bin/game/GameBin/Game.c`: `sub_A0FE90` registers the callback, and
`sub_A0FBE0` contains the reflected parameter decode plus the ordinary/homing
branch. The decompile is a cross-reference aid rather than executable source,
but it is now the preferred C-shaped reference for 103; the resurrection
project's modern `Game.c` remains labeled comparison evidence.

The current evidence supports this packet-facing timeline, with confidence kept
explicit:

| Authored point | Authoritative event | Build-103 client presentation | Confidence |
| --- | --- | --- | --- |
| Shot offset `0.17s` | Pay cooldown once; allocate projectile role; create `Ability_Fireball.Noun` at caster center plus facing times footprint radius; copy team and attribute snapshot. Poison Cloud's enemy target type does not set projectile owner ID. | Replicate a generic, non-combatant network object at the computed launch point. Exact `ObjectCreate` reflection fields and accompanying locomotion snapshot remain open. | Creation semantics confirmed; packet shape open. |
| Immediately after create | Allocate the first free effect slot and attach `life_disease_spore_projectile`. | `ServerEvent` (`0x9b`) with field `1` one-based effect-slot index, field `4` `forceAttach=true`, field `6` event asset, and field `7` projectile object ID. The derived minimal application packet is 16 bytes. | Native sender and byte shape confirmed; live framed golden still needed. |
| Travel | Integrate speed `6`, base travel distance `12`, the authored Lua-plus-native double application of nonzero `RangeIncrease`, collision spheres, target validity, and optional homing rules on the simulation clock. With zero RangeIncrease the maximum flight is `2s`, slightly beyond the `1.86s` ability release. | Replicate projectile transform/locomotion. Fixed `0x95` is ruled out for ballistic startup. Variable reflected `0x94` can carry `mProjectileParams`, target, direction, and collision state and is the strongest candidate, but its actual Poison Cloud field selection remains untraced. | Authority confirmed; candidate packet family narrowed. |
| Collision | Resume the projectile-bound tracking thread; emit a collision impact for a previously unseen direct-hit object, then select radius-`1` targets and apply rank-one damage `1-4`. Every accepted damaged target emits the configured hit event; because Poison Cloud has no distinct `onHitEvent`, it reuses `life_disease_spore_cloud` at the same returned position/facing. A successful direct hit can therefore author two cloud notifications. | An at-position impact recipe is exactly reflection fields `6` asset, `10` position, and `11` facing, terminated by `0xff`; the Lua table does not include object field `7`. Then emit native minimal `CombatEvent` mask `0x9b` and the HP combatant-state delta. | Lua branches and CombatEvent shape confirmed; live double-effect behavior, controlled-object text binding, and exact HP-only `0x97` mask remain to verify. |
| Miss/range exhaustion | Resume with no accepted hit and execute the template fallback. Poison Cloud has no `missEvent`, so the configured `impactEvent` is used at the terminal position/facing instead. | Emit one at-position `life_disease_spore_cloud` using fields `6`, `10`, `11`; a miss is not visually silent. | Bytecode branch confirmed; live playback open. |
| Cleanup | Poison Cloud inherits `timeAfterImpact=0`, so it does not explicitly stop its projectile trail. The projectile is marked for deletion immediately after tracking completes; object deletion owns attached-slot cleanup. The template only sends a hard effect removal before a positive post-impact delay. | Send `ObjectDelete` after the final impact/damage intents. Do not invent a separate trail-stop packet for Poison Cloud unless a trace contradicts the authored branch. | Lua cleanup confirmed; delete/effect teardown timing needs live verification. |

The packaged-bytecode decompile also changes the cancellation model. Target
validation occurs near ability-thread entry, before the shot wait; the template
does not perform the previously assumed second validation at `0.17s`. If the
object target is no longer usable, the generic projectile path can retain the
ability's target position and launch toward that fixed point. Go may reject an
invalid command at authoritative acceptance, but the parity fixture for a
target lost between acceptance and launch must not automatically expect “no
projectile.” It should assert the recovered fixed-position fallback separately
from server policy.

The projectile tracking coroutine is created with
`nThread.CreateThreadForObject(projectile, TrackProjectile, ...)`, while the
ability's `deactivate` function is empty. Release or ordinary ability
deactivation therefore does not itself delete an in-flight projectile. The
tracking thread owns collision/miss presentation and final `MarkForDelete`;
explicit projectile deletion, session teardown, or a Go phase-cancellation
policy are separate cancellation sources. This matters for the base flight:
distance `12` at speed `6` can finish `0.14s` after the ability's `1.86s`
release without being an authored stale callback.

For Poison Cloud's enemy target type, creation performs `SetTeam` and
`SetAttributeSnapshot` but skips `SetOwnerID`; that call is made only for
non-enemy target types. The first generic projectile `ObjectCreate` should
therefore not include owner field `18` merely to associate damage with the
caster. Damage attribution remains in the authoritative tracking arguments and
combat event, not the projectile object's owner reflection.

The full reflected `ServerEvent` structure has 26 fields. Native `AddEffect`
proves the attached-effect subset above; adjacent reflection evidence also maps
field `0` simple-swarm effect, `2` remove, `3` hard stop, `5` critical, `8`
secondary object, `9` attacker, `10` position, `11` facing, `12` orientation,
`13` target point, `14` text value, and `15` client event, followed by the loot
tail. These fields are one message family but different recipes: an attached
trail, an at-position impact, a slot removal, and a UI client event must not be
encoded as one unconditional asset/object/position shape. Fields not yet tied
to a build-103 native sender or receiver branch remain comparison evidence, not
an implementation contract.

The build-103 Lua-to-wire bridge is now version-locally identified. Registration
function `sub_A11670` binds `nEvent.Notify` to `sub_A11560` (and
`NotifyPlayer` to `sub_A115D0`). `sub_A11560` reflection-decodes the Lua table
as type hash `0x8619ff24`, rejects an all-default event, and passes the resulting
`ServerEvent` to `sub_A20910`. That sender selects logical game message `28`,
reflection-encodes the same type, and terminates its field stream with `0xff`;
the client dispatches that message as wire byte `0x9b`. This closes the
authored path from `nEvent.Notify({...})` to the already-mapped receiver without
borrowing the Build-127 `sub_A0C1D0`/`sub_A25F50` addresses.

Chunk `215` now has an exact event recipe (bytecode/native-proven). Build-103
`nUtil.SPID` is case-insensitive FNV-1 with seed `0x811c9dc5`, so
`SPID("PlayerUnlockedSecondCreature") == 0x71e2bc36`. The authored table sets
only reflection field `15` `clientEventID`; its complete pre-RakNet application
message is therefore `9b 0f 36 bc e2 71 ff`, seven bytes including the logical
message's wire ID and field terminator. `nEvent.Notify` queues this through
`sub_9C14B0` with target byte `0xff`, the simulator-wide broadcast form.
`NotifyPlayer` is a separate binding: after coercing Lua argument `2` to an
integer player index, it writes the event selector byte at structure offset
`+96` as `~(1 << playerIndex)` before calling the same sender and queue. This
is an exact native expression; whether downstream code names the byte an
exclusion mask or applies another interpretation is not proven here. Chunk
`215` does not call that scoped form. A typed intent must first accept and apply every eligible
`UnlockSecondCreature` mutation, then broadcast this one presentation event;
it must not emit one event per player or treat receipt as persistence.

The queue boundary is also exact (native-proven). `sub_9C14B0` appends a
24-byte simulator-outbox record with kind `2`, a zero word, target byte `0xff`,
and the reference-counted encoded-message pair. It contains no RakNet priority,
reliability, or ordering-channel member. Simulator destruction releases the
message reference in every retained record and resets the vector, but the
allowed decompilation exposes no direct consumer of that inline vector.
Consequently reliability, ordering channel, normal-tick flush timing, and the
queue-to-transport adapter cannot be inferred from `nEvent.Notify`; they still
require the external owner path or a live transport trace.

The build-103 client consumer supplies the exact presentation meaning
(native/content-proven). Its dedicated `0x71e2bc36` branch opens locale table
instance `0xd4c3c464`, resolves key `0x0b51513a`, and passes the resulting text
to the generic popup dispatcher. English `Text.package` ordinal `76`, decoded
size `2748`, SHA-256
`01b5ddbe1b61c2ad992b3c67675fb8fdf46c4cae2ca0b0f65be6b648065e357e`,
identifies that table as `Labs Popup Localization` and maps the key to
`Enemies of the same type as your hero will deal DOUBLE damage with their
attacks.` Thus the event named `PlayerUnlockedSecondCreature` presents the
type-matching double-damage lesson; it is not an unlock mutation, an account
grant, or even a literal "second creature unlocked" alert. The authoritative
Sage mutation and this broadcast popup are ordered but semantically distinct.

Consequently the Poison Cloud impact recipe has an exact minimal application
packet shape, before RakNet framing: `9b 06 3e ac 5a 2a 0a <x y z> 0b <fx fy
fz> ff`. The catalog hash for `life_disease_spore_cloud.ServerEventDef` is
`0x2a5aac3e`; vectors are three little-endian IEEE-754 floats. This is `33`
bytes total: one message byte plus a `32`-byte reflected payload. There is no
object ID in this recipe. The attached trail uses catalog hash `0xc665c5f6`
for `life_disease_spore_projectile.ServerEventDef` and the minimal shape `9b 01
<one-based slot> 04 01 06 f6 c5 65 c6 07 <projectile object ID> ff`, `16`
bytes total. These byte shapes are derivations from the recovered native field
codec and packaged Lua tables; a retail capture is still valuable as an
end-to-end framing and playback check.

### Sequence-parity fixture design

Parity tests should prove that the Go simulation reproduces recovered authored
semantics; they should not claim that Go executes arbitrary Lua bytecode. Each
fixture has two inputs: a normalized oracle trace transcribed from identified
Lua instructions/native calls, and the trace produced by the Go director under
the same deterministic world. Compare these semantic traces before encoding
RakNet packets so a packet-layout correction cannot disguise a director-order
regression.

Use a small versioned trace vocabulary:

- `AuthorityEvent`: validate actor/target/phase, begin cooldown, create role,
  copy team/snapshot, integrate projectile, collide, select targets, apply
  damage, mark/delete role, advance phase.
- `PresentationIntent`: animate, create/show/move/delete object, attach/stop
  effect slot, emit effect, emit combat presentation, cinematic, dialogue,
  objective, and unlock hint.
- `WaitCondition`: duration, absolute simulation deadline, projectile collision,
  player radius, object death, or explicit client command.
- `CancelScope`: session, phase, actor, projectile, or authored thread.

Every oracle event should carry provenance: content asset ID/name, function,
bytecode instruction or recovered source location, native address when relevant,
and a confidence label. Stable roles such as `openingInfectorA`, `player`, and
`poisonCloudProjectile#1` replace runtime object IDs. Times are offsets from a
fixture epoch; a separate adapter test maps offsets to build-103 timestamps.
The deterministic inputs are a fake monotonic clock, seeded random/damage
stream, role allocator, initial transforms, collision world/result, and scripted
target-validity changes. A mismatch should report the first divergent semantic
event, including provenance, rather than only a large packet-byte diff.

The initial `TutorialPoisonCloud` fixture should place the actor and target at a
known distance within eight units and inject the exact collision time. Expected
events are animation/orientation and initial target validation at `t=0`;
cooldown start, projectile creation at the computed center/facing/footprint
launch point, copied state, trail-slot attachment, normalized direction,
linear velocity `direction * 6`, zero angular velocity from zero acceleration,
and distance-`12` travel at `t=0.17`; collision impact, radius selection,
accepted-hit impact, and damage at the injected collision offset; release at
`t=1.86`; and direct object deletion when tracking ends. The ordinary fixture
must assert that no homing target state or movement type `3` is introduced.
Release and cooldown are independent: the three-second cooldown begins at the
shot/payment point and extends beyond release, while a base maximum-range
flight ends at `t=2.17`.

Required cases are a direct hit with two cloud notifications, radius splash to
multiple accepted targets, range-expiry miss with one cloud fallback, target
loss before shot with fixed-position launch, target invalid during travel,
projectile deletion during travel, release before flight completion, phase
cancellation before and after creation, and session restart. Piercing-capable
shared-template fixtures must additionally prove duplicate-hit suppression.
One shared-template fixture must use nonzero `RangeIncrease` and assert the
recovered squared multiplier across the Lua and native boundary.
Restart must leave no scheduled callback, effect slot, projectile role,
cooldown mutation, damage, or stale packet intent. Only after semantic parity
passes should presentation-adapter goldens assert the exact generic
`ObjectCreate`, the now-derived 16-byte attached `ServerEvent`, movement update,
the now-derived 33-byte fields-`6/10/11` impact `ServerEvent`, combat update,
and `ObjectDelete` bytes. Poison Cloud must
not require an effect-stop golden because its authored branch does not send one.

One native scripting primitive used by that missing logic is recovered.
`sub_A0BD90` registers the literal `UnlockSecondCreature` to `sub_A050C0` at
`0x00A0BF23-0x00A0BF2E`. The primitive resolves the numeric script argument to
a simulation/player object and increments its field at `+0x1378`, capped at
`2`. Adjacent registrations map `UnlockNextAbility` to `sub_A05120` and
`UnlockOverdrive` to `sub_A05180`. This strongly explains Sage becoming the
second immediately switchable mission creature, while remaining a local
simulation-state operation: `sub_A050C0` makes no account or creature-unlock
submission and therefore does not prove persistent Sage ownership.

`UnlockNextAbility` is now instruction-level recovered. It resolves the script
argument to the simulation player, increments the `int32` at player offset
`+0x1370`, and replaces values above `5` with `9`. The ability-availability
predicate at `sub_9C2A60` compares the requested ability index against this
field, with special handling for indices `6`-`8` by active deck slot. The
simulation-player constructor initializes `+0x1370` to `9`; player snapshot
copy code explicitly copies it. A runtime dump of build 103's registered
`LabsPlayer` reflection metadata confirms top-level wire field `21` has offset
`4976` (`0x1370`), element size `4`, and count `1`. Earlier tests sent count `1`
initially and count `2` at the authored trigger. Count `1` exposed exactly the
red basic-attack icon and permitted index-0 attacks; count `0` hid every icon,
while count `4` exposed three. Build-103 HUD tracing now resolves the apparent
count-2 failure: `sub_422820` populates action-bar data from descriptor indices
`0,2,3,6,7,8`, while `sub_9C25C0` maps index `1` to `passiveAbility` and index
`2` to `specialAbility1`. Thus boundary `2` still shows only basic attack, and
the lesson must advance `2 -> 3` to expose the first pressable ability. A clean
build-103 run now confirms that boundary pair: initial count `2` retains only
basic attack, crossing the authored box with count `3` immediately adds the
second red action-bar icon, and key `1` emits `ActionUseCharacterAbility`
index `2`. Blitz visibly plays the activation animation. In the focused
no-enemy snapshot the request targets object `0`, so the server acknowledges
it without applying combat damage; a targeted-enemy run remains necessary to
recover the real damage, power, cooldown, and progression behavior.
Tutorial observation identifies this first ability as a cursor-directed
teleport. Decompiled build-103 logic establishes that a hostile target is
damaged and shocked; it is not an arrival-area attack. A controlled build-103 capture confirms
target object `0` is the ground-target form. Both `CursorPosition` and
`TargetPosition` carried `(229.66934,-87.43901,10.627861)`, about six units from
the previous movement goal `(231.76483,-93.1185,10.088001)`. darkspin now accepts
only finite destinations, advances its authoritative session position, and
responds with the action acknowledgement followed by `ObjectTeleport`. A clean
build-103 run showed the red teleport column and a normal landing at the echoed
destination. The remaining implementation order is authored range/navigation
validation, visible Shock, kill-reset progression, and combat XP. Live
build-103 tooltips identify Voltic Slash at 4-12 physical damage and Ride
the Lightning at 50 m range, 14-22 physical damage, and a three-second shock,
with an immediate cooldown reset when it kills its target. These binary-local
values supersede the older public 21-32 damage figure. The observed power cost
is 13 and the live tooltip reports a ten-second cooldown. The build-103 receiver for
`CooldownUpdate` reads exactly 36 payload bytes: object ID followed by eight
32-bit words forming a 64-bit runtime ability key and three 64-bit millisecond
intervals. HUD tracing identifies Special One's exact runtime key as
`0x43c6b5f7`. Removing invented hero attributes 23 and 24 corrected the former
zero-second tooltip; the normal tooltip path applies attribute 24 as cooldown
reduction, so `1` had meant 100% reduction. Native C1 tracing recovered a usable
relative sequence without the uninitialized source-clock converter: an all-zero
C1 snapshots the entry end to the client's current gameplay clock, and a second
C1 with zero source start plus duration `10000` extends that end by ten seconds.
One live application recorded `clock_now=36190`, reset `end=36190`, then
`end=46190`. The Ride icon visibly dimmed for approximately ten seconds, but its
clock began around 75% and filled to 100% because the entry start remained zero.
Retail footage starts this sweep at 0%, so the final path must also seed
`start=clock_now`. Native handler analysis shows a nonzero source start selects
the absolute start/end branch. A focused source-start `1`, duration `10000`
run produced no cooldown indicator at all, proving the unanchored remote
converter cannot treat `1` as client-now. That experiment is removed. The
supposed 25-byte `ObjectJump` clock sample was also invalid: build 103's
`sub_5390F0` reflection-decodes a 44-byte structure, so the unsupported encoder
and all live uses were removed. A type-0 `ActionCommandResponse` now uses the
recovered Ride runtime key and HUD index, but the live path retains the
confirmed two-C1 fallback while the real protocol clock epoch/unit is traced.
The same footage shows a tutorial arrow pointing at the newly
unlocked Ride icon until the player presses it. The extracted retail
`Tutorial_IntroAbilities.lua` supplies the presentation contract directly: it
waits two seconds and calls
`nUIManager.PlayAbilityBlink(nAbilityType.Ability_Enrage1)`. Static analysis of
the native wrapper shows this is HUD ability index `2`: play sets its `+0x0D`
flag and stop uses `+0x16`. Those fields are launch-DLL observation targets only.
The later clean live run confirmed objective `0xAC4273F3` does display the
arrow; the earlier 2.188-second observer sample was premature. The first
accepted Ride cast now sends that objective hidden/completed; live testing
confirms this stops the hint. Directly mutating HUD
flags from the DLL remains explicitly rejected. A rapid
second request is also rejected authoritatively without teleporting. The hero combatant update visibly reduces the cyan
power arc by 13. The focused ability snapshot now creates only stationary
object `14` at unlock time so targeting and Shock can be tested without
replaying earlier encounters. Decompiled retail
`ability_lightningrogue_active.lua` corrects the earlier area-damage assumption:
Ride validates one hostile target, finds a melee position, teleports, damages
that target, applies `ShockModifier`, and removes its cooldown only if that
target dies. A ground-target cast teleports but has no damage target. Authored
rank-one constants are 50 range, 10-second cooldown, 12 base mana, a 12-18
base damage range plus attribute scaling, `0.37` seconds to hit, and `0.77`
seconds to release. `modifier_shocked.lua` resolves by Game's
case-insensitive FNV-1 basename hash to `Modifiers/0xEABB0526.lua` (content
chunk 901, SHA-256
`f2a8a51e3cb4c52d255784c43f85edc89e0d949f423cbbd90f6177fa6876003a`) and
supplies descriptor 32 plus the three-second rank-one Shock duration and
immobilized/stunned priority. Build-103 handler analysis
confirms `ModifierCreated` consumes exactly 37 packed bytes: target object,
modifier GUID, instance ID, duration, overdrive, stack count, 64-bit start time,
source object, and bind byte. `ModifierDeleted` is target plus instance ID in
eight bytes. darkspin now encodes both exact shapes and schedules Shock deletion
after three seconds. The first live targeted attempt involved a manually
damaged enemy; it disappeared and Ride was immediately bright, which is
consistent with kill-reset behavior but does not yet isolate the cast from the
interleaved manual controls. A full-health survivor remains the visual-Shock
confirmation case.

The focused ability snapshot originally replicated all four objects `11`-`14`
during dungeon setup. This made later encounter mobs exist from the first frame
and allowed them to drift toward Blitz before their tutorial beat. They now
remain absent until the player enters an eight-unit perimeter around their
authored marker positions; the focused spawn is outside that perimeter. On
creation, tutorial enemies receive an explicit authored-position update and a
stopped locomotion goal at that same coordinate. Encounter AI sends a different
movement goal only when its authored pursuit begins. The normal opening pair
retains its separate authored spawn pause and pursuit path.

Build-103 runtime observation also recovered the tutorial's relevant account-XP
boundary from tuning property `0xC0B32F0F`. The vector begins `100, 200, 3000`,
and the native mapper advances only when XP is strictly greater than a bound;
level 2 therefore begins at cumulative XP `101`. Top-level `LabsPlayerUpdate`
fields `15` and `16` are the level and float XP fields. darkspin now encodes that
partial update and sends level `2`, XP `101` once when the opening encounter
stage clears. A focused live run visibly presented `You reached Crogenitor
Level 2!` and immediately spawned the next three-enemy wave. The passive lesson
is not attached directly to that level update. The build-103 Cryos audio
markerset defines
`vo_ship_tut_abilities_passive` as a client-owned (`serverOnly=false`),
one-shot `AudioTrigger_OnEnter` at `(118.88472,-85.76919,14.97249)`. Its trigger
is a 50x2x20 box rotated 82.28936 degrees around Z, forming a thin line across
the authored approach to the opening fight. A clean authored-start replay
crossed it on the accepted movement segment from `(124.21403,-95.78429)` to
`(99.72728,-93.68147)` and visibly presented the client's blue voiceover
callout before the enemies engaged. A focused boundary-3 run did not present it
while stationary. No server packet should duplicate this client-owned marker,
and waiting after the level banner is not a valid test for it.

Follow-up runs distinguish Ride's two request forms. Ground casts arrive as
`index=2, target=0` and correctly teleport without damage. Casting directly on
the mob arrives with target object `14`; the server teleports to that encounter
target and applies single-target damage and Shock. It must not infer a nearby
target for a target-zero ground cast. For fast iteration, crossing the ability
lesson creates one stationary 20-HP
`TutorialBasicPoisonNoOrbs` six units north of the player's accepted movement
goal and inserts the same dynamic definition into authoritative encounter
state. No other focused enemies are loaded or spawned. This is a test snapshot,
not the final authored encounter lifecycle. Retail mob creation also includes a
spawn-in particle effect. No effect was visible in a live run with the current
`character_teleport_in` state, so that animation message alone is insufficient;
recover the exact animation/`ServerEvent` ordering and add it independently of
aggro or locomotion. The extracted assets make
`generic_spawn.ServerEventDef` (`generic_appearance_cover`) the strongest
generic candidate. Build-103 dispatch maps application message `28` / packet
`0x9b` to `sub_539E90`; it reflection-decodes into a 0x98-byte ServerEvent
structure rather than consuming a fixed packet body. Its reflection schema
was recovered and a minimal `generic_spawn` asset/object/position event was
emitted successfully, but a live run showed no particle. The ineffective event
has been removed from the live spawn sequence; species beam-in and teleport
events plus their attach/force fields remain candidates.

The first unlock-time test mob rendered in the correct relative location but
immediately walked northeast, consistent with the client's locomotion state
retaining a zero goal. The focused spawn now explicitly sends the authored
position through `ObjectUpdate`, then `ObjectPlayerMove` with stopped flags
`0x20` and the same position as its goal, followed by
`character_teleport_in`. A deterministic packet test locks the six-message
create/state/update/stop/animation order and all three goal coordinates. Live
testing confirms the mob remains at that coordinate. A direct targeted Ride
then reduced it from 20 to 2 HP; rapid retries were rejected, and a subsequent
kill reset the cooldown immediately. The visible Shock effect remains
unconfirmed.

The extracted `ability_firstaggro_beamin_tutorial.lua` resolves the authored
spawn presentation: its first-aggro template enables visibility and plays
`character_teleport_in`; it does not add a separate ServerEvent effect.
Build-103 receiver `sub_53A1F0` reads the exact 25-byte animation body and
passes its 64-bit timestamp through the shared source-clock converter before
applying it. Build-103 sender `sub_A1FDD0` independently confirms the same
field order and a zero-capable final source/echo-gate value. The previously
cited `sub_A25410` belongs to the separate Build-127 decompile and must not be
used as Build-103 address provenance. The attempted preceding `ObjectJump` was not a valid clock anchor:
its build-103 handler expects a reflection-decoded 44-byte structure, not the
removed 25-byte body. The minimal `generic_spawn` event remains a valid negative
result. The next focused test should delay visibility/animation briefly after
creation so the creature model is present, while tracing whether the receiver
resolves the object and animation component.

The cooldown observer also explains the later 99%-to-100% sweep precisely. The
working relative C1 update stores `start=0` and `end=clock_now+10000`, so a
long-running client renders almost the entire radial as already elapsed.
Reusing the accepted type-0 response's darkspin-elapsed timestamp as C1
`source_start` did not establish a valid absolute start: source `284345` at
client clock `190492` converted to `103079215176`, and the icon showed no
cooldown at all. That live experiment was removed. A valid protocol clock
epoch/unit remains necessary; server process uptime is not interchangeable
with it.

The clock initializer is carried by logical application `GameState`, not
`Connected` or ObjectJump. Receiver `sub_537CE0` fixed-decodes the 25-byte
GameState body and passes its first 64-bit field to `sub_9D2830`, which stores
the remote origin consumed by `sub_9D2A00`. Runtime inspection confirms logical
GMS message `11` maps to transport ID `0x8a`. Substituting `0x8a` into darkspin's
`Connected` handshake response with zero-valued state/type fields reproducibly
diverted three clean launches to the squad START/menu path instead of
auto-entering the tutorial. The experiment is reverted: `Connected` remains
the empty `0x82` trigger for HelloPlayerRequest. GameState must instead be
populated with a valid dungeon state and Tutorial game type and sent at its
actual gameplay setup phase. Sending Dungeon/Tutorial GameState immediately
before world replication preserves tutorial auto-entry and initializes the
clock mapper (`remote_origin=24404`, `local_origin=3612`, scale `1`) in a clean
run. A first
Unix-millisecond epoch experiment converted to `0x0000000400000000`, not near
the millisecond gameplay clock, and the nonzero C1 update again displayed no
cooldown. That epoch and C1 field were removed. The next build uses the same
monotonic server-uptime millisecond epoch already emitted by RakNet's
connection-accepted exchange and every timed gameplay message.
The expanded read-only trace records the converter's current/origin values,
local origin, scale, and flags for validation.

In the same fresh run, landing Ride on the test mob visibly displayed the
stun/Shock effect. Modifier presentation is therefore confirmed.

The shared special-ability path later regressed to teleport-and-release only.
Targeted Ride the Lightning now retains the selected NPC through its authored
hit deadline, commits projected single-target damage from the landing point,
applies the confirmed three-second Shock to survivors, and resets both server
and client cooldown state on a kill. Target-zero ground casts remain
teleport-only. Fixed tutorial actors also retain their marker rotation through
the spawn plan and enemy create packet; the first encountered Poison No Orbs
marker is authored with Z rotation `-61.919216`, rather than a server-authored
north-facing default.

A later clean stable-path retest rendered the ten-second cooldown from roughly
50% to 100%, showed no spawn-in effect, and showed no Ride-use arrow. After the
valid GameState was added at dungeon setup, tutorial auto-entry still worked,
the mapper initialized, and the same authored `character_teleport_in` became
visible. Spawn-in presentation is therefore confirmed. The cooldown remained
partial in that running build because C1 still carried `source_start=0`; the
next focused build anchored C1 to the accepted cast's source time. Its trace
converted `source_start=53206` to local `start=32601` while `clock_now=32411`
and retained the full 10,000 ms duration; live observation confirms the sweep
now looks correct.

Call-level blink tracing found the intermittent arrow cause. The client-authored
one-shot `Tutorial_IntroAbilities` marker ran about 2.4 seconds after scene
change at the former debug spawn, before the ability-count update. Its Lua did
call `PlayAbilityBlink(2)`, but `sub_9C2A60` rejected index `2` because player
field `+0x1370` was still `2`; the one-shot was then consumed. The safe
`(237.008,-103.335,10.150)` spawn produces no early call. Walking north sends
count `3` and enters the client marker together; the traced call then returns
accepted (`0x10002`), the arrow is visibly present, and the accepted Ride cast
clears it. Future hidden objective updates are no longer sent at dungeon setup,
so their one-shot scripts are not consumed before their lesson.

The build-103 Voltic Slash tooltip reports a rounded 0.4-second cooldown. It is authored
as `LightningRogueBasic` (runtime hash `0x33cb6b29`) and arrives as ability index
`0`. Its Lua registration carries the exact `0.366s` cooldown; darkspin now
models it as a proper accepted basic attack with deterministic 8 damage,
server-side cooldown plus release enforcement, runtime key `0x33cb6b29`, and the
same cast-time-anchored C1 path. A live target cast reduced 20 HP to 12 and
traced `start=131563`, `clock_now=131380`, and `end=131963`. Two right-clicks
60 ms apart produced only one client request, confirming the client honors the
400 ms cooldown; the server enforces the same interval authoritatively.

A later Shift+left-click trace resolves the targetless held-input boundary.
Pressing and holding emitted one type-7 ability command with target `0`,
`unknown=1`, and the cursor copied into the target position. Releasing emitted
one 64-byte type-10 `ActionCancel`; no repeated type-7 commands were sent while
the button remained held. The server therefore retains only targetless basic
input and schedules subsequent swings at each selected authoritative maturity.
The type-10 command invalidates the retained generation without interrupting
an already committed swing. Targeted right-click attacks continue to use the
client's repeated type-7 requests.

The same generated reflection registration maps nested character
`mCreatureType`, `mAbilityPoints`, and nine `mAbilityRanks` to fixed offsets
`0x388`, `0x398`, and `0x39c`. darkspin's former offsets were all 0x30 too high;
they are now corrected and wire-tested. A live run accepted those corrected
records but, as expected from the boundary mapping, the old count `2` still
showed only basic attack and key `1` emitted no ability command.
Resending focused field `3` reflections for all three character records with
field `21 = 2` was accepted by the live client without a disconnect, but it
still did not add the second icon. That negative experiment was removed; it
does not justify a build-103 character-refresh packet. The first focused test
spawn, 6.5 units west of the lesson center, also entered the client's wider
authored dialog trigger immediately despite remaining outside darkspin's old
five-unit packet trigger. Asset inspection resolves the discrepancy: shape `0`
uses the authored `boxWidth=40`, `boxLength=1`, and `boxHeight=20`, rotated
`7.17999` degrees; `sphereRadius=5` is not the active shape. Both 15-unit and
22.5-unit west offsets also presented the lesson immediately, showing that the
40-unit width acts as the long local X extent while the one-unit length is the
thin local Y extent. A clean run at `(238.987,-138.448,10.150)` remained outside
without the `Press 1` lesson or a server milestone. Walking from
`(237.008,-103.335)` to `(230.343,-98.431)` then presented the client lesson;
the endpoint lies within the box after padding it by Blitz's authored
`0.5 * graphicsScale 1.6 = 0.8` bounding radius. darkspin now checks the complete
movement segment against that player-padded oriented box. Later call tracing
showed `(234.049,-99.257,10.150)` was not actually safe: the client's one-shot
`Tutorial_IntroAbilities` job fired about 2.4 seconds after scene change, before
the server unlock, and its availability check rejected HUD index 2. The focused
snapshot now uses the previously live-confirmed outside point
`(237.008,-103.335,10.150)` so walking north synchronizes the client marker with
the server-authored ability-count update.
This is a player reflection field, not an `AttributeDataUpdate` ID.

The exit route is also explicit. `BossSecurityTeleporter` marker `495984414`
at `(260.833, 232.784, 20.168)` targets destination marker `174193625` at
`(-347.576, -224.608, 10.088)`, placing the player beside the arena director.
The director marker `1730050752` triggers within radius `30`; its normalized
callback is `DirectorTrigger_SpawnBoss` beside event `horde triggered02`, and
exactly two listener-named `HordeSpawner_Register` records carry the same
level-scoped event key. The missing server implementation must still prove the
actual publish/consume dispatch. The
security-teleporter assets and chunks `144`/`349` now reveal the generic
enemy-presence predicate and teleport request path. They still do not prove
which subset of the Cryos encounter the retail server includes in that radius,
or the transport owner that replicates the resulting effects and modifier.

#### Packaged horde-gate modifier candidate

Two additional `content.db` chunks bound the teleporter/horde transition
without recovering its route owner:

| Chunk | Identity and registration | SHA-256 | Exact opcode inventory | VM-unsupported |
| ---: | --- | --- | --- | --- |
| `199` | resource `13736`, `Modifiers/0xB4EC30D4.lua`; global `nModifier_Horde_Gate_Teleporter`, registered as `HordeGateTeleporter`, callbacks `Activate/Deactivate` | `7674d1af31c91b3f4a5ddbad2c5a87c491a26199fb829846e268f87f18b2b4ec` | `ADD×3, CALL×19, CLOSURE×2, GETGLOBAL×37, GETTABLE×27, JMP×1, LOADK×8, LT×1, MOVE×35, MUL×3, NEWTABLE×5, RETURN×3, SETGLOBAL×1, SETLIST×2, SETTABLE×13, SUB×3` | `LT×1, MUL×3, SETLIST×2` |
| `426` | resource `13977`, `Modifiers/0xB9FB3F9C.lua`; global template `nModifier_Cage_Template`, methods `ShouldContinueCage/activate/tick/deactivate` | `aeb4377f3748b5b4dfb724f5750a482f9913a332489dcd00aab81dae2afd23b8` | `ADD×1, CALL×42, CLOSURE×4, EQ×4, GETGLOBAL×72, GETTABLE×62, JMP×9, LOADBOOL×4, LOADK×16, LT×1, MOVE×18, NEWTABLE×2, RETURN×9, SETGLOBAL×1, SETTABLE×24, TEST×3` | `LT×1` |

**Chunk 199 behavior (bytecode/native-proven).** Both callbacks have no
upvalues and there is no loop. `Activate()` obtains the agent and modifier
instance, reads float properties `0..2` as a requested destination, stops the
agent, and adds scoped `Immobilized=1` and `Intangible=1` attributes. It reads
the current position and clamps the displacement to `slideDistance=7` when the
requested point is farther away; the authored misspelled `sllideSpeed=14`
field is never read. It calls
`GetClosestPosition(agent, requestedOrClampedX, Y, Z)->(x,y,z)` before any
presentation, then notifies
`{simpleSwarmEffectID=SPID("character_teleport_beam_in_red"),
position={oldX,oldY,oldZ}}`, calls `TeleportObject(agent,x,y,z)`, executes one
untimed `coroutine.yield()`, and notifies the matching
`character_teleport_beam_out_red` event at the new position. `Deactivate()` is
empty: it does not remove either attribute or emit an event.

The exact native ABI is
`GetModifierInstanceID()->number`, `GetMyAgentID()->objectID`,
`GetFloatProperty(instanceID,index)->number`, `Stop(object)->boolean`,
`AddAttributeModifier(object,attribute,number)->handle`,
`GetClosestPosition(object,x,y,z)->(x,y,z)`,
`Notify(table)->()`, and `TeleportObject(object,x,y,z[,optionalBoolean])->()`.
The optional fifth native argument is a boolean whose higher-level meaning is
not named by the recovered C; the script omits it. Build-103 `sub_A09A90` performs
the authoritative position/facing mutation and calls `sub_A20110`, which sends
the 32-byte `ObjectTeleport` body (wire `0x90`: object ID, position, and
orientation). The pre/post effects use the already recovered reflected
`ServerEvent` path. `GetPosition` is registered to weak `sub_A00EE0`, whose
body is absent from `Game.c`; its three-result ABI is bytecode-proven, not
implementation-proven. The resume instant after bare `coroutine.yield()`
remains unresolved. The two scoped attributes are removed by native modifier
teardown after the empty Lua `Deactivate` callback, as proven below.

**Route boundary.** This is server-authoritative modifier behavior with
replicated teleport presentation, but its name is not a Cryos ownership link.
Neither the normalized level tables nor the recovered Cryos marker events
attach chunk `199` to marker `495984414`. A typed director may adopt it only
after validating the exact gate/phase owner, live deployed agent, finite
caller-supplied destination, reachable navmesh result, and cancellation across
the post-teleport yield. Preserve the authored order: resolve destination,
beam-in event, teleport, yield, beam-out event. Do not replace the proven
marker-target transition solely because the modifier is named
`HordeGateTeleporter`.

**Chunk 426 negative boundary (bytecode/native-proven).** Its
`ShouldContinueCage(object, target)` predicate returns false when the object is
dead, the current player count differs from private
`playersConnectedOnCast`, or `IsHordeActive()` is true; otherwise it returns
true. `IsHordeActive` remains the getter for reflected
`mbHordeSpawned`. This makes horde state a cancellation condition for a generic
cage modifier, not a horde starter, wave-clear predicate, teleporter owner, or
packet sender. Chunk `426` explicitly requires `Lua!Vector.lua`, now mapped to
exact packaged chunk `439`; chunk `199` implicitly depends on that same global
`nVector` surface without a `require` instruction. The resolved helper identity
does not create a Cryos ownership edge for either modifier.

#### Boss security teleporter state machine

The content edge is exact: the `BossSecurityTeleporter` noun names
`BossSecurityTeleporter.AIDefinition`; its same-instance companion names
`BossSecurityTeleporterPassive`; and chunk `144` registers that exact passive.
This is content-link proof for the security platform. It is not evidence that
chunk `199` owns the platform.

| Chunk | Group/instance; registration | SHA-256 | Exact opcode inventory | VM-unsupported |
| ---: | --- | --- | --- | --- |
| `144` | resource `13674`, `0xFC0FF8F5/0xA36FCF98`, `Modifiers/0xA36FCF98.lua`; template `nModifier_Security_Teleporter_Template`; registers `SecurityTeleporterPassive` and `BossSecurityTeleporterPassive` | `c2e727c6cf4c13708fa315321fb51f0402a3a4a9763eaf2f8d782c1eca3267f0` | `ADDx1, CALLx55, CLOSUREx13, EQx6, GETGLOBALx105, GETTABLEx82, GETUPVALx5, JMPx17, LOADBOOLx2, LOADKx21, LTx1, MOVEx36, NEWTABLEx1, RETURNx17, SELFx8, SETGLOBALx4, SETTABLEx42, TESTx4, TFORLOOPx1` | `LTx1, TFORLOOPx1` |
| `349` | resource `13896`, `0xFC0FF8F5/0x8B5016B1`, `Modifiers/0x8B5016B1.lua`; global `nModifier_Teleporter`, registered as `TeleporterModifier` | `9417dfe9163dbb857dc1b423d4e127170a6ba9e128f583d648fe7f258a1c1cd1` | `CALLx14, CLOSUREx2, GETGLOBALx35, GETTABLEx22, LOADKx5, MOVEx11, NEWTABLEx1, RETURNx3, SETGLOBALx1, SETTABLEx12` | none |

**Closure and callback ownership (bytecode-proven).** Chunk `144` has thirteen
child prototypes. The entrant predicate has no upvalues. The first trigger
callback captures that predicate; the third callback captures the first
callback; and `activate` captures all three trigger callbacks. `tick`,
`deactivate`, and the six generic/boss registration wrappers have no upvalues.
Chunk `349` has two direct, no-upvalue callbacks. A parity VM must preserve
those closure identities because the native trigger retains the three Lua
callbacks beyond `activate`.

**Chunk 144 activation and entry (bytecode-proven).** `activate()` stores the
agent as `private.teleportObjectID`, starts the active effect, and creates a
spherical trigger at the agent position with radius `2`. An entrant qualifies
when it is player-controlled or its owner is player-controlled. While the
private state is active, the first callback resolves the destination with
`GetTeleporterDestination(teleporterID)` and requests modifier GUID
`0x502f1932` on the entrant, with the teleporter as initiator, `kObjIDNone`,
rank/count `1`, and destination `(x,y,z)`. The second callback is empty. The
third checks `GetFirstModifierByGUID(entrant,0x502f1932)` and re-invokes the
first callback only when the result equals `kObjIDNone`. This equality is exact:
Lua 5.1 `EQ A=0` skips the following jump when the operands are equal, exposing
the captured callback call. A live matching modifier therefore suppresses a
duplicate request; callback three retries only after native removal. Physics
dispatch below proves callback one is entry and callback two is exit. Explicit
trigger teardown does not synthesize either callback.

**Native trigger dispatch now proves the third role.** `CreateTriggerVolume`
reads numeric `(x,y,z,radius)`, retains Lua callbacks `5..7` plus the creating
Lua thread ID, and returns the trigger ID. The physics adapter
`sub_A24F90` decodes contact mask bit `1` as callback one, bit `2` as callback
two, and bit `4` as callback three. All three are invoked through
`sub_A15F00(trigger, entrant)`, which supplies the retained Lua closure with
exactly `(triggerID, entrantObjectID)`. The dynamic sphere constructor used by
chunk `144` initializes filter mode `0`, so `sub_A16130` accepts every object at
the native layer; the captured Lua predicate is what restricts entry to a
player-controlled object or an object with a player-controlled owner.

For bit `4`, `sub_A16960` invokes callback three only when the trigger's delay
field `+48` is nonpositive. Chunk `144` creates the sphere with native delay
zero. Its dynamic-trigger repeat flag `+60` is also zero, so the callback is not
latched after its first successful invocation even though `sub_A15FB0` records
success at `+62`. Callback three is therefore the repeated contact/stay
callback, native-proven rather than inferred. Its cadence is each physics
contact notification carrying bit `4`; no fixed frame or time interval is
proven. This repeated callback still only retries after
`GetFirstModifierByGUID` returns `kObjIDNone` and does not itself supply the
missing chunk `349` release owner.

`DestroyTriggerVolume(triggerID)` resolves the retained record, unreferences
callbacks one through three from the creating Lua state, removes the physics
volume, and erases the trigger record. There is no explicit callback-two call
in `sub_A167C0 -> sub_A15D10`; a parity director must cancel the volume and its
closures without inventing an exit notification. Invalid IDs are native no-ops.

**Chunk 144 security loop (bytecode-proven).** `tick()` runs forever. Every
pass queries objects within radius `20` using
`nSporeLabs.organicDamageableObjectTypes` and counts only objects that are
alive, have team `0`, and have
`InvisibleToSecurityTeleporters == 0`. A nonzero count changes active to
inactive exactly once: remove the current effect, add `powerDown`, wait `1s`,
and add the inactive effect without explicitly removing the replaced power-down
handle. A zero count changes inactive to active: remove the current effect, add
`powerUp`, wait `1s`, remove it, add the active
effect, and set state active. Stable states wait `0.5s`. The authored
`teleportDeactivationTime=1` field is never read; the active-to-inactive branch
uses the field named `teleportActivationTime`. `deactivate()` destroys the
trigger volume but does not explicitly remove the current effect.

The boss registration supplies active/inactive/power-up/power-down assets
`zelem_boss_teleporter`, `zelem_boss_teleporter_inactive`,
`zelem_boss_teleporter_powerup`, and `zelem_boss_teleporter_powerdown`.
Effect selection is bytecode-proven server state with replicated client
presentation; it does not make the client authoritative for the predicate.

**Lua-visible ABI (bytecode-proven unless called out below).** The predicate
uses `IsPlayerControlledObject(objectID)->boolean` and
`GetOwnerID(objectID)->objectID`. Lifecycle uses
`GetAgentID()->objectID`, `GetPosition(objectID)->(number,number,number)`,
`CreatePrivateTable()->table`, `GetPrivateTable()->table`,
`AddEffect(objectID,asset)->effectHandle`,
`RemoveEffect(objectID,effectHandle)->()`, and
`WaitForXSeconds(number)->()`. The scan uses
`GetObjectsInRadius(x,y,z,radius,typeCollection)->table<objectID>`,
`ipairs(table)->(iterator,state,control)`, `IsAlive(objectID)->boolean`,
`GetTeam(objectID)->number`, and
`GetAttributeValue(objectID,attribute)->number`. The bytecode consumes exactly
those result shapes; it does not by itself prove the native implementation of
every binding.

**Cryos guard membership (content/bytecode-proven spatial and exclusion
set; first-aggro selector edge still open).** The retained AI marker set has SHA-256
`5067ab36c70cf7aa726d755f39eccf9c7bcdec5f54f6d3eca289e5023cc6698f`.
Its fixed marker records use stride `0xc8`, not the current normalized
importer's `0xd0`; therefore normalized rows after ordinal zero are shifted and
must not be used as coordinates. Direct decoding yields exactly five fixed AI
placements inside chunk `144`'s radius `20` around
`(260.833,232.784,20.168)`:

| AI ordinal | Marker / noun | Position | 3D distance |
| ---: | --- | --- | ---: |
| `0` | `TutorialSpecialOne.Noun-1` / `TutorialSpecialOne.Noun` | `(268.716,231.483,20.394)` | `7.99` |
| `1` | `TutorialBasicPoison.Noun` / `TutorialBasicPoison.Noun` | `(266.769,238.135,20.102)` | `7.99` |
| `14` | `TutorialBasicPoison.Noun-31` / `TutorialBasicPoison.Noun` | `(266.950,228.763,20.108)` | `7.32` |
| `15` | `TutorialBasicRanged.Noun` / `TutorialBasicRanged.Noun` | `(267.985,235.413,19.943)` | `7.62` |
| `21` | `TutorialBasicRanged.Noun-6` / `TutorialBasicRanged.Noun` | `(264.021,226.897,20.088)` | `6.69` |

The other 22 fixed placements are at least `88.11` units away. This proves the
only authored fixed-enemy candidates for the passive's spatial query. Focused
noun payloads for all three families begin with little-endian type GUID
`0x06c27d00`. Recovered `GlobalDefinitions` chunk `659` defines that GUID as
`kType_Creature` and defines
`nSporeLabs.organicDamageableObjectTypes={kType_Creature}` exactly. The type
filter is therefore bytecode/content-proven for all five placements, not an
inheritance-name inference.

All three AI definitions name `nBehavior_Invisible`, a tutorial phase, and
`FirstAggro_BeamIn_Tutorial`; Poison and Ranged also name `nBehavior_Wander`.
The exact behavior body is chunk `865`, resource `14443`, group/instance
`0xC130A42A/0x104FCC83`, `behaviors/0x104FCC83.lua`, SHA-256
`dcdab6384645e166b5dd88e23c15246cf421c72946d78b5eaba686976dd40ecf`.
It installs no-upvalue `Activate`, `Tick`, and `Deactivate` closures on global
`nBehavior_Invisible`; its exact inventory is `CALLx13, CLOSUREx3,
GETGLOBALx21, GETTABLEx18, LOADBOOLx2, LOADKx6, MOVEx10, NEWTABLEx1,
RETURNx4, SETGLOBALx1, SETTABLEx3`, with no current VM-unsupported opcode.

**Current playable policy.** The recovered server/director edge between object
`42` and the guard reveal remains absent, so darkspin uses that defeat as the
explicit transition that deploys the five proven placements above. They are
authoritative objects `43..47`; chunk 144 remains inactive while any is alive
and powers up only after their clear. Cryos itself already constructs marker
`495984414`, runtime object `28`, so setup sends only its inactive/active effect
state and does not create a second overlapping teleporter noun.

**Invisible lifecycle (bytecode-proven).** `Activate()` obtains the behavior
object, calls `SetIsVisible(object,false)`, adds `Intangible=1` and
`InvisibleToSecurityTeleporters=1`, and stores the returned modifier handles in
thread-data integer slots `1` and `2`. Chunk `659` fixes the latter attribute
selector at `112`. `Tick()` calls `WaitForever()`. `Deactivate()` makes the
object visible, reads both handles, and removes both attribute modifiers. The
calls consume object IDs, attribute numbers, numeric modifier amounts/handles,
and integer thread-data slots; none returns a script-visible result except
`GetMyObjectID`, both `AddAttributeModifier` calls, and both `GetInt` calls.
This is server-authoritative object state with visibility replication, not
client presentation ownership.

**Invisible native boundary (native-proven where bodies survive).**
`AddAttributeModifier(object,attribute,amount[,optionalBoolean=true])->handle` is
registered to `sub_A04000`; it returns `-1` when the object or attribute
container cannot be resolved and records handles for ability/behavior cleanup.
The higher-level meaning of the optional boolean is not named by the recovered
C. `RemoveAttributeModifier(object,handle)->()` is `sub_A04110` and also
removes that cleanup registration. Thread-data `SetInt(slot,number)->()` and
`GetInt(slot)->number` are `sub_9FF100/sub_9FF080`; they address the current
thread's integer array, ignore out-of-range writes, and return `0` for a
missing/out-of-range slot. `WaitForever()` is `sub_9FFF80`; it clears the
current continuation timing fields and installs an indefinite resume callback.
`SetIsVisible` and `GetMyObjectID` are registered to weak
`sub_A02B80/sub_A00390`, whose bodies are absent from `Game.c` but are
recovered in the retained IDA listing
`bin/game/logs/ida-dump-weak.log` (SHA-256
`37785501662d9bd70240253b7542d09dde9a30874a5ebe39bfd2371ce21ca565`).
`GetMyObjectID()->objectID` reads object field `+16` from the current Lua
execution context and returns `0` when no context exists; it always pushes one
numeric result. `SetIsVisible(objectID,boolean)->()` resolves argument 1, coerces
argument 2 to boolean, and writes object byte `+95` when the object exists. It
returns no Lua values and contains no sender call. The mutation is
native-proven; any visibility packet comes from a later simulator
state-diff/publication path, not directly from this binding.

The behavior callback wrapper is fully present and fixes thread ownership and
ordering. `sub_A68F10` resolves the object's single Lua thread ID at object
offset `+676`, points thread offset `+224` at a behavior-local thread-data block,
calls callback index `1` (`Activate`) synchronously, and then prepares callback
index `2` (`Tick`) with `sub_8F0EC0`; preparation does not resume it. A fresh
behavior context uses callback indices `1/2/3`, records the object ID, and owns
exactly 16 integer thread-data slots. `sub_A68E10` is the matching deactivation
wrapper: it resolves the same object-owned thread, invokes callback index `3`
(`Deactivate`) first, resets the Lua thread to inactive state, and clears its
thread-data pointer. It does not allocate or destroy the object-owned thread;
full object teardown remains responsible for destroying the `+676` record.
Thus chunk `865` must make the object visible and remove handles from slots
`1/2` before its indefinite Tick is cancelled/reset. The native wrapper ignores
the callback return, so typed parity should make this cleanup guaranteed rather
than leave invisibility or either attribute latched after a script error.

**Team deployment (content/native-proven).** Build-103 object deployment
`sub_9D1DF0` resolves the noun definition and copies its byte at `+224`
directly into the new runtime object's team byte at `+84`; `GetTeam` reads that
same unsigned runtime byte. The focused decoded noun payloads for Ranged,
Poison, and Special One all contain `00` at fixed noun offset `0xe0` (`224`).
Their deployed default is therefore team `0`. A later `SetTeam` call could
still override it; no such override is currently content-linked to these five
placements.

**Single-phase AI graph (content/native-proven structure; scheduler partly
recovered).** Each AI definition has phase count `1`. Its sole linked phase is
a type `0x30728ce7`, group `0`, same-instance resource:

| Enemy | Instance / package ordinal | Decoded SHA-256 | Sole phase ability |
| --- | --- | --- | --- |
| Ranged | `0x3aec5fe2` / `2665` | `c2dc65edcc88bbc324c18401d356d213ad67c761437c7cf7167f1faf859c1ced` | `TutorialPlasmaLightning` |
| Poison | `0x09a3ac7f` / `6675` | `2827c3df46dd0f9fff912c42bc98b82d03d960a12b216d3d349e5c79b660180d` | `TutorialPoisonMelee` |
| Special One | `0xcc7ecbe0` / `3197` | `56acdd76cea236ef9cbdaa4b7ca3aeef4dcfa4156b63fbd127128e2105f575d6` | `BurstShot` |

Native `sub_A22A50` resolves the AI definition from noun field `+212`, reads
the phase count at AI offset `+4`, indexes 28-byte phase entries, and selects a
preferred start phase only when the linked phase byte at `+12` is nonzero.
All three focused phase payloads have byte `+12 == 0`; with one entry, the
initialized index `0` remains the only phase. Thus there is no alternate
authored phase that can own invisibility removal. The AI resource itself owns
the `nBehavior_Invisible` behavior reference and first-aggro ability reference;
Ranged and Poison additionally own `nBehavior_Wander`, while Special One does
not.

**Behavior-tree transition (native/content-proven structure; exact stimulus
selector still unresolved).** The reflection metadata fixes the relevant AI
definition offsets, correcting earlier semantic guesses: `firstAggroAbility`
is `+0x34`, `firstAggroAbility2` is `+0x44`, `firstAlertAbility` is `+0x54`,
`subsequentAggroAbility` is `+0x64`, `preAggroIdle` is `+0x7c`,
`preAggroIdle2` is `+0xcc`, `combatIdle` is `+0x11c`, `combatIdle2` is
`+0x1c0`, and `passiveIdle` is `+0x170`. In particular, scheduler reads at
`+0x11c/+0x1c0` are combat-idle variants, not first-aggro/first-alert fields.
The focused payloads place `nBehavior_Invisible` in `preAggroIdle`; Poison and
Ranged place `nBehavior_Wander` in `passiveIdle`, while Special One leaves
`passiveIdle` empty.

The creature behavior tree is an ordered 12-child selector rooted at static
node `0x11814e8`. Its eleventh child (`0x11813c8`) is the authored
pre-aggro/passive Lua-behavior leaf. Native `sub_9E3D00` selects
`preAggroIdle` or `preAggroIdle2` according to combatant byte `+1416`.
Predicate `0xA21E20` admits that leaf while `passiveIdle` is nonempty, or while
the selected pre-aggro field is nonempty and `sub_9E3C70` does not expose first
aggro as consumed. Activation `0xA214D0` starts the selected pre-aggro behavior
before consumption; after consumption it instead selects nonempty
`passiveIdle`. Tick `0xA215D0` only tests whether the object-owned Lua behavior
thread remains active, so a running `WaitForever()` Invisible behavior does not
self-switch merely because byte `+1365` changes.

Selector replacement ordering is exact. `sub_A329B0` calls `sub_A31F70` on the
old child before assigning and descending into the new child; that teardown
invokes callback-table slot `+12`, which reaches `sub_A68E10` and chunk `865`
`Deactivate`. Only afterward does `sub_A329B0` call the new leaf's callback-table
slot `+4` Activate. Consequently, whenever a higher-priority aggro/ability leaf
replaces Invisible, visibility restoration and both attribute-handle removals
are ordered before the replacement leaf's Activate. After that leaf exits,
Poison/Ranged can re-enter the same authored leaf as `nBehavior_Wander` through
`passiveIdle`; Special One cannot, because its `passiveIdle` is empty.

The remaining edge is narrower but still material. `sub_9E4640` synchronously
posts stimulus `0x20`, marks first aggro consumed at byte `+1365`, and records
the target, but no recovered `sub_A64540` consumer directly fetches mask
`0x20`. The static tree's event-expression path is now recovered and **rules
out** the earlier hypothesis that it directly selects a `0x20` first-aggro
leaf. `sub_A21800` passes the active 64-bit stimulus mask returned by
`sub_A64810` into `sub_A329B0`; `sub_A322A0` applies each node's 32-bit state
gate and then calls `sub_A32110` on the node's 48-byte event expression at
`+0x40`.

The event expression accepts only when `(events & mask64@+0x48) ==
required64@+0x40`, an optional any-bit gate at `+0x50/+0x58` succeeds, and an
optional all-bits exclusion at `+0x60/+0x68` does not match. Of the root's 12
children, only child indices `3`, `4`, and `5` have nonzero event expressions:
their required and mask pairs are exactly `0x4/0x4`, `0x1/0x1`, and
`0x10/0x10`. Every other child's full event expression is zero, including the
pre-aggro/passive leaf at index `10`; no child requires or masks `0x20`.
Therefore the first-aggro ability is selected by a still-unrecovered engine
edge outside this static node event gate, not by a missing interpretation of
the gate itself. This does not weaken the proven old-child Deactivate before
new-child Activate ordering once replacement is requested, but it still
prevents assigning that request to the same evaluation as stimulus `0x20`.
The temporary deadline at behavior-tree data `+1824` also masks
`sub_9E3C70` after the alert-style `0x200` path. Do not equate that deadline or
the combat-idle `+0x11c/+0x1c0` leaves with first aggro.

**Direct aggro-slot consumer audit (native-proven negative; indirect/server
owner unresolved).** Runtime object `+684` is the agent-brain instance, not the
AI definition. Brain initialization `sub_A22A50` copies the noun definition's
AI resource link at noun `+212` into brain `+8`, resolves that definition, and
initializes its phase table. Runtime object `+688` is the separate agent
blackboard. This fixes the pointer chain used by the behavior wrappers and
prevents treating arbitrary object-`+684` reads as AI-definition fields.

A focused audit enumerated all build-103 code references to resource resolver
`sub_9C8E70`. There are `281`. The agent-brain subset—resolver calls with a
nearby object `+0x2ac` load—directly reads only `preAggroIdle`/`preAggroIdle2`,
`combatIdle`/`combatIdle2`, and `passiveIdle` at `+0x7c/+0xcc`,
`+0x11c/+0x1c0`, and `+0x170`. A second scan followed every resolver call to
its return or at most 120 subsequent instructions and found no instruction-
linked inline-string access to AI `firstAggroAbility`, `firstAggroAbility2`,
`firstAlertAbility`, or `subsequentAggroAbility` at
`+0x34/+0x44/+0x54/+0x64`. Combined with the static-tree result above, build
103 contains no recovered **direct** path from stimulus `0x20` to the
`firstAggroAbility` string.

This is a bounded negative, not proof that the retail owner never existed. A
helper receiving an already selected string, a reflection-driven lookup, or
unrecovered authoritative server code can evade direct offset tracing. The
defensible typed boundary is therefore two operations: accept and latch the
first hostile aggro insertion, then have an authoritative AI/director owner
select the authored first-aggro ability exactly once. The client presentation
timeline begins only after that second operation; it must not be inferred from
the static tree or assigned an invented next-tick delay. Retained diagnostics
are `bin/game/logs/ida-brain-ai-resolvers.log` (SHA-256
`b67ba096c9b85648c1dad7d714f7e4e83fe15e6adb40707970cbe69eacb1f991`)
and `bin/game/logs/ida-resolved-inline-fields.log` (SHA-256
`0e7c0ea5f2ae726ec5628bf24d2eb8fbbd0360f3640eac8c1374849c99bb564f`).

**Ability registration and start boundary (native-proven; authoritative
selector unresolved).** `nAbility.RegisterAbility` and
`nModifier.RegisterModifier` both bind to `sub_A67D70`. Registration
case-insensitively hashes the authored name, creates the shared reflected
`ability` definition only when it is not already registered (or when the Lua
test override is active), stores that GUID at definition `+96`, and retains the
registration string at `+100`. It does not create an instance, start a job, or
publish a packet. This proves that loading chunk `502` merely makes
`FirstAggro_SpecialOne` addressable; it cannot be the missing activation
producer.

The explicit Lua start API is separately bound as `nAbility.RequestAbility`.
`sub_A67920` decodes exactly nine positional arguments: ability ID, agent ID,
target ID, target-position `x/y/z`, rank (default `1`), one byte-sized request
field (default `0`), and an opaque namespace/context integer (default `-1`).
It returns no Lua result and calls `sub_9E1540`. That routine first invokes the
single generic admission routine `sub_9E0660`; only result `1` proceeds. The
admission checks a resolvable definition, rank `1..8`, player-slot availability
where applicable, required target state, blocking attributes, cooldown, mana,
range/hit predicates, and an already-running blocking ability. Other results
represent rejection, cooldown, pursuit, or hit failure and do not create the
ability instance. This is the same validation boundary already proven for
chunk `134`, but it is now explicitly separated from modifier admission:
`nModifier.RequestModifier` alone enters `sub_9E16C0` and its activation-type
switch.

On immediate accepted ability execution, `sub_9E0AE0` allocates the shared
2,048-entry runtime-instance type, stores definition, agent, target,
target-position, rank, request byte, namespace/context, and simulation-clock
timestamps, links the instance into the agent's active-ability collection, and
calls `sub_9DDA00`. `sub_9DDA00` allocates a Lua thread, binds the instance as
its native context, records the thread handle at instance `+40`, and starts
registered callback slot `2`. Coroutine completion therefore belongs to the
ability instance/thread owner; deletion, cancellation, or agent teardown must
invalidate that instance rather than merely canceling presentation timers.
The deletion path is also explicit. Both ability and modifier
`MarkForDelete` bind to `sub_A65800`, which resolves the supplied 32-bit
instance ID and calls `sub_9E12D0`. If callback-in-progress guard byte `+356`
is set, deletion is deferred by setting byte `+357`; otherwise
`sub_9E11F0` runs immediately. Its first step, `sub_9E09D0`, invokes registered
callback slot `3` when present, releases the Lua thread, clears instance
thread handle `+40`, and releases the retained attachment handle at `+52`.
Only afterward does `sub_9E11F0` remove owned effects, unlink the instance from
the agent's active ability/modifier collection, and return its generational
handle to pool type `216`. This proves one instance-owned cancellation path
and its callback-before-unlink order. It does not identify which missing
Quadra phase owner requests that cancellation on death, restart, peer teardown,
or natural route exit.
Definition byte `+90` is now identified rather than treated as an unknown
reflected flag. During Lua registration finalization, `sub_9DA000` sets it to
the cached result of looking up method `IsAbleToHit`; adjacent bytes `+89`,
`+91`, `+92`, and `+93` cache `GetWarmupData`, `AbsorbDamage`,
`ImmuneToDamage`, and `CanUseModifier`. When a definition supplies the custom
hit method, accepted `sub_9E1540` does not create its instance immediately: it
posts ability/agent event records through `sub_9DF940` and `sub_9DF8A0`, adds a
behavior event through `sub_A20C30`, and stores the pending ability, target,
position, rank, request byte, and namespace/context on that event. The later
behavior-event consumer remains unresolved.

This alternate branch does **not** apply to the Quadra fixture.
Bytecode-proven chunks `502` and inherited template `848` contain no
`IsAbleToHit` method or constant. Their registered definition therefore has
byte `+90 == 0`, so an accepted explicit request reaches `sub_9E0AE0` and
starts callback slot `2` immediately. The remaining uncertainty is who issues
that accepted request after first aggro, not whether chunk `502` waits in the
custom-hit behavior queue.

Unlike `sub_9E0FB0`, the modifier creator, the ability creator never calls
`sub_A1FB20` and therefore does not publish logical message `36` / the
`ModifierCreated` packet. It starts Lua callback slot `2`; the callback's own
cinematic, visibility, and animation natives are the first proven replicated
effects for `FirstAggro_SpecialOne`. Consequently a typed server intent for
the missing owner is `startAIAbility(agent, abilityGUID, target, position,
rank, phaseGeneration)`, not `requestModifier` and not a synthetic
ability-created packet. Director validation must require the still-live
Quadra role, the separately accepted and latched first hostile aggro target,
an unconsumed once-only first-aggro token, the authored registered GUID, no
conflicting active instance, and the current phase generation. The build-103
binary contains only the Lua `RequestAbility -> sub_9E1540 -> sub_9E0AE0`
direct call chain; it contains no recovered internal caller that joins stimulus
`0x20` to this start path. That missing selector remains server-authoritative
homework rather than permission to route it through the client request API.

`OverrideFirstAggro` only writes an override object ID at AI runtime offset
`+1412`; `IgnoreFirstAggro` sets byte `+1365` and posts engine event `672`.
Neither directly calls `RemoveAttributeModifier` or stops the Invisible thread.
Retained native diagnostics are
`bin/game/logs/ida-invisible-scheduler.log` (SHA-256
`c2876b96454f5a4136d0630ea63f5eb78de9ceeb91bd8869dac5185e887eadd99`),
`bin/game/logs/ida-ai-field-xrefs-all.log` (SHA-256
`57c87e11a7572aca073677003a9277876ad83a7975d166fccdbf876ad68ea2ad`),
and `bin/game/logs/ida-lua-behavior-wrappers.log` (SHA-256
`30f782dcf35fecbcca3855bfd2e67b7e95ecae392c683e55d47b10ca35fe79fd`).
The exact gate evaluator is at `Game.c:1462733-1462763` and
`Game.c:1462621-1462637`; the static-node words are retained in the first
diagnostic above.

`FirstAggro_BeamIn_Tutorial` chunk `509` specializes shared template chunk
`848`. The template makes the agent visible, clears stealth, plays
`character_teleport_in`, and waits for the authored animation/effect. It does
not remove attribute modifier `112`. Therefore a behavior-tree replacement
that invokes chunk `865` deactivation is required before a living guard can be
counted by the security passive. The replacement's teardown-before-Activate
order is native-proven. The exact stimulus-`0x20` selector edge associating it
with the authored first-aggro ability remains unresolved, and is now proven
not to be the static tree node event expression.

Native `GetTeam` (`sub_A02DA0`) reads the runtime object's unsigned byte at
`+84`; a missing object produces no Lua result. Native `GetAttributeValue`
(`sub_A041A0`) reads the runtime attribute container and returns `0` when the
object/container is absent. Native `nObjectManager.GetObjectsInRadius`
(`sub_A10260`) treats argument five as a table filter, spatially queries at
most 256 candidates, and retains only objects whose runtime type occurs in
that table. Thus a typed
director must validate each candidate's deployed type, team byte, alive state,
and current behavior-owned security-invisibility handle rather than
substituting distance alone. Type, default team `0`, and exclusion ownership
are now proven; the stimulus selector that causes the proven teardown and a
live five-guard clear remain the exact open predicates.

Focused decoded evidence under `bin/game/logs/cryos-route` is:

| Family | Noun ordinal / SHA-256 | AI ordinal / SHA-256 |
| --- | --- | --- |
| `TutorialBasicRanged` | `2667` / `f3530e2d3f0d7f346b906ace41dac76457fc4c1cab44d5f175aaf69e3362c82c` | `2669` / `d7842ae3a1a0f05692b84ca68eee1b42cfdf02a1842c3ad54c89f44da002cd04` |
| `TutorialSpecialOne` | `3199` / `7f701d8df5762af189f9d5f6c325fee149d2009dd5a21fde3ff4149fa52d739a` | `3201` / `ea4bfb86f50b383dcdd875d1a4d65ada9c45ebd64d6b41190ccf1a2d49894693` |
| `TutorialBasicPoison` | `6673` / `ab0ed9b141763a95afae20e6699f6e0f07e7a8ad81ffdc0f5c9b0be674336d14` | `6671` / `b9b05103bc2d89463d964f8e070cddfa1187ba92a2b17fc7a0251f15997e1fca` |

**Requested teleport modifier (bytecode/native-proven).** Native
`RegisterModifier` case-insensitively FNV-1 hashes its registration string and
stores that GUID at modifier-definition offset `+96`.
`SPID("TeleporterModifier") == 0x502f1932`, and
`GetFirstModifierByGUID` compares its argument with that field. Therefore chunk
`144` requests chunk `349`, not chunk `199`.

Chunk `349` reads destination properties `0..2`, stops the entrant, adds
`Immobilized=1`, sets `character_teleport_out`, waits `0.5s`, calls
`TeleportObject(entrant,x,y,z)`, performs an authored zero-duration wait, sets
`character_teleport_in`, and waits another `0.5s`. Its deactivate callback is
empty and does not explicitly remove immobilization; native modifier teardown
removes the retained handle after that callback. The three waits all belong to
the yielding **Activate** callback; chunk `349` registers no Tick callback.
Coroutine completion is therefore not evidence that the modifier is deleted.
For the Cryos platform, marker `495984414` is runtime object `28` and its
authored destination is marker `174193625` at
`(-347.57553,-224.60829,10.08803)`. **Native/replication-proven ordering:** the
exact request is
`RequestModifier(entrant,teleporter,0x502f1932,kObjIDNone,1,x,y,z)`;
`ModifierCreated` (`0xa2`) is published before Activate, followed by teleport-
out animation (`0xa5`), the authoritative teleport (`0x90`) after `0.5s`, the
authored zero-duration yield, teleport-in animation (`0xa5`), and its final
`0.5s` wait. Only a separate explicit teardown publishes `ModifierDeleted`
(`0xa4`); Activate return and zero duration do not do so.

**Teleporter live modifier contract (local-live/native-proven).** The focused
audit in `notes/tutorial/teleporter-live.md` confirms that the created instance
ID is a 32-bit generational handle `(generation << 16) | slot` from the
2,048-entry modifier pool. The observed `ModifierCreated` carries duration
`0`, overdrive `1`, stack `1`, source object `0`, and bound flag `1`, followed
in order by teleport-out, authoritative `ObjectTeleport`, and teleport-in.
Chunk `349` completion emits no deletion and does not remove its scoped
`Immobilized` contribution. Only a distinct lifecycle release invokes native
teardown, removes that contribution, and publishes `ModifierDeleted`. This
live result validates the already recovered lifecycle split without proving
which retail server owner performs ordinary first-instance release.

Recovered `GlobalDefinitions` chunk `659` proves
`nActivationType.Unique == 1`: its table constructor assigns constant
`Unique` the numeric constant `1` before publishing `nActivationType`.
Native `RequestModifier` switches on the registered definition's activation
type at `+400`; case `1` calls `sub_9E12D0` for every existing same-GUID
instance and only then creates the replacement through `sub_9E0FB0`. Thus a
subsequent request for `TeleporterModifier` has a native-proven
deactivate/cleanup/remove-before-create boundary. It does **not** prove when
the first request is normally released: neither chunk `349` nor caller chunk
`144` retains the returned instance ID or calls modifier `MarkForDelete`, and
the inspected native path does not connect Activate coroutine return to
`sub_9E12D0`. The ordinary release owner remains unresolved and must not be
modeled as an implicit `1.5s` lifetime. Session/phase teardown and a later
Unique replacement are proven cleanup paths; any additional engine owner
remains unresolved.

A direct IDA xref audit strengthens that negative boundary. The retained audit
is `bin/game/logs/ida-modifier-lifecycle.log`, SHA-256
`3b5345f56a5203919f2fad9843daa532e31512b0189ab160891e1809ea912297`;
the focused instruction ranges are `bin/game/logs/ida-lifecycle-gap.log`,
SHA-256
`78db277a55866790419e225e375692bd3def88f879eea62bb4ab23bdebba4645`.
No direct caller of `sub_9E12D0` is the generic Lua scheduler or the normal
return path in `sub_8F0FB0`. The previously unclassified resume at
`0x00a000f0` is inside `nThread.CreateThreadForObject`: it performs the initial
resume of the newly allocated object-owned worker and submits every non-error
result, including normal completion, to the thread manager. It is not a
modifier release callback.

The previously unclassified deletion call at `0x00a20df7` is also bounded. Its
adjacent behavior path prepares `nBehavior_MoveToRange`, reads one retained
modifier ID from its behavior state, resolves that instance, and calls
`sub_9E12D0` only when instance byte `+44` is nonzero; every branch then clears
the retained ID.
That is a behavior-owned conditional cleanup slot, not a scan for completed Lua
threads and not a generic `nDeactivationType.Default` policy. The complete
direct-call set still contains explicit Lua `MarkForDelete`, replacement/
request policies, retained owner slots, and broader owner teardown, but no
Activate-return-to-delete edge. This does not prove that an indirect event or
unrecovered external route owner cannot release chunk `349`; it proves that
normal Lua completion alone is insufficient in the inspected native paths.

Chunk `349` also omits `deactivationType`. Chunk `659` constructs the exact
table `nDeactivationType={Default=0,OnAgentDestroyed=1,OnAgentDeath=2}`, so the
teleport modifier uses `Default == 0`; it is not authored as either agent-
lifecycle policy. By contrast, chunk `144` explicitly assigns
`OnAgentDestroyed` to the long-lived security-teleporter passive. This proves
the two modifiers have different release policies but does not name the native
meaning of `Default`. Callback three's retry-on-`kObjIDNone` condition supports
the expectation that chunk `349` eventually disappears while an entrant
remains inside, but does not prove the deletion deadline or owner.

The native reflection initializer independently registers the literal
`deactivationType` at modifier-definition offset `+0x194`. This confirms that
the field is engine-consumed definition data rather than a Lua-only annotation;
the xref audit still finds no scheduler-return branch that interprets its
default value as immediate removal.

A whole-executable displacement audit is retained as
`bin/game/logs/ida-deactivation-policy.log`, SHA-256
`5ec5d9f384aec06385df5b35a7b8b6d8bf88fb2f96382baf5d0e371e68744420`.
It contains `447` raw `+0x194` instructions, most belonging to unrelated
structures or stack frames. In the modifier-definition construction/copy
region, the only exact accesses initialize the field to zero and copy it with
the rest of the definition; no direct post-registration read is present.
`RegisterModifier` reflects the Lua definition through generic `sub_A0F750`, so
an indirect reflection/event consumer is not excluded. This is native negative
evidence against a simple `switch(Default)` completion timer, not proof that
`deactivationType` is unused. Full object teardown independently removes every
active owned modifier through `sub_9D1B30 -> sub_9DEA80`, regardless of the
unresolved ordinary-release policy.

The authored duration input is now also bytecode-proven absent. Recovered
utility chunk `827` (resource `14404`, SHA-256
`59566e1528d3850efd8dc905aa6e12dd2cf113d06fa750bef35cbe82ed6a6126`)
defines `nAbility.GetDuration(definition,agent,initiator,rank)`: it first calls
`_G[definition].GetDuration(agent,initiator,rank)` when present, otherwise
rank-resolves `_G[definition].duration`, and otherwise returns numeric `0`.
Chunk `349`'s `nModifier_Teleporter` table contains neither field, so native
construction stores duration `0` at modifier-instance `+352`. Native
`nModifier.GetDuration` exposes that field as seconds, and
`GetRemainingDuration` computes `max(duration + startTime - now, 0)`.
This rules out a positive authored expiry timer for the teleporter, but does
not by itself identify the engine owner that eventually releases a zero-
duration instance. Chunk `827` requires hashed
`0x3681d755!GlobalDefinitions.lua`; the package/linker audit and successful
compile probes map that exact authored import to recovered chunk `659`. The
bootstrap body is therefore present and resolved; it does not supply the still
missing owner that releases a zero-duration teleporter instance.

**Native thread/release boundary (native-proven).** Positive
`WaitForXSeconds` installs readiness predicate `sub_9FFCC0` at Lua-thread
offset `+88`, stores the remaining seconds at `+92`, and yields. Each
scheduler check subtracts the current frame delta from `+92` and resumes only
when the result is at most zero. A yielded resume leaves thread state `0`; a
normally returned coroutine becomes state `-1`. The scheduler's runnable test
accepts states `0`, `2`, and `3`, but skips state `-1`; it contains no call to
`sub_9E12D0` or any modifier-release routine. Therefore chunk `349`'s final
return leaves a completed, non-runnable Lua thread and does not itself delete
the modifier. Any later removal still requires a separate native owner.

Explicit modifier teardown is the other half of the contract. `sub_9E09D0`
resolves the modifier's thread ID at instance `+40`, invokes callback `3`
(Deactivate) when registered, then calls `sub_8EFFE0` to release that Lua
thread and clears instance `+40`. `sub_9E11F0` subsequently removes retained
attribute handles and the modifier instance. For chunk `349`, whose Deactivate
body is empty, this native sequence is the only proven removal of the retained
`Immobilized` handle. A later Unique replacement reaches exactly this path
before creating its new instance. This closes the coroutine-ownership gap:
Activate completion is dormant thread state, while modifier deletion is an
independent explicit lifecycle operation.

Native `RequestModifier` returns a numeric modifier instance/request ID; chunk
`144` discards it. Native
`GetTeleporterDestination(object)->(number,number,number)` resolves the linked
destination object's position and returns `(0,0,0)` on any failed link.
`CreateTriggerVolume(x,y,z,radius,callback1,callback2,callback3)->number` retains
the callbacks; `DestroyTriggerVolume(number)->()` accepts either Lua numeric
representation used by the engine. `TeleportObject` performs the authoritative
transform mutation and reaches the recovered 32-byte `ObjectTeleport` sender.

**Typed director boundary.** An adopted server intent needs a live expected
teleporter noun/passive, one phase-owned trigger, an entrant controlled by the
requesting player, an allowlisted organic threat query, and finite linked
destination coordinates. State changes and modifier requests must be
idempotent across the `0.5s` polling loop. Trigger destruction, retained Lua
callbacks, modifier cancellation, and immobilization cleanup must be owned by
phase/session teardown. Reject a zero destination caused by a missing content
link; do not silently teleport to origin. Chunk `144` has no explicit
`require`, but standalone parity still needs typed equivalents of globals
normally supplied by `Lua!GlobalDefinitions.lua`, including `kObjIDNone`, the
activation/deactivation enums, `nAttributeType`, and the
`nSporeLabs.organicDamageableObjectTypes` collection.

**Separate horde-gate asset boundary (content/native negative proof).** The
decoded asset whose instance is `SPID("HordeGateTeleporter") == 0x1968eb10`
contains an independent event map: `horde triggered` ->
`ActivateHordeGateTeleporter`, and `horde complete` ->
`DeactivateHordeGateTeleporter`. Neither callback is present in the packaged
Lua corpus or `Game.c`, and the asset contains neither GUID `0x502f1932`
nor a link to the boss-security noun. Those callbacks remain undocumented
server commands; they must not be assigned to the Cryos boss platform without
a separate level/content edge. The same asset also binds its trigger component
to `HordeGateTeleporter_OnEnter` (`SPID 0x310c053a`). That callback is likewise
absent from all indexed Lua and `Game.c`. The likely chain
`OnEnter -> request chunk 199 with authored destination properties` is an
inference from the noun callback and modifier shape, not a recovered native
contract. Its exact entrant validation, modifier arguments, activation-state
gate, and duplicate-entry behavior remain server-command homework.

The retained binary makes that separation reproducible. Horde gate is
`AssetData_Binary.package` resource `4894`, ordinal `4893`, type
`0x76a8f7d8`, group `0`, instance `0x1968eb10`, decoded size `960`, SHA-256
`8f613f29e7104d0d4f5ae4fce41071635fc4edaa506d68ec7762afda66829e71`.
Its serialized trigger section has one callback entry, whose identifier is
`0x310c053a`, while the following event section has two entries and retains the
four event/command strings above. The comparison boss-security object is the
same asset type and group but instance `0x0463c020`, resource `10294`, ordinal
`10293`, decoded size `656`, SHA-256
`b0cec5f62c655740b4eec2b18eacad2be235aca08a0b1517f122b2c1c2eea7d3`.
It contains none of the three horde-gate callbacks or either horde event; its
authored link is instead `BossSecurityTeleporter.AIDefinition`. This is direct
content-negative proof against treating the horde gate as an alternate name
for the boss-security passive. A focused same-instance query also returns only
resource `4894`: unlike the boss-security instance, `0x1968eb10` has no
same-instance AI-definition, passive, noun, or other companion resource in
`AssetData_Binary.package`. Consequently neither chunk `199` nor any passive
can be attached by the boss-security same-instance convention. It still does
not identify which level marker, if any, deploys the horde-gate asset; a
separate marker or callback edge is required.

**Missing callback/native boundary (native-proven and inferred).** Native
`GetTeleporterDestination(object)->(number,number,number)` only resolves the
object's linked destination and returns that object's authoritative position;
every failed object, definition, link, or destination lookup returns
`(0,0,0)`. Native `RequestModifier` decodes numeric object/modifier/initiator
arguments, supplies defaults for omitted optional arguments, forwards any
trailing modifier properties to the modifier constructor, and returns one
numeric request/instance ID (or `0` when the target lookup fails). Neither
binding checks player control, gate activation, duplicate modifiers, or a
nonzero destination. Therefore those policies, plus the exact chunk-`199`
property ordering, must come from the absent `HordeGateTeleporter_OnEnter`
server command or its caller. A typed director intent must validate a live
phase-owned gate and entrant, an active transition state, a resolved finite
destination, the exact modifier GUID and property schema, and idempotence per
entrant before requesting the modifier. It must reject the native failure
sentinels rather than teleporting to origin or accepting request ID `0`.

Build-103 IDA resolves the client callback boundary precisely. Static startup
registers `DirectorTrigger_SpawnBoss` to `sub_9FACF0` and
`HordeTrigger_OnEnterPlayer` to `sub_9FACB0`. Both handlers validate the
authority/player branch and retrieve the simulator's inline director, but then
return false without mutating it or sending GMS. The executable contains no
`HordeSpawner_Register` string at all, matching its absence from all 1,029
packaged Lua chunks. The Cryos event and its two listener records are therefore
server-owned commands whose implementation is not shipped in this client.
This is distinct from `DirectorTrigger_SpawnSurvivorHorde`, whose registered
handler does allocate a local director record.

The exact inert boss-handler predicate is narrower than its semantic name:
`sub_9FACF0(trigger, entrant)` never reads `trigger`; it proceeds only when the
TLS simulator byte at `+27` is zero and `entrant != 0 && entrant[+92] != 0`.
It then computes the inline director address `simulator+1368`, discards it, and
returns `false` on every path. The `+92` field is native-proven as the accepted
entrant discriminator here; calling it a player flag is supported by its
surrounding uses but remains a semantic label. There is no event-name argument,
listener collection, wave count, spawn request, wait, mutation, or replication
hidden in this client callback.

**Raw Cryos fanout evidence and normalization limit (content-proven).** The
design payload is level marker set `283`, SHA-256
`651b17394e29be66a3e05c64649ff648e3274d9c9293b9008edd1f1c2ba95376`.
Its relevant records are exactly:

| Ordinal / marker row | Authored marker ID and noun | Position | Raw component strings |
| --- | --- | --- | --- |
| `1` / `16907` | `1730050752`, `SpawnPoint_DirectorBoss.Noun` | `(-350.04453,-223.36415,10.08803)` | `DirectorTrigger_SpawnBoss`, `horde triggered02` |
| `3` / `16909` | `3741961587`, `SpawnPoint_DirectorHorde.Noun` | `(-355.26929,-207.71077,10.08800)` | `horde triggered02`, `HordeSpawner_Register` |
| `6` / `16912` | `3740600745`, `SpawnPoint_DirectorHorde.Noun` | `(-345.89496,-239.54680,10.16637)` | `horde triggered02`, `HordeSpawner_Register` |

No other authored string occurs between each of those marker boundaries. That
excludes a hidden textual noun list, callback, delay, or budget in the three
records, but does not assign semantics to their remaining numeric fields. The
normalized rows are `6063`, `6065`, and `6067`; all are conservatively labeled
`SharedComponentData`, `listener_or_trigger`, slot `unknown`, radius `0`,
`is_server_only=0`, and `is_trigger_once_only=0`. Those zero/false fields are
decoder defaults for standalone event/callback string pairs, not proof of an
authored radius or authority policy. The callback and noun names strongly
identify one director and two registration loci, while event publication,
delivery order, listener lifetime, and wave mutation remain server-native or
live-observation questions.

The director's `waveOverride=4` is now instruction- and payload-proven rather
than inferred from four live waves. IDA registers reflected
`SpawnPointDef.waveOverride` as a four-byte field at definition offset `+0x18`;
the director record's `SpawnPointDef` component begins at payload offset
`0x9ab` and stores little-endian `04 00 00 00` at `0x9c3`. The focused audit is
`bin/game/logs/ida-wave-override.log`, SHA-256
`a98bf557d84b2e6ff3a627cd0983c91c81738af35b08eabacb83cbdfedcafd1`.
This proves the requested wave count is four. It does not define a wave budget,
enemy count, selection order, retry policy, or completion predicate; those
semantics are absent from the reflected field registration and remain work for
the missing server director.

The adjacent override is also exact. IDA registers four-byte
`SpawnPointDef.challengeOverride` at `+0x14`; the same Cryos component stores
`00 00 00 00` at payload offset `0x9bf`, followed by `waveOverride=4` at
`0x9c3`. Thus the arena explicitly overrides wave count but not challenge.
The focused field-consumer audit is
`bin/game/logs/ida-spawnpoint-fields.log`, SHA-256
`6559e004c4bd257106a279ca1b0432aeac5f22dbf7bf3fb3353e2159aee5441f`.
Each `waveOverride` descriptor global has only its initializer xref, and a scan
of the client director function range `0x009fa000..0x00a00000` finds no direct
instruction reading displacement `+0x18`. Generic reflection access remains
possible, but there is no direct client director consumer that turns the field
into a wave loop. Treat four waves as authoritative content input whose
execution belongs to the absent server path, not as client-owned scheduling.

The native type-`9` marker-load path does not close that gap. `sub_9FB420`
classifies the boss spawn point, calls `sub_9FAE10`, and immediately returns.
`sub_9FAE10` requires a definition at object offset `+168`, allocates one
special-region record, copies the marker's XYZ and collision-derived radius,
and retains the definition pointer. It never
reads `SpawnPointDef.waveOverride`, invokes either authored callback, creates
an enemy, changes any reflected director flag, waits, or replicates a packet.
Thus build 103 proves client spatial registration separately from the absent
authoritative contact/fanout/wave consumer.

**Director flag split (native/reflection-proven).** The focused audit
`bin/game/logs/ida-horde-state.log`, SHA-256
`20d73bbc10296c21267dae1b4949165c9c8d2ab6a947c906fca42c733eb67da5`,
resolves the low director fields: `mbBossHorde` is byte `+0x0e`,
`mbCaptainSpawned` is `+0x0f`, `mbBossComplete` is `+0x10`, and `mBossId` is a
four-byte object ID at `+0x14`. The separate active state remains
`mActiveHordeWaves` at `+0x47c` and `mbHordeSpawned` at `+0x48c`. Both director
constructors initialize all these flags/fields to zero.

Consequently the write at the end of `sub_9FAEB0` is specifically
`mbBossHorde=true`, not a generic dirty bit and not `mbHordeSpawned=true`.
Only `DirectorTrigger_SpawnSurvivorHorde` reaches that helper, and it does so
only after rejecting a duplicate key from source-object offset `+132` and
adding a previously unregistered region. Ordinary type-`9` marker
loading instead calls `sub_9FAE10`, while `DirectorTrigger_SpawnBoss` calls
neither helper. Therefore Cryos boss-marker loading and boss contact do not set
`mbBossHorde` through these client paths. `IsHordeActive()` reads only
`mbHordeSpawned`; the client still contains no recovered transition that makes
that byte true or populates `mActiveHordeWaves`. A typed director must keep
region registration, survivor-horde classification, boss contact, wave
activation, and terminal completion as separate states.

**Packaged director-native caller closure (bytecode-proven).** Across all
1,029 recovered chunks, `ActivateHordeSpawn` occurs only in tutorial unlock
chunks `62`, `222`, and `769`, whose exact hashes and no-op tail schedules are
recorded above. `SetBossId` occurs only in unrelated Scaldron modifiers:
chunk `81`, `nModifier_ScaldronBoss_AIPassiveStage2`, SHA-256
`27c0bce10b664fbbc1f496e104ff990fd559f92a2d1879dbd10df7a479388e83`;
chunk `305`, `nModifier_ScaldronBoss_AIPassive`, SHA-256
`af63a43d71a445d887f3c18401be652d0da3f2a1a360d89814828755c20634fb`;
and chunk `446`, `nModifier_ScaldronBoss_Stage2Death`, SHA-256
`c940cacbcb5bf66de3b3599b9e7f285b78250bb53286c78972103f9d2f663a2d`.
No packaged chunk contains `IsBossDead`. The UI invokes that client gate
outside packaged gameplay Lua. Thus no recovered Cryos Lua caller sets the
boss ID, starts the arena horde, marks it active, or marks it complete; those
transitions remain server reflection/director responsibilities.

**Dormant mode-`5` selector boundary (native-proven).** The xref audit in
`bin/game/logs/ida-director-xrefs.log`, SHA-256
`a4b25b156952d33690a8b6b212f82ee64f72d4bd94053009d89c31c963c1707b`,
finds exactly one code reference to selector `sub_9FE270` and no raw stored
function pointer. The sole caller is `sub_9FE7D0` at `0x009fea1b`; it passes
mode `1`, literal budget `10`, and current difficulty while performing a
conditional ordinary-spawn substitution. No build-103 client caller passes
mode `5`.

The dormant branch itself is exact but not Cryos-owned. It filters the
director's cached 16-byte `DirectorClass` records by inclusive current-difficulty
range, randomly chooses from the matching candidate pointers, and accepts at
most `15` nouns. For accepted index `i` (zero based), its prospective running
cost is `(1 + i*0.1) * (priorRawCost + nounCost)`. If the randomly chosen noun
would exceed the caller budget, it removes that candidate, returns failure,
and the mode-`5` loop stops rather than retrying a cheaper noun. On acceptance
it appends the noun pointer, adds `nounCost` to raw cost, records the inflated
cost, and increments the count. Empty candidates, exhausted budget, first
failed choice, or count `15` terminate selection.

**Selector record provenance (native/reflection-proven).** The focused
reflection audit `bin/game/logs/ida-horde-bucket.log`, SHA-256
`31f3c34499f552eaf74ce457bbb86d1279ff8d690570a8763ab1ddf563bb2cf6`,
identifies each `DirectorBucket` as a 16-byte value with `numMinions` at
`+0x00`, `numSpecials` at `+0x04`, `difficulty` at `+0x08`, and floating-point
`chance` at `+0x0c`. This `chance` field is unrelated to the separate
`hordeLegal` boolean at `+0x0c` in the level director-class record.
`DirectorTuning` is also 16 bytes, containing the reflected
`OrbDifficultyScale` array at `+0x00` and `HordeDifficultyWave` array at
`+0x08`; the latter is a field name, not evidence for an independently named
record type.

During director initialization, `sub_9FC430` resolves the active director
noun/configuration, requires its `LevelConfig` pointer at noun-data offset
`+88`, and calls `sub_9FBD10(config, 3, firstOutput, 3, secondOutput)`. The
focused LevelConfig reflection block in
`bin/game/logs/ida-director-class-fields.log`, SHA-256
`ebfbdc70d4256fc8aeecd663b22991ae6e98853fb6213da96659816a45b8530d`,
proves a `0x28`-byte five-field configuration ordered `minion`, `special`,
`boss`, `agent`, and `captain`. Native access proves its five record pointers
at `+0/+4/+8/+12/+16` and matching counts at `+20/+24/+28/+32/+36`.

`sub_9FBD10` reads the authored `minion` pointer/count at `+0/+20` into its
first output and `special` at `+4/+24` into its second. For each it copies only
16-byte records whose integer bounds at record offsets `+4` and `+8`
inclusively contain the current difficulty, then independently shuffles and
truncates matches to three. `sub_9FC430` caches at most three minion records in
the director vector at dword index `181` and at most three special records at
index `210`; selector mode `5` later filters the latter special vector. Thus
the cached mode-`5` candidates have an exact `special` class provenance, but
they still are not the Cryos roster: the caller/budget edge is absent and the
records originate in the active director noun configuration rather than the
six level `firstTimeConfig` horde-legal noun rows.

**Global director-tuning asset (content/reflection-proven; consumer
unresolved).** Case-insensitive FNV-1 maps `DirectorTuning` to instance
`0x4c24cf9a`. `content.db` contains exactly one matching build-103 resource:
`content_source_resource.id=4535`, type `0x9342c4d3`, group `0`, decoded size
`592`, decoded SHA-256
`6a8d50940fda4fbd86054180920a482df35892f2d07bab45c05c6da611c24b85`.
The extracted RefPack member is retained as
`bin/game/logs/director-tuning.raw`, raw SHA-256
`ffd9646a1d588e46d879f75e8c03d333c17eed7d5c8c327b2e031bbcd6d9ea2d`;
its decoded bytes are `bin/game/logs/director-tuning.bin`.

Reflection orders `OrbDifficultyScale` before `HordeDifficultyWave`, and the
asset contains two corresponding 72-element arrays. Every
`OrbDifficultyScale[0..71]` value is floating-point `5.0`.
`HordeDifficultyWave` contains integer runs `0..3 => 4`, `4..35 => 5`,
`36..59 => 6`, and `60..71 => 7`. The payload contains no `DirectorBucket`
records and therefore supplies no `numMinions`, `numSpecials`, or selection
chance. Its values also are not the Cryos `waveOverride=4`: only the first four
indices happen to equal four, while the remaining 68 entries are five through
seven.

The exact native consumer remains unresolved. The binary-immediate audit
`bin/game/logs/ida-director-tuning-xrefs.log`, SHA-256
`36c57d886484a2fba2819eae46019c70640cb514efa8ad3142a6fb61a126b9fe`,
finds no embedded `0x4c24cf9a` instance ID or either serialized field ID in
the executable, consistent with generic reflected asset loading. Consequently
the field name supports a generic difficulty-to-wave-tuning interpretation,
but assigning these numbers as Cryos enemy counts, budgets, delays, wave
indices, or listener allocations would be inference. They add no retail
composition authority to mode `5` or the boss-marker callback.

**SectionConfig bucket consumer (content/native-proven; Cryos call edge
absent).** Case-insensitive FNV-1 maps `SectionConfig` to instance
`0x45b7b07c`. `content.db` again has exactly one matching build-103 resource:
row `8797`, type `0xa38fa119`, group `0`, decoded size `1160`, decoded SHA-256
`f5bc27977b7b06abb75862bd761a3e014f64376fa0b4b49f4ee0b94fd39fb2e6`.
The retained RefPack member `bin/game/logs/section-config.raw` has raw SHA-256
`9d21d641ead587312601b6f81478eb71a6753841786871c4aee00cc2ba38c87f`;
`bin/game/logs/section-config.bin` is its decoded form. Reflection proves that
the eight-byte `SectionConfig` contains one `bucket` array of 16-byte
`DirectorBucket` records.

The stock asset has 72 buckets. For each difficulty `1..18`, it contains
exactly four records: two copies of
`{numMinions=2,numSpecials=1,difficulty=d,chance=0.0}` and two copies with
`numSpecials=2`; every other field is unchanged. This is generic composition
data rather than Cryos level data.

`sub_9FC430` is the positive native consumer. It resolves `SectionConfig`,
collects indices whose bucket `difficulty` exactly equals the current director
difficulty, and proceeds only with at least three matches. For section indices
`0`, `1`, and `2`, it uniformly picks one remaining matching bucket, reads
`numMinions` and `numSpecials`, fills that section from the cached minion and
special `DirectorClass` candidates, then removes the bucket index before the
next section draw. It does not read `chance` on this path. Candidate indices
are unique within a section but rebuilt for the next section, so a noun class
may repeat across sections. The focused reflection/consumer audit is
`bin/game/logs/ida-director-bucket-container.log`, SHA-256
`24eb5958978f3103618d3ca30d6b6120f672db0c5ec929f56e4a00feee770056`.

With the stock buckets, initialization therefore prepares exactly two minion
entries per section and either one or two special entries per section: six
minion plus four or five special entries across the three sections. This is
not yet a Cryos retail wave count. The initializer prepares cached section
state; no recovered edge connects the Cryos boss callback to its later spawn
consumer, maps its three section indices to the two Cryos listener markers, or
explains the level's four-wave override. Likewise the unresolved
`HordeDifficultyWave` consumer prevents treating its values `4..7` as the
difficulty input used here. A typed arena fixture may test this generic
initializer separately, but must not substitute its `10/11` prepared entries
for the current two-listener compatibility wave.

**Reachable selector boundary (native call-graph-proven).** Director section
state has four entries with a 58-dword stride: each section's minion vector
starts at relative index `7` and special vector at `36`. Consequently the
initializer's master vectors at indices `181 = 7 + 3*58` and
`210 = 36 + 3*58` are section `3`; the three bucket draws populate derived
sections `0`, `1`, and `2` from those capped master pools.

The build-103 executable has only one caller of `sub_9FE270`, and that caller
supplies mode `1`. Both Lua wrappers first filter their caller-provided
`npcList` by current difficulty and choose an original noun uniformly:
`ActivateMinionSpawn` selects list class `0`, while
`ActivateLieutenantSpawn` selects class `1`. Their common creator then passes
fixed section `3`. The optional substitution branch therefore reads the
master minion pool, not any of the three bucket-populated sections and not the
master special pool. It uses literal budget `10` and is considered only when
the original noun has nonpositive field `+116`, the current tier is at least
the configured threshold, its resolved classification field `+68` is `1`, and
a simulator random draw is below the configured substitution threshold. If
the branch is not accepted, creation continues with the noun
chosen from the caller list.

The runtime thresholds are content-proven rather than defaults. `sub_9CF190`
loads asset `0x3b01d7f6`, which `content.db` identifies as
`LabsTuning/0x3B01D7F6.prop`, `server_data.content_source_resource_id=13516`.
Its decoded payload is retained as `bin/game/logs/labs-tuning.bin`, SHA-256
`b57ca3826416ecc6f23db0ad92b9d6bdce2faa376809a5505bb2869a0014bcce`.
Property `0x0d57df60` is integer `13`, overriding the compiled tier fallback
`25`; property `0xb7a59b8b` is float `0.02`, overriding the compiled random
fallback `20.0`. `sub_9BCF40 -> sub_AE54A0` proves a uniform result in
`[0,1)`, so the accepted runtime substitution gate is exactly a 2% draw at
tier `13` or higher, subject to the two noun-field predicates above. This is a
rare generic spawn substitution, not a horde-wave activation probability.

The mode-`2` special path, the bucket-populated section compositions, and the
mixed mode-`4`/`7` helpers exist only behind other `sub_9FE270` cases for which
no client caller exists. Thus `SectionConfig` counts do not control the
reachable Lua spawn bindings. A missing authoritative server may use
equivalent data, but its call, section/listener mapping, and replication remain
homework.

This is useful parity knowledge for the generic director, but it cannot supply
the Cryos retail composition without a recovered server caller and exact
server-side active-director configuration. In particular, do not equate the level's
six horde-legal noun records with this separate candidate vector, and do not
replace the authored two listener loci with a mode-`5` ownership claim. A
future typed selector fixture requires deterministic simulator RNG,
difficulty-range filtering, noun-cost lookup, the cumulative `0.1` multiplier,
first-overbudget termination, and the 15-result cap; the Cryos director intent
must remain independent until its server budget/input edge is proven.

**Arena timer evidence split (local-live versus retail-unresolved).** As
recorded in `notes/campaign/arena-director.md`, darkspin currently schedules its first
arena pair exactly `2s` after the accepted movement-handler boundary and
anchors spawn timestamps to the incoming source time plus `2000ms`; a focused
build-103 run consumed the incoming alert before that pair appeared at the two
listener loci. Its clear-to-next-wave delay is `1.5s`. These timings are exact
for the tested compatibility implementation only. No recovered Lua, director
record, listener record, client callback, or retail packet capture contains
either delay. Moreover that handler bypasses chunk `349`, so its same-batch
teleport/alert trace cannot be combined with chunk `349`'s authored
`0.5s / 0s / 0.5s` modifier timeline and labeled retail ordering. A typed
director should keep both timers server-owned and configurable until a retail
trace or server callback supplies their values.

This callback vocabulary is generic rather than Cryos-exclusive: the indexed
content contains `30` `DirectorTrigger_SpawnBoss` rows and `393`
`HordeSpawner_Register` rows, with reused names such as `boss triggered` and
`horde triggered`. Consequently the string `horde triggered02` is meaningful
only inside this level/marker-set relationship; it is not a process-global
route identifier. A typed director must scope registrations to the active
level instance and phase, reject stale listeners after restart/removal, and
fan out atomically to the two authored Cryos loci only after the absent server
callback contract is recovered.

The teleporter destination is only `2.76475` world units from the boss director
center (and `18.56664` / `15.03295` from the two listener loci), so the authored
radius-`30` spatial entry follows directly from a successful chunk-`349`
teleport. This proves geometric overlap, not callback execution order: the
server still owns trigger contact delivery, once/re-entry policy, and the
transition from accepted contact to event fanout.

IDA also names the byte returned by `IsHordeActive` as reflected field
`mbHordeSpawned` at director offset `0x48c`; reflected container
`mActiveHordeWaves` begins at `0x47c`. Across the build-103 executable, the only
direct accesses to the byte are two zero-initializations and the Lua getter,
and the active-wave container likewise has only constructor writes. A generic
reflected server update could populate them, but no concrete sender/decoder
route has yet been recovered. They cannot yet serve as evidence for the
server's horde wave-clear predicate.

darkspin now implements and live-confirms that gate boundary for build 103. The
boss teleporter is runtime object `28`; it is created at its authored marker
with `zelem_boss_teleporter_inactive.ServerEventDef`. Once Sage is available,
the gate remains locked while the authoritative encounter contains any living
enemy. Removing the last enemy now sends
`zelem_boss_teleporter_powerup.ServerEventDef`, then publishes the continuous
`zelem_boss_teleporter.ServerEventDef` one second later. An earlier
implementation sent both in the same batch and rendered two overlapping
teleporters; a later single-active shortcut suppressed the startup animation.
Production delayed publication preserves the recovered transition, while a
test adapter without a scheduler retains power-up-before-active ordering in one
response. The trigger/handoff becomes authoritative at that same deadline;
contact during the startup effect cannot teleport early. The focused checkpoint `(250,225,20.16802)` begins outside the trigger and creates
one stationary Chlorosaur blocker. Live testing showed the gate platform, the final hit and
enemy removal, then the large power-up dome over the platform; the server
logged the enemy-clear transition exactly once. A swept 6.67-unit entry test
then sends `ObjectTeleport` for the deployed hero to destination marker
`174193625`. A clean build-103 crossing visibly moved Blitz and the camera into
the red arena at `(-347.57553,-224.60829,10.08803)`.

The first playable arena director is now implemented without coupling it to
the opening encounter's stage progression. Arena entry lands inside the
authored radius-30 director and schedules wave one after its authored two-second
activation delay. The two registered listeners are preserved exactly at
`(-355.26929,-207.71077,10.08800)` and
`(-345.89496,-239.54680,10.16637)`. The level's first-time configuration is the
only recovered composition source: `TutorialBasicDiseased`,
`TutorialBasicRanged`, `TutorialBasicPoison`, `TutorialSloth`, and special
`TutorialSpecialOne`. Until the retail random budget is recovered, darkspin uses
a deterministic pair per wave, one at each listener, across the authored four
waves. Each object receives its exact stationary spawn coordinate and the
packaged `horde_beam_in` animation; clearing a pair schedules the next wave
after 1.5 seconds. Instruction-level decoding now identifies that animation's
retail modifier boundary. `SpawnModifier` stops locomotion, applies
`Immobilized=1`, adds `generic_spawn.ServerEventDef`, sets
`horde_beam_in`, and waits `0.5s`; deactivation removes the effect and resets
the animation. The caller that attaches/deactivates the modifier remains to be
recovered; immobilization cleanup is native modifier-instance ownership, not a
missing Lua call.

A focused build-103 pass live-confirmed the complete entry boundary: one JWT
login request reached gameplay, the active security platform moved Blitz into
the red arena, wave one appeared at both authored sides after two seconds, both
objects accepted authoritative Voltic Slash damage, and removing the pair
spawned wave two. The client resolved the second listener's wave-two noun and
displayed `Simulated Game Sloth`. Static tests enforce the two loci and
exactly four terminal waves. The retail enemy counts/budget, arena
aggro/attacks, live health-obelisk interaction, and the final green exit remain
unresolved. The two alert packets and both health-obelisk objects are
implemented below, with their remaining visual checks called out separately.

Three one-use loot obelisks and two one-use health obelisks are authored. The
loot obelisks use `InteractWithObelisk` and challenge value `500`; the health
obelisks use `InteractHealthObelisk` and marker override `100`. The level binds
no tutorial-specific `LevelObjectives`, concrete item reward IDs, completion
callback, or exit callback. `Electro Claws`, `Onyx Barrier`, Sage persistence,
and completion rewards therefore remain video-observed/runtime questions, not
asset-confirmed grants.

The two arena health obelisks are marker `785296806` at
`(-353.94858,-192.45534,10.11817)` and marker `2012042454` at
`(-340.43002,-248.87521,10.11818)`. Both use
`prefab_health_obelisk.Noun`, scale `0.55`, default graphics state `boss_2`,
one allowed use, and the tutorial marker's challenge value `100` (overriding
the noun default of `150`). darkspin reserves runtime objects `37` and `38`,
creates them only when the boss teleporter enters the arena, and accepts each
from the active deployed hero once. Current-build observation corrects the old
direct-heal reconstruction: use moves the obelisk to consumed state
`1`/`boss_1`, emits `effect_obeliskactive.ServerEventDef`, and creates one
nearby `HealthOrbPlaced.Noun` capsule. Health changes only when movement later
intersects that capsule's ordinary collection sphere. A full-health contact
emits the local full-resource event without consuming or deleting the capsule.
Static tests pin the markers, one-use ownership, spawned object, movement
pickup, and full-resource preservation. The exact retail ejection direction
and live activation presentation remain open.

The packaged English `LabsAlerts.locale` maps `0x09ed690c` to `Horde
incoming!` and `0x09ed6f0c` to `Horde defeated!`. IDA resolves the transport:
the build-103 ServerEvent handler reflection-decodes field 15
`clientEventID`, and when it is nonzero calls the generic local alert
dispatcher directly. That dispatcher maps `0x1d42121d` to the incoming string
and build-103 ID `0x8047eaf4` to the defeated string. The former is exactly the
case-insensitive SPID of `HordeIncoming`; the latter deliberately uses the
observed build-103 branch value rather than the differently hashed name found
in the newer reference. GMS `SystemMessage` wire ID `0xa0` is unrelated to
these two tutorial alerts. darkspin emits a standalone ServerEvent field-15
message at arena entry and on terminal wave clear. Codec and gameplay marshal
tests pin both IDs. A focused build-103
pass then killed the checkpoint blocker, observed the enemy-clear power-up,
and crossed the trigger with a left-click ground-move request. The server sent
the incoming field-15 packet in the same response batch as `ObjectTeleport`,
and the client entered the red arena before wave one spawned two seconds later.
The earlier apparent gate failure was a right-click on empty ground, which the
client encoded as a targetless Voltic Slash rather than movement. The incoming
banner was then captured visibly as `Horde incoming!`.

A subsequent focused pass cleared all eight deterministic enemies across all
four waves. On the final hit the server sent the defeated field-15 event and
then `TutorialGameMsgs` subtype `0`, exactly in that order. The client
immediately faded to a permanent black screen before `Horde defeated!` could
render and never created the recorded green exit or `RETURN TO SHIP`. This
live result disproves the fourth-wave kill as the `0xC8` send boundary. darkspin
now emits only the defeated alert there and keeps the completed session in the
arena; completion persistence and `0xC8` are deferred until the retail green
exit and return interaction are recovered.

The first loot obelisk on the playable route is marker `2655870385`, noun
`prefab_boss_obelisk.Noun`, at `(191.91814,22.70923,29.76409)`, rotation
`-33.99210` degrees and scale `0.75`. Its authored interaction allows one
`InteractWithObelisk` use with challenge value `500` and default graphics state
`boss_2`. darkspin reserves runtime object ID `15` for it and creates it after the
currently implemented four-stage encounter sequence clears. Build-103 packet
`0x98` dispatches to `sub_539B70`: it reads a four-byte object ID, resolves the
object's `cInteractableData`, and reflection-decodes its three registered
fields. Those fields are `mNumTimesUsed` at offset `0x08`,
`mNumUsesAllowed` at `0x0c`, and `mInteractableAbility` at `0x14`, so the full
initial update is bitmap `0x07`, values `0`, `1`, and the SPID of
`InteractWithObelisk`. The enclosing `sporelabsObject` reflection fields `21`
and `22` are `mInteractableState` and `sourceMarkerKey.markerId`. Client state
logic accepts states `2` and `3`; the normal enable path writes state `3`.

The focused checkpoint at `(181.5,22.70923,30.088)` proved this contract live.
The obelisk rendered at its authored position, clicking it walked Blitz into
range, and the client sent type-11 `ActionUseInteractable` with source `1`,
target `15`, and position `(185.36333,19.537325,28.978449)`. Creation alone had
previously produced only movement requests. darkspin now accepts that exact use
once within an 8-unit compatibility boundary around its authoritative position
(the exact retail maximum remains unrecovered), sends
`mNumTimesUsed=1`, and transitions the object to consumed state
`1`. Focused wire tests cover the response. A follow-up build-103 run verifies
the response live: the server records exactly one accepted use, and clicking
the object again produces no second `ActionUseInteractable`, confirming that
the client considers it consumed. The model remains visible after use, so this
run does not yet distinguish the expected `boss_1` graphics state.

A later action-admission guard exposed a response-contract regression: the
server committed the use and scheduled the authored timeline without returning
an accepted action response. Admission consequently canceled that same
timeline as `terminal response missing`, leaving the obelisk exhausted while
Blitz continued the client's approach drift. The interaction now returns the
content-timed accepted response, stops Blitz at his authoritative contact
position, applies the authored interaction animation to the activating player
rather than the obelisk, and emits the matching release response after the
open/drop/final sequence.

The login failure that initially blocked this follow-up was not a Blaze or TCP
multiplexer collision. The client socket trace showed two identical 276-byte
login sends on the same connection and millisecond from separate threads. The
Fang directly invoked the login manager and also set controller bytes
`0x29`/`0x2a`, which prompted the UI thread to submit the stored credentials a
second time. Removing the controller mutation was necessary but proved
insufficient in a later horde test: calling the manager from a worker still
raced the UI and produced the same pair. The final hook invokes the manager
once from the login-screen initialization/UI thread. The clean horde run has
one `jwt_login_submit`, one 276-byte socket send, one command-40 request, and
reaches gameplay.

The item-acquired route is now recovered through the retail UI branch.
`ServerEvent.asset` is `loot_acquired.ServerEventDef`, while field 15
`clientEventID` is `LootAwarded`; fields 17 through 25 carry the loot tail.
The event's `lootRigblockId` is not the numeric `rigblockId` metadata or the
resulting `cLootData.rigblockAsset`. Build 103 passes it through `sub_9C5FF0`,
whose global map is initialized from `AssetCatalog/AssetGlobals`, then copies
the resolved asset into the temporary pickup-card record. Electro Claws uses
catalog key `_Generated/LootRigblock268.LootRigblock` (`0x096B7C20`), which
maps to packaged asset `0x646569E2`; content metadata ID `268` is not the wire
key. The corrected key is live-confirmed: retail conversion and formatting
both return success, and the client renders “You've Received — Encrypted Item
— Electro Claws.” The server grants the authenticated user one level-5 basic
part with rigblock metadata ID `268`; `darkspin db part get user_id=1` confirms
the durable row. Repeating the authored grant is idempotent, while a failed
save removes the tentative in-memory part before returning an error. The
focused checkpoint remains preserved, and the default player spawn is restored
to the authored tutorial route start.

The first obelisk interaction and reward collection are separate authorities.
The authored sequence consumes the obelisk at `0.6s` and publishes object `16`
at `1.6s`; it does not mutate inventory at click time. The pickup is created at
the obelisk and moves to a nearby ground position. Entering a two-unit
collection sphere then persists rigblock `268`, emits `LootAwarded`, deletes the
world object, and opens the Sage approach phase. A persistence failure clears
only the pending collection and leaves the published pickup available for
retry. Available observation shows the reward can land on any side and the
obelisk has already disappeared, so landing direction has no gameplay meaning.
The simulator uses the interaction side as a deterministic compatibility
choice; exact retail lob coefficients still require an original build-103
server capture.

Beta footage shows alternate obelisk loot including Photonic Repulsor and
Thermic Shard, plus equipment drops from Quadra-class enemies including Pulsar
Compensator, Thermic Watcher, Thermic Ridge-Eye, and Hyperion Oculus. It also
shows horde-room
obelisks ejecting as many as four mixed health/power capsules. These
observations are explicitly non-authoritative for the current build and are not
implemented simulator requirements unless build-103 evidence confirms them.

### Lessons visible in authored audio triggers

The tutorial audio marker set contains names for:

- attacking and power use;
- active and passive abilities;
- health;
- obelisks;
- hero switching;
- Blitz;
- Sage (`vo_ship_tut_sage_v2`);
- a `SecondCreature_trigger` zone.

This is **asset-confirmed** evidence that the mission teaches these mechanics
and introduces at least a second creature. It does not establish the exact
lesson order, when Sage becomes persistent, or which operation grants each
hero. Those require gameplay and account snapshots at each trigger.

### Video-observed mission order

The local 1080p copy of
[`0-tutorial.mkv`](../bin/video/walkthrough/0-tutorial.mkv) is a 6:36 capture of
[Game: Complete Story Playthrough - Part 0: Tutorial](https://www.youtube.com/watch?v=wy0nroTZaBo),
SHA-256
`123E71E4BDCEF199328A3867EC4D8D253C6F78984AE9B7DC4758A00D73B44149`.
Generated contact sheets and selected full-resolution reference frames are in
[`bin/video/walkthrough/tutorial/`](../bin/video/walkthrough/tutorial/).
The contact sheets cover broad 10-second and 5-second sampling plus focused
1-second ranges; `frame-NNN.png` identifies the approximate second from the
start of the recording.
The exact client build and account fixture are not shown, so the sequence below
is **video-observed**, not build-103 or persistence confirmation.

#### Video research status

The observation pass over the available recording is complete. It establishes
the visible lesson, reward, unlock, teleporter, horde, and victory order. It
does not include the click on `RETURN TO SHIP`, an account response, or any
gameplay protocol capture, so protocol and persistence correlation remains
active research rather than completed tutorial reconstruction.

The paired files named `client-tutorial-complete.jsonl` and
`server-tutorial-complete.jsonl` are not completion evidence. In the server
trace, sequence `39` is GameManager join command `0x09` and sequence `40`
rejects it with error `0x0002`; no connected gameplay RakNet/GMS exchange
follows. Retain the pair as an IDA-01 failure baseline and do not use it to
promote the `3000` write from binary-confirmed to live-confirmed.

#### Video-observed XP events

The narrow white bar at bottom center, immediately to the right of the
Crogenitor level number, is the visible account-XP bar. A frame-level pass at
10 samples per second measured its filled width across the stable 567-pixel
interior. Each transition below coincides with a visible enemy defeat. The
percentages are therefore estimates from rendered pixels, not decoded account
state; rounding, capture scaling, and the one-pixel bar border introduce about
`+/-0.3` percentage-point uncertainty.

| Video time | Visible defeated target or context | White XP bar transition | Estimated award | Threshold interpretation |
| --- | --- | --- | ---: | --- |
| `1:08.0` | Simulated Game Chlorosaur | level 1: `0.0% -> 8.8%` | `+8.8%` | About 9 XP on the 100-XP level-1 span. |
| `1:13.4` | Simulated Game Chlorosaur | level 1: `8.8% -> 22.9%` | `+14.1%` | About 14 XP. |
| `1:16.3` | Simulated Game Chlorosaur | level 1: `22.9% -> 35.1%` | `+12.2%` | About 12 XP. |
| `1:17.5` | Simulated Game Chlorosaur | level 1: `35.1% -> 49.4%` | `+14.3%` | About 14 XP. |
| `1:44.1` | Simulated Game Chlorosaur | level 1: `49.4% -> 61.6%` | `+12.2%` | About 12 XP. |
| `1:45.5` | Simulated Game Chlorosaur | level 1: `61.6% -> 73.5%` | `+12.0%` | About 12 XP. |
| `1:46.7` | Simulated Game Chlorosaur | level 1: `73.5% -> 87.7%` | `+14.1%` | About 14 XP. |
| `1:47.4` | Simulated Game Chlorosaur | level 1: `87.7% -> 100.0%` | `+12.3%` | About 13 XP; the bar becomes full but there is no level-up yet. |
| `2:07.6` | Simulated Game Chlorosaur | level 1 full -> level 2 `8.8%` | about `+8.8%` of level 2 | **Ding to Crogenitor level 2.** The visible `You reached Crogenitor Level 2!` banner and reset bar corroborate the recovered strict `XP > 100` boundary. About 9 new XP makes the inferred cumulative total 109. |
| `2:12.4` | Simulated Game Chlorosaur | level 2: `8.8% -> 16.9%` | `+8.1%` | About 8 XP on the 100-XP level-2 span. |
| `2:15.5` | Simulated Game Infector | level 2: `16.9% -> 25.0%` | `+8.1%` | About 8 XP. |
| `2:18.3` | Simulated Game Infector | level 2: `25.0% -> 35.1%` | `+10.1%` | About 10 XP. |
| `2:19.0` | Simulated Game Chlorosaur | level 2: `35.1% -> 43.4%` | `+8.3%` | About 8 XP. |
| `2:52.7` | Simulated Game Chlorosaur | level 2: `43.4% -> 51.3%` | `+7.9%` | About 8 XP. |
| `2:54.0` | Simulated Game Chlorosaur | level 2: `51.3% -> 59.4%` | `+8.1%` | About 8 XP. |
| `2:58.2` | Simulated Game Infector | level 2: `59.4% -> 67.5%` | `+8.1%` | About 8 XP. |
| `3:44.3` | Simulated Game Quadra | level 2 `67.5%` -> level 3 `0.7%` | remaining `32.5%` of level 2 plus `0.7%` of level 3 | **Ding to Crogenitor level 3.** Using the captured 200/3,000 bounds, the cross-level award is about 54 XP and the inferred cumulative total is about 221. |
| `5:23.8` | Simulated Game Sloth during the horde | level 3: `0.7% -> 2.8%` | `+2.1%` | About 60 XP on the 2,800-XP level-3 span. |
| `6:05.0` | Simulated Game Chlorosaur during the horde | level 3: `2.8% -> 4.8%` | `+1.9%` | About 54 XP; inferred cumulative XP reaches roughly 335. |

The first eight awards fill level 1 to exactly 100% when rounded to the
build-103 100-XP span. Remaining at level 1 with a full bar until the next
award is direct presentation evidence for the binary-confirmed strict
comparison: level 2 begins at cumulative XP `101`, not `100`. The level-2 bar
similarly reaches about 67% before the larger Quadra award crosses the `200`
bound and leaves a small level-3 remainder. During the red horde arena the bar
changes only at `5:23.8` and `6:05.0`; the other visible arena kills do not
produce distinct rendered XP jumps. The bar then remains at about `4.8%`
through `Horde defeated!`, with no level-4 ding or visible completion-XP award
before the recording cuts.

The approximate raw-XP values are inferred by applying the measured bar
fractions to the captured build-103 spans (`100` XP for levels 1 and 2, then
`2,800` XP from the 200 bound to the 3,000 bound). They are useful hypotheses
for a trace or tuning-table comparison, not proof that this recording used the
same client build, XP modifier, or server award formula.

darkspin now applies the first 16 measured awards to all 16 authored pre-obelisk
enemy definitions: `9,14,12,14,12,12,14,13,9,8,8,10,8,8,8,8`. Each
accepted defeat uses a feature-owned, persistent monotonic XP mutation and sends
the resulting cumulative XP/level through `LabsPlayerUpdate`; XP `100` remains
level 1 and XP `101` becomes level 2. The former one-time stage-clear jump to
`101` is removed. Marker correlation also fixed the stage order: the old code
started at the five western enemies and spawned every later group behind the
advancing player. The route now follows four enemies at
`(197..233,-162..-116)`, four at `(170..184,-94..-80)`, five at
`(86..95,-83..-67)`, then the previously omitted pair of poison enemies and one
diseased enemy at `(138..156,20..25)`. Those last three are the exact remaining
pre-obelisk XP events in the footage and bring the implemented cumulative total
to `167`. Frame-by-frame review corrects the Sage/Quadra ordering: crossing the
authored sphere after the loot prerequisite reveals Sage and presents the hero
switch lesson before the `TutorialSpecialOne_Intro.Noun` Quadra introduction.
darkspin now emits the confirmed Sage squad reveal/deploy refresh first and the
Quadra object packets second at that boundary. Quadra's defeat awards the
measured `54` XP and reaches cumulative XP `221`/level 3; it is not the Sage
grant condition. The special defeat bypasses the completed pre-obelisk stage
machine, so it cannot respawn object `15`. The two visible horde jumps are
assigned as compatibility policy to the deterministic wave-two Sloth (`+60`)
and wave-three Poison (`+54`), producing the observed cumulative `335`/level 3.
Their actor identities follow the current two-enemy wave reconstruction and
remain less certain than the measured amounts until the retail director budget
is recovered. Tests pin the route ledger, strict boundaries, special-enemy
isolation, reveal-before-Quadra packet order, persistence success, rejection,
and rollback. The earlier focused checkpoint proved the individual spawn,
award, reveal, and switching packets, but its Quadra-before-Sage ordering was a
bad reconstruction and is superseded by the walkthrough evidence. The complete
director/camera sequence that centers Quadra before returning control is now a
priority reconstruction task. The authored route start is restored, and profile
1 was reset to XP `0`, level `1`, and onboarding `0` after the test.

The recording begins at the ship's `START` prompt rather than at login or the
beginning of the ship tour. It cuts after the tutorial offers `RETURN TO SHIP`;
the player does not click it. Consequently it cannot establish when progress
becomes `3000`, what the completion request contains, or which account changes
survive the return to the collection room.

| Approximate video time | Visible behavior | Correlation and remaining question |
| --- | --- | --- |
| `0:20-0:30` | The player selects `START`, the ship view clears, and Blitz materializes in the Cryos tutorial. | Corroborates the native `MapRoomUI.StartGame` path and tutorial level selection, but does not expose Blaze/RakNet setup. |
| `0:35` | The tutorial identifies Blitz as a Plasma Genesis hero. Only Blitz is present in the hero HUD and the Crogenitor level display starts at 1. | Corroborates Blitz as the initial playable hero. It does not prove whether ownership was persisted before game creation or synthesized for the tutorial session. |
| `0:45-1:20` | Movement, basic attacks, and the first simulated Game encounters are introduced. | Matches `vo_ship_tut_attack` and the authored tutorial enemy set. Exact spawn and death packets remain unknown. |
| `1:24-1:35` | The prompt says `Press 1 to activate an Ability`; Blitz's first ability appears in the action bar and is used against enemies. | Correlates with `nTutorial_IntroAbilities.main`. The ability-unlock state and the packet that authorizes use still need tracing. |
| `1:47-2:08` | Combat fills the white Crogenitor XP bar to 100% at `1:47.4`; the next kill at `2:07.6` produces the visible level-2 ding. The tutorial then explains that Blitz's passive increases critical-strike damage. | Supports in-mission account/Crogenitor advancement, the level-2 tutorial cadence, and the strict greater-than XP threshold. Whether each kill is persisted immediately or only at completion is unknown. |
| `2:25-2:40` | Green capsules restore health; the green HUD arc is health. Blue capsules restore power; the blue arc pays for abilities. | Correlates with `nTutorial_IntroHealthAndPower.main`, `vo_ship_tut_health`, and health/power orb assets. The video displays `Health + 15` and `Health Full` feedback. |
| `2:55-3:20` | An obelisk is introduced as an item source. Activating it produces an item-acquired message and an encrypted `Electro Claws` reward. | Confirms that tutorial presentation includes an in-session inventory item before mission completion. Stable inventory ID, roll source, and persistence timing remain unknown. |
| `3:29-3:35` | The tutorial states that enemies of the same type as the active hero deal double damage. | Correlates with the authored resistance lesson and demonstrates that genetic type affects incoming damage, not merely UI classification. The exact multiplier path still needs code/trace confirmation. |
| `3:39-3:44` | `Hero Unlocked! Press W to switch to Sage, a Bio Genesis Hero.` Sage immediately appears as the second HUD portrait and becomes switchable. | Directly correlates the server-only `nTutorial_IntroSecondCreatureUnlock.main` trigger with the visible Sage unlock. Persistent ownership still requires an account snapshot after return. |
| `3:44.3` | Defeating the Simulated Game Quadra advances the bar from about 67.5% of level 2 to about 0.7% of level 3 and displays `You reached Crogenitor Level 3!`. | Shows that the mission crosses a second account-level boundary. The bar geometry and build-103 thresholds imply an approximately 54-XP award, but the recording does not expose the mutation protocol. |
| `4:05-4:25` | Combat continues with the expanded action bar. The tutorial explains that capsules regenerate health and power for all heroes. | Confirms shared squad-resource recovery semantics at the presentation level; server authority and per-hero attribute updates remain unknown. |
| `4:35-4:58` | A teleporter remains disabled while Game are nearby, then becomes usable after combat. | Correlates the authored teleporter and security-teleporter objects with an enemy-presence gate. The controlling objective/server event remains to be identified. |
| `about 5:00` | Another item is displayed near the teleporter; the visible tooltip identifies `Onyx Barrier`, item level 5, Plasma, Defense Slot. | Confirms authored tutorial loot can include typed, slotted equipment. The recording also repeats that items are used in the Editor. |
| `5:08-6:20` | Teleportation leads to a red arena and a announced horde. Multiple waves spawn while the HUD warns when health is critical. | Correlates the horde director spawn point and separate arena assets. The video does not establish that a distinct boss object is required. |
| `6:20-6:30` | `Horde defeated!` appears, a green completion/exit object activates, and the client displays `RETURN TO SHIP`. | Establishes the visible terminal sequence. The recording ends before the return action, completion message, progress write, or collection-room transition. |

This order sharpens the earlier asset-only interpretation: Blitz is the sole
initial hero; Sage is added mid-mission and is immediately usable; no third
hero is visible before the tutorial ends. A third persistent starter hero, if
one is granted during onboarding, must therefore be established after this
mission or by evidence absent from this recording.

Three marker callbacks are especially important to the server reconstruction:

- `nTutorial_IntroOverdriveActivate.main` is marked `serverOnly=true`;
- `nTutorial_IntroAbilitySecond.main` is marked `serverOnly=true`;
- `nTutorial_IntroSecondCreatureUnlock.main` is marked `serverOnly=true`.

The Sage footage confirms that at least the last callback has a material
player-visible result. darkspin cannot reproduce the retail tutorial by treating
all audio-layer trigger volumes as client decoration. The first implementation
slice should map these server-only trigger coordinates to explicit, idempotent
tutorial events and trace the resulting ability/hero update packets.

That first Sage slice is now live-confirmed. Crossing the authored
`(259.346,81.391,25.088)` radius-25 sphere after consuming the loot obelisk
sends a full build-103 `LabsPlayerUpdate` whose second fixed and reflected
character records identify `PC_LF_Mage.Noun`, asset `0x55D1408F`, version `1`,
creature type `2`, while top-level field `23` advances from `1` to `2`. The
client remains stable, displays the unlock presentation, and emits type-5
action commands for creature indices `1` and `0` when W and Q are pressed.
darkspin validates those requests and answers with the normal action response
and `PlayerCharacterDeploy`; live testing showed the selected portrait and
ability bar switch between Sage's Bio kit and Blitz's Plasma kit. The world
object still renders Blitz, proving deploy alone does not change its graphics
when both creature indices point at the same object. That first result made
`SetObjectGFXState` or the native deploy-side equivalent the next packet target.

The next static pass corrected that model. The historical RakNet server's
`SendPlayerCharacterDeploy` resolves the selected creature's own character
object and sends that object's ID; its adjacent `SendObjectGfxState` packet is
only `objectID`, `state`, and `timestamp`. It therefore cannot replace Blitz's
noun with Sage's noun. darkspin now assigns Sage object ID `17` after the obelisk
and retains Blitz object ID `1`. The unlock creates Sage at the shared player
position with her build-103 noun/asset and hero state, then refresh-deploys
Blitz. W/Q position and deploy object `17`/`1` respectively. The server also
tracks the deployed object as the only accepted action source, and does not
misroute Sage ability indices into Blitz's incomplete ability implementation.
A focused build-103 run confirmed the complete immediate-switch presentation.
W hid object `1`, showed Sage alone, selected her portrait/Bio action bar, and
the next movement request used source object `17`. Q then hid object `17`,
showed Blitz alone at the shared position, and selected his Plasma action bar.
The client also displayed the authored banner exactly as `Hero Unlocked! Press
W to switch to Sage, a Bio Genesis Hero.` The remaining Sage work is her
tutorial-required ability behavior and mission-return persistence, not the
immediate world-model switch.

The next recovery slice uses the ten fixed smart objects from
`Game_Tutorial_cryos_1_smartobjects.Markerset`: five
`HealthOrbPlaced.Noun` and five `ManaOrbPlaced.Noun` objects. Their authored
trigger component is instantiated by the client, names `Orb_Pickup` on enter,
and uses each one-unit authored box. darkspin reserves runtime object IDs `18`
through `27`. Crossing the authored health-and-power lesson creates each noun
at its exact marker position and sends a stopped locomotion goal at that same
position; the capsules remain absent before their introduction. The same
lesson enables subsequent enemy capsule drops. The authoritative fallback tests the complete
player movement segment against the projected pickup box expanded by the
deployed hero's content-owned collision footprint (`0.8` for Blitz and `0.825`
for Sage), so a long click cannot skip a
capsule and corner contacts are not lost to the former spherical approximation.
Collection is recorded once per session. A pickup restores 15 health or power
up to the current 200 maximum on the deployed entrant, sends the matching
`health_orb_*` or `mana_orb_*` `ServerEventDef`, and deletes the capsule object.
The active-only resource policy preserves independent Blitz/Sage state but
remains evidence-constrained until a post-unlock retail capture confirms it.

The packaged noun contract confirms that collection is walk-over, not click or
remote interaction. `HealthOrb.Noun` has noun type `0x75b43ff2`, asset hash
`0x0a764dd8`; `manaorb.Noun` has noun type `0xf402465f`, asset hash
`0x48792d0f`. All four placed/dropped variants are networked, non-combatant
objects with locomotion type `6` and a non-server-only `Orb_Pickup` trigger.
The variants are not otherwise identical. `HealthOrbPlaced.Noun` and
`ManaOrbPlaced.Noun` have lifetime zero and authored box dimensions `1x1x1`;
the enemy-dropped `HealthOrb.Noun` and `manaorb.Noun` have lifetime `30s` and
authored box dimensions `2x2x4`. All select box shape `1`, request game-object
dimensions, are kinematic, activate immediately for each object, and are not
latched once-only. Their shared graphics bounding box is from
`(-0.5,-0.5,-0.5)` to `(0.5,0.5,2.5)`. The exact effective overlap after the
`useGameObjectDimensions` substitution remains a runtime/collision fixture;
the XML's unused sphere and capsule values must not be added to the box.

The build-103 client callback path is now recovered, and it is stronger
negative authority evidence than the noun flag alone. Object construction
`sub_9D1930` reads the noun's trigger definition at noun offset `+292`, calls
`sub_A16F40`, and stores the trigger handle at object offset `+712`.
`sub_A16F40` skips a definition only when its `serverOnly` byte is set and
`sub_9BCF80` reports the client role; these orb definitions are therefore
installed locally. Trigger entry travels through `sub_A16810 -> sub_A15FB0`.
The static initializer at `0x00F53C10` binds literal `Orb_Pickup` to
`sub_9C98A0` through the callback map `sub_A173A0`.

That client callback is an intentional stub: its complete body calls the role
query `sub_9BCF80` and returns false. It does not mutate HP or mana, send a GMS
message, emit `ServerEvent`, or delete the orb. `sub_A15FB0` treats the false
return as an unhandled callback and returns without latching the trigger. By
contrast the adjacent `DNA_Pickup -> sub_9C98B0` returns true only outside the
client role, confirming the role-query interpretation. The original retail
server must have owned a different/shared-server implementation that is absent
from this client binary.

Consequently there is no orb-pickup request packet to wait for in build 103.
Accepted player movement is the server's overlap input; the authoritative
simulation decides resource amount/cap, emits state and presentation, and
deletes or expires the orb. `triggerOnceOnly=false` does not permit duplicate
grants because authoritative object consumption is the one-shot latch. For a
dropped orb, object construction also converts its nonzero noun lifetime into
an absolute simulation deadline at object offset `+104`; placed tutorial
capsules have no such 30-second deadline.

The preloaded presentation assets and hashes are:

| Event | Asset ID | Purpose |
| --- | ---: | --- |
| `health_orb_drop.ServerEventDef` | `0x3a67f5c3` | Green-orb drop presentation. |
| `mana_orb_drop.ServerEventDef` | `0x1c4d89b8` | Blue-orb drop presentation. |
| `health_orb_pickup.ServerEventDef` | `0xbb9b1966` | Pickup audio plus `combat_text_health_effect`; requires numeric text value. |
| `mana_orb_pickup.ServerEventDef` | `0xcae95865` | Pickup audio plus `combat_text_mana_effect`; requires numeric text value. |
| `health_orb_full.ServerEventDef` | `0x4f338a4b` | Already-full health feedback. |
| `mana_orb_full.ServerEventDef` | `0xf042f4dc` | Already-full power feedback. |

Client `PopupTip_HealthOrb` and `PopupTip_ManaOrb` do not authorize collection.
They observe the controlled player's replicated HP/mana after nearby-orb hints
and mark their lesson learned only if the value later reaches at least 50% HP
or 20% mana. That is further evidence that resource mutation must precede and
drive client-local tutorial feedback; popup Lua is not a pickup request.

Build-103 registration gives the recovery feedback one corrected wire detail.
After `targetPoint`, the four-byte ServerEvent member at structure offset
`0x58` is named `textValue`; it is reflection field `14`, immediately before
field `15` `clientEventID`. The old Go loot-tail model called field 14
`PlayerIndex`, but the loot event does not have such a registered member.
Pickup events now place the restored amount in field 14 so assets with
`bHasTextValue=true` can render `Health + 15` or the mana equivalent; full
events omit it. A focused build-103 run began at the preserved checkpoint
`(68,-38,20.41953)` with health and power at 185. The capsules rendered in their
authored route line and did not drift. One movement segment crossed two green
capsules: the server recorded object `18` restoring 15 and object `19` as full,
both were deleted, and the client visibly rendered `Health Full`. Crossing blue
object `21` restored the power arc, deleted the capsule, and visibly rendered
`Power + 15`. This confirms noun creation, stopped placement, proximity
authority, resource replication, field-14 numeric text, full feedback, and
one-shot removal in build 103. Audio was not independently captured, and a
post-Sage live pass is still needed to visually confirm both hero records share
the restored resource values.

The build-103 combatant reflection narrows the recovery half further. Wire
`0x97` is logical message `24`; its payload begins with the object ID and then
the two-bit change mask for type `0xfab3107f`. Bit `0` is hit points at
combatant offset `+64`, and bit `1` is mana at `+68`. The native component keeps
its comparison baseline at `+76`, so these are deltas rather than an obligation
to resend the complete component. The native-compatible application recipes
are therefore:

```text
health only: [97][object ID u32][01][HP float32]
mana only:   [97][object ID u32][02][mana float32]
both:        [97][object ID u32][03][HP float32][mana float32]
```

The first two forms are ten bytes including the application ID; the combined
form is fourteen. A normal green pickup changes only HP and a normal blue
pickup changes only mana. An already-full pickup changes neither field and
should not emit `0x97` at all. The packet model now has a separate sparse delta:
damage and green pickups select `0x01`, mana spend and blue pickups select
`0x02`, and full feedback emits no `0x97`. Initial combatant state still uses
the complete `0x03` baseline.
The missing direct logical-24 sender in the client binary still leaves the
health-only and mana-only forms as strong native-compatible derivations rather
than retail capture goldens.

The mutation APIs reinforce the separation between authority and
presentation. Build-103 `SetManaPoints -> sub_A028C0` validates the object and
writes only combatant offset `+68` through `sub_9D8F90`; it does not manufacture
a pickup effect. `FullHealObject -> sub_A02800 -> sub_9E57B0` writes maximum HP
to offset `+64` and only invokes the revival path when the previous HP was not
positive. `HealDamage -> sub_A025C0` uses the general health-change machinery,
but no native evidence makes mana restoration a health event. The orb
`ServerEvent` and the combatant-state delta must therefore remain distinct
outputs of one accepted overlap.

`HealDamage` is not presentation-free: after applying an accepted positive
heal, `sub_9E5D10` calls `sub_9E5800`, which sign-flips the health delta and
passes it to the same `sub_9E4D90 -> sub_A207F0` `CombatEvent` sender used by
damage. The resulting healing event has negative `deltaHealth`, flag bit `1`,
and a signed integer change derived from old HP minus new HP. It selects the
same minimal fields `0/1/3/4/7` and mask `0x9b` as ordinary damage, but flags
are `0x0002` instead of `0x0001`. A subtle cap rule is visible in native code:
the float delta is the negated requested heal, while `integerHpChange` reflects
the actual old-to-new integer change; the actual applied float is only used to
suppress the event when nothing changed. This does **not**
yet prove an orb sends `0xba`: the native `Orb_Pickup` handler has not been
identified, and the pickup `ServerEventDef` already owns its `Health + N`
combat-text asset. The focused darkspin run displayed pickup text without a
healing `CombatEvent`. Do not add a redundant `0xba` by analogy; first recover
or capture whether the retail orb path calls `HealDamage`, writes HP directly,
or suppresses generic healing feedback.

The currently live-proven presentation recipe is also now bounded precisely.
A non-full pickup uses wire `0x9b` with fields `6` asset, `7` active hero object,
`10` authored pickup position, and `14` actual restored amount, then `0xff`;
this is a 30-byte application message. Full feedback omits field `14` and is 25
bytes. Object deletion is wire `0x8e` followed by the consumed object ID. The
working darkspin ordering is presentation event, resource delta, then deletion.
That ordering is live-compatible, but retail ordering is not yet captured. In
all cases the server must first accept a swept overlap and mutate its
authoritative resource value; the client-owned noun trigger is not permission
to heal, refill power, or delete an object.

Retail squad resource ownership remains deliberately unresolved. Blitz object
`1` and Sage object `17` each possess a combatant component, so the `0x97` codec
does not itself select active-only or mirrored ownership. Darkspin now applies
the narrowest evidence-constrained policy: the deployed object that entered the
trigger owns the mutation and receives the sole sparse update; the reserve
retains its independent cached resource. A post-unlock capture must still
determine whether retail instead mirrors a squad pool or copies a resource
during deploy. Parity fixtures should continue to make that evidence boundary
explicit rather than treating the compatibility choice as recovered Lua truth.

Pickup application encoding is transactional with that authoritative mutation.
Each accepted result retains its entrant object, squad slot, prior resource,
resulting resources, and collection identity. The immediate and delayed-contact
paths marshal while holding the session lock. If any pickup in the same movement
command fails to encode, the packet prefix is discarded and every resource and
collection mutation is restored in reverse order. This is an application-state
guarantee only: it does not make later RakNet/UDP publication transactional.

A rejected sparse-update experiment is also useful protocol evidence. Sending
top-level character field `3` under the ordinary `00 10` update header followed
by the three raw `0x620` records crashed at `0x009C8EAB`: the client interpreted
primary noun hash `0x6367B6CD` as a reference-counted pointer. The fixed array
therefore cannot be introduced as that tagged partial shape. The stable full
snapshot path is retained until the exact nested-array change mask is recovered.

## Client entry and cinematic behavior

The post-login scene is `SP_SporeLabs/cSpaceshipState`. Its observed camera
sequence includes:

1. a 14-node, 23.75-second ship tour;
2. the bridge `START` prompt;
3. a six-node, 2.75-second transition after a normal room selection.

With `--skip-cinematic`, Fang completes these splines through the native
cleanup path. Once ship navigation is ready, it checks the account's
new-player progress:

- at `0`, `1000`, or `2000`, it calls native `MapRoomUI.StartGame`, producing
  the tutorial request;
- at `3000` or later, it invokes the normal room-`1` navigation callback;
- after returning from the Hero Editor, it waits for the spaceship controller
  to settle before re-entering room `1` and skipping that short transition.

This distinction is important: cinematic skipping may accelerate a native
transition, but must not substitute the Arsenal callback for the tutorial
launch path.

## Protocol and persistence contract

### Tutorial completion gameplay message

Build 103 registers `TutorialGameMsgs` at string VA `0x0103035C` as gameplay
opcode `0xC8`. The registration sequence at `0x00451C16-0x00451C36` attaches
callback `sub_4510F0`. That callback reads one subtype byte; only subtype `0`
dispatches to `sub_4507C0`, which reads exactly four payload bytes.

For a signed value greater than zero, `sub_4507C0`:

- emits the local `LABS_TUTORIAL_COMPLETE` event;
- stores the raw value as cumulative account XP at account offset `+0x2AA4`;
- calls `sub_9CEA00` to derive a one-based level from the cumulative XP
  threshold vector and stores the result in the account level fields;
- sets two completion/state flags to `1` and sets `new_player_progress` to
  `3000`.

For zero or a negative value, it clears the XP and level-like fields and sets
`new_player_progress` to `2000`. Implementations must therefore never use zero
as a successful-completion placeholder. `sub_9CEA00` compares the supplied
total against the threshold vector at `0x01164CA0-0x01164CA4`; its companion
`sub_9CEA40` maps a level back to its threshold. The account HTTP parser also
maps the literal `xp` into the account response, independently corroborating
the field interpretation.

The leaf handler does not call the account update/submission path. It mutates
client memory and ends through a local state transition, so the packet is best
modeled as a server-authored XP/completion snapshot. It does not establish
whether persistence occurs before the packet, on `RETURN TO SHIP`, during a
later HTTP request, or on account refresh. The enclosing RakNet data,
ACK/NACK, reliability, ordering, and split header formats are now recovered in
`architecture/raknet-gameplay-exchange.md`, but this message's byte order,
priority/reliability/channel, exact sender, and send point remain unresolved.
It also performs no creature, part, loot, or reward mutation; those grants must
arrive through separate gameplay messages or account operations.

A bounded whole-code search for the logical type behind `0xC8` (`0x49` after
subtracting the GMS base `0x7F`) found the receiver registration at
`0x00451C1D`, but no direct `cProtocolTransport::CreateMessage` caller that
constructs type `0x49`. This supports treating the retail client as a receiver
for this server-authored snapshot. It does not rule out an indirect sender;
the next productive anchors are the dev/server executable, generic broadcast
machinery, or a future capture rather than the already-recovered leaf handler.

darkspin models that recovered leaf as a six-byte application packet:
`C8 00 65 00 00 00`, where subtype `0` is followed by positive cumulative XP
`101` in little-endian signed form. A focused live pass sent it immediately
after the defeated alert on the second kill of wave four; the client faded to
a permanent black screen instead of presenting tutorial victory. The codec is
retained, but gameplay no longer emits it at that boundary. Its correct return
interaction placement and enclosing RakNet priority/reliability/order remain
open.

The packaged success prompt is `HUD_BeamOut.swf`, not the failure-only
`HUD_Death.swf`. Package extraction resolves its native class to
`SP_UI/cBeamOut`, its callback to `MaxisBeamOut.OnBeamOutClicked`, and its copy
to `RETURN TO SHIP`. Release IDA shows that the HUD calls
`SetReadyForBeamOut` only after the director event exposed to Lua as
`nGameDirector.IsBossDead` is signaled and the authored delay expires.
`HUD_Death.swf` separately hardcodes `MISSION FAILED`, `COMMAND RELAY
TERMINATED`, and its own return button; it must not be used for tutorial
success.

The build-103 director message is opcode `0x8B` followed immediately by the
seven-field director reflection. Field 3 is `mbBossComplete`, so the minimal
success update is exactly `8B 08 01`. The message-name table also stores
logical enum value `55` beside `kGmsDirectorState`; that metadata is not the
transport opcode. An intermediate live probe incorrectly inserted four zero
bytes before the reflection and did nothing. Removing them produced the clean
in-game Beam Out button after `Horde defeated!`, with no failure overlay.

Clicking Beam Out sends `PlayerStatusUpdate` with status `0x20` and progress
`1`. The reference server handles that status by marking the chain complete,
sending `ReconnectPlayer(ChainVoting)` and `DebugPing`, and then sending
`PlayerDeparted`. A focused build-103 run confirmed that recovered packet
sequence leaves the arena for a departure/chain-voting screen, not the ship.
That screen shows `UNDEFINED` result copy, so its tutorial chain-result payload,
the route onward to the ship, and the final collection-room/reward transition
remain open. darkspin now gates status `0x20`
on the authenticated terminal horde boundary and performs the idempotent
tutorial completion write at that acceptance point. A focused candidate then
answered the click with positive `TutorialGameMsgs` subtype `0` immediately
followed by `AF 00`. It reached a scene-ready/UI-ready ship but crashed with
access violation `0xc0000005`; active generic gameplay treats both messages as
state changes. A later `0.7.2` report showed that `AF 00` alone also reaches the
ship with stale in-memory tutorial progress before the same access violation.
The implemented path therefore sends only the delayed positive tutorial
completion snapshot, whose native handler updates progression and selects the
ship state atomically.

The packaged normal-Cryos `LevelExitPoint.Noun` is not part of the tutorial's
terminal flow. It is an invisible base-effect AI marker at
`(-454.78226,263.62323,60.03500)` in `cryos_1_design_spawners`, and
`Game_Tutorial_cryos_1.Level` does not include that layer. The tutorial's
visible green object is the southern health obelisk: consecutive reference
frames show its intact model, activation/heal effect, and consumed vortex while
`RETURN TO SHIP` remains independently visible. No separate green exit object
needs to be created.

The local-event consumer is now recovered too. `sub_541130` compares incoming
Labs event names at `0x0054116A-0x00541225`. `LABS_TUTORIAL_COMPLETE` pushes
classification `1` into `sub_53B110`; `LABS_ALL_PCS_DIED` uses `0`, and a valid
`LABS_LEVEL_END` uses `2`. A deeper construction-path dump identifies the
target as `SP_SporeLabs/cSurveySystem`, whose adjacent code creates
`SP_UI/cWebBrowserUI` and builds `?surveyID=` requests. This is survey/event
classification, not the mission-result controller. It therefore does not prove
that subtype `0` alone creates the green completion object or `RETURN TO SHIP`,
and a follow-on result/exit event remains possible. The normal-level invisible
exit marker is still ruled out for the tutorial.

darkspin invokes its server-owned completion operation at the confirmed Beam
Out boundary before publishing the ship transition. It advances only pending accounts
to onboarding progress `3000`, raises cumulative account XP/level to at least
`101`/`2` without reducing later progression, and is idempotent. A failed save
restores the account without leaking a partial onboarding transition. Gameplay
then marks the active tutorial instance complete. Blaze remove-player reason
`6` retains the same idempotent completion operation for clients that use that
authenticated return boundary instead of the direct QuickGame transition.

The accepted return publishes the deployed hero's positioned
`character_teleport_beam_out` event followed by its `character_teleport_out`
animation and one positive `TutorialGameMsgs` subtype `0` (`C8 00` plus the
cumulative XP). The native handler writes the in-memory tutorial XP/level and
progress `3000`, then selects ship state `2`. A 2026-08-19 report proved that
stacking `AF 00` after it reaches ship scene-ready and UI-ready but then crashes
build 103 with access violation `0xc0000005`. A 2026-08-29 report proved that
substituting `AF 00` also reaches the ship but leaves the in-memory tutorial
snapshot stale before the same access violation. The terminal path therefore
uses the single positive completion packet for both snapshot and transition.

Tutorial entry also owns a clean mission inventory allocation of exactly `100`
DNA. It is serialized in the initial `LabsPlayerUpdate` field `12` without
mutating persistent account currency, and tutorial restart reconstructs the
same value. Non-tutorial games continue to use the account-owned DNA amount.
This account/Crogenitor progression is distinct from inspectable hero level:
newly activated Blitz and Sage remain level `0` heroes in the ship collection
even though tutorial completion leaves the account at Crogenitor level `3`.

Build 103 parses `tutorial_completed`, but selects the tutorial game mode from
`new_player_progress`. darkspin therefore uses one authoritative value:

```text
onboarding_progress < 3000  => tutorial pending
onboarding_progress >= 3000 => tutorial completed
```

The comparison above remains the recovered client mode-selection contract.
Darkspin's current successful-completion policy deliberately persists `9000`
rather than `3000`: resumable Arsenal/Editor/Navigation lessons have produced
multiple invalid states. The same atomic operation enables the inventory,
ensures Blitz/Sage/Wraith ownership and a full PvE squad, grants the first live
post-tutorial Arsenal activation choice, and grants the known tutorial part.
The activation request consumes that one-time choice; requesting an already
owned starter is idempotent. This is a documented server compatibility
decision, not a claim about retail progression ordering.

The account response derives `tutorial_completed` from that comparison and
serializes `onboarding_progress` as the protocol field
`new_player_progress`. The client submits progress through
`api.account.setNewPlayerStats`. darkspin currently validates but rejects that
submitted progress value: persisted onboarding progress remains read-only
until the authoritative tutorial event flow has been reconstructed. Each
rejected submission is logged with its remote IP, account, value, result, and
the `server_owned` reason so milestones such as `1000` and `2000` remain useful
reverse-engineering evidence.

`new_player_inventory` is a separate submitted scalar. The client sets it to
`1` after a creature/collection condition and can submit it while sending zero
for progress. Its retail meaning remains **unknown**, so it must not be folded
into onboarding progress.

Current darkspin game creation forces tutorial-pending accounts to mode `1`,
level `Game_Tutorial_cryos_1_v2`, and difficulty `0`. Legal forward-only,
idempotent server-owned transitions remain to be implemented once the complete
event sequence is captured.

### Live RakNet entry boundary

A 2026-07-16 build-103 run confirmed that Blaze game setup must advertise the
darkspin gameplay endpoint, not the private endpoint supplied by the client.
Advertising `10.0.0.5:3659` caused an immediate GameManager remove-player
request with reason `6`. Advertising `127.0.0.1:42127` instead made the ship
render normally and allowed `START` to begin the gameplay connection attempt.

The first observed UDP request at that corrected endpoint was 1,464 bytes and
began:

```text
09 0d 04 90 00 13 d5 92 da 79 00 ff ff 00 fe fe
fe fe fd fd fd fd 12 34 56 78 ...
```

Static analysis now parses this as the older single-stage RakNet opening
request: `0x09`, protocol `0x0d`, an eight-byte client GUID, the standard
16-byte offline magic, a six-byte target address, and zero padding. The bytes
`04 90` are the beginning of the GUID, **not an MTU field**. The client tries
UDP payload lengths `1464`, `1172`, and `548`, four times each at 500 ms
intervals. A compatible server must return a `0x0a` reply padded to the
successful request length, then handle a reliable internal `0x04` connection
request. Static analysis confirms that the server then sends internal `0x0e`
and the client answers `0x11`; their address arrays, timestamps, reliability
datagrams, ACK/NACK envelope, and ordered header are now implemented and have
completed in a live build-103 run.

The 2026-07-16 live boundary is later in tutorial loading. darkspin sends
application `Connected`, validates the exact eight-byte hello identity against
Blaze membership, and sends local hello/party state. The client acknowledges a
chain vote/countdown, a split initial `LabsPlayerUpdate` containing a temporary
tutorial Blitz reflection, status `4`, a follow-up player update, and the
16-byte `GamePrepare`. It remains on the black loading spinner and never emits
status `8`. Forcing `GameStart` at status `4` closes the gameplay connection,
so the next blocker is the missing world/player deployment state required by
the build-103 prepare receiver, not the RakNet opening. The full byte layouts,
transport anchors, current gaps, and tutorial-specific `0xc8` boundary are in
[the RakNet/gameplay exchange note](architecture/raknet-gameplay-exchange.md).

## State that onboarding does not replace

- `chain_progression` permanently gates campaign stages using the recovered
  `candidateLevelIndex <= chainProgression + 1` rule.
- `level` and `xp` track Crogenitor/account advancement.
- `creature_rewards` tracks spendable hero-unlock selections.
- creatures, squads, parts, and equipment are persistent collections.
- `unlock_*` values represent account upgrades.
- cap, upsell, and access fields appear entitlement-related.

None of these should be derived solely from `onboarding_progress`. Tutorial
events may award or mutate them, but each mutation needs independent evidence.

## Next capture checklist

For a clean account, record an account response and database snapshot at:

1. login before the ship scene;
2. progress `1000`;
3. progress `2000` immediately before tutorial launch;
4. tutorial game creation and RakNet entry;
5. every audio/trigger lesson in the Cryos tutorial;
6. the first and second Crogenitor level-up, including XP before and after;
7. the `Electro Claws` and `Onyx Barrier` grants before and after inventory persistence;
8. the Sage-unlock trigger, immediate hero-switch packet, and post-return creature ownership;
9. teleporter lock/unlock and the event that opens the horde arena;
10. tutorial victory before and after the write to `3000`;
11. collection-room arrival and the writes to `4000` and `5000`;
12. editor entry/exit and the write to `6000`;
13. map-room unlock at `6500`;
14. the `6800`, `8000`, and `9000` planet-screen transitions.

At each point capture `new_player_inventory`, account level/XP, creature
ownership, squad membership, reward tokens, parts, and all client/server event
IDs. This will reveal which tutorial rewards are genuine server mutations and
which are only presentation state.

## Source map

- Quadra perception and first-aggro stimulus boundary:
  [`quadra-perception.md`](quadra-perception.md)
- Cryos opening-enemy placement, activation, and attack boundary:
  [`cryos-opening-enemy-ownership.md`](cryos-opening-enemy-ownership.md)
- Post-Quadra teleporter/director ownership:
  [`post-quadra-route.md`](post-quadra-route.md)
- Teleporter modifier live/native lifecycle contract:
  [`teleporter-live-contract.md`](teleporter-live-contract.md)
- Post-teleport arena director and retail/live boundary:
  [`arena-director.md`](arena-director.md)
- Binary progression analysis:
  [`confirmed/game-5.3.0.103-progression.md`](confirmed/game-5.3.0.103-progression.md)
- Ship, cinematic, and tutorial-entry analysis:
  [`confirmed/game-5.3.0.103-first-pass.md`](confirmed/game-5.3.0.103-first-pass.md)
- Wider campaign and progression model:
  [`game-design/progression-and-campaign.md`](game-design/progression-and-campaign.md)
- Hero/account-level tutorial cadence:
  [`game-design/leveling-experience.md`](game-design/leveling-experience.md)
- Research and video-capture tracker:
  [`architecture/reverse-engineering-research-roadmap.md`](architecture/reverse-engineering-research-roadmap.md)
- Tutorial walkthrough index:
  [`../bin/video/walkthrough/list.md`](../bin/video/walkthrough/list.md)
- Extracted tutorial level:
  `bin/server/data/level/Game_Tutorial_cryos_1.Level.xml`

## Immediate bytecode parity fixtures

1. **Chunk `865` invisible lifecycle.** It needs no additional VM opcode;
   exercise direct no-upvalue `CLOSURE` callbacks. Bind
   `nBehaviorTree.GetMyObjectID`, `nGameObject.SetIsVisible`,
   `nAttribute.AddAttributeModifier`, `nAttribute.RemoveAttributeModifier`,
   `nThreadData.SetInt/GetInt`, and `nThread.WaitForever`, with typed
   `nAttributeType.Intangible=29` and
   `InvisibleToSecurityTeleporters=112`. Assert handle slots `1/2`, indefinite
   tick ownership on the object's `+676` Lua thread, the native
   Activate -> prepared Tick sequence, a 16-integer thread-data block, and
   visible false/true ordering. Deactivation must call callback `3`, remove both
   handles, then reset the thread and clear its thread-data pointer; initialize
   the deployed fixture with the noun-proven team byte `0`. Add a scheduler
   subfixture: seed `preAggroIdle=nBehavior_Invisible`, start chunk `865`, then
   select an explicit higher-priority test leaf and assert old-leaf Deactivate
   completes before new-leaf Activate. Separately assert reflected offsets
   `firstAggroAbility=0x34`, `firstAlertAbility=0x54`,
   `preAggroIdle=0x7c`, and `passiveIdle=0x170`; do not claim stimulus `0x20`
   owns that replacement until the external engine selector edge is recovered.
   The static node event expression is already proven not to contain a `0x20`
   gate.
2. **Chunk `349` activation timeline.** It needs no additional VM opcode.
   Bind `nModifier.RegisterModifier`, `GetMyAgentID`,
   `GetModifierInstanceID`, `GetFloatProperty`, `nLocomotion.Stop`,
   `nAttribute.AddAttributeModifier`, `nGameObject.SetAnimationState`,
   `TeleportObject`, and `nThread.WaitForXSeconds`. Assert the `0.5s`, `0s`, and `0.5s`
   checkpoints on the yielding Activate callback, destination property order,
   the absence of Tick, and the empty Lua deactivate callback. Supply chunk
   `659`'s `nActivationType.Unique=1`; assert native same-GUID
   deactivate/cleanup/remove-before-replacement and retained-handle cleanup on
   explicit release. Assert a 32-bit `(generation << 16) | slot` instance ID
   from the 2,048-entry pool and `ModifierCreated` fields duration `0`,
   overdrive `1`, stack `1`, source `0`, and bound `1` before teleport-out.
   After the third wait, assert normal coroutine completion
   produces dormant thread state `-1` while the modifier and its thread ID
   remain owned; explicit release must invoke Deactivate, release that thread,
   and clear instance `+40`. Do not make Activate return imply deletion or a
   `1.5s` modifier lifetime; the external normal-release trigger is still
   unresolved. Supply `nDeactivationType.Default=0` and assert that neither
   `OnAgentDestroyed=1` nor `OnAgentDeath=2` is substituted.
3. **Chunk `144` complete passive.** Add VM `LT` and generic-for `TFORLOOP`
   plus the `ipairs` iterator contract; retained `CLOSURE` upvalues and
   `GETUPVAL` are already implemented. Bind private tables,
   player-control/owner predicates, `GetTeleporterDestination`,
   `Create/DestroyTriggerVolume`, `RequestModifier`,
   `GetFirstModifierByGUID`, `GetObjectsInRadius`, `IsAlive`, `GetTeam`,
   `GetAttributeValue`, effects, and timed waits. Supply the recovered chunk
   `659` globals through a typed shim; executing chunk `659` itself additionally
   requires VM `SETLIST`. Assert GUID `0x502f1932`, radius `20`, all three
   threat filters, callback retention/order, stable `0.5s` polling,
   active-to-inactive `1s`, inactive-to-active `1s`, and typed rejection of
   an unresolved `(0,0,0)` destination. Add an exact `EQ A=0` branch fixture:
   callback three must suppress `RequestModifier` while a matching instance
   exists and invoke the captured entry callback only when
   `GetFirstModifierByGUID` returns `kObjIDNone`. Drive native trigger masks
   `1/2/4` as entry/exit/repeated-stay, pass `(triggerID, entrantObjectID)` to
   each closure, and do not replace bit-`4` physics notifications with an
   invented fixed timer.
